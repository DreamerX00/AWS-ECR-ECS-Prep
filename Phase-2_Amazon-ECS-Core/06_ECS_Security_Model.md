# 🔐 ECS Security Model — IAM Roles & Networking

---

## 📖 The 3 IAM Roles of ECS

**This is the #1 interview topic!** Most candidates confuse these 3 roles.

```
┌──────────────────────────────────────────────────────────────────┐
│                    ECS ROLES SUMMARY                              │
│                                                                   │
│  TASK ROLE          = What your APP CODE can do in AWS           │
│  EXECUTION ROLE     = What ECS AGENT can do on your behalf        │
│  INSTANCE ROLE      = What the EC2 HOST can do in AWS            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ TASK ROLE — Your Application's Identity

```
Who uses it?     Your application code running inside the container
What for?        AWS SDK calls your app makes at runtime
When active?     While task is RUNNING
```

### Example:
```python
# Inside your container, your app code does this:
import boto3

# AWS SDK automatically uses Task Role credentials!
# (from http://169.254.170.2/v2/credentials/<id>)
s3 = boto3.client('s3')
s3.get_object(Bucket='my-data-bucket', Key='file.json')

# DynamoDB:
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('users')
table.get_item(Key={'user_id': '123'})
```

### Task Role Policy (Least Privilege):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-data-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123:table/users"
    }
  ]
}
```

### Task Role Trust Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "ecs-tasks.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

### Set in Task Definition:
```json
{
  "family": "myapp",
  "taskRoleArn": "arn:aws:iam::123:role/myapp-task-role",  ← Here!
  ...
}
```

---

## 2️⃣ TASK EXECUTION ROLE — ECS Agent's Identity

```
Who uses it?     ECS Container Agent (not your app!)
What for?        
  - Pull image from ECR
  - Fetch secrets from Secrets Manager / SSM
  - Write logs to CloudWatch
  - Send metrics to CloudWatch Container Insights
When active?     During TASK LAUNCH (mostly before app code runs)
```

### Required Permissions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",            ← ECR auth token
        "ecr:BatchCheckLayerAvailability",      ← Layer check before pull
        "ecr:GetDownloadUrlForLayer",           ← Download layer
        "ecr:BatchGetImage"                     ← Pull image manifest
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",                 ← CloudWatch logs
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:123:log-group:/ecs/myapp:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",        ← Fetch DB passwords etc.
        "ssm:GetParameter",
        "ssm:GetParameters",
        "kms:Decrypt"                           ← Decrypt KMS-encrypted secrets
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123:secret:myapp/*",
        "arn:aws:ssm:us-east-1:123:parameter/myapp/*"
      ]
    }
  ]
}
```

### Set in Task Definition:
```json
{
  "family": "myapp",
  "executionRoleArn": "arn:aws:iam::123:role/ecsTaskExecutionRole",  ← Here!
  ...
}
```

> 💡 AWS provides: `AmazonECSTaskExecutionRolePolicy` managed policy (ECR pull + CloudWatch logs). Extend it for secrets access.

---

## 3️⃣ INSTANCE ROLE — EC2 Host's Identity (EC2 mode only)

```
Who uses it?     EC2 instance hosting ECS tasks
What for?:
  - ECS Container Agent ↔ ECS Control Plane communication
  - CloudWatch Logs agent
  - SSM Session Manager (if enabled)
When active?     Always (as long as EC2 is running)
```

### Required Permissions:
```json
// AWS Managed: AmazonEC2ContainerServiceforEC2Role
{
  "Version": "2012-10-17", 
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ecs:CreateCluster",
      "ecs:DeregisterContainerInstance",
      "ecs:DiscoverPollEndpoint",
      "ecs:Poll",
      "ecs:RegisterContainerInstance",
      "ecs:StartTelemetrySession",
      "ecs:Submit*",
      "ecr:GetAuthorizationToken",
      "ecr:BatchCheckLayerAvailability",
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]
  }]
}
```

> ⚠️ **Never give application-level permissions to Instance Role!**
> All tasks on that EC2 would inherit those permissions!
> Use Task Role for application permissions.

---

## 🔑 Role Summary — Side by Side

```
Scenario: ECS Fargate task runs Node.js app that reads from S3 and DynamoDB
+---------------------+------------------+---------------------------+
| Role                | Used By          | Permissions                |
+---------------------+------------------+---------------------------+
| Task Role           | Node.js app code | s3:GetObject, dynamo:Query |
| Execution Role      | ECS Agent        | ecr:*, logs:*, secrets:*   |
| Instance Role       | EC2 host (N/A for Fargate)                      |
+---------------------+------------------+---------------------------+

Fargate mein Instance Role concept nahi hota — no EC2 to have a role!
```

---

## 🛡️ Security Best Practices

### 1. Never Use Instance Credentials in Containers
```bash
# BAD: Container gets EC2 credentials (broad access!)
# EC2 instance profile = any container on that host = all permissions!

# GOOD: Give each task ONLY the permissions it needs
{
  "taskRoleArn": "arn:aws:iam::123:role/payment-service-role",
  # Only: sqs:SendMessage, dynamodb:PutItem (minimum necessary!)
}
```

### 2. Prevent Credential Sharing Between Tasks
```bash
# ECS_ENABLE_TASK_IAM_ROLE=true in ecs.config (default in modern AMIs)
# Prevents container from calling EC2 metadata endpoint directly
# Each task gets its own credential endpoint

# Block EC2 metadata from containers with custom network rules:
# ECS creates iptables rules to prevent containers from accessing
# http://169.254.169.254 (EC2 IMDS) → forces use of task-level credentials
```

### 3. SSM Session Manager > SSH
```bash
# Don't open port 22! Use ECS Exec (SSM-based)
# Requirements:
# 1. Task role must have ssmmessages:* permissions
# 2. Service must have enableExecuteCommand=true

aws ecs update-service \
  --cluster production \
  --service myapp \
  --enable-execute-command

# Then exec into container:
aws ecs execute-command \
  --cluster production \
  --task <task-arn> \
  --container app \
  --interactive \
  --command "/bin/bash"

# Audit: Every exec session logged in CloudTrail + S3 (if configured)
```

---

## 🌍 Real-World Scenario: Microservices Security

```
E-commerce platform security design:

payment-service task:
  Task Role: "payment-task-role"
    → KMS:Decrypt (payment keys)
    → DynamoDB:GetItem, PutItem (orders table ONLY)
    → SQS:SendMessage (payment-events queue ONLY)
  
user-service task:
  Task Role: "user-task-role"
    → DynamoDB:GetItem, PutItem (users table ONLY)
    → S3:GetObject (profile-pics bucket ONLY)
    → Cognito:AdminGetUser

inventory-service task:
  Task Role: "inventory-task-role"  
    → DynamoDB:GetItem, PutItem, UpdateItem (inventory table ONLY)
    → S3:GetObject, PutObject (product-images bucket ONLY)

Shared Execution Role for ALL tasks:
  "ecsTaskExecutionRole"
    → ECR: Pull images
    → CloudWatch: Write logs
    → SecretsManager: Fetch DB passwords

RESULT:
- payment-service CANNOT access user table (no permission!)
- user-service CANNOT decrypt payment keys (no permission!)
- If any container is compromised → blast radius is LIMITED!
```

---

## ⚙️ Hands-On: Create and Attach Roles

```bash
# Create Task Role
aws iam create-role \
  --role-name myapp-task-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach policies to Task Role
aws iam attach-role-policy \
  --role-name myapp-task-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Custom inline policy for specific dynamo table
aws iam put-role-policy \
  --role-name myapp-task-role \
  --policy-name dynamo-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123:table/myapp-table"
    }]
  }'

# Create Execution Role with managed policy
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-trust-policy.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

---

## 🚨 Gotchas & Edge Cases

### 1. "Access Denied" — Which Role is Wrong?
```
Error: Unable to pull image from ECR → Execution Role issue!
Error: S3 access denied in app code → Task Role issue!
Error: ECS agent can't register → Instance Role issue!

Debug:
  CloudTrail → Look for denied API calls
  Who made the call? → which role to fix
  "arn:aws:sts::123:assumed-role/ecsTaskExecutionRole/..." → Execution Role
  "arn:aws:sts::123:assumed-role/myapp-task-role/..."     → Task Role
```

### 2. Task Role Works Without ECS Exec Permissions
```
ECS Exec (execute-command) requires ADDITIONAL permissions in task role:
{
  "Effect": "Allow",
  "Action": [
    "ssmmessages:CreateControlChannel",
    "ssmmessages:CreateDataChannel",
    "ssmmessages:OpenControlChannel",
    "ssmmessages:OpenDataChannel"
  ],
  "Resource": "*"
}
# Missing these → "The execute command failed because execute command was not enabled..."
```

### 3. Execution Role Can't Be Assumed After Task Starts
```
Execution Role = ONLY for task startup
  - Image pull
  - Secret fetch
  - Log stream creation

Once container is running:
  - App code uses TASK ROLE
  - Execution Role no longer active for the app
  - BUT Container Agent still uses it for ongoing log writing
```

---

## 🎤 Interview Angle

**Q: "ECS mein Task Role aur Execution Role ka kya farq hai?"**

> Task Role: Your APPLICATION CODE ke liye. S3, DynamoDB, SQS access → Task Role pe.
> AWS SDKs automatically use Task Role via 169.254.170.2 metadata endpoint.
> 
> Execution Role: ECS AGENT ke liye. ECR se image pull karna, Secrets Manager se secrets fetch karna, CloudWatch mein logs likhna → Execution Role pe.
> App code ke start hone se pehle agent is role ko use karta hai.
>
> Simple rule: "Meri app code kya kare?" → Task Role. "ECS ko task start karne ke liye kya chahiye?" → Execution Role.

**Q: "Ek ECS task compromise ho jaye toh blast radius kaise limit karein?"**

> Least-privilege Task Roles: Har service ka alag Task Role — sirf wahi permissions jo use chahiye.
> Na EC2 Instance Role inherit hoga (ECS automatically blocks IMDS from containers if configured).
> Secrets Rotation: Secrets Manager auto-rotate karta hai — compromise hone par rotate.
> Network Isolation: Security Group se task ko sirf specific destinations pe allow karo.
> VPC Flow Logs: Suspicious traffic detect karo immediately.

---

*Next: [07_ECS_Networking_Modes.md →](./07_ECS_Networking_Modes.md)*
