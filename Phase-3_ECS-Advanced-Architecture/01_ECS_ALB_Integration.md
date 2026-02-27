# ECS + ALB Integration — Deep Dive

---

## Concept

ALB (Application Load Balancer) + ECS = Production-Grade Traffic Management

```
Internet
   │
   ▼
┌──────────────────────────────────────────────────────┐
│          Application Load Balancer                    │
│                                                       │
│  Listener: HTTPS 443                                  │
│  ├── Rule: /api/users/* → Target Group: user-service  │
│  ├── Rule: /api/payments/* → Target Group: payment-svc│
│  └── Default: → Target Group: frontend               │
└──────────────────────────────────────────────────────┘
         │              │                │
         ▼              ▼                ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │User Task1│   │Pay Task1 │   │Front Task │
   │10.0.1.54 │   │10.0.2.33 │   │10.0.3.12  │
   └──────────┘   └──────────┘   └──────────┘
   ┌──────────┐   ┌──────────┐
   │User Task2│   │Pay Task2 │
   │10.0.1.88 │   │10.0.2.91 │
   └──────────┘   └──────────┘
```

---

## Internal Architecture — ECS + ALB Registration

### How ECS Auto-Registers Tasks with ALB:

```
1. New ECS task starts
          │
          ▼
2. Task passes health check (startPeriod)
          │
          ▼
3. ECS Container Agent calls ALB API:
   RegisterTarget(targetGroupArn, ipAddress, port)
          │
          ▼
4. ALB health checks task directly (task IP:port)
   - HTTP GET /health → 200 OK → HEALTHY
   - If fails → task not included in rotation
          │
          ▼
5. Task serves live traffic!

When task stops:
1. ECS deregisters from ALB: DeregisterTarget()
2. ALB drains connections (deregistrationDelay = 30s default)
3. In-flight requests complete
4. Task stops (SIGTERM → graceful shutdown)
```

The key insight here is that ECS manages the entire lifecycle of ALB registration automatically. You do not need to write any registration code in your application. The ECS Container Agent handles all calls to the Elastic Load Balancing API on your behalf.

---

## ALB Listener Rules

### Path-Based Routing:
```bash
# Create listener rule for /api/users/*
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/myalb/xxx \
  --priority 10 \
  --conditions '[
    {"Field": "path-pattern", "Values": ["/api/users/*"]}
  ]' \
  --actions '[
    {"Type": "forward", "TargetGroupArn": "arn:...targetgroup/user-service-tg/..."}
  ]'

# Host-based routing (different domains):
aws elbv2 create-rule \
  ...
  --conditions '[
    {"Field": "host-header", "Values": ["api.example.com"]}
  ]' \
  --actions '[{"Type": "forward", "TargetGroupArn": "...api-tg..."}]'
```

### Internal vs Internet-Facing ALB:
```
Internet-Facing ALB:
  - Public IP/DNS
  - Accessible from internet
  - scheme: internet-facing
  - Use for: user-facing APIs

Internal ALB:
  - Private IP only
  - Only accessible within VPC
  - scheme: internal
  - Use for: service-to-service communication

Example:
  Internet → internet-facing ALB → user-service →
                              internal ALB → payment-service (internal only!)
```

A common production pattern is to have one internet-facing ALB for external traffic and one (or more) internal ALBs for service-to-service communication. This limits the blast radius of any security incident and reduces the attack surface of internal services.

---

## Health Checks — Deep Dive

```json
// Target Group health check settings:
{
  "Protocol": "HTTP",
  "Port": "traffic-port",     // Same port as task
  "Path": "/health",
  "HealthCheckIntervalSeconds": 30,  // Check every 30s
  "HealthCheckTimeoutSeconds": 5,    // Timeout after 5s
  "HealthyThresholdCount": 2,        // 2 consecutive successes = healthy
  "UnhealthyThresholdCount": 3,      // 3 consecutive failures = unhealthy
  "Matcher": {"HttpCode": "200-299"} // 2xx = healthy
}
```

### ECS Health Check vs ALB Health Check:
```
TWO health checks! Do not confuse them:

1. ECS (Container) Health Check:
   - Defined in Task Definition
   - Runs INSIDE the container
   - CMD: curl -f http://localhost:3000/health
   - If fails → ECS marks container UNHEALTHY → task stopped

2. ALB Health Check:
   - Runs from the ALB to the task IP
   - Network call from ALB
   - If fails → task removed from rotation (but NOT stopped by ECS!)
   - ECS Service integration: failing ALB health = ECS considers for replacement

startPeriod in ECS health check:
  - Grace period when container starts
  - ECS ignores health check failures during startPeriod
  - Prevents healthy containers from being killed before app finishes booting
  - Set to: (app boot time) + 30 seconds buffer
```

Understanding both health checks is critical. A common mistake is configuring only one and wondering why the other behaves unexpectedly. The ECS container health check is about whether the process inside the container is alive. The ALB health check is about whether the container can serve network traffic. Both must pass for a task to receive production traffic.

---

## Sticky Sessions

```json
// Enable sticky sessions on Target Group:
{
  "StickynessDuration": 86400,   // 24 hours
  "StickyType": "lb_cookie"      // ALB-managed cookie
}

// Use case: Stateful apps, WebSocket connections
// Downside: Uneven load distribution!

// Better: Use Redis/DynamoDB for state → stateless services → no sticky needed!
```

Sticky sessions cause problems at scale because they create uneven traffic distribution. If one task accumulates many sticky sessions and the cluster scales in, all those users will experience a session reset. The recommended approach is to externalize all session state to Redis or DynamoDB so that any task can serve any user.

---

## Complete ALB + ECS Setup:
```bash
# Step 1: Create Target Group
aws elbv2 create-target-group \
  --name myapp-tg \
  --protocol HTTP \
  --port 3000 \
  --vpc-id vpc-abc123 \
  --target-type ip \           # "ip" for awsvpc/Fargate, "instance" for bridge
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Step 2: Create ECS Service linked to Target Group
aws ecs create-service \
  --cluster production \
  --service-name myapp \
  --task-definition myapp:5 \
  --desired-count 3 \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/myapp-tg/...",
    "containerName": "app",        # Must match container name in task def!
    "containerPort": 3000          # Must match containerPort in task def!
  }]'
```

---

## Gotchas & Edge Cases

### 1. ALB Target Type: `ip` vs `instance`
```
bridge mode: target-type=instance (ALB → EC2 IP → NAT to container)
awsvpc mode: target-type=ip (ALB → Task IP directly!)
WRONG TYPE = health checks fail, no traffic!
```

This is one of the most common setup mistakes. Fargate always uses `awsvpc` networking and therefore always requires `target-type=ip`. If you use `target-type=instance` with Fargate, health checks will fail immediately because ALB will try to connect to the EC2 instance IP rather than the task IP.

### 2. Security Groups Must Allow ALB → Task
```
ALB → Task traffic:
  ALB SecurityGroup: SG-alb (allow outbound 3000 to sg-task)
  Task SecurityGroup: sg-task (allow inbound 3000 from sg-alb)

NEVER: Open tasks to 0.0.0.0/0!
ALWAYS: Source from ALB security group specifically.
```

### 3. Deregistration Delay (Connection Draining)
```
Default: 300 seconds (5 minutes!)
During rolling deployment → old tasks registered for 5 min before being killed!
= deployments take forever with long-running connections

Tune down for stateless HTTP apps:
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:...:targetgroup/myapp-tg/... \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

The 300-second default deregistration delay is designed for long-lived TCP connections. For typical REST APIs with short HTTP requests, this is excessive and will make every deployment take at least 5 minutes longer than necessary. Set it to 30 seconds for HTTP/REST services. Keep it higher (120-300 seconds) for services that handle WebSockets or long-polling connections.

### 4. Multiple Target Groups per Service

An ECS service can be registered with multiple target groups simultaneously (up to 5). This is useful for:
- Serving on both HTTP and HTTPS
- Using different ALBs for different traffic types (e.g., internal vs external)
- A/B testing with weighted routing between target groups

### 5. ALB Access Logs for Debugging

Enable ALB access logs to an S3 bucket for detailed debugging. Access logs capture every request including response times, status codes, and the specific target that served each request. This is invaluable when diagnosing why certain requests fail.

---

## Interview Angle

**Q: "How does an ECS task register with the ALB? How is zero-downtime deployment achieved?"**

> When a task starts, it passes the ECS container health check, after which the ECS Container Agent calls the ALB RegisterTarget API with the task's IP address and container port. The ALB then begins health checking the task directly. Once the ALB marks the task healthy, traffic is forwarded to it.
>
> For zero-downtime deployment with minimumHealthyPercent=50 and maximumPercent=200: new tasks launch and register with the ALB. Once they pass health checks and receive the healthy status, old tasks are deregistered. The ALB drains existing connections over the deregistrationDelay period, then the old tasks stop. Traffic is never interrupted because some healthy tasks are always serving throughout the transition.

---

*Next: [02_Service_Discovery_And_Connect.md →](./02_Service_Discovery_And_Connect.md)*
