# Image Immutability, Tagging vs Digest

---

## Concept Explanation

### Image Immutability
Docker image layers are **immutable** once built. Once a layer's SHA256 hash is computed from its content, that hash never changes — the content is frozen. This is what makes images reliable and reproducible.

**However** — tags are mutable. A tag is simply a human-readable pointer to a manifest digest. That pointer can be reassigned at any time. This mismatch between immutable content and mutable labels is the source of many production incidents.

### Tag vs Digest — Critical Distinction

| | **Tag** | **Digest** |
|--|--|--|
| Format | `nginx:latest` or `myapp:v1.0` | `nginx@sha256:abc123...` |
| Mutable? | Yes — a tag can be reassigned to a different image | No — a cryptographic hash of the manifest content |
| Security guarantee | None — you cannot verify what you will get | Strong — the digest cryptographically identifies exact content |
| Example | `latest` points to different images over time | `sha256:abc...` always resolves to the identical image |
| Appropriate use case | Development, convenience | Production deployments, compliance, security auditing |

---

## Internal Architecture

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
                          │ sha256:ccddee..  │  ← immutable
                          └────────┬────────┘
                                   │
                          ┌────────▼────────┐
                          │    Layers []     │
                          │  sha256:layer1   │  ← immutable
                          │  sha256:layer2   │  ← immutable
                          └─────────────────┘

# When someone pushes a new image with the same tag (e.g., :latest):
# → The 'latest' tag pointer moves to the NEW manifest
# → The OLD manifest content remains in ECR (orphaned, without a tag)
# → The old content is still accessible by its digest
# → The old content will eventually be cleaned up by a lifecycle policy
```

### ECR Immutable Tags Feature:
```bash
# Enable immutable tags on a repository
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# Now, attempting to push to an already-used tag fails:
# docker push myapp:v1.0.0   → Error: tag already exists and is immutable
# You must increment the version: docker push myapp:v1.0.1
```

This is one of the most important ECR settings to enable in production. It prevents accidental or malicious overwriting of existing image versions.

---

## Analogy — The Mailing Address Problem

**Tag = Street Address**
- "123 Main Street" might have been a hospital last year and is now a shopping mall
- The address is the same; what it points to has changed
- Using `:latest` is like shipping important packages to "123 Main Street" — you cannot know what you will find there

**Digest = GPS Coordinates**
- 28.6139° N, 77.2090° E always points to the exact same location, without ambiguity, regardless of what building is constructed there
- `sha256:abc123...` always resolves to the exact same image content — guaranteed by cryptographic proof

---

## Real-World Scenarios

### Scenario 1: The `:latest` Tag Disaster

```
Production incident timeline at a startup:

Week 1: Deploy myapp:latest → Working correctly
Week 2: Developer pushes a new image to myapp:latest (new code, different behavior)
Week 3: Auto-scaling event triggers ECS to launch new tasks
        → New tasks pull myapp:latest → They get the Week 2 code
        → Existing tasks are still running the Week 1 code
        → Two different versions of the application are running simultaneously
        → Cascading failures begin as the API contract between them diverges

Root cause: Using :latest tag in the ECS Task Definition

Fix:
1. Pin to a specific version tag: myapp:v2.3.1
2. Better still, pin to digest: myapp@sha256:abc123...
3. Enable immutable tags in ECR to prevent future accidents
```

### Scenario 2: Compliance and Security Auditing

```
Security audit requirement (SOC 2, ISO 27001, PCI-DSS):
"Provide cryptographic proof of exactly what code ran in production on 2024-12-15."

With tags only:   "We deployed myapp:v3.2.1"
                  → Was that tag ever overwritten? Impossible to prove without additional records.
                  → Auditor cannot independently verify.

With digests:     "We deployed myapp@sha256:abc123def456..."
                  → The digest is a cryptographic fingerprint of the exact manifest
                  → ECR stores immutable digest records with timestamps
                  → Any party can independently verify that sha256:abc123... matches
                     the specific layers, config, and content you claim to have run

ECR stores image scan results, push timestamps, and digest history —
all keyed by digest — making this audit trail fully automated.
```

### Scenario 3: Multi-arch Tag Strategy
```bash
# A single tag can point to an OCI manifest list covering multiple architectures
docker buildx create --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myrepo/myapp:v2.0.0 \
  --push .

# The tag points to a manifest list, not a single manifest
# Each architecture has its own independent digest:
docker manifest inspect myrepo/myapp:v2.0.0
# {
#   "manifests": [
#     {"digest": "sha256:amd64hash...", "platform": {"architecture": "amd64"}},
#     {"digest": "sha256:arm64hash...", "platform": {"architecture": "arm64"}}
#   ]
# }

# The tag is one label; the architecture-specific digests are immutable references
```

---

## Hands-On Examples

### Working with Digests:
```bash
# Pull by digest — guarantees the exact same image regardless of tag state
docker pull nginx@sha256:e4f0474a75c510f40b37b6b7dc2516241ffa8bde5a442bde3d372c9519c84d90

# Get the digest of a locally pulled image
docker inspect nginx:latest | jq '.[0].RepoDigests'
# ["nginx@sha256:e4f0474a75c5..."]

# Get the digest of an image in ECR
aws ecr describe-images \
  --repository-name myapp \
  --image-ids imageTag=v1.0.0 \
  --query 'imageDetails[0].imageDigest'
# "sha256:abcdef123456..."

# Reference an image by digest in an ECS Task Definition:
# "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp@sha256:abcdef..."
```

### ECR Immutable Tags Setup:
```bash
# Check current mutability setting
aws ecr describe-repositories \
  --repository-names myapp \
  --query 'repositories[0].imageTagMutability'
# "MUTABLE"  ← dangerous default for production

# Enable immutable tags (recommended for all production repositories)
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# Attempt to push to an existing tag:
docker tag myapp:latest myapp:v1.0.0
docker push myapp:v1.0.0
# Error: "Tag v1.0.0 already exists in ECR repository AND is IMMUTABLE"
# You are now forced to use a new version tag.
```

### Tagging Strategy Best Practices:
```bash
# Semantic versioning (human-readable, recommended for release tracking)
docker tag myapp:build myapp:1.2.3          # Exact patch version
docker tag myapp:build myapp:1.2            # Minor version (floating)
docker tag myapp:build myapp:1              # Major version (floating)

# Git-based tagging (recommended for CI/CD traceability)
GIT_SHA=$(git rev-parse --short HEAD)
BUILD_DATE=$(date +%Y%m%d)

docker tag myapp:build myapp:$GIT_SHA                    # e.g., myapp:a1b2c3d
docker tag myapp:build myapp:$BUILD_DATE-$GIT_SHA        # e.g., myapp:20240115-a1b2c3d

# In ECS Task Definitions: use the GIT_SHA tag
# → Exact traceability: you can always look up which commit is running in production
# → Immutable (a given git SHA never changes)
# → Easily correlated with your source code, CI builds, and deployments
```

### ECR Lifecycle Policy — Automatic Cleanup of Old Tags:
```bash
# Keep only the 10 most recent images, expire everything older
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

Without lifecycle policies, ECR repositories accumulate images indefinitely, incurring storage costs. Define lifecycle policies on every production repository from day one.

---

## Gotchas & Edge Cases

### 1. Never Use `:latest` in an ECS Task Definition
```json
// BAD: Tag is mutable — auto-scaling will pull whatever "latest" is at that moment
{
  "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest"
}

// GOOD: Pinned to a specific immutable version
{
  "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:a1b2c3d"
}

// BEST: Pinned to a cryptographic digest for absolute immutability
{
  "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp@sha256:abcdef..."
}
```

### 2. Rebuilding the Same Dockerfile Produces a Different Digest
```bash
# Build the image the first time:
docker build -t myapp .
docker push myapp:v1.0.0  # sha256:aaabbb...

# Rebuild with an identical Dockerfile and identical source:
docker build -t myapp .   # Build timestamps differ in image metadata!
docker push myapp:v1.0.2  # sha256:cccdd...  ← Different digest!

# The image config includes build timestamps, causing digest differences
# even when the Dockerfile and source are byte-for-byte identical.
# Solutions:
# - Use SOURCE_DATE_EPOCH for reproducible builds
# - Use --no-cache to ensure deterministic layer ordering
# - Accept that digests change per build and use git SHA tags instead
```

### 3. Deleting a Tag Does Not Delete the Image Layers
```bash
# Remove a tag from ECR
aws ecr batch-delete-image \
  --repository-name myapp \
  --image-ids imageTag=old-feature-branch

# The underlying image layers are NOT deleted if other tags reference the same manifest.
# Only when NO remaining tags point to a manifest digest does the image become "untagged."
# Untagged images are then eligible for removal via lifecycle policies.
# This is important for cost management — storage costs continue until layers are
# actually deleted, not just untagged.
```

### 4. ECR Image Scanning Results Are Tied to Digest, Not Tag
```bash
# Vulnerability scan results are stored against the image digest
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageDigest=sha256:abc123...

# If a tag is overwritten with a new image, the old scan results are lost
# because the tag no longer points to the previously-scanned digest.
# This is another reason to use immutable tags:
# each digest has a permanent, queryable scan history.
```

### 5. ECS Task Definition Caching Behavior
When ECS launches a new task, it pulls the specified image from ECR onto the container instance. If the task definition references a tag (like `v1.2.3`), ECS checks if that tag's manifest is already cached on the instance. If the tag has been overwritten (and tags are mutable), ECS may use a stale cached version rather than pulling the new image. Pinning to a digest eliminates this ambiguity entirely — if the digest is already cached, it is by definition the correct image.

---

## Interview Angle

**Q: "Why should you never use the `:latest` tag in a production ECS Task Definition?"**

> Tags are mutable pointers — `:latest` can be reassigned to a completely different image at any time.
> In ECS, when a service scales out, new tasks pull whatever the tag currently resolves to.
> If `:latest` has been updated since the last deployment, new tasks run different code than existing tasks.
> This creates version inconsistency across your running service and is extremely difficult to debug.
> The solution is to always pin to a specific, immutable reference: either a version tag that you protect with ECR's immutable tag setting, or a digest for absolute cryptographic certainty.

**Q: "What is an image digest and when should you use it?"**

> An image digest is the SHA256 hash of the image manifest — it is a cryptographic fingerprint that uniquely and immutably identifies a specific image version.
> Unlike a tag, a digest cannot be reassigned. If you reference `myapp@sha256:abc123...`, you are guaranteed to always get the exact same image, regardless of tag changes or repository modifications.
> Use digests in production ECS task definitions for complete immutability, in compliance-regulated environments where you must prove exactly what ran, and when you need to reference a specific architecture variant of a multi-arch image.

**Q: "How does ECR's immutable tags feature help in production?"**

> By default, ECR allows any tag to be overwritten by pushing a new image with the same tag name. This is a footgun in production — a developer could accidentally (or intentionally) push broken code over a known-good version tag.
> Enabling immutable tags on an ECR repository makes it impossible to push to an existing tag. You are forced to use a new tag for every new image. This enforces a strict versioning discipline, preserves audit trails, and prevents the accidental deployment of untested code to production.

---

*Phase 0 Complete! Next: [Phase-1_Amazon-ECR-Deep-Dive →](../Phase-1_Amazon-ECR-Deep-Dive/01_What_Is_ECR.md)*
