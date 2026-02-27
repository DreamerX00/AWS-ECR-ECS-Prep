# What is Amazon ECR?

---

## Concept Explanation

**Amazon ECR (Elastic Container Registry)** is AWS's fully managed Docker container registry. Just as GitHub stores source code, ECR stores **container images**.

Simple definition:
> ECR is a private/public Docker registry that AWS manages, natively integrated with AWS services (ECS, EKS, Lambda).

### Types of ECR:

#### Private ECR
- Your personal container image store
- Account-level isolation
- IAM-based access control
- Default for production workloads
- URI format: `<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>`

#### Public ECR (public.ecr.aws)
- Anyone can publicly pull images
- Like Docker Hub, but AWS managed
- Free egress for public images
- URI format: `public.ecr.aws/<alias>/<repo-name>`
- Used for: open source projects, sharing base images

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

## Internal Architecture

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

## Analogy — A Library System

**ECR = College Library:**

- **Registry** = The entire library building
- **Repository** = A specific bookshelf (e.g., "Computer Science shelf")
- **Image** = A particular book (e.g., "AWS Cookbook")
- **Tag** = Book edition (1st edition, 2nd edition, latest)
- **Digest** = ISBN number (unique identifier for each book)
- **Layers** = Book chapters (which can be shared across multiple books)

The librarian (IAM) decides who can access which shelf:
- Lab students → only the Computer Science shelf
- Faculty → all shelves
- Public → reading room only

---

## Real-World Scenario

### Multi-Team Startup Architecture:

```
Startup "RapidCart" has 5 teams:
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
└── rapidcart/base/python      → Infra team manages; all other teams PULL

Benefits:
✅ Single place for all images
✅ IAM controls who pushes/pulls what
✅ Layer deduplication: base image stored ONCE
✅ Native ECS integration: no auth config needed
✅ Enhanced scanning: CVEs detected automatically
```

---

## Hands-On Examples

### Create ECR Repository:
```bash
# Create a private repository
aws ecr create-repository \
  --repository-name myapp/backend \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

# With KMS encryption:
aws ecr create-repository \
  --repository-name myapp/backend \
  --encryption-configuration \
    encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123:key/abc-123

# Create a Public ECR repository
aws ecr-public create-repository \
  --repository-name my-open-source-tool
```

### Login, Tag, Push:
```bash
# Authenticate to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag a local image
docker tag my-local-image:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/backend:v1.0.0

# Push the image
docker push \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/backend:v1.0.0

# Pull the image
docker pull \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/backend:v1.0.0
```

### List and Describe Repositories:
```bash
# List all repositories
aws ecr describe-repositories

# Get details for a specific repo
aws ecr describe-repositories --repository-names myapp/backend

# List images in a repo
aws ecr list-images --repository-name myapp/backend

# Get image details with sizes
aws ecr describe-images \
  --repository-name myapp/backend \
  --query 'imageDetails[*].{Tag:imageTags[0],Size:imageSizeInBytes,Pushed:imagePushedAt}' \
  --output table
```

---

## Gotchas & Edge Cases

### 1. ECR is Regional — Endpoints Are Region-Specific
```
ECR is regional!
- us-east-1 registry: 123456789.dkr.ecr.us-east-1.amazonaws.com
- ap-south-1 registry: 123456789.dkr.ecr.ap-south-1.amazonaws.com

ECS task in us-east-1 → pulls from us-east-1 ECR (free, within region)
ECS task in ap-south-1 → pulls from us-east-1 ECR → DATA TRANSFER CHARGES!

Solution: Cross-region replication (covered in Topic 6)
```

### 2. ECR Has No Pull Rate Limits — Unlike Docker Hub
```
Docker Hub: 100 pulls/6 hours for anonymous users (a CI/CD killer!)
ECR Private: NO rate limits — pull as much as you need
ECR Public: Very high limits for AWS traffic; limited for non-AWS IPs
```

### 3. VPC Endpoint for ECR Eliminates NAT Gateway Costs
```bash
# Without VPC endpoint: ECR pulls route over the internet (requires NAT Gateway!)
# NAT Gateway = $0.045/hour + $0.045/GB = expensive for large or frequent image pulls!

# With VPC Endpoint: traffic stays on the private AWS network — no NAT needed.
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345 \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-12345 \
  --security-group-ids sg-12345

# Fargate always uses VPC endpoints if they are configured.
# This results in significant cost savings for image-heavy workloads.
```

### 4. Token Expiry During Long CI/CD Pipelines
```
The ECR authorization token is valid for 12 hours.
If your CI/CD pipeline takes longer than 12 hours (unlikely but possible in large
monorepos or parallel test suites), the docker push will fail with a 401 error.
Solution: Re-authenticate just before the push step, not at the start of the pipeline.
```

### 5. Repository Names Are Case-Sensitive
```
ECR repository names must be lowercase and can contain:
  - Lowercase letters, digits, hyphens, underscores, periods, and forward slashes
  - Maximum 256 characters
  - "myapp/Backend" and "myapp/backend" are DIFFERENT repositories
Always enforce a lowercase naming convention in your team.
```

---

## Interview Questions & Answers

**Q: "Why use ECR instead of Docker Hub when both serve the same purpose?"**

> ECR is a native citizen of the AWS ecosystem:
> 1. **IAM Auth:** No password management — IAM roles provide automatic authentication for ECS, EKS, and CodeBuild.
> 2. **No rate limits:** Docker Hub free plan allows only 100 pulls/6 hours, which breaks CI pipelines.
> 3. **VPC Pull:** Images can be pulled from private subnets without a NAT Gateway using VPC endpoints.
> 4. **Native Scanning:** AWS Inspector provides deep CVE scanning with no third-party setup.
> 5. **KMS encryption:** You control your own encryption keys.
> 6. **Cross-region replication:** Built-in, configurable with a single API call.

**Q: "What is the difference between an image tag and an image digest in ECR?"**

> A tag (e.g., `v1.2.3` or `latest`) is a mutable pointer — it can be reassigned to a different image unless immutable tags are enabled. A digest (e.g., `sha256:abc123...`) is an immutable, content-addressed identifier derived from the image manifest. For reproducible deployments, always reference images by digest in production task definitions, not by tag alone. This guarantees that the exact same image is deployed every time, even if someone accidentally overwrites the tag.

---

*Next: [02_ECR_Internal_Architecture.md →](./02_ECR_Internal_Architecture.md)*
