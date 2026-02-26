# 🔐 ECR Authentication Model — Deep Dive

---

## 📖 Concept Explanation

ECR me authenticate karna Docker Hub se bilkul alag hai. ECR IAM use karta hai — **no username/password in code**.

### Authentication Flow — 3 Steps:

```
1. AWS IAM credentials verify karo
                ↓
2. ECR se temporary Docker token (12 hours valid) obtain karo
                ↓
3. Docker us token se login karo + pull/push karo
```

### The Key Command:
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
# Username hamesha "AWS" hota hai (literal string)
# Password = temporary Bearer token (12 hours)
```

---

## 🏗️ Internal Architecture — Auth Flow

```
┌──────────────┐     1. STS API Call      ┌──────────────────────┐
│   Your App   │ ─────────────────────────► │   AWS IAM / STS      │
│  EC2/Lambda  │ ◄───────────────────────── │  (Verify identity)   │
│  ECS Task    │     Temporary credentials  └──────────────────────┘
└──────┬───────┘
       │
       │ 2. ecr:GetAuthorizationToken
       │    (API call with IAM creds)
       ▼
┌──────────────────────────────────────────────┐
│         ECR Authorization Service            │
│                                              │
│  Validates IAM permission:                   │
│  ecr:GetAuthorizationToken ← on your entity  │
│                                              │
│  Returns: base64(AWS:token) valid 12 hours   │
└──────────────────┬───────────────────────────┘
                   │
                   │ 3. Docker login with token
                   ▼
┌──────────────────────────────────────────────┐
│         Docker Registry Auth                 │
│                                              │
│  Stores token in ~/.docker/config.json       │
│  Subsequent docker pull/push uses this token │
└──────────────────────────────────────────────┘
```

### What `get-login-password` Returns:
```bash
# Decode the token to see what's inside:
TOKEN=$(aws ecr get-login-password --region us-east-1)
echo $TOKEN | base64 -d
# → AWS:{"token":"...","expiresAt":"2024-01-15T14:30:00Z"}
#    ↑ Always "AWS" as username
#                ↑ Actual token (JWT-like)
#                                 ↑ 12 hours from now
```

---

## 🔑 Authentication Scenarios

### Scenario 1: Developer Local Machine
```bash
# AWS CLI configured
aws configure  # or use AWS_PROFILE, AWS_ACCESS_KEY_ID etc.

# One-time login (valid 12 hours)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Now push/pull freely for 12 hours
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

### Scenario 2: EC2 Instance with Instance Profile
```bash
# EC2 pe IAM Instance Profile attached hai
# No credentials needed → Instance Profile auto-provides!

# Simplified (AWS credential chain automatically resolves):
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# EC2 → automatically uses instance profile
# NO access key/secret needed!
```

### Scenario 3: ECS Tasks — ZERO AUTH CONFIG NEEDED!
```
ECS = Magic Land where ECR auth is AUTOMATIC!

How it works:
1. Task Definition mein image specify karo: 123456789.dkr.ecr.../myapp:v1
2. ECS Execution Role (Task Execution Role) ko ECR permissions do
3. ECS Container Agent automatically:
   - Gets token from ECR using Execution Role
   - Pulls image
   - Starts container
   
YOU DON'T RUN docker login IN ECS! It's handled automatically!
```

```json
// Execution Role Policy (minimum required):
{
  "Effect": "Allow",
  "Action": [
    "ecr:GetAuthorizationToken",        ← Get token
    "ecr:BatchCheckLayerAvailability",  ← Check if layers exist
    "ecr:GetDownloadUrlForLayer",        ← Get layer download URL
    "ecr:BatchGetImage"                 ← Get image manifest
  ],
  "Resource": "*"
}
```

### Scenario 4: GitHub Actions CI/CD
```yaml
# .github/workflows/deploy.yml
name: Build and Push to ECR

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    permissions:
      id-token: write   # ← For OIDC
      contents: read
    
    steps:
    - uses: actions/checkout@v4
    
    # OIDC-based (NO STATIC SECRETS NEEDED! Best practice)
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-ecr-push
        aws-region: us-east-1
    
    - name: Login to ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2
    
    - name: Build, tag, push to ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/myapp:$IMAGE_TAG .
        docker push $ECR_REGISTRY/myapp:$IMAGE_TAG
```

### Scenario 5: Cross-Account Pull
```
Account A (Dev): 111111111111
Account B (Prod): 222222222222

ECR is in Account A.
ECS runs in Account B.
How to pull image from Account A's ECR?

Step 1: Account A ECR pe repository policy add karo:
```
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CrossAccountPull",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:root"  // Account B
      },
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }
  ]
}
```
```bash
# Also: Account B's ECS Execution Role needs GetAuthorizationToken
# on Account A's registry:
{
  "Effect": "Allow",
  "Action": "ecr:GetAuthorizationToken",
  "Resource": "*"   ← GetAuthorizationToken doesn't support resource restriction
}
```

---

## 🎯 Analogy — Hotel Key Card System 🏨

**Traditional Docker Hub = Old fashioned key:**
- Username = room number
- Password = physical key
- Risk: Key copied/stolen = permanent access

**ECR Auth = Smart Hotel Key Card System:**
- IAM = Hotel reception (verifies your ID)
- `get-login-password` = Requests a temporary key card
- Key card valid = 12 hours
- After checkout (12 hours) → card auto-deactivates
- Hotel never gives you a permanent key
- IAM policy = "This guest can access floors 2-5 only"

---

## ⚙️ Hands-On Examples

### Token Management in Production:
```bash
# Cron job to refresh ECR token every 6 hours (safe buffer before 12hr expiry)
# /etc/cron.d/ecr-refresh
0 */6 * * * root aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# For Kubernetes (EKS):
# Use external-secrets operator or amazon-ecr-credential-helper
```

### ECR Credential Helper (No Manual Token Refresh):
```bash
# Install
go install github.com/awslabs/amazon-ecr-credential-helper/ecr-login/cli/docker-credential-ecr-login@latest

# Configure Docker (~/.docker/config.json)
{
  "credHelpers": {
    "123456789012.dkr.ecr.us-east-1.amazonaws.com": "ecr-login",
    "public.ecr.aws": "ecr-login"
  }
}

# Now docker automatically handles token refresh!
# No manual aws ecr get-login-password needed
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
# Token auto-fetched and refreshed by the helper!
```

### Repository Policy for Cross-Account:
```bash
# Set repository policy
aws ecr set-repository-policy \
  --repository-name myapp \
  --policy-text file://cross-account-policy.json

# Get current policy
aws ecr get-repository-policy --repository-name myapp

# Delete policy (reset to default)
aws ecr delete-repository-policy --repository-name myapp
```

---

## 🚨 Gotchas & Edge Cases

### 1. Token is 12 Hours — Not 24, Not Forever!
```bash
# This process fails after 12 hours:
aws ecr get-login-password | docker login ...
# ... 13 hours later ...
docker push ecr-url/myapp:latest
# Error: "no basic auth credentials" or "401 Unauthorized"

# Solution: Use credential helper or refresh before each push in CI/CD
```

### 2. `ecr:GetAuthorizationToken` is Account-Wide
```
This permission cannot be scoped to specific repositories!
It's always: "Resource": "*"

You CAN scope other ECR permissions:
"Resource": "arn:aws:ecr:us-east-1:123:repository/myapp"
But GetAuthorizationToken = always global!
```

### 3. Private vs Public ECR — Different Auth!
```bash
# Private ECR login (per region)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Public ECR login (us-east-1 ALWAYS, even for other regions!)
aws ecr-public get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  public.ecr.aws

# Public ECR push needs us-east-1 auth regardless of your location!
```

### 4. VPC Endpoint Requires Additional ECR Endpoint
```
ECR consists of TWO endpoints in VPC:
1. com.amazonaws.region.ecr.api     ← For API calls (GetAuthorizationToken, etc.)
2. com.amazonaws.region.ecr.dkr     ← For image layer pulls

Miss either one → "No connectivity" error even with VPC endpoint configured!
Also need: S3 gateway endpoint (layers stored in S3!)
```

---

## 🎤 Interview Angle

**Q: "ECS task ko ECR se image kaise milti hai? Authentication kaise kaam karta hai?"**

> ECS task start karte waqt, ECS Container Agent task definition padhta hai.
> Container Agent Task Execution Role assume karta hai.
> Us role se `ecr:GetAuthorizationToken` call karta hai → 12-hour token milta hai.
> Token se registry authenticate karke `ecr:BatchGetImage` + `ecr:GetDownloadUrlForLayer` calls karta hai.
> Image layers serial/parallel download hote hain.
> Sab kuch Container Agent automatically handle karta hai — developer kuchh configure nahi karta for auth.

**Q: "Cross-account ECR pull kaise setup karein?"**

> Two-step process:
> 1. Source account (ECR owner) mein repository policy add karo — target account/role ko pull permissions do.
> 2. Target account (ECS) ke Execution Role mein `ecr:GetAuthorizationToken` permission do (resource: *).
> Repository policy pulls, IAM policy token fetching allow karta hai. Dono chahiye!

---

*Next: [04_ECR_Security_Concepts.md →](./04_ECR_Security_Concepts.md)*
