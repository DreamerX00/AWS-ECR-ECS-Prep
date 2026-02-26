# 🏗️ ECR Internal Architecture — Deep Dive

---

## 📖 Concept Explanation

ECR ko ek black box mat samjho. Andar kya chal raha hai ye samajhna:
- Debug fast karte ho
- Cost optimize karte ho
- Security design better karte ho

### ECR ke 4 Core Components:

1. **Registry** — Account-level container (1 per account per region by default)
2. **Repository** — Logical grouping of related images
3. **Image** — OCI-compliant container image
4. **Manifest** — JSON file describing image layers and config

---

## 🏗️ Internal Architecture — Full Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AMAZON ECR SERVICE                           │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    CONTROL PLANE                             │    │
│  │  ┌────────────────┐   ┌──────────────┐   ┌──────────────┐  │    │
│  │  │  ECR API       │   │   IAM Auth   │   │  Lifecycle   │  │    │
│  │  │  (Registry API)│   │   Service    │   │  Policy Mgr  │  │    │
│  │  └────────────────┘   └──────────────┘   └──────────────┘  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    DATA PLANE                                │    │
│  │                                                             │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │   METADATA STORE (DynamoDB-like)                      │   │    │
│  │  │   - Repository info                                   │   │    │
│  │  │   - Image manifests                                   │   │    │
│  │  │   - Tag ↔ Digest mappings                            │   │    │
│  │  │   - Scan results                                      │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                                                             │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │   LAYER STORE (S3-backed)                             │   │    │
│  │  │   s3://ecr-intl-us-east-1-<hash>/                    │   │    │
│  │  │   ├── layers/                                         │   │    │
│  │  │   │   ├── sha256:aaa...  ← Compressed layer blob     │   │    │
│  │  │   │   ├── sha256:bbb...  ← Referenced by 3 repos!    │   │    │
│  │  │   │   └── sha256:ccc...                               │   │    │
│  │  │   └── manifests/                                      │   │    │
│  │  │       └── sha256:manifest-hash...                     │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                 SECURITY PLANE                               │    │
│  │  ┌────────────────┐   ┌──────────────┐   ┌──────────────┐  │    │
│  │  │  AWS Inspector │   │   KMS Svc    │   │  CloudTrail  │  │    │
│  │  │  (Scanning)    │   │ (Encryption) │   │  (Audit)     │  │    │
│  │  └────────────────┘   └──────────────┘   └──────────────┘  │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Repository Deep Dive

### Repository Naming Convention:
```
Registry URL Format:
<account-id>.dkr.ecr.<region>.amazonaws.com/<namespace>/<name>

Examples:
123456789.dkr.ecr.us-east-1.amazonaws.com/myapp              ← Simple
123456789.dkr.ecr.us-east-1.amazonaws.com/team/microservice  ← Namespaced
123456789.dkr.ecr.us-east-1.amazonaws.com/prod/api/v2        ← Multi-level (up to 3 levels)
```

### Image Manifest Structure (OCI):
```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 7023,
    "digest": "sha256:config-digest-here"   ← Points to env, cmd, labels
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 27119534,
      "digest": "sha256:layer1-digest"    ← Base OS layer (100MB compressed)
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 2418641,
      "digest": "sha256:layer2-digest"    ← App layer (9MB compressed)
    }
  ]
}
```

### How Storage Deduplication Works:
```
BEFORE ECR dedup (naively storing):
  myapp:v1 = [base-layer + node-layer + app-v1-layer]  = 300MB
  myapp:v2 = [base-layer + node-layer + app-v2-layer]  = 305MB
  TOTAL = 605MB stored

AFTER ECR dedup (actual):
  base-layer (stored ONCE)  = 100MB
  node-layer (stored ONCE)  = 150MB
  app-v1-layer              = 50MB
  app-v2-layer              = 55MB
  TOTAL = 355MB → 41% savings!

ECR tracks: "Which repos reference this layer digest?"
  sha256:base-layer → referenced by: myapp, nginx, payment-svc
  Delete myapp → layer stays (still used by 2 others)
  Only when ALL references gone → layer actually deleted
```

---

## 🎯 Analogy — Smart Library with Shared Chapters 📚

Imagine ek bahut smart library jo:
1. Common chapters (like "Introduction to Python") detect karta hai
2. Us ek chapter ko sirf ek baar print karta hai
3. Har book mein sirf "Chapter 3 is on shelf 7B" note karta hai
4. Jab koi reader Chapter 3 mangta hai, wah common shelf se laata hai

**ECR ka deduplication = exact same system:**
- Layers (chapters) detect karta hai jo multiple images mein same hain
- Ek hi baar store karta hai (S3 backend pe)
- Manifests (books) mein sirf digest references store hote hain
- Pull time pe: already downloaded layer? → SKIP, reuse local!

---

## 🌍 Real-World Scenario

### Calculating Real ECR Storage Cost:

```
Company: 10 microservices
Base image: node:18-alpine = 120MB (compressed)
Each service adds ~30MB unique layers

NAIVE CALCULATION:
10 × (120 + 30)MB = 1500MB per version

ECR DEDUPLICATION:
Stored: 120MB (base) + 10 × 30MB (unique) = 420MB per version!
Savings: 1080MB → 72% less storage!

With 20 versions in ECR (before lifecycle policy):
NAIVE: 20 × 1500MB = 30GB
ACTUAL: 120MB + 20 × 10 × 30MB = 6120MB ≈ 6GB
COST SAVINGS: At $0.10/GB = $2.40/month saved vs $3.00 actual

At scale: hundreds of services → MASSIVE savings
```

---

## ⚙️ Hands-On Examples

### Inspect ECR Repository Architecture:
```bash
# Get repository details
aws ecr describe-repositories --repository-names myapp

# Get image manifest (the actual OCI JSON)
aws ecr batch-get-image \
  --repository-name myapp \
  --image-ids imageTag=v1.0.0 \
  --query 'images[0].imageManifest' \
  --output text | jq .

# Get detailed image info including layer digests
aws ecr describe-images \
  --repository-name myapp \
  --query 'imageDetails[*].{
    Tags:imageTags,
    Digest:imageDigest,
    Size:imageSizeInBytes,
    PushedAt:imagePushedAt,
    Vulnerabilities:imageScanFindingsSummary.findingCounts
  }'

# Calculate total storage used by repository
aws ecr describe-images \
  --repository-name myapp \
  --query 'sum(imageDetails[*].imageSizeInBytes)'
```

### Registry Settings:
```bash
# Get registry-level settings
aws ecr get-registry-policy

# Check registry scanning config
aws ecr get-registry-scanning-configuration

# Enable registry-level enhanced scanning
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{
    "repositoryFilters": [{"filter": "*", "filterType": "WILDCARD"}],
    "scanFrequency": "SCAN_ON_PUSH"
  }]'
```

### Image Layer Inspection:
```bash
# Get image download URL for specific layer
aws ecr get-download-url-for-layer \
  --repository-name myapp \
  --layer-digest sha256:abc123...

# Check if layer exists (for push optimization)
aws ecr batch-check-layer-availability \
  --repository-name myapp \
  --layer-digests sha256:abc123... sha256:def456...

# Response:
# {
#   "layers": [
#     {"layerDigest": "sha256:abc...", "layerAvailability": "AVAILABLE"}
#   ]
# }
# "AVAILABLE" → docker push will SKIP this layer (already there!)
```

---

## 🚨 Gotchas & Edge Cases

### 1. ECR Storage = Compressed Size
```bash
# docker images shows UNCOMPRESSED size
docker images myapp:v1  # SIZE: 450MB (uncompressed on disk)

# ECR charges COMPRESSED size
aws ecr describe-images --repository-name myapp
# imageSizeInBytes: 180000000  ← 180MB compressed (60% smaller!)

# Budget based on compressed size for ECR cost estimates
```

### 2. Image Size Limit = 10GB per image
```
ECR max image size: 10GB per image (uncompressed)
Practical limit: Keep images under 1GB for fast ECS task starts
Fargate: Larger images = longer task startup time = slower scaling
```

### 3. Repository Deletion = ALL Images Deleted
```bash
# DANGEROUS:
aws ecr delete-repository --repository-name myapp
# This deletes ALL images in the repo!

# Safe deletion (force=false first to check):
aws ecr delete-repository --repository-name myapp  
# Fails if images exist! Good.

# Force delete (use carefully!):
aws ecr delete-repository --repository-name myapp --force
```

### 4. ECR is Regional — Global ≠ Automatic
```
Your team in Mumbai pulls images from us-east-1 ECR:
→ Cross-region data transfer charges: $0.02-0.09/GB
→ Slower pulls (latency)
→ Risk: If us-east-1 has outage, ap-south-1 team can't pull!

Solution: Set up cross-region replication to ap-south-1
(Covered in Advanced ECR Topics)
```

---

## 🎤 Interview Angle

**Q: "ECR internally kaise images store karta hai? Deduplication kaise kaam karta hai?"**

> ECR S3-backed layer store use karta hai jahan layers content-addressed hain (sha256 digest ke basis par).
> Agar do images share karein ek layer → sirf ek copy store hoti hai S3 pe.
> Manifests (metadata DB mein) track karte hain ki kaun sa layer kaun si image use karti hai.
> Delete operation mein: layer sirf tab delete hota hai jab koi bhi image use nahi kar rahi.
> Result: Multi-service setups mein 40-70% storage savings.

**Q: "ECR image ka size calculation kaise hota hai billing ke liye?"**

> ECR **compressed size** charge karta hai, not uncompressed.
> `imageSizeInBytes` in ECR API = compressed bytes.
> Typically 3-4x smaller than what `docker images` dikhata hai.
> Rate: $0.10/GB/month for private ECR.

---

*Next: [03_ECR_Authentication_Model.md →](./03_ECR_Authentication_Model.md)*
