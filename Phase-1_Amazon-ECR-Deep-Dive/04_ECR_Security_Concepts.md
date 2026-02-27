# ECR Security Concepts

---

## Concept Explanation

ECR security is a multi-layered system. In production, simply pushing an image is not sufficient — you must also ensure:

1. **Vulnerable images do not reach production** (scanning)
2. **Unauthorized access is prevented** (IAM + repository policies)
3. **Data is tamper-proof** (immutable tags + encryption)
4. **Compliance requirements are met** (audit trail via CloudTrail)

---

## Image Scanning

### Two Levels of Scanning:

#### Basic Scanning (Included at no extra cost)
```
Technology: Clair (open-source)
When it runs: On push (if configured) or manually triggered
What it detects: OS-level CVEs only
Database: NVD (National Vulnerability Database)
Scan delay: Seconds to a few minutes
Cost: FREE
```

#### Enhanced Scanning (AWS Inspector Integration — Paid)
```
Technology: Amazon Inspector + Snyk
When it runs: Continuously — not just on push
   - New CVE published → existing images are automatically re-scanned
   - Detects language-level package vulnerabilities (not just OS)
What it detects:
   - OS packages (same as Basic)
   - Application dependencies:
     → Node.js (package.json)
     → Python (requirements.txt / setup.py)
     → Java (JAR, WAR, pom.xml)
     → Ruby (Gemfile)
     → Go binaries
     → .NET assemblies
Cost: Additional cost per image per month
```

### Scanning Architecture:
```
docker push myapp:v1.0
         │
         ▼
   ECR receives image
         │
         ▼ (if scanOnPush=true)
   BASIC SCAN:          ENHANCED SCAN:
   Clair engine         AWS Inspector agent
   OS layer scan        OS + app dependencies
   NVD database         Inspector vulnerability DB
         │                     │
         ▼                     ▼
   Results stored in    Results stored +
   ECR metadata         EventBridge events fired!
         │
         ▼
   Console shows:
   Critical: 3
   High: 12
   Medium: 45
   Low: 89
   Info: 120
```

---

## Vulnerability Severity Levels:

| Severity | CVSS Score | Recommended Action |
|----------|------------|-------------------|
| CRITICAL | 9.0 - 10.0 | **Block deployment immediately** |
| HIGH | 7.0 - 8.9 | Fix within 24-48 hours |
| MEDIUM | 4.0 - 6.9 | Fix within 30 days |
| LOW | 0.1 - 3.9 | Fix in next sprint |
| INFORMATIONAL | N/A | Best effort |

---

## Immutable Tags (Security Context)

```bash
# Enable at repository creation:
aws ecr create-repository \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# Enable on an existing repository:
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# What this prevents:
# An attacker with push access attempts to overwrite :production with a malicious image.
# WITH IMMUTABLE TAGS: Pushing to an existing tag FAILS → the attack is blocked.
```

Immutable tags also provide an important audit benefit: the tag `:v1.2.3` will always refer to the exact same image digest. This makes it possible to reliably trace what was deployed in production at any point in time.

---

## Encryption at Rest

### Two Encryption Options:

**AES-256 (default, no additional cost):**
```bash
aws ecr create-repository \
  --repository-name myapp \
  --encryption-configuration encryptionType=AES256
# AWS manages the encryption keys entirely.
# Appropriate for most general workloads.
```

**KMS (customer-managed keys):**
```bash
# Create your own KMS key (or use an existing one)
aws kms create-key \
  --description "ECR encryption key" \
  --key-usage ENCRYPT_DECRYPT

# Use the key for ECR:
aws ecr create-repository \
  --repository-name myapp \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123:key/abc-123

# Benefits of KMS:
# ✅ You control key rotation schedule
# ✅ CloudTrail logs every decrypt operation (who pulled what, and when)
# ✅ Disabling the key instantly prevents ALL image pulls — a powerful security lever
# ✅ Supports cross-account key sharing for complex multi-account architectures
```

---

## Lifecycle Policies

Lifecycle policies automate image cleanup to control storage costs and keep repositories tidy. Rules are evaluated in priority order (lower number = higher priority).

```json
// Example: Keep last 5 production images, delete everything older
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 5 images tagged with prod-",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["prod-"],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 3,
      "description": "Keep last 10 tagged images (any tag)",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": [""],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    }
  ]
}
```

---

## Real-World Scenario — Security Gate in CI/CD

```
Company: FinTech startup (PCI DSS compliance required)

Security Requirements:
1. No CRITICAL vulnerabilities allowed in production
2. Images must be built only from approved base images
3. Audit log of who pushed which image and when
4. Immutable production image tags

Implementation:

CI/CD Pipeline Security Gate:
─────────────────────────────────────────────────────
Code Commit (GitHub)
         │
         ▼
 Build Image (CodeBuild)
         │
         ▼ Push to ECR (dev-repo)
         │
         ▼ Trigger: ECR scan started (EventBridge)
         │
         ▼
 Lambda: Check scan results
   IF CRITICAL/HIGH found:
     → Block deployment (fail pipeline)
     → Alert security team (SNS)
     → Create JIRA ticket (via API)
   ELSE:
     → Tag image as 'approved' in ECR
     → Proceed to staging deployment

 Staging Deploy
         │
         ▼ Integration tests pass?
         │
         ▼
 Copy image to prod-repo (using digest, not tag)
         │
         ▼
 Production Deploy (immutable tag)
─────────────────────────────────────────────────────
```

---

## Hands-On Examples

### Setup Enhanced Scanning:
```bash
# Enable Enhanced Scanning for ALL repositories
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{
    "repositoryFilters": [
      {"filter": "*", "filterType": "WILDCARD"}
    ],
    "scanFrequency": "CONTINUOUS_SCAN"
  }]'

# AWS Inspector now continuously monitors all images in the registry.
```

### Check Scan Results:
```bash
# Get scan finding counts for a specific image
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=v1.0.0 \
  --query 'imageScanFindings.findingCounts'
# {
#   "CRITICAL": 2,
#   "HIGH": 15,
#   "MEDIUM": 42,
#   "LOW": 89,
#   "INFORMATIONAL": 12
# }

# Get detailed critical vulnerability information
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=v1.0.0 \
  --query 'imageScanFindings.findings[?severity==`CRITICAL`]' \
  | jq '.[] | {name:.name, description:.description, cvss:.attributes}'
```

### Block Deployment on Critical Vulnerabilities (Lambda Example):
```python
import boto3
import json

def check_ecr_scan(event, context):
    """EventBridge triggered when ECR scan completes"""
    ecr = boto3.client('ecr')

    detail = event['detail']
    repo = detail['repository-name']
    tag = detail['image-tags'][0]

    # Get findings
    response = ecr.describe_image_scan_findings(
        repositoryName=repo,
        imageId={'imageTag': tag}
    )

    counts = response['imageScanFindings']['findingCounts']
    critical = counts.get('CRITICAL', 0)
    high = counts.get('HIGH', 0)

    if critical > 0:
        # Block and send an alert
        print(f"BLOCKED: {repo}:{tag} has {critical} CRITICAL vulnerabilities!")
        # sns.publish(...)
        raise Exception("Critical vulnerabilities found — deployment blocked")
    elif high > 5:
        print(f"WARNING: {repo}:{tag} has {high} HIGH vulnerabilities")
        # Allow but alert

    # Approve: tag the image as 'security-approved'
    ecr.put_image(
        repositoryName=repo,
        imageManifest=response['imageManifest'],
        imageTag='security-approved'
    )

    return {"status": "approved", "repo": repo, "tag": tag}
```

### EventBridge Rule for Scan Events:
```json
{
  "source": ["aws.ecr"],
  "detail-type": ["ECR Image Scan"],
  "detail": {
    "scan-status": ["COMPLETE"],
    "repository-name": ["myapp"]
  }
}
```

---

## Gotchas & Edge Cases

### 1. Basic Scanning Runs Only Once per Image
```
Basic (Clair): Scans on push and never again.
If a new CVE is published tomorrow, existing images are NOT re-scanned.

Enhanced (Inspector): Continuous.
When a new CVE is published, Inspector re-scans all affected images automatically.
An EventBridge event is fired, which can trigger a Lambda to alert or block deployments.

For any compliance-sensitive workload: always use Enhanced Scanning.
```

### 2. Scanning Does NOT Automatically Block Pulls
```
ECR finds a CRITICAL vulnerability → the image remains pullable by default!
ECR does not automatically prevent pulling vulnerable images.

You must implement additional controls:
1. CI/CD pipeline gate: check scan results before deploying
2. Lambda + EventBridge: react to new findings automatically
3. OPA Admission Controller in Kubernetes: enforce at the cluster level
4. Custom deny logic in repository policies (limited support)
```

### 3. KMS Key Grant Required for Cross-Account Pulls of Encrypted Images
```
If repository A (Account 1) uses a customer-managed KMS key K1,
and Account 2 attempts to pull that image — decryption is required.
Account 2's role must be granted access to the KMS key.

aws kms create-grant \
  --key-id arn:aws:kms:...:key/K1 \
  --grantee-principal arn:aws:iam::222222222222:role/ecsTaskRole \
  --operations Decrypt GenerateDataKey

Without the KMS grant, the pull will fail with a decryption error,
even if the repository policy and IAM policies are correctly configured.
```

### 4. Lifecycle Policy Rule Order Is Critical
```
Rules are evaluated in ascending priority order (rule 1 evaluated first).
If an image matches rule 1 → it is scheduled for deletion;
rule 2 is never evaluated for that image.

Example:
  rule 1 (priority 1): "delete tagged images older than 30 days"
  rule 2 (priority 2): "keep last 5 images tagged with prod-*"

→ prod-* images older than 30 days are deleted by rule 1 before rule 2 can protect them!

Correct approach: Always assign lower priority numbers (higher precedence)
to "keep" rules that protect important images.
```

### 5. Basic Scan Results Expire
```
Basic scan results are retained for 30 days by default.
After 30 days, the scan findings are no longer visible in the console or API.
If you need long-term vulnerability records, export findings to Security Hub
or ship EventBridge events to an S3 bucket or SIEM.
```

---

## Interview Questions & Answers

**Q: "What is the difference between ECR Basic and Enhanced scanning? Which one should you use in production?"**

> Basic scanning: runs once on push, detects only OS-level CVEs, uses the Clair engine, and is free. Enhanced scanning: runs continuously, detects OS and application dependency CVEs, uses AWS Inspector and Snyk, and is paid but far more comprehensive. For production and compliance use cases, Enhanced scanning is always the right choice — new CVEs are published daily, and a one-time scan is insufficient. Enhanced scanning fires EventBridge events that allow Lambda functions to automatically alert or block vulnerable deployments.

**Q: "Why should image tags be immutable in production?"**

> Mutable tags allow a push to an existing tag to silently replace the underlying image. An attacker who gains push access could overwrite the `:production` tag with a malicious image, which all new ECS tasks would then pull. With immutable tags, any attempt to push to an existing tag fails with an error — this prevents tag-based supply chain attacks. An additional benefit is audit clarity: `:v1.2.3` will always refer to the exact same image digest, making deployments fully reproducible and traceable.

**Q: "A security scan finds a critical CVE in an image that is already running in production ECS. What do you do?"**

> Immediate steps:
> 1. Identify the affected package and check if a patched version exists.
> 2. Rebuild the image with the patched dependency and push to ECR.
> 3. Trigger a new ECS deployment (rolling update) to replace running tasks with the patched image.
> 4. Tag the vulnerable image digest in ECR with a label like `cve-blocked` and update lifecycle policies to prevent it from being re-deployed.
> 5. If the CVE is exploitable and critical (e.g., CVSS 9.8+), consider stopping the affected tasks immediately rather than waiting for a rolling update.
> Long term: set up EventBridge + Lambda to automatically trigger deployments when Critical findings are detected.

---

*Next: [05_ECR_Cost_Components.md →](./05_ECR_Cost_Components.md)*
