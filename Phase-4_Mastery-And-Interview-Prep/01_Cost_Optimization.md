# 💰 Cost Optimization — Complete Guide

---

## 📖 Cost Formula

```
Total AWS Container Cost =
  ECR Storage + 
  ECR Data Transfer + 
  ECS Compute (Fargate vCPU/Memory OR EC2 instances) + 
  ALB + 
  CloudWatch Logs/Metrics +
  Secrets Manager +
  Data Transfer (Cross-AZ, Cross-Region)
```

---

## 💡 Optimization Strategy 1: Right-Size Your Containers

### Container CPU/Memory Analysis:
```bash
# Find over-provisioned containers
# Container Insights query (CloudWatch Logs Insights):
fields @timestamp, ServiceName, CPUUtilized, CPUReserved, MemoryUtilized, MemoryReserved
| filter ClusterName = "production" 
| stats avg(CPUUtilized/CPUReserved*100) as avgCPUPercent,
        avg(MemoryUtilized/MemoryReserved*100) as avgMemPercent
  by ServiceName
| sort avgCPUPercent asc
```

### Right-Sizing Example:
```
Task definition: 2 vCPU, 8GB RAM
Actual usage: avg 0.3 vCPU, 1.5 GB RAM

Overprovisioned by: 85% CPU, 81% Memory!

Fargate cost (current): $0.04048×2×720 = $58.29/month for vCPU alone
Fargate cost (right-sized: 0.5 vCPU, 2GB): $0.04048×0.5×720 = $14.57 + $0.004445×2×720 = $6.40
Total right-sized: $20.97/month

Savings per task: $37.32/month (64% savings!)
For 20 tasks: $746/month savings
```

---

## 💡 Optimization Strategy 2: Fargate Spot for Non-Critical Workloads

```python
# Business impact assessment for Fargate Spot eligibility:
# 
# CAN use Fargate Spot:
#   ✅ Batch processing (retryable)
#   ✅ Development/staging environments
#   ✅ Background workers (idempotent)
#   ✅ ML training jobs
#   ✅ Report generation (can retry)
#
# CANNOT use Fargate Spot:
#   ❌ Payment processing (in-flight transactions)
#   ❌ Real-time user-facing APIs (latency sensitive on interruption)
#   ❌ Stateful services without graceful shutdown

# Hybrid strategy:
capacity_provider_strategy = [
    {"capacityProvider": "FARGATE", "weight": 1, "base": 2},      # 40% × 2 = base
    {"capacityProvider": "FARGATE_SPOT", "weight": 4, "base": 0}  # 60% Spot
]
# Result: 40% regular Fargate cost, 60% Spot (70% cheaper each)
# Average savings: 60% × 70% = 42% total cost reduction!
```

---

## 💡 Optimization Strategy 3: Savings Plans / Reserved Instances

### EC2 Mode — Reserved Instances:
```
On-Demand c5.2xlarge: $0.34/hr = $2,440/year
Reserved  c5.2xlarge (1-year, All Upfront): $1,373/year
                                             = 44% savings!

Reserved  c5.2xlarge (3-year, All Upfront): $850/year
                                             = 65% savings!

Compute Savings Plans (more flexible):
  - Commit to $/hour of compute usage
  - 50-66% discount vs On-Demand
  - Works across instance types AND Fargate!
  - Example: Commit $1/hour Compute SP
    → Automatically applies to EC2 + Fargate usage
```

### Compute Savings Plan for Fargate:
```bash
# Calculate monthly Fargate spend:
# Check Cost Explorer → service = Fargate → last 3 months average

# Purchase Compute Savings Plan:
# AWS Cost Explorer → Savings Plans → Recommendations
# Or manually:
aws savingsplans create-savings-plan \
  --savings-plan-offering-id <id-from-catalog> \
  --commitment 2.0 \           # $2/hour committed
  --purchase-time "2024-01-01T00:00:00Z"

# Fargate Savings Plan gives 20-52% discount
# Best for: Consistent Fargate usage (not spiky)
```

---

## 💡 Optimization Strategy 4: ECR Cost Reduction

### Automated Cleanup Script:
```python
import boto3
from datetime import datetime, timedelta

ecr = boto3.client('ecr')

def cleanup_ecr_images(repo_name, keep_count=10, keep_prefixes=['prod-', 'release-']):
    """Keep only recent images and important tags"""
    
    # Get all images
    response = ecr.describe_images(repositoryName=repo_name)
    images = response['imageDetails']
    
    # Sort by push date (newest first)
    images.sort(key=lambda x: x['imagePushedAt'], reverse=True)
    
    # Keep: specified prefixes + last keep_count
    to_delete = []
    kept_count = 0
    
    for img in images:
        tags = img.get('imageTags', [])
        keep = False
        
        # Keep images with important prefix tags
        for prefix in keep_prefixes:
            if any(t.startswith(prefix) for t in tags):
                keep = True
                break
        
        # Keep last N images regardless
        if kept_count < keep_count:
            keep = True
            kept_count += 1
        
        if not keep:
            to_delete.append({'imageDigest': img['imageDigest']})
    
    if to_delete:
        ecr.batch_delete_image(repositoryName=repo_name, imageIds=to_delete)
        print(f"Deleted {len(to_delete)} images from {repo_name}")
    
    return len(to_delete)
```

---

## 💡 Optimization Strategy 5: Reduce Data Transfer Costs

```
Data Transfer Costs (The Hidden Budget Killer):

Cross-AZ data transfer: $0.01/GB
Cross-Region data transfer: $0.02-0.09/GB
Internet egress (ECR to public): $0.09/GB first 10TB

Optimizations:

1. Same-AZ task placement for services that talk to each other:
   user-service → user-db: Both in us-east-1a → FREE transfer
   (vs user-service in 1a, user-db in 1b → $0.01/GB!)
   
   Tradeoff: Less resilience vs less cost
   For dev/staging: Use single-AZ
   For production: Multi-AZ resilience worth the cost

2. ECR VPC Endpoint:
   Without: ECR pull → NAT Gateway → $0.045/GB processing
   With VPC endpoint: $0.01/GB → 78% savings!
   
3. CloudWatch Logs: Use compression
   FireLens: compress output before sending (configurable in Fluent Bit)
   50-70% reduction in log data transfer

4. S3 for large files (not ECS env vars or task definitions):
   Large binary → S3 → presigned URL → container downloads directly
   No data flows through ECS/ALB
```

---

## 📊 Complete Monthly Cost Estimate Template

```
# Template for cost estimation:

ECR:
  Storage: ___ GB × $0.10 = $___
  Data transfer: ___ GB × rate = $___
  Enhanced scanning: ___ images × $0.09 = $___
  Subtotal: $___

ECS Compute (Fargate):
  Always-on tasks: ___ vCPU × $0.04048 × 720hr + ___ GB × $0.004445 × 720hr = $___
  Spot tasks: × 0.3 (70% discount) = $___
  Subtotal: $___

ECS Compute (EC2):
  Instance type × hours × On-Demand rate = $___
  (After Reserved/SP discount): $___
  Subtotal: $___

Networking:
  ALB: $0.008/LCU/hr + $0.0008/hour = $___
  NAT Gateway (if applicable): $0.045/hr + $0.045/GB = $___
  VPC Endpoints: $0.01/GB × ___ GB = $___
  Subtotal: $___

Observability:
  CloudWatch Logs: $0.50/GB ingested × ___ GB = $___
  Container Insights: $0.0075/GB × ___ GB = $___
  X-Ray: $5/million traces × ___ traces = $___
  Subtotal: $___

GRAND TOTAL: $___
```

---

## 🎤 Interview Angle

**Q: "Fargate costs kaise optimize karein? EC2 kab better hai?"**

> Fargate optimization:
> 1. Right-size CPU/Memory — Container Insights se actual usage dekho, then reduce by 20-30% buffer.
> 2. Fargate Spot for batch/non-critical workloads — 70% savings.
> 3. Compute Savings Plans — 20-52% savings on committed Fargate usage.
>
> EC2 kab better: Predictable 24/7 load + Reserved Instances = 44-65% savings vs Fargate.
> Calculation: Monthly Fargate = $X. If EC2 Reserved < $X → use EC2.
> But factor in: Ops cost (EC2 patching, management) vs Fargate (zero ops).

---

*Next: [02_Performance_Engineering.md →](./02_Performance_Engineering.md)*
