# 🔧 Performance Engineering — CPU, Memory & Network Tuning

---

## 📖 CPU Units in ECS

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

---

## 💾 Memory — Hard vs Soft Limits

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

---

## 🌐 Network Throughput Considerations

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
  
  Modern ECS-optimized AMIs have ENA by default!
  
  c5n.18xlarge: 100 Gbps network bandwidth!
  Good for: Large-scale data processing, ML training data loading
```

---

## 🚀 Task Startup Optimization

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
  - Eliminate heavy startup tasks (don't run migrations at startup!)
  - Lazy initialization (connect to DB on first request, not immediately)
  - Health check startPeriod = actual boot time + 30s buffer
```

---

## ⚡ JVM Optimization for ECS/Fargate

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

---

## 🎯 Performance Testing Pattern

```bash
# Before deploying to production: Load test ECS service
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

## 🎤 Interview Angle

**Q: "ECS container mein CPU units aur memory limits kaise kaam karte hain?"**

> CPU: 1024 units = 1 vCPU. Task-level CPU = total budget. Container-level CPU = soft reservation (for scheduling + proportional time sharing). CPU kan burst beyond reservation if host has slack — no hard CPU cap in ECS (unlike K8s CPU limits).
>
> Memory: Hard limit (`memory`) = enforcement. Container exceeds this → OOM kill (exit code 137). Soft limit (`memoryReservation`) = scheduling consideration. Set memory = (peak usage + 20% buffer). Set memoryReservation = typical usage.
>
> Gotcha: JVM defaults to 25% of HOST RAM for heap. In containers, use `-XX:+UseContainerSupport` (Java 11+ default) + `-XX:MaxRAMPercentage=75` to properly use container memory.

---

*Next: [03_CI_CD_Full_Pipeline.md →](./03_CI_CD_Full_Pipeline.md)*
