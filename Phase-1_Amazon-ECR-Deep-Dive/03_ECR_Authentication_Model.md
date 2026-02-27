# ECR Authentication Model — Deep Dive

---

## Concept Explanation

Authenticating to ECR is fundamentally different from Docker Hub. ECR uses IAM — **no username/password embedded in code or CI secrets**.

### Authentication Flow — 3 Steps:

```
1. Verify AWS IAM credentials
                ↓
2. Obtain a temporary Docker token from ECR (valid 12 hours)
                ↓
3. Use that token to log in with Docker, then pull/push as needed
```

### The Key Command:
```bash
aws ecr get-login-password --region us-east-1 | \
  docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
# Username is always the literal string "AWS"
# Password = temporary Bearer token (valid 12 hours)
```

---

## Internal Architecture — Auth Flow

```
┌──────────────┐     1. STS API Call      ┌──────────────────────┐
│   Your App   │ ─────────────────────────► │   AWS IAM / STS      │
│  EC2/Lambda  │ ◄───────────────────────── │  (Verify identity)   │
│  ECS Task    │     Temporary credentials  └──────────────────────┘
└──────┬───────┘
       │
       │ 2. ecr:GetAuthorizationToken
       │    (API call with IAM credentials)
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
# Decode the token to inspect its contents:
TOKEN=$(aws ecr get-login-password --region us-east-1)
echo $TOKEN | base64 -d
# → AWS:{"token":"...","expiresAt":"2024-01-15T14:30:00Z"}
#    ↑ Always "AWS" as username
#                ↑ Actual token (JWT-like)
#                                 ↑ 12 hours from now
```

---

## Authentication Scenarios

### Scenario 1: Developer Local Machine
```bash
# AWS CLI configured with credentials
aws configure  # or use AWS_PROFILE, AWS_ACCESS_KEY_ID, etc.

# One-time login (valid 12 hours)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Push and pull freely for the next 12 hours
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
```

### Scenario 2: EC2 Instance with Instance Profile
```bash
# EC2 has an IAM Instance Profile attached.
# No credentials need to be hardcoded — the instance profile provides them automatically.

# The same command works; the AWS CLI resolves credentials from the metadata service:
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# No access key or secret key required!
```

### Scenario 3: ECS Tasks — Zero Auth Configuration Required
```
ECS handles ECR authentication completely automatically.

How it works:
1. Specify the ECR image URI in the Task Definition: 123456789.dkr.ecr.../myapp:v1
2. Grant ECR permissions to the ECS Task Execution Role
3. The ECS Container Agent automatically:
   - Obtains a token from ECR using the Task Execution Role
   - Pulls the image
   - Starts the container

You do NOT run docker login in ECS. It is handled transparently by the agent.
```

```json
// Minimum required permissions on the Task Execution Role:
{
  "Effect": "Allow",
  "Action": [
    "ecr:GetAuthorizationToken",        // Get token
    "ecr:BatchCheckLayerAvailability",  // Check if layers exist
    "ecr:GetDownloadUrlForLayer",        // Get layer download URL
    "ecr:BatchGetImage"                 // Get image manifest
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
      id-token: write   # Required for OIDC
      contents: read

    steps:
    - uses: actions/checkout@v4

    # OIDC-based authentication — NO static AWS secrets needed (best practice)
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
Goal: Allow Account B to pull images from Account A's ECR.

Step 1: Add a repository policy to Account A's ECR repository:
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
# Step 2: Account B's ECS Task Execution Role also needs GetAuthorizationToken.
# Note: GetAuthorizationToken cannot be scoped to a specific repository.
{
  "Effect": "Allow",
  "Action": "ecr:GetAuthorizationToken",
  "Resource": "*"   // GetAuthorizationToken always requires Resource: *
}
```

---

## Analogy — Hotel Key Card System

**Traditional Docker Hub = Old-fashioned physical key:**
- Username = room number
- Password = physical key
- Risk: Key copied or stolen = permanent access until manually revoked

**ECR Auth = Smart hotel key card system:**
- IAM = Hotel reception (verifies your identity)
- `get-login-password` = Requests a temporary key card
- Key card is valid for 12 hours
- After 12 hours, the card deactivates automatically
- The hotel never issues permanent keys
- IAM policy = "This guest may only access floors 2 through 5"

---

## Hands-On Examples

### Token Management in Production:
```bash
# Cron job to refresh ECR token every 6 hours (safe buffer before the 12-hour expiry)
# /etc/cron.d/ecr-refresh
0 */6 * * * root aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# For Kubernetes (EKS):
# Use the external-secrets operator or the amazon-ecr-credential-helper
```

### ECR Credential Helper (Automatic Token Refresh):
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

# Docker now handles token refresh automatically — no manual re-authentication needed.
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
# Token is fetched and refreshed transparently by the credential helper.
```

### Repository Policy for Cross-Account Access:
```bash
# Set a repository policy
aws ecr set-repository-policy \
  --repository-name myapp \
  --policy-text file://cross-account-policy.json

# Read the current policy
aws ecr get-repository-policy --repository-name myapp

# Remove the policy (resets to default — same-account IAM only)
aws ecr delete-repository-policy --repository-name myapp
```

---

## Gotchas & Edge Cases

### 1. Token is Valid for 12 Hours — Not Longer
```bash
# This workflow fails after 12 hours:
aws ecr get-login-password | docker login ...
# ... 13 hours pass ...
docker push ecr-url/myapp:latest
# Error: "no basic auth credentials" or "401 Unauthorized"

# Solution: Use the credential helper for long-lived processes,
# or re-authenticate immediately before each push in CI/CD.
```

### 2. `ecr:GetAuthorizationToken` Cannot Be Scoped to a Repository
```
This permission always applies account-wide. It cannot be restricted to specific repositories.
It must always be: "Resource": "*"

You CAN scope other ECR permissions to a specific repository:
"Resource": "arn:aws:ecr:us-east-1:123:repository/myapp"

But GetAuthorizationToken is always global — it grants the ability to
obtain a login token for the entire registry, not for individual repositories.
```

### 3. Private ECR and Public ECR Require Different Login Commands
```bash
# Private ECR login (per region — specify your region)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Public ECR login (ALWAYS use us-east-1, regardless of your deployment region!)
aws ecr-public get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  public.ecr.aws

# Attempting to authenticate to public.ecr.aws using any region other than
# us-east-1 will fail. This is a common gotcha.
```

### 4. VPC Endpoint Requires TWO Separate ECR Endpoints
```
ECR traffic uses two distinct VPC endpoints:
1. com.amazonaws.<region>.ecr.api     ← API calls (GetAuthorizationToken, etc.)
2. com.amazonaws.<region>.ecr.dkr     ← Image layer pulls (the actual data)

If either endpoint is missing, you will see connectivity errors even if one is configured.
Also required: an S3 gateway endpoint — because ECR layers are stored in S3!

Missing any of the three = "No connectivity" error during image pulls from a private subnet.
```

### 5. OIDC Is Always Preferred Over Static Access Keys in CI/CD
```
Never store AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY as GitHub secrets.
These are long-lived credentials that can be leaked via log output or misconfigured secrets.

Instead, configure an OIDC trust relationship between GitHub Actions and AWS IAM.
The pipeline assumes a role with short-lived credentials scoped to exactly what it needs.
This eliminates the risk of static credential leakage entirely.
```

---

## Interview Questions & Answers

**Q: "How does an ECS task obtain an image from ECR? Walk through the authentication flow."**

> When ECS starts a task, the ECS Container Agent reads the task definition to find the image URI. The agent assumes the Task Execution Role and calls `ecr:GetAuthorizationToken` to obtain a 12-hour authorization token. Using that token, the agent authenticates to the Docker registry and then makes `ecr:BatchGetImage` and `ecr:GetDownloadUrlForLayer` calls to retrieve the image manifest and download each layer. The entire process is handled by the agent — the developer does not configure any authentication for ECS tasks.

**Q: "How do you set up cross-account ECR pull?"**

> Two separate pieces of configuration are required:
> 1. In the source account (the ECR owner), add a repository policy granting the target account's IAM principal permission to call `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, and `ecr:BatchCheckLayerAvailability`.
> 2. In the target account (where ECS runs), ensure the Task Execution Role has permission to call `ecr:GetAuthorizationToken` with `Resource: *`.
> The repository policy controls who can pull layers; the IAM policy controls who can obtain a login token. Both must be present for cross-account pulls to succeed.

---

*Next: [04_ECR_Security_Concepts.md →](./04_ECR_Security_Concepts.md)*
