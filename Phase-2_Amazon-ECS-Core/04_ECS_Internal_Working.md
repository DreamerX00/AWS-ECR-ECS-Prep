# ⚙️ ECS Internal Working — From ECR to Running Container

---

## 📖 The Full Journey — ECR Image to Running ECS Task

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

## 🏗️ ECS Control Plane — How It Works

### Desired State ↔ Actual State Reconciliation

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

## 📋 Task Placement Flow (EC2 Mode)

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

## 🔄 ECS Service Deployment Flow

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
                 If health check fails 3 times → UNHEALHTY
                 If healthy → RUNNING ✅

Total cold start: 60-180s typically
Warm start (layers cached): 20-30s
```

---

## 🔒 ECS Container Agent — Security & Credentials

### How Task IAM Credentials Work:
```
EC2 Instance has an Instance Role.
But tasks should NOT inherit instance role! (over-privileged)

Solution: ECS provides per-task IAM credentials via metadata endpoint.

When task starts:
1. ECS Container Agent calls STS: AssumeRole for task's taskRoleArn
2. Temp credentials received (valid 6 hours, auto-refreshed)
3. Credentials served via HTTP:
   http://169.254.170.2/v2/credentials/<credential_id>
4. AWS SDKs inside container auto-fetch from this endpoint!
   
Container doesn't see: Access Key / Secret Key
Container doesn't see: Instance role credentials
Container ONLY gets: Task role credentials (least privilege!)
```

---

## 🎯 Analogy — Supply Chain Factory 🏭

```
ECS Control Plane = Factory management system
Container Agent   = Floor supervisor
Docker/containerd = Actual machine operators
ECR               = Parts warehouse
Task Definition   = Assembly instructions
Running Task      = Assembled product on the line
Service           = "We always need 100 units assembled"

When one unit (task) breaks:
  → Supervisor (agent) reports to management (control plane)
  → Management says: "Make a new one!"
  → New task assembled from same blueprint (task definition)
  → Quality check (health checks)
  → Send to shipping (add to ALB target group)
```

---

## 🌍 Real-World Scenario

### Production Incident: Task Keeps Failing

```
Scenario: New deployment → tasks crash loop

Timeline:
T+0:00  Service update: new image myapp:v2
T+0:30  New tasks start provisioning
T+1:00  Tasks enter RUNNING state
T+1:45  Health check: /health returns 500 error
T+2:15  Health check fails 3 times → task marked UNHEALTHY
T+2:30  ALB deregisters task
T+2:35  ECS stops task (stopTimeout=30s + SIGTERM)
T+2:35  Service: "Desired=3, Actual=2" → Launch new task
T+3:00  New task launches (same v2 image - same crash pattern!)
T+3:45  Same failure → LOOP!

Circuit Breaker kicks in (if enabled):
  After 10 consecutive failures → deployment ROLLED BACK automatically!
  Service reverts to myapp:v1 → stable state restored

Debug:
  aws ecs describe-tasks → stoppedReason field!
  aws logs get-log-events → see actual error
  aws ecs execute-command → exec into a failing task to investigate
```

---

## ⚙️ Hands-On: Debugging ECS Internals

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

## 🚨 Gotchas & Edge Cases

### 1. ECS Container Agent Polling Rate
```
Default: Agent polls ECS Control Plane every 5 seconds
Under load: Can back off to prevent overloading
Result: Task placement can take 5-10 seconds to initiate

Tune with ECS_POLL_METRICS=true and logging
Don't mistake polling delay for scheduling failure
```

### 2. Image Pull Time = Task Startup Killer
```
First pull: 800MB image → 60-90 seconds on cold host
  Fargate: ~45 seconds (dedicated bandwidth)
  EC2: Depends on instance network (up to 2 minutes on small instances)

Recommendations:
  - Keep images < 500MB (aim for < 200MB)
  - Multi-stage builds (remove build tools from runtime)
  - Alpine base images
  - ECR Pull-Through Cache for common base images
  - Pre-warm EC2 hosts with base image (base running → layers cached)
```

### 3. ECS Metadata Endpoint — v3 vs v4
```
v3: http://169.254.170.2/v3/ (older, still works)
v4: $ECS_CONTAINER_METADATA_URI_V4 (preferred, more info)

Inside container:
curl $ECS_CONTAINER_METADATA_URI_V4/task
→ Task ARN, cluster name, ECS metadata

Usage: Auto-configure logging, tracing with task ARN
```

---

## 🎤 Interview Angle

**Q: "ECS task start hote waqt image kaise pull hoti hai? Security kaise ensure hoti hai?"**

> EC2 mode mein Container Agent task definition se image URI padhta hai.
> Execution Role assume karke ECR token fetch karta hai (12-hour token).
> Image pull begins — layers parallel download ho sakte hain.
> Layer caching: Already downloaded layers skip hote hain (only changed layers download).
> Fargate mein: AWS ka managed infrastructure same process handle karta hai transparently.
> Security: Instance Role task ko nahi milta — sirf Task Role milti hai via 169.254.170.2 metadata.

**Q: "ECS deployment kaise kaam karta hai? Rolling update ke dauraan kya hota hai?"**

> Rolling update: Nayi tasks purani tasks ke parallel launch hoti hain.
> minimumHealthyPercent ensures kutch tasks hamesha serve karte rahein.
> maximumPercent controls kitni extra tasks ek saath chal sakti hain.
> Nayi task ALB health check pass karein → purani task deregisted + stop.
> Circuit breaker: Agar nayi tasks baar baar fail karein → auto-rollback to previous version.

---

*Next: [05_ECS_Scheduler_Deep_Dive.md →](./05_ECS_Scheduler_Deep_Dive.md)*
