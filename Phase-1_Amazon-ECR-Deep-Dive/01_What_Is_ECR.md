# 🗄️ What is Amazon ECR?

---

## 📖 Concept Explanation

**Amazon ECR (Elastic Container Registry)** AWS ka **fully managed Docker container registry** hai. Jaise GitHub code store karta hai, ECR **container images** store karta hai.

Simple definition:
> ECR = Private/Public Docker Registry jo AWS manage karta hai, natively AWS services (ECS, EKS, Lambda) ke saath integrate hota hai.

### Types of ECR:

#### 🔵 Private ECR
- Tera personal container image store
- Account-level isolation
- IAM-based access control
- Default for production workloads
- URI format: `<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>`

#### 🟢 Public ECR (public.ecr.aws)
- Anyone publicly pull kar sakta hai
- Like Docker Hub, but AWS managed
- Free egress for public images
- URI format: `public.ecr.aws/<alias>/<repo-name>`
- Used for: open source projects, base images sharing

### ECR vs Docker Hub vs Quay.io:

| Feature | ECR | Docker Hub | Quay.io |
|---------|-----|------------|---------|
| Managed by | AWS | Docker Inc | Red Hat |
| AWS Integration | Native ✅ | Manual | Manual |
| IAM Auth | ✅ | ❌ | ❌ |
| VPC Pull (private) | ✅ | Needs NAT | Needs NAT |
| Image Scanning | Enhanced (AWS Inspector) | Basic | Basic |
| Rate Limits | None | Yes (free: 100/6hr) | None |
| Cost | $0.10/GB/month | Free + paid tiers | Free + paid |
| Geo-replication | ✅ | Paid | Paid |

---

## 🏗️ Internal Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Account                                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    AMAZON ECR                             │   │
│  │                                                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │  Repository  │  │  Repository  │  │   Repository    │  │   │
│  │  │  "myapp"     │  │  "nginx"     │  │   "payment-svc" │  │   │
│  │  │             │  │             │  │                  │  │   │
│  │  │  v1.0 ─┐   │  │  latest ─┐  │  │  main ─┐        │  │   │
│  │  │  v1.1 ─┤   │  │  1.24  ─┤  │  │  v3.2 ─┤        │  │   │
│  │  │  v2.0 ─┘   │  │  1.25  ─┘  │  │  v3.3 ─┘        │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  │                                                           │   │
│  │  ┌──────────────────────────────────────────────────┐    │   │
│  │  │              LAYER STORAGE                        │    │   │
│  │  │   (S3-backed, encrypted, deduplicated)            │    │   │
│  │  │                                                   │    │   │
│  │  │  sha256:layer1 ─ 100MB  (shared by 3 images)     │    │   │
│  │  │  sha256:layer2 ─ 50MB   (unique to myapp)        │    │   │
│  │  │  sha256:layer3 ─ 200MB  (shared by 2 images)     │    │   │
│  │  └──────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│           ▲ Pull              ▲ Pull          ▲ Pull             │
│           │                  │               │                  │
│        ECS Tasks           EKS Pods       Lambda              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Analogy — A Library System 📚

**ECR = College Library:**

- **Registry** = Puri library building
- **Repository** = Ek book shelf (jaise "Computer Science shelf")
- **Image** = Ek particular book (jaise "AWS Cookbook")
- **Tag** = Book edition (1st edition, 2nd edition, latest)
- **Digest** = ISBN number (har kitab ka unique identifier)
- **Layers** = Book ke chapters (jo across books share ho sakte hain)

Librarian (IAM) decide karta hai kaun kaunsi shelf access kar sakta hai:
- Kuch labo ke students → sirf Computer Science shelf
- Faculty → sab shelves
- Public → sirf reading room

---

## 🌍 Real-World Scenario

### Multi-Team Startup Architecture:

```
Startup "RapidCart" ke 5 teams hain:
├── Backend Team → myapp/backend:v*.*.* 
├── Frontend Team → myapp/frontend:v*.*.*
├── ML Team → myapp/model-server:v*.*.*
├── Data Team → myapp/batch-processor:v*.*.*
└── Infra Team → myapp/base-images/python:3.11-custom

ECR Structure:
├── rapidcart/backend          → Backend team pushes here
├── rapidcart/frontend         → Frontend team pushes here
├── rapidcart/model-server     → ML team pushes here
├── rapidcart/batch-processor  → Data team pushes here
└── rapidcart/base/python      → Infra team → all other teams PULL

Benefits:
✅ Single place for all images
✅ IAM controls who pushes/pulls what
✅ Layer deduplication: base image stored ONCE
✅ Native ECS integration: no auth config needed
✅ Enhanced scanning: CVEs detected automatically
```

---

## ⚙️ Hands-On Examples

### Create ECR Repository:
```bash
# Private repository banao
aws ecr create-repository \
  --repository-name myapp/backend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# KMS encryption ke saath:
aws ecr create-repository \
  --repository-name myapp/backend \
  --encryption-configuration \
    encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123:key/abc-123

# Public ECR repository banao  
aws ecr-public create-repository \
  --repository-name my-open-source-tool
```

### Login, Tag, Push:
```bash
# ECR mein authenticate karo
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Local image tag karo
docker tag my-local-image:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/backend:v1.0.0

# Push karo
docker push \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/backend:v1.0.0

# Pull karo
docker pull \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/backend:v1.0.0
```

### List and Describe Repositories:
```bash
# All repositories
aws ecr describe-repositories

# Specific repo ka details
aws ecr describe-repositories --repository-names myapp/backend

# Images in a repo
aws ecr list-images --repository-name myapp/backend

# Image details with sizes
aws ecr describe-images \
  --repository-name myapp/backend \
  --query 'imageDetails[*].{Tag:imageTags[0],Size:imageSizeInBytes,Pushed:imagePushedAt}' \
  --output table
```

---

## 🚨 Gotchas & Edge Cases

### 1. Region-Specific Endpoints
```
ECR is regional! 
- us-east-1 ki registry: 123456789.dkr.ecr.us-east-1.amazonaws.com
- ap-south-1 ki registry: 123456789.dkr.ecr.ap-south-1.amazonaws.com

ECS task us-east-1 mein → pull from us-east-1 ECR (free, within region)
ECS task ap-south-1 mein → pull from us-east-1 ECR → DATA TRANSFER CHARGES!

Solution: Cross-region replication (Phase 1, Topic 6)
```

### 2. ECR Pull Rate — Unlike Docker Hub
```
Docker Hub: 100 pulls/6 hours for anonymous users (CI/CD killer!)
ECR Private: NO rate limits — pull as much as you want
ECR Public: Limited for non-AWS IPs but very high limits for AWS traffic
```

### 3. VPC Endpoint for ECR
```bash
# Without VPC endpoint: ECR pull → goes over internet (needs NAT Gateway!)
# NAT Gateway = $0.045/hour + $0.045/GB = EXPENSIVE for large images!

# With VPC Endpoint: Private network, no NAT needed!
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345 \
  --security-group-ids sg-12345

# Fargate ALWAYS uses VPC endpoints if configured
# = Significant cost savings for image-heavy workloads
```

---

## 🎤 Interview Angle

**Q: "ECR kyu use karein Docker Hub ki jagah, jab dono same kaam karte hain?"**

> ECR AWS ecosystem ka native citizen hai:
> 1. **IAM Auth:** Password management nahi → IAM roles se auto-auth (ECS, EKS, CodeBuild)
> 2. **No rate limits:** Docker Hub free plan = 100 pulls/6hr (CI breaks!)
> 3. **VPC Pull:** Private subnet se without NAT Gateway ECR pull ho sakta hai
> 4. **Native Scanning:** AWS Inspector se deep CVE scanning (no third-party setup)
> 5. **KMS encryption:** Your key, your control
> 6. **Cross-region replication:** Built-in, click to configure

---

*Next: [02_ECR_Internal_Architecture.md →](./02_ECR_Internal_Architecture.md)*
