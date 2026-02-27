# CI/CD Full Pipeline — Code to Production

---

## Complete Pipeline Architecture

```
Developer pushes code to main branch
         │
         ▼ (GitHub/GitLab/CodeCommit webhook)
  ┌──────────────┐
  │ Source Stage  │  GitHub Actions / CodePipeline
  └──────┬───────┘
         │
         ▼
  ┌──────────────────────────┐
  │ Build Stage (CodeBuild)  │
  │  1. Run unit tests        │
  │  2. docker build          │
  │  3. docker push to ECR   │
  │  4. Output: image URI     │
  └──────────────┬───────────┘
                 │
                 ▼ (Trigger Lambda)
  ┌──────────────────────────┐
  │ Security Gate             │
  │  Wait for ECR scan        │
  │  Check: CRITICAL = 0?     │
  │  If yes: proceed; else FAIL│
  └──────────────┬───────────┘
                 │
                 ▼
  ┌──────────────────────────┐
  │ Staging Deploy            │
  │  Update task definition   │
  │  Deploy to staging ECS    │
  │  Run integration tests    │
  └──────────────┬───────────┘
                 │
                 ▼ (Manual approval OR auto)
  ┌──────────────────────────┐
  │ Production Deploy         │
  │  Blue/Green (CodeDeploy)  │
  │  Monitor 30 min (bake)    │
  │  Rollback if alerts fire  │
  └──────────────────────────┘
```

---

## GitHub Actions — Complete Production Pipeline

```yaml
# .github/workflows/deploy.yml
name: CI/CD — Build, Test, Deploy to ECS

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  ECS_CLUSTER: production
  ECS_SERVICE: myapp-service
  CONTAINER_NAME: app

permissions:
  id-token: write   # OIDC auth — no static AWS keys stored!
  contents: read

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - run: npm ci
      - run: npm test
      - run: npm run lint

  build-and-push:
    name: Build Image & Push to ECR
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'  # Only on main branch

    outputs:
      image: ${{ steps.build.outputs.image }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}
          role-session-name: GitHubActions-${{ github.run_id }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, push to ECR
        id: build
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build with build metadata
          docker build \
            --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
            --build-arg GIT_COMMIT=${{ github.sha }} \
            --build-arg GIT_BRANCH=${{ github.ref_name }} \
            -t $REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $REGISTRY/$ECR_REPOSITORY:latest \
            .

          docker push $REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $REGISTRY/$ECR_REPOSITORY:latest

          echo "image=$REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Get image digest for traceability
        run: |
          DIGEST=$(docker inspect ${{ steps.login-ecr.outputs.registry }}/$ECR_REPOSITORY:${{ github.sha }} --format='{{index .RepoDigests 0}}')
          echo "Image digest: $DIGEST"
          echo "DIGEST=$DIGEST" >> $GITHUB_ENV

  security-gate:
    name: Security Scan Gate
    needs: build-and-push
    runs-on: ubuntu-latest

    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Wait for ECR scan and check results
        run: |
          echo "Waiting 90 seconds for ECR scan to complete..."
          sleep 90

          # Get scan findings
          RESULT=$(aws ecr describe-image-scan-findings \
            --repository-name $ECR_REPOSITORY \
            --image-id imageTag=${{ github.sha }} \
            --query 'imageScanFindings.findingCounts' \
            --output json)

          echo "Scan results: $RESULT"

          CRITICAL=$(echo $RESULT | jq -r '.CRITICAL // 0')
          HIGH=$(echo $RESULT | jq -r '.HIGH // 0')

          if [ "$CRITICAL" -gt "0" ]; then
            echo "DEPLOYMENT BLOCKED: $CRITICAL CRITICAL vulnerabilities found!"
            echo "Fix the vulnerabilities and redeploy."
            exit 1
          fi

          if [ "$HIGH" -gt "10" ]; then
            echo "WARNING: $HIGH HIGH vulnerabilities found. Proceeding but alert sent."
            # Optionally: send Slack alert
          fi

          echo "Security gate passed!"

  deploy-staging:
    name: Deploy to Staging
    needs: security-gate
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition myapp-staging \
            --query taskDefinition > task-def.json

      - name: Update task definition with new image
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-def.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.build-and-push.outputs.image }}

      - name: Deploy to staging ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: myapp-staging-service
          cluster: staging
          wait-for-service-stability: true

  integration-tests:
    name: Run Integration Tests
    needs: deploy-staging
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run integration tests against staging
        env:
          STAGING_URL: https://staging.myapp.com
        run: |
          npm ci
          npm run test:integration

  deploy-production:
    name: Deploy to Production (Blue/Green)
    needs: integration-tests
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval in GitHub!

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}

      - name: Download current task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition myapp \
            --query taskDefinition > task-def-prod.json

      - name: Update task definition
        id: task-def-prod
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-def-prod.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.build-and-push.outputs.image }}

      - name: Register new task definition
        run: |
          TASK_DEF_ARN=$(aws ecs register-task-definition \
            --cli-input-json file://${{ steps.task-def-prod.outputs.task-definition }} \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)
          echo "TASK_DEF_ARN=$TASK_DEF_ARN" >> $GITHUB_ENV

      - name: Deploy via CodeDeploy (Blue/Green)
        run: |
          aws deploy create-deployment \
            --application-name myapp-deploy \
            --deployment-group-name myapp-dg \
            --revision '{
              "revisionType": "AppSpecContent",
              "appSpecContent": {
                "content": "{\"version\":0.0,\"Resources\":[{\"TargetService\":{\"Type\":\"AWS::ECS::Service\",\"Properties\":{\"TaskDefinition\":\"'"$TASK_DEF_ARN"'\",\"LoadBalancerInfo\":{\"ContainerName\":\"app\",\"ContainerPort\":3000}}}}]}"
              }
            }'

      - name: Post-deployment notification
        if: success()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-type: application/json' \
            -d '{"text": "Production deployment successful! Commit: ${{ github.sha }} by ${{ github.actor }}"}'
```

---

## OIDC Setup — No Static AWS Credentials

```bash
# One-time setup: GitHub OIDC Provider in AWS

# Create OIDC Identity Provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Create IAM Role for GitHub Actions
cat > github-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:MyOrg/MyRepo:*"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name github-actions-deploy \
  --assume-role-policy-document file://github-trust-policy.json

# Attach required permissions:
aws iam attach-role-policy \
  --role-name github-actions-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess

# Add ECR permissions
aws iam attach-role-policy \
  --role-name github-actions-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

# Now GitHub Actions can use this role WITHOUT storing any AWS access keys!
```

OIDC is the modern best practice for CI/CD authentication to AWS. The alternative — storing long-lived AWS access keys as GitHub secrets — creates a significant security risk. Those keys can be accidentally committed to code, included in logs, or compromised if the secrets store is breached. OIDC tokens are short-lived (1 hour) and scoped to specific repositories and branches.

---

## Pipeline Security Best Practices

```
Image Tagging Strategy:
  Tag images with Git SHA → full traceability
  git SHA → ECR image → ECS task → production request
  "Which commit is running right now?" → immediately answerable

  Bad: tag=latest (no traceability, any push overwrites it!)
  Good: tag=$GITHUB_SHA (immutable, traceable)

Least Privilege IAM:
  GitHub Actions role:
    - ecr:GetAuthorizationToken (resource: *)
    - ecr:BatchGetImage, ecr:PutImage (resource: specific repo ARN)
    - ecs:RegisterTaskDefinition (resource: specific task def ARN)
    - ecs:UpdateService (resource: specific service ARN)
    - codedeploy:CreateDeployment (resource: specific deployment group)

  Avoid: AmazonECS_FullAccess (too broad for CI/CD)

Environment Isolation:
  staging/ and production/ environments in GitHub
  production requires manual approval
  Different IAM roles for staging vs production deployments
```

---

## Rollback Strategy in CI/CD

```bash
# Emergency rollback script:
#!/bin/bash
CLUSTER=$1
SERVICE=$2
PREVIOUS_REVISION=$3

# Get the previous task definition ARN
TASK_DEF_ARN=$(aws ecs describe-task-definition \
  --task-definition ${SERVICE}:${PREVIOUS_REVISION} \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

# Update service to previous revision
aws ecs update-service \
  --cluster $CLUSTER \
  --service $SERVICE \
  --task-definition $TASK_DEF_ARN

# Wait for rollback to complete
aws ecs wait services-stable \
  --cluster $CLUSTER \
  --services $SERVICE

echo "Rollback complete to revision $PREVIOUS_REVISION"
```

---

## Interview Angle

**Q: "How do you integrate ECR and ECS in a CI/CD pipeline? What are the best practices?"**

> The complete flow is: code push triggers CI → build stage runs unit tests → Docker build creates image with git SHA tag → push to ECR → wait for ECR vulnerability scan → security gate blocks deployment if CRITICAL vulnerabilities found → deploy to staging → run integration tests against staging → manual approval gate → Blue/Green production deployment via CodeDeploy → monitor 30-minute bake period → automated rollback if CloudWatch alarms fire.
>
> Best practices:
> 1. OIDC authentication — use GitHub OIDC to assume an IAM role rather than storing static AWS access keys in GitHub secrets
> 2. Tag images with the git SHA — provides full traceability from production request back to the specific commit
> 3. Security gate — block deployment if CRITICAL CVEs are found in the ECR scan results
> 4. Staging validation — run integration tests against a real staging environment before allowing production deployment
> 5. Blue/Green in production — zero-downtime deployment with instant rollback capability
> 6. Bake time — keep the Blue environment alive for 30 minutes after the Green cutover as a rollback insurance window

---

*Next: [04_Edge_Cases_And_Debugging.md →](./04_Edge_Cases_And_Debugging.md)*
