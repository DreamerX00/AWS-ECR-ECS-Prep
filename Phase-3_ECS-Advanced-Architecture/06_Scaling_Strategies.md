# ⚡ Scaling Strategies — Auto-Scaling ECS

---

## 📖 Two Levels of Scaling

```
ECS Scaling has TWO distinct levels:

Level 1: SERVICE Auto Scaling    → Scale number of TASKS
Level 2: CAPACITY Auto Scaling  → Scale number of EC2 INSTANCES

Fargate: Only Level 1 (AWS handles compute automatically)
EC2 mode: Both levels needed!
```

---

## 📈 Service Auto Scaling — 3 Strategies

### 1. Target Tracking (Recommended)
```
"Keep CPU at 70%, scale tasks automatically to achieve this"

Like a cruise control — set target, AWS adjusts.

Example: Desired CPUUtilization = 70%
  Traffic spikes → CPU goes to 90%
  ECS: Increase tasks → CPU drops back to 70% ✓
  Traffic drops → CPU at 30%
  ECS: Remove tasks → CPU returns to 70% ✓
```

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production/myapp \
  --min-capacity 2 \   ← Always keep at least 2
  --max-capacity 20    ← Never exceed 20

# Create Target Tracking scaling policy (CPU)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production/myapp \
  --policy-name myapp-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleInCooldown": 300,   ← Wait 5 min before scaling in again
    "ScaleOutCooldown": 30,   ← Wait 30s before scaling out again
    "DisableScaleIn": false
  }'

# Memory-based scaling:
aws application-autoscaling put-scaling-policy \
  ...
  --target-tracking-scaling-policy-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
    },
    "TargetValue": 75.0,
    ...
  }'

# ALB Request Count per target (BEST for web services!):
aws application-autoscaling put-scaling-policy \
  ...
  --target-tracking-scaling-policy-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/myalb/abc123/targetgroup/myapp-tg/def456"
    },
    "TargetValue": 1000.0,   ← 1000 req/target/min
    ...
  }'
```

### 2. Step Scaling
```
"When CPU > 85%, add 3 tasks. When CPU > 95%, add 10 tasks."

More granular control but more configuration.

Scale Out steps:
  50-70% CPU: No action
  70-85% CPU: +2 tasks
  85-95% CPU: +4 tasks
  95%+  CPU:  +8 tasks

Scale In:
  30-50% CPU: -1 task (gentle)
  < 30% CPU:  -3 tasks
```

```bash
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production/myapp \
  --policy-name myapp-step-scaling \
  --policy-type StepScaling \
  --step-scaling-policy-configuration '{
    "AdjustmentType": "ChangeInCapacity",
    "Cooldown": 60,
    "StepAdjustments": [
      {
        "MetricIntervalLowerBound": 0,
        "MetricIntervalUpperBound": 15,
        "ScalingAdjustment": 2
      },
      {
        "MetricIntervalLowerBound": 15,
        "MetricIntervalUpperBound": 25,
        "ScalingAdjustment": 4
      },
      {
        "MetricIntervalLowerBound": 25,
        "ScalingAdjustment": 8
      }
    ]
  }'
# This policy triggered by CloudWatch alarm when CPU > 70%
```

### 3. Scheduled Scaling
```
"Every weekday at 8 AM, scale to 10 tasks. At 8 PM, scale to 3."

Predictable patterns → schedule in advance.

Examples:
  - Office hours: 9 AM-6 PM → high capacity
  - Cricket match = peak TV streaming → pre-scale before the match
  - Month-end payroll processing → pre-scale on last day of month
```

```bash
# Scale up on Monday morning (cron: 8 AM Monday UTC)
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production/myapp \
  --scheduled-action-name scale-up-weekday \
  --schedule "cron(0 8 ? * MON-FRI *)" \
  --scalable-target-action MinCapacity=10,MaxCapacity=20

# Scale down in evening (8 PM Monday-Friday)
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --resource-id service/production/myapp \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name scale-down-evening \
  --schedule "cron(0 20 ? * MON-FRI *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=5
```

---

## 🖥️ Capacity Auto Scaling (EC2 Mode)

```
ECS Managed Scaling:
  ECS calculates needed capacity → tells Auto Scaling Group → ASG scales

Without it:
  ECS: "I need to place 5 more tasks but no capacity!"
  → Tasks stuck in PROVISIONING state

With it:
  ECS: "Need 5 more tasks" → 
  ECS calculates: "Need 2 more EC2 instances" →
  ASG: Launch 2 EC2s →
  Tasks placed → problem solved
```

```bash
# Capacity Provider with Managed Scaling:
aws ecs create-capacity-provider \
  --name my-ec2-cp \
  --auto-scaling-group-provider '{
    "autoScalingGroupArn": "arn:aws:autoscaling:...",
    "managedScaling": {
      "status": "ENABLED",
      "targetCapacity": 80,      ← Keep ASG at 80% task capacity
      "minimumScalingStepSize": 1,
      "maximumScalingStepSize": 5,
      "instanceWarmupPeriod": 300  ← Wait 5 min for new EC2 to be ready
    },
    "managedTerminationProtection": "ENABLED"  ← Don't kill EC2 with running tasks!
  }'
```

---

## 🎯 Analogy — Restaurant Scaling 🍕

**Target Tracking = Restaurant Auto Staffing:**
- "Keep waiter utilization at 70%"
- Rush hour: 10 waiters busy each serving 10 tables → hire 2 more temp waiters
- Slow hour: 4 waiters idle → send 3 home
- Manager adjusts automatically!

**Scheduled Scaling = Pre-planned Staffing:**
- "Saturday evening is always busy — schedule 15 staff for 6-10 PM"
- Don't wait for crowd to arrive → staff in advance!

**Step Scaling = Emergency Response:**
- "Normal: 5 waiters. If tables > 20: add 3 more. If tables > 40: add 10 more"
- Graduated response based on severity

---

## 🌍 Real-World: E-commerce Black Friday

```
Normal traffic: 100 req/sec → 5 tasks
Black Friday prediction: 10,000 req/sec → 500 tasks

Strategy:
  1. Scheduled: 6 AM (before Black Friday begins) → scale to 200 tasks
  2. Target Tracking (CPU 70%): Auto-add as load increases
  3. Custom metric: ALBRequestCountPerTarget = 500 → scale
  4. Step scaling for emergencies: if error rate > 5% → +50 tasks immediately
  
  Day before:
  - Test scale: Scale to 300 tasks in staging → verify behavior
  - Pre-warm EC2 (EC2 mode): Launch instances in morning, cache ECR images
  
  During event:
  - Monitor: CPU, Memory, Error rate, Latency
  - Scale out cooldown: 30s (fast response!)
  - Scale in cooldown: 600s (don't scale in during flash sale!)
  
  After event:
  - 11 PM: Manually scale back to 10 tasks (no more need for 500)
```

---

## ⚙️ Scaling Best Practices

```bash
# ALWAYS set min and max capacity
--min-capacity 2    ← Never go below 2 (HA: at least 2 for rolling deploy)
--max-capacity 50   ← Cost protection (prevents runaway scaling)

# Scale out fast, scale in slow:
ScaleOutCooldown: 30   ← Respond quickly to traffic spikes
ScaleInCooldown: 300   ← Don't be trigger-happy to remove capacity

# Use multiple policies together:
# 1. Scheduled: Pre-scale before known events
# 2. Target Tracking: Handle variable load automatically
# 3. Step Scaling: Emergency rapid scale (triggered by alarm)

# Watch out for:
# Desired count reaches maxCapacity → triggers alarm "MaxCapacityReached"
# → Manual intervention needed!
```

---

## 🚨 Gotchas & Edge Cases

### 1. Scale-In Protection
```bash
# Prevent specific tasks from being scale-in terminated
# Useful for tasks processing long-running jobs

aws ecs update-task-protection \
  --cluster production \
  --tasks <arn> \
  --protection-enabled \
  --expires-in-minutes 60
  
# Task won't be terminated for 60 minutes during scale-in
# After 60 min → protection expires → eligible for scale-in
```

### 2. Scale Out Faster Than EC2 Acquisition (EC2 Mode)
```
ECS needs 10 more tasks → Capacity Provider: need 3 more EC2s
EC2 launch: 3-5 minutes
Tasks waiting: PENDING for 5 minutes!

Solutions:
  - Fargate: No EC2 to provision, tasks start in ~30-60s
  - Warm pool: Pre-started EC2s waiting in ASG
  - Oversized instances: Fewer, larger instances → more tasks per instance
```

### 3. Task Count vs Request Rate Mismatch
```
Metric: CPU 70% → scale enabled
Problem: All CPUspent on 1 long-running background task, not HTTP requests
HTTP requests fast → response time OK
But CPU scaling keeps adding tasks unnecessarily!

Better metric: ALBRequestCountPerTarget + Latency-based scaling
CPU: Use only for CPU-bound workloads
```

---

## 🎤 Interview Angle

**Q: "ECS auto scaling strategies kya hain? Kab kaunsi use karein?"**

> 3 Service Scaling strategies:
> 1. Target Tracking: Metric target maintain karo (CPU 70%, ReqPerTarget 1000) — simplest, recommended most cases.
> 2. Step Scaling: Different actions at different thresholds — traffic spikes ke liye graduated response.
> 3. Scheduled: Predictable patterns ke liye — office hours, events, month-end.
>
> EC2 mode mein additionally Capacity Provider Managed Scaling needed — ECS automatically tells ASG to add/remove instances.
>
> Best combo: Scheduled (pre-scale) + Target Tracking (auto-adjust) + Step Scaling (emergency).

---

*Next: [07_Secrets_Handling.md →](./07_Secrets_Handling.md)*
