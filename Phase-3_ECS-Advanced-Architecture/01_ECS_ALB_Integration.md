# 🌐 ECS + ALB Integration — Deep Dive

---

## 📖 Concept

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

## 🏗️ Internal Architecture — ECS + ALB Registration

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

---

## 🔧 ALB Listener Rules

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

---

## 🏥 Health Checks — Deep Dive

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
TWO health checks! Don't confuse them:

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

---

## 🍯 Sticky Sessions

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

---

## ⚙️ Complete ALB + ECS Setup:
```bash
# Step 1: Create Target Group
aws elbv2 create-target-group \
  --name myapp-tg \
  --protocol HTTP \
  --port 3000 \
  --vpc-id vpc-abc123 \
  --target-type ip \           ← "ip" for awsvpc/Fargate, "instance" for bridge
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
    "containerName": "app",        ← Must match container name in task def!
    "containerPort": 3000          ← Must match containerPort in task def!
  }]'
```

---

## 🚨 Gotchas & Edge Cases

### 1. ALB Target Type: `ip` vs `instance`
```
bridge mode: target-type=instance (ALB → EC2 IP → NAT to container)
awsvpc mode: target-type=ip (ALB → Task IP directly!)
WRONG TYPE = health checks fail, no traffic!
```

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

---

## 🎤 Interview Angle

**Q: "ECS task ALB ke saath kaise register hota hai? Zero-downtime deployment kaise hota hai?"**

> Task starts → passes ECS health check → ECS Container Agent calls ALB API to register task IP.
> ALB apne health checks chalata hai → task healthy → traffic forward karna shuru.
> 
> Deployment: minimumHealthyPercent=50 + maximumPercent=200 ke saath:
> New tasks launch + register with ALB → healthy hone par old tasks deregister (drain connections for deregistrationDelay seconds) → stop.
> Traffic never interrupted as long as some tasks are healthy through transition.

---

*Next: [02_Service_Discovery_And_Connect.md →](./02_Service_Discovery_And_Connect.md)*
