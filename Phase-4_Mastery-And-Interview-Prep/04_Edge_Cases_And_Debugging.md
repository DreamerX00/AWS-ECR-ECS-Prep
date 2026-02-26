# 🧨 Edge Cases & Debugging Mastery

---

## 🔴 EDGE CASE 1: Image Pull Failures

### Scenario A: Private Subnet + No VPC Endpoint
```
Error: "CannotPullContainerError: Get https://...amazonaws.com: dial tcp: i/o timeout"

Root cause: Task in private subnet → ECR needs internet → No NAT Gateway!

Fix options:
  1. Add VPC endpoints (recommended):
     - com.amazonaws.us-east-1.ecr.dkr   (image pulls)
     - com.amazonaws.us-east-1.ecr.api   (API calls)
     - com.amazonaws.us-east-1.s3        (S3 gateway, for layer blobs)
  
  2. Add NAT Gateway (expensive but quick fix)
  
Debug command:
  # Test from EC2 in same subnet:
  curl -I https://123456789.dkr.ecr.us-east-1.amazonaws.com/v2/
  # Timeout = no connectivity. 200 = OK.
```

### Scenario B: Wrong Execution Role Permissions
```
Error: "AccessDeniedException: User: arn:aws:sts::123:assumed-role/ecsTaskExecutionRole/..."

Debug:
  aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=GetAuthorizationToken
  # → See who called it, what denied

  # Add to Execution Role:
  ecr:GetAuthorizationToken   → Resource: * (required)
  ecr:BatchGetImage
  ecr:GetDownloadUrlForLayer
  ecr:BatchCheckLayerAvailability
```

---

## 🔴 EDGE CASE 2: ENI Exhaustion (EC2 awsvpc mode)

```
Symptom: Tasks stuck in PROVISIONING for minutes
Error in agent logs: "no free ENI"

Debug:
  # Count ENIs per instance:
  aws ec2 describe-instances --instance-ids i-abc \
    --query 'Reservations[0].Instances[0].NetworkInterfaces[*].NetworkInterfaceId'
  
  # Check ENI limit for instance type:
  aws ec2 describe-instance-types --instance-types c5.xlarge \
    --query 'InstanceTypes[0].NetworkInfo.MaximumNetworkInterfaces'

Fix:
  1. Short term: Use fewer but larger instances (more ENIs per instance)
  2. Best: Enable ENI Trunking:
     aws ecs update-cluster-settings \
       --cluster production \
       --settings name=awsvpcTrunking,value=enabled
     → Up to 120 tasks per instance!
  3. Alternative: Move to Fargate (no ENI limit problem)
```

---

## 🔴 EDGE CASE 3: Out of Memory Kill

```
Symptom: Container exits with code 137 (SIGKILL from kernel OOM killer)
         OR: "OutOfMemoryError: Container killed due to memory usage exceeding" in ECS events

Debug:
  # Check reason
  aws ecs describe-tasks --cluster prod --tasks <arn> \
    --query 'tasks[0].containers[0].{exitCode:exitCode,reason:reason}'
  # → {"exitCode": 137, "reason": "...OOM..."}

  # CloudWatch Container Insights: Memory trend
  # MemoryUtilized → approaching MemoryReserved limit? → OOM likely

Fix (immediate):
  # Increase memory hard limit:
  "memory": 2048  →  "memory": 4096
  
Fix (root cause):
  1. Memory leak: Analyze heap dumps, fix leak
  2. Natural growth: Increase limit appropriately
  3. JVM: Set -XX:MaxRAMPercentage=75 (proper container awareness)
  4. Node.js: --max-old-space-size=XYYY where XYYY < (memory limit × 0.8 in MB)
  
Node.js memory cap:
  Container memory: 1024MB
  Node heap: 1024 × 0.8 = 819MB → --max-old-space-size=819
  This prevents Node from hitting OOM before GC can run!
```

---

## 🔴 EDGE CASE 4: Health Check Flapping

```
Symptom: Task keeps getting deregistered from ALB → registered → deregistered
         ALB errors: HTTPCode_Target_5XX
         ECS events: "Service has reached a steady state (but tasks being replaced)"

Debug:
  # Check ALB target health:
  aws elbv2 describe-target-health \
    --target-group-arn arn:...targetgroup/myapp-tg/...
  # Look for: "State": "unhealthy", "Reason": "Target.ResponseCodeMismatch"

  # Check ALB access logs (S3) for health check requests:
  grep "GET /health" access-logs.txt | awk '{print $9}'  # HTTP status codes

Fix strategies:
  1. Lightweight health endpoint:
     /health → 200 OK instantly (no DB check, no external calls)
  
  2. Increase health check tolerances:
     unhealthyThresholdCount: 5 (vs default 2)
     healthCheckIntervalSeconds: 60 (vs default 30)
  
  3. GC pause causing timeouts:
     healthCheckTimeoutSeconds: 10 (increase timeout)
     Or: Add JVM GC tuning to reduce pause duration
  
  4. Slow start (Spring Boot, heavy JVM):
     startPeriod: 120  # in Task Definition ECS health check
     ALB: healthCheckIntervalSeconds: 60 + unhealthyThresholdCount: 5
```

---

## 🔴 EDGE CASE 5: IAM Permission Denial at Runtime

```
Symptom: App working locally, fails in ECS with "AccessDeniedException"

Debug process:
  # Step 1: Find which API call is failing
  aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventSource,AttributeValue=s3.amazonaws.com \
    --lookup-attributes AttributeKey=ErrorCode,AttributeValue=AccessDenied \
    --start-time 2024-01-15T10:00:00Z \
    --end-time 2024-01-15T11:00:00Z
  
  # Step 2: See exact resource and action:
  {
    "eventName": "GetObject",
    "requestParameters": {
      "bucketName": "my-bucket",
      "key": "some/key"
    },
    "userIdentity": {
      "arn": "arn:aws:sts::123:assumed-role/myapp-task-role/..."  ← Task Role!
    }
  }
  
  # Step 3: Simulate the permission check:
  aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123:role/myapp-task-role \
    --action-names s3:GetObject \
    --resource-arns arn:aws:s3:::my-bucket/some/key

  # If "DENY" → add to Task Role!
  # If "ALLOW" → check resource-based policy (bucket policy blocking?)
```

---

## 🔴 EDGE CASE 6: Rolling Update Stuck at 50%

```
Symptom: "2 of 4 tasks updated" message stays for 10+ minutes
         aws ecs describe-services shows 2 ACTIVE deployments

Debugging decision tree:

1. New tasks launching?
   aws ecs list-tasks --cluster prod --service-name myapp --desired-status RUNNING
   
   a. NEW TASKS EXIST → they're just not passing health checks
      Check: aws ecs describe-tasks → container health status
      Check: ALB target health (are new tasks HEALTHY in target group?)
      → Fix the health check issue!
   
   b. NO NEW TASKS → placement issue
      Check: aws ecs describe-services → events → "unable to place"
      Common: Not enough capacity (EC2 mode) OR Fargate limit hit
      Fix: Add capacity OR reduce task CPU/memory

2. Tasks are healthy but deployment not completing?
   Check: deregistrationDelay setting
   aws elbv2 describe-target-group-attributes \
     --target-group-arn arn:...:targetgroup/myapp-tg/...
   → deregistration_delay.timeout_seconds = 300 (5 minutes!)
   
   Fix: Set to 30-60 seconds for stateless HTTP services!

3. Circuit breaker triggered (if enabled)?
   aws ecs describe-services | jq '.services[0].deployments[0].rolloutState'
   → "FAILED" or "COMPLETED"
   Rollout state FAILED with rollback=true → Already rolled back automatically!
```

---

## 🛠️ Master Debug Cheat Sheet

```bash
# === QUICK TRIAGE COMMANDS ===

# 1. Service health overview
aws ecs describe-services --cluster production --services myapp \
  | jq '{desired:.services[0].desiredCount, running:.services[0].runningCount, pending:.services[0].pendingCount, events:.services[0].events[:3]}'

# 2. Why did task stop?
TASK=$(aws ecs list-tasks --cluster production --service-name myapp --desired-status STOPPED --query 'taskArns[0]' --output text)
aws ecs describe-tasks --cluster production --tasks $TASK \
  | jq '{stoppedReason:.tasks[0].stoppedReason, containers:.tasks[0].containers[].{name:.name,exit:.exitCode,reason:.reason}}'

# 3. Real-time logs
aws logs tail /ecs/myapp --follow --format short

# 4. Get into a running container
TASK=$(aws ecs list-tasks --cluster production --service-name myapp --query 'taskArns[0]' --output text)
aws ecs execute-command --cluster production --task $TASK --container app --interactive --command "/bin/sh"

# 5. Check CloudTrail for permission denials (last 1 hour)
aws cloudtrail lookup-events \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --lookup-attributes AttributeKey=ErrorCode,AttributeValue=AccessDenied \
  | jq '.Events[] | {time:.EventTime, who:.Username, what:.EventName, service:.EventSource}'

# 6. Manually stop and force replacement
aws ecs stop-task --cluster production --task $TASK --reason "Manual replacement for debug"

# 7. Force new deployment (replace all tasks)
aws ecs update-service --cluster production --service myapp --force-new-deployment
```

---

## 🎤 Interview Angle

**Q: "ECS Container jo chalat nahi isko kaise debug karoge step by step?"**

> My systematic approach:
> 1. **Service events first:** `describe-services` → `.events` → "unable to place" (capacity issue) or "task stopped"
> 2. **Stopped task reason:** `describe-tasks` → `stoppedReason` + `containers[].exitCode`
>    - Exit 137 = OOM → increase memory
>    - TaskFailedToStart → image pull or secret fetch issue
>    - Exit 1 = app error → check logs
> 3. **Logs:** `aws logs tail /ecs/myapp --follow` → actual error message
> 4. **If IAM:** CloudTrail → AccessDenied events → which API call failing → add to Task Role
> 5. **If network:** VPC Flow logs or test connectivity from same VPC/subnet
> 6. **Live debug:** `ecs execute-command` → exec into running container → investigate live
> 7. **Permission simulation:** `iam simulate-principal-policy` → test before changing policies

---

*🎉 Curriculum Complete! You are now equipped to be an AWS ECR + ECS God!*
