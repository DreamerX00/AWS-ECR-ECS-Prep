# 🎯 ECS Scheduler Deep Dive

---

## 📖 Concept Explanation

ECS Scheduler = The brain of ECS. Decides: **"Which container goes where?"**

Understanding the scheduler means you can:
- Design high-availability deployments
- Prevent resource starvation
- Explain exactly how ECS places tasks
- Debug "unable to place task" errors

---

## 🏗️ Scheduler Architecture

```
ECS Scheduler (Control Plane Component)
         │
         │ Receives:
         │  - Task Definition (CPU/Memory requirements)
         │  - Placement Constraints
         │  - Placement Strategy  
         │  - Capacity Provider preference
         │
         ▼
STEP 1: FILTER eligible instances/capacity
         │
         ▼
STEP 2: APPLY CONSTRAINTS
         │
         ▼
STEP 3: APPLY STRATEGY (optimize)
         │
         ▼
STEP 4: PLACE task on selected target
```

---

## 🔍 Scheduling Types

### Type 1: REPLICA Scheduling (Service)
```
"Keep N copies running at all times."

Service desired-count=3:
  - ECS places 3 tasks across cluster
  - Task dies? → Replace immediately
  - Respects placement strategy (spread across AZs)
  - Scale to 10? → ECS places 7 more tasks
```

### Type 2: DAEMON Scheduling (Service)
```
"Run one task on EVERY container instance."

Daemon service desired-count = (NOT SET — automatic)
  - New EC2 joins cluster → Daemon task auto-placed on it
  - EC2 terminates → Daemon task removed
  
Use cases:
  - Log collectors (Fluentd, FluentBit)
  - Monitoring agents (Datadog, CloudWatch)
  - Security scanners
  
ONLY works with EC2 launch type!
```

---

## 📍 Placement Strategies

### 1. SPREAD — Distribute Evenly
```
Goal: Place tasks away from each other
Fields:
  - attribute:ecs.availability-zone (spread across AZs)
  - instanceId (spread across instances)
  - attribute:ecs.instance-type (spread across instance types)

Best for: HIGH AVAILABILITY!
Task 1 → AZ us-east-1a
Task 2 → AZ us-east-1b
Task 3 → AZ us-east-1c
AZ failure → only 1/3 tasks affected!
```

### 2. BINPACK — Maximize Density
```
Goal: Pack as many tasks as possible on one instance before using another
Fields: cpu | memory

Best for: COST OPTIMIZATION!
Instance A: [Task1 Task2 Task3 Task4 Task5] ← PACKED!
Instance B: [Task6 Task7...]
Instance C: Empty → CAN TERMINATE (save cost!)
```

### 3. RANDOM — Place Anywhere
```
Goal: No optimization, just place it
Use when: You don't care about distribution
Rarely used in production
```

### Combining Strategies (RECOMMENDED):
```bash
# Best practice: AZ spread FIRST, then binpack within AZ
--placement-strategy \
  type=spread,field=attribute:ecs.availability-zone \
  type=binpack,field=memory
```

---

## 🚧 Placement Constraints

### Type 1: distinctInstance
```bash
# Each task on a DIFFERENT EC2 instance!
--placement-constraints type=distinctInstance

Use case: 
  "No 2 replicas of payment-service on same EC2"
  Single instance failure → only 1 task lost
```

### Type 2: memberOf (cluster query language)
```bash
# Advanced filtering expression

# Only instances of specific type:
--placement-constraints \
  type=memberOf,expression="attribute:ecs.instance-type == c5.4xlarge"

# Only instances with custom attribute (e.g., SSD):
--placement-constraints \
  type=memberOf,expression="attribute:storage-type == ssd"

# Only instances in specific AZ:
--placement-constraints \
  type=memberOf,expression="attribute:ecs.availability-zone in [us-east-1a, us-east-1b]"

# Combine:
--placement-constraints \
  type=memberOf,expression="attribute:ecs.instance-type =~ g5.* and attribute:ecs.availability-zone == us-east-1a"
```

### Custom Instance Attributes:
```bash
# Set custom attribute on specific EC2 instances
aws ecs put-attributes \
  --cluster production \
  --attributes \
    name=storage-type,value=ssd,targetType=container-instance,targetId=<instance-id>

# Now tasks can target only SSD instances!
```

---

## 🔄 AZ Balancing

### How ECS Maintains AZ Balance:

```
Service desired=6, 3 AZs (a, b, c)

Initial placement (with spread strategy):
  AZ-a: [Task1, Task2]
  AZ-b: [Task3, Task4]
  AZ-c: [Task5, Task6]
  → Balanced!

Task2 crashes:
  ECS needs to replace 1 task
  
Without rebalancing: Could place in any AZ
With AZ-spread strategy: ECS prefers AZ-a (it now has 1 task vs others 2)
  AZ-a: [Task1, Task7-new] ← new task restores balance
  AZ-b: [Task3, Task4]
  AZ-c: [Task5, Task6]
  → Rebalanced!
```

---

## 🏋️ Capacity Provider Logic (Advanced Scheduling)

```
Cluster has Capacity Providers:
  FARGATE (weight=1, base=2)
  FARGATE_SPOT (weight=4)

Service desired-count=10:

Calculation:
  Base FARGATE tasks: 2 (always guaranteed regular Fargate)
  Remaining: 8 tasks
  Ratio: FARGATE:FARGATE_SPOT = 1:4
  
  FARGATE_SPOT tasks: 8 × (4/5) = 6.4 → 6 tasks
  FARGATE tasks:      8 × (1/5) = 1.6 → 2 tasks
  
  Final:
    FARGATE:      2 (base) + 2 (weighted) = 4 tasks
    FARGATE_SPOT: 6 tasks
    Total: 10 ✅

Benefits:
  4 × Fargate: Higher cost, guaranteed availability
  6 × Fargate Spot: 70% cheaper, might be interrupted
  If Spot interrupted: Service still runs on 4 Fargate tasks
  → Scale back up to 10 from remaining capacity
```

---

## 🎯 Analogy — Theater Seat Assignment 🎭

**ECS Scheduler = Theater Usher:**

- **Spread strategy:** "Let's distribute audience across all rows, don't pack row 1 first"
- **Binpack:** "Let's fill each row before moving to next (maximize row usage)"  
- **Placement constraints:** "VIP section only for premium ticket holders"
- **distinctInstance:** "No two friends from same group in same row (if one row has fire, they're not all affected)"
- **AZ balance:** "Spread audience between left section and right section equally"

---

## 🌍 Real-World Scenario

### Financial Services — Zero Single-Point-of-Failure Design

```
Service: payment-processor (CRITICAL)
Requirements:
  - Never 2 tasks on same EC2 (hardware failure isolation)
  - Spread across all 3 AZs
  - Only run on "security-hardened" instances
  - At least 4 tasks always running

Service configuration:
  Desired count: 6 (2 per AZ)
  
  Placement strategy:
    1. spread: attribute:ecs.availability-zone
    2. spread: instanceId
  
  Placement constraints:
    - distinctInstance: true
    - memberOf: attribute:security-hardened == true
  
  Deployment config:
    minimumHealthyPercent: 100   (never have < 6 tasks)
    maximumPercent: 150          (up to 9 during deployment)
  
  Result:
    AZ-a: Instance #1 [Task1], Instance #2 [Task2]
    AZ-b: Instance #3 [Task3], Instance #4 [Task4]
    AZ-c: Instance #5 [Task5], Instance #6 [Task6]
    
    EC2 failure (Instance #3): Task3 stops
    Scheduler: "Where can I place? 
      distinctInstance: not #1,#2,#4,#5,#6
      AZ balance: AZ-b has 1, others have 2 → prefer AZ-b
      security-hardened: must be tagged"
    → Finds Instance #7 (security-hardened, AZ-b) → places Task7
    → Balance restored!
```

---

## ⚙️ Hands-On Examples

### Placement Strategy Examples:
```bash
# HA pattern: AZ spread + instance density
aws ecs create-service \
  --cluster production \
  --service-name myapp \
  --task-definition myapp:5 \
  --desired-count 6 \
  --placement-strategy \
    type=spread,field=attribute:ecs.availability-zone \
    type=binpack,field=memory \
  --placement-constraints \
    type=distinctInstance

# Monitoring daemon (one per instance):
aws ecs create-service \
  --cluster production \
  --service-name monitoring-agent \
  --task-definition monitoring:3 \
  --scheduling-strategy DAEMON

# Debug placement failures:
aws ecs describe-services --cluster prod --services myapp | \
  jq '.services[0].events[] | select(.message | contains("unable to place"))'
```

### "Unable to Place Task" Debugging:
```bash
# Common causes:
# 1. Not enough CPU/Memory on any instance
aws ecs list-container-instances --cluster production | \
  xargs -I{} aws ecs describe-container-instances --cluster production --container-instances {} | \
  jq '.containerInstances[] | {instanceId:.ec2InstanceId, cpuFree:.remainingResources[] | select(.name=="CPU") | .integerValue, memFree:.remainingResources[] | select(.name=="MEMORY") | .integerValue}'

# 2. Constraint too restrictive
# Check if hosts match constraint:
aws ecs list-container-instances \
  --cluster production \
  --filter "attribute:security-hardened == true"

# 3. No instances in required AZ
aws ecs list-container-instances \
  --cluster production \
  --filter "attribute:ecs.availability-zone == us-east-1c"
```

---

## 🚨 Gotchas & Edge Cases

### 1. Binpack Can Cause AZ Imbalance
```
Binpack: Fill one instance before another
If all instances in AZ-a have highest capacity:
  → All tasks go to AZ-a → SINGLE AZ!
  
Fix: Use spread first (AZ), then binpack within AZ
placement-strategy: spread(AZ) → binpack(memory)
```

### 2. distinctInstance With Fargate
```
distinctInstance constraint = EC2 only concept
Fargate doesn't have "instances" — each task IS its own isolated VM
Don't apply distinctInstance with Fargate (it's a no-op but confusing)
```

### 3. minimumHealthyPercent During Deploys
```
minimumHealthyPercent=100 (no downtime) + maximumPercent=200:
  Deploy 6 new tasks → wait for all healthy → stop 6 old
  = Blue/green-ish deployment
  = Takes 2x capacity temporarily
  
minimumHealthyPercent=50 (allow some downtime) + maximumPercent=150:
  Stop 3 old → start 3 new → wait → stop remaining 3 old → start 3 new
  = Less capacity overhead
  = Brief capacity reduction during deployment (acceptable for some apps)
```

---

## 🎤 Interview Angle

**Q: "ECS scheduler placement decision kaise karta hai?"**

> Scheduler 3 phases mein kaam karta hai:
> 1. Eligible hosts filter (connected agents, non-draining, has capacity)
> 2. Constraint filtering (distinctInstance, memberOf expressions)
> 3. Strategy application (spread/binpack/random on filtered hosts)
> First constraint violations eliminate hosts, then strategy picks the optimal host from remaining.

**Q: "Daemon service kya hai aur kab use karte hain?"**

> Daemon service = har EC2 container instance pe exactly 1 task.
> New instance cluster join kare → daemon task auto-start.
> Instance terminate → daemon task auto-stop.
> Use cases: Log agents (Fluentd), monitoring exporters, security scanners.
> Only EC2 mode — Fargate pe daemon service concept nahi hai (tasks = isolated VMs).

---

*Next: [06_ECS_Security_Model.md →](./06_ECS_Security_Model.md)*
