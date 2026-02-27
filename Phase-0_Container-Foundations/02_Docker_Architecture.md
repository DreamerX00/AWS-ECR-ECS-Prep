# Docker Architecture

---

## Concept Explanation

Docker follows a **client-server architecture**. When you run a simple command like `docker run nginx`, multiple internal components coordinate to make it happen.

### Core Components:
1. **Docker CLI** — The user-facing command-line interface
2. **Docker Daemon (dockerd)** — The backend server that manages all Docker objects
3. **containerd** — A container lifecycle manager (high-level runtime)
4. **runc** — An OCI-compliant low-level container runtime
5. **Docker Registry** — Image storage system (e.g., ECR, Docker Hub)

Understanding this layered architecture is essential for debugging failures, understanding Kubernetes internals, and working with ECS effectively.

---

## Internal Architecture

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

#### Docker CLI
```bash
docker run -d -p 80:80 nginx
```
The CLI is purely a **REST API client**. It translates your command into an HTTP request and sends it to the Docker Daemon over a Unix socket. The CLI itself performs no container management work.

#### Docker Daemon (dockerd)
- Binary location: `/usr/bin/dockerd`
- Manages: images, containers, networks, and volumes
- Listens on a Unix socket: `/var/run/docker.sock`
- Communicates with containerd via gRPC
- Handles higher-level concerns: authentication, image caching, volume management, networking abstraction

#### containerd
- Responsible for **container lifecycle management**: create, start, stop, pause, delete
- Handles image management: pulling, pushing, and extracting layers into a snapshot store
- Pluggable runtime support: alternative runtimes like Kata Containers can plug in here
- **Kubernetes uses containerd directly** (bypassing dockerd entirely since Kubernetes 1.24 removed Dockershim)

#### runc
- Does the actual work of **creating a container**
- Makes Linux syscalls directly: `clone()` for namespace creation, `setrlimit()` for cgroup enforcement
- Strictly follows the OCI Runtime Specification
- After `runc` starts the container process, it exits — it does not remain running as a daemon
- `runc run my-container` is all that is needed to start a container at this level

---

## Analogy — Restaurant Kitchen

```
Customer (You)          → Docker CLI
Head Waiter             → Docker Daemon (receives and routes orders)
Kitchen Manager         → containerd (coordinates cooking operations)
Individual Chef         → runc (actually prepares the dish)
Recipe Book             → Docker Image
Finished Plate of Food  → Running Container
Ingredient Store        → Docker Registry (ECR)
```

When a customer places an order:
1. The waiter (CLI) communicates the order to the manager (dockerd)
2. The manager (dockerd) delegates the task to the kitchen manager (containerd)
3. The kitchen manager directs the chef (runc) to prepare the specific dish
4. The chef creates the dish (starts the container) and steps away

The key insight is that the customer does not need to know any of this delegation chain exists. They just place the order.

---

## Real-World Scenario

### How `docker run nginx` Works Internally:

```bash
$ docker run -d -p 80:80 --name webserver nginx
```

**Step-by-step under the hood:**

1. **CLI → Daemon:** REST POST to `/containers/create`
2. **Daemon checks:** Is the `nginx` image available locally?
3. **Image not found → Pull:** Contacts Docker Hub registry
4. **Layer download:** Downloads each layer separately; deduplicates against layers already on disk
5. **containerd extracts:** Layers into a snapshot store
6. **runc creates:** Linux namespaces (pid, net, mnt, uts)
7. **runc configures cgroups:** CPU and memory limits applied
8. **Process starts:** `nginx -g "daemon off;"` runs as PID 1 inside the container
9. **Network setup:** Creates a veth pair and connects it to the docker bridge network
10. **Port mapping:** iptables rules added for 80:80
11. Container ID returned to the CLI

```bash
# Verify the whole chain:
systemctl status dockerd
systemctl status containerd
ls /run/containerd/runc/
ps aux | grep runc
```

---

## Hands-On Examples

### Inspect Docker Architecture Components:
```bash
# Docker daemon configuration
cat /etc/docker/daemon.json

# Docker info (shows runtime configuration)
docker info | grep -A5 "Runtime"

# containerd version
containerd --version

# runc version
runc --version

# List containers managed directly by runc
runc list
```

### Docker Socket — Direct API Calls:
```bash
# Call the Docker REST API directly, bypassing the CLI
curl --unix-socket /var/run/docker.sock http://localhost/version

# List all containers via API
curl --unix-socket /var/run/docker.sock http://localhost/containers/json

# Create and start a container via the API directly
curl --unix-socket /var/run/docker.sock \
  -XPOST \
  -H "Content-Type: application/json" \
  http://localhost/containers/create \
  -d '{"Image":"nginx","HostConfig":{"PortBindings":{"80/tcp":[{"HostPort":"8080"}]}}}'
```

This is exactly what the Docker CLI does internally. Understanding the API directly is valuable for building automation, CI/CD integrations, and debugging daemon-level issues.

### Docker Context — Multiple Daemon Management:
```bash
# List configured contexts
docker context ls

# Create a context pointing to a remote Docker host
docker context create remote-server \
  --docker "host=ssh://user@remote-host"

# Switch context
docker context use remote-server
docker ps  # Now shows containers on the remote server
```

---

## Docker Security — Socket Security

```bash
# DANGER: Mounting the Docker socket gives a container full host access
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
# A container with socket access can create privileged containers,
# mount host filesystems, and fully escape the container boundary!

# In AWS ECS: Never mount the Docker socket in a task definition.
# Use AWS DinD (Docker-in-Docker) patterns or Kaniko for container builds in CI.
```

### Rootless Docker (Production Best Practice):
```bash
# Run Docker daemon as a non-root user
dockerd-rootless-setuptool.sh install
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
docker run hello-world  # Daemon and containers run without root!
```

Rootless Docker significantly reduces the blast radius of a container escape by ensuring that the daemon itself never runs as root on the host.

---

## Gotchas & Edge Cases

### 1. Docker ≠ containerd ≠ runc
```
Kubernetes (since v1.24) → uses containerd directly, bypasses dockerd entirely
ECS Fargate              → uses a custom AWS runtime built on containerd/runc principles
ECS EC2 mode             → uses the ECS container agent, which talks to Docker/containerd
```

This means Docker knowledge translates well to Kubernetes and ECS, but the specific components involved differ at each layer.

### 2. Docker Socket Equals Root Access
```bash
# Any process or container with access to the Docker socket
# has effective root privileges on the host.
# Never expose the socket to untrusted workloads in ECS or EKS.
```

### 3. Daemon Restart and Container Behavior
```bash
# Default behavior: restarting dockerd stops all running containers.
# Enable live restore to keep containers running during daemon restarts:
# /etc/docker/daemon.json
{"live-restore": true}
# With this flag, containers continue running even when dockerd restarts.
```

This is critical for production environments where planned maintenance on the Docker daemon should not affect running workloads.

### 4. Multi-stage Build Performance
```dockerfile
# Stage 1: Build environment (large image with all build tools)
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# Stage 2: Runtime environment (minimal production image)
FROM node:18-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
# Final image: ~180MB instead of ~1.2GB
```

Multi-stage builds are one of the highest-impact optimizations available. They reduce image size, cut ECR storage costs, speed up ECS task startup, and reduce the attack surface of production containers.

---

## Interview Angle

**Q: "What is the difference between Docker and containerd? What does Kubernetes use?"**

> - Docker is a complete platform: CLI + daemon + containerd + runc + networking + volumes
> - containerd is just the container runtime component — no CLI, no volume management, no networking abstraction
> - Kubernetes v1.24+ uses containerd directly via the CRI (Container Runtime Interface), bypassing dockerd
> - ECS Fargate uses AWS's own custom runtime built on OCI-compliant principles similar to containerd/runc

**Q: "If the Docker daemon crashes, what happens to running containers?"**

> By default, all running containers stop when dockerd stops.
> This can be prevented by setting `"live-restore": true` in `/etc/docker/daemon.json`.
> With live restore enabled, containers keep running during daemon restarts, which is essential for zero-downtime maintenance.

**Q: "What is the Docker socket and why is it a security concern?"**

> `/var/run/docker.sock` is the Unix socket that the Docker daemon listens on. Any process that can write to this socket can issue arbitrary Docker API calls — including creating privileged containers that mount the host filesystem. This effectively gives that process root access to the host. It should never be mounted into containers in production environments.

---

*Next: [03_Image_vs_Container.md →](./03_Image_vs_Container.md)*
