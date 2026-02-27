# OCI Standard — Open Container Initiative

---

## Concept Explanation

**OCI (Open Container Initiative)** is an **open governance structure** that defines standards for container technology. It was founded in 2015 by Docker, CoreOS, and others under the Linux Foundation.

**The problem OCI solves:** Before OCI, Docker had its own proprietary image format and runtime specification. If you built Docker images, they could only run on Docker Engine. Kubernetes, CoreOS, Red Hat, and other ecosystem players had to build their own incompatible container implementations — creating fragmentation across the entire industry.

**OCI's solution:** Define open, vendor-neutral standards so that any conforming tool can build, store, and run containers regardless of who made the runtime.

### OCI's Three Specifications:

| Specification | What It Defines |
|------|---------------------|
| **Image Spec** | How a container image is structured (manifest format, layer packaging) |
| **Runtime Spec** | How a container is executed (namespaces, cgroups, mounts, process config) |
| **Distribution Spec** | How images are pushed and pulled from a registry (the HTTP API contract) |

---

## Internal Architecture

### OCI Image Format:
```
OCI Image (stored in a registry like ECR)
├── index.json              ← Entry point (supports multi-arch via manifest lists)
├── oci-layout              ← OCI version marker file
└── blobs/
    └── sha256/
        ├── <manifest-digest>    ← Image manifest
        │     ├── config digest  → Points to the image config
        │     └── layers[]       → Points to each layer blob
        ├── <config-digest>      ← Config (env vars, cmd, entrypoint, labels)
        └── <layer1-digest>      ← Layer 1 as a compressed tar archive (tar.gz)
        └── <layer2-digest>      ← Layer 2 as a compressed tar archive
```

Every piece of content in an OCI image is identified by its SHA256 digest, making the entire image a content-addressed, immutable artifact.

### OCI Runtime Spec (what runc implements):
```json
// config.json — the complete container specification
{
  "ociVersion": "1.0.2",
  "process": {
    "args": ["nginx", "-g", "daemon off;"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin"],
    "user": {"uid": 0, "gid": 0}
  },
  "root": {
    "path": "rootfs",   // Container filesystem root directory
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

This `config.json` is the complete contract between a container image and a runtime. Any OCI-compliant runtime (runc, kata, gVisor) can read this file and produce a running container from it.

### OCI Distribution Spec — Registry API:
```
GET  /v2/                              ← Registry API version check
GET  /v2/<name>/tags/list              ← List all tags in a repository
GET  /v2/<name>/manifests/<reference>  ← Get image manifest by tag or digest
GET  /v2/<name>/blobs/<digest>         ← Download a specific layer blob
PUT  /v2/<name>/blobs/uploads/         ← Initiate a layer upload
PUT  /v2/<name>/manifests/<reference>  ← Push a completed manifest
```

Any client tool that implements this API (Docker CLI, buildah, skopeo, crane) can interoperate with any registry that implements it (ECR, Docker Hub, GCR, GHCR).

---

## Analogy — USB Standard

Before USB, every device manufacturer had its own proprietary connector:
- Dell laptops had one charging port format
- HP had a different one
- Sony had another entirely different one

Then USB was standardized — one connector format that works everywhere, regardless of manufacturer.

**OCI is the USB standard for containers:**
- Before OCI: Docker containers only ran on Docker Engine
- After OCI: A single OCI-compliant image runs anywhere

```
OCI Image (standardized format)
      │
      ├── AWS ECS Fargate      ✅ Runs
      ├── Google Kubernetes    ✅ Runs
      ├── Azure Container Inst ✅ Runs
      ├── docker run (locally) ✅ Runs
      └── runc directly        ✅ Runs
```

---

## Real-World Scenarios

### Scenario 1: Avoiding Vendor Lock-in

```
A company starts on AWS ECS.
Later, leadership decides to adopt a multi-cloud strategy.

PRE-OCI WORLD:
  Docker-specific proprietary format → Must rewrite for Google Cloud
  Estimated cost: 6 months of engineering, $500K+

OCI WORLD:
  The same OCI-compliant images work everywhere
  The only change needed is the deployment target configuration
  Estimated cost: 2 weeks, minimal engineering work
```

### Scenario 2: Custom Container Runtimes (AWS Fargate)

```
AWS Fargate uses a custom OCI-compliant runtime internally (not standard runc).
Your standard Docker image works on Fargate because of OCI compliance:

1. Your image follows OCI Image Spec ✅
2. Fargate's runtime implements OCI Runtime Spec ✅
3. ECR implements OCI Distribution Spec ✅

You never need to know anything about Fargate's internal runtime implementation.
OCI defines the contract — both sides honor it.
```

### Scenario 3: Multi-architecture Images

```bash
# OCI Image Index supports one tag pointing to multiple architecture-specific manifests
docker buildx build --platform linux/amd64,linux/arm64 \
  --tag myrepo/myapp:latest \
  --push .

# ECR stores both manifests under the same tag.
# When pulling:
# - EC2 (x86_64) → runtime selects the amd64 manifest
# - AWS Graviton (arm64) → runtime selects the arm64 manifest
# Same image tag, correct binary for each architecture automatically
```

This is entirely enabled by the OCI Image Index specification, which allows a single tag to point to a list of platform-specific manifests.

---

## Hands-On Examples

### Explore OCI Image Structure Locally:
```bash
# Save an image as a tar archive
docker save nginx:latest -o nginx.tar
mkdir nginx-oci && tar xf nginx.tar -C nginx-oci/
ls nginx-oci/
# manifest.json  layers/  config/

# Read the manifest
cat nginx-oci/manifest.json | jq .

# Read the image config (entrypoint, cmd, env, labels)
cat nginx-oci/config.json | jq .

# Each layer is a gzip-compressed tar archive
# You can extract a layer to inspect its filesystem contents:
mkdir layer1 && tar xzf nginx-oci/<layer-sha256>.tar.gz -C layer1/
ls layer1/  # Actual filesystem content of that layer
```

### OCI with Buildah (Build Without a Docker Daemon):
```bash
# Buildah is an OCI-native build tool that requires no running daemon
buildah from ubuntu:22.04
buildah run ubuntu-working-container -- apt-get update
buildah run ubuntu-working-container -- apt-get install -y nginx
buildah commit ubuntu-working-container my-nginx:latest

# Push the OCI image directly to ECR
buildah push my-nginx:latest \
  aws_account_id.dkr.ecr.region.amazonaws.com/my-repo:latest
```

Buildah is commonly used in Kubernetes environments where running a Docker daemon inside a pod is undesirable for security reasons.

### Skopeo — Registry Operations Without Docker:
```bash
# Copy an image between registries without a Docker daemon or local storage
skopeo copy \
  docker://nginx:latest \
  docker://myaccount.dkr.ecr.us-east-1.amazonaws.com/nginx:latest

# Inspect an image manifest without pulling the full image
skopeo inspect docker://nginx:latest

# Copy from ECR to a local directory in OCI format for offline inspection
skopeo copy \
  docker://myaccount.dkr.ecr.us-east-1.amazonaws.com/myapp:v1 \
  oci:./local-oci-dir:myapp-v1
```

---

## Gotchas & Edge Cases

### 1. Docker Image Format vs OCI Image Format
```
Docker Image Manifest v2 schema 2 → Very close to OCI spec, widely compatible
OCI Image Spec                     → Stricter standard, purpose-built for interoperability

ECR supports both formats.
- docker push → produces Docker manifest v2 format
- buildah push → produces OCI format
- Both work correctly with ECS and most container runtimes
```

In practice, the formats are nearly identical and most tools handle both transparently. However, if you are building for strict OCI compliance (e.g., for a multi-runtime environment), using buildah or `docker buildx` with explicit OCI output is preferred.

### 2. OCI Runtime Variants — runc vs Kata vs gVisor
```
runc   → Standard OCI runtime; containers share the host kernel
         Fast, low overhead, widely used

kata   → OCI runtime that uses lightweight VMs for each container
         Stronger isolation; AWS uses a variant of this for Fargate tasks
         Slower startup, higher overhead, better security boundaries

gVisor → Google's OCI runtime with a user-space kernel intercept layer
         Used in GKE Sandbox mode; intercepts syscalls before they reach the host kernel
         Provides strong isolation with lower overhead than full VMs
```

All three are OCI-compliant, so the same container image runs on all of them without modification.

### 3. ECR as a General-Purpose OCI Artifact Store
```bash
# ECR can store ANY OCI artifact, not just container images.
# The OCI Distribution Spec is content-agnostic.

# Push a Helm chart to ECR as an OCI artifact:
helm package ./my-chart
helm push my-chart-1.0.0.tgz \
  oci://public.ecr.aws/my-namespace/my-chart

# This enables a single registry to store:
# - Container images
# - Helm charts
# - WebAssembly modules
# - AI model weights
# - Any arbitrary artifact that follows the OCI artifact layout
```

### 4. OCI and Kubernetes CRI
The Container Runtime Interface (CRI) is Kubernetes' internal API for communicating with OCI-compliant runtimes. When Kubernetes schedules a pod, the kubelet uses CRI to tell containerd (or another CRI-compatible runtime) to pull the OCI image and start the container. Docker itself is not CRI-compatible, which is why Dockershim was deprecated in Kubernetes v1.24.

---

## Interview Angle

**Q: "What is OCI and how does it relate to ECR?"**

> OCI (Open Container Initiative) defines three open standards for the container ecosystem:
> 1. **Image Spec** — how container images are structured and stored (layered tar archives with a JSON manifest)
> 2. **Runtime Spec** — how a container is executed (defines namespaces, cgroups, mounts, process arguments)
> 3. **Distribution Spec** — the HTTP API contract for pushing and pulling images from a registry
>
> ECR implements the OCI Distribution Spec, which means any OCI-compliant client (Docker CLI, buildah, skopeo, crane, Helm) can push and pull from ECR using the standard registry API. You are not locked into the Docker CLI.

**Q: "What container runtime does AWS Fargate use?"**

> Fargate uses a custom AWS-managed OCI-compliant runtime built on technology similar to Kata Containers. Each Fargate task runs inside a dedicated MicroVM (using AWS Firecracker), providing much stronger isolation than containers sharing a host kernel. Because the runtime is OCI-compliant, standard Docker images built with any OCI-compliant tool work on Fargate without modification.

**Q: "What is the difference between runc and containerd?"**

> runc is the low-level OCI runtime — it receives a rootfs directory and a config.json file, makes the necessary Linux syscalls (clone, setrlimit, pivot_root), and starts the container process. It is a short-lived binary that exits after starting the container.
> containerd is the high-level runtime that sits above runc. It manages the full lifecycle of containers (create, start, stop, delete), handles image pulling and layer extraction, and exposes a gRPC API. containerd delegates the actual container creation to runc via OCI calls.

---

*Next: [06_Image_Immutability_Tagging_Digest.md →](./06_Image_Immutability_Tagging_Digest.md)*
