# 🚀 ECR Advanced Topics

---

## 📖 Topics Covered

1. Multi-architecture images
2. Cross-region replication
3. Pull-Through Cache (detailed)
4. Registry vs Repository level permissions
5. Vulnerability remediation workflow
6. CI/CD integration patterns

---

## 🏗️ 1. Multi-Architecture Images

### Why? The ARM Revolution
```
AWS Graviton3 (ARM64) chips:
- 40% better price/performance vs x86
- Same ECS task definition → run on Graviton (arm64) or Intel (amd64)

Problem: 
  docker build → produces image for ONE architecture
  Build on Intel → image not runnable on Graviton!

Solution: Multi-arch manifest lists (OCI Image Index)
```

### Building Multi-arch Images:
```bash
# Setup buildx (Docker multi-platform builder)
docker buildx create --use --name mybuilder --platform linux/amd64,linux/arm64
docker buildx inspect mybuilder --bootstrap

# Build and push multi-arch image to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v2.0.0 \
  --push \
  .

# ECR now stores BOTH architectures under ONE tag!
# ECS Fargate (amd64) pulls → gets amd64 version automatically
# ECS Graviton (arm64) pulls → gets arm64 version automatically
```

### Verify Multi-arch Manifest:
```bash
docker buildx imagetools inspect \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v2.0.0

# Output:
# Name: ...
# MediaType: application/vnd.oci.image.index.v1+json
# Digest: sha256:toplevel-manifest-hash
#
# Manifests:
#   Name: .../myapp:v2.0.0@sha256:amd64-hash
#   MediaType: application/vnd.oci.image.manifest.v1+json
#   Platform: linux/amd64
#
#   Name: .../myapp:v2.0.0@sha256:arm64-hash
#   MediaType: application/vnd.oci.image.manifest.v1+json
#   Platform: linux/arm64
```

---

## 🌍 2. Cross-Region Replication

### Architecture:
```
                    Source Region (us-east-1)
                    ┌─────────────────────────┐
                    │  ECR Registry             │
                    │  ├── myapp:v1  ─────────┤──→ Replication rule fires
                    │  └── nginx:latest        │
                    └─────────────────────────┘
                                │
                    (async replication, eventual consistency)
                                │
               ┌────────────────┼────────────────┐
               ▼                ▼                ▼
        ap-south-1       eu-west-1        ap-southeast-1
        ┌──────────┐    ┌──────────┐      ┌──────────┐
        │  myapp:v1│    │  myapp:v1│      │  myapp:v1│
        └──────────┘    └──────────┘      └──────────┘
        
        ECS in Mumbai  ECS in Dublin    ECS in Singapore
        pulls locally! pulls locally!   pulls locally!
        FREE transfer   FREE             FREE
```

### Configure Replication:
```bash
# Replicate all images to ap-south-1
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [
      {
        "destinations": [
          {"region": "ap-south-1", "registryId": "123456789012"},
          {"region": "eu-west-1", "registryId": "123456789012"}
        ],
        "repositoryFilters": [
          {
            "filter": "myapp/*",
            "filterType": "PREFIX_MATCH"
          }
        ]
      }
    ]
  }'

# Verify replication status
aws ecr describe-registry
```

### Cross-Account Replication:
```bash
# Replicate from Account A to Account B
# Step 1: Account B ko grant karo
# Account B (destination) ECR registry policy:
aws ecr put-registry-policy \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowReplicationFromAccountA",
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::111111111111:root"  // Account A
        },
        "Action": [
          "ecr:CreateRepository",
          "ecr:ReplicateImage"
        ],
        "Resource": "arn:aws:ecr:us-east-1:222222222222:repository/*"
      }
    ]
  }'

# Step 2: Account A configure replication to Account B
aws ecr put-replication-configuration --replication-configuration '{
  "rules": [{
    "destinations": [{"region": "us-east-1", "registryId": "222222222222"}]
  }]
}'
```

---

## 🔄 3. Pull-Through Cache — Deep Dive

### Problem it Solves:
```
Public registries:
  Docker Hub: 100 pulls/6hr (anonymous), 200/6hr (free user)
  In CI/CD: 10 services × pipelines = Docker Hub rate limit hit daily!
  Also: Public images going through NAT = cost + latency

Pull-Through Cache:
  ECR stores a cached copy of public images
  First pull: ECR downloads from source, caches
  Subsequent pulls: Served from ECR (fast + no rate limits + free within region)
```

### Supported Upstream Registries:
| Source | ECR Prefix | Auth Required? |
|--------|-----------|----------------|
| Docker Hub | `docker-hub/` | Optional (avoids rate limits) |
| ECR Public | `ecr-public/` | No |
| Quay.io | `quay/` | No |
| Kubernetes Registry (k8s.gcr.io) | `k8s/` | No |
| GitHub Container Registry | `ghcr/` | Yes |

### Setup Pull-Through Cache:
```bash
# Rule for Docker Hub (with auth for higher rate limits)
# First store Docker Hub credentials in Secrets Manager:
aws secretsmanager create-secret \
  --name ecr-pullthroughcache/dockerhub \
  --secret-string '{"username": "mydockerhubuser", "accessToken": "dckr_pat_xxx"}'

# Create rule
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "docker-hub" \
  --upstream-registry-url "registry-1.docker.io" \
  --credential-arn arn:aws:secretsmanager:us-east-1:123:secret:ecr-pullthroughcache/dockerhub

# Test it:
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:latest
# → ECR pulls from Docker Hub, caches, returns to you
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:latest
# → Served from ECR cache! Fast!
```

### ECS Task Definition with Pull-Through Cache:
```json
{
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:1.25",
      "portMappings": [{"containerPort": 80}]
    }
  ]
}
// No more Docker Hub rate limits!
// nginx image cached in ECR → layer dedup between your services
```

---

## 🔐 4. Registry vs Repository Level Permissions

### Two Levels of Access Control:

```
REGISTRY LEVEL (Registry Policy)
├── Cross-account replication sources
└── Registry-wide operations

    ≠

REPOSITORY LEVEL (Repository Policy)
├── Cross-account pull permissions
├── Specific repo access grants
└── Service-specific permissions
```

### Registry Policy (Account-wide):
```bash
# Registry policy applies to ENTIRE registry
# Use for: Granting cross-account replication rights
aws ecr put-registry-policy --policy-text '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCrossAccountReplication",
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::ACCOUNT-B:root"},
    "Action": ["ecr:CreateRepository", "ecr:ReplicateImage"],
    "Resource": "arn:aws:ecr:us-east-1:ACCOUNT-A:repository/*"
  }]
}'
```

### Repository Policy (Per-repository):
```bash
# Repository policy = specific permissions for specific repos
# Use for: Cross-account pulls, service-specific access

aws ecr set-repository-policy \
  --repository-name production/myapp \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowECSPull",
        "Effect": "Allow",
        "Principal": {
          "Service": "ecs-tasks.amazonaws.com"
        },
        "Action": [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability"
        ]
      },
      {
        "Sid": "AllowCrossAccountPull",
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::222222222222:role/ECSTaskExecutionRole"
        },
        "Action": [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage"
        ]
      }
    ]
  }'
```

### IAM Policy vs Repository Policy — Which Wins?
```
ECR Access = IAM Policy OR Repository Policy (EITHER is sufficient for same-account)
             IAM Policy AND Repository Policy (BOTH needed for cross-account)

Same account:
  IAM Policy allows → access granted (no repo policy needed)
  Repo Policy allows → access granted (no IAM policy needed)
  
Cross account:
  BOTH must allow:
  Source account: Repository policy must grant access
  Target account: IAM policy must allow ECR access
```

---

## 🔧 5. Vulnerability Remediation Workflow

### End-to-End Automated Remediation:
```
Flow:
                             New CVE Published
                                    │
                            AWS Inspector
                            detects in image
                                    │
                          EventBridge Event fired
                          {
                            "source": "aws.ecr",
                            "detail-type": "ECR Image Scan",
                            "detail": {
                              "finding-severity-counts": {"CRITICAL": 2}
                            }
                          }
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
              Lambda Function   SNS Topic      Security Hub
              - check severity  - Alert DevOps - Aggregate
              - create ticket   - Page on-call   findings
              - block deploy    - Slack msg
                    │
              If CRITICAL:
              - Find affected services (task defs using this image)
              - Create JIRA ticket with exact CVE details
              - Tag image as "vulnerable" in ECR
              - Block ECS service update if tag="vulnerable"
```

### Lambda for Automated Jira Ticket:
```python
import boto3
import json
import requests  # Jira API

def handle_ecr_finding(event, context):
    detail = event.get('detail', {})
    
    repo = detail.get('repository-name')
    tag = detail.get('image-tags', ['unknown'])[0]
    findings = detail.get('finding-severity-counts', {})
    
    # Get detailed findings
    ecr = boto3.client('ecr')
    scan_result = ecr.describe_image_scan_findings(
        repositoryName=repo,
        imageId={'imageTag': tag}
    )
    
    critical_findings = [
        f for f in scan_result['imageScanFindings']['findings']
        if f['severity'] == 'CRITICAL'
    ]
    
    for finding in critical_findings:
        # Create Jira ticket
        package = next((a['value'] for a in finding['attributes'] 
                       if a['key'] == 'package_name'), 'unknown')
        version = next((a['value'] for a in finding['attributes'] 
                       if a['key'] == 'package_version'), 'unknown')
        fixed_version = next((a['value'] for a in finding['attributes'] 
                             if a['key'] == 'fixed_in_versions'), 'check NVD')
        
        create_jira_ticket({
            'summary': f'[SECURITY] {finding["name"]} in {repo}:{tag}',
            'description': f'''
CVE: {finding["name"]}
Package: {package} v{version}
Fix: Upgrade to {fixed_version}
Repository: {repo}
Image: {tag}
Severity: CRITICAL
CVSS: {finding.get("attributes", {}).get("CVSS2_SCORE", "N/A")}
            ''',
            'priority': 'Critical',
            'labels': ['security', 'CVE', 'ECR']
        })
    
    return {'processed': len(critical_findings)}
```

---

## ⚙️ 6. CI/CD Integration

### Complete Pipeline (GitHub Actions → ECR → ECS):
```yaml
name: Deploy to ECS

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  ECS_SERVICE: myapp-service
  ECS_CLUSTER: production
  CONTAINER_NAME: myapp

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials (OIDC - no static keys!)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2
    
    - name: Build, tag, push
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
    
    - name: Wait for ECR scan (security gate!)
      run: |
        echo "Waiting for ECR scan..."
        sleep 60  # Give Inspector time to scan
        
        CRITICAL=$(aws ecr describe-image-scan-findings \
          --repository-name $ECR_REPOSITORY \
          --image-id imageTag=${{ github.sha }} \
          --query 'imageScanFindings.findingCounts.CRITICAL' \
          --output text)
        
        if [ "$CRITICAL" != "None" ] && [ "$CRITICAL" -gt "0" ]; then
          echo "CRITICAL vulnerabilities found: $CRITICAL"
          exit 1  # Block deployment!
        fi
        echo "Security gate passed!"
    
    - name: Download current task definition
      run: |
        aws ecs describe-task-definition \
          --task-definition myapp \
          --query taskDefinition > task-definition.json
    
    - name: Update task definition with new image
      id: task-def
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: task-definition.json
        container-name: ${{ env.CONTAINER_NAME }}
        image: ${{ steps.build-image.outputs.image }}
    
    - name: Deploy to ECS
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ env.ECS_SERVICE }}
        cluster: ${{ env.ECS_CLUSTER }}
        wait-for-service-stability: true
```

---

## 🚨 Gotchas & Edge Cases

### 1. Multi-arch Build Requires QEMU for Cross-Compilation
```bash
# Running arm64 build on x86 machine needs QEMU emulation
docker run --privileged --rm tonistiigi/binfmt --install all
# SLOW: 10x slower than native build
# Better: Use CI runners with both arches (AWS Graviton + Intel runners)
```

### 2. Cross-Region Replication → Eventual Consistency
```
You push image in us-east-1.
ECS in ap-south-1 immediately tries to deploy → image not there yet!
Replication can take seconds to minutes.

Solution: 
1. Wait before cross-region deploy
2. Or: Use EventBridge to trigger deploy only after replication confirms
```

### 3. Pull-Through Cache → Lifecycle Policy Applies
```
Pull-Through cached images subject to ECR lifecycle policies!
Set lifecycle policy → cached public images might get deleted.
Next pull → re-fetches from source (might hit rate limits again).
Always exclude pull-through cache repos from aggressive lifecycle policies.
```

---

## 🎤 Interview Angle

**Q: "Multi-region ECS deployment kaise design karein? ECR replication kaise kaam karti hai?"**

> Primary ECR registry ek region mein (maan lo us-east-1).
> Cross-region replication configure karo target regions ke liye.
> Replication async hai → deploy trigger hone se pehle confirm karo image replicated hai (EventBridge event).
> ECS in each region apne local region ka ECR use kare → no cross-region data transfer → free + fast.

**Q: "Docker Hub rate limits avoid karne ke liye ECR mein kya karte hain?"**

> Pull-Through Cache rules banate hain. Public registries (Docker Hub, Quay, ECR Public) ECR mein cache hote hain.
> Task definitions mein ECR Pull-Through Cache URL use karo instead of direct Docker Hub URL.
> Benefits: No rate limits, layer dedup with your own images, VPC access (no NAT), scanning support.

---

*Next: [Phase-2_Amazon-ECS-Core →](../Phase-2_Amazon-ECS-Core/01_What_Is_ECS.md)*
