# Image vs Container — The Core Distinction

---

## Concept Explanation

This distinction is the most common source of confusion in Docker interviews. The relationship is analogous to a class and an instance in object-oriented programming.

| | **Docker Image** | **Docker Container** |
|--|--|--|
| Definition | Blueprint / Template | Running instance of an image |
| State | Immutable (read-only) | Mutable (writable layer on top) |
| Storage | Disk only | Memory + thin writable layer on disk |
| Multiplicity | One image → many containers | Each container is an independent instance |
| Lifecycle | Built once, stored indefinitely | Created, started, stopped, deleted |

An image is never "running". A container is the live execution of an image with its own writable state, network identity, and process tree.

---

## Internal Architecture

### Image = Stack of Read-Only Layers
```
┌─────────────────────────────┐
│   App Layer (your code)     │  ← COPY . /app  (Layer 4)
├─────────────────────────────┤
│   Dependency Layer          │  ← RUN npm install (Layer 3)
├─────────────────────────────┤
│   Package Manifest Layer    │  ← COPY package.json (Layer 2)
├─────────────────────────────┤
│   Base OS Layer             │  ← FROM node:18-alpine (Layer 1)
└─────────────────────────────┘
        ALL READ-ONLY (Image)
```

### Container = Image + Writable Layer (Copy-on-Write)
```
┌─────────────────────────────┐
│   WRITABLE LAYER (CoW)      │  ← Container-specific changes
├═════════════════════════════╡
│   App Layer (READ-ONLY)     │
├─────────────────────────────┤
│   Dependency Layer (R/O)    │  ← Shared across ALL containers
├─────────────────────────────┤
│   Package Layer (R/O)       │     using the same image
├─────────────────────────────┤
│   Base OS Layer (R/O)       │
└─────────────────────────────┘
```

**Copy-on-Write (CoW):** When a container modifies a file from the read-only image layers:
1. The runtime locates the original file in the read-only layer
2. Copies that file up into the container's **writable layer**
3. The modification is applied only in the writable layer
4. The original read-only layer remains **completely unchanged**

This means the image layers can be safely shared by hundreds of containers simultaneously — each container only pays storage cost for the files it actually modifies.

---

## Analogy — Blueprint vs Constructed Building

**Image = Architectural Blueprint:**
- The same blueprint can produce hundreds of identical buildings
- The blueprint itself is never modified
- The blueprint specifies all materials and dimensions (layers)

**Container = An Actual Building:**
- Constructed from the blueprint
- Tenants move in and make changes (furniture, wall paint — the writable layer)
- Demolishing one building does not affect the blueprint or other buildings
- Ten different tenants, ten independent buildings — all from the same blueprint

```
docker build -t my-app .    # Create the blueprint (image)
docker run my-app            # Construct building (instance 1)
docker run my-app            # Construct building (instance 2)
docker run my-app            # Construct building (instance 3)
# Three independent containers, one shared image
```

---

## Real-World Scenario

### Production Scale: E-commerce Flash Sale

During a high-traffic event:
- 1 image: `ecommerce-backend:v2.1.4`
- 50 containers running from this single image simultaneously
- Each container has its own session data and logs stored in its writable layer
- Total image storage on disk: 800MB (stored once per host, regardless of how many containers run)
- Total container overhead: 50 × ~10MB CoW layers = 500MB
- **Without layer sharing:** 50 × 800MB = 40GB of redundant storage

```bash
# Scale up before the flash sale
docker service scale ecommerce-backend=50

# All 50 containers reference the same underlying image layers
docker inspect ecommerce-backend --format '{{.RootFS.Layers}}'
# All 50 containers share the same 6 layer hashes
```

This storage efficiency directly translates to cost savings in ECR (you store one image, not one copy per running container) and to faster ECS task launches (cached layers are not re-downloaded).

### Inspect Real Layer Sharing:
```bash
# Build two similar images
docker build -t app:v1 .
docker build -t app:v2 .  # Only the last layer changed

# Compare layer IDs
docker history app:v1
docker history app:v2
# Layers with identical hashes are shared on disk — Docker deduplicates automatically
```

---

## Hands-On Examples

### Image Deep Inspection:
```bash
# View all layers with their sizes
docker history nginx:latest
# IMAGE          CREATED         CREATED BY                                      SIZE
# f9c14fe76d50   2 weeks ago     /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B
# <missing>      2 weeks ago     /bin/sh -c #(nop)  EXPOSE 80                    0B
# <missing>      2 weeks ago     /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT           0B
# ...

# Inspect image metadata including layer digests
docker inspect nginx:latest | jq '.[0].RootFS'

# List local images with sizes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### Container Lifecycle — Full State Machine:
```bash
# Pull the image
docker pull nginx:latest

# Create a container (does not start it)
docker create --name test-nginx -p 8080:80 nginx:latest

# Start the container
docker start test-nginx

# Stop the container (sends SIGTERM, waits, then SIGKILL)
docker stop test-nginx

# Delete the container (writable layer is permanently destroyed)
docker rm test-nginx

# The image still exists:
docker images | grep nginx  # Still present

# Shorthand: create + start in one step
docker run -d --name test-nginx -p 8080:80 nginx
```

### Inspect the Container's Writable Layer:
```bash
# Start a container and write a file inside it
docker run -d --name demo nginx
docker exec demo sh -c "echo 'hello' > /tmp/myfile.txt"

# Find the writable layer path on the host
docker inspect demo | jq '.[0].GraphDriver'
# Shows UpperDir — this is the writable layer on the host filesystem

UPPER=$(docker inspect demo | jq -r '.[0].GraphDriver.Data.UpperDir')
ls $UPPER/tmp/
# myfile.txt is visible here on the host!

# Delete the container — the writable layer is gone
docker rm -f demo
ls $UPPER/  # Directory no longer exists
```

---

## Gotchas & Edge Cases

### 1. Container Deletion Means Permanent Data Loss
```bash
# NEVER run a database container without a volume:
docker run -d mysql:8  # No volume mounted!
# MySQL stores data at /var/lib/mysql — inside the writable layer
# When this container is deleted, ALL database data is gone permanently!

# Correct approach:
docker run -d -v mysql-data:/var/lib/mysql mysql:8
# Named volume persists independently of the container lifecycle
```

### 2. `docker commit` — An Anti-Pattern
```bash
# Enter a running container and make changes
docker exec -it myapp bash
apt-get install -y vim  # Changes accumulate in the writable layer

# Commit the writable layer to create a new image
docker commit myapp myapp:with-vim
# This is an anti-pattern!
# The result is not reproducible, not auditable, and not version-controlled.
# All changes should go through a Dockerfile.
```

The `docker commit` workflow produces images that no one can rebuild from source. If the container is lost, the changes are unrecoverable. Always use Dockerfiles.

### 3. Image Size Directly Affects ECR Costs and ECS Task Start Time
```bash
# Problematic: Large monolithic layer
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3 && pip install -r requirements.txt && ...

# Better: Minimal base image with multi-stage build
FROM python:3.11-slim
# A 50MB final image vs a 1.2GB image = lower ECR storage costs,
# faster image pulls, and significantly faster ECS task cold starts
```

### 4. Containers Run as Root by Default
```bash
# The default behavior is to run as root inside the container:
docker run alpine whoami  # root

# This is a security risk — a process escape gives root access
# Always specify a non-root user in your Dockerfile:
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

Running as root inside a container increases the impact of any container escape vulnerability. AWS ECS task definitions also support specifying a non-root user at the task level.

### 5. `docker run` vs `docker start`
A common confusion: `docker run` always creates a **new** container. `docker start` resumes an **existing stopped** container, preserving its writable layer and state. If you `docker run` the same image twice, you get two completely separate containers with independent writable layers.

---

## Interview Angle

**Q: "What is the difference between a Docker image and a container? How do multiple containers share one image?"**

> An image is an immutable, layered filesystem stored on disk — it is the read-only blueprint.
> A container is a running instance of that image with a thin writable layer added on top (Copy-on-Write model).
> Multiple containers can share the same image layers because those layers are never modified — any writes go to the container's own isolated writable layer.
> This is why 100 containers from the same image cost almost no additional disk space — only each container's unique changes are stored separately.

**Q: "Why is data lost when a container is deleted?"**

> All writes made inside a container go into its writable layer, which is stored on the host as an OverlayFS upper directory.
> When the container is deleted with `docker rm`, that writable layer is permanently removed.
> To persist data across container lifecycles, use Docker Volumes (which exist independently of any container) or in AWS ECS, use EFS-backed volumes mounted into the task.

**Q: "Is it safe to use `docker commit` in a CI/CD pipeline?"**

> No. `docker commit` produces an image that is not reproducible from source. There is no record of what commands were run, what packages were installed, or what files were changed. It bypasses all the auditability that Dockerfiles provide. In a security audit, you cannot verify what is inside a committed image without inspecting every layer manually. Always use Dockerfiles.

---

*Next: [04_Layers_And_UnionFS.md →](./04_Layers_And_UnionFS.md)*
