# 🧠 Advanced Architecture Patterns

---

## 1️⃣ Sidecar Pattern

### What is it?
```
Attach a "helper" container to your main app container.
Same task → same network → work together.

Main container: Does the core business logic
Sidecar:        Handles cross-cutting concerns (logging, metrics, TLS, etc.)
```

### Pattern A: Log Router Sidecar
```json
{
  "containerDefinitions": [
    {
      "name": "app",
      "image": "myapp:v1",
      "logConfiguration": {
        "logDriver": "awsfirelens"   ← Sends logs to sidecar!
      }
    },
    {
      "name": "log-router",
      "image": "amazon/aws-for-fluent-bit:latest",
      "essential": false,
      "firelensConfiguration": {
        "type": "fluentbit"
      }
      // Receives logs from app, routes to CloudWatch + S3 + OpenSearch
    }
  ]
}
```

### Pattern B: Service Mesh Proxy (Envoy)
```json
// ECS Service Connect does this automatically
// But manual approach (App Mesh):
{
  "containerDefinitions": [
    {"name": "app", "image": "myapp:v1"},
    {
      "name": "envoy",
      "image": "840364872350.dkr.ecr.us-east-1.amazonaws.com/aws-appmesh-envoy:latest",
      "essential": false
      // Intercepts all network traffic
      // Adds: mTLS, retries, circuit breaking, observability
    }
  ]
}
```

### Pattern C: Monitoring Sidecar
```json
{
  "containerDefinitions": [
    {"name": "app", "image": "myapp:v1"},
    {
      "name": "datadog-agent",
      "image": "datadog/agent:latest",
      "essential": false,
      "environment": [
        {"name": "DD_API_KEY", "value": "..."},
        {"name": "ECS_FARGATE", "value": "true"}
      ]
      // Scrapes metrics from app via StatsD
      // Collects container metrics automatically
    }
  ]
}
```

---

## 2️⃣ Daemon Service Pattern

```
"Run exactly one task per EC2 instance, always."

Use cases:
  - Log collection agent (one per host)
  - Node-level metrics collector
  - Security scanner
  - EBS volume backup agent
```

```bash
aws ecs create-service \
  --cluster production \
  --service-name fluentbit-daemon \
  --task-definition fluentbit:5 \
  --scheduling-strategy DAEMON   ← Magic!
  # No desiredCount needed (auto = number of instances)

# Task def: Give daemon access to host paths
# volumes:
#   - host path: /var/log (to read all container logs from host)
#   - host path: /var/run/docker.sock (to get container metadata)
```

```json
// Task definition for daemon log collector:
{
  "volumes": [{
    "name": "varlog",
    "host": {"sourcePath": "/var/log"}
  }],
  "containerDefinitions": [{
    "name": "log-collector",
    "image": "fluentbit:latest",
    "mountPoints": [{
      "sourceVolume": "varlog",
      "containerPath": "/var/log/host"   ← access host /var/log
    }]
  }]
}
```

---

## 3️⃣ Multi-Container Task Pattern

```
Multiple containers in ONE task for tightly coupled functionality.

When to use:
  - Containers share localhost network (communicate via 127.0.0.1)
  - Containers share volumes (one writes, one reads)
  - Containers need to start together (init container)
  - Sidecar patterns (any pattern above)
```

### Init Container Pattern
```json
// Start a short-lived container before the main app
{
  "containerDefinitions": [
    {
      "name": "migrate",            // Run DB migrations first!
      "image": "myapp:v1",
      "command": ["node", "migrate.js"],
      "essential": false,           // Not essential - will exit when done
      "dependsOn": []
    },
    {
      "name": "app",
      "image": "myapp:v1",
      "essential": true,
      "dependsOn": [{
        "containerName": "migrate",
        "condition": "SUCCESS"    ← Wait for migrations to succeed!
      }]
    }
  ]
}
```

### Nginx + App Pattern
```json
{
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:alpine",
      "portMappings": [{"containerPort": 80}],
      "essential": true
      // nginx.conf: proxy_pass http://127.0.0.1:3000
      // Handles: TLS termination, caching, rate limiting
    },
    {
      "name": "app",
      "image": "myapp:v1",
      // No port mapping exposed! Only reachable via nginx on 127.0.0.1
      "essential": true
    }
  ]
}
// awsvpc: both containers share same namespace → can use localhost
```

---

## 4️⃣ Event-Driven ECS Pattern

```
Trigger ECS tasks from SQS/SNS/EventBridge events.

Architecture:
  SQS Queue → EventBridge Pipe → ECS Task
  (message arrives) → (trigger) → (process)

Use case: Video transcoding, report generation, email sending
```

```bash
# EventBridge Pipe: SQS → ECS Task
aws pipes create-pipe \
  --name "video-transcode-pipe" \
  --source arn:aws:sqs:us-east-1:123:video-jobs-queue \
  --target arn:aws:ecs:us-east-1:123:cluster/production \
  --target-parameters '{
    "EcsTaskParameters": {
      "TaskDefinitionArn": "arn:aws:ecs:...:task-definition/video-transcoder:3",
      "TaskCount": 1,
      "LaunchType": "FARGATE",
      "NetworkConfiguration": {
        "AwsvpcConfiguration": {
          "Subnets": ["subnet-abc"],
          "SecurityGroups": ["sg-xyz"],
          "AssignPublicIp": "DISABLED"
        }
      },
      "Overrides": {
        "ContainerOverrides": [{
          "Name": "transcoder",
          "Environment": [{
            "Name": "VIDEO_ID",
            "Value": "$.body.videoId"   ← From SQS message!
          }]
        }]
      }
    }
  }' \
  --role-arn arn:aws:iam::123:role/pipes-execution-role
```

---

## 5️⃣ Blue/Green Deployment Pattern with CodeDeploy (Architecture View)

```
ALB Production Listener (443)
         │
         ├── [Blue] Target Group → Blue Tasks (current production)
         └── [Green] Target Group → Green Tasks (new version)

Deployment Flow:
  Step 1: Launch green tasks (0% traffic)
  Step 2: Test listener (8080) → 100% traffic to green
  Step 3: Run automated tests via test listener!
  Step 4: If tests pass → Production listener switch (5% canary first, then 100%)
  Step 5: Blue tasks: keep for 30 minutes (rollback window)
  Step 6: Terminate blue tasks

Rollback:
  One command → switch listener back to blue
  < 1 second switchback!
```

---

## 🌍 Real-World: Complete Production Architecture

```
Production Architecture for 10M users:

┌──────────────────────────────────────────────────────────────┐
│  AWS Region: us-east-1                                         │
│                                                               │
│  Route53 → CloudFront (CDN)                                   │
│       │                  │                                    │
│       ▼                  ▼                                    │
│  Internet-facing ALB    S3 (static assets)                    │
│       │                                                       │
│  ┌────┼────────────────────────────────┐                      │
│  │    │   ECS Cluster "production"      │                      │
│  │    │                                │                      │
│  │  /api/users/* → user-service         │ ← Fargate, 5 tasks  │
│  │  /api/orders/* → order-service       │ ← Fargate, 10 tasks │
│  │  /api/payments/* → payment-service   │ ← Fargate, 3 tasks  │
│  │    │                                │                      │
│  │    │  [sidecar: fluent-bit]         │                      │
│  │    │  [sidecar: xray-daemon]        │                      │
│  │    │                                │                      │
│  │  Daemon Service: node-exporter       │ ← EC2 capacity prov │
│  └────────────────────────────────────┘                      │
│                                                               │
│  Service Connect (internal routing):                          │
│    user-service.myapp.local                                   │
│    order-service.myapp.local                                  │
│                                                               │
│  Data Layer:                                                  │
│    Aurora PostgreSQL (Multi-AZ)                               │
│    ElastiCache Redis Cluster                                  │
│    S3 (documents, media)                                      │
│    SQS (async processing)                                     │
└──────────────────────────────────────────────────────────────┘

ECR:
  account.dkr.ecr.us-east-1.amazonaws.com/
    ├── myapp/user-service (repl: ap-south-1, eu-west-1)
    ├── myapp/order-service (repl: ap-south-1, eu-west-1)
    └── myapp/payment-service (repl: ap-south-1, eu-west-1)

Security:
  - All services: awsvpc networking
  - Task-level Security Groups
  - Secrets: Secrets Manager (auto-rotation)
  - Scanning: Enhanced (AWS Inspector)

Observability:
  - Container Insights enabled
  - FireLens → CloudWatch + OpenSearch
  - X-Ray tracing (5% sampling)
  - Custom metrics → CloudWatch
  - Grafana dashboards
```

---

## 🎤 Interview Angle

**Q: "Sidecar pattern kya hai? Production mein kaise use karte hain?"**

> Sidecar = secondary container in same ECS task as main app.
> Same task network → localhost communication.
> Use cases: Log routing (Fluent Bit), observability (X-Ray, Datadog), service mesh (Envoy).
> essential: false — sidecar crash kare → main app chalti rahe.
> Pattern: App → awsfirelens → Fluent Bit sidecar → CloudWatch + S3.

**Q: "Multi-container ECS task mein dependency ordering kaise karte hain?"**

> `dependsOn` field in container definition.
> Conditions: START (container started), COMPLETE (exited any code), SUCCESS (exited 0), HEALTHY (health check passed).
> Init container pattern: DB migration container → SUCCESS → then main app starts.
> Useful for: schema migrations, data seeding, wait-for-dependency patterns.

---

*Phase 3 Complete! Next: [Phase-4_Mastery-And-Interview-Prep →](../Phase-4_Mastery-And-Interview-Prep/01_Cost_Optimization.md)*
