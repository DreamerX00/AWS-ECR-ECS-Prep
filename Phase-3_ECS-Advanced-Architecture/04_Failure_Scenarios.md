# 🛑 Failure Scenarios — What Happens When Things Go Wrong

---

## 📖 Why This Matters

A senior engineer's value = knowing what FAILS and designing around it. ECS failures come in predictable patterns.

---

## 💥 Scenario 1: Container Crash

### What Happens:
```
Container process exits (any exit code ≠ 0)

If essential=true:
  → ENTIRE TASK stops
  → ECS Service detects: running < desired
  → NEW task launched
  
If essential=false:
  → Only that container stops
  → Other containers continue
  → Task stays RUNNING (if healthy containers remain)
```

### Exit Code Meanings:
```
Exit Code 0   → Graceful shutdown (expected, normal)
Exit Code 1   → Generic error (app bug)
Exit Code 127 → Command not found in container
Exit Code 137 → SIGKILL (OOM killed! Memory limit exceeded)
Exit Code 139 → Segmentation fault
Exit Code 143 → SIGTERM received, graceful shutdown

# Check exit code:
aws ecs describe-tasks --cluster production --tasks <arn> \
  --query 'tasks[0].containers[0].exitCode'
```

### ECS Restart Policy (EC2 mode, Standalone Tasks):
```json
{
  "restartPolicy": {
    "enabled": true,
    "ignoredExitCodes": [0],    // Don't restart on graceful exit
    "restartAttemptPeriod": 300  // 5 min window between restart attempts
  }
}
// Note: Services auto-replace tasks — this is for standalone tasks
```

---

## 💻 Scenario 2: Instance Failure (EC2 Mode)

### What Happens:
```
EC2 instance dies → Container Agent disconnects

ECS Control Plane:
  - Agent missing → instance marked as "inactive" after ~3 minutes
  - All tasks on that instance marked STOPPED (stopCode: TaskSetNotFound or similar)
  - Service Controller: running < desired → launch new tasks on remaining instances!
  - New tasks placed on healthy instances
  
Time to recovery: 3-5 minutes typically
```

### Capacity Rebalance for Spot Interruptions:
```
EC2 Spot instance gets 2-minute warning before termination.

With Capacity Rebalance enabled:
  - At 2-min warning → AWS starts NEW instance proactively!
  - New tasks warm up BEFORE old spot terminated
  - Zero-downtime Spot interruption handling!
  
Enable:
aws autoscaling put-warm-pool \
  --auto-scaling-group-name myapp-asg \
  --pool-state Running  ← Pre-warmed instances ready!

--capacity-rebalance  ← On the ASG
```

### Task Rebalancing After Instance Loss:
```
Before failure:
  AZ-a: [Task1, Task2, Task3]
  AZ-b: [Task4, Task5, Task6]
  
AZ-a instance fails → Task1, Task2, Task3 gone

Service has spread strategy:
  Needs to restart 3 tasks
  AZ-a has 1 remaining instance, AZ-b has 2
  
  AZ-a: [Task7-new]
  AZ-b: [Task4, Task5, Task6, Task8-new, Task9-new]
  
  Imbalanced! But ECS won't rebalance without task forced replacement.
  
  Manual rebalance:
  aws ecs update-service --cluster production --service myapp --force-new-deployment
  # Force new deployment → replaces all tasks with spread strategy
```

---

## 🌍 Scenario 3: AZ Failure

### AWS Approach:
```
Netflix chaos engineering: "What if us-east-1a goes down?"

Setup:
  Desired=6, Spread across 3 AZs
  AZ-a: [Task1, Task2]
  AZ-b: [Task3, Task4]
  AZ-c: [Task5, Task6]

AZ-a failure:
  Task1, Task2 → STOPPED
  
  ECS: Running=4, Desired=6 → launch 2 more tasks
  
  With spread strategy on AZ only:
    AZ-b and AZ-c remain → tasks placed in AZ-b and AZ-c
    [Task3, Task4, Task7-new] + [Task5, Task6, Task8-new]
    
  ALB: Detects AZ-a unhealthy → routes ALL traffic to AZ-b and AZ-c
  
  SERVICE CONTINUES! Just with more load on remaining AZs
  (Monitor: CPU/Memory in remaining AZs!)
```

### Multi-AZ Design Principles:
```
ALWAYS spread across minimum 2 AZs (3 preferred)
  
Minimum instances: 2 (one losing one still surviving)
Minimum tasks: 3 (losing one AZ, still 2 remain for redundancy)

For critical services:
  Desired=6 across 3 AZs = 2 per AZ
  AZ failure: 4 remaining = 66% capacity
  Design max load = 50% capacity → 66% capacity sufficient!
```

---

## 🏥 Scenario 4: Health Check Flapping

### Problem:
```
Task passes health check → added to ALB
Task fails health check → removed from ALB rotation
Task passes again → added back

Users see intermittent failures! Classic "health check storm"
```

### Causes and Fixes:
```
Cause 1: Health check endpoint too heavy
  /health does DB query → sometimes slow > timeout
  Fix: /health should be lightweight (memory check, uptime only)
       DB check → /readiness (separate endpoint)

Cause 2: startPeriod too short
  App takes 45 seconds to boot → ECS health check starts at 30s → fails!
  Fix: startPeriod=120 (give app time to fully boot)

Cause 3: Memory pressure causes GC pauses
  App at 95% memory → GC pause → health check times out → flagged unhealthy!
  Fix: Increase memory limit OR reduce startupThreshold
  
Cause 4: Outbound dependency failing
  /health checks external DB → DB slow → health check timeout → task killed!
  Fix: Health check should be INTERNAL only (can I serve requests?)
       Dependency issues = separate alerting
```

### Properly Designed Health Check:
```javascript
// GOOD: Lightweight health endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    uptime: process.uptime(),
    memory: process.memoryUsage()
  });
});

// SEPARATE: Deep/readiness check (not used for ECS health check!)
app.get('/ready', async (req, res) => {
  try {
    await db.ping();
    await redis.ping();
    res.json({ ready: true });
  } catch(err) {
    res.status(503).json({ ready: false, error: err.message });
  }
});
```

---

## 📦 Scenario 5: Image Pull Failure

### Causes:
```
1. ECR auth token expired (12 hours)
   Error: "no basic auth credentials"
   Fix: ECS handles auto-refresh in tasks — likely Execution Role issue
   
2. Image doesn't exist
   Error: "image not found" or "manifest for tag not found"
   Fix: Check task definition image tag — is it pushed to ECR?
   
3. Execution Role missing ECR permissions
   Error: "access denied" during pull
   Fix: Add ecr:BatchGetImage, ecr:GetDownloadUrlForLayer to Execution Role
   
4. VPC endpoint not configured (private subnet)
   Error: "connection timeout" during pull
   Fix: Create ECR VPC endpoint (dkr + api) + S3 gateway endpoint
   
5. Image too large → timeout
   Error: Pull timeout
   Fix: Reduce image size OR configure ECS imagePullTimeout
```

### Debug Image Pull Failures:
```bash
# Check stopped reason
aws ecs describe-tasks --cluster production --tasks <arn> \
  --query 'tasks[0].stoppedReason'
# "CannotPullContainerError: Pull image manifest from ECR has failed..."

# Check container agent logs (EC2 mode)
ssh ec2-host
sudo tail -f /var/log/ecs/ecs-agent.log | grep -i "pull"
```

---

## 🧠 Scenario 6: Rolling Update Stuck

### Problem:
```
Deployment initiated. Stuck at 50%. Won't complete.
aws ecs describe-services → deployments shows 2 deployments (PRIMARY + ACTIVE)
```

### Common Causes:
```
1. New tasks keep failing health check
   → Old tasks not replaced (to maintain minimum healthy %)
   → Deployment stuck!
   → Circuit breaker should catch this if enabled
   
2. Not enough capacity
   → maximumPercent=150, EC2 instances full
   → Can't launch extra tasks!
   → Add capacity OR reduce container size requirements
   
3. ALB health check too strict
   → New tasks healthy by ECS health check
   → But ALB health check failing (different endpoint, wrong port)
   → ECS won't drain old tasks (thinks new ones not ready)
   
4. Stuck deregistration
   → Old tasks draining connections (deregistrationDelay=300s)
   → Appears stuck for 5 minutes
   → Reduce deregistrationDelay for HTTP APIs
```

### Debug Stuck Deployment:
```bash
# Check deployment events
aws ecs describe-services --cluster production --services myapp \
  --query 'services[0].events'
# Look for "steady state" or "unable to place" messages

# Check if new tasks are passing health checks
aws ecs list-tasks --cluster production --service-name myapp

# Check individual task health
aws ecs describe-tasks --cluster production --tasks <new-task-arn> \
  --query 'tasks[0].healthStatus'

# Force rollback (if circuit breaker not enabled):
aws ecs update-service \
  --cluster production \
  --service myapp \
  --task-definition myapp:PREVIOUS_REVISION
```

---

## 📋 Failure Response Runbook

```
Incident: ECS service degraded

1. Check RunningTaskCount vs DesiredCount (CloudWatch alarm?)
   aws ecs describe-services --cluster prod --services myapp | jq '.services[0].runningCount, .services[0].desiredCount'

2. Check recent events
   aws ecs describe-services --cluster prod --services myapp | jq '.services[0].events[:5]'

3. Find stopped tasks (last 1 hour)  
   aws ecs list-tasks --cluster prod --service-name myapp --desired-status STOPPED | \
     xargs aws ecs describe-tasks --cluster prod --tasks | \
     jq '.tasks[] | {stoppedAt:.stoppedAt, reason:.stoppedReason}'

4. Check logs for the service
   aws logs tail /ecs/myapp --follow

5. Check recent deployment
   aws ecs describe-services | jq '.services[0].deployments'

6. Rollback if needed
   aws ecs update-service --cluster prod --service myapp \
     --task-definition myapp:PREVIOUS_WORKING_VERSION
```

---

## 🎤 Interview Angle

**Q: "ECS task crash hone par kya hota hai? Service kaise recover karta hai?"**

> Essential container exit → entire task stops.
> ECS Service Controller detects: runningCount < desiredCount.
> New task launched based on task definition.
> If tasks consistently crash → Circuit Breaker triggers (if enabled) → deployment stopped/rolled back.
> Graceful shutdown: SIGTERM sent first → stopTimeout (default 30s) → SIGKILL if not stopped.
> Design: Handle SIGTERM in app code, close connections gracefully.

**Q: "AZ failure pe ECS kaise handle karta hai?"**

> AZ failure → tasks on instances in that AZ become unreachable → marked STOPPED.
> ECS Service: Desired count not met → new tasks placed on remaining AZs.
> ALB automatically routes traffic only to healthy AZs (AZ-based health checking).
> Design for AZ failure: Minimum 3 AZs, minimum 1 task per AZ guaranteed, max load = 50% capacity so 66% remaining AZs can handle it.

---

*Next: [06_Scaling_Strategies.md →](./06_Scaling_Strategies.md)*
