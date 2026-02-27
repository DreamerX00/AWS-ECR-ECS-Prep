# Performance Engineering — CPU, Memory & Network Tuning

---

## CPU Units in ECS

```
ECS CPU units:
  1024 CPU units = 1 vCPU

Task-level CPU:
  256   = 0.25 vCPU
  512   = 0.5 vCPU
  1024  = 1 vCPU
  2048  = 2 vCPU
  4096  = 4 vCPU

Container-level CPU (within task):
  - Soft reservation: task CPU shared among containers
  - Hard limit: not available (CPU is elastic)
  - Container CPU = SHARES, not hard limit (can burst)
```

### CPU Throttling — The Silent Performance Killer:
```
Node.js app crashing periodically with "Event loop lag spike"?
Latency p99 spikes every few seconds?
→ Could be CPU THROTTLING!

How it works:
  Container CPU: 256 units (0.25 vCPU)
  App suddenly needs 1 vCPU (burst)
  → THROTTLED! Must wait for next CPU scheduling slot
  → Latency spikes!

Detection:
  CloudWatch metric: CPUThrottledTime (Container Insights)
  High CPUThrottledTime → increase CPU units!

Fix: Increase container CPU allocation
  256 → 512 → observe if latency improves
  Or: Optimize app code (reduce CPU usage per request)
```

CPU throttling is insidious because it does not appear as an error — it manifests as intermittent latency spikes that are hard to reproduce in lower environments. The `CPUThrottledTime` metric in Container Insights is the key diagnostic. If this metric is non-zero and trending upward under load, your containers are CPU-constrained and need more CPU allocation.

---

## Memory — Hard vs Soft Limits

```
Task Definition container memory settings:

"memory": 512         ← Hard limit (OOM kill if exceeded!)
"memoryReservation": 256  ← Soft limit (reservation for scheduling)

How they work:
  Scheduler: "Does host have 256MB?" (reservation = scheduling decision)
  Runtime:   "Container used 513MB? → SIGKILL it!" (hard limit = enforcement)

Example:
  Task memory: 1024MB total
  Container A: memory=512, memoryReservation=256
  Container B: memory=256, memoryReservation=128
  Container C (sidecar): memory=128, memoryReservation=64

  Scheduler checks: 256 + 128 + 64 = 448MB available on host? Schedule!
  Runtime: If A uses 600MB → OOM killed (hard limit 512 exceeded!)

OOM kill = exit code 137 in CloudWatch logs!
```

### Memory Leak Detection:
```bash
# Container Insights query: Memory growing over time?
fields @timestamp, MemoryUtilized
| filter ClusterName = "production" and ServiceName = "myapp"
| stats avg(MemoryUtilized) as avgMem by bin(5min)
| sort @timestamp asc
# If trend is consistently upward → MEMORY LEAK!

# Alert: Memory utilization > 80% for 10 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "High-Container-Memory" \
  --metric-name MemoryUtilization \
  --namespace ECS/ContainerInsights \
  --dimensions Name=ServiceName,Value=myapp Name=ClusterName,Value=production \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

A memory leak in a long-running container will eventually cause an OOM kill, which restarts the task. The symptom is periodic task restarts every few hours or days. The diagnostic is the MemoryUtilized metric trending upward over time without a corresponding increase in traffic. Use heap profiling tools specific to your language runtime to identify the leak source.

---

## Network Throughput Considerations

### Fargate Network Bandwidth:
```
Fargate network bandwidth scales with task CPU:
  0.25 vCPU: ~625 Mbps
  0.5 vCPU:  ~1.25 Gbps
  1 vCPU:    ~3 Gbps
  2 vCPU:    ~5 Gbps
  4 vCPU:    ~8 Gbps

High-bandwidth workloads (video streaming, large file transfers):
  Use larger Fargate tasks (more vCPU → more bandwidth)
  OR: Offload to S3 presigned URLs (data flows client→S3 directly, bypassing ECS)
```

### EC2 Enhanced Networking:
```
ECS on EC2:
  Enable: Enhanced Networking (ENA) on instance
  Benefit: Lower latency, higher PPS (packets per second)

  Modern ECS-optimized AMIs have ENA enabled by default!

  c5n.18xlarge: 100 Gbps network bandwidth!
  Good for: Large-scale data processing, ML training data loading
```

---

## Task Startup Optimization

### Fargate Task Boot Time Optimization:
```
Total startup time components:
  1. Fargate VM provision: 10-20s (AWS side, not controllable)
  2. Image pull: 5-120s (MOST CONTROLLABLE!)
  3. Container start: 1-5s
  4. App initialization: YOUR app's startup time

Optimize image pull:
  - Keep image < 200MB: 8-15s pull time on Fargate
  - Large image (1GB): 60-90s pull time!
  - ECR in same region: No cross-region latency
  - Docker layer caching: Fargate caches layers between tasks (somewhat)

Optimize app init:
  - Eliminate heavy startup tasks (do not run migrations at startup!)
  - Lazy initialization (connect to DB on first request, not immediately)
  - Health check startPeriod = actual boot time + 30s buffer
```

Image size is the most impactful factor you can control for Fargate startup time. Multi-stage Docker builds are the primary tool for reducing image size:

```dockerfile
# Multi-stage build example:
# Stage 1: Build (includes all build tools)
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production (minimal runtime only)
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
# Result: 50-80% smaller than single-stage build
CMD ["node", "dist/server.js"]
```

---

## JVM Optimization for ECS/Fargate

```java
// JVM in containers needs special tuning!

// Problem: JVM sees HOST CPU/Memory, not container limits
// JVM max heap = 25% of HOST RAM... but container only has 2GB!
// Container OOM kill before JVM even feels memory pressure!

// Fix Java 11+:
-XX:+UseContainerSupport  // Java automatically respects container limits!
// (Default ON in Java 11+)

// Set heap relative to container:
-XX:MaxRAMPercentage=75.0  // Use 75% of container memory for heap
-XX:InitialRAMPercentage=50.0

// Full JVM flags for ECS Fargate (add to JAVA_OPTS or CMD in Dockerfile):
// Task: 2 vCPU, 4GB memory
CMD ["java",
  "-XX:+UseContainerSupport",
  "-XX:MaxRAMPercentage=75",
  "-XX:+UseG1GC",
  "-XX:MaxGCPauseMillis=200",
  "-Djava.security.egd=file:/dev/./urandom",
  "-jar", "app.jar"]

// Result: Max heap = 3GB (75% of 4GB container), much better!
```

Without `-XX:+UseContainerSupport`, a JVM running in a 2GB Fargate container on a host with 64GB RAM will calculate its default heap as 25% of 64GB = 16GB. The container's hard memory limit of 2GB will then OOM-kill the JVM before it even starts up properly. This is one of the most common Java-in-containers bugs. Java 11+ enables `UseContainerSupport` by default, but older JVMs require it to be set explicitly.

---

## Performance Testing Pattern

```bash
# Before deploying to production: load test ECS service
# Tools: k6, locust, JMeter

# k6 load test against ECS service (via ALB):
k6 run --vus 100 --duration 5m script.js

# While test runs, monitor:
# 1. ALB: TargetResponseTime (p50, p90, p99)
# 2. ECS: CPUUtilization, MemoryUtilization
# 3. Container Insights: CPUThrottledTime
# 4. App: Custom metrics (request queue depth, DB connection pool)

# Scaling test:
# Start with 2 tasks → gradually increase load
# Watch auto-scaling kick in
# Measure: time from "CPU > 70%" to "new tasks serving traffic"
# Typical Fargate: 60-90 seconds (task start + health check)

# Identify bottleneck:
# CPU maxed? → Scale out (more tasks) or right-size vCPU
# Memory maxed? → Increase memory or investigate leak
# DB latency high? → DB bottleneck (not ECS)
# External API slow? → Implement circuit breaker + caching
```

---

## Connection Pool Sizing for Containerized Services

When running many small ECS tasks, each task's database connection pool multiplies:

```
Example: RDS PostgreSQL, max_connections = 500

10 tasks × 10 connections per pool = 100 connections (safe)
100 tasks × 10 connections per pool = 1000 connections (exceeded limit!)

Solution 1: Reduce pool size per task
  10 connections → 3 connections per task
  100 tasks × 3 = 300 connections (safe)

Solution 2: RDS Proxy
  All tasks connect to RDS Proxy → Proxy multiplexes to RDS
  Tasks can have large pools, Proxy reuses connections
  Max task connections ≠ Max RDS connections (proxied!)
```

RDS Proxy is essential for workloads that scale ECS tasks aggressively, especially with auto-scaling. Without it, database connection exhaustion becomes the scaling ceiling.

---

## Latency Optimization Checklist

```
Network Latency:
  ✅ ECS tasks and RDS in same AZ (dev/staging) or same region (prod)
  ✅ ECR VPC endpoint (eliminates NAT Gateway latency for image pulls)
  ✅ ElastiCache for caching repeated queries
  ✅ Connection pooling enabled

Application Latency:
  ✅ Async non-blocking I/O (Node.js, Java NIO, Go goroutines)
  ✅ Avoid synchronous calls in hot path
  ✅ Cache expensive computations in memory
  ✅ Database query optimization (indexes, query plans)

Infrastructure Latency:
  ✅ Fargate task size adequate for throughput (CPU not throttling)
  ✅ ALB targets in same region as ALB
  ✅ Keep-alive connections between ALB and targets
  ✅ HTTP/2 enabled on ALB for multiplexing
```

---

## Interview Angle

**Q: "How do CPU units and memory limits work in ECS containers?"**

> CPU: 1024 units equals 1 vCPU. The task-level CPU is the total budget. Container-level CPU is a soft reservation that influences proportional time-sharing and scheduling, but containers can burst beyond their reservation if the host has spare CPU cycles. There is no hard CPU cap in ECS (unlike Kubernetes CPU limits). High `CPUThrottledTime` in CloudWatch indicates the container is CPU-constrained.
>
> Memory: The hard limit (`memory`) is enforced at runtime. If the container exceeds this limit, the Linux kernel sends SIGKILL and the container exits with code 137. The soft limit (`memoryReservation`) is used only for scheduling decisions. Set the hard memory limit to (peak observed usage + 20% buffer), and set memoryReservation to the typical usage.
>
> Gotcha: JVM defaults to calculating heap size based on host RAM, not container limits. In containers, use `-XX:+UseContainerSupport` (enabled by default in Java 11+) and `-XX:MaxRAMPercentage=75` to correctly scope the heap to the container's memory allocation.

---

*Next: [03_CI_CD_Full_Pipeline.md →](./03_CI_CD_Full_Pipeline.md)*
