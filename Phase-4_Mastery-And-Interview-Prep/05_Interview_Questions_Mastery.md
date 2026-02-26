# 🎤 Interview Questions Mastery — Zero to God Level Answers

---

## 📋 Question Bank — Organized by Difficulty

---

## 🟢 FUNDAMENTALS (Fresher - 1 year)

### Q1: "Docker image aur container mein kya fark hai?"

**Power Answer:**
> Docker image ek **immutable layered filesystem** hai — ek blueprint jo read-only layers ka stack hai. Har `RUN`/`COPY` instruction ek nayi layer banata hai, SHA256 hash se identified.
>
> Container = Image + **thin writable layer** (Copy-on-Write). Ek image se 100 containers chal sakte hain — sab shared read-only layers use karte hain, sirf unka writable layer alag hota hai. Yahi layering containers ko VMs se 10-100x resource-efficient banati hai.
>
> Real impact: 10 services, same base image → ECR mein base layer sirf ek baar stored — 40-70% storage savings.

---

### Q2: "ECR aur Docker Hub mein kya difference hai?"

**Power Answer:**
> ECR AWS-native aur Docker Hub third-party hai. Key differences:
>
> **Auth:** ECR = IAM (no username/password in code). Docker Hub = username + password/token.
>
> **Rate limits:** Docker Hub free = 100 pulls/6 hours (CI/CD killer). ECR = no limits.
>
> **AWS integration:** ECR → ECS/EKS auto-authenticate via IAM roles — zero configuration. Docker Hub → manual auth setup in every environment.
>
> **Security:** ECR Enhanced Scanning (AWS Inspector) = OS + application dependencies continuously. VPC Endpoint = private pulls without NAT.
>
> **Cost:** ECR = $0.10/GB/month. Docker Hub free tier exists but rate-limited.
>
> Production choice: Always ECR for AWS workloads.

---

### Q3: "ECS Task aur Service mein kya difference hai?"

**Power Answer:**
> **Task:** Ek running instance of Task Definition. Ek baar run hota hai — complete hota hai ya crash hota hai, ECS automatically restart nahi karta. Use case: DB migration, batch job, one-off script.
>
> **Service:** Long-running manager. Ensures N tasks (desiredCount) ALWAYS running. Task crash → automatically replace. Health check fail → replace. Scale karo → service manages. ALB registration/deregistration automatically handle karta hai.
>
> **Rule of thumb:** Web server, API, worker = Service. One-time execution = Task.

---

## 🟡 INTERMEDIATE (2-4 years)

### Q4: "ECS Task Role aur Execution Role mein kya difference hai? Interview mein most common question!"

**Power Answer:**
> **Execution Role:** ECS Container Agent use karta hai **task start karne ke liye**:
> - ECR se image pull karna (ecr:BatchGetImage)
> - Secrets Manager/SSM se secrets fetch karna
> - CloudWatch mein logs likhna
> → Task start hone se PEHLE active
>
> **Task Role:** Aapka **application code** use karta hai runtime pe:
> - S3 read/write
> - DynamoDB query
> - SQS message send
> → AWS SDKs 169.254.170.2 metadata endpoint se automatically credentials lete hain
>
> **Simple memory trick:** "ECS ko task start karne ke liye kya chahiye?" = Execution Role. "Meri app ko AWS mein kya access chahiye?" = Task Role.
>
> Jab S3 access denied aaye → Task Role check. Jab image pull fail hoi → Execution Role check.

---

### Q5: "ECS Fargate vs EC2 kab choose karein? Tradeoffs kya hain?"

**Power Answer:**
> **Fargate choose karo when:**
> - DevOps bandwidth nahi hai (EC2 patching, AMI updates, agent management skip)
> - Variable/spiky traffic (pay per task, not idle instances)
> - Time-to-market priority
> - No GPU/special hardware needed
> - New team / MVP phase
>
> **EC2 choose karo when:**
> - GPU workloads (ML inference, AI) — Fargate GPU support nahi
> - Daemon services (one per instance — log collectors, monitoring)
> - 24/7 predictable load + Reserved Instances = 44-65% savings vs Fargate
> - Custom AMI needed (security-hardened, pre-warmed images)
> - Very high container density needed (pack more on large instances)
>
> **My production recommendation:** Start with Fargate. 90% teams ko EC2 ki zarurat nahi padti. GPU ya daemon patterns pe jaake EC2 add karo.

---

### Q6: "ECS ma rolling update kaise kaam karta hai? Zero downtime kaise guarantee karte hain?"

**Power Answer:**
> Rolling update 2 parameters se control hota hai:
> `minimumHealthyPercent`: Deployment ke dauraan minimum kitne % tasks healthy rahein (e.g., 50% = 10 tasks mein 5 kabhi bhi down nahi)
> `maximumPercent`: Maximum kitne % tasks ek saath run kar sakte hain (e.g., 200% = temporarily 2× tasks)
>
> Flow:
> 1. New tasks launch hote hain (tak ki max% nahi pahuncha)
> 2. New tasks ALB health check pass karte hain
> 3. Old tasks deregister from ALB (connection drain = deregistrationDelay seconds)
> 4. Old tasks SIGTERM → graceful shutdown → stop
>
> **Zero downtime:** minimumHealthyPercent=100, maximumPercent=200 → new tasks start → pass health check → JOIN ALB → THEN old tasks removed. Never capacity drops below 100%.
>
> **Better:** Blue/Green (CodeDeploy) → all new tasks in parallel → instant traffic switch → rollback in <1s.

---

### Q7: "ECR image scanning kaise kaam karta hai? Enhanced vs Basic mein kya fark hai?"

**Power Answer:**
> **Basic Scanning:**
> - Engine: Clair (open-source)
> - When: On push only (one-time)
> - What: OS package CVEs only
> - Cost: Free
> - Limitation: New CVEs published tomorrow → existing images NOT re-scanned
>
> **Enhanced Scanning (AWS Inspector):**
> - Engine: AWS Inspector + Snyk
> - When: **Continuously!** — push pe + jab naya CVE publish hota hai, existing images rescan
> - What: OS packages + application dependencies (npm, pip, maven, gem, go modules)
> - Cost: Additional charge ($0.09/image/month)
> - EventBridge events fired on new findings → Lambda → alert/block deployment
>
> **For production:** Enhanced always. Basic = one-time scan aur phir bhoolo. New CVE daily aati hain — continuous scanning mandatory.

---

## 🔴 ADVANCED (4+ years / Staff/Principal level)

### Q8: "ECS Scheduler task placement kaise decide karta hai? Detail mein explain karo."

**Power Answer:**
> ECS Scheduler placement 3-phase process:
>
> **Phase 1: Eligible Instance Filter**
> - Only connected Container Agents (not disconnected/draining)
> - Has sufficient CPU + Memory for the task
> - Correct launch type (EC2 instances for EC2-mode tasks)
>
> **Phase 2: Constraint Filtering**
> - `distinctInstance`: Each task on different EC2 instance (eliminates hosts already running this service)
> - `memberOf` (cluster query language): Complex expressions like `attribute:ecs.instance-type =~ c5.*` or custom attributes
> - Hosts not matching constraints → eliminated from candidate pool
>
> **Phase 3: Strategy Application (on remaining candidates)**
> - `spread(attribute:ecs.availability-zone)`: Pick host in AZ with fewest tasks of this service
> - `binpack(memory)`: Pick host with LEAST remaining memory (pack dense)
> - `random`: Pick randomly
> - Strategies evaluated in order → final host selected
>
> **Combined example (production best practice):**
> `spread(AZ)` then `binpack(memory)` = AZ balance maintained WHILE maximizing instance density within each AZ.

---

### Q9: "ECS mein zero downtime Blue/Green deployment kaise design karein? Rollback strategy kya hogi?"

**Power Answer:**
> **Setup (CodeDeploy + ECS):**
> 1. Two ALB Target Groups: blue-tg (current) + green-tg (empty)
> 2. One ALB production listener → blue-tg
> 3. One ALB test listener (port 8080) → used for testing before switch
>
> **Deployment Flow:**
> 1. CodeDeploy launches new tasks in green-tg (zero production traffic yet)
> 2. Test listener routes 100% traffic to green → automated tests run
> 3. If tests pass → canary (10% prod traffic → green) → monitor
> 4. If healthy → full cutover: 100% prod traffic → green-tg (millisecond switch!)
> 5. Blue tasks remain running for "bakeTime" (e.g., 30 min)
>
> **Rollback:**
> - Within bakeTime window: One API call → switch listener back to blue → <1s rollback
> - After bakeTime: Blue terminated — must re-deploy previous version
>
> **Key advantage over rolling:** At no point are mixed-version tasks serving prod traffic simultaneously. Either 100% old OR 100% new (after cutover). Critical for breaking API changes.

---

### Q10: "Production mein ECR + ECS cost $10K/month tak pahunch gayi. Kaise optimize karoge?"

**Power Answer (structured approach):**
> **Step 1: Identify cost breakdown (Cost Explorer by service)**
> - ECR storage/transfer percentage
> - ECS Fargate compute percentage
> - Data transfer percentage
>
> **Step 2: Quick wins first (< 1 week)**
> - **ECR Lifecycle Policies**: Keep only last 5-10 images per repo → immediate storage reduction
> - **Image size audit**: Multi-stage builds → 50-70% image size reduction → storage + pull time savings
> - **Fargate Spot**: Batch jobs, workers → 70% discount on eligible tasks
>
> **Step 3: Medium-term (1-4 weeks)**
> - **Right-size containers**: Container Insights → actual CPU/Memory usage → reduce task definition sizes 20-30%
> - **VPC Endpoints**: Enable ECR + S3 endpoints → eliminate NAT Gateway costs for image pulls
> - **Compute Savings Plans**: Analyze average Fargate spend → commit 1-year → 20-52% discount
>
> **Step 4: Architectural (4-12 weeks)**
> - **Cross-region optimization**: Teams running in other regions → ECR replication → local pulls (no inter-region data transfer)
> - **EC2 Reserved Instances for baseline load**: Identify constant tasks → EC2 RI vs Fargate = 40-60% savings for 24/7
>
> **Expected outcome**: 40-65% cost reduction is achievable with all strategies combined.

---

### Q11: "ECS task ENI exhaustion issue kya hai aur kaise solve karein?"

**Power Answer:**
> **Problem:** awsvpc networking mode mein har task ko apna dedicated ENI chahiye. EC2 instances have ENI limits:
> - t3.micro: max 2 ENIs → 1 host ENI + 1 task only!
> - c5.xlarge: max 4 ENIs → 3 tasks max
> - c5.4xlarge: max 8 ENIs → 7 tasks max
>
> Symptom: Tasks stuck in PROVISIONING state with "no available ENIs" error. awsvpc task density severely limited.
>
> **Solutions:**
> 1. **Larger instances**: c5.18xlarge = 15 ENIs → 14 concurrent tasks
> 2. **ENI Trunking** (ECS Managed Networking): Enable `awsvpcTrunking` on cluster → uses "trunk ENI" technique → up to 120 tasks per instance! Requires nitro-based instances and ECS Agent >= 1.55.0.
> 3. **Fargate**: No ENI limit worries — AWS handles it
> 4. **Mix EC2 + Fargate**: Baseline on EC2 (known density), burst to Fargate (unlimited)

---

### Q12: "Ek ECS production incident describe karo aur kaise debug karoge?"

**Framework Answer (STAR method):**
> **Incident:** "Service degraded — 50% requests returning 503"
>
> **Triage Steps (live):**
> 1. ALB metrics: HTTPCode_Target_5XX → which target group? Target health → unhealthy tasks?
> 2. ECS: `describe-services` → running vs desired count. Events → "unable to place" or "task stopped"?
> 3. Stopped tasks: `describe-tasks` → `stoppedReason` + `containers[].exitCode`
>   - Exit 137 = OOM → increase memory limit
>   - Exit 1 = App crash → check logs
>   - TaskFailedToStart → image pull or secret fetch failed
> 4. CloudWatch Logs: `logs tail /ecs/myservice --follow`
> 5. If task running but failing: `ecs execute-command` → exec into running task
>
> **Common root causes:**
> - Memory leak → OOM kills (exit 137) → gradually escalating: increase memory limit, fix leak
> - Bad deployment → circuit breaker → check recent task definition changes
> - External dependency failing → health check marking tasks as unhealthy → fix or circuit-break dependency
> - ECR image pull failing → check Execution Role ECR permissions + VPC endpoint if private subnet

---

## 🏆 BONUS: System Design Question

### Q13: "100M users ke liye Video Streaming Platform ke liye ECS + ECR architecture design karo"

**Answer Framework:**
```
Scale: 100M users, 1M concurrent at peak, 10K videos processed/day

ECR:
  Repositories (with cross-region replication to ap-south-1, eu-west-1):
    streaming-service (replicated)
    transcoder-service (optimized, replicated)
    recommendation-ml (GPU-optimized image)
    api-gateway (replicated)

ECS Clusters:
  "streaming" cluster:
    user-api-service: Fargate, 50-500 tasks (target tracking: ALBRequestCountPerTarget=500)
    cdn-origin: Fargate, 20-200 tasks (scales with traffic)
  
  "processing" cluster:
    video-transcoder: EC2 (GPU instances g5.4xlarge), 10-50 tasks
    transcoder daemon: EventBridge Pipe → SQS → ECS Task per video
    ML-recommendation: EC2 Graviton, 5-10 tasks (ARM64 optimized images)

Deployment:
  user-api: Blue/Green (CodeDeploy) → zero downtime, instant rollback
  transcoder: Rolling (batch, can tolerate brief downtime)

Cost Optimization:
  Video processing (transcoders): Fargate Spot → 70% savings on GPU costs
  API layer: Compute Savings Plan (predictable baseline)
  ECR: Lifecycle policy → keep last 5 production images per service

Security:
  Payment handling: Fargate (MicroVM isolation)
  Each service: Task-specific IAM Role (least privilege)
  Secrets Manager: DB passwords, API keys (auto-rotation)
  Enhanced scanning: All ECR repos

Observability:
  Container Insights + X-Ray + FireLens → OpenSearch
  Custom metric: videos_processed_per_hour, streaming_latency_p99  
  Alert: p99 > 5s → PagerDuty

Target: <200ms API latency (p99), 99.99% uptime
```

---

## 📝 Quick Reference Cheat Sheet

```
ECR Auth:
  Developer: aws ecr get-login-password | docker login
  ECS:       Auto via Execution Role (zero config!)
  CI/CD:     OIDC → IAM Role → no static keys

IAM Roles:
  Task Role      → App code (S3, DynamoDB)
  Execution Role → ECS Agent (ECR pull, logs, secrets)
  Instance Role  → EC2 host (ECS API calls)

Networking:
  bridge  → EC2 only, dynamic ports, shared EC2 SG
  host    → EC2 only, max performance, port conflicts possible
  awsvpc  → Fargate required, task-level SG, own ENI

Deployment:
  Rolling    → Simple, may have mixed versions
  Blue/Green → Zero downtime, instant rollback, CodeDeploy
  Canary     → 1-10% traffic test, gradual rollout

Scaling:
  Target Tracking → "Keep CPU at 70%"    → Autopilot
  Step Scaling    → "If CPU>80%, +3"     → Manual steps
  Scheduled       → "Mon 8AM: +10 tasks" → Known patterns
```

---

*You are now ready for any AWS ECS/ECR interview, from fresher to Staff Engineer level!*
