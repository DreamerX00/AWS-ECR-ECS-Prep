# Interview Questions Mastery — Zero to Expert-Level Answers

---

## Question Bank — Organized by Difficulty

---

## FUNDAMENTALS (Fresher - 1 year)

### Q1: "What is the difference between a Docker image and a container?"

**Power Answer:**
> A Docker image is an **immutable layered filesystem** — a blueprint composed of read-only layers stacked on top of each other. Every `RUN`, `COPY`, and `ADD` instruction in a Dockerfile creates a new layer, identified by a SHA256 hash.
>
> A container is an image plus a **thin writable layer** (Copy-on-Write). You can run 100 containers from a single image — all of them share the same read-only base layers, and each has only its own isolated writable layer. This layer-sharing model makes containers 10-100x more resource-efficient than VMs, which each require a full OS copy.
>
> Real-world impact: If 10 microservices share the same base image (e.g., `node:18-alpine`), that base layer is stored only once in ECR — resulting in 40-70% storage savings compared to storing 10 independent images.

---

### Q2: "What is the difference between ECR and Docker Hub?"

**Power Answer:**
> ECR is AWS-native and Docker Hub is a third-party registry. The key differences that matter in production:
>
> **Authentication:** ECR uses IAM — no usernames or passwords in code. Docker Hub requires a username and access token that must be managed as a secret.
>
> **Rate limits:** Docker Hub free tier limits unauthenticated pulls to 100 per 6 hours — a CI/CD pipeline killer. ECR has no pull rate limits.
>
> **AWS integration:** ECR authentication to ECS, EKS, and CodeBuild is automatic via IAM roles — zero configuration needed. Docker Hub requires manual authentication setup in every environment.
>
> **Security:** ECR Enhanced Scanning (powered by AWS Inspector + Snyk) continuously scans both OS packages and application dependencies. New CVEs published after image creation trigger automatic rescans of existing images. Docker Hub scanning is more limited.
>
> **Cost:** ECR costs $0.10/GB/month for storage. Docker Hub's free tier is rate-limited; paid plans start at $7/month.
>
> For any AWS workload, ECR is the correct choice.

---

### Q3: "What is the difference between an ECS Task and a Service?"

**Power Answer:**
> **Task:** A single running instance of a Task Definition. It starts, does its work, and either completes normally or crashes. ECS does not automatically restart a stopped standalone task. Use cases: database migrations, one-off batch jobs, scheduled scripts.
>
> **Service:** A long-running manager that maintains a desired count of tasks. If a task crashes, the Service Controller automatically launches a replacement. It handles ALB registration and deregistration, rolling deployments, and auto-scaling integration. Use cases: web APIs, background workers, any continuously running process.
>
> **Rule of thumb:** If you want it to run forever — use a Service. If you want to run it once — use a Task.

---

## INTERMEDIATE (2-4 years)

### Q4: "What is the difference between ECS Task Role and Execution Role? This is one of the most common interview questions!"

**Power Answer:**
> **Execution Role:** Used by the **ECS Container Agent** to set up the task before the application starts:
> - Pull the container image from ECR (`ecr:BatchGetImage`)
> - Fetch secrets from Secrets Manager or SSM Parameter Store
> - Write container logs to CloudWatch Logs
> — Active BEFORE the task starts
>
> **Task Role:** Used by your **application code** at runtime:
> - Read/write S3
> - Query DynamoDB
> - Send messages to SQS
> - Call any AWS API
> — AWS SDKs automatically retrieve credentials from the task metadata endpoint at `169.254.170.2`
>
> **Simple memory trick:** "What does ECS need to start the task?" = Execution Role. "What does my application need to call AWS services?" = Task Role.
>
> When S3 access is denied → check Task Role. When image pull fails → check Execution Role.

---

### Q5: "When should you choose Fargate vs EC2 for ECS? What are the tradeoffs?"

**Power Answer:**
> **Choose Fargate when:**
> - You want to avoid managing EC2 patching, AMI updates, and agent management
> - Traffic is variable or spiky (you pay per task-second, not for idle instances)
> - Time-to-market is the priority
> - No GPU or special hardware requirements
> - You are in MVP/startup phase or the team has limited DevOps capacity
>
> **Choose EC2 when:**
> - GPU workloads for ML inference or AI — Fargate does not support GPU
> - Daemon services (one task per host) — Fargate does not support DAEMON scheduling
> - 24/7 predictable load — EC2 Reserved Instances provide 44-65% savings vs Fargate
> - Custom AMI requirements (security-hardened images, pre-installed agents)
> - Very high container density is needed (pack many tasks per large instance)
>
> **My production recommendation:** Start with Fargate. 90% of teams never need EC2. Add EC2 capacity providers only when you have GPU workloads, daemon service requirements, or when the Reserved Instance math clearly favors EC2 over Fargate On-Demand.

---

### Q6: "How does a rolling update work in ECS? How do you guarantee zero downtime?"

**Power Answer:**
> Rolling update is controlled by two parameters:
> - `minimumHealthyPercent`: The minimum percentage of desired tasks that must remain healthy during the deployment (e.g., 50% means at least 5 of 10 tasks must always be running)
> - `maximumPercent`: The maximum percentage of desired tasks that can run simultaneously (e.g., 200% means temporarily up to 2x desired tasks)
>
> Deployment flow:
> 1. New tasks launch (up to maximumPercent limit)
> 2. New tasks register with ALB and pass health checks
> 3. Old tasks deregister from ALB (connection drain = deregistrationDelay seconds)
> 4. Old tasks receive SIGTERM → graceful shutdown → stop
>
> **Zero downtime configuration:** `minimumHealthyPercent=100`, `maximumPercent=200` — new tasks start and pass health checks before any old task is removed. Capacity never drops below 100%.
>
> **Better option:** Blue/Green via CodeDeploy — all new tasks launch in parallel, traffic switches instantly at the ALB listener level, and rollback takes less than one second.

---

### Q7: "How does ECR image scanning work? What is the difference between Basic and Enhanced scanning?"

**Power Answer:**
> **Basic Scanning:**
> - Scanning engine: Clair (open-source)
> - Timing: On push only (one-time scan)
> - Scope: OS package CVEs only
> - Cost: Free
> - Critical limitation: CVEs published after the push are never detected — the image is never rescanned
>
> **Enhanced Scanning (AWS Inspector):**
> - Scanning engine: AWS Inspector + Snyk
> - Timing: **Continuously** — on push AND whenever a new CVE is published, all existing images matching that CVE are automatically rescanned
> - Scope: OS packages AND application dependencies (npm, pip, Maven, gem, Go modules)
> - Cost: Additional charge (~$0.09/image/month)
> - EventBridge events fired on new findings → can trigger Lambda to alert or block deployment
>
> **For production:** Always use Enhanced Scanning. Basic scanning gives you a one-time snapshot and then forgets the image. New CVEs are published daily — continuous rescanning is mandatory for any production workload.

---

## ADVANCED (4+ years / Staff/Principal level)

### Q8: "How does the ECS Scheduler decide where to place tasks? Explain in detail."

**Power Answer:**
> ECS Scheduler task placement is a 3-phase process:
>
> **Phase 1: Eligible Instance Filtering**
> - Only instances with connected Container Agents (not disconnected or draining)
> - Sufficient available CPU and Memory for the task
> - Correct launch type (EC2 instances for EC2-launch-type tasks)
>
> **Phase 2: Constraint Filtering**
> - `distinctInstance`: Each task must be on a different EC2 instance (eliminates hosts already running this service's tasks)
> - `memberOf` (cluster query language): Complex expressions like `attribute:ecs.instance-type =~ c5.*` or custom instance attributes
> - Instances not matching constraints are eliminated from the candidate pool
>
> **Phase 3: Strategy Application (on remaining candidates)**
> - `spread(attribute:ecs.availability-zone)`: Pick the host in the AZ with the fewest tasks of this service
> - `binpack(memory)`: Pick the host with the least remaining memory (maximize density)
> - `random`: Pick randomly from remaining candidates
> - Multiple strategies are evaluated in order — first strategy narrows candidates, subsequent strategies choose within those candidates
>
> **Best practice for production:**
> `spread(AZ)` then `binpack(memory)` = AZ balance is maintained WHILE maximizing instance density within each AZ. This gives you both high availability and cost efficiency.

---

### Q9: "How do you design a zero-downtime Blue/Green deployment in ECS? What is the rollback strategy?"

**Power Answer:**
> **Setup (CodeDeploy + ECS):**
> 1. Two ALB Target Groups: `blue-tg` (current production) and `green-tg` (initially empty)
> 2. One ALB production listener routing to `blue-tg`
> 3. One ALB test listener (port 8080) used for pre-production validation
>
> **Deployment Flow:**
> 1. CodeDeploy launches new tasks and registers them with `green-tg` (zero production traffic)
> 2. Test listener routes 100% traffic to `green-tg` → automated integration tests run
> 3. If tests pass → canary: 10% production traffic shifts to `green-tg`, monitored for 5 minutes
> 4. If healthy → full cutover: 100% production traffic to `green-tg` (millisecond listener switch)
> 5. Blue tasks remain running for the bake time window (e.g., 30 minutes)
>
> **Rollback:**
> - Within the bake time window: one API call switches the listener back to `blue-tg` — rollback in under 1 second
> - After bake time: Blue tasks are terminated — rollback requires redeploying the previous task definition revision
>
> **Key advantage over rolling update:** At no point during Blue/Green are mixed-version tasks serving production traffic simultaneously. Traffic is either 100% on the old version or 100% on the new version after cutover. This is critical for breaking API changes or any change requiring atomicity.

---

### Q10: "Production ECR + ECS costs have reached $10K/month. How would you optimize them?"

**Power Answer (structured approach):**
> **Step 1: Identify cost breakdown (Cost Explorer by service)**
> - What percentage is ECR storage vs ECS compute vs data transfer?
> - Which services consume the most Fargate compute?
>
> **Step 2: Quick wins (less than 1 week)**
> - **ECR Lifecycle Policies**: Keep only last 5-10 images per repository → immediate storage reduction
> - **Image size audit**: Multi-stage Docker builds → 50-70% image size reduction → storage + pull time savings
> - **Fargate Spot**: Migrate batch jobs and workers → 70% discount on eligible tasks
>
> **Step 3: Medium-term (1-4 weeks)**
> - **Right-size containers**: Use Container Insights to analyze actual CPU/Memory usage over 2-4 weeks → reduce task definition sizes by 20-40%
> - **VPC Endpoints**: Enable ECR and S3 endpoints → eliminate NAT Gateway data processing costs for image pulls
> - **Compute Savings Plans**: Analyze average Fargate spend → commit 1-year → 20-52% discount
>
> **Step 4: Architectural (4-12 weeks)**
> - **Cross-region optimization**: Enable ECR replication so services in other regions pull locally
> - **EC2 Reserved Instances for baseline load**: Identify services with constant 24/7 load → EC2 RIs vs Fargate = 40-60% savings
>
> **Expected outcome**: 40-65% cost reduction is achievable with all strategies combined.

---

### Q11: "What is ECS task ENI exhaustion? How do you solve it?"

**Power Answer:**
> **Problem:** In `awsvpc` networking mode, every task requires its own dedicated Elastic Network Interface (ENI). EC2 instances have hard ENI limits based on instance type:
> - t3.micro: 2 ENIs → 1 host ENI + only 1 task
> - c5.xlarge: 4 ENIs → 1 host ENI + 3 tasks maximum
> - c5.4xlarge: 8 ENIs → 1 host ENI + 7 tasks maximum
>
> Symptom: Tasks stuck in PROVISIONING state with "no available ENIs" error. Task density is severely constrained.
>
> **Solutions:**
> 1. **Larger instances**: c5.18xlarge supports 15 ENIs → 14 concurrent tasks
> 2. **ENI Trunking** (best solution): Enable `awsvpcTrunking` on the cluster → uses a "trunk ENI" technique to support up to 120 tasks per Nitro-based instance. Requires ECS Agent >= 1.55.0 and Nitro-based instances.
> 3. **Fargate**: No ENI limit concerns — AWS manages it transparently
> 4. **Mixed EC2 + Fargate**: Maintain known-density baseline on EC2, burst to Fargate for unlimited task count during peaks

---

### Q12: "Describe an ECS production incident and how you would debug it."

**Framework Answer (STAR method):**
> **Incident:** "Service degraded — 50% of requests returning 503"
>
> **Triage Steps:**
> 1. ALB metrics: HTTPCode_Target_5XX → which target group? → describe target health → identify unhealthy tasks
> 2. ECS: `describe-services` → running count vs desired count; events → "unable to place" or "task stopped"?
> 3. Stopped tasks: `describe-tasks` → `stoppedReason` + `containers[].exitCode`
>    - Exit 137 = OOM → increase memory limit immediately
>    - Exit 1 = App crash → check CloudWatch Logs for error
>    - TaskFailedToStart → image pull failure or secret fetch failed
> 4. CloudWatch Logs: `aws logs tail /ecs/myservice --follow` → find the actual error
> 5. If task is running but failing: `ecs execute-command` → exec into running container → investigate live
>
> **Common root causes I have seen:**
> - Memory leak causing OOM kills (exit 137) → gradually escalating over days → fix: increase memory limit, profile for leak
> - Bad deployment → circuit breaker triggered → check recent task definition changes → rollback
> - External dependency (RDS, Redis) failing → health check marking tasks unhealthy → fix the dependency or add circuit breaking
> - ECR image pull failing after Execution Role change → check CloudTrail for AccessDenied → restore ECR permissions

---

## BONUS: System Design Question

### Q13: "Design an ECS + ECR architecture for a Video Streaming Platform serving 100M users"

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
    transcoder jobs: EventBridge Pipe → SQS → ECS Task per video
    ML-recommendation: EC2 Graviton, 5-10 tasks (ARM64 optimized images)

Deployment:
  user-api: Blue/Green (CodeDeploy) → zero downtime, instant rollback
  transcoder: Rolling (batch jobs, can tolerate brief downtime)

Cost Optimization:
  Video processing (transcoders): Fargate Spot → 70% savings on GPU costs
  API layer: Compute Savings Plan (predictable baseline)
  ECR: Lifecycle policy → keep last 5 production images per service

Security:
  Payment handling: Fargate (MicroVM isolation)
  Each service: Task-specific IAM Role (least privilege)
  Secrets Manager: DB passwords, API keys (auto-rotation)
  Enhanced scanning: All ECR repositories

Observability:
  Container Insights + X-Ray + FireLens → OpenSearch
  Custom metrics: videos_processed_per_hour, streaming_latency_p99
  Alert: p99 > 5s → PagerDuty

Target: <200ms API latency (p99), 99.99% uptime
```

---

## Quick Reference Cheat Sheet

```
ECR Auth:
  Developer: aws ecr get-login-password | docker login
  ECS:       Auto via Execution Role (zero config!)
  CI/CD:     OIDC → IAM Role → no static keys

IAM Roles:
  Task Role      → App code (S3, DynamoDB, SQS)
  Execution Role → ECS Agent (ECR pull, logs, secrets)
  Instance Role  → EC2 host (ECS API calls)

Networking:
  bridge  → EC2 only, dynamic ports, shared EC2 security group
  host    → EC2 only, maximum performance, port conflicts possible
  awsvpc  → Fargate required, task-level security group, own ENI

Deployment:
  Rolling    → Simple, may have mixed versions simultaneously
  Blue/Green → Zero downtime, instant rollback, requires CodeDeploy
  Canary     → 1-10% traffic test first, gradual rollout

Scaling:
  Target Tracking → "Keep CPU at 70%"     → Autopilot
  Step Scaling    → "If CPU > 80%, +3"    → Manual thresholds
  Scheduled       → "Mon 8AM: +10 tasks"  → Known patterns

Exit Codes:
  0   → Clean exit (normal)
  1   → Generic error
  137 → OOM kill (increase memory!)
  143 → SIGTERM (graceful shutdown)

Debug Commands:
  Service health: aws ecs describe-services → .events
  Stopped tasks:  aws ecs describe-tasks → .stoppedReason + .exitCode
  Live logs:      aws logs tail /ecs/myapp --follow
  Live exec:      aws ecs execute-command --interactive --command /bin/sh
  CloudTrail:     aws cloudtrail lookup-events → AccessDenied errors
```

---

*You are now prepared for any AWS ECS/ECR interview, from entry level to Staff Engineer.*
