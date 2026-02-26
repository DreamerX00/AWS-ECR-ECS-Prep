# 🔍 Service Discovery & ECS Service Connect

---

## 📖 Problem Statement

ECS tasks ke IPs change hote rehte hain. Service A ko Service B ka address kaise pata chalega?

```
user-service task: Today   → 10.0.1.54
                   Tomorrow → 10.0.1.91  (task restarted)
                   
payment-service kaise find karega user-service?
```

---

## 🗺️ Solution 1: Cloud Map + Service Discovery

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

## 🌐 Solution 2: ECS Service Connect (Recommended — Newer)

```
Service Connect = ECS-native service mesh
Built on Envoy proxy (like Istio, but simpler)

Architecture:
┌─────────────────────────────────────────────────┐
│  Task (user-service)                             │
│  ┌─────────────────┐  ┌──────────────────────┐  │
│  │   app process   │  │  Envoy Proxy (auto)  │  │
│  │   port: 3000    │  │  (injected by ECS)   │  │
│  └────────┬────────┘  └──────────┬───────────┘  │
│           │  localhost:80         │               │
└───────────┼────────────────────  │ ──────────────┘
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
✅ No client-side LB code needed
✅ Works with any language/framework
```

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
      "portName": "http",           ← Name in task def portMappings
      "discoveryName": "user-service",  ← How others find this service
      "clientAliases": [{
        "port": 80,                 ← Port clients use to call this service
        "dnsName": "user-svc"       ← DNS name clients use
      }]
    }]
  }'

# Task definition port must have a name:
{
  "portMappings": [{
    "name": "http",           ← Used in Service Connect config above!
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

## 🎯 Analogy — Hotel Concierge 🏨

**Cloud Map = Phone Directory:**
- "What's user-service's number?"  
- Directory says: "Try 10.0.1.54 or 10.0.1.91"
- You make the call yourself
- If number busy → your problem (no retry help)

**Service Connect = Hotel Concierge:**
- You say: "I need to reach user-service"
- Concierge handles: finding the right person, dialing, retrying if busy, tracking the call
- You don't deal with raw phone numbers
- Concierge reports: "You made 500 calls today, avg wait 200ms"

---

## ⚙️ Service Discovery vs Service Connect vs ALB

| | Cloud Map | Service Connect | ALB |
|--|-----------|----------------|-----|
| Protocol | TCP/UDP/HTTP | HTTP/TCP | HTTP/HTTPS |
| Load Balancing | Client-side | Proxy-side | Server-side |
| Retries | Manual | ✅ Automatic | ✅ |
| mTLS | Manual | ✅ Built-in | ✅ (cert mgr) |
| Metrics | Basic | ✅ Per-route | ✅ |
| External traffic | ❌ | ❌ | ✅ |
| Internal S2S | ✅ | ✅ (better) | Overkill |
| Complexity | Low | Medium | Low |

**Rule of thumb:**
- External traffic (internet → ECS) → ALB
- Internal service-to-service → Service Connect
- Legacy/simple → Cloud Map

---

## 🚨 Gotchas

### TTL Too High = Stale IPs
```
Cloud Map DNS TTL = 60s (default)
Task restarted (new IP) → DNS cache still has old IP for 60 seconds
→ Requests to dead IP for 60 seconds!

Fix: Lower TTL to 10 or 5 seconds (trade: more DNS queries)
Or: Use Service Connect (Envoy handles this automatically)
```

---

## 🎤 Interview Angle

**Q: "ECS mein microservices ek dusre ko kaise dhundhte hain? Service Discovery kaise kaam karta hai?"**

> ECS Service Discovery (Cloud Map): Task starts → DNS record register hota hai namespace mein. Service B DNS lookup karta hai → IPs milte hain → request direct.
> ECS Service Connect (newer, recommended): Envoy proxy automatically inject hota hai har task mein. Service B calls http://service-a:80 → local Envoy intercepts → routes to Service A ka Envoy → Service A. Built-in retry, circuit break, mTLS, metrics.
> ALB: External traffic ke liye ALB, internal S2S ke liye Service Connect.

---

*Next: [04_Failure_Scenarios.md →](./04_Failure_Scenarios.md)*
