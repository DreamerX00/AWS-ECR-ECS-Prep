# 📋 OCI Standard — Open Container Initiative

---

## 📖 Concept Explanation

**OCI (Open Container Initiative)** ek **open governance structure** hai jo container technology ke standards define karta hai. 2015 mein Docker, CoreOS aur doosron ne milke Linux Foundation ke under banaya.

**Problem jo OCI solve karta hai:** Pehle Docker ka apna proprietary format tha. Agar aap Docker use karte the, toh sirf Docker kaam karta tha. Kubernetes, CoreOS, Red Hat — sabke alag formats → **chaos!**

**OCI Solution:** Industry-wide standards banao taaki container ecosystem interoperable ho.

### OCI ke 3 Specifications:

| Spec | Kya define karta hai |
|------|---------------------|
| **Image Spec** | Container image kaise format mein hogi (manifest, layers) |
| **Runtime Spec** | Container kaise chalayi jaaye (namespaces, cgroups, mounts) |
| **Distribution Spec** | Image registry se kaise push/pull ho |

---

## 🏗️ Internal Architecture

### OCI Image Format:
```
OCI Image (stored in registry like ECR)
├── index.json              ← Entry point (multi-arch support)
├── oci-layout              ← OCI version marker
└── blobs/
    └── sha256/
        ├── <manifest-digest>    ← Image manifest
        │     ├── config digest  → Points to config
        │     └── layers[]       → Points to each layer
        ├── <config-digest>      ← Config (env, cmd, entrypoint)
        └── <layer1-digest>      ← Layer 1 tar.gz
        └── <layer2-digest>      ← Layer 2 tar.gz
```

### OCI Runtime Spec (what runc implements):
```json
// config.json — container specification
{
  "ociVersion": "1.0.2",
  "process": {
    "args": ["nginx", "-g", "daemon off;"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin"],
    "user": {"uid": 0, "gid": 0}
  },
  "root": {
    "path": "rootfs",   // ← Container filesystem root
    "readonly": false
  },
  "namespaces": [
    {"type": "pid"},
    {"type": "network"},
    {"type": "ipc"},
    {"type": "mount"},
    {"type": "uts"}
  ],
  "linux": {
    "resources": {
      "memory": {"limit": 536870912},  // 512MB
      "cpu": {"quota": 50000, "period": 100000}  // 0.5 CPU
    }
  }
}
```

### OCI Distribution Spec — Registry API:
```
GET  /v2/                              ← Registry version check
GET  /v2/<name>/tags/list              ← List tags
GET  /v2/<name>/manifests/<reference>  ← Get manifest
GET  /v2/<name>/blobs/<digest>         ← Pull layer
PUT  /v2/<name>/blobs/uploads/         ← Push layer
PUT  /v2/<name>/manifests/<reference>  ← Push manifest
```

---

## 🎯 Analogy — USB Standard 🔌

Pehle computers mein alag-alag ports the:
- Dell ka alag charger
- HP ka alag charger
- Sony ka alag charger
**= Chaos!**

Phir USB standard aaya → ek hi standard jo sabpe kaam kare.

**OCI = Container ka USB standard:**
- Pehle: Docker containers sirf Docker Engine pe chalte the
- Ab: OCI image ECR mein store karo, AWS Fargate pe chalao, ya GKE pe bhi chalao, ya locally runc se bhi chalao
- Standard = interoperability!

```
OCI Image (standardized)
      │
      ├── AWS ECS Fargate      ✅ Runs
      ├── Google Kubernetes    ✅ Runs  
      ├── Azure Container Inst ✅ Runs
      ├── docker run           ✅ Runs
      └── runc directly        ✅ Runs
```

---

## 🌍 Real-World Scenarios

### Scenario 1: Vendor Lock-in Avoidance
```
Company starts on AWS ECS.
Later, board decides: "Multi-cloud strategy!"

OLD WORLD (pre-OCI): 
  - Docker-specific format → Need to rewrite for GCE
  - Time: 6 months, Cost: $500K

OCI WORLD:
  - Same OCI-compliant images work everywhere
  - Just change deployment target
  - Time: 2 weeks, Cost: Minimal
```

### Scenario 2: Custom Container Runtime (AWS Fargate)
```
AWS Fargate uses a custom OCI-compliant runtime (not runc).
Your standard Docker image works on Fargate because:
1. Your image follows OCI Image Spec ✅
2. Fargate runtime implements OCI Runtime Spec ✅
3. ECR implements OCI Distribution Spec ✅

You don't need to know Fargate's internal runtime!
OCI handles the contract.
```

### Scenario 3: Multi-architecture Images
```bash
# OCI Image Index — one tag, multiple architectures:
docker buildx build --platform linux/amd64,linux/arm64 \
  --tag myrepo/myapp:latest \
  --push .

# ECR stores both! When you pull on:
# - EC2 (x86_64) → gets amd64 image
# - AWS Graviton (arm64) → gets arm64 image
# Same image name, OCI handles routing!
```

---

## ⚙️ Hands-On Examples

### Explore OCI Image Locally:
```bash
# Save OCI image to directory
docker save nginx:latest -o nginx.tar
mkdir nginx-oci && tar xf nginx.tar -C nginx-oci/
ls nginx-oci/
# manifest.json  layers/  config/

# Read manifest
cat nginx-oci/manifest.json | jq .

# Read config (entrypoint, cmd, env)
cat nginx-oci/config.json | jq .

# Each layer is a tar file
ls nginx-oci/  # You'll see sha256 directories
```

### OCI with Buildah (Build without Docker daemon!):
```bash
# Buildah = OCI-native build tool, no daemon needed
buildah from ubuntu:22.04
buildah run ubuntu-working-container -- apt-get update
buildah run ubuntu-working-container -- apt-get install -y nginx
buildah commit ubuntu-working-container my-nginx:latest

# Push OCI image to ECR
buildah push my-nginx:latest \
  aws_account_id.dkr.ecr.region.amazonaws.com/my-repo:latest
```

### Skopeo — OCI Registry Operations:
```bash
# Copy image between registries (no Docker daemon!)
skopeo copy \
  docker://nginx:latest \
  docker://myaccount.dkr.ecr.us-east-1.amazonaws.com/nginx:latest

# Inspect image without pulling
skopeo inspect docker://nginx:latest

# Copy from ECR to local directory in OCI format
skopeo copy \
  docker://myaccount.dkr.ecr.us-east-1.amazonaws.com/myapp:v1 \
  oci:./local-oci-dir:myapp-v1
```

---

## 🚨 Gotchas & Edge Cases

### 1. Docker Image Format ≠ OCI Image Format (but mostly compatible)
```
Docker Image Manifest v2 schema 2 → Very close to OCI, widely supported
OCI Image Spec → Stricter, pure standard

ECR supports both! But:
- docker push → docker format
- buildah push → OCI format
- Both work with ECS!
```

### 2. runc vs kata vs gVisor
```
runc     → Standard OCI runtime (containers share kernel)
kata     → OCI runtime using VMs (stronger isolation, AWS uses a variant for Fargate)
gVisor   → Google's OCI runtime with user-space kernel (GKE sandbox)

All OCI-compliant → All accept same container image format!
```

### 3. ECR OCI Artifact Storage
```bash
# ECR can store ANY OCI artifact, not just images!
# Helm charts, WASM modules, AI model weights, etc.

# Push Helm chart to ECR as OCI artifact:
helm package ./my-chart
helm push my-chart-1.0.0.tgz \
  oci://public.ecr.aws/my-namespace/my-chart
```

---

## 🎤 Interview Angle

**Q: "OCI kya hai aur ECR se kaise relate karta hai?"**

> OCI (Open Container Initiative) container ecosystem ke 3 standards define karta hai:
> 1. Image Spec (image format)
> 2. Runtime Spec (how containers run - runc implements this)
> 3. Distribution Spec (registry API)
>
> ECR OCI Distribution Spec implement karta hai, isliye koi bhi OCI-compliant tool (docker, buildah, skopeo, crane) ECR se images pull/push kar sakta hai.

**Q: "Fargate kaunsa container runtime use karta hai?"**

> Fargate AWS ka custom OCI-compliant runtime use karta hai (kata containers based, runc nahi).
> Stronger isolation milti hai — har Fargate task apni dedicated MicroVM mein chalti hai.
> OCI compliance ki wajah se aapki standard Docker images bina changes ke kaam karti hain.

---

*Next: [06_Image_Immutability_Tagging_Digest.md →](./06_Image_Immutability_Tagging_Digest.md)*
