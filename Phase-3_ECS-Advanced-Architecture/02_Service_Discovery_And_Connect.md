# Service Discovery & ECS Service Connect

---

## Problem Statement

ECS task IP addresses change constantly. When a task restarts, it gets a new IP address. How does Service A reliably find and communicate with Service B?

```
user-service task: Today   → 10.0.1.54
                   Tomorrow → 10.0.1.91  (task restarted)

How does payment-service find user-service reliably?
```

This is a fundamental challenge in microservices architecture. Three solutions exist in the AWS ECS ecosystem, each with different tradeoffs.

---

## Solution 1: Cloud Map + Service Discovery

```
AWS Cloud Map = Service registry (DNS-based)

Registration:
  ECS task starts → automatically registers in Cloud Map
  task IP: 10.0.1.54 → record: user-service.myapp.local → 10.0.1.54

  New task starts → Cloud Map: user-service.myapp.local → [10.0.1.54, 10.0.1.91]
  Task dies → auto-deregistered

Service B calls:
  dns.resolve('user-service.myapp.local')
  → ['10.0.1.54', '10.0.1.91']
  → Pick one (client-side load balancing)
```

Cloud Map provides a managed DNS namespace within your VPC. ECS integrates with Cloud Map natively — when you link an ECS service to a Cloud Map service, the ECS Container Agent automatically registers and deregisters task IP addresses as tasks start and stop.

### Cloud Map Setup:
```bash
# Create Cloud Map namespace
aws servicediscovery create-private-dns-namespace \
  --name myapp.local \
  --vpc vpc-abc123

# Create service in namespace
aws servicediscovery create-service \
  --name user-service \
  --namespace-id ns-abc123 \
  --dns-config 'NamespaceId=ns-abc123,RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=10}]' \
  --health-check-custom-config FailureThreshold=1

# Link to ECS service:
aws ecs create-service \
  --cluster production \
  --service-name user-service \
  --task-definition user-service:5 \
  --service-registries '[{
    "registryArn": "arn:aws:servicediscovery:...:service/srv-abc123",
    "port": 3000
  }]'
```

---

## Solution 2: ECS Service Connect (Recommended — Newer)

```
Service Connect = ECS-native service mesh
Built on Envoy proxy (like Istio, but simpler)

Architecture:
┌─────────────────────────────────────────────┐
│  Task (user-service)                         │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │   app process   │  │  Envoy Proxy     │  │
│  │   port: 3000    │  │  (injected by    │  │
│  └────────┬────────┘  │   ECS)           │  │
│           │            └──────────┬───────┘  │
│           │  localhost:80         │          │
└───────────┼──────────────────────┼──────────┘
            │                      │
            │             ↑ outbound calls to payment-service
            │             ↓ inbound calls → routed via Envoy
            └────── 127.0.0.1:80 ─►[ proxy ] → payment-service
```

### Service Connect Advantages over Cloud Map:
```
✅ Built-in connection pooling
✅ Automatic retries (configurable)
✅ Circuit breaking
✅ TLS encryption between services (mTLS!)
✅ Built-in metrics (connection count, request count, latency per service)
✅ No client-side load balancing code needed
✅ Works with any language/framework
```

ECS Service Connect injects an Envoy proxy sidecar into every task automatically — you do not write any proxy configuration. The Envoy proxy intercepts all inbound and outbound traffic, adds retry logic, applies circuit breaking rules, enforces mTLS, and reports per-route metrics to CloudWatch. Your application code simply makes HTTP calls using the logical service name.

### Service Connect Setup:
```bash
# Create service with Service Connect enabled
aws ecs create-service \
  --cluster production \
  --service-name user-service \
  --task-definition user-service:5 \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "myapp.local",
    "services": [{
      "portName": "http",           # Name in task def portMappings
      "discoveryName": "user-service",  # How others find this service
      "clientAliases": [{
        "port": 80,                 # Port clients use to call this service
        "dnsName": "user-svc"       # DNS name clients use
      }]
    }]
  }'

# Task definition port must have a name:
{
  "portMappings": [{
    "name": "http",           # Used in Service Connect config above!
    "containerPort": 3000,
    "protocol": "tcp"
  }]
}

# Now from payment-service container:
curl http://user-svc:80/api/users/123
# → Envoy proxy intercepts → routes to user-service → returns response
# Automatic: retry, circuit break, metrics!
```

---

## Analogy — Hotel Concierge

**Cloud Map = Phone Directory:**
- "What is user-service's number?"
- Directory says: "Try 10.0.1.54 or 10.0.1.91"
- You make the call yourself
- If the number is busy, that is your problem (no retry help)

**Service Connect = Hotel Concierge:**
- You say: "I need to reach user-service"
- Concierge handles: finding the right person, dialing, retrying if busy, tracking the call
- You do not deal with raw IP addresses
- Concierge reports: "You made 500 calls today, average wait 200ms"

---

## Service Discovery vs Service Connect vs ALB

| | Cloud Map | Service Connect | ALB |
|--|-----------|----------------|-----|
| Protocol | TCP/UDP/HTTP | HTTP/TCP | HTTP/HTTPS |
| Load Balancing | Client-side | Proxy-side | Server-side |
| Retries | Manual | Automatic | Automatic |
| mTLS | Manual | Built-in | Via cert manager |
| Metrics | Basic | Per-route | Per-target group |
| External traffic | No | No | Yes |
| Internal S2S | Yes | Yes (better) | Overkill |
| Complexity | Low | Medium | Low |

**Rule of thumb:**
- External traffic (internet to ECS) → ALB
- Internal service-to-service → Service Connect
- Legacy or simple setups → Cloud Map

---

## Gotchas

### TTL Too High = Stale IPs
```
Cloud Map DNS TTL = 60s (default)
Task restarted (new IP) → DNS cache still has old IP for 60 seconds
→ Requests to dead IP for 60 seconds!

Fix: Lower TTL to 10 or 5 seconds (tradeoff: more DNS queries)
Or: Use Service Connect (Envoy handles stale IP detection automatically)
```

### Service Connect Namespace Must Match Across Services
All services that need to communicate via Service Connect must be in the same Cloud Map namespace. If you create services with different namespaces, they cannot discover each other. Define a single namespace (e.g., `myapp.local`) at the cluster level and use it consistently.

### Envoy Sidecar Resource Overhead
Service Connect injects an Envoy proxy container into every task. This Envoy sidecar consumes CPU and memory. For very small tasks (0.25 vCPU, 512 MB), the Envoy overhead may be significant relative to the task size. Account for this in your task sizing and cost estimates. Typically Envoy uses 32-64 CPU units and 64-128 MB memory.

### Service Connect Does Not Support UDP
Service Connect currently supports only TCP-based protocols (HTTP/1.1, HTTP/2, gRPC). If your services communicate over UDP, you must use Cloud Map instead.

### Health Check Propagation Delay
When a Cloud Map-registered task becomes unhealthy and is deregistered, there can be a propagation delay of several seconds before DNS resolvers stop returning the old IP. Client applications should implement connection-level retry logic to handle this gracefully, even when using Cloud Map.

---

## Interview Angle

**Q: "How do microservices in ECS find each other? How does service discovery work?"**

> ECS Service Discovery (Cloud Map): When a task starts, it automatically registers a DNS A record in the Cloud Map namespace (e.g., `user-service.myapp.local`). Service B resolves this DNS name to get the list of task IPs and calls one directly. If a task restarts and gets a new IP, Cloud Map updates the DNS record automatically. The limitation is that the calling service handles load balancing itself, and DNS TTL means stale IPs can be cached briefly.
>
> ECS Service Connect (newer, recommended): ECS automatically injects an Envoy proxy into each task. Service B calls `http://user-svc:80` — the local Envoy intercepts the call, looks up the current healthy instances from the ECS control plane, and routes the request. Service Connect provides built-in retries, circuit breaking, mTLS encryption, and per-route metrics in CloudWatch — all without any application code changes.
>
> For external traffic coming from the internet, use an ALB. For internal service-to-service communication, use Service Connect. Cloud Map is appropriate for simpler setups or when you need UDP support.

---

*Next: [03_Deployment_Strategies.md →](./03_Deployment_Strategies.md)*
