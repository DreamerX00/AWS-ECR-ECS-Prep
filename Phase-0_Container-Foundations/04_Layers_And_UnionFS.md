# 🧅 Layers & Union Filesystem (UnionFS)

---

## 📖 Concept Explanation

Docker images **layered hoti hain** — har Dockerfile instruction ek nayi layer banata hai. Yahi layers Docker ko efficient, fast, aur storage-friendly banati hain.

**Union Filesystem (UnionFS):** Ek filesystem jo **multiple layers ko ek single directory tree** ki tarah dikhata hai.

Popular implementations:
- **OverlayFS** — Modern Linux (Docker default)
- **AUFS** — Older Ubuntu systems
- **Btrfs/ZFS** — Alternative storage backends
- **DeviceMapper** — RHEL/CentOS based

---

## 🏗️ Internal Architecture

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
                       WorkDir   (OverlayFS internal use)
```

### OverlayFS Mechanics:
```bash
# How OverlayFS mounts work (conceptually):
mount -t overlay overlay \
  -o lowerdir=layer1:layer2:layer3, \
  -o upperdir=container_writable, \
  -o workdir=work_tmp \
  /merged_view

# Docker does this automatically for each container!
```

### Layer Building Process (Dockerfile):
```dockerfile
FROM ubuntu:22.04          # Layer 1: Pull base image
RUN apt-get update         # Layer 2: Package cache
RUN apt-get install -y nginx  # Layer 3: nginx binary
COPY nginx.conf /etc/nginx/   # Layer 4: Config file
COPY ./app /var/www/html      # Layer 5: Your app
EXPOSE 80                  # Metadata (no new layer)
CMD ["nginx", "-g", "daemon off;"]  # Metadata
```

Each `RUN`, `COPY`, `ADD` = new layer with SHA256 content hash.

---

## 🎯 Analogy — Transparency Sheets (OHP) 🎦

Yaad hai school mein OHP projector? Transparent sheets hoti thi jin par likha hota tha aur unhe ek dusre ke upar rakhte the?

**Image Layers = Stacked Transparency Sheets:**
```
Sheet 1 (bottom): Base map of India (Ubuntu)
Sheet 2:          Cities marked (packages)
Sheet 3:          Roads drawn (nginx)
Sheet 4:          Your overlay (app)
─────────────────────────────────
RESULT:           Complete map with everything
```

- Bottom sheets kabhi change nahi hoti (read-only)
- Nayi sheet daal do upar (new layer)
- Hata do sheet (container delete) → baaki sab safe
- 2 alag maps ke liye base sheet 1 ek hi hoti (shared layers!)

---

## 🌍 Real-World Scenarios

### Scenario 1: Efficient CI/CD Pipeline

```
Team ke 10 services hain, sab FROM node:18-alpine se start hote hain.

Without layers: 10 different images × 100MB base = 1GB base storage
With layers:    node:18-alpine = 100MB (ONCE stored per host)
                10 apps × 10MB each = 100MB extra
                TOTAL = 200MB → 80% savings!

CI/CD mein: 
- Pipeline step 1: `docker pull` → Already cached → FAST (0 seconds)
- `package.json` change nahi → RUN npm install layer CACHED → FAST
- Sirf final COPY ./app layer rebuild hoti hai!
```

### Scenario 2: ECS Task Launch Speed
```
ECS Fargate pe task launch karte waqt:
- ECR se image pull hoti hai
- Agar base layers already host pe hain → sirf new layers pull
- Result: Task startup time: 30 sec → 8 sec (cached layers!)

AMI banana time pe:
- Pre-warm ECS hosts with common base images
- Production tasks start much faster
```

### Layer Cache Invalidation:
```dockerfile
# BAD ORDER - destroys cache benefit:
FROM node:18-alpine
COPY . .                    # Code change → yahan invalidate
RUN npm install             # Har baar re-run (slow!)
CMD ["node", "index.js"]

# GOOD ORDER - maximize cache:
FROM node:18-alpine
COPY package*.json ./       # package.json change hona rare hai
RUN npm install             # CACHED unless package.json changes
COPY . .                    # Code changes here (only)
CMD ["node", "index.js"]
```

---

## ⚙️ Hands-On Examples

### Inspect Image Layers:
```bash
# Layer hashes dekho
docker inspect nginx:latest | jq '.[0].RootFS.Layers'
# [
#   "sha256:2a92d6ac9e4f9...",  ← Layer 1 (base OS)
#   "sha256:71a1f2ac59b0e...",  ← Layer 2 (nginx install)
#   "sha256:a0a227bf03ddc..."   ← Layer 3 (config)
# ]

# Size breakdown by layer
docker history nginx:latest --no-trunc
```

### OverlayFS at Host Level:
```bash
# Run a container
docker run -d --name overlay-demo nginx

# Find overlay mount
docker inspect overlay-demo | jq '.[0].GraphDriver.Data'
# {
#   "LowerDir": "/var/lib/docker/overlay2/abc.../diff:...",
#   "MergedDir": "/var/lib/docker/overlay2/abc.../merged",
#   "UpperDir": "/var/lib/docker/overlay2/abc.../diff",  ← writable
#   "WorkDir": "/var/lib/docker/overlay2/abc.../work"
# }

# See the actual filesystem
sudo ls /var/lib/docker/overlay2/

# Write something in container
docker exec overlay-demo sh -c "echo 'test' > /container-file"

# Find it in UpperDir (writable layer)
UPPER=$(docker inspect overlay-demo | jq -r '.[0].GraphDriver.Data.UpperDir')
sudo cat $UPPER/container-file  # 'test' is here!
```

### Layer Caching in Action:
```bash
# Build with timing
time docker build -t myapp:v1 .
# [+] Building 45.2s (8/8) DONE

# Change only app code (not package.json)
echo "// minor change" >> src/index.js

# Rebuild - watch cache hits!
time docker build -t myapp:v2 .
# [+] Building 2.1s (8/8) DONE   ← MUCH FASTER!
# Step 5/8 : COPY package.json .
#  ---> Using cache               ← CACHED!
# Step 6/8 : RUN npm install
#  ---> Using cache               ← CACHED!
# Step 7/8 : COPY . .
#  ---> Running... (only this ran)
```

### Deduplicate Image Storage:
```bash
# Docker system df - see actual disk usage
docker system df
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          15        8         3.6GB     1.2GB (33%)
# Containers      8         5         124kB     0B

# Prune unused layers
docker system prune --volumes

# Remove dangling images (untagged intermediate layers)
docker image prune
```

---

## 🚨 Gotchas & Edge Cases

### 1. Secrets in Layers = Security Breach!
```dockerfile
# DANGEROUS: Secret permanently in image layer!
FROM ubuntu:22.04
RUN echo "DB_PASSWORD=supersecret" > /etc/app.env
# Even if you DELETE it in next layer, it exists in PREVIOUS layer!

# Safe approach:
ENV DB_PASSWORD=${DB_PASSWORD}  # Pass at runtime
# Or use ARG (build-time, not stored in final image):
ARG DB_PASSWORD
RUN echo "build uses password" && unset DB_PASSWORD
```

### 2. Layer Bloat — Wrong RUN Commands:
```dockerfile
# BAD: apt cache stored in layer!
RUN apt-get update
RUN apt-get install -y nginx

# GOOD: Clean up in SAME layer!
RUN apt-get update && \
    apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/*
    
# Each RUN = 1 layer. Combine related commands!
```

### 3. Large COPY Layers:
```bash
# .dockerignore use karo! (like .gitignore for Docker)
echo "node_modules/
.git/
*.log
dist/
.env*" > .dockerignore

# Without .dockerignore: COPY . . might include gigabytes of node_modules!
```

### 4. Layer Order Optimization for ECR Pulls:
```dockerfile
# Put LEAST-frequently-changing things FIRST
FROM python:3.11-slim       # Changes almost never
RUN pip install -r requirements-base.txt  # Changes rarely
COPY requirements.txt .     # Changes sometimes
RUN pip install -r requirements.txt       # Changes when ^ changes
COPY . .                    # Changes every commit
```

### 5. Maximum Layer Limit:
Docker has a soft limit of **127 layers**. Multi-stage builds help avoid hitting this.

---

## 🎤 Interview Angle

**Q: "Cache invalidation Docker mein kaise kaam karta hai? Kaise optimize karein?"**

> Dockerfile mein har instruction ka output hash calculate hota hai.
> Agar koi instruction change hoti hai → us point ke baad SAARI layers invalidate.
> Optimization: Stable/slow-changing content pehle rakho (base image, package manifests).
> Fast-changing content baad mein (app code).
> Result: `npm install` sirf `package.json` change par chale, har commit pe nahi.

**Q: "Secret kaise Docker image mein leak ho jaata hai?"**

> Agar pehle ki kisi layer mein secret tha aur baad ki layer mein delete kiya →
> Secret still exists in that intermediate layer's filesystem.
> `docker history --no-trunc` se extract kar sakte hain.
> Solution: Build-time secrets (`--secret`), runtime env injection (ECS task definition secrets), ya multi-stage build.

---

*Next: [05_OCI_Standard.md →](./05_OCI_Standard.md)*
