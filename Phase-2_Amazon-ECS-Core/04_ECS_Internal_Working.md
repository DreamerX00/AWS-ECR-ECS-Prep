# ECS Internal Working — From ECR to Running Container

---

## The Full Journey — ECR Image to Running ECS Task

```
ECR Image
   │
   ▼ Pull image URI from task definition
Task Definition (myapp:5)
   │
   ▼ Submitted to ECS Scheduler
Scheduler decides placement
   │
   ▼ (Fargate) → AWS provisions MicroVM
   │  (EC2)    → Container Agent on existing EC2 receives task
ECS Container Agent
   │
   ▼ Pull image from ECR (if not cached)
Docker Runtime / containerd
   │
   ▼ Create container (namespaces, cgroups)
Running Container
   │
   ▼ Health checks begin
ALB Target Group (if configured)
   │
   ▼ Traffic flows
Active, healthy container serving traffic!
```

---

## ECS Control Plane — How It Works

### Desired State vs Actual State Reconciliation

```
Desired State: Service wants 3 tasks
Actual State:  2 tasks running

                ECS Service Controller
                    continuously polls
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
        Task 1          Task 2      (MISSING!)
        RUNNING         RUNNING

Controller detects: Need 1 more task!
→ Calls ECS Scheduler to place new task
→ New task starts
→ Actual State = Desired State ✅
```

This reconciliation loop is the heart of ECS. It runs continuously. Whether a task crashes, an EC2 instance is terminated, or a health check fails, the controller detects the divergence between desired and actual state and takes corrective action within seconds.

### Container Agent Polling Loop (EC2 mode):
```
ECS Container Agent (on EC2 host)
    │
    ├── Every N seconds: Poll ECS Control Plane
    │   "Any tasks for me? Resource: 4 vCPU, 16GB remaining"
    │
    ├── Control Plane response: "Yes! Run task myapp:5 (0.5 vCPU, 1GB)"
    │
    ├── Agent instructs Docker:
    │   docker pull <ecr-image>
    │   docker create <container-options>
    │   docker start <container-id>
    │
    ├── Agent monitors container:
    │   Health check status
    │   CPU/Memory metrics
    │   Exit codes
    │
    └── Agent reports back to Control Plane:
        "Task myapp:5 is RUNNING on 172.31.5.10:3000"
```

---

## Task Placement Flow (EC2 Mode)

```
New task request comes to ECS Scheduler:
  Task needs: 256 CPU, 512MB Memory

Step 1: CLUSTER FILTERING
  Filter out disconnected/draining instances → ELIGIBLE HOSTS

Step 2: CONSTRAINT FILTERING
  Apply placement constraints:
  - "memberOf: attribute:ecs.instance-type == c5.2xlarge"
  - "distinctInstance: true"
  → Further filter eligible hosts

Step 3: STRATEGY APPLICATION
  Apply placement strategy to REMAINING hosts:
  - binpack: Pick host with LEAST available resources (pack dense)
  - spread: Pick host in different AZ/instance
  - random: Pick randomly

Step 4: PLACE TASK
  Task placed on selected host
  Container Agent receives instruction
```

---

## ECS Service Deployment Flow

### Rolling Update (Default):
```
Service: myapp, desired=10, new image: myapp:v2

minimumHealthyPercent=50, maximumPercent=200

Phase 1:
  Running: v1 × 10
  Action: Launch v2 tasks (up to 200% = 20 total allowed)

Phase 2:
  Running: v1 × 10 + v2 × 10 (= 20 total)

Phase 3:
  Wait for v2 tasks to pass health checks (ALB health check)

Phase 4:
  Deregister v1 tasks from ALB
  Stop v1 tasks

Phase 5:
  Running: v2 × 10 ✅ Deployment complete!

Time: Depends on health check period + new task startup
```

### Task Startup Sequence:
```
1. PROVISIONING  (Fargate: 10-15s for VM boot)
2. PENDING       (Image pull: 10-120s depending on cache)
3. ACTIVATING    (Container start: 1-5s)
                 ↓
4. Health check: startPeriod = 60s (grace period)
                 interval = 30s
                 If health check fails 3 times → UNHEALTHY
                 If healthy → RUNNING ✅

Total cold start: 60-180s typically
Warm start (layers cached): 20-30s
```

### Deployment Configuration Options:
```
minimumHealthyPercent=100, maximumPercent=200:
  → Zero-downtime deploy
  → Doubles capacity briefly (expensive for large fleets)

minimumHealthyPercent=50, maximumPercent=150:
  → Brief capacity reduction (acceptable for most services)
  → Less costly during deployment

minimumHealthyPercent=0, maximumPercent=100:
  → Stop all old tasks first, then start new tasks
  → Causes downtime! Only appropriate for non-production environments
```

---

## ECS Container Agent — Security & Credentials

### How Task IAM Credentials Work:
```
EC2 Instance has an Instance Role.
But tasks should NOT inherit the instance role! (over-privileged)

Solution: ECS provides per-task IAM credentials via metadata endpoint.

When task starts:
1. ECS Container Agent calls STS: AssumeRole for task's taskRoleArn
2. Temp credentials received (valid 6 hours, auto-refreshed)
3. Credentials served via HTTP:
   http://169.254.170.2/v2/credentials/<credential_id>
4. AWS SDKs inside container auto-fetch from this endpoint!

Container does NOT see: Access Key / Secret Key stored anywhere
Container does NOT see: Instance role credentials
Container ONLY gets: Task role credentials (least privilege!)
```

This design ensures that even if a container is compromised, the attacker only gains access to the specific permissions granted to that task's role — not the broader permissions of the underlying EC2 instance.

---

## Analogy — Supply Chain Factory

```
ECS Control Plane = Factory management system
Container Agent   = Floor supervisor
Docker/containerd = Machine operators
ECR               = Parts warehouse
Task Definition   = Assembly instructions
Running Task      = Assembled product on the line
Service           = "We always need 100 units assembled"

When one unit (task) breaks:
  → Supervisor (agent) reports to management (control plane)
  → Management says: "Assemble a new one!"
  → New task assembled from same blueprint (task definition)
  → Quality check (health checks)
  → Send to shipping (add to ALB target group)
```

---

## Real-World Scenario

### Production Incident: Task Crash Loop

```
Scenario: New deployment → tasks enter a crash loop

Timeline:
T+0:00  Service update: new image myapp:v2
T+0:30  New tasks start provisioning
T+1:00  Tasks enter RUNNING state
T+1:45  Health check: /health returns 500 error
T+2:15  Health check fails 3 times → task marked UNHEALTHY
T+2:30  ALB deregisters task
T+2:35  ECS stops task (stopTimeout=30s + SIGTERM)
T+2:35  Service: "Desired=3, Actual=2" → Launch new task
T+3:00  New task launches (same v2 image — same crash pattern!)
T+3:45  Same failure → LOOP!

Circuit Breaker kicks in (if enabled):
  After 10 consecutive failures → deployment ROLLED BACK automatically!
  Service reverts to myapp:v1 → stable state restored

Debug steps:
  aws ecs describe-tasks → check stoppedReason field!
  aws logs get-log-events → see actual application error
  aws ecs execute-command → exec into a failing task to investigate live
```

Without a circuit breaker, this loop continues indefinitely — every crashed task is replaced with the same broken image. Enabling `deploymentCircuitBreaker` with `rollback: true` is essential for production services.

---

## Hands-On: Debugging ECS Internals

```bash
# Step 1: Find why task stopped
aws ecs describe-tasks \
  --cluster production \
  --tasks <task-arn> \
  --query 'tasks[0].{
    Status:lastStatus,
    StoppedReason:stoppedReason,
    StopCode:stopCode,
    Containers:containers[*].{
      Name:name,
      Status:lastStatus,
      Reason:reason,
      ExitCode:exitCode
    }
  }'

# Common stopCode values:
# EssentialContainerExited → essential container exited (check exit code)
# TaskFailedToStart → image pull failed, secrets fetch failed
# UserInitiated → someone called stop-task manually

# Step 2: Check CloudWatch Logs
aws logs get-log-events \
  --log-group-name /ecs/myapp \
  --log-stream-name ecs/app/<task-id>

# Step 3: ECS Exec (like kubectl exec)
aws ecs execute-command \
  --cluster production \
  --task <task-arn> \
  --container app \
  --interactive \
  --command "/bin/sh"

# Step 4: Check agent health on EC2
ssh ec2-host
sudo systemctl status ecs
sudo cat /var/log/ecs/ecs-agent.log | tail -100

# Step 5: Check image pull issues
sudo cat /var/log/ecs/ecs-agent.log | grep "pull"
```

---

## Gotchas & Edge Cases

### 1. ECS Container Agent Polling Rate
```
Default: Agent polls ECS Control Plane every 5 seconds
Under load: Can back off to prevent overwhelming the control plane
Result: Task placement can take 5-10 seconds to initiate after a crash is detected

This polling delay is normal — do not mistake it for a scheduling failure.
Monitor via CloudWatch Container Insights to distinguish polling delay from
genuine scheduling issues (capacity exhaustion, constraint violations).
```

### 2. Image Pull Time — The Biggest Startup Killer
```
First pull of a large image: 800MB image → 60-90 seconds on a cold host
  Fargate: ~45 seconds (dedicated bandwidth allocation)
  EC2: Depends on instance network bandwidth (up to 2 minutes on small instances)

Recommendations:
  - Keep images < 500MB (aim for < 200MB)
  - Multi-stage builds (remove build tools from final runtime image)
  - Alpine or distroless base images
  - ECR Pull-Through Cache for common base images
  - Pre-warm EC2 hosts by running a task that pulls the base image
    (subsequent launches only pull the changed layers — often under 1MB)
```

### 3. ECS Metadata Endpoint — v3 vs v4
```
v3: http://169.254.170.2/v3/ (older, still supported)
v4: $ECS_CONTAINER_METADATA_URI_V4 (preferred, more information)

Inside container:
curl $ECS_CONTAINER_METADATA_URI_V4/task
→ Returns: Task ARN, cluster name, task definition, container details

Usage:
  - Auto-configure structured logging with task ARN as a field
  - Set distributed tracing resource attributes automatically
  - Expose service identity to health check endpoints
```

### 4. Stopped Tasks Are Retained for Debugging
```
ECS retains stopped tasks in the console and API for 1 hour after they stop.
After 1 hour, they are garbage collected.

If you are debugging a crash:
  - Act quickly — the stopped task record with stoppedReason disappears after 1 hour
  - Enable CloudWatch Logs on all containers to preserve output beyond this window
  - Consider enabling Container Insights for persistent metric retention
```

---

## Interview Questions

**Q: "How does ECS pull an image at task start time? How is security maintained?"**

> In EC2 mode, the Container Agent reads the image URI from the task definition.
> It assumes the Execution Role to obtain a 12-hour ECR authorization token.
> Image pull begins — layers are downloaded in parallel where possible.
> Layer caching means only changed layers are downloaded on subsequent pulls; unchanged layers are served from the local cache.
> In Fargate mode, AWS's managed infrastructure handles the same process transparently.
> Security is maintained because the container only receives the Task Role credentials via the `169.254.170.2` metadata endpoint — not the Instance Role or the Execution Role credentials.

**Q: "How does a rolling deployment work in ECS? What happens during the process?"**

> A rolling update launches new tasks using the updated task definition in parallel with the existing tasks.
> `minimumHealthyPercent` ensures that a minimum number of tasks remain in service at all times, so traffic is never completely interrupted.
> `maximumPercent` controls how many total tasks can run simultaneously, which limits the cost of the double-running period.
> Each new task must pass ALB health checks before the old task is deregistered and stopped.
> If the deployment circuit breaker is enabled and a threshold of consecutive health check failures is reached, ECS automatically rolls back to the previous task definition revision — preventing a broken deployment from fully replacing a healthy fleet.

---

*Next: [05_ECS_Scheduler_Deep_Dive.md →](./05_ECS_Scheduler_Deep_Dive.md)*
