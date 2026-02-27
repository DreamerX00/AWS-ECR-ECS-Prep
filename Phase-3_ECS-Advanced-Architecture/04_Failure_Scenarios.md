# Failure Scenarios — What Happens When Things Go Wrong

---

## Why This Matters

A senior engineer's value lies in knowing what fails and designing systems around those failure modes. ECS failures come in predictable patterns. Understanding each scenario — what triggers it, what ECS does automatically, and what you must design for — is what separates production-ready architectures from fragile ones.

---

## Scenario 1: Container Crash

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

Exit code 137 (OOM kill) is one of the most common production issues with containerized workloads. When a container exceeds its hard memory limit, the Linux kernel's OOM killer sends SIGKILL. There is no warning and no graceful shutdown. The application cannot catch or handle SIGKILL. Always set memory limits with a buffer above the expected peak usage, and monitor memory utilization trends for signs of memory leaks.

### Graceful Shutdown — Handling SIGTERM:
```
ECS shutdown sequence:
  1. SIGTERM sent to container's PID 1
  2. stopTimeout seconds grace period (default 30s)
  3. If still running → SIGKILL (exit code 137)

Your app MUST handle SIGTERM:
```

```javascript
// Node.js graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully...');

  // Stop accepting new requests
  server.close(async () => {
    // Finish in-flight requests
    await db.close();          // Close DB connections
    await consumer.stop();     // Stop SQS/Kafka consumers
    console.log('Shutdown complete');
    process.exit(0);
  });

  // Force exit after 25 seconds (before ECS sends SIGKILL at 30s)
  setTimeout(() => process.exit(1), 25000);
});
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

## Scenario 2: Instance Failure (EC2 Mode)

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
  --pool-state Running  # Pre-warmed instances ready!

--capacity-rebalance  # On the ASG
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

## Scenario 3: AZ Failure

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

Minimum instances: 2 (one lost, one still surviving)
Minimum tasks: 3 (losing one AZ still leaves 2 for redundancy)

For critical services:
  Desired=6 across 3 AZs = 2 per AZ
  AZ failure: 4 remaining = 66% capacity
  Design max load = 50% capacity → 66% remaining capacity is sufficient!
```

The capacity math is important: if you design your service to handle 100% load at 50% of total task capacity, then losing one of three AZs still leaves you with 66% capacity — enough to handle the full load. If you design it to handle 100% load at 80% capacity, you will experience degradation or outages when an AZ fails.

---

## Scenario 4: Health Check Flapping

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
  Fix: Increase memory limit OR tune GC settings

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

The health check endpoint should answer exactly one question: "Can this container serve HTTP requests right now?" It should not check database connectivity, external APIs, or any dependency outside the container. Those checks are important for observability, but using them for the health check endpoint will cause false positives and trigger unnecessary task replacements.

---

## Scenario 5: Image Pull Failure

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

## Scenario 6: Rolling Update Stuck

### Problem:
```
Deployment initiated. Stuck at 50%. Will not complete.
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
   → Cannot launch extra tasks!
   → Add capacity OR reduce container size requirements

3. ALB health check too strict
   → New tasks healthy by ECS health check
   → But ALB health check failing (different endpoint, wrong port)
   → ECS will not drain old tasks (thinks new ones not ready)

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

## Failure Response Runbook

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

## Interview Angle

**Q: "What happens when an ECS task crashes? How does the service recover?"**

> When an essential container exits with any non-zero exit code, the entire task stops. The ECS Service Controller detects that the running task count is below the desired count and launches a replacement task on available capacity. If tasks consistently crash — for example, due to a bad deployment — the Circuit Breaker (if enabled) triggers after 10 consecutive failures and stops the deployment. With `rollback: true`, ECS automatically restores the previous working task definition revision.
>
> Graceful shutdown is handled via SIGTERM: ECS sends SIGTERM first, then waits for `stopTimeout` seconds (default 30 seconds), and sends SIGKILL if the container has not stopped. Applications should handle SIGTERM to close open connections and finish in-flight requests before exiting.

**Q: "How does ECS handle an AZ failure?"**

> When an AZ fails, all tasks running on instances in that AZ become unreachable and are marked STOPPED. The ECS Service Controller detects that the running count is below the desired count and launches replacement tasks on instances in the remaining healthy AZs. The ALB automatically stops routing traffic to targets in the failed AZ, so users are served by the remaining healthy tasks.
>
> The key design principle is to always run tasks across a minimum of three AZs, and to design maximum load capacity at 50% of total task capacity. This ensures that even after losing one of three AZs (leaving 66% capacity), the remaining tasks can handle the full traffic load.

---

*Next: [05_Observability_Logs_Metrics_Tracing.md →](./05_Observability_Logs_Metrics_Tracing.md)*
