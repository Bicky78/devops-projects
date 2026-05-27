# AWS S3 Interview Questions and Answers

---

### 1. What is S3 and what are its key features?

**Answer:** Amazon S3 (Simple Storage Service) is an **object storage** service with 99.999999999% (11 9s) durability.

**Key features:** Unlimited storage, versioning, lifecycle policies, encryption, replication, event notifications, static website hosting, and multiple storage classes.

---

### 2. What are S3 storage classes and when do you use each?

**Answer:**

| Class | Availability | Min Duration | Use Case | Cost (per GB/month) |
|-------|-------------|-------------|----------|-------------------|
| **S3 Standard** | 99.99% | None | Frequently accessed data | ~$0.023 |
| **S3 Intelligent-Tiering** | 99.9% | None | Unknown access patterns | ~$0.023 + monitoring fee |
| **S3 Standard-IA** | 99.9% | 30 days | Infrequent but rapid access | ~$0.0125 |
| **S3 One Zone-IA** | 99.5% | 30 days | Reproducible infrequent data | ~$0.01 |
| **S3 Glacier Instant** | 99.9% | 90 days | Archive with ms retrieval | ~$0.004 |
| **S3 Glacier Flexible** | 99.99% | 90 days | Archive (min-hours retrieval) | ~$0.0036 |
| **S3 Glacier Deep Archive** | 99.99% | 180 days | Long-term archive (12-48h) | ~$0.00099 |

---

### 3. How does S3 versioning work?

**Answer:** Versioning keeps **all versions** of an object, including deleted ones.

```bash
# Enable versioning
aws s3api put-bucket-versioning --bucket my-bucket --versioning-configuration Status=Enabled

# List versions
aws s3api list-object-versions --bucket my-bucket --prefix important-file.txt

# Restore deleted object (delete the delete marker)
aws s3api delete-object --bucket my-bucket --key important-file.txt --version-id "delete-marker-id"
```

**Key points:**
- Once enabled, versioning can only be **suspended**, not disabled
- MFA Delete adds extra protection against accidental deletion
- Old versions consume storage — use lifecycle rules to expire them

---

### 4. What is S3 lifecycle management?

**Answer:**

```json
{
  "Rules": [{
    "ID": "LogRetention",
    "Status": "Enabled",
    "Filter": { "Prefix": "logs/" },
    "Transitions": [
      { "Days": 30, "StorageClass": "STANDARD_IA" },
      { "Days": 90, "StorageClass": "GLACIER" }
    ],
    "NoncurrentVersionTransitions": [
      { "NoncurrentDays": 7, "StorageClass": "GLACIER" }
    ],
    "NoncurrentVersionExpiration": { "NoncurrentDays": 365 },
    "Expiration": { "Days": 2555 }
  }]
}
```

---

### 5. How do you secure an S3 bucket?

**Answer:**

| Security Layer | How |
|---------------|-----|
| **Block Public Access** | Enable at account + bucket level (default since 2023) |
| **Bucket Policy** | Resource-based policy for cross-account, VPC endpoints |
| **ACLs** | Legacy — disable with `BucketOwnerEnforced` |
| **Encryption** | SSE-S3 (default), SSE-KMS, SSE-C |
| **VPC Endpoint** | S3 Gateway endpoint — private access without internet |
| **Access Points** | Simplified access for multi-team buckets |
| **Object Lock** | WORM — prevent deletion (compliance) |
| **Logging** | Server access logs or CloudTrail data events |

---

### 6. Scenario: How do you set up S3 cross-region replication for disaster recovery?

**Answer:**

```bash
resource "aws_s3_bucket_replication_configuration" "dr" {
  bucket = aws_s3_bucket.primary.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"
      encryption_configuration {
        replica_kms_key_id = aws_kms_key.dr.arn
      }
    }
    source_selection_criteria {
      sse_kms_encrypted_objects { status = "Enabled" }
    }
  }
}
```

**Requirements:** Versioning enabled on both buckets. IAM role for replication.

---

### 7. Scenario: Your S3 bill is unexpectedly high. How do you investigate and reduce costs?

**Answer:**

```bash
# Check bucket sizes with S3 Storage Lens or CLI
aws s3api list-buckets --query 'Buckets[*].Name' --output text | while read bucket; do
  echo "$bucket: $(aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name BucketSizeBytes --dimensions Name=BucketName,Value=$bucket Name=StorageType,Value=StandardStorage --start-time $(date -d '-1 day' +%Y-%m-%dT%H:%M:%S) --end-time $(date +%Y-%m-%dT%H:%M:%S) --period 86400 --statistics Average --query 'Datapoints[0].Average' --output text) bytes"
done
```

**Cost reduction:**
- **Lifecycle policies** — Move infrequent data to IA/Glacier
- **Intelligent-Tiering** — Auto-moves between tiers
- **Delete old versions** — Lifecycle rule for noncurrent versions
- **Abort incomplete multipart uploads** — `AbortIncompleteMultipartUpload: Days: 7`
- **S3 Storage Lens** — Identify anomalies and trends
- **Requester Pays** — If others access your data

---

### 8. What are S3 pre-signed URLs?

**Answer:** Pre-signed URLs grant **time-limited access** to private S3 objects without making them public.

```python
import boto3

s3 = boto3.client('s3')

# Generate pre-signed URL for download (default: 1 hour)
url = s3.generate_presigned_url('get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'reports/q1.pdf'},
    ExpiresIn=3600)

# Generate pre-signed URL for upload
url = s3.generate_presigned_url('put_object',
    Params={'Bucket': 'my-bucket', 'Key': 'uploads/user-photo.jpg'},
    ExpiresIn=300)
```

---

### 9. What is the difference between S3 Event Notifications and S3 Event Bridge?

**Answer:**

| Feature | S3 Event Notifications | EventBridge |
|---------|----------------------|-------------|
| **Destinations** | Lambda, SQS, SNS only | 20+ targets (Step Functions, ECS, etc.) |
| **Filtering** | Prefix/suffix only | Advanced rules (metadata, size, etc.) |
| **Fan-out** | One destination per rule | Multiple targets per event |
| **Recommended** | Simple triggers | Complex event-driven architectures |

---

### 10. Scenario: An application is getting S3 throttling errors (503 Slow Down). How do you fix it?

**Answer:** S3 supports **5,500 GET/s** and **3,500 PUT/s** per prefix.

**Fixes:**
- **Distribute across prefixes:** `bucket/a1/`, `bucket/b2/` instead of `bucket/data/`
- **Use CloudFront** for read-heavy workloads
- **Implement retry with exponential backoff** in application
- **S3 Transfer Acceleration** for large uploads from distant locations
- **Multi-part upload** for large files
