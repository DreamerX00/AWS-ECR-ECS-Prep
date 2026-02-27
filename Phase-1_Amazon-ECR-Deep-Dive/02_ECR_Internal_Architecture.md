# ECR Internal Architecture — Deep Dive

---

## Concept Explanation

Do not treat ECR as a black box. Understanding what happens internally enables you to:
- Debug issues faster
- Optimize costs effectively
- Design better security architectures

### ECR's 4 Core Components:

1. **Registry** — Account-level container (1 per account per region by default)
2. **Repository** — Logical grouping of related images
3. **Image** — OCI-compliant container image
4. **Manifest** — JSON file describing image layers and configuration

---

## Internal Architecture — Full Stack

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

## Repository Deep Dive

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
WITHOUT ECR deduplication (naive storage):
  myapp:v1 = [base-layer + node-layer + app-v1-layer]  = 300MB
  myapp:v2 = [base-layer + node-layer + app-v2-layer]  = 305MB
  TOTAL = 605MB stored

WITH ECR deduplication (actual behavior):
  base-layer (stored ONCE)  = 100MB
  node-layer (stored ONCE)  = 150MB
  app-v1-layer              = 50MB
  app-v2-layer              = 55MB
  TOTAL = 355MB → 41% savings!

ECR tracks: "Which repos reference this layer digest?"
  sha256:base-layer → referenced by: myapp, nginx, payment-svc
  Delete myapp → layer stays (still used by 2 others)
  Only when ALL references are gone → layer is actually deleted
```

---

## Analogy — A Smart Library with Shared Chapters

Imagine a very smart library that:
1. Detects common chapters (like "Introduction to Python") across multiple books
2. Physically prints that chapter only once
3. Records "Chapter 3 is on shelf 7B" in each book's table of contents
4. When a reader requests Chapter 3, the library retrieves the single shared copy

**ECR's deduplication works the same way:**
- It detects layers (chapters) that are identical across multiple images
- Stores each unique layer exactly once in the S3 backend
- Manifests (image metadata) store only digest references pointing to shared layers
- At pull time: if a layer already exists locally, Docker skips downloading it entirely

---

## Real-World Scenario

### Calculating Real ECR Storage Cost:

```
Company: 10 microservices
Base image: node:18-alpine = 120MB (compressed)
Each service adds ~30MB of unique layers

NAIVE CALCULATION:
10 × (120 + 30)MB = 1500MB per version

WITH ECR DEDUPLICATION:
Stored: 120MB (base) + 10 × 30MB (unique) = 420MB per version
Savings: 1080MB → 72% less storage!

With 20 versions in ECR (before lifecycle policy):
NAIVE:  20 × 1500MB = 30GB
ACTUAL: 120MB + 20 × 10 × 30MB = 6120MB ≈ 6GB
COST SAVINGS: At $0.10/GB → $2.40/month actual vs $3.00 naive

At scale with hundreds of services → the savings become enormous.
```

---

## Hands-On Examples

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
# Get image download URL for a specific layer
aws ecr get-download-url-for-layer \
  --repository-name myapp \
  --layer-digest sha256:abc123...

# Check if a layer already exists (useful for push optimization)
aws ecr batch-check-layer-availability \
  --repository-name myapp \
  --layer-digests sha256:abc123... sha256:def456...

# Response:
# {
#   "layers": [
#     {"layerDigest": "sha256:abc...", "layerAvailability": "AVAILABLE"}
#   ]
# }
# "AVAILABLE" → docker push will SKIP uploading this layer (already present)
```

---

## Gotchas & Edge Cases

### 1. ECR Storage Is Based on Compressed Size
```bash
# docker images shows UNCOMPRESSED size
docker images myapp:v1  # SIZE: 450MB (uncompressed on disk)

# ECR charges based on COMPRESSED size
aws ecr describe-images --repository-name myapp
# imageSizeInBytes: 180000000  ← 180MB compressed (60% smaller!)

# Always use compressed size for ECR cost estimates, not docker images output.
```

### 2. Image Size Limit = 10GB per Image
```
ECR max image size: 10GB per image (uncompressed)
Practical guideline: Keep images under 1GB for fast ECS task starts.
Fargate: Larger images = longer task startup time = slower auto-scaling response.
```

### 3. Repository Deletion Removes ALL Images
```bash
# DANGEROUS:
aws ecr delete-repository --repository-name myapp
# This deletes ALL images in the repository!

# Safe deletion check (fails if images exist — use this first):
aws ecr delete-repository --repository-name myapp
# Returns an error if the repository is not empty. Good safety check.

# Force delete (use with extreme caution):
aws ecr delete-repository --repository-name myapp --force
```

### 4. ECR is Regional — Global Availability is NOT Automatic
```
Your team in Mumbai pulls images from us-east-1 ECR:
→ Cross-region data transfer charges: $0.02-0.09/GB
→ Higher latency on each pull
→ Risk: If us-east-1 has an outage, the Mumbai team cannot pull images!

Solution: Set up cross-region replication to ap-south-1.
(Covered in Advanced ECR Topics)
```

### 5. Multi-Level Repository Namespacing Has a Depth Limit
```
ECR supports up to 3 forward-slash levels in repository names:
  Valid:   prod/api/v2
  Invalid: prod/api/v2/extra   ← Exceeds depth limit!

Plan your naming hierarchy before creating repositories at scale.
```

### 6. Layer Digests Are Content-Addressed — Not Random
```
A layer's sha256 digest is computed from its compressed content.
This means:
- Two images using the exact same base layer will share the digest
- Deduplication works across repositories within the same registry
- However, deduplication does NOT work across accounts or regions
  (replicated images are stored independently in each region)
```

---

## Interview Questions & Answers

**Q: "How does ECR store images internally? How does deduplication work?"**

> ECR uses an S3-backed layer store where layers are content-addressed by their sha256 digest. If two images share a layer, only one copy is stored in S3. The metadata store (similar to DynamoDB) tracks which images reference which layer digests. During a delete operation, a layer is only removed when no image references it anymore. In multi-service setups, this typically results in 40-70% storage savings.

**Q: "Why is the size shown by `docker images` different from what ECR reports?"**

> `docker images` shows the uncompressed size of the image as it would exist on the host filesystem after unpacking. ECR stores layers in their compressed (gzip) form and charges based on compressed size. Compression typically reduces image size by 50-70%, so what appears as 450MB locally may only consume 150MB in ECR storage.

---

*Next: [03_ECR_Authentication_Model.md →](./03_ECR_Authentication_Model.md)*
