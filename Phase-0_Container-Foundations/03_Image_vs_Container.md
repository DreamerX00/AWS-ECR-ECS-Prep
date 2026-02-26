# 🖼️ Image vs Container — The Core Distinction

---

## 📖 Concept Explanation

Ye distinction **sabse common interview confusion** hai. Chalte hain roots tak:

| | **Docker Image** | **Docker Container** |
|--|--|--|
| Definition | Blueprint/Template | Running instance of image |
| State | Immutable (read-only) | Mutable (writable layer on top) |
| Storage | Disk | Memory + Disk (thin writable layer) |
| Multiplicity | One image → many containers | Each container = independent instance |
| Lifecycle | Built once, stored forever | Created, started, stopped, deleted |

---

## 🏗️ Internal Architecture

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

### Container = Image + Writable Layer (CoW)
```
┌─────────────────────────────┐
│   WRITABLE LAYER (CoW)      │  ← Container-specific changes
├═════════════════════════════╡
│   App Layer (READ-ONLY)     │
├─────────────────────────────┤
│   Dependency Layer (R/O)    │  ← Shared across ALL containers
├─────────────────────────────┤
│   Package Layer (R/O)       │     using same image
├─────────────────────────────┤
│   Base OS Layer (R/O)       │
└─────────────────────────────┘
```

**Copy-on-Write (CoW):** Jab container file modify karta hai:
1. Original file check karo (read-only layer mein)
2. File ko writable layer mein **copy** karo
3. Writable layer mein modification karo
4. Original read-only layer **unchanged** rehta hai

---

## 🎯 Analogy — Blueprint vs House 🏠

**Image = Architectural Blueprint:**
- Ek hi blueprint se 100 ghar ban sakte hain
- Blueprint khud change nahi hota
- Blueprint mein material specify hai (bricks, windows, etc.)

**Container = Actual Constructed House:**
- Blueprint se banaya gaya actual ghar
- Log isme rehte hain → furniture add karte hain (changes)
- Ek ghar tod do → blueprint safe hai → naaya ghar bana lo
- 10 log, 10 houses all from same blueprint → but each independent

```
docker build -t my-app .          # Blueprint create karo
docker run my-app                 # Ghar banao (instance 1)
docker run my-app                 # Ghar banao (instance 2)
docker run my-app                 # Ghar banao (instance 3)
# Teeno independent, same blueprint
```

---

## 🌍 Real-World Scenario

### Production Scale: E-commerce Flash Sale

**Black Friday pe:**
- 1 image: `flipkart-backend:v2.1.4`
- 50 containers running from this single image
- Each container has its own session data, logs (in writable layer)
- Total image storage: 800MB (stored once on each host)
- Total container overhead: 50 × ~10MB CoW layers = 500MB
- **Without layers:** 50 × 800MB = 40GB wasted!

```bash
# Scale up before flash sale
docker service scale flipkart-backend=50

# They all share the same image layers!
docker inspect flipkart-backend --format '{{.RootFS.Layers}}'
# All 50 containers reference same 6 layers
```

### Inspect Real Layer Sharing:
```bash
# Build two similar images
docker build -t app:v1 .
docker build -t app:v2 .  # Only changed last layer

# Check layer IDs
docker history app:v1
docker history app:v2
# Common layers show identical hash → SHARED on disk!
```

---

## ⚙️ Hands-On Examples

### Image Deep Inspection:
```bash
# See all layers with sizes
docker history nginx:latest
# IMAGE          CREATED         CREATED BY                                      SIZE
# f9c14fe76d50   2 weeks ago     /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B
# <missing>      2 weeks ago     /bin/sh -c #(nop)  EXPOSE 80                    0B
# <missing>      2 weeks ago     /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT           0B
# ...

# See image metadata
docker inspect nginx:latest | jq '.[0].RootFS'

# List local images with sizes
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### Container Lifecycle:
```bash
# Image pull karo
docker pull nginx:latest

# Container banao (but start mat karo)
docker create --name test-nginx -p 8080:80 nginx:latest

# Start karo
docker start test-nginx

# Stop karo
docker stop test-nginx

# Delete karo (writable layer gone!)
docker rm test-nginx

# Image abhi bhi hai:
docker images | grep nginx  # Still there!

# SHORT WAY:
docker run -d --name test-nginx -p 8080:80 nginx  
# = create + start in one command
```

### Container Writable Layer Inspect:
```bash
# Run a container and write a file
docker run -d --name demo nginx
docker exec demo sh -c "echo 'hello' > /tmp/myfile.txt"

# Container ka storage layer dekho
docker inspect demo | jq '.[0].GraphDriver'
# Shows UpperDir (writable layer path)

UPPER=$(docker inspect demo | jq -r '.[0].GraphDriver.Data.UpperDir')
ls $UPPER/tmp/
# myfile.txt is here! On host filesystem!

# Container delete karo → file gone
docker rm -f demo
ls $UPPER/  # Directory gone!
```

---

## 🚨 Gotchas & Edge Cases

### 1. Container Delete = Data Loss!
```bash
# DATABASE CONTAINER PE KABHI YAH MAT KARO:
docker run -d mysql:8  # No volume!
# MySQL data /var/lib/mysql mein hai (container writable layer)
# docker rm karo → ALL DATA GONE!

# SAHI TARIKA:
docker run -d -v mysql-data:/var/lib/mysql mysql:8
# Volume mounted → data survives container deletion
```

### 2. docker commit — Anti-Pattern!
```bash
# Log container mein jaake changes karo
docker exec -it myapp bash
apt-get install -y vim  # Changes in writable layer

# Commit karo to create image
docker commit myapp myapp:with-vim
# YAH ANTI-PATTERN HAI! 
# Reproducible nahi hai, Dockerfile se banana chahiye hamesha
```

### 3. Image Size Matters for ECR Costs + Pull Time
```bash
# Bad: Single layer, huge image
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3 && pip install requirements.txt && ...

# Good: Multi-stage, minimal final image
FROM python:3.11-slim
# 50MB vs 1.2GB difference = real money in ECR storage + faster ECS task starts
```

### 4. Same Image Could Run as Root Inside Container
```bash
# Default: container runs as root!
docker run alpine whoami  # root

# Security risk! Always specify non-root user in Dockerfile:
USER 1000  # or RUN adduser -D appuser && USER appuser
```

---

## 🎤 Interview Angle

**Q: "Docker image aur container mein kya fark hai? Multiple containers ek image share kaise karte hain?"**

> Image = immutable layered filesystem (read-only).
> Container = image ke upar ek thin writable layer add hoti hai (CoW model).
> 100 containers ek image share kar sakte hain — sirf unka writable layer alag hota hai.
> Isliye container overhead bahut kam hota hai vs VMs.

**Q: "Container delete karne par data kyun jaata hai?"**

> Data container ke writable layer mein hota hai.
> Container delete hone par writable layer bhi delete ho jaati hai.
> Persistence ke liye Docker Volumes ya EFS (ECS mein) use karo.

---

*Next: [04_Layers_And_UnionFS.md →](./04_Layers_And_UnionFS.md)*
