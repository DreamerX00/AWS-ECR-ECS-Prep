# Advanced Architecture Patterns

---

## 1. Sidecar Pattern

### What is it?
```
Attach a "helper" container to your main app container.
Same task → same network → work together.

Main container: Does the core business logic
Sidecar:        Handles cross-cutting concerns (logging, metrics, TLS, etc.)
```

The sidecar pattern keeps your main application container focused on business logic, while infrastructure concerns (logging, observability, security) are handled by dedicated sidecar containers. This separation makes it easy to swap out infrastructure tooling without touching application code, and allows platform teams to standardize sidecars across all services.

### Pattern A: Log Router Sidecar
```json
{
  "containerDefinitions": [
    {
      "name": "app",
      "image": "myapp:v1",
      "logConfiguration": {
        "logDriver": "awsfirelens"   // Sends logs to sidecar!
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

### Key Rule: Sidecar Should Be `essential: false`

If the sidecar is marked `essential: true`, a sidecar crash will bring down the entire task — including your main application. Mark sidecars as `essential: false` so that the main application continues running even if a logging or monitoring sidecar fails. The exception is when the sidecar handles something truly critical (e.g., a security proxy that all traffic must pass through).

---

## 2. Daemon Service Pattern

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
  --scheduling-strategy DAEMON   # Magic!
  # No desiredCount needed (auto = number of instances)

# Task definition: Give daemon access to host paths
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
      "containerPath": "/var/log/host"   // Access host /var/log
    }]
  }]
}
```

Daemon services are one of the primary reasons to run EC2-based ECS clusters instead of Fargate. Fargate does not support the DAEMON scheduling strategy because Fargate abstracts away the host EC2 instances. If you need to run exactly one agent per host (log forwarders, security scanners, disk backup agents), you need EC2 mode.

---

## 3. Multi-Container Task Pattern

```
Multiple containers in ONE task for tightly coupled functionality.

When to use:
  - Containers share localhost network (communicate via 127.0.0.1)
  - Containers share volumes (one writes, one reads)
  - Containers need to start in order (init container)
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
      "essential": false,           // Not essential — will exit when done
      "dependsOn": []
    },
    {
      "name": "app",
      "image": "myapp:v1",
      "essential": true,
      "dependsOn": [{
        "containerName": "migrate",
        "condition": "SUCCESS"    // Wait for migrations to succeed!
      }]
    }
  ]
}
```

The `dependsOn` conditions are:
- `START`: Wait for the container to start (PID 1 running)
- `COMPLETE`: Wait for the container to exit (any exit code)
- `SUCCESS`: Wait for the container to exit with code 0
- `HEALTHY`: Wait for the container to pass its health check

The `SUCCESS` condition is the most useful for init containers — it ensures the migration script completed successfully before starting the main application. If the migration fails, the main app never starts, preventing the application from running against an incompatible schema.

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
// awsvpc: both containers share same network namespace → can use localhost
```

---

## 4. Event-Driven ECS Pattern

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
            "Value": "$.body.videoId"   // From SQS message!
          }]
        }]
      }
    }
  }' \
  --role-arn arn:aws:iam::123:role/pipes-execution-role
```

The event-driven pattern is powerful for workloads where task count should directly correspond to the number of pending jobs. Each SQS message triggers exactly one ECS task. The task processes the job and exits. This is a clean model for batch processing — no need to manage worker pools, no idle workers consuming resources between jobs.

A key operational consideration: ensure your tasks handle SQS visibility timeouts correctly. If a task takes longer than the visibility timeout to process a job, SQS will make the message visible again and potentially trigger a duplicate task.

---

## 5. Blue/Green Deployment Pattern with CodeDeploy (Architecture View)

```
ALB Production Listener (443)
         │
         ├── [Blue] Target Group → Blue Tasks (current production)
         └── [Green] Target Group → Green Tasks (new version)

Deployment Flow:
  Step 1: Launch green tasks (0% production traffic)
  Step 2: Test listener (8080) → 100% traffic to green
  Step 3: Run automated tests via test listener
  Step 4: If tests pass → Production listener switch (5% canary first, then 100%)
  Step 5: Blue tasks: keep for 30 minutes (rollback window)
  Step 6: Terminate blue tasks

Rollback:
  One API call → switch listener back to blue
  < 1 second switchback!
```

---

## Real-World: Complete Production Architecture

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

## Interview Angle

**Q: "What is the sidecar pattern? How is it used in production?"**

> The sidecar pattern involves running a secondary container in the same ECS task as the main application. Since they share the same task network (awsvpc mode), they can communicate via localhost. The sidecar handles cross-cutting concerns so the main application stays focused on business logic.
>
> Common production uses: Log routing with Fluent Bit (receives logs from app via awsfirelens driver, routes to CloudWatch, S3, and OpenSearch simultaneously), observability with X-Ray daemon (receives trace data from app, batches and forwards to X-Ray service), and service mesh proxying with Envoy (intercepts all network traffic for retries, circuit breaking, mTLS).
>
> The sidecar should be marked `essential: false` so that a sidecar crash does not kill the main application task.

**Q: "How do you control container startup order in a multi-container ECS task?"**

> The `dependsOn` field in the container definition controls startup ordering. It supports four conditions: `START` (container has started), `COMPLETE` (container has exited, any code), `SUCCESS` (container exited with code 0), and `HEALTHY` (container passed its health check).
>
> The most common use case is the init container pattern: a migration container runs DB schema migrations first. The main application container has `dependsOn: [{containerName: "migrate", condition: "SUCCESS"}]`. If the migration fails (non-zero exit), the main app never starts. This prevents application bugs where code runs against an incompatible database schema.

---

*Phase 3 Complete! Next: [Phase-4_Mastery-And-Interview-Prep →](../Phase-4_Mastery-And-Interview-Prep/01_Cost_Optimization.md)*
