# AWS Lambda Interview Questions and Answers

---

### 1. What is AWS Lambda and how does it work?

**Answer:** Lambda is a **serverless compute service** that runs code in response to events without provisioning or managing servers.

**Key properties:**
- **Triggers:** API Gateway, S3, DynamoDB Streams, SQS, EventBridge, CloudWatch, ALB
- **Runtimes:** Python, Node.js, Java, Go, .NET, Ruby, custom (container images)
- **Limits:** 15-min max timeout, 10 GB memory, 10 GB `/tmp` storage, 1000 concurrent (default)
- **Billing:** Per request ($0.20/million) + per ms of compute time

---

### 2. What is a cold start and how do you minimize it?

**Answer:** A **cold start** occurs when Lambda creates a new execution environment (download code, init runtime, run init code). Subsequent invocations reuse the environment (warm start).

| Factor | Impact on cold start |
|--------|---------------------|
| **Runtime** | Java/C# = slow (~1-3s), Python/Node = fast (~100-300ms) |
| **Package size** | Larger = slower (minimize dependencies) |
| **VPC** | Adds ~1-2s (ENI creation) — improved with Hyperplane |
| **Memory** | More memory = more CPU = faster init |

**Mitigation:**
- **Provisioned Concurrency** — Pre-warm N environments (eliminates cold starts)
- **SnapStart** (Java only) — Snapshot init state
- **Keep package small** — Use layers, tree-shaking
- **Increase memory** — Faster CPU proportionally
- **Avoid VPC** unless necessary

---

### 3. What are Lambda Layers?

**Answer:** Layers are **reusable packages** (libraries, dependencies, custom runtimes) shared across functions.

```bash
# Create layer
zip layer.zip -r python/
aws lambda publish-layer-version \
  --layer-name common-libs \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.12

# Attach to function
aws lambda update-function-configuration \
  --function-name my-function \
  --layers arn:aws:lambda:us-east-1:xxx:layer:common-libs:1
```

**Benefits:** Smaller deployment packages, shared dependencies, separate update cycles.

---

### 4. How does Lambda scale and what are concurrency controls?

**Answer:**

- **Default:** Scales to 1,000 concurrent executions per region (soft limit)
- **Burst:** 3,000 initial burst, then +500/minute
- **Reserved Concurrency:** Guarantees N concurrent for a function (caps it too)
- **Provisioned Concurrency:** Pre-initialized environments (no cold starts)

```bash
# Reserve 100 concurrent executions
aws lambda put-function-concurrency --function-name my-function --reserved-concurrent-executions 100
```

---

### 5. Scenario: Lambda function times out when connecting to RDS. How do you fix it?

**Answer:**

| Cause | Fix |
|-------|-----|
| **VPC cold start** | Use Provisioned Concurrency or accept delay |
| **Security Group** | Lambda SG → RDS SG must allow port 3306/5432 |
| **Subnet** | Lambda needs private subnet with NAT (or VPC endpoints) |
| **Connection exhaustion** | Use **RDS Proxy** — pools and reuses connections |
| **Timeout too low** | Increase Lambda timeout (max 15 min) |

```python
# Use RDS Proxy to avoid connection exhaustion
# RDS Proxy pools connections — Lambda connects to proxy, not directly to RDS
import pymysql
conn = pymysql.connect(host='my-rds-proxy.proxy-xxx.us-east-1.rds.amazonaws.com', ...)
```

---

### 6. Scenario: How do you process S3 uploads asynchronously using Lambda?

**Answer:**

```
S3 Upload → S3 Event Notification → Lambda → Process file → Write to DynamoDB
```

```python
import boto3, json

def handler(event, context):
    s3 = boto3.client('s3')
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        obj = s3.get_object(Bucket=bucket, Key=key)
        data = json.loads(obj['Body'].read())
        # Process data...
        
    return {'statusCode': 200}
```

---

### 7. What is the difference between synchronous and asynchronous Lambda invocation?

**Answer:**

| Mode | Behavior | Retries | Use Case |
|------|---------|---------|----------|
| **Synchronous** | Caller waits for response | None (caller handles) | API Gateway, ALB |
| **Asynchronous** | Event queued, caller returns immediately | 2 retries | S3, SNS, EventBridge |
| **Event source mapping** | Lambda polls source | Depends on source | SQS, DynamoDB Streams, Kinesis |

---

### 8. How do you deploy Lambda in a CI/CD pipeline?

**Answer:**

```yaml
# GitHub Actions — deploy Lambda
- name: Deploy Lambda
  run: |
    zip function.zip -r .
    aws lambda update-function-code \
      --function-name my-function \
      --zip-file fileb://function.zip
    
    # Publish version + update alias (for traffic shifting)
    VERSION=$(aws lambda publish-version --function-name my-function --query 'Version' --output text)
    aws lambda update-alias --function-name my-function --name prod \
      --function-version $VERSION \
      --routing-config AdditionalVersionWeights={}
```

**Advanced:** Use **SAM** (Serverless Application Model) or **CDK** for infrastructure-as-code Lambda deployments with canary/linear traffic shifting.

---

### 9. Scenario: Lambda is processing SQS messages but some keep reappearing. Why?

**Answer:**

| Cause | Fix |
|-------|-----|
| **Function errors** | Fix bug — failed messages return to queue after visibility timeout |
| **Timeout** | Increase Lambda timeout (must be < SQS visibility timeout) |
| **Partial batch failure** | Enable `ReportBatchItemFailures` — only retry failed messages |
| **Dead letter queue** | Configure DLQ — after N retries, move to DLQ for investigation |

```python
# Partial batch failure reporting
def handler(event, context):
    failures = []
    for record in event['Records']:
        try:
            process(record)
        except Exception:
            failures.append({"itemIdentifier": record['messageId']})
    return {"batchItemFailures": failures}
```

---

### 10. What are Lambda best practices for production?

**Answer:**

- **Keep functions focused** — One function, one responsibility
- **Use environment variables** for config, **Secrets Manager** for secrets
- **Set appropriate timeout and memory** — Don't use defaults
- **Use Layers** for shared dependencies
- **Enable X-Ray tracing** for debugging
- **Use DLQ/destinations** for async error handling
- **Monitor:** Invocations, errors, duration, throttles, concurrent executions
- **Use ARM (Graviton)** — 20% cheaper, same or better performance
- **Minimize package size** — Faster cold starts
- **Connection reuse** — Initialize SDK clients outside handler
