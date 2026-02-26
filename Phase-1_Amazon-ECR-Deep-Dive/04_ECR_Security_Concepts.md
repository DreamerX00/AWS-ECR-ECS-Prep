# 🛡️ ECR Security Concepts

---

## 📖 Concept Explanation

ECR security ek multi-layered system hai. Production mein sirf "image push kar diya" kaafi nahi — tumhe ye bhi ensure karna hai ki:

1. **Galat image production mein na jaaye** (scanning)
2. **Unauthorized access na ho** (IAM + repo policies)  
3. **Data tamper-proof ho** (immutable tags + encryption)
4. **Compliance requirements meet hon** (audit trail)

---

## 🔍 Image Scanning

### Two Levels of Scanning:

#### 🟡 Basic Scanning (Free tier included)
```
Technology: Clair (open-source)
When runs: On push (if configured) or manual
What detects: OS-level CVEs only
Database: NVD (National Vulnerability Database)
Delay: Seconds to few minutes
Cost: FREE
```

#### 🟢 Enhanced Scanning (AWS Inspector Integration — Paid)
```
Technology: Amazon Inspector + Snyk
When runs: Continuously! Not just on push
   - New CVE published → existing images rescan
   - Language package vulnerabilities (not just OS)
What detects:
   - OS packages (like Basic)
   - Application dependencies:
     → Node.js (package.json)
     → Python (requirements.txt / setup.py)
     → Java (JAR, WAR, pom.xml)
     → Ruby (Gemfile)
     → Go binaries
     → .NET assemblies
Cost: Additional cost per image
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
   OS layer scan        OS + app deps
   NVD database         Inspector vuln DB
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

| Severity | CVSS Score | Action Required |
|----------|------------|-----------------|
| CRITICAL | 9.0 - 10.0 | **Block deployment immediately** |
| HIGH | 7.0 - 8.9 | Fix within 24-48 hours |
| MEDIUM | 4.0 - 6.9 | Fix within 30 days |
| LOW | 0.1 - 3.9 | Fix in next sprint |
| INFORMATIONAL | N/A | Best effort |

---

## 🔒 Immutable Tags (Revisited in Security Context)

```bash
# Enable at repository creation:
aws ecr create-repository \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# Enable on existing repo:
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE

# What this prevents:
# Attacker gains push access → tries to overwrite :production tag with malicious image
# WITH IMMUTABLE TAGS: Push to existing tag FAILS → Attack blocked!
```

---

## 🗝️ Encryption at Rest

### Two Encryption Options:

**AES-256 (default, free):**
```bash
aws ecr create-repository \
  --repository-name myapp \
  --encryption-configuration encryptionType=AES256
# AWS manages keys, you have no control over them
# Fine for most workloads
```

**KMS (customer-managed keys):**
```bash
# Create your own KMS key first (or use existing)
aws kms create-key \
  --description "ECR encryption key" \
  --key-usage ENCRYPT_DECRYPT

# Use it for ECR:
aws ecr create-repository \
  --repository-name myapp \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123:key/abc-123

# Benefits of KMS:
# ✅ Key rotation control
# ✅ CloudTrail logs every decrypt operation (who pulled what, when)
# ✅ Disable key → instantly prevent ALL pulls!
# ✅ Cross-account key sharing for complex architectures
```

---

## 📋 Lifecycle Policies

```json
// Keep last 5 PRODUCTION images, delete everything older
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
      "description": "Keep last 10 any tagged images",
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

## 🌍 Real-World Scenario — Security Gate in CI/CD

```
Company: FinTech startup (PCI DSS compliance required)

Security Requirements:
1. No CRITICAL vulnerabilities in production
2. Images must be built from approved base images only
3. Audit log of who pushed what image
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
 Copy image to prod-repo (digest-based, not tag)
         │
         ▼
 Production Deploy (immutable tag)
─────────────────────────────────────────────────────
```

---

## ⚙️ Hands-On Examples

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

# Result: Inspector now continuously monitors ALL images!
```

### Check Scan Results:
```bash
# Get scan findings for specific image
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

# Get detailed critical vulnerabilities
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
        # Block! Send alert!
        print(f"BLOCKED: {repo}:{tag} has {critical} CRITICAL vulnerabilities!")
        # sns.publish(...)
        raise Exception("Critical vulnerabilities found — deployment blocked")
    elif high > 5:
        print(f"WARNING: {repo}:{tag} has {high} HIGH vulnerabilities")
        # Still allow but alert
    
    # Approve: tag image as 'security-approved'
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

## 🚨 Gotchas & Edge Cases

### 1. Basic Scan Runs ONCE Per Image
```
Basic (Clair): Scans on push, never again
New CVE published tomorrow → old images NOT re-scanned

Enhanced (Inspector): Continuous!
New CVE published → Inspector rescans affected images
→ EventBridge event fired → Lambda can alert/block

For compliance: USE ENHANCED SCANNING
```

### 2. Scanning Doesn't Block Pull by Default
```
ECR scan finds CRITICAL vulnerability → image still pullable!
ECR doesn't automatically prevent pulling vulnerable images.

You must implement:
1. CI/CD gate (check before deploy)
2. Lambda + EventBridge (react to findings)
3. OPA/Admission controller in K8s
4. Custom ECR resource policy with conditions (limited support)
```

### 3. KMS Key Grant Required for Cross-Account
```
Repo A (Account 1) uses KMS key K1
Account 2 tries to pull → Decryption needed → Needs KMS key access!

Must add KMS key grant for Account 2:
aws kms create-grant \
  --key-id arn:aws:kms:...:key/K1 \
  --grantee-principal arn:aws:iam::222222222222:role/ecsTaskRole \
  --operations Decrypt GenerateDataKey
```

### 4. Lifecycle Policy Order Matters!
```
Rules evaluated in rulePriority ORDER (1 first, then 2, etc.)
If image matches rule 1 → deleted, rule 2 never evaluated for it
If rule 1: "delete tagged older than 30 days"
   rule 2: "keep last 5 with tag prod-*"
→ prod-* images older than 30 days DELETED by rule 1 before rule 2 can save them!

ALWAYS put "keep" rules with lower priority numbers (higher priority)
```

---

## 🎤 Interview Angle

**Q: "ECR Basic vs Enhanced scanning mein kya difference hai? Production ke liye kaunsa?"**

> Basic: Runs once on push, OS CVEs only, Clair engine, free.
> Enhanced: Continuous scanning, OS + application dependencies, AWS Inspector + Snyk, paid but much more comprehensive.
> Production/compliance: Always Enhanced. New CVEs published daily — one-time scan is insufficient.
> Enhanced triggers EventBridge → Lambda can automatically block vulnerable deployments.

**Q: "Image tags immutable kyu karne chahiye production mein?"**

> Mutable tags = attacker gains push access → overwrite `:production` tag with malicious image → all new ECS tasks pull malicious image.
> Immutable tags = push to existing tag FAILS → supply chain attack blocked.
> Extra benefit: Audit clarity — `:v1.2.3` hamesha wahi exact image hai.

---

*Next: [05_ECR_Cost_Components.md →](./05_ECR_Cost_Components.md)*
