# ECR Advanced Topics

---

## Topics Covered

1. Multi-architecture images
2. Cross-region replication
3. Pull-Through Cache (detailed)
4. Registry vs Repository level permissions
5. Vulnerability remediation workflow
6. CI/CD integration patterns

---

## 1. Multi-Architecture Images

### Why? The ARM Revolution
```
AWS Graviton3 (ARM64) chips:
- 40% better price/performance compared to x86
- The same ECS task definition can run on Graviton (arm64) or Intel (amd64)

Problem:
  docker build → produces an image for ONE architecture only
  Build on Intel → image cannot run on Graviton!

Solution: Multi-arch manifest lists (OCI Image Index)
  A single tag points to a manifest list that contains separate
  platform-specific manifests. Docker pulls the correct one automatically.
```

### Building Multi-arch Images:
```bash
# Set up buildx (Docker multi-platform builder)
docker buildx create --use --name mybuilder --platform linux/amd64,linux/arm64
docker buildx inspect mybuilder --bootstrap

# Authenticate to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Build and push a multi-arch image to ECR
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v2.0.0 \
  --push \
  .

# ECR now stores BOTH architectures under a single tag.
# ECS Fargate (amd64) pulls → receives the amd64 variant automatically
# ECS Graviton (arm64) pulls → receives the arm64 variant automatically
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

## 2. Cross-Region Replication

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
# Replicate all images matching a prefix to ap-south-1 and eu-west-1
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

# Verify the replication configuration
aws ecr describe-registry
```

### Cross-Account Replication:
```bash
# Replicate from Account A to Account B.

# Step 1: Account B (destination) grants Account A permission to replicate into it.
# Run this command in Account B:
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

# Step 2: Configure replication in Account A to target Account B.
# Run this command in Account A:
aws ecr put-replication-configuration --replication-configuration '{
  "rules": [{
    "destinations": [{"region": "us-east-1", "registryId": "222222222222"}]
  }]
}'
```

---

## 3. Pull-Through Cache — Deep Dive

### Problem it Solves:
```
Public registry rate limits:
  Docker Hub: 100 pulls/6hr (anonymous), 200/6hr (free authenticated user)
  In a CI/CD environment: 10 services × multiple pipelines = rate limit hit daily!
  Also: Public images pulled through NAT Gateway = unnecessary cost + latency

Pull-Through Cache solution:
  ECR stores a cached copy of public images.
  First pull: ECR downloads from the source registry and caches the image.
  Subsequent pulls: Served directly from ECR — fast, no rate limits, no NAT cost.
```

### Supported Upstream Registries:
| Source | ECR Prefix | Auth Required? |
|--------|-----------|----------------|
| Docker Hub | `docker-hub/` | Optional (avoids rate limits with auth) |
| ECR Public | `ecr-public/` | No |
| Quay.io | `quay/` | No |
| Kubernetes Registry (k8s.gcr.io) | `k8s/` | No |
| GitHub Container Registry | `ghcr/` | Yes |

### Setup Pull-Through Cache:
```bash
# For Docker Hub with authentication (higher rate limits):
# First store Docker Hub credentials in Secrets Manager:
aws secretsmanager create-secret \
  --name ecr-pullthroughcache/dockerhub \
  --secret-string '{"username": "mydockerhubuser", "accessToken": "dckr_pat_xxx"}'

# Create the pull-through cache rule:
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "docker-hub" \
  --upstream-registry-url "registry-1.docker.io" \
  --credential-arn arn:aws:secretsmanager:us-east-1:123:secret:ecr-pullthroughcache/dockerhub

# Test it:
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:latest
# → ECR pulls from Docker Hub, caches the image, and returns it
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:latest
# → Served from ECR cache — fast and free within region!
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
// No more Docker Hub rate limits.
// The nginx image is cached in ECR — layer deduplication applies with your own images.
```

---

## 4. Registry vs Repository Level Permissions

### Two Levels of Access Control:

```
REGISTRY LEVEL (Registry Policy)
├── Cross-account replication sources
└── Registry-wide operations

    ≠

REPOSITORY LEVEL (Repository Policy)
├── Cross-account pull permissions
├── Specific repository access grants
└── Service-specific permissions
```

### Registry Policy (Account-wide):
```bash
# Registry policy applies to the ENTIRE registry.
# Primary use case: granting cross-account replication rights.
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
# Repository policy applies to a specific repository.
# Primary use cases: cross-account pulls, service-specific access grants.

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

### IAM Policy vs Repository Policy — Which One Wins?
```
ECR Access = IAM Policy OR Repository Policy (EITHER is sufficient for same-account access)
             IAM Policy AND Repository Policy (BOTH are required for cross-account access)

Same account:
  IAM Policy allows → access granted (no repo policy needed)
  Repo Policy allows → access granted (no IAM policy needed)

Cross account:
  BOTH must explicitly allow:
  Source account: Repository policy must grant the target account/role access
  Target account: IAM policy must allow the role to call ECR actions
```

---

## 5. Vulnerability Remediation Workflow

### End-to-End Automated Remediation:
```
Flow:
                             New CVE Published
                                    │
                            AWS Inspector
                            detects affected images
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
              - block deploy    - Slack message
                    │
              If CRITICAL:
              - Identify affected services (task defs using this image digest)
              - Create JIRA ticket with exact CVE details and fix version
              - Tag image as "vulnerable" in ECR
              - Block ECS service update if image tag = "vulnerable"
```

### Lambda for Automated JIRA Ticket Creation:
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

## 6. CI/CD Integration

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

    - name: Configure AWS credentials (OIDC — no static keys)
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

    - name: Wait for ECR scan (security gate)
      run: |
        echo "Waiting for ECR scan to complete..."
        sleep 60  # Allow Inspector time to complete the scan

        CRITICAL=$(aws ecr describe-image-scan-findings \
          --repository-name $ECR_REPOSITORY \
          --image-id imageTag=${{ github.sha }} \
          --query 'imageScanFindings.findingCounts.CRITICAL' \
          --output text)

        if [ "$CRITICAL" != "None" ] && [ "$CRITICAL" -gt "0" ]; then
          echo "CRITICAL vulnerabilities found: $CRITICAL — blocking deployment"
          exit 1
        fi
        echo "Security gate passed — proceeding with deployment"

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

## Gotchas & Edge Cases

### 1. Multi-arch Cross-Compilation Requires QEMU and Is Slow
```bash
# Building an arm64 image on an x86 machine requires QEMU emulation
docker run --privileged --rm tonistiigi/binfmt --install all
# Cross-compiled builds run approximately 10x slower than native builds.

# Better approach for production CI/CD:
# Use separate native runners for each architecture (AWS Graviton runner + Intel runner)
# and merge the manifests with docker buildx imagetools create.
# This gives native build speed for both platforms.
```

### 2. Cross-Region Replication Is Asynchronous — Not Instant
```
You push an image to us-east-1.
ECS in ap-south-1 immediately tries to deploy using the new image → image not available yet!
Replication can take anywhere from a few seconds to several minutes.

Solutions:
1. Add a wait step in your pipeline before triggering the cross-region deployment.
2. Use EventBridge to detect when the replicated image is available in the target region,
   and trigger the ECS deployment only after confirmation.
3. Use a CI/CD pipeline with regional stages gated on replication completion.
```

### 3. Pull-Through Cache Images Are Subject to Lifecycle Policies
```
Pull-Through cached images exist as normal ECR images within your registry.
If an aggressive lifecycle policy is applied to the entire registry,
it may delete cached public images.

The next pull from ECS would then re-fetch the image from the upstream source,
potentially hitting rate limits again.

Best practice: Exclude pull-through cache repository prefixes from
any lifecycle policies that aggressively expire images by age or count.
```

### 4. Cross-Account Replication Requires Registry Policy in the Destination Account
```
A common mistake: configuring replication on the source account but forgetting
to add the registry policy on the destination account.

Symptom: Images appear to replicate silently but never show up in the destination.
Check replication status with:
aws ecr describe-image-replication-status \
  --repository-name myapp \
  --image-id imageTag=v1.0.0
```

### 5. Immutable Tags and Multi-arch Manifests
```
With immutable tags enabled, you cannot push a new multi-arch manifest
to an existing tag if any of the referenced image digests change.

To update a multi-arch image with an immutable tag:
  1. Always use a new version tag (v2.0.1 instead of v2.0.0).
  2. Never rely on mutable tags like "latest" in production task definitions.
  3. Reference images by digest in production:
     123456789.dkr.ecr.us-east-1.amazonaws.com/myapp@sha256:abc123...
```

---

## Interview Questions & Answers

**Q: "How would you design a multi-region ECS deployment using ECR? How does replication work?"**

> The primary ECR registry is maintained in one region (e.g., us-east-1). Cross-region replication is configured to automatically copy images to all deployment regions. Because replication is asynchronous, CI/CD pipelines must wait for confirmation that the image has arrived in the target region before triggering ECS deployments there — this can be done using EventBridge events. Each regional ECS cluster pulls from its local ECR registry, which eliminates cross-region data transfer charges and reduces pull latency. The result is faster task startup and zero inter-region transfer costs at scale.

**Q: "How do you avoid Docker Hub rate limits in a production ECS environment?"**

> Configure Pull-Through Cache rules for Docker Hub and other public registries. Update ECS task definitions to reference images via the ECR pull-through cache URL instead of the upstream registry URL directly. On the first pull, ECR fetches and caches the image. All subsequent pulls are served from ECR within the region — with no rate limits, lower latency, VPC private network access, and the option to apply Enhanced Scanning to cached images.

**Q: "What is the difference between a registry policy and a repository policy in ECR?"**

> A registry policy is an account-level resource policy applied to the entire ECR registry. It is primarily used to grant cross-account permissions for replication operations (CreateRepository, ReplicateImage). A repository policy is applied to an individual repository and is used for more granular access control — for example, allowing a specific IAM role in another account to pull images from that particular repository. For same-account access, IAM identity policies alone are sufficient. For cross-account access, both the repository policy (on the source account's repository) and an IAM policy (on the target account's role) must allow the relevant actions.

---

*Next: [Phase-2_Amazon-ECS-Core →](../Phase-2_Amazon-ECS-Core/01_What_Is_ECS.md)*
