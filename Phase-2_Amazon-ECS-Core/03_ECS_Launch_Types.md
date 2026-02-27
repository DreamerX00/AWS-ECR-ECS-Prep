# ECS Launch Types — Fargate vs EC2

---

## Concept Explanation

Launch Type = "Where does my container actually run?"

```
ECS Service
   │
   ├── FARGATE        → AWS manages the servers (serverless)
   │
   └── EC2            → You manage the servers (more control)
```

---

## FARGATE — Serverless Containers

### What is Fargate?
```
"I want to run a container. I don't care about servers."

With Fargate:
✅ No EC2 instance to launch or manage
✅ No Docker daemon to configure
✅ No capacity planning (per cluster)
✅ Per-task billing (second-level granularity)
✅ Automatic multi-AZ distribution
✅ ENI per task (maximum network isolation)
```

### How Fargate Works Internally:
```
Your ECS Service says: "Run task myapp:v5, 0.5 vCPU, 1GB RAM"
          │
          ▼
ECS Scheduler → Fargate Capacity Provider
          │
          ▼
AWS provisions a Micro-VM (kata containers / firecracker based)
  - Dedicated virtual machine for YOUR task
  - No other customer's containers on same VM
  - VM is destroyed when task ends
          │
          ▼
Your container runs inside this MicroVM
  - Full isolation from other tenants
  - No shared kernel exploits possible
          │
          ▼
Task ends → VM destroyed → Billing stops (to the second!)
```

### Fargate Pricing:
```
vCPU:   $0.04048 per vCPU per hour
Memory: $0.004445 per GB per hour

Example: 0.5 vCPU, 1GB RAM for 1 month (720 hours):
  vCPU cost:   0.5 × $0.04048 × 720 = $14.57
  Memory cost: 1 × $0.004445 × 720  = $3.20
  Total:       $17.77/month per task

  3 tasks (desired count=3) × $17.77 = $53.31/month
```

### Fargate Spot — 70% Cheaper!
```
Fargate Spot = Fargate with interruption risk
- Uses AWS spare capacity
- 70% cheaper than regular Fargate
- AWS can interrupt with 2-minute warning → SIGTERM → SIGKILL
- Perfect for: batch jobs, fault-tolerant workloads, ML training

Capacity Provider Strategy with Spot:
  - Base 1 always on regular Fargate (for reliability)
  - Remaining tasks on Fargate Spot (for cost savings)
```

```bash
# Fargate Spot strategy:
aws ecs create-service \
  --cluster production \
  --service-name batch-processor \
  --capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=4 \
  ...
```

---

## EC2 LAUNCH TYPE — Full Control

### What is EC2 Mode?
```
"I want to run containers on MY EC2 instances."

With EC2 mode:
✅ You provision EC2 instances (in Auto Scaling Group)
✅ ECS Container Agent runs on each EC2
✅ Container Agent receives tasks from ECS Control Plane
✅ More control over instance types and placement
✅ Can use:
   - GPU instances (p3, p4, g4, g5)
   - Custom AMIs
   - Specific instance families (c7g, m7g for Graviton)
   - Dedicated hosts (HIPAA/compliance)
   - Daemon containers (one per instance)
```

### EC2 ECS Cluster Setup:
```bash
# Launch EC2 instances with ECS-optimized AMI
# (Container Agent pre-installed)

# Find latest ECS-optimized AMI
aws ssm get-parameter \
  --name /aws/service/ecs/optimized-ami/amazon-linux-2/recommended \
  --query "Parameter.Value" | jq -r . | jq '.image_id'

# User Data to join ECS cluster:
#!/bin/bash
echo ECS_CLUSTER=production >> /etc/ecs/ecs.config
echo ECS_ENABLE_CONTAINER_METADATA=true >> /etc/ecs/ecs.config
echo ECS_ENABLE_TASK_IAM_ROLE=true >> /etc/ecs/ecs.config

# Create Auto Scaling Group with this launch template
# Then create ECS Capacity Provider linked to this ASG
```

### EC2 ECS Container Agent Config:
```bash
# /etc/ecs/ecs.config — important settings
ECS_CLUSTER=production
ECS_ENGINE_TASK_CLEANUP_WAIT_DURATION=3h      # Clean stopped task containers after 3h
ECS_IMAGE_PULL_BEHAVIOR=prefer-cached         # Use cached image if available
ECS_NUM_IMAGES_DELETE_PER_CYCLE=10           # Images to delete per GC cycle
ECS_RESERVED_MEMORY=256                       # Reserve 256MB for agent/OS on host
ECS_ENABLE_CONTAINER_METADATA=true
ECS_AVAILABLE_LOGGING_DRIVERS=["awslogs","fluentd","syslog"]
```

### EC2 Exclusive Features:
```
1. GPU instances:
   Task definition: {"resourceRequirements": [{"type": "GPU", "value": "1"}]}
   → ECS schedules on GPU-enabled EC2 instance automatically

2. Daemon Service:
   → One task per EC2 instance (monitoring, log collection)

3. Custom AMIs:
   → Bring your own pre-warmed images, security-hardened OS

4. Windows Containers:
   → Windows Server on EC2 (Fargate has Windows Fargate too, but limited)

5. Placement Constraints:
   → "Only run on instance type c5.4xlarge"
   → "Don't put 2 tasks of same service on same instance"
```

---

## Fargate vs EC2 — Decision Matrix

| Criteria | Fargate | EC2 |
|----------|---------|-----|
| **Ops Overhead** | Very Low | High (patch, maintain instances) |
| **Cost (continuous load)** | Higher | Lower (Reserved/Savings Plan) |
| **Cost (spiky/bursty)** | Lower | Higher (idle instances) |
| **GPU workloads** | Not supported | Supported |
| **Daemon containers** | Not supported | Supported |
| **Container density control** | AWS manages | Full control |
| **Start time** | Slightly slower (VM provision) | Faster (if instance warm) |
| **Security isolation** | MicroVM per task | Shared kernel (namespace isolation) |
| **Compliance (HIPAA)** | Supported | Supported |
| **Custom instance types** | Memory/CPU combos only | Any EC2 family |
| **Spot pricing** | FARGATE_SPOT (70% off) | EC2 Spot (up to 90% off) |

---

## Analogy — Co-Working Space vs Dedicated Office

**Fargate = WeWork / Co-Working Space:**
- "I need a desk tomorrow → Book it → Use it → Leave"
- No lease, no maintenance, no security staff to hire
- Pay per hour you sit
- Cannot permanently customize the desk configuration
- Perfect for: startups, variable load, teams without dedicated infrastructure engineers

**EC2 = Your Own Office:**
- Buy or rent dedicated space → set up desks → hire security → more control
- Fixed cost regardless of utilization
- Need a GPU workstation? Add it yourself
- Perfect for: predictable workload, special hardware requirements, large-scale cost optimization with Reserved Instances

---

## Real-World Scenarios

### Scenario 1: Startup — Correct Choice is Fargate
```
Startup "InstantAI":
  - 3 developers, no dedicated DevOps engineer
  - Traffic: 100-500 req/sec (variable)
  - No GPU needed
  - Budget: $2K/month

Decision: Fargate
Reason:
  - No EC2 patching or management overhead
  - Pay only when requests are being served
  - Team focuses on product, not infrastructure
  - Auto-scales without manual capacity planning

Cost:
  - 2 tasks always running: 2 × $18 = $36/month
  - Scale to 10 during peak: $180/month peak
  - Average: ~$80/month → well within budget
```

### Scenario 2: ML Company — EC2 is the Right Choice
```
MLCo:
  - ML inference service requiring GPU
  - 4 NVIDIA A10G GPUs needed
  - 24/7 continuous load
  - Aggressive cost optimization needed

Decision: EC2 with GPU instances
Reason:
  - Fargate does not support GPU
  - p3.8xlarge (4 NVIDIA V100): $12.24/hr on-demand
  - Reserved Instance (3-year): $3.18/hr (74% savings!)
  - Dedicated host available for compliance requirements

EC2 ECS setup:
  - g5.12xlarge instances in Capacity Provider
  - Task def: {"resourceRequirements": [{"type": "GPU", "value": "4"}]}
```

### Scenario 3: Hybrid Strategy (Most Common in Production)
```
"HealthApp" Production:
  API services:      FARGATE (stateless, variable load)
  ML inference:      EC2 (GPU instances)
  Batch jobs:        FARGATE_SPOT (70% cheaper, interruptions are acceptable)
  Log collectors:    EC2 Daemon service (one per instance)
  Monitoring agent:  EC2 Daemon service

One ECS Cluster, multiple Capacity Providers — each service selects its own provider!
```

---

## Hands-On Examples

### Fargate Task Launch:
```bash
aws ecs run-task \
  --cluster production \
  --launch-type FARGATE \
  --task-definition myapp:5 \
  --count 1 \
  --platform-version "1.4.0" \  # Latest Fargate platform
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-abc123", "subnet-def456"],
      "securityGroups": ["sg-myapp-tasks"],
      "assignPublicIp": "DISABLED"
    }
  }'
```

### EC2 Task Launch with Placement Strategy:
```bash
aws ecs run-task \
  --cluster production \
  --launch-type EC2 \
  --task-definition myapp:5 \
  --placement-strategy \
    type=spread,field=attribute:ecs.availability-zone \
    type=binpack,field=memory \
  --placement-constraints \
    type=memberOf,expression="attribute:ecs.instance-type == c5.4xlarge"
```

### Fargate Spot — Handling Interruption Gracefully:
```python
# In your app: handle Spot interruption gracefully
import requests
import signal
import threading

def check_spot_interruption():
    """Poll EC2 instance metadata for Fargate Spot interruption notice"""
    # Fargate Spot: check task metadata endpoint
    while True:
        try:
            resp = requests.get(
                'http://169.254.170.2/v3/metadata/task',
                timeout=1
            )
            # Check if "SPOT_INTERRUPTION" in stop reason
        except:
            pass
        time.sleep(5)

# Start background check
threading.Thread(target=check_spot_interruption, daemon=True).start()

def graceful_shutdown(signum, frame):
    print("SIGTERM received - gracefully shutting down...")
    # Finish in-flight requests
    # Checkpoint state
    # Close connections
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

---

## Gotchas & Edge Cases

### 1. Fargate Platform Version Matters
```
Fargate Platform Versions: 1.3.0, 1.4.0, LATEST
1.4.0 supports:
  ✅ Ephemeral storage up to 200GB (per task)
  ✅ EFS volumes
  ✅ Task-level network interface logging

LATEST = current stable (typically 1.4.0 now)
Specify "LATEST" in service definition for automatic updates:
aws ecs update-service --platform-version LATEST ...
```

### 2. EC2 Mode — Container Agent Must Be Running
```
If an EC2 instance's Container Agent stops → no new tasks are accepted
The instance appears ACTIVE in EC2 but ECS will not schedule tasks on it

Check agent connection status:
aws ecs list-container-instances \
  --cluster production \
  --filter "agentConnected==true"

Reconnect the agent:
sudo systemctl restart ecs
```

### 3. Fargate Ephemeral Storage
```
Default: 20GB ephemeral storage per task
Max: 200GB (requires Fargate 1.4.0)
Cost: $0.04/GB/month beyond 20GB

For large-data tasks: Use EFS or S3 instead of ephemeral storage
Ephemeral storage is LOST when the task stops — it is not a persistent volume!
```

### 4. Fargate Does Not Support All EC2 Instance Capabilities
```
The following are NOT available in Fargate:
  - GPU resources
  - Custom AMIs / OS configurations
  - Host networking mode
  - Privileged containers
  - Daemon scheduling
  - Placement constraints beyond CPU/memory

If any of these are required, EC2 launch type is mandatory.
```

### 5. EC2 Mode — Over-Provisioning Risk
```
EC2 clusters must be monitored for idle capacity:
  - Instances with no tasks running still incur hourly charges
  - Enable Managed Scaling on capacity providers to let ECS scale the ASG
  - Set target capacity to 70-80% to balance cost vs. burst headroom
  - Use Graviton instances (m7g, c7g) for 20-40% cost savings vs. x86
    on compatible workloads
```

---

## Interview Questions

**Q: "When do you choose Fargate over EC2? What are the trade-offs?"**

> Fargate is the right choice when you want minimal operational overhead, have variable load, do not need GPU, and want to reach market faster without infrastructure management.
> EC2 is the right choice when you need GPU instances, daemon containers, aggressively cost-optimized deployments with Reserved Instances, or custom AMIs and OS configurations.
> Most teams should start with Fargate and only add EC2 capacity providers for specific workloads that require it.
> The hybrid approach — API on Fargate, ML on EC2 GPU, batch on Fargate Spot — is the most common production pattern at scale.

**Q: "How does Fargate achieve security isolation between customers?"**

> Fargate provisions a dedicated MicroVM for each task using AWS Firecracker (a lightweight VMM built on KVM).
> No container runtime is shared between different customers' tasks.
> Each MicroVM has its own kernel, and the VM is destroyed when the task completes.
> In contrast, EC2 mode uses Linux namespaces and cgroups to isolate containers — multiple customer workloads could theoretically share the same kernel if AWS's multi-tenancy were not designed correctly.
> Fargate's stronger isolation makes it the preferred choice for PCI DSS and HIPAA workloads where kernel-level isolation is a compliance requirement.

---

*Next: [04_ECS_Internal_Working.md →](./04_ECS_Internal_Working.md)*
