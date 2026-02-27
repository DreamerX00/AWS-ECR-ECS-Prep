# ECS Security Model — IAM Roles & Networking

---

## The 3 IAM Roles of ECS

**This is the most frequently tested topic in ECS interviews.** Most candidates confuse these 3 roles.

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

## 1. TASK ROLE — Your Application's Identity

```
Who uses it?     Your application code running inside the container
What for?        AWS SDK calls your app makes at runtime
When active?     While the task is RUNNING
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

## 2. TASK EXECUTION ROLE — ECS Agent's Identity

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

> AWS provides the managed policy `AmazonECSTaskExecutionRolePolicy` which covers ECR pull and CloudWatch log writing. Extend it with an inline policy to add Secrets Manager or SSM access as needed.

---

## 3. INSTANCE ROLE — EC2 Host's Identity (EC2 mode only)

```
Who uses it?     EC2 instance hosting ECS tasks
What for?
  - ECS Container Agent ↔ ECS Control Plane communication
  - CloudWatch Logs agent
  - SSM Session Manager (if enabled)
When active?     Always (as long as the EC2 instance is running)
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

> **Never add application-level permissions to the Instance Role.**
> All tasks running on that EC2 instance would be able to access those permissions via the EC2 instance metadata endpoint — unless ECS is configured to block IMDS access from containers (which it does by default in modern setups).
> Always use the Task Role for application permissions.

---

## Role Summary — Side by Side

```
Scenario: ECS Fargate task runs Node.js app that reads from S3 and DynamoDB
+---------------------+------------------+---------------------------+
| Role                | Used By          | Permissions                |
+---------------------+------------------+---------------------------+
| Task Role           | Node.js app code | s3:GetObject, dynamo:Query |
| Execution Role      | ECS Agent        | ecr:*, logs:*, secrets:*   |
| Instance Role       | EC2 host (not applicable for Fargate)           |
+---------------------+------------------+---------------------------+

In Fargate mode, there is no Instance Role — there is no EC2 instance to have a role.
```

---

## Security Best Practices

### 1. Never Use Instance Credentials in Containers
```bash
# BAD: Container inherits EC2 instance profile credentials (overly broad access)
# Every container on that host gets the full instance profile permissions

# GOOD: Give each task only the permissions it needs
{
  "taskRoleArn": "arn:aws:iam::123:role/payment-service-role",
  # Only: sqs:SendMessage, dynamodb:PutItem (minimum necessary permissions!)
}
```

### 2. Prevent Credential Sharing Between Tasks
```bash
# ECS_ENABLE_TASK_IAM_ROLE=true in ecs.config (enabled by default in modern AMIs)
# This setting causes ECS to create iptables rules that block containers from
# reaching the EC2 instance metadata endpoint (169.254.169.254) directly.
# Each task instead receives its credentials from the task-level metadata endpoint
# at 169.254.170.2, scoped to only the task's own role.

# This ensures that even if two tasks are co-located on the same EC2 instance,
# neither can access the other's credentials or the instance's own credentials.
```

### 3. Use ECS Exec Instead of SSH
```bash
# Do not open port 22! Use ECS Exec (SSM Session Manager-based)
# Requirements:
# 1. Task role must have ssmmessages:* permissions
# 2. Service must have enableExecuteCommand=true

aws ecs update-service \
  --cluster production \
  --service myapp \
  --enable-execute-command

# Then exec into a container:
aws ecs execute-command \
  --cluster production \
  --task <task-arn> \
  --container app \
  --interactive \
  --command "/bin/bash"

# Every exec session is logged in CloudTrail + S3 (if session logging is configured)
# This provides a complete audit trail of all interactive access to containers
```

### 4. Secrets in Task Definitions — Never Use Environment Variables for Sensitive Data
```bash
# BAD: Secret stored as plaintext environment variable
{
  "environment": [
    {"name": "DB_PASSWORD", "value": "plaintext-password-here"}
  ]
}
# Visible in ECS console, DescribeTaskDefinition API, CloudTrail logs!

# GOOD: Secret fetched at runtime from Secrets Manager
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:myapp/db-password"
    }
  ]
}
# Value never stored in task definition; fetched at task launch using execution role
# Supports automatic rotation without redeploying the task definition
```

---

## Real-World Scenario: Microservices Security

```
E-commerce platform security design:

payment-service task:
  Task Role: "payment-task-role"
    → KMS:Decrypt (payment encryption keys)
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
- payment-service CANNOT access the users table (no permission in its task role)
- user-service CANNOT decrypt payment keys (no permission in its task role)
- If any container is compromised → blast radius is LIMITED to that service's permissions
```

---

## Hands-On: Create and Attach Roles

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

# Custom inline policy for a specific DynamoDB table
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

## Gotchas & Edge Cases

### 1. "Access Denied" — Which Role Is Wrong?
```
Error: Unable to pull image from ECR → Execution Role issue!
Error: S3 access denied in app code → Task Role issue!
Error: ECS agent cannot register with cluster → Instance Role issue!

Debug using CloudTrail:
  Look for denied API calls in CloudTrail → Event history
  Identify the IAM principal that made the call:
  "arn:aws:sts::123:assumed-role/ecsTaskExecutionRole/..." → Execution Role
  "arn:aws:sts::123:assumed-role/myapp-task-role/..."     → Task Role
  "arn:aws:sts::123:assumed-role/ecsInstanceRole/..."     → Instance Role
```

### 2. ECS Exec Requires Additional Task Role Permissions
```
ECS Exec (execute-command) requires the following permissions in the TASK ROLE
(not the execution role!):
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

Missing these permissions causes the error:
"The execute command failed because execute command was not enabled..."
```

### 3. Execution Role Is Not Available to the Application
```
Execution Role = ONLY for task startup
  - Image pull from ECR
  - Secret fetch from Secrets Manager / SSM
  - Log stream creation in CloudWatch

Once the container is running:
  - Application code uses the TASK ROLE
  - The Execution Role is no longer active for the app
  - HOWEVER, the Container Agent continues to use it for ongoing log event writing

A common mistake is putting application-level permissions on the Execution Role.
This works initially but is a security anti-pattern: you are giving the launch-time
agent broader permissions than it needs, and the application permissions are not
scoped to the specific task family.
```

### 4. Task Role Credentials Auto-Rotate
```
Task Role credentials are short-lived STS tokens (valid for 6 hours).
They are automatically refreshed by the ECS infrastructure before expiry.

This means:
  - No credential rotation logic is needed in your application
  - AWS SDKs handle this transparently via the credential provider chain
  - If you cache boto3 clients or AWS SDK clients at startup, they will
    automatically use refreshed credentials on subsequent API calls
    (as long as you do not cache the raw credentials themselves)
```

---

## Interview Questions

**Q: "What is the difference between a Task Role and an Execution Role in ECS?"**

> The Task Role is the identity of your application code. Any AWS SDK calls your container makes at runtime — reading from S3, writing to DynamoDB, publishing to SQS — use the Task Role. AWS SDKs automatically retrieve Task Role credentials from the `169.254.170.2` metadata endpoint inside the container.
>
> The Execution Role is the identity of the ECS Agent itself, used during task launch. It is what allows ECS to pull your image from ECR, fetch secrets from Secrets Manager, and write log events to CloudWatch — before your application code even starts running.
>
> A simple rule: "What does my application code need to do in AWS?" goes into the Task Role. "What does ECS need to do to start the task?" goes into the Execution Role.

**Q: "How would you limit the blast radius if an ECS task is compromised?"**

> Use least-privilege Task Roles: Give each microservice its own Task Role with only the permissions it needs for its specific tables, buckets, and queues — no shared broad roles.
> Block EC2 instance metadata from containers: ECS creates iptables rules to prevent containers from reaching the EC2 IMDS endpoint, so a compromised container cannot access the instance's credentials.
> Use Secrets Manager for credentials and enable automatic rotation: If credentials are compromised, rotate them immediately without redeploying.
> Apply task-level Security Groups (awsvpc mode): Restrict network access so that even a compromised container can only reach its designated downstream services.
> Enable VPC Flow Logs on task ENIs to detect suspicious outbound connections early.

---

*Next: [07_ECS_Networking_Modes.md →](./07_ECS_Networking_Modes.md)*
