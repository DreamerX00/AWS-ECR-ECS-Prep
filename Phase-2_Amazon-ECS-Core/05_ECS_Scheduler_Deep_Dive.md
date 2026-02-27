# ECS Scheduler Deep Dive

---

## Concept Explanation

The ECS Scheduler is the brain of ECS. It answers one question: **"Which container goes where?"**

Understanding the scheduler means you can:
- Design high-availability deployments
- Prevent resource starvation
- Explain exactly how ECS places tasks
- Debug "unable to place task" errors confidently

---

## Scheduler Architecture

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

This pipeline is strictly sequential. Constraints eliminate candidate hosts; strategy selects the optimal host among those that remain. If constraints eliminate all candidates, the task cannot be placed and remains in PENDING state with an "unable to place task" event.

---

## Scheduling Types

### Type 1: REPLICA Scheduling (Service)
```
"Keep N copies running at all times."

Service desired-count=3:
  - ECS places 3 tasks across the cluster
  - Task crashes? → Replace immediately
  - Respects placement strategy (spread across AZs)
  - Scale to 10? → ECS places 7 more tasks
```

### Type 2: DAEMON Scheduling (Service)
```
"Run one task on EVERY container instance."

Daemon service desired-count = (NOT SET — automatic)
  - New EC2 joins cluster → Daemon task auto-placed on it
  - EC2 terminates → Daemon task removed automatically

Use cases:
  - Log collectors (Fluentd, FluentBit)
  - Monitoring agents (Datadog, CloudWatch)
  - Security scanners

ONLY works with EC2 launch type!
```

Daemon services are fundamentally different from replica services. You never set a desired count; instead, ECS calculates the desired count as equal to the number of registered container instances. This makes daemon services ideal for infrastructure-level agents that must be present on every host.

---

## Placement Strategies

### 1. SPREAD — Distribute Evenly
```
Goal: Place tasks as far apart from each other as possible
Fields:
  - attribute:ecs.availability-zone (spread across AZs)
  - instanceId (spread across instances)
  - attribute:ecs.instance-type (spread across instance types)

Best for: HIGH AVAILABILITY
Task 1 → AZ us-east-1a
Task 2 → AZ us-east-1b
Task 3 → AZ us-east-1c
AZ failure → only 1/3 tasks affected!
```

### 2. BINPACK — Maximize Density
```
Goal: Pack as many tasks as possible on one instance before using another
Fields: cpu | memory

Best for: COST OPTIMIZATION
Instance A: [Task1 Task2 Task3 Task4 Task5] ← PACKED
Instance B: [Task6 Task7...]
Instance C: Empty → CAN BE TERMINATED (save cost!)
```

### 3. RANDOM — Place Anywhere
```
Goal: No optimization, place wherever capacity exists
Use when: Distribution does not matter for your workload
Rarely used in production (spread or binpack are almost always preferable)
```

### Combining Strategies (Best Practice):
```bash
# Best practice: AZ spread FIRST, then binpack within AZ
--placement-strategy \
  type=spread,field=attribute:ecs.availability-zone \
  type=binpack,field=memory
```

Strategies are applied in order. Spread across AZs first ensures high availability. Binpack within each AZ then maximizes cost efficiency by filling instances before provisioning new ones. This combination gives you both HA and cost control simultaneously.

---

## Placement Constraints

### Type 1: distinctInstance
```bash
# Each task on a DIFFERENT EC2 instance
--placement-constraints type=distinctInstance

Use case:
  "No 2 replicas of payment-service on same EC2"
  → Single instance failure affects only 1 task
  → Prevents correlated failures from wiping all replicas at once
```

### Type 2: memberOf (Cluster Query Language)
```bash
# Advanced filtering expression

# Only instances of a specific type:
--placement-constraints \
  type=memberOf,expression="attribute:ecs.instance-type == c5.4xlarge"

# Only instances with a custom attribute (e.g., SSD storage):
--placement-constraints \
  type=memberOf,expression="attribute:storage-type == ssd"

# Only instances in specific AZs:
--placement-constraints \
  type=memberOf,expression="attribute:ecs.availability-zone in [us-east-1a, us-east-1b]"

# Combined expression:
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

Custom attributes are powerful for workload segmentation. You can label specific instances with attributes like `security-hardened=true`, `gpu=true`, or `tier=premium`, and then use `memberOf` constraints to pin services to the appropriate hosts without relying solely on instance type.

---

## AZ Balancing

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

AZ rebalancing is automatic when the spread strategy is configured. This is important to verify after scale-out events, because rapid scaling may temporarily create AZ imbalance. ECS does not continuously rebalance existing tasks (it only applies the strategy when placing new tasks), so after a scale-in event you may need to manually trigger a redeployment to restore even distribution.

---

## Capacity Provider Logic (Advanced Scheduling)

```
Cluster has Capacity Providers:
  FARGATE (weight=1, base=2)
  FARGATE_SPOT (weight=4)

Service desired-count=10:

Calculation:
  Base FARGATE tasks: 2 (always guaranteed on regular Fargate)
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
  If Spot tasks are interrupted: Service still runs on 4 Fargate tasks
  → Service scales back up to 10 from remaining regular Fargate capacity
```

---

## Analogy — Theater Seat Assignment

**ECS Scheduler = Theater Usher:**

- **Spread strategy:** "Distribute the audience across all rows evenly — do not fill row 1 before opening row 2"
- **Binpack:** "Fill each row completely before moving to the next (maximize row utilization)"
- **Placement constraints:** "VIP section is reserved for premium ticket holders only"
- **distinctInstance:** "No two members of the same group in the same row — if that row has an emergency, the entire group is not affected"
- **AZ balance:** "Distribute the audience evenly between the left section and right section of the theater"

---

## Real-World Scenario

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
    minimumHealthyPercent: 100   (never reduce below 6 tasks during deployment)
    maximumPercent: 150          (up to 9 tasks allowed during deployment)

  Result:
    AZ-a: Instance #1 [Task1], Instance #2 [Task2]
    AZ-b: Instance #3 [Task3], Instance #4 [Task4]
    AZ-c: Instance #5 [Task5], Instance #6 [Task6]

    EC2 failure (Instance #3): Task3 stops
    Scheduler: "Where can I place?
      distinctInstance: not on #1, #2, #4, #5, #6
      AZ balance: AZ-b has 1, others have 2 → prefer AZ-b
      security-hardened: must be tagged"
    → Finds Instance #7 (security-hardened, AZ-b) → places Task7
    → Balance restored!
```

---

## Hands-On Examples

### Placement Strategy Examples:
```bash
# High availability pattern: AZ spread + memory binpack within AZ
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
# Check if any hosts match the constraint:
aws ecs list-container-instances \
  --cluster production \
  --filter "attribute:security-hardened == true"

# 3. No instances in required AZ
aws ecs list-container-instances \
  --cluster production \
  --filter "attribute:ecs.availability-zone == us-east-1c"
```

---

## Gotchas & Edge Cases

### 1. Binpack Can Cause AZ Imbalance
```
Binpack fills one instance before another.
If all instances with available capacity happen to be in AZ-a:
  → All new tasks go to AZ-a → single-AZ deployment risk!

Fix: Always use spread (AZ) first, then binpack within each AZ.
placement-strategy: spread(AZ) → binpack(memory)

This is the most common scheduling misconfiguration in production ECS clusters.
```

### 2. distinctInstance Does Not Apply to Fargate
```
distinctInstance is an EC2-only concept.
Fargate does not have "instances" — each task IS its own isolated VM.
Do not apply distinctInstance with Fargate (it is a no-op but adds confusion
when reading service configurations).
```

### 3. minimumHealthyPercent During Deployments
```
minimumHealthyPercent=100 + maximumPercent=200:
  → Deploy 6 new tasks → wait for all to pass health checks → stop 6 old tasks
  → Zero-downtime blue/green-style deployment
  → Requires 2x compute capacity temporarily

minimumHealthyPercent=50 + maximumPercent=150:
  → Stop 3 old tasks → start 3 new tasks → wait → repeat
  → Brief capacity reduction during deployment (acceptable for most services)
  → Lower compute cost during deployment

Choose 100/200 for critical services where no capacity reduction is tolerable.
Choose 50/150 for services where brief reduced capacity is acceptable.
```

### 4. Scheduler Does Not Rebalance Existing Running Tasks
```
ECS applies placement strategy only when placing NEW tasks.
Existing tasks are NOT moved after placement.

This means after a scale-in event that removes tasks from AZ-b,
the remaining tasks may be unevenly distributed across AZs.

To rebalance:
  aws ecs update-service --cluster prod --service myapp --force-new-deployment

This triggers a rolling replacement that places each new task
according to the spread strategy, restoring AZ balance.
```

### 5. "Unable to Place Task" on Fargate
```
"Unable to place task" errors on Fargate are NOT resource exhaustion errors
(Fargate has effectively unlimited capacity).

Common causes for Fargate placement failures:
  - Invalid CPU/memory combination (e.g., 256 CPU / 4096 MB — not a valid Fargate pair)
  - Subnet has no available IP addresses
  - Security group ID does not exist
  - Fargate Spot capacity temporarily exhausted in the specified AZs
    (solution: add a regular Fargate fallback in the capacity provider strategy)
```

---

## Interview Questions

**Q: "How does the ECS scheduler make placement decisions?"**

> The scheduler works in three sequential phases:
> 1. Eligible host filtering: Disconnected, draining, or capacity-exhausted instances are eliminated.
> 2. Constraint filtering: Hosts that violate distinctInstance or memberOf expressions are eliminated.
> 3. Strategy application: The optimal host is selected from the remaining candidates using spread, binpack, or random.
> Constraints are non-negotiable eliminators. Strategy is an optimizer that works on whatever candidates survive the constraint phase. If no hosts survive the constraint phase, the task cannot be placed.

**Q: "What is a Daemon service and when do you use it?"**

> A Daemon service runs exactly one task on every container instance in the cluster.
> When a new instance joins the cluster, ECS automatically starts the daemon task on it.
> When an instance is terminated, the daemon task is automatically removed.
> Use cases include log agents (Fluentd, FluentBit), monitoring exporters (Datadog, CloudWatch agent), and security scanners.
> Daemon services are EC2-only. Fargate has no "instances" to attach a daemon to — each Fargate task is already an isolated VM.

---

*Next: [06_ECS_Security_Model.md →](./06_ECS_Security_Model.md)*
