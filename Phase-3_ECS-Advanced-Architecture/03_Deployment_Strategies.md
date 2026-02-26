# 🔄 Deployment Strategies — Zero Downtime & Beyond

---

## 📖 Concept

ECS supports multiple deployment strategies. Choosing the right one = difference between 2AM incident and peaceful sleep.

---

## 1️⃣ Rolling Update (Default)

```
Deploy new image with controlled rollout.

Parameters:
  minimumHealthyPercent: 50 → At least 50% of desired tasks stay running
  maximumPercent: 200 → Up to 2x desired tasks during deployment

Execution with desired=4:
  Initial:  [v1, v1, v1, v1]
  Phase 1:  [v1, v1, v1, v1, v2, v2, v2, v2]  ← (200% = max 8 tasks)
            Wait for v2 to pass health checks...
  Phase 2:  [v2, v2, v2, v2]  ← Old v1 stopped
  Complete!
```

### Rolling Update Configuration:
```bash
aws ecs update-service \
  --cluster production \
  --service myapp \
  --task-definition myapp:6 \   # New revision!
  --deployment-configuration '{
    "maximumPercent": 200,
    "minimumHealthyPercent": 50,
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }'
```

### Circuit Breaker:
```
Problem: New image has a bug → tasks crash immediately → ECS keeps launching → infinite launch-crash loop!

Circuit Breaker:
  - Monitors consecutive task failures
  - Default threshold: 10 failures
  - After 10 failures → STOP deployment
  - If rollback=true → automatically revert to previous task definition!

States:
  COMPLETED    → Deployment succeeded
  FAILED       → Circuit breaker triggered (without rollback)
  ROLLED_BACK  → Circuit breaker triggered + rollback applied
```

---

## 2️⃣ Blue/Green Deployment (via CodeDeploy)

```
The Gold Standard for zero-downtime deployments.

Blue Environment (CURRENT):
  ALB → Blue Target Group → [v1, v1, v1]

Green Environment (NEW):
  ALB → Green Target Group → [v2, v2, v2]  (all new tasks!)

Phases:
  Phase 1: Launch GREEN tasks (v2) → pass health checks (no traffic yet)
  Phase 2: "Test Traffic" (optional): 5% to green → verify
  Phase 3: "Cut-over": ALL traffic → GREEN (instant!)
  Phase 4: Keep BLUE running for X minutes (rollback window!)
  Phase 5: Terminate BLUE tasks
  
Rollback:
  Switch ALB listener back to Blue Target Group → instant rollback!
  (Automated if deployment fails health checks)
```

### Blue/Green Setup:
```bash
# 1. Create CodeDeploy Application
aws deploy create-application \
  --application-name myapp-deploy \
  --compute-platform ECS

# 2. Create CodeDeploy Deployment Group
aws deploy create-deployment-group \
  --application-name myapp-deploy \
  --deployment-group-name myapp-dg \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --ecs-services '[{
    "clusterName": "production",
    "serviceName": "myapp"
  }]' \
  --load-balancer-info '{
    "targetGroupPairInfoList": [{
      "targetGroups": [
        {"name": "myapp-blue-tg"},   ← Blue target group
        {"name": "myapp-green-tg"}   ← Green target group
      ],
      "prodTrafficRoute": {
        "listenerArns": ["arn:...:listener/myalb/xxx/prod-listener"]
      },
      "testTrafficRoute": {
        "listenerArns": ["arn:...:listener/myalb/xxx/test-listener"]
      }
    }]
  }' \
  --service-role-arn arn:aws:iam::123:role/CodeDeployECSRole \
  --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_REQUEST

# 3. Trigger Deployment
aws deploy create-deployment \
  --application-name myapp-deploy \
  --deployment-group-name myapp-dg \
  --revision '{
    "revisionType": "AppSpecContent",
    "appSpecContent": {
      "content": "{\"version\":0.0,\"Resources\":[{\"TargetService\":{\"Type\":\"AWS::ECS::Service\",\"Properties\":{\"TaskDefinition\":\"arn:aws:ecs:...:task-definition/myapp:6\",\"LoadBalancerInfo\":{\"ContainerName\":\"app\",\"ContainerPort\":3000}}}}]}"
    }
  }'
```

### Blue/Green Deployment Configurations:
```
CodeDeployDefault.ECSAllAtOnce:
  Shift ALL traffic to green immediately
  Highest risk, fastest deployment
  
CodeDeployDefault.ECSLinear10PercentEvery1Minutes:
  10% traffic → green every 1 minute
  Full cutover in 10 minutes
  Good for gradual validation
  
CodeDeployDefault.ECSCanary10Percent5Minutes:
  10% traffic → green for 5 minutes
  If healthy: remaining 90% switched
  Good for testing with real traffic
```

---

## 3️⃣ Canary Deployment (Custom)

```
Golden rule: "Expose 1% of users to new version first"

Implementation (custom via weighted target groups):

ALB Listener Rule:
  → Weighted Forward:
      Blue TG: weight=99  → 99% traffic to v1
      Green TG: weight=1  → 1% traffic to v2

Monitor for 30 minutes...
If metrics OK → shift more:
  Blue: weight=90, Green: weight=10

Continue until:
  Blue: weight=0, Green: weight=100
  
Rollback: Blue: weight=100, Green: weight=0 (instant!)
```

---

## ⚙️ minimumHealthyPercent Deep Dive

```
Setting          Effect During Deployment        When to Use
─────────────────────────────────────────────────────────────
100%             No downtime. Extra tasks needed. Critical production apps
50%              Brief capacity reduction OK.     Most web services
0%               All old tasks stop first!         Dev/test, batch processors
```

---

## 🌍 Real-World: SaaS Product Deploy Decision

```
daily traffic: 10K req/s, SLA: 99.9% uptime

Deployment Strategy Assessment:

Rolling Update:
  Pro: Simple, built-in to ECS
  Con: 30-50% capacity reduction during deploy = risk during peak
  Risk: If new version broken, some users affected before rollback
  Verdict: OK for low-risk updates (config changes)

Blue/Green:
  Pro: Zero traffic interruption, clean rollback
  Pro: Can test at test port before prod traffic
  Con: Requires CodeDeploy setup, 2x capacity briefly
  Verdict: ✅ BEST for feature deploys in production

Canary:
  Pro: 1% of users see new version first
  Pro: Real traffic validation without full risk
  Con: More complex monitoring, longer deployment time
  Verdict: ✅ BEST for risky changes, major versions
```

---

## 🚨 Gotchas & Edge Cases

### 1. Rolling Update Can Have TWO Versions Simultaneously
```
minimumHealthyPercent=50, desired=4:
While 2 old tasks are being replaced:
  2 tasks running v1 + 2 tasks running v2 = both serve traffic!

If v2 has BREAKING API CHANGES → clients see different responses!
Solution: Use Blue/Green (all-or-nothing switch)
```

### 2. ECS Service Wait — Deployment Completion Detection
```bash
# Wait for deployment to complete (useful in CI/CD)
aws ecs wait services-stable \
  --cluster production \
  --services myapp

# Check deployment status
aws ecs describe-services \
  --cluster production \
  --services myapp \
  --query 'services[0].deployments'
# deployments array: PRIMARY (current), ACTIVE (old being drained)
# Once only PRIMARY remains → deployment complete
```

### 3. Blue/Green Bake Time
```
bakeTime (in CodeDeploy): How long to keep Blue running after Green gets 100% traffic
Default: 0 (terminate immediately)
Recommendation: 15-30 minutes bake time → quick rollback if issue detected post-deploy

Metric to monitor during bake time:
  - Error rate
  - Latency p99
  - Business metrics (order rate, login rate)
```

---

## 🎤 Interview Angle

**Q: "Zero-downtime deployment ECS mein kaise achieve karte hain?"**

> Rolling Update: minimumHealthyPercent=100, maximumPercent=200 → New tasks start + registered with ALB → old tasks drain and stop → zero downtime.
> Blue/Green: CodeDeploy ke saath → complete green environment launch → instant traffic switch → rollback in seconds.
> Blue/Green preferred for: large releases, breaking changes, high-traffic apps.
> Rolling update better for: quick patches, minor changes.

**Q: "Circuit breaker ECS mein kaise kaam karta hai?"**

> Deployment ke dauraan agar new tasks repeatedly fail → Circuit Breaker track karta hai failures.
> Default 10 consecutive failures → deployment automatically STOPPED.
> If rollback=true → previous working task definition automatically restored.
> Business continuity during bad deploys!

---

*Next: [04_Failure_Scenarios.md →](./04_Failure_Scenarios.md)*
