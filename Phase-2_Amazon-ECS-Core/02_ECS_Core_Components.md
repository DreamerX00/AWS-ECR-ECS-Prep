# ECS Core Components — Complete Reference

---

## Overview

ECS has 6 core components. This document explains each one in depth using a pizza delivery company as a running analogy, followed by real-world technical detail.

---

## 1. CLUSTER — The Company

```
A logical grouping of infrastructure where your containers run.
```

```
aws ecs create-cluster --cluster-name production

ECS Cluster "production":
├── Infrastructure (Fargate + EC2)
├── IAM settings
├── Logging configuration
├── Capacity providers
└── Container Insights settings
```

**Analogy:** The pizza company "RapidPizza" is the ECS Cluster. The company is a logical boundary that contains everything: capacity, services, tasks, and configuration.

**Real-world:** Typically one cluster per environment:
- `myapp-dev`
- `myapp-staging`
- `myapp-production`

```bash
# Create cluster with Container Insights
aws ecs create-cluster \
  --cluster-name production \
  --settings name=containerInsights,value=enabled \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,base=1,weight=1 \
    capacityProvider=FARGATE_SPOT,weight=4
```

**Key consideration:** You can share a cluster across multiple services and teams, but separating clusters per environment (dev/staging/prod) provides cleaner IAM boundaries, independent scaling behavior, and avoids accidental cross-environment impact during deployments or incidents.

---

## 2. TASK DEFINITION — The Recipe

```
A JSON blueprint describing:
- Which container image to run
- How much CPU and memory
- Environment variables
- Networking mode
- IAM roles
- Logging
- Health checks
- Volumes
- Secrets
```

**Analogy:** A pizza recipe card — "Margherita Pizza: 200g dough, 150g sauce, 100g cheese, bake at 250°C for 12 min"

```json
// Complete Task Definition Example:
{
  "family": "myapp",                        // Task def family name
  "taskRoleArn": "arn:aws:iam::...:role/myapp-task-role",      // App permissions
  "executionRoleArn": "arn:aws:iam::...:role/ecsTaskExecutionRole", // ECR pull, logs
  "networkMode": "awsvpc",                  // Each task gets own ENI (required for Fargate)
  "requiresCompatibilities": ["FARGATE"],   // or EC2
  "cpu": "512",                             // 0.5 vCPU (for the whole task)
  "memory": "1024",                         // 1GB RAM

  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2.3",
      "cpu": 256,       // Container-level (subset of task CPU)
      "memory": 512,    // Hard limit (OOM kill if exceeded)
      "memoryReservation": 256,  // Soft limit (can burst to memory hard limit)
      "essential": true,         // If this crashes, ENTIRE task stops

      "portMappings": [{"containerPort": 3000, "protocol": "tcp"}],

      "environment": [
        {"name": "NODE_ENV", "value": "production"},
        {"name": "PORT", "value": "3000"}
      ],

      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:myapp/db-password"
        }
      ],

      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },

      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60   // Grace period for slow-starting apps
      }
    },

    // SIDECAR CONTAINER (optional)
    {
      "name": "log-router",
      "image": "amazon/aws-for-fluent-bit:latest",
      "essential": false,   // If this crashes, main app keeps running
      "cpu": 64,
      "memory": 128,
      ...
    }
  ],

  "volumes": [
    {
      "name": "shared-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-abc12345",
        "rootDirectory": "/data"
      }
    }
  ]
}
```

### Task Definition Versioning:
```bash
# Register new revision
aws ecs register-task-definition --cli-input-json file://task-def.json
# Creates: myapp:1 → myapp:2 → myapp:3 ...

# List revisions
aws ecs list-task-definitions --family-prefix myapp

# Describe specific revision
aws ecs describe-task-definition --task-definition myapp:3

# Deregister old revision (cleanup)
aws ecs deregister-task-definition --task-definition myapp:1
# Note: Deregistered, not deleted. Still viewable but can't launch new tasks.
```

**Important:** Task definitions are immutable — each update creates a new numbered revision. Services reference a specific revision (e.g., `myapp:5`). When you update a service to use `myapp:6`, the old `myapp:5` tasks are gracefully replaced during a rolling deployment.

---

## 3. TASK — One Running Order

```
A running (or stopped) instance of a Task Definition.
= actual pizza being made from the recipe
```

**Analogy:** A specific pizza order #4521 — made from the recipe, in a specific oven, at a specific time.

```
Task States:
PROVISIONING → PENDING → ACTIVATING → RUNNING → DEACTIVATING → STOPPING → DEPROVISIONING → STOPPED
```

```bash
# Run a one-off task (no service, just run once)
aws ecs run-task \
  --cluster production \
  --task-definition myapp:5 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-abc", "subnet-def"],
      "securityGroups": ["sg-xyz"],
      "assignPublicIp": "DISABLED"
    }
  }'

# Good for: DB migrations, batch jobs, one-time scripts

# Check task status
aws ecs describe-tasks --cluster production --tasks <task-arn>

# Stop a task
aws ecs stop-task --cluster production --task <task-arn> \
  --reason "Manual stop for maintenance"
```

### Task Lifecycle Deep Dive:
```
PROVISIONING:
  Fargate: AWS provisions compute, attaches ENI
  EC2: Container Agent finds suitable host

PENDING:
  Image pull starts (from ECR)
  Secrets fetched (Secrets Manager / SSM)

ACTIVATING:
  Container starting
  Health check startPeriod begins

RUNNING:
  Container healthy and running
  Service maintains this state

DEACTIVATING/STOPPING:
  Task stop requested
  ECS sends SIGTERM → waits stopTimeout → sends SIGKILL
  stopTimeout default: 30 seconds (configurable up to 120s)
```

**Best practice for PENDING tasks stuck at image pull:** Ensure the execution role has ECR permissions, the ECR repository policy allows the account, and the subnet has NAT Gateway or VPC endpoint access to ECR. A task stuck in PENDING for more than 5 minutes almost always indicates an image pull failure or a missing secret.

---

## 4. SERVICE — The Manager

```
Long-running manager that ensures N tasks are always running.
Handles: task deployment, health monitoring, scaling, load balancer registration.
```

**Analogy:** The floor manager at a pizza outlet — "We always need 3 cashiers. If one quits, hire another immediately. During rush hour, bring in 10."

```bash
# Create ECS Service
aws ecs create-service \
  --cluster production \
  --service-name myapp-service \
  --task-definition myapp:5 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-abc", "subnet-def"],
      "securityGroups": ["sg-myapp"],
      "assignPublicIp": "DISABLED"
    }
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/myapp-tg/abc",
    "containerName": "app",
    "containerPort": 3000
  }]' \
  --deployment-configuration '{
    "maximumPercent": 200,
    "minimumHealthyPercent": 50,
    "deploymentCircuitBreaker": {"enable": true, "rollback": true}
  }' \
  --enable-execute-command
```

### Service vs Task — Key Differences:
```
TASK:
  - Run once, collect metrics, done
  - Database migration: run to completion, exit
  - No restart on failure

SERVICE:
  - Long-lived: web server, API, worker
  - Auto-restart on failure
  - Health check integration with ALB
  - Auto-scaling support
  - Rolling deployments
```

### Deployment Circuit Breaker (Critical for Production):
When `deploymentCircuitBreaker` is enabled, ECS monitors the rolling deployment. If a configurable number of consecutive task launches fail health checks, ECS automatically rolls back the service to the last known stable task definition revision. This prevents a broken deployment from replacing all healthy tasks with unhealthy ones before anyone notices.

---

## 5. CONTAINER — The Worker

```
The actual running process inside a task.
One task can have multiple containers (multi-container task).
```

**Analogy:** An individual worker performing a specific job inside the pizza outlet.

### Essential vs Non-Essential Containers:
```json
// Essential container:
{"name": "app", "essential": true}
// App crashes → ENTIRE TASK STOPS → Service replaces task

// Non-essential container (sidecar):
{"name": "metrics-exporter", "essential": false}
// Metrics crashes → Task keeps running (app still serves traffic!)
```

### Multi-Container Task:
```json
{
  "containerDefinitions": [
    {
      "name": "nginx",           // Reverse proxy
      "essential": true,
      "image": "nginx:latest",
      "portMappings": [{"containerPort": 80}]
    },
    {
      "name": "app",             // Backend app
      "essential": true,
      "image": "myapp:v1",
      // localhost:3000 accessible from nginx (shared network in awsvpc mode)
    },
    {
      "name": "fluentbit",       // Log router
      "essential": false,
      "image": "amazon/aws-for-fluent-bit:latest"
    }
  ]
}
```

### Container Dependencies:
For multi-container tasks where one container must start before another, use `dependsOn`:

```json
{
  "name": "app",
  "dependsOn": [
    {
      "containerName": "init-db",
      "condition": "SUCCESS"   // Wait for init-db to exit 0 before starting app
    }
  ]
}
```

This is how init containers (database migrations, config setup) are implemented in ECS without Kubernetes's dedicated init container concept.

---

## 6. CAPACITY PROVIDER — The Venue

```
Defines the infrastructure (EC2 ASG or Fargate/Fargate Spot)
where tasks get scheduled.
```

**Analogy:** Decides which kitchen — cloud kitchen, physical outlet, or ghost kitchen — the pizza order goes to.

### Types:
```
FARGATE          → AWS managed serverless (recommended for most workloads)
FARGATE_SPOT     → 70% cheaper, can be interrupted (good for batch processing)
EC2 ASG          → You manage EC2 instances, linked via Auto Scaling Group
```

```bash
# Associate EC2 ASG as capacity provider
aws ecs create-capacity-provider \
  --name my-ec2-capacity-provider \
  --auto-scaling-group-provider '{
    "autoScalingGroupArn": "arn:aws:autoscaling:...:autoScalingGroup:...:my-asg",
    "managedScaling": {
      "status": "ENABLED",
      "targetCapacity": 80,     // Keep ASG at 80% capacity utilization
      "minimumScalingStepSize": 1,
      "maximumScalingStepSize": 10
    },
    "managedTerminationProtection": "ENABLED"  // Don't terminate EC2 with running tasks!
  }'
```

**Managed scaling explained:** When `managedScaling` is enabled on a capacity provider, ECS calculates the number of EC2 instances required to satisfy the number of pending tasks. It then sends a scale-out or scale-in signal to the Auto Scaling Group automatically, so you do not need to manually tune ASG scaling policies. The `targetCapacity` of 80% leaves headroom for burst without triggering immediate scale-out.

---

## Putting It All Together — Component Flow

```
                    ┌─────────────────────────────┐
                    │      ECS CLUSTER             │
                    │     "production"             │
                    │                             │
                    │  ┌──────────────────────────┐│
                    │  │  SERVICE "myapp-service"  ││ ← Manages desired state
                    │  │  Desired count: 3         ││
                    │  │                          ││
                    │  │  ┌────┐ ┌────┐ ┌────┐   ││
                    │  │  │Task│ │Task│ │Task│   ││ ← 3 running tasks
                    │  │  └──┬─┘ └──┬─┘ └──┬─┘   ││
                    │  │     │      │      │     ││
                    │  │  ┌──▼──────▼──────▼──┐  ││
                    │  │  │   Task Definition  │  ││ ← Blueprint
                    │  │  │   "myapp:5"        │  ││
                    │  │  └───────────────────-┘  ││
                    │  └──────────────────────────┘│
                    │                             │
                    │  CAPACITY PROVIDERS:         │
                    │  ├── FARGATE (weight=1)      │ ← Where tasks run
                    │  └── FARGATE_SPOT (weight=4) │
                    └─────────────────────────────┘
```

---

## Gotchas & Edge Cases

### 1. Task Stop — SIGTERM Then SIGKILL
```
When a task is stopped:
→ ECS sends SIGTERM to container
→ App has 30 seconds (stopTimeout) to gracefully shut down
→ If not stopped → SIGKILL (forced kill)

If your app ignores SIGTERM (many Node.js apps do!):
→ In-flight requests are dropped, DB connections are not closed cleanly
→ Fix: Handle SIGTERM in your app!

Node.js:
process.on('SIGTERM', () => {
  server.close(() => {
    db.disconnect();
    process.exit(0);
  });
});
```

### 2. Task Definition CPU/Memory — Fargate Valid Combinations Only!
```
Fargate has valid CPU/memory combinations:
CPU: 256 (.25 vCPU): Memory 512, 1024, 2048
CPU: 512 (.5 vCPU):  Memory 1024-4096 (1GB increments)
CPU: 1024 (1 vCPU):  Memory 2048-8192 (1GB increments)
CPU: 2048 (2 vCPU):  Memory 4096-16384 (1GB increments)
CPU: 4096 (4 vCPU):  Memory 8192-30720 (1GB increments)

Invalid combinations fail at task launch!
```

### 3. Service Desired Count = 0 Does Not Delete the Service
```bash
# Stop all tasks without deleting service:
aws ecs update-service --cluster prod --service myapp --desired-count 0

# Service remains; no tasks run; no Fargate cost is incurred
# Scale back up when ready: update-service --desired-count 3
# Useful for maintenance windows and cost reduction in non-prod environments!
```

### 4. Task Definition Secrets vs Environment Variables
```
Environment variables in task definitions are stored in PLAINTEXT in the ECS console
and API responses. Never put passwords, API keys, or certificates as environment variables.

Use "secrets" field instead:
  - Values are fetched from Secrets Manager or SSM Parameter Store at task launch
  - The actual secret value is never stored in the task definition
  - Requires the execution role to have secretsmanager:GetSecretValue permission
  - Supports automatic secret rotation without redeploying the task definition
```

---

## Interview Questions

**Q: "What is the difference between a Task and a Service? When do you use each?"**

> A Task is a temporary running instance of a task definition. It is used for database migrations, batch jobs, and one-time scripts — it runs to completion and stops.
> A Service is a long-running manager that ensures a desired number of tasks are always running. It is used for web servers, APIs, and background workers.
> A Service handles auto-restart on failure, rolling deployments, ALB registration, and auto-scaling. A Task does none of these automatically.
> For production services, always use a Service. Use standalone Tasks only for jobs with a defined completion point.

**Q: "What does the essential flag do on an ECS container definition?"**

> When an essential container exits (for any reason, including a crash), ECS stops the entire task.
> The ECS Service then replaces the stopped task with a new one to maintain the desired count.
> Non-essential containers, such as log routers and monitoring sidecars, should have `essential: false`.
> This way, a sidecar crash does not cause the entire task to restart and interrupt application traffic.
> A common production mistake is marking all containers as essential, causing outages when a sidecar has a transient failure.

---

*Next: [03_ECS_Launch_Types.md →](./03_ECS_Launch_Types.md)*
