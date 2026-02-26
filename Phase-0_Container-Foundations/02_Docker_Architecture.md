# 🐳 Docker Architecture

---

## 📖 Concept Explanation

Docker ek **client-server architecture** follow karta hai. Sirf `docker run nginx` likhne ke peeche kaafi saari components kaam kar rahi hain.

### Core Components:
1. **Docker CLI** — User interface (command line)
2. **Docker Daemon (dockerd)** — Backend server
3. **containerd** — Container lifecycle manager
4. **runc** — OCI-compliant container runtime
5. **Docker Registry** — Image storage (ECR, Docker Hub)

---

## 🏗️ Internal Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     USER MACHINE                            │
│                                                             │
│   ┌──────────────┐        REST API        ┌─────────────┐  │
│   │  Docker CLI  │ ──────────────────────► │  dockerd    │  │
│   │  (docker)    │   /var/run/docker.sock  │  (Daemon)   │  │
│   └──────────────┘                         └──────┬──────┘  │
│                                                   │         │
│                                            gRPC   │         │
│                                                   ▼         │
│                                          ┌──────────────┐   │
│                                          │ containerd   │   │
│                                          │ (High-level  │   │
│                                          │  runtime)    │   │
│                                          └──────┬───────┘   │
│                                                 │           │
│                                      OCI Calls  │           │
│                                                 ▼           │
│                                          ┌──────────────┐   │
│                                          │    runc      │   │
│                                          │ (Low-level   │   │
│                                          │  runtime)    │   │
│                                          └──────┬───────┘   │
│                                                 │           │
│                                     Namespaces/ │           │
│                                     cgroups     ▼           │
│                                         ┌─────────────┐     │
│                                         │  CONTAINER  │     │
│                                         │  (Process)  │     │
│                                         └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
         │
         │ docker push / pull
         ▼
┌─────────────────┐
│  Docker Registry │
│  (ECR/Hub)       │
└─────────────────┘
```

### Layer-by-Layer Explanation:

#### 🔵 Docker CLI
```bash
docker run -d -p 80:80 nginx
```
Yeh sirf ek **REST API call** karta hai Docker Daemon ko. CLI khud koi kaam nahi karta.

#### 🔵 Docker Daemon (dockerd)
- Linux pe `/usr/bin/dockerd` binary
- Manages: images, containers, networks, volumes
- Socket location: `/var/run/docker.sock`
- Controls **containerd** via gRPC

#### 🔵 containerd
- **Container lifecycle management:** create, start, stop, delete
- Image management: pull, push, extract layers
- **Kata containers** aur dusre runtimes bhi yahan plug-in hote hain
- Kubernetes bhi directly containerd use karta hai (Docker bypass karke!)

#### 🔵 runc
- Actual container **banane ka kaam** karta hai
- Linux syscalls karta hai: `clone()` (namespace), `setrlimit()` (cgroup)
- OCI Runtime Specification follow karta hai
- `runc run my-container` = container start

---

## 🎯 Analogy — Restaurant Kitchen 🍳

```
Customer (You)          → Docker CLI
Head Waiter             → Docker Daemon (receives orders)
Kitchen Manager         → containerd (coordinates cooking)
Individual Chef         → runc (actually cooks the dish)
Recipe Book             → Docker Image
Finished Plate of Food  → Running Container
Ingredient Store        → Docker Registry (ECR)
```

Jab customer order karta hai:
1. Waiter (CLI) → Manager ko batata hai (dockerd)
2. Manager (dockerd) → Kitchen Manager (containerd) ko task deta hai
3. Kitchen Manager → Chef (runc) ko actual dish banana kehta hai
4. Chef → dish banata hai (container start)

---

## 🌍 Real-World Scenario

### How `docker run nginx` Actually Works Internally:

```bash
$ docker run -d -p 80:80 --name webserver nginx
```

**Step-by-step under the hood:**

1. **CLI → Daemon:** REST POST to `/containers/create`
2. **Daemon checks:** Is `nginx` image available locally?
3. **Image not found → Pull:** Contacts Docker Hub registry
4. **Layer download:** Downloads each layer separately (deduplicated)
5. **containerd extracts:** Layers to a snapshot store
6. **runc creates:** Linux namespaces (pid, net, mnt, uts)
7. **runc sets cgroups:** CPU/memory limits
8. **Process starts:** `nginx -g "daemon off;"` as PID 1 in container
9. **Network setup:** Creates veth pair, connects to bridge network
10. **Port mapping:** iptables rules for 80:80
11. Container ID returned to CLI!

```bash
# Verify the whole chain:
systemctl status dockerd
systemctl status containerd
ls /run/containerd/runc/
ps aux | grep runc
```

---

## ⚙️ Hands-On Examples

### Inspect Docker Architecture Components:
```bash
# Docker daemon config
cat /etc/docker/daemon.json

# Docker info (shows runtime config)
docker info | grep -A5 "Runtime"

# containerd version
containerd --version

# runc version
runc --version

# See runc containers directly
runc list
```

### Docker Socket — Direct API Calls:
```bash
# Docker CLI ke bina REST API call karo
curl --unix-socket /var/run/docker.sock http://localhost/version

# List all containers via API
curl --unix-socket /var/run/docker.sock http://localhost/containers/json

# Run a container via API directly!
curl --unix-socket /var/run/docker.sock \
  -XPOST \
  -H "Content-Type: application/json" \
  http://localhost/containers/create \
  -d '{"Image":"nginx","HostConfig":{"PortBindings":{"80/tcp":[{"HostPort":"8080"}]}}}'
```

### Docker Context — Multiple Daemon Management:
```bash
# List contexts
docker context ls

# Create context for remote Docker
docker context create remote-server \
  --docker "host=ssh://user@remote-host"

# Switch context
docker context use remote-server
docker ps  # Shows containers on remote server!
```

---

## 🔐 Docker Security — Socket Security!

```bash
# DANGER: Mounting Docker socket gives container FULL HOST ACCESS
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
# → Container can now create privileged containers, escape to host!

# In AWS ECS: NEVER mount Docker socket in task
# Use AWS DinD or Kaniko for Docker-in-Docker builds
```

### Rootless Docker (Production Best Practice):
```bash
# Docker as non-root user
dockerd-rootless-setuptool.sh install
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
docker run hello-world  # Runs as non-root!
```

---

## 🚨 Gotchas & Edge Cases

### 1. Docker ≠ containerd ≠ runc
```
Kubernetes (since v1.24) → uses containerd directly, BYPASSES dockerd!
ECS Fargate → uses custom runtime, NOT standard dockerd
ECS EC2 mode → has its own container agent talks to Docker/containerd
```

### 2. Docker Socket = Root Access
```bash
# Anyone with docker socket access = effective root on host
# ECS/EKS se kabhi bhi socket mount mat karo production mein
```

### 3. Daemon Restart = Container Restart
```bash
# Default behavior: restarting dockerd kills all containers
# Use --live-restore flag:
# /etc/docker/daemon.json
{"live-restore": true}
# dockerd restart karne par bhi containers chalta rahega!
```

### 4. Multi-stage Build Performance
```dockerfile
# Stage 1: Build (builder image - heavy)
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# Stage 2: Runtime (tiny final image)
FROM node:18-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
# Final image: 180MB instead of 1.2GB!
```

---

## 🎤 Interview Angle

**Q: "Docker aur containerd mein kya difference hai? Kubernetes kya use karta hai?"**

> - Docker = full platform (CLI + daemon + containerd + runc + networking + volumes)
> - containerd = just the container runtime (no networking, no volumes on its own)
> - Kubernetes v1.24+ uses containerd directly (Dockershim deprecated)
> - ECS Fargate uses AWS's own runtime built on containerd/runc

**Q: "Docker daemon process crash ho jaye toh kya hoga running containers ka?"**

> Default: containers bhi stop ho jayenge.
> Solution: `"live-restore": true` in daemon.json. Containers running rahenge during daemon restart.

---

*Next: [03_Image_vs_Container.md →](./03_Image_vs_Container.md)*
