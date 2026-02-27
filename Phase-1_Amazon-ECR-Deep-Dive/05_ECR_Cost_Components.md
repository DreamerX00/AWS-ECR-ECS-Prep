# ECR Cost Components — Every Penny Counts

---

## Concept Explanation

Understanding ECR costs is important because uncontrolled image storage and cross-region pulls can result in real, avoidable expenses. The three main cost drivers are:

1. **Storage** — Storing image data
2. **Data Transfer** — Pushing and pulling images
3. **Scanning** — Enhanced scanning charges (AWS Inspector)

---

## Pricing Breakdown

### 1. Storage

| Type | Price | Notes |
|------|-------|-------|
| Private ECR | **$0.10/GB/month** | After 500MB free tier |
| Public ECR | **Free** | For storage under 50GB |
| Free Tier | **500MB/month** | Per account for Private ECR (across all regions) |

> Storage charges are based on **compressed** size, not the uncompressed size shown by `docker images`.

### 2. Data Transfer

| Transfer Type | Price |
|---------------|-------|
| Same region ECR → ECS | **Free** (within the same region) |
| Cross-region | **$0.02–0.09/GB** (varies by region pair) |
| ECR → Internet | **$0.09/GB** for the first 10TB |
| ECR Public → Internet (non-AWS) | **Free** up to 500GB/month, then standard rates |
| ECR Public → AWS services | **Free** always |

### 3. Enhanced Scanning (AWS Inspector)

| Item | Price |
|------|-------|
| Container image scanning | **$0.09/image/month** per repository |
| Initial scan | Included in Inspector base pricing |

---

## Cost Calculation — Real Examples

### Example 1: Single Service Startup

```
Service: Node.js e-commerce backend
Image size: 800MB uncompressed → ~250MB compressed in ECR
Versions retained: 20 (with lifecycle policy)
Region: us-east-1
ECS deployment: Same region
Enhanced scanning: Enabled

STORAGE COST:
  250MB × 20 versions = 5,000MB = 5GB total
  After 500MB free: 4.5GB × $0.10 = $0.45/month

DATA TRANSFER COST:
  Same-region pull: FREE
  ECS pulls only changed layers on each deploy (delta pull)
  Average delta per deploy: 50MB
  Deploys per month: 30
  Total data transferred: 30 × 50MB = 1.5GB ← FREE (same region)

SCANNING COST:
  Basic scanning: FREE
  Enhanced scanning: $0.09/image/month × 20 images = $1.80/month

TOTAL: ~$2.25/month — very cost-effective
```

### Example 2: Enterprise at Scale

```
Company: 50 microservices
Average image size: 300MB compressed
Versions per service: 30
Teams: Engineering in US (us-east-1), APAC team in ap-south-1

STORAGE COST (us-east-1):
  50 services × 30 versions × 300MB = 450GB
  Free tier: 500MB (negligible at this scale)
  450GB × $0.10 = $45/month

DATA TRANSFER COST (the hidden cost driver):
  APAC ECS in ap-south-1 pulling from us-east-1:
  20 services × 10 deploys/month × 300MB pull
  = 60GB cross-region transfer
  60GB × $0.02 = $1.20/month (manageable)

  On a mass deployment day (all services at once):
  50 services × 300MB = 15GB in one burst
  = $0.30 per deployment event × 10 deployments/month = $3/month cross-region

SCANNING COST:
  50 × 30 = 1,500 image-months × $0.09 = $135/month!

TOTAL: ~$183/month

OPTIMIZED (with lifecycle policy → keep only 5 versions):
  Storage: 50 × 5 × 300MB = 75GB × $0.10 = $7.50/month
  Scanning: 50 × 5 images × $0.09 = $22.50/month
  TOTAL: ~$31/month
  SAVINGS: 83% reduction — achieved just by setting a lifecycle policy
```

---

## Analogy — Streaming Video Storage

**ECR image storage works like a video streaming platform storing shows:**

- A show is stored once, but a million users can stream it — storage is paid once.
- The same show exists in multiple quality tiers (4K/HD/SD) that share common video segments.
- Old shows (lifecycle policy) are eventually removed to free up storage space.

**Layer deduplication:**
- A Node.js base image (100MB) is like a common opening sequence shared across many shows.
- Each service only pays for its unique layers on top of the shared base.

**Cross-region transfer:**
- A user in Mumbai streaming from a US server incurs bandwidth costs.
- The platform's solution is edge caching — ECR's equivalent is **cross-region replication + Pull-Through Cache**.

---

## Real-World Cost Optimization Strategies

### Strategy 1: Aggressive Lifecycle Policy
```bash
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 3 production images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["prod-", "release-"],
          "countType": "imageCountMoreThan",
          "countNumber": 3
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "Keep last 10 tagged images (any tag)",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": [""],
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 3,
        "description": "Delete untagged images within 1 hour",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "hours",
          "countNumber": 1
        },
        "action": {"type": "expire"}
      }
    ]
  }'
```

### Strategy 2: Minimize Image Size with Multi-Stage Builds
```dockerfile
# BEFORE: 1.2GB image
FROM node:18
COPY . .
RUN npm install

# AFTER: 180MB image (85% smaller!)
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY ./src ./src
USER node
CMD ["node", "src/index.js"]

# Storage cost comparison: 1.2GB vs 0.18GB
# ECR cost: $0.12/month vs $0.018/month per version
# ECS task start time: 45 seconds vs 8 seconds
```

### Strategy 3: VPC Endpoint to Eliminate NAT Costs
```bash
# Without VPC endpoint: image pulls from a private subnet go through NAT Gateway.
# NAT Gateway: $0.045/hr + $0.045/GB data processing
# 100GB/month pulled → $4.50/month NAT cost just for image pulls!

# WITH ECR VPC endpoint: direct private network connection.
# VPC Interface Endpoint: ~$0.01/GB — significantly cheaper than NAT.

# Create the ECR DKR endpoint (for image layer pulls):
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface

# Create the ECR API endpoint (for auth and metadata calls):
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --vpc-endpoint-type Interface

# Also create an S3 Gateway endpoint — ECR layers are stored in S3!
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway

# After setup: ECR pulls use private networking, no NAT, lower cost.
```

### Strategy 4: Pull-Through Cache for Public Images
```bash
# Problem: DockerHub → NAT Gateway → ECS (with Docker Hub rate limits!)
# Solution: DockerHub → ECR Pull-Through Cache → VPC → ECS

# Setup:
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "docker-hub" \
  --upstream-registry-url "registry-1.docker.io"

aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "quay" \
  --upstream-registry-url "quay.io"

# Usage: instead of pulling directly from Docker Hub:
# docker pull nginx:latest
# Use the ECR-cached version:
# docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:latest

# Benefits:
# ✅ No Docker Hub rate limits
# ✅ Layer deduplication between public and private images
# ✅ VPC access — no NAT Gateway needed
# ✅ Enhanced scanning applies to cached images too
```

---

## Hands-On Cost Monitoring

### Calculate Your Current Storage Costs:
```bash
# Total storage across ALL repositories
aws ecr describe-repositories | \
  jq -r '.repositories[].repositoryName' | \
  while read repo; do
    aws ecr describe-images --repository-name "$repo" \
      --query 'sum(imageDetails[*].imageSizeInBytes)' \
      --output text 2>/dev/null
  done | \
  awk '{sum+=$1} END {printf "Total: %.2f GB\n", sum/1024/1024/1024}'

# Per-repository image count
aws ecr describe-repositories \
  --query 'repositories[*].repositoryName' \
  --output text | tr '\t' '\n' | \
  while read repo; do
    COUNT=$(aws ecr list-images --repository-name "$repo" \
      --query 'length(imageIds)' --output text)
    echo "$repo: $COUNT images"
  done
```

### Cost Alerts (CloudWatch):
```bash
# Alert when ECR monthly costs exceed $50
aws cloudwatch put-metric-alarm \
  --alarm-name "ECR-High-Storage-Cost" \
  --alarm-description "ECR storage exceeding budget" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 86400 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ServiceName,Value=AmazonECR \
              Name=Currency,Value=USD \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123:billing-alerts
```

---

## Gotchas & Edge Cases

### 1. Lifecycle Policy Evaluation Is Asynchronous
```
Setting a lifecycle policy does not delete images immediately.
ECR evaluates lifecycle policies asynchronously — deletions may happen hours later.

Before applying a policy in production, use the dry-run preview:
aws ecr start-lifecycle-policy-preview \
  --repository-name myapp \
  --lifecycle-policy-text file://my-policy.json

aws ecr get-lifecycle-policy-preview \
  --repository-name myapp
# Shows which images WOULD be deleted without actually deleting anything.
```

### 2. Free Tier Is Per Account, Not Per Region
```
The 500MB free tier applies to total private ECR storage across ALL regions in the account.
us-east-1: 400MB + ap-south-1: 200MB = 600MB total
Charged amount: (600 - 500) × $0.10 = $0.01/month

If you have ECR repositories in multiple regions, all storage counts against
the single 500MB free tier budget.
```

### 3. Deleted Images Are Not Immediately Reflected in Billing
```
ECR billing is based on a daily storage snapshot.
Deleting an image today reduces your bill starting from tomorrow's snapshot.
Cost savings from deletions do not take effect on the same day.
```

### 4. Enhanced Scanning Charges per Repository, Not per Digest
```
If the same image digest exists in two different repositories,
you are charged twice for Enhanced Scanning — once per repository.

Where possible, consolidate images into fewer repositories
to reduce redundant scanning costs.
```

### 5. Data Transfer to the Internet Is Charged Even for Accidental Public Exposure
```
If an ECR repository policy is misconfigured to allow public reads,
and someone pulls the image over the internet, you are billed for
data transfer out at $0.09/GB.

Always restrict repository policies to specific IAM principals.
Regularly audit repository policies with:
aws ecr get-repository-policy --repository-name myapp
```

---

## Interview Questions & Answers

**Q: "What strategies would you use to control ECR costs?"**

> Four main strategies:
> 1. **Lifecycle policies** — Automatically delete old images. This has the biggest impact on storage and scanning costs. Keeping only 3-5 versions instead of 30+ typically reduces costs by 70-80%.
> 2. **Image size optimization** — Multi-stage builds and Alpine base images typically reduce image size by 60-85%, which reduces both storage cost and ECS task startup time.
> 3. **VPC Endpoints** — Eliminate NAT Gateway data processing charges for ECR pulls from private subnets. For high-throughput workloads, this alone can save tens of dollars per month.
> 4. **Pull-Through Cache** — Cache public registry images in ECR to avoid Docker Hub rate limits, reduce NAT costs, and benefit from layer deduplication with your own images.

**Q: "Does ECR bill based on compressed or uncompressed image size?"**

> Compressed. ECR stores image layers in gzip-compressed form and charges based on the compressed bytes. The `imageSizeInBytes` field in the ECR API reflects compressed size. The `docker images` command shows uncompressed size. In practice, compression reduces the stored size by 50-70%, so a 450MB image on disk may only consume 150MB in ECR storage. Never estimate ECR costs from `docker images` output — always use `aws ecr describe-images` for accurate numbers.

**Q: "A team accidentally pushed 500 versions of an image to ECR over the past year and the bill is high. How do you fix it?"**

> Immediate fix:
> 1. Run `aws ecr start-lifecycle-policy-preview` with an aggressive policy (keep last 5 versions) to see what would be deleted.
> 2. If the preview looks correct, apply the policy with `aws ecr put-lifecycle-policy`.
> 3. ECR will asynchronously delete the old images, reducing storage costs starting the next day.
>
> Long-term prevention:
> 1. Apply lifecycle policies to all repositories at creation time using a Terraform module or CloudFormation template.
> 2. Set a CloudWatch billing alarm to alert when ECR costs exceed a defined threshold.
> 3. Integrate cost reporting into your monthly engineering review.

---

*Next: [06_ECR_Advanced_Topics.md →](./06_ECR_Advanced_Topics.md)*
