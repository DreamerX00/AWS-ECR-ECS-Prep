# Secrets Handling in ECS

---

## The Problem with Plain Environment Variables

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

**Why this is bad:**
- Visible in: ECS Console → Task definition → Container details
- Visible in: CloudTrail logs (DescribeTaskDefinition API call)
- Visible in: Docker inspect output on EC2 host
- No rotation support (rotating a password requires deploying a new task definition revision)
- No audit trail of who accessed the secret
- Secret is embedded in the task definition JSON, which may be stored in version control or CI/CD systems

---

## Solution 1: AWS Secrets Manager

### How it works:
```
1. Store secret in Secrets Manager
2. Reference ARN in task definition
3. ECS Execution Role → fetch secret → inject as env var (or file mount)
4. Secret is never visible in the console! Only accessible to authorized Execution Role.
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
        //                                                                                   ↑ extract specific JSON key!
      },
      {
        "name": "DB_USERNAME",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:myapp/production/database:username::"
      }
    ]
  }]
}
```

The ability to extract individual keys from a JSON secret is very powerful. You can store all database credentials in a single Secrets Manager secret as a JSON object, and then extract individual keys (`host`, `port`, `username`, `password`, `dbname`) into separate environment variables. This keeps your secrets organized and minimizes the number of secrets you need to manage.

### Execution Role Must Have Access:
```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue",
    "kms:Decrypt"    // If secret encrypted with custom KMS key
  ],
  "Resource": [
    "arn:aws:secretsmanager:us-east-1:123:secret:myapp/production/*"
  ]
}
```

---

## Solution 2: SSM Parameter Store

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
  --key-id alias/aws/ssm    # AWS managed key, or your custom KMS key
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

## Secrets Manager vs SSM Parameter Store

| Feature | Secrets Manager | SSM Parameter Store |
|---------|-----------------|---------------------|
| Cost | $0.40/secret/month | Free (Standard), $0.05/param/month (SecureString > 10K params) |
| Automatic Rotation | Built-in | Manual (Lambda needed) |
| JSON support | Yes (extract keys) | No (full string only) |
| Cross-account | Yes | Limited |
| Secret versioning | Advanced | Basic |
| Audit (CloudTrail) | Yes | Yes |
| Max size | 65KB | 8KB (Standard), 8KB (Advanced) |
| **Best for:** | DB passwords, API keys, rotation needed | Config values, feature flags, non-rotating secrets |

The $0.40/secret/month cost of Secrets Manager adds up if you have hundreds of secrets. A common pattern is to use Secrets Manager for secrets that require rotation (database passwords, API keys) and SSM Parameter Store for configuration values and non-sensitive data.

---

## Secret Rotation + ECS

```
IMPORTANT: Secrets Manager rotates the secret value itself.
But ECS tasks already running STILL HAVE THE OLD VALUE in memory!

New tasks: get new secret value (fetched at task start)
Running tasks: still using old value!

Two patterns:

Pattern 1: Accept brief inconsistency
  - Rotation happens at 2 AM → minimal user impact
  - New tasks get new password → work fine
  - Running tasks: DB rejects old password after rotation → they crash → ECS restarts them → pick up new password
  - Recovery: 30-60 seconds

Pattern 2: In-app secret refresh
  - App polls Secrets Manager periodically (e.g., every 5 minutes)
  - Gets new value → reconnects to DB
  - No restart needed!
  - More complex but zero-downtime rotation
```

For Pattern 2, the AWS SDKs and AWS SecretsManager Cache libraries provide in-memory caching with TTL-based refresh. This avoids calling Secrets Manager on every request while still picking up rotation within a configurable time window.

---

## Best Practices

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
  --add-replica-regions Region=ap-south-1   # Also replicate for DR!
```

---

## Gotchas & Edge Cases

### 1. Secrets Injected at Task START Only
```
ECS fetches secrets when a task starts → injects as environment variables
If you rotate a secret AFTER the task starts → the task still uses the old value
Only NEW tasks get the new secret!

For instant rotation impact → Force new deployment on the ECS service:
aws ecs update-service --cluster prod --service myapp --force-new-deployment
All tasks are replaced → all get the new secret value!
```

### 2. Secret Value Size Limit
```
Environment variable size: ~32KB per variable
But: Multiple secrets = multiple environment variables
Total environment variable space: Varies by OS (~2MB on Linux)
Secrets Manager: 65KB per secret value

For very large configurations: Use S3 (reference S3 object ARN in secret value, fetch in app code)
```

### 3. Secrets in Logs — A Common Accident
```python
# BAD: Secret value in logs!
db_password = os.environ['DB_PASSWORD']
print(f"Connecting to DB with password: {db_password}")  # LOGGED!

# GOOD: Never log secret values
db_password = os.environ['DB_PASSWORD']
logger.info(f"Connecting to DB as: {os.environ['DB_USER']}")  # Only log non-secret data
```

This is a critical security issue. If secrets appear in CloudWatch Logs, they can be read by anyone with CloudWatch Logs access — which is a much larger group than those with Secrets Manager access. Code review and static analysis tools should catch patterns like this before they reach production.

### 4. Cross-Account Secret Access

If your ECS tasks in Account A need to access secrets in Account B, you need:
1. A resource-based policy on the secret in Account B allowing Account A's Execution Role
2. A KMS key policy allowing Account A to decrypt (if using a custom KMS key)

Cross-account secret access is common in multi-account AWS Organizations setups where secrets are centralized in a shared services account.

### 5. VPC Endpoint for Secrets Manager

In private subnets without a NAT Gateway, tasks cannot reach Secrets Manager over the internet. Create a VPC endpoint for Secrets Manager:

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-a subnet-b subnet-c \
  --security-group-ids sg-endpoint
```

---

## Interview Angle

**Q: "How do you securely inject secrets into ECS tasks? Why are plain environment variables bad practice?"**

> Plain environment variables are problematic because the values are stored in the task definition JSON, which is visible in the ECS Console, readable via DescribeTaskDefinition API calls (which appear in CloudTrail), and accessible via Docker inspect on the EC2 host. There is no rotation support and no audit trail.
>
> The secure approach uses AWS Secrets Manager or SSM Parameter Store. In the task definition, you reference the secret by ARN using the `secrets` field rather than `environment`. ECS uses the Execution Role to call GetSecretValue at task startup and injects the value as an environment variable. The actual secret value never appears in the task definition.
>
> For rotation: after rotating a secret, force a new deployment on the ECS service (`--force-new-deployment`). All tasks are replaced and the new tasks pick up the rotated secret value at startup.

---

*Next: [08_Advanced_Architecture_Patterns.md →](./08_Advanced_Architecture_Patterns.md)*
