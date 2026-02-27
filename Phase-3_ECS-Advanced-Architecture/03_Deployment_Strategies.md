# Deployment Strategies — Zero Downtime & Beyond

---

## Concept

ECS supports multiple deployment strategies. Choosing the right one is the difference between a 2 AM incident and a peaceful night's sleep. The strategy you select determines how traffic is shifted, how rollbacks work, and what happens to users during the transition.

---

## 1. Rolling Update (Default)

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

The Circuit Breaker is one of the most important production safety features in ECS. Without it, a bad deployment can enter an infinite loop where ECS keeps launching tasks that immediately crash, consuming resources and generating noise. Always enable the Circuit Breaker with `rollback: true` for production services.

---

## 2. Blue/Green Deployment (via CodeDeploy)

```
The gold standard for zero-downtime deployments.

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
        {"name": "myapp-blue-tg"},   # Blue target group
        {"name": "myapp-green-tg"}   # Green target group
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

## 3. Canary Deployment (Custom)

```
Golden rule: "Expose 1% of users to new version first"

Implementation (custom via weighted target groups):

ALB Listener Rule:
  → Weighted Forward:
      Blue TG: weight=99  → 99% traffic to v1
      Green TG: weight=1  → 1% traffic to v2

Monitor for 30 minutes...
If metrics look acceptable → shift more:
  Blue: weight=90, Green: weight=10

Continue until:
  Blue: weight=0, Green: weight=100

Rollback: Blue: weight=100, Green: weight=0 (instant!)
```

Canary deployments are particularly effective when you want to validate a change with real user traffic before committing to a full rollout. The key is to instrument your services well enough to detect problems at 1% traffic before they impact all users. Monitor error rates, latency p99, and business metrics during the canary phase.

---

## minimumHealthyPercent Deep Dive

```
Setting          Effect During Deployment        When to Use
─────────────────────────────────────────────────────────────
100%             No downtime. Extra tasks needed. Critical production apps
50%              Brief capacity reduction OK.     Most web services
0%               All old tasks stop first!         Dev/test, batch processors
```

Setting `minimumHealthyPercent=100` and `maximumPercent=200` is the safest rolling update configuration. New tasks are added (up to 200%) and confirmed healthy before any old tasks are removed. The downside is that this temporarily doubles your compute costs and requires sufficient capacity for the extra tasks.

---

## Real-World: SaaS Product Deploy Decision

```
Daily traffic: 10K req/s, SLA: 99.9% uptime

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
  Verdict: Best for feature deploys in production

Canary:
  Pro: 1% of users see new version first
  Pro: Real traffic validation without full risk
  Con: More complex monitoring, longer deployment time
  Verdict: Best for risky changes, major versions
```

---

## Gotchas & Edge Cases

### 1. Rolling Update Can Have TWO Versions Simultaneously
```
minimumHealthyPercent=50, desired=4:
While 2 old tasks are being replaced:
  2 tasks running v1 + 2 tasks running v2 = both serve traffic!

If v2 has BREAKING API CHANGES → clients see different responses!
Solution: Use Blue/Green (all-or-nothing switch)
```

This is one of the most dangerous aspects of rolling deployments. If your new API version removes a field that old clients depend on, or changes the format of a response, users will see inconsistent behavior during the rollout window. Always use Blue/Green for breaking API changes.

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

Metrics to monitor during bake time:
  - Error rate
  - Latency p99
  - Business metrics (order rate, login rate)
```

The bake time is your rollback insurance policy. During this window, if any monitoring alarm fires, you can revert to the Blue environment with a single API call. After the bake window expires, Blue tasks are terminated and rollback requires redeploying the previous version from scratch.

### 4. Task Definition Immutability

When you update a service with a new task definition revision, ECS does not modify running tasks in place. It always starts new tasks with the new revision and stops old ones. This means every deployment, even a minor configuration change, involves a full task replacement cycle. Plan your health check grace periods and deregistration delays accordingly.

### 5. Deployment Alarms (CodeDeploy)

CodeDeploy supports CloudWatch alarms that automatically stop or roll back a deployment if specific metrics breach thresholds. Configure alarms on error rate, latency, and task health count before starting a deployment. This provides an automated safety net without requiring manual monitoring during every deploy.

---

## Interview Angle

**Q: "How do you achieve zero-downtime deployments in ECS?"**

> Rolling Update: Set minimumHealthyPercent=100 and maximumPercent=200. New tasks start and register with the ALB, old tasks drain their connections and stop, and traffic is never interrupted because healthy tasks are always available throughout the transition.
>
> Blue/Green via CodeDeploy: A complete green environment is launched with all new tasks. Traffic is switched instantaneously at the ALB listener level. Rollback takes less than one second by switching the listener back to the Blue target group.
>
> Blue/Green is preferred for: large releases, breaking API changes, and high-traffic applications. Rolling update is better for: quick patches and minor changes.

**Q: "How does the circuit breaker work in ECS?"**

> During a deployment, if new tasks repeatedly fail (crash loops, health check failures), the Circuit Breaker tracks consecutive failures. After the default threshold of 10 consecutive failures, the deployment is automatically stopped. If `rollback: true` is configured, ECS automatically restores the previous working task definition revision. This prevents runaway deployment loops and ensures business continuity when a bad image is deployed.

---

*Next: [04_Failure_Scenarios.md →](./04_Failure_Scenarios.md)*
