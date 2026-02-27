# Layers & Union Filesystem (UnionFS)

---

## Concept Explanation

Docker images are **composed of layers** — every instruction in a Dockerfile that produces a filesystem change creates a new layer. These layers are what make Docker efficient, fast, and storage-friendly.

**Union Filesystem (UnionFS):** A class of filesystem that presents **multiple distinct directories as a single unified directory tree**. When the same path exists in multiple layers, the topmost layer wins.

Popular implementations used by Docker:
- **OverlayFS** — Default on modern Linux kernels (Docker's current default)
- **AUFS** — Used on older Ubuntu systems (now largely deprecated)
- **Btrfs / ZFS** — Alternative storage backends with their own advantages
- **DeviceMapper** — Used on RHEL/CentOS-based systems

---

## Internal Architecture

### OverlayFS — How Docker Actually Works

```
  USER SEES:           REALITY:

  /                    UpperDir  (writable - container changes)
  ├── etc/             ──────────────────────────────
  │   └── nginx.conf   LowerDir3 (app layer)
  ├── app/             LowerDir2 (deps layer)
  │   └── main.js      LowerDir1 (base OS layer)
  └── bin/             ──────────────────────────────
      └── bash         MergedDir (unified view = what you see)
                       WorkDir   (OverlayFS internal scratch space)
```

The container process sees only the MergedDir — a seamless unified filesystem. All complexity is handled transparently by the kernel.

### OverlayFS Mechanics:
```bash
# How OverlayFS mounts work (conceptually):
mount -t overlay overlay \
  -o lowerdir=layer1:layer2:layer3, \
  -o upperdir=container_writable, \
  -o workdir=work_tmp \
  /merged_view

# Docker performs this mount automatically for each container.
```

### Layer Building Process (Dockerfile):
```dockerfile
FROM ubuntu:22.04          # Layer 1: Pull base image
RUN apt-get update         # Layer 2: Package cache update
RUN apt-get install -y nginx  # Layer 3: nginx binary added
COPY nginx.conf /etc/nginx/   # Layer 4: Config file added
COPY ./app /var/www/html      # Layer 5: Application files
EXPOSE 80                  # Metadata only (no new layer created)
CMD ["nginx", "-g", "daemon off;"]  # Metadata only
```

Each `RUN`, `COPY`, and `ADD` instruction produces a new layer identified by its SHA256 content hash. Metadata instructions (`EXPOSE`, `CMD`, `ENV`) are stored in the image config but do not create filesystem layers.

---

## Analogy — Stacked Transparency Sheets

Think of an overhead projector with a stack of transparent sheets:

**Image Layers = Stacked Transparency Sheets:**
```
Sheet 1 (bottom): Base map of a country (Ubuntu base OS)
Sheet 2:          Cities and towns marked (installed packages)
Sheet 3:          Roads and highways (nginx installed)
Sheet 4:          Your own custom overlay (application code)
─────────────────────────────────────────────────────────
RESULT:           Complete map with all information visible
```

- The lower sheets are never modified — they are read-only
- Adding a new sheet adds a new layer without touching existing ones
- Removing the top sheet (deleting a container) leaves the others intact
- Two different maps can share the same base sheet — this is layer sharing

---

## Real-World Scenarios

### Scenario 1: Efficient CI/CD Pipeline

```
A team has 10 microservices, all starting FROM node:18-alpine.

Without layer sharing: 10 images × 100MB base = 1GB of base OS storage
With layer sharing:    node:18-alpine = 100MB (stored ONCE per host)
                       10 apps × ~10MB each = 100MB of unique content
                       TOTAL = ~200MB — an 80% reduction in storage

In CI/CD:
- docker pull → base layer already cached → 0 seconds to download
- package.json unchanged → RUN npm install layer is CACHED → fast build
- Only the final COPY ./app layer needs to be rebuilt on each commit
```

### Scenario 2: ECS Task Launch Speed
```
When launching an ECS Fargate task:
- ECR delivers the image layer by layer
- If base layers are already present on the host → only new layers are downloaded
- Result: Task cold start drops from ~30 seconds → ~8 seconds with warm cache

Production optimization:
- Pre-warm ECS hosts with common base images (node:18-alpine, python:3.11-slim)
- This is achievable with EC2 launch templates and user data scripts
- ECS capacity providers with managed scaling also benefit from this
```

### Layer Cache Invalidation:
```dockerfile
# BAD ORDERING — destroys cache efficiency:
FROM node:18-alpine
COPY . .                    # Any code change invalidates HERE
RUN npm install             # Re-runs on every single commit (slow!)
CMD ["node", "index.js"]

# GOOD ORDERING — maximizes cache reuse:
FROM node:18-alpine
COPY package*.json ./       # Only changes when dependencies change
RUN npm install             # CACHED unless package.json changes
COPY . .                    # Code changes land here only
CMD ["node", "index.js"]
```

The principle is: place instructions that change least frequently at the top, and instructions that change most frequently (application code) at the bottom.

---

## Hands-On Examples

### Inspect Image Layers:
```bash
# View layer content hashes
docker inspect nginx:latest | jq '.[0].RootFS.Layers'
# [
#   "sha256:2a92d6ac9e4f9...",  ← Layer 1 (base OS)
#   "sha256:71a1f2ac59b0e...",  ← Layer 2 (nginx install)
#   "sha256:a0a227bf03ddc..."   ← Layer 3 (config)
# ]

# View size breakdown by layer
docker history nginx:latest --no-trunc
```

### OverlayFS at the Host Level:
```bash
# Run a container
docker run -d --name overlay-demo nginx

# Inspect the OverlayFS mount paths
docker inspect overlay-demo | jq '.[0].GraphDriver.Data'
# {
#   "LowerDir": "/var/lib/docker/overlay2/abc.../diff:...",
#   "MergedDir": "/var/lib/docker/overlay2/abc.../merged",
#   "UpperDir": "/var/lib/docker/overlay2/abc.../diff",  ← writable layer
#   "WorkDir": "/var/lib/docker/overlay2/abc.../work"
# }

# Explore the actual filesystem on the host
sudo ls /var/lib/docker/overlay2/

# Write something inside the container
docker exec overlay-demo sh -c "echo 'test' > /container-file"

# Find it in the UpperDir (writable layer) on the host
UPPER=$(docker inspect overlay-demo | jq -r '.[0].GraphDriver.Data.UpperDir')
sudo cat $UPPER/container-file  # Outputs 'test' — visible on the host!
```

### Layer Caching in Action:
```bash
# First build — all layers are built from scratch
time docker build -t myapp:v1 .
# [+] Building 45.2s (8/8) DONE

# Change only application code (not package.json)
echo "// minor change" >> src/index.js

# Rebuild — observe cache hits
time docker build -t myapp:v2 .
# [+] Building 2.1s (8/8) DONE   ← Dramatically faster!
# Step 5/8 : COPY package.json .
#  ---> Using cache               ← CACHED
# Step 6/8 : RUN npm install
#  ---> Using cache               ← CACHED
# Step 7/8 : COPY . .
#  ---> Running...                ← Only this step ran
```

### Disk Usage and Cleanup:
```bash
# Show actual disk usage including shared layer deduplication
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          15        8         3.6GB     1.2GB (33%)
# Containers      8         5         124kB     0B

# Remove all stopped containers, unused networks, dangling images
docker system prune --volumes

# Remove only dangling images (untagged intermediate layers)
docker image prune
```

---

## Gotchas & Edge Cases

### 1. Secrets Written Into Layers Are Permanently Exposed
```dockerfile
# DANGEROUS: Secret is permanently baked into an image layer!
FROM ubuntu:22.04
RUN echo "DB_PASSWORD=supersecret" > /etc/app.env
RUN rm /etc/app.env   # This does NOT remove it from the previous layer!

# The secret exists in the layer created by the first RUN instruction.
# Anyone with access to the image can extract it with:
# docker run --rm -it <image> sh  (from the intermediate layer)
# or: docker history --no-trunc <image>

# Safe alternatives:
# 1. Pass secrets at runtime via environment variables (ECS task definition secrets)
# 2. Use BuildKit's --secret mount (not stored in any layer):
#    RUN --mount=type=secret,id=db_password cat /run/secrets/db_password
# 3. Use AWS Secrets Manager and fetch at runtime inside the container
```

### 2. Layer Bloat from Incorrect RUN Commands
```dockerfile
# BAD: The apt cache is preserved in its own layer
RUN apt-get update
RUN apt-get install -y nginx

# GOOD: Update, install, and clean up in a single layer
RUN apt-get update && \
    apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/*

# Each RUN instruction creates exactly one layer.
# Always combine related commands and clean up in the same instruction.
```

### 3. Missing .dockerignore Inflates COPY Layers
```bash
# Create a .dockerignore file (works like .gitignore for the build context)
cat > .dockerignore << 'EOF'
node_modules/
.git/
*.log
dist/
.env*
coverage/
.DS_Store
EOF

# Without .dockerignore, COPY . . might include hundreds of megabytes
# of node_modules, git history, or build artifacts that should never
# be in the image.
```

### 4. Layer Order Optimization for ECR Pull Performance
```dockerfile
# Arrange layers from least-frequently-changing to most-frequently-changing
FROM python:3.11-slim                            # Changes almost never
COPY requirements-base.txt .
RUN pip install -r requirements-base.txt         # Changes rarely
COPY requirements.txt .
RUN pip install -r requirements.txt              # Changes when deps change
COPY . .                                         # Changes on every commit
```

Layers that change rarely become stable cache entries on ECS hosts, reducing pull time for every new deployment.

### 5. Maximum Layer Limit
Docker has a practical limit of **127 layers** per image. Deep chains of `RUN` instructions in older Dockerfiles can approach this limit. Multi-stage builds are the cleanest solution — the final stage starts fresh and only copies in the artifacts it needs.

---

## Interview Angle

**Q: "How does Docker layer caching work, and how do you optimize for it?"**

> Docker computes a cache key for each Dockerfile instruction based on the instruction itself and the content hash of any files it references.
> If the cache key matches a previously built layer, Docker reuses that layer without re-executing the instruction.
> A cache miss at any layer invalidates the cache for **all subsequent layers** in that build.
> Optimization strategy: put slow, stable operations (package installs) before fast, volatile operations (application code). Copy only the dependency manifest first, install dependencies, then copy the full source tree. This way `npm install` or `pip install` only re-runs when the dependency file actually changes, not on every code commit.

**Q: "How can a secret accidentally end up in a Docker image?"**

> Any data written to the filesystem during a `RUN`, `COPY`, or `ADD` instruction is permanently embedded in that layer's content-addressed blob.
> Deleting the file in a subsequent layer does not remove it from the previous layer — both layers are stored in the registry.
> Anyone with image pull access can extract the secret by inspecting intermediate layers.
> The correct approaches are: use BuildKit `--secret` mounts (which are never stored in any layer), inject secrets at runtime via environment variables or AWS Secrets Manager, or use multi-stage builds where secrets are used only in a build stage that is discarded.

**Q: "What is OverlayFS and what are its four directories?"**

> OverlayFS is the Linux union filesystem used by Docker to merge multiple read-only image layers with a single writable container layer.
> - **LowerDir**: The read-only image layers stacked together
> - **UpperDir**: The writable container layer where all modifications go
> - **MergedDir**: The unified view that the container process sees — what appears as the root filesystem inside the container
> - **WorkDir**: Internal scratch space used by OverlayFS during copy-on-write operations (not directly user-visible)

---

*Next: [05_OCI_Standard.md →](./05_OCI_Standard.md)*
