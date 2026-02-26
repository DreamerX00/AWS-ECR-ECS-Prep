# 🔒 Image Immutability, Tagging vs Digest

---

## 📖 Concept Explanation

### Image Immutability
Docker images **immutable** (unchangeable) hoti hain once built. Ek baar layer ka SHA256 hash calculate ho gaya, woh change nahi hoga.

**BUT** — tagging mutable hai! Yahan problem aati hai.

### Tag vs Digest — Critical Distinction

| | **Tag** | **Digest** |
|--|--|--|
| Format | `nginx:latest` or `myapp:v1.0` | `nginx@sha256:abc123...` |
| Mutable? | ✅ YES — tag can be re-assigned | ❌ NO — cryptographic hash |
| Security | Risky for production | Safe — exactly what you expect |
| Example | `latest` points to different image over time | `sha256:abc...` always same image |
| Use case | Development, convenience | Production, compliance |

---

## 🏗️ Internal Architecture

### How Image Identification Works in ECR:
```
ECR Repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp

                          ┌─────────────────┐
TAG: latest  ──────────►  │  Image Manifest  │  ← sha256:aabbcc...
                          │  (pointer table) │
TAG: v1.2.3  ──────────►  │  image-index    │  ← can point to same manifest!
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │   Image Config   │
                          │ sha256:ccddee..  │  ← immutable!
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │    Layers []     │
                          │  sha256:layer1   │  ← immutable!
                          │  sha256:layer2   │  ← immutable!
                          └─────────────────┘

# Agar koi docker push myapp:latest kare with a NEW image:
# → 'latest' tag points to NEW manifest (different sha256!)
# → Old image ki content abhi bhi ECR mein hai (unlabeled)
# → Old sha256 diya toh puraani image mil jaayegi
```

### ECR Immutable Tags Feature:
```bash
# Immutable tags enable karo per repository
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# Ab: already-pushed tag dobara push karna = ERROR!
# docker push myapp:v1.0.0   → Already exists, fails!
# Force new version: docker push myapp:v1.0.1
```

---

## 🎯 Analogy — The Mailing Address Problem 📮

**Tag = House Address (like "Main Street")**
- "Main Street" pe pehle hospital tha, ab mall hai
- Address same hai, location different!
- `latest` = "Main Street" — kya pata kya delivera ho!

**Digest = GPS Coordinates (like 28.6139° N, 77.2090° E)**
- Latitude/longitude hamesha same jagah point karta hai
- No ambiguity, no surprise
- `sha256:abc123...` = exact same container contents garanteed!

---

## 🌍 Real-World Scenarios

### Scenario 1: The "latest" Tag Disaster

```
Production incident at a startup:

Week 1: Deploy myapp:latest → Works fine ✅
Week 2: Developer pushes new image → myapp:latest now = new code
Week 3: Auto-scaling event → New ECS tasks pull myapp:latest
        → Gets DIFFERENT code than running tasks!
        → INCONSISTENCY! Some tasks old, some new code
        → Cascading failures begin...
        
Root cause: Using :latest tag in ECS Task Definition ❌

Fix: 
1. Use specific versions: myapp:v2.3.1
2. Or better, use digest: myapp@sha256:abc123...
3. Enable immutable tags in ECR
```

### Scenario 2: Compliance & Audit Trail

```
Security audit (SOC 2, ISO 27001):
"Prove exactly what code ran in production on 2024-12-15"

With tags:    "We ran myapp:v3.2.1" 
              → But was that tag overwritten? Prove it...

With digests: "We ran myapp@sha256:abc123..."
              → Cryptographic proof! Immutable artifact.
              → Exact layer hashes, config, everything provable
              
AWS ECR has image scan history + digest records for compliance!
```

### Scenario 3: Multi-arch Tag Strategy
```bash
# One tag, multiple architectures using manifest list
docker buildx create --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myrepo/myapp:v2.0.0 \
  --push .

# Single tag points to manifest LIST
# Manifest list has different digests for each arch:
docker manifest inspect myrepo/myapp:v2.0.0
# {
#   "manifests": [
#     {"digest": "sha256:amd64hash...", "platform": {"arch": "amd64"}},
#     {"digest": "sha256:arm64hash...", "platform": {"arch": "arm64"}}
#   ]
# }
```

---

## ⚙️ Hands-On Examples

### Working with Digests:
```bash
# Pull by digest (guaranteed same image forever)
docker pull nginx@sha256:e4f0474a75c510f40b37b6b7dc2516241ffa8bde5a442bde3d372c9519c84d90

# Get digest of local image
docker inspect nginx:latest | jq '.[0].RepoDigests'
# ["nginx@sha256:e4f0474a75c5..."]

# Get digest from ECR
aws ecr describe-images \
  --repository-name myapp \
  --image-ids imageTag=v1.0.0 \
  --query 'imageDetails[0].imageDigest'
# "sha256:abcdef123456..."

# Reference image by digest in ECS Task Definition:
# Image: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp@sha256:abcdef...
```

### ECR Immutable Tags Setup:
```bash
# Check current mutability
aws ecr describe-repositories \
  --repository-names myapp \
  --query 'repositories[0].imageTagMutability'

# Make tags immutable (RECOMMENDED for production!)
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# Now try pushing to existing tag:
docker tag myapp:latest myapp:v1.0.0
docker push myapp:v1.0.0
# Error: "Tag v1.0.0 already exists in ECR repository AND is IMMUTABLE"
```

### Tagging Strategy Best Practices:
```bash
# Semantic versioning (recommended)
docker tag myapp:latest myapp:1.2.3          # Exact version
docker tag myapp:latest myapp:1.2            # Minor version
docker tag myapp:latest myapp:1              # Major version
docker tag myapp:latest myapp:latest         # Convenience

# Git-based tagging (great for CI/CD)
GIT_SHA=$(git rev-parse --short HEAD)
BUILD_DATE=$(date +%Y%m%d)

docker tag myapp:latest myapp:$GIT_SHA                    # e.g., myapp:a1b2c3d
docker tag myapp:latest myapp:$BUILD_DATE-$GIT_SHA        # e.g., myapp:20240115-a1b2c3d

# In ECS: Use the GIT_SHA tag!
# Exact traceability: know EXACTLY which commit is in production
```

### ECR Lifecycle Policy (Clean Old Tags):
```bash
# Policy to keep only last 10 images per feature branch
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 images",
        "selection": {
          "tagStatus": "any",
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {"type": "expire"}
      }
    ]
  }'
```

---

## 🚨 Gotchas & Edge Cases

### 1. `latest` in ECS Task Definition — NEVER DO THIS
```json
// BAD:
{
  "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest"
}
// → Auto-scaling pulls different images!

// GOOD:
{
  "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:a1b2c3d"
}
// → All tasks run exact same code
```

### 2. Digest Changes After Re-push (Even Same Content!)
```bash
# Rebuild image with same Dockerfile:
docker build -t myapp .  
docker push myapp:v1.0.0  # First push: sha256:aaa...

# Rebuild again (nothing changed):
docker build -t myapp .   # BUILD TIMESTAMPS ARE DIFFERENT!
docker push myapp:v1.0.2  # New digest: sha256:bbb...

# Build timestamp in metadata → different sha256!
# Solution: Use --no-cache and reproducible builds
```

### 3. Tag Deletion Doesn't Delete Image Layers
```bash
# Delete tag from ECR
aws ecr batch-delete-image \
  --repository-name myapp \
  --image-ids imageTag=old-tag

# Image content (layers) still exists if other tags reference it!
# Only when NO tags reference a digest → image becomes "untagged"
# Lifecycle policy cleanup needed for cost management
```

### 4. ECR Image Scanning by Digest
```bash
# Scan results are tied to digest, not tag!
# If tag is overwritten → old scan results gone
# ANOTHER reason to use immutable tags
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageDigest=sha256:abc123...
```

---

## 🎤 Interview Angle

**Q: "Why should you never use :latest tag in production ECS?"**

> Tags are mutable — `latest` can point to different images over time.
> If ECS service scales out, new tasks pull whatever `latest` currently is.
> This creates inconsistency — some tasks run old code, some run new.
> Solution: Always pin to a specific immutable version (git SHA or semantic version).
> Enable ECR Immutable Tags to prevent accidents.

**Q: "Image digest kya hai aur kab use karein?"**

> Image digest = SHA256 hash of the image manifest — cryptographically immutable.
> Tag se farq: Tag reassignable hai, digest never changes for the same content.
> Production mein use karo: compliance, exact reproducibility, security audits.
> ECS task definition mein `image@sha256:...` format use karo for complete immutability.

---

*Phase 0 Complete! Next: [Phase-1_Amazon-ECR-Deep-Dive →](../Phase-1_Amazon-ECR-Deep-Dive/01_What_Is_ECR.md)*
