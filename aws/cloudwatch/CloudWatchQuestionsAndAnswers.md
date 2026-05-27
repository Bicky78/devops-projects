# AWS CloudWatch Interview Questions and Answers

---

### 1. What are the key components of CloudWatch?

**Answer:**

| Component | Purpose |
|-----------|---------|
| **Metrics** | Numeric time-series data (CPU, memory, custom metrics) |
| **Logs** | Log collection, storage, search (Log Groups → Log Streams) |
| **Alarms** | Trigger actions based on metric thresholds |
| **Dashboards** | Custom visualization of metrics and logs |
| **Log Insights** | SQL-like query engine for log analysis |
| **Events / EventBridge** | React to state changes, schedule tasks |
| **Container Insights** | EKS/ECS metrics and logs |
| **Synthetics** | Canary scripts to monitor endpoints |
| **Contributor Insights** | Find top talkers (top N analysis) |

---

### 2. What is the difference between basic and detailed monitoring?

**Answer:**

| Feature | Basic | Detailed |
|---------|-------|---------|
| **Interval** | 5 minutes | 1 minute |
| **Cost** | Free | Extra charge |
| **Enabled by** | Default | Must enable per instance |

**Custom metrics** can be published at any interval (high-resolution: 1-second).

---

### 3. How do you create custom metrics?

**Answer:**

```bash
# CLI
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "ActiveConnections" \
  --value 42 \
  --dimensions Service=API,Environment=Production

# Python (boto3)
import boto3
cw = boto3.client('cloudwatch')
cw.put_metric_data(
    Namespace='MyApp',
    MetricData=[{
        'MetricName': 'OrderProcessingTime',
        'Value': 235.5,
        'Unit': 'Milliseconds',
        'Dimensions': [
            {'Name': 'Service', 'Value': 'OrderService'},
            {'Name': 'Environment', 'Value': 'Production'}
        ]
    }]
)
```

---

### 4. How does CloudWatch Log Insights work?

**Answer:**

```sql
-- Top error messages in last hour
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as cnt by @message
| sort cnt desc
| limit 10

-- Lambda cold starts
filter @type = "REPORT"
| filter @initDuration > 0
| stats count(*) as cold_starts, avg(@initDuration) as avg_init
  by bin(30m)

-- P95 latency from application logs
filter @message like /duration/
| parse @message "duration=* ms" as duration
| stats pct(duration, 95) as p95, avg(duration) as avg_duration
  by bin(5m)
```

---

### 5. How do you set up CloudWatch Alarms?

**Answer:**

```bash
# Metric alarm — high CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-prod" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:xxx:ops-alerts \
  --ok-actions arn:aws:sns:us-east-1:xxx:ops-alerts \
  --dimensions Name=InstanceId,Value=i-0abc123

# Composite alarm — multiple conditions
aws cloudwatch put-composite-alarm \
  --alarm-name "ServiceDegraded" \
  --alarm-rule 'ALARM("HighCPU-prod") AND ALARM("HighErrorRate-prod")'
```

**Alarm states:** `OK` → `ALARM` → `INSUFFICIENT_DATA`

---

### 6. Scenario: EC2 instances don't report memory and disk metrics. Why?

**Answer:** CloudWatch **does not collect memory or disk metrics by default** — they require the **CloudWatch Agent**.

```bash
# Install CloudWatch Agent
sudo yum install amazon-cloudwatch-agent

# Agent config — collect memory, disk, custom logs
{
  "metrics": {
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"] },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/"]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [{
          "file_path": "/var/log/myapp/*.log",
          "log_group_name": "/myapp/application",
          "log_stream_name": "{instance_id}"
        }]
      }
    }
  }
}

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:config.json -s
```

---

### 7. What is CloudWatch Container Insights?

**Answer:** Container Insights collects **EKS/ECS metrics** (cluster, node, pod, container level).

**Metrics:** CPU, memory, network, disk, pod count, container restarts.

```bash
# Enable for EKS
aws eks update-cluster-config \
  --name my-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Or install CloudWatch Agent as DaemonSet via Helm add-on
```

---

### 8. Scenario: How do you set up centralized logging across multiple accounts using CloudWatch?

**Answer:**

```
Account A → CloudWatch Logs → Cross-account subscription filter → Kinesis → Central Account
Account B → CloudWatch Logs → Cross-account subscription filter → Kinesis → Central Account
Account C → CloudWatch Logs → Cross-account subscription filter → Kinesis → Central Account
                                                                              ↓
                                                                         S3 / OpenSearch
```

**Alternatives:**
- **CloudWatch cross-account observability** — Share metrics/logs/traces across accounts
- **Kinesis + Lambda** — Fanout and transform logs
- **Fluent Bit** — Ship directly from instances to central backend (bypasses CloudWatch)

---

### 9. Scenario: CloudWatch alarm is in INSUFFICIENT_DATA state. Why?

**Answer:**

| Cause | Fix |
|-------|-----|
| **Metric not being published** | Check if instance/service is running and sending metrics |
| **Wrong dimensions** | Verify instance ID, namespace, metric name match exactly |
| **Missing data treatment** | Set `TreatMissingData` to `breaching`, `notBreaching`, `ignore`, or `missing` |
| **Evaluation period** | New alarm needs enough data points (e.g., 2 × 5-min periods = 10 min wait) |
| **Region mismatch** | Alarm and metric must be in same region |

---

### 10. What are CloudWatch Synthetics Canaries?

**Answer:** Canaries are **scheduled scripts** that monitor endpoints and APIs by simulating user interactions.

```python
# Canary script (Node.js or Python)
# Runs on schedule (e.g., every 5 min)
# Checks: HTTP status, response time, page content, screenshots

def handler(event, context):
    import urllib.request
    response = urllib.request.urlopen('https://api.example.com/health')
    if response.status != 200:
        raise Exception(f"Health check failed: {response.status}")
    body = response.read().decode()
    if '"status":"ok"' not in body:
        raise Exception("Unexpected response body")
```

**Use cases:** Uptime monitoring, API contract testing, broken link detection, visual regression.
