# 💰 ECR Cost Components — Every Penny Counts

---

## 📖 Concept Explanation

ECR costs samajhna important hai kyunki uncontrolled image storage + cross-region pulls real money burn karte hain. Main costs:

1. **Storage** — Images ka data store karna
2. **Data Transfer** — Images pull/push karna  
3. **Scanning** — Enhanced scanning charges

---

## 💵 Pricing Breakdown

### 1. Storage

| Type | Price | Notes |
|------|-------|-------|
| Private ECR | **$0.10/GB/month** | After 500MB free tier |
| Public ECR | **Free** | For storage under 50GB |
| Free Tier | **500MB/month** | Per account for Private ECR |

> ⚠️ Storage = COMPRESSED size, not uncompressed!

### 2. Data Transfer

| Transfer Type | Price |
|---------------|-------|
| Same region ECR → ECS | **Free** (within same region) |
| Cross-region | **$0.02–0.09/GB** (varies by region) |
| ECR → Internet | **$0.09/GB** first 10TB |
| ECR Public → Internet (non-AWS) | **$0.00** Up to 500GB/month, then standard |
| ECR Public → AWS services | **Free** always |

### 3. Enhanced Scanning (AWS Inspector)

| Item | Price |
|------|-------|
| Container image scanning | **$0.09/image/month** per repo |
| First time scan | Included in Inspector base |

---

## 🧮 Cost Calculation — Real Examples

### Example 1: Single Service Startup

```
Service: node.js e-commerce backend
Image size: 800MB uncompressed → ~250MB compressed in ECR
Versions kept: 20 (with lifecycle policy)
Region: us-east-1
ECS deployment: Same region
Enhanced scanning: Enabled

STORAGE COST:
  250MB × 20 versions = 5,000MB = 5GB total
  After 500MB free: 4.5GB × $0.10 = $0.45/month

DATA TRANSFER COST:
  Same region pull: FREE
  Each ECS task launch = pull only changed layers (delta pull)
  Average delta each deploy: 50MB
  Deploys/month: 30
  Data transfer: 30 × 50MB = 1.5GB ← FREE (same region)

SCANNING COST:
  Basic scanning: FREE
  Enhanced scanning: $0.09/image/month × 20 images = $1.80/month

TOTAL: ~$2.25/month ← Very cheap!
```

### Example 2: Enterprise at Scale

```
Company: 50 microservices
Average image size: 300MB compressed
Versions per service: 30
Teams: Engineering in US (us-east-1), APAC team in ap-south-1

STORAGE COST (us-east-1):
  50 services × 30 versions × 300MB = 450GB
  Free tier: 500MB ≈ nothing
  450GB × $0.10 = $45/month

DATA TRANSFER COST (THE HIDDEN KILLER):
  APAC team ECS in ap-south-1 pulls from us-east-1:
  20 services × 10 deploys/month × 300MB pull
  = 60GB cross-region transfer
  60GB × $0.02 = $1.20/month (manageable)

  BUT on deployment day (all services at once):
  50 services × 300MB = 15GB in one go
  = $0.30 per deployment event × 10 deployments/month = $3/month cross-region

SCANNING COST:
  50 × 30 = 1500 image-months × $0.09 = $135/month!

TOTAL: ~$183/month

OPTIMIZED (with lifecycle policy → keep 5 versions):
  Storage: 50 × 5 × 300MB = 75GB × $0.10 = $7.50/month
  Scanning: 50 × 5 images × $0.09 = $22.50/month
  TOTAL: ~$31/month
  SAVINGS: 83%! By just setting lifecycle policy
```

---

## 🎯 Analogy — Netflix Video Storage 📺

**ECR Image Storage = Netflix kaise shows store karta hai:**

- Netflix ek show store karta hai, 10 crore log watch karte hain — storage ek baar pay
- Same show 4K/HD/SD versions = same base data, different encoding
- Purane shows (lifecycle policy) eventually delete hote hain

**Layer deduplication:**
- Node.js base image (100MB) = Netflix ka common intro animation
- Har show pe same intro = store ek baar, reference har jagah

**Cross-region transfer:**
- User Mumbai mein hai, video US pe store hai = bandwidth charges
- Netflix solution: CDN/edge caching = ECR ka equivalent: **cross-region replication + ECR Pull-Through Cache**

---

## 🌍 Real-World Cost Optimization Strategies

### Strategy 1: Aggressive Lifecycle Policy
```bash
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 3 prod images",
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
        "description": "Keep last 10 any tag",
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
        "description": "Delete untagged immediately",
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

### Strategy 2: Minimize Image Size
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

# Storage cost: 1.2GB vs 0.18GB
# ECR cost: $0.12/month vs $0.018/month per version
# Pull time on ECS: 45 sec vs 8 sec
```

### Strategy 3: VPC Endpoint (Eliminate NAT Costs)
```bash
# Without VPC endpoint: Image pull goes through NAT Gateway
# NAT Gateway: $0.045/hr + $0.045/GB data processing
# 100GB/month pulled → $4.50/month NAT cost!

# WITH ECR VPC endpoint: Direct private connection
# Pull via VPC endpoint: $0.01/GB endpoint charges (less than NAT!)

# Create endpoints:
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface

aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --vpc-endpoint-type Interface

# Also need S3 Gateway endpoint (layers stored in S3!)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway
  
# ECR pulls now: Private network, no NAT, cheaper!
```

### Strategy 4: Pull-Through Cache (For Public Images)
```bash
# Instead of: DockerHub → NAT → ECS (with rate limits!)
# Use:        DockerHub → ECR Pull-Through Cache → VPC → ECS

# Setup:
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "docker-hub" \
  --upstream-registry-url "registry-1.docker.io"

aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "quay" \
  --upstream-registry-url "quay.io"

# Now instead of: docker pull nginx:latest
# Use:            docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:latest
# → ECR caches it, deduplicates, no Docker Hub rate limits!
# → Benefits: No rate limits, layer dedup, VPC access, scanning!
```

---

## ⚙️ Hands-On Cost Monitoring

### Calculate Your Current Costs:
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
# Alert when ECR costs exceed $50/month
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

## 🚨 Gotchas & Edge Cases

### 1. Lifecycle Policy Evaluation Delay
```
ECR lifecycle policy runs asynchronously!
Policy set kiya → images kal delete honge (not immediately)
Testing ke liye: aws ecr start-lifecycle-policy-preview (dry run!)
```

### 2. Free Tier is Per ACCOUNT, Not Per Region
```
Free tier: 500MB total for private ECR across ALL regions
us-east-1 mein 400MB + ap-south-1 mein 200MB = 600MB total
100MB charged (600-500 = 100MB × $0.10 = $0.01/month)
```

### 3. Deleted Images → Not Immediately From Billing
```
ECR billing based on daily snapshot
Delete image aaj → billing tomorrow changes
Same-day cost not reduced
```

### 4. Enhanced Scanning Charges Per Unique Image
```
Same image (same digest) in 2 different repos:
→ Charged TWICE (once per repo)
Consolidate repos where possible for scanning cost optimization
```

---

## 🎤 Interview Angle

**Q: "ECR costs control karne ke liye kya strategies hain?"**

> 4 main strategies:
> 1. **Lifecycle policies** — Purani images auto-delete karo (biggest impact)
> 2. **Image size optimization** — Multi-stage builds, alpine base images → 60-80% size reduction  
> 3. **VPC Endpoints** — NAT Gateway charges eliminate karo for ECR pulls (for private subnet ECS)
> 4. **Pull-Through Cache** — Public registry images ECR mein cache karo → Docker Hub rate limits avoid + dedup

**Q: "ECR billing compressed ya uncompressed size pe hota hai?"**

> Compressed! ECR stores layers compressed (gzip).
> `imageSizeInBytes` in ECR API = compressed bytes.
> `docker images` dikhata uncompressed size.
> Real billing typically 3-4x less than what docker images dikhata hai.
> Never estimate ECR costs from `docker images` output!

---

*Next: [06_ECR_Advanced_Topics.md →](./06_ECR_Advanced_Topics.md)*
