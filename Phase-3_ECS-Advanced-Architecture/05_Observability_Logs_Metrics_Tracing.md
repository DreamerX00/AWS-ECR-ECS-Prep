# 📊 Observability — Logs, Metrics, Tracing

---

## 📖 The Three Pillars of Observability

```
LOGS     → What happened? (text events)
METRICS  → How much? How many? (numerical over time)
TRACES   → How long? Where? (distributed request journey)
```

---

## 🗃️ LOGS — ECS Logging Strategies

### Option 1: awslogs (Simplest)
```json
// Task Definition container log config:
{
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": {
      "awslogs-group": "/ecs/myapp",
      "awslogs-region": "us-east-1",
      "awslogs-stream-prefix": "ecs",
      "awslogs-create-group": "true"    ← Auto-create log group
    }
  }
}

// Log stream format: ecs/<container-name>/<task-id>
// e.g.: ecs/app/abc123def456789abc123
```

```bash
# View logs in real-time
aws logs tail /ecs/myapp --follow

# Get logs for specific task
aws logs get-log-events \
  --log-group-name /ecs/myapp \
  --log-stream-name ecs/app/<task-id>
```

### Option 2: FireLens — Flexible Log Routing (Recommended)
```
FireLens: ECS-native log router using Fluent Bit or Fluentd
Allows routing logs to:
  - CloudWatch Logs
  - Amazon OpenSearch (Elasticsearch)
  - Amazon S3 (archive)
  - Splunk
  - Datadog
  - Any HTTP endpoint
```

```json
// Sidecar container for log routing:
{
  "containerDefinitions": [
    {
      "name": "log_router",
      "image": "906394416424.dkr.ecr.us-east-1.amazonaws.com/aws-for-fluent-bit:stable",
      "essential": false,
      "firelensConfiguration": {
        "type": "fluentbit",
        "options": {
          "enable-ecs-log-metadata": "true"  ← Adds ECS metadata to every log line!
        }
      },
      "logConfiguration": {
        "logDriver": "awslogs",   // Log router's own logs
        "options": {
          "awslogs-group": "/ecs/firelens",
          "awslogs-region": "us-east-1"
        }
      }
    },
    {
      "name": "app",
      "image": "myapp:v1",
      "logConfiguration": {
        "logDriver": "awsfirelens",   ← Use FireLens!
        "options": {
          "Name": "cloudwatch",
          "region": "us-east-1",
          "log_group_name": "/ecs/myapp",
          "log_stream_prefix": "ecs/"
        }
        // OR send to S3:
        // "Name": "s3",
        // "bucket": "my-logs-bucket",
        // "region": "us-east-1",
        // "total_file_size": "100M",
        // "s3_key_format": "/ecs/myapp/%Y/%m/%d/$TAG[1]"
      }
    }
  ]
}
```

### Option 3: Sidecar Pattern (Custom Logging)
```json
// Custom log agent as sidecar:
{
  "containerDefinitions": [
    {
      "name": "app",
      "image": "myapp:v1",
      // Writes logs to shared volume
    },
    {
      "name": "log-agent",
      "image": "datadog/datadog-agent:latest",
      "essential": false,
      "mountPoints": [{
        "sourceVolume": "app-logs",
        "containerPath": "/var/log/app"   ← Read logs written by app
      }]
    }
  ],
  "volumes": [{
    "name": "app-logs"
  }]
}
```

---

## 📈 METRICS — ECS Monitoring

### CloudWatch Container Insights (Recommended)
```bash
# Enable Container Insights on cluster:
aws ecs update-cluster-settings \
  --cluster production \
  --settings name=containerInsights,value=enabled

# Metrics automatically collected:
# Service-level:
#   - CPUUtilization
#   - MemoryUtilization
#   - NetworkRxBytes / NetworkTxBytes
#   - StorageReadBytes / StorageWriteBytes
#
# Task-level (granular):
#   - CpuReserved vs CpuUtilized
#   - MemoryReserved vs MemoryUtilized
#
# Instance-level (EC2 mode):
#   - Instance CPU/Memory/Network/Storage
```

### Key Metrics to Monitor:
```
CPUUtilization:
  > 80% sustained → scale out
  < 20% sustained → scale in (over-provisioned)
  Spike to 100% → CPU throttling → latency increase!
  
MemoryUtilization:
  > 80% → risk of OOM kill
  OOM kill → container exits with code 137
  Trend upward over time → MEMORY LEAK!
  
RunningTaskCount vs DesiredCount:
  RunningTaskCount < DesiredCount → tasks failing to start/crashing
  
TargetResponseTime (ALB):
  p50, p90, p99 latencies
  Spike in p99 → some requests very slow (tail latency issue)
```

### Custom Metrics:
```python
# Push custom business metrics from your app
import boto3

cloudwatch = boto3.client('cloudwatch')

def record_order_metric(order_value):
    cloudwatch.put_metric_data(
        Namespace='MyApp/Business',
        MetricData=[{
            'MetricName': 'OrderValue',
            'Value': order_value,
            'Unit': 'None',
            'Dimensions': [
                {'Name': 'Service', 'Value': 'order-service'},
                {'Name': 'Environment', 'Value': 'production'}
            ]
        }]
    )
```

---

## 🔍 TRACING — AWS X-Ray Integration

```
Problem: User request slow. Which service is the bottleneck?

user-service → order-service → inventory-service → payment-service
  50ms           200ms             500ms             150ms
                                     ↑ 
                               Bottleneck found!

X-Ray shows the ENTIRE journey of one request.
```

### ECS + X-Ray Sidecar Setup:
```json
{
  "containerDefinitions": [
    {
      "name": "app",
      "image": "myapp:v1",
      "environment": [
        {"name": "AWS_XRAY_DAEMON_ADDRESS", "value": "127.0.0.1:2000"}
      ]
      // App sends traces to X-Ray daemon on localhost
    },
    {
      "name": "xray-daemon",
      "image": "amazon/aws-xray-daemon",
      "essential": false,
      "portMappings": [{"containerPort": 2000, "protocol": "udp"}],
      "cpu": 32,
      "memory": 256
      // Daemon receives traces from app, forwards to X-Ray service
    }
  ]
}
```

### Instrument Your App (Node.js):
```javascript
const AWSXRay = require('aws-xray-sdk');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

// Capture all outgoing HTTP requests:
AWSXRay.captureHTTPsGlobal(require('https'));

// Express middleware:
const express = require('express');
const app = express();
app.use(AWSXRay.express.openSegment('myapp'));

app.get('/api/orders', async (req, res) => {
  // Create subsegment for database call
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment('dynamodb-query');
  
  try {
    const orders = await dynamo.query({...}).promise();
    subsegment.close();
    res.json(orders);
  } catch(err) {
    subsegment.addError(err);
    subsegment.close();
    throw err;
  }
});

app.use(AWSXRay.express.closeSegment());
```

---

## 🌍 Real-World Observability Stack

```
Production observability setup:

LOGS:
  FireLens (Fluent Bit)
    ├── Debug/Info → CloudWatch Logs (cheap, searchable)
    ├── Errors → S3 (archive, long-term)
    └── Security audit → Amazon OpenSearch (complex queries)

METRICS:
  Container Insights → CloudWatch (CPU, Memory, Network)
  Custom metrics → CloudWatch (business KPIs: orders/sec, revenue)
  Dashboards → CloudWatch Dashboard + Grafana

TRACES:
  AWS X-Ray → Service Map (visual topology)
  → Trace individual slow requests
  → Error rate per service segment

ALERTS:
  CPU > 80% → SNS → PagerDuty (on-call engineer)
  Error rate > 1% → SNS → Slack #incidents
  Task crash → EventBridge → Lambda → Slack + JIRA ticket
  
COST:
  CloudWatch Logs: ~$0.50/GB ingested + $0.03/GB stored
  Container Insights: ~$0.0075/GB metrics
  X-Ray: $5/million traces recorded
```

---

## ⚙️ Hands-On: CloudWatch Alarms

```bash
# Alarm: High container CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-HighCPU-myapp" \
  --alarm-description "ECS service CPU > 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --dimensions \
    Name=ServiceName,Value=myapp \
    Name=ClusterName,Value=production \
  --statistic Average \
  --period 300 \        # 5 minutes
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \   # 2 consecutive periods = 10 min sustained
  --alarm-actions arn:aws:sns:...:ops-alerts \
  --ok-actions arn:aws:sns:...:ops-alerts

# Alarm: Task count below desired (tasks crashing)
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-TaskCount-Low" \
  --metric-name RunningTaskCount \
  --namespace ECS/ContainerInsights \
  --dimensions Name=ServiceName,Value=myapp Name=ClusterName,Value=production \
  --statistic Minimum \
  --period 60 \
  --threshold 2 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:...:critical-alerts
```

---

## 🚨 Gotchas & Edge Cases

### 1. awslogs Doesn't Support Structured JSON
```
App sends: {"level":"error","msg":"DB connection failed","userId":123}
awslogs stores as raw string → hard to query!

FireLens with CloudWatch: JSON preserved!
CloudWatch Logs Insights query:
fields @timestamp, level, msg, userId
| filter level = "error"
| sort @timestamp desc
```

### 2. X-Ray Sampling — Not Every Request Traced
```
Default: 1 req/sec + 5% of remaining requests per service
High traffic (10K req/s): ~500 + 500 = 1000 traces/s

For debugging: Temporarily increase sampling
aws xray put-sampling-rule → adjust FixedRate to 1.0 (100%)
Remember to reduce after debugging!
```

### 3. Container Insights Additional Cost
```
Container Insights is NOT free like basic CloudWatch metrics!
Cost: ~$0.0075/GB metrics data + retention costs

For large clusters: Can be significant
Alternative: Prometheus + Grafana (OSS) for metrics
             → No per-GB CloudWatch cost
             → More flexibility
```

---

## 🎤 Interview Angle

**Q: "ECS mein logs kaise collect karte hain? FireLens kya hai?"**

> awslogs: Simplest — directly to CloudWatch Logs. Good for most cases.
> FireLens: ECS-native log routing via Fluent Bit sidecar container. Multiple destinations — CloudWatch, S3, OpenSearch, Splunk, Datadog. Adds ECS metadata to every log line. Recommended for enterprise observability.
> Sidecar log agent: Custom log agent (Datadog Agent, etc.) as non-essential container sharing volume or network with main container.

**Q: "Production mein kya metrics monitor karte hain ECS ke liye?"**

> Key metrics:
> - CPUUtilization/MemoryUtilization → capacity planning, scaling triggers
> - RunningTaskCount vs DesiredCount → detect crashing services  
> - ALB TargetResponseTime (p99) → latency monitoring
> - ALB HTTPCode_Target_5XX_Count → error rate
> - Custom business metrics via app code
> Container Insights automatic karta hai infrastructure metrics, app metrics custom code se push.

---

*Next: [06_Scaling_Strategies.md →](./06_Scaling_Strategies.md)*
