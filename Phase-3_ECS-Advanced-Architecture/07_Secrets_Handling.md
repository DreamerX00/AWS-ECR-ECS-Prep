# 🔑 Secrets Handling in ECS

---

## 📖 The Problem with Plain Environment Variables

```dockerfile
# BAD — DO NOT DO THIS:
ENV DB_PASSWORD=mysupersecretpassword

# BAD in task definition:
{
  "environment": [
    {"name": "DB_PASSWORD", "value": "mysupersecretpassword"}
  ]
}
```

**Why BAD:**
- Visible in: ECS Console → Task definition → Container details
- Visible in: CloudTrail logs (DescribeTaskDefinition API call)
- Visible in: Docker inspect output on EC2 host
- No rotation support (rotate password = deploy new task def revision!)
- No audit trail of who accessed the secret

---

## ✅ Solution 1: AWS Secrets Manager

### How it works:
```
1. Store secret in Secrets Manager
2. Reference ARN in task definition
3. ECS Execution Role → fetch secret → inject as env var (or file mount)
4. Secret never visible in console! Only accessible to authorized Execution Role.
```

### Setup:
```bash
# Store secret
aws secretsmanager create-secret \
  --name myapp/production/db-password \
  --secret-string "VeryStr0ngP@ssword!" \
  --description "Production DB password for myapp"

# Store complex secret (JSON):
aws secretsmanager create-secret \
  --name myapp/production/database \
  --secret-string '{
    "username": "myapp_prod",
    "password": "VeryStr0ngP@ssword!",
    "host": "mydb.cluster-abc.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "myapp_prod"
  }'

# Enable automatic rotation (Lambda rotates every 30 days):
aws secretsmanager rotate-secret \
  --secret-id myapp/production/db-password \
  --rotation-lambda-arn arn:aws:lambda:...:function:SecretsManagerRotation \
  --rotation-rules AutomaticallyAfterDays=30
```

### Reference in Task Definition:
```json
{
  "containerDefinitions": [{
    "name": "app",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:myapp/production/db-password"
      },
      {
        "name": "DB_HOST",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:myapp/production/database:host::"
        //                                                                                     ↑ extract specific JSON key!
      },
      {
        "name": "DB_USERNAME",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:myapp/production/database:username::"
      }
    ]
  }]
}
```

### Execution Role Must Have Access:
```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue",
    "kms:Decrypt"    ← If secret encrypted with custom KMS key
  ],
  "Resource": [
    "arn:aws:secretsmanager:us-east-1:123:secret:myapp/production/*"
  ]
}
```

---

## ✅ Solution 2: SSM Parameter Store

### Setup:
```bash
# Standard parameter (free):
aws ssm put-parameter \
  --name "/myapp/production/db-host" \
  --value "mydb.cluster-abc.us-east-1.rds.amazonaws.com" \
  --type String

# SecureString parameter (encrypted with KMS, costs $0.05/param/month):
aws ssm put-parameter \
  --name "/myapp/production/db-password" \
  --value "VeryStr0ngP@ssword!" \
  --type SecureString \
  --key-id alias/aws/ssm    ← AWS managed key, or your custom KMS key
```

### Reference in Task Definition:
```json
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:ssm:us-east-1:123:parameter/myapp/production/db-password"
    },
    {
      "name": "DB_HOST",
      "valueFrom": "arn:aws:ssm:us-east-1:123:parameter/myapp/production/db-host"
    }
  ]
}
```

---

## 🆚 Secrets Manager vs SSM Parameter Store

| Feature | Secrets Manager | SSM Parameter Store |
|---------|-----------------|---------------------|
| Cost | $0.40/secret/month | Free (Standard), $0.05/param/month (SecureString > 10K params) |
| Automatic Rotation | ✅ Built-in | Manual (Lambda needed) |
| JSON support | ✅ (extract keys) | ❌ (full string only) |
| Cross-account | ✅ | Limited |
| Secret versioning | ✅ Advanced | Basic |
| Audit (CloudTrail) | ✅ | ✅ |
| Max size | 65KB | 8KB (Standard), 8KB (Advanced) |
| **Best for:** | DB passwords, API keys, rotation needed | Config values, feature flags, non-rotating secrets |

---

## 🔄 Secret Rotation + ECS

```
IMPORTANT: Secrets Manager rotates the secret itself.
But ECS tasks already running STILL HAVE OLD VALUE in memory!

New tasks: get new secret value (fetched at task start)
Running tasks: still using old value!

Two patterns:

Pattern 1: Accept brief inconsistency
  - Rotation happens at 2 AM → minimal impact
  - New tasks get new password → work fine
  - Running tasks: DB rejects old password after rotation → they crash → ECS restarts them → pick up new password
  - Recovery: 30-60 seconds

Pattern 2: In-app secret refresh
  - App polls Secrets Manager periodically (e.g., every 5 minutes)
  - Gets new value → reconnects DB
  - No restart needed!
  - More complex but zero-downtime rotation
```

---

## ⚙️ Best Practices

```bash
# 1. Namespace secrets per environment:
/myapp/dev/db-password
/myapp/staging/db-password
/myapp/production/db-password

# 2. Use resource-based policies (Secrets Manager):
aws secretsmanager put-resource-policy \
  --secret-id myapp/production/db-password \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123:role/ecsTaskExecutionRole"
      },
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*"
    }]
  }'

# 3. Audit who accessed secrets:
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue
# See: WHO called GetSecretValue, WHEN, from WHERE

# 4. Enable secret deletion protection (prevent accidental delete):
aws secretsmanager update-secret \
  --secret-id myapp/production/db-password \
  --add-replica-regions Region=ap-south-1   ← Also replicate for DR!
```

---

## 🚨 Gotchas & Edge Cases

### 1. Secrets Injected at Task START Only
```
ECS fetches secrets when task starts → injects as env vars
If you rotate Secret AFTER task starts → task still uses old value
Only NEW tasks get the new secret!

For instant rotation impact → Update ECS Service:
aws ecs update-service --cluster prod --service myapp --force-new-deployment
All tasks replaced → all get new secret!
```

### 2. Secret Value Size Limit
```
Environment variable size: ~32KB per variable
But: Multiple secrets = multiple env vars 
Total env variable space: Varies by OS (~2MB on Linux)
Secrets Manager: 65KB per secret value

For very large configs: Use S3 (reference S3 object ARN in secret value, fetch in app code)
```

### 3. Secrets in Logs — Common Accident
```python
# BAD: Secret in logs!
db_password = os.environ['DB_PASSWORD']
print(f"Connecting to DB with password: {db_password}")  # LOGGED!

# GOOD: Never log secret values
db_password = os.environ['DB_PASSWORD']
logger.info(f"Connecting to DB as: {os.environ['DB_USER']}")  # Only log non-secret
```

---

## 🎤 Interview Angle

**Q: "ECS task mein secrets kaise securely inject karte hain? Plain env vars kyu bad practice hai?"**

> Plain env vars: Task definition mein visible (console, API), Docker inspect se readable on host, no rotation support, no audit trail.
>
> Secure approach: AWS Secrets Manager ya SSM Parameter Store.
> Task definition mein: `{"name": "DB_PASS", "valueFrom": "arn:aws:secretsmanager:..."}`
> Execution Role ko GetSecretValue permission → ECS automatically fetches at task start → injects as env var.
> Secret console mein kabhi plain text nahi dikhta.
> Rotation ke liye: For-new-deployment on ECS service after rotation.

---

*Next: [08_Advanced_Architecture_Patterns.md →](./08_Advanced_Architecture_Patterns.md)*
