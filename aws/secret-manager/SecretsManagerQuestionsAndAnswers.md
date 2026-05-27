# AWS Secrets Manager Interview Questions and Answers

---

### 1. What is AWS Secrets Manager and how does it differ from Parameter Store?

**Answer:**

| Feature | Secrets Manager | SSM Parameter Store |
|---------|----------------|-------------------|
| **Purpose** | Secrets (passwords, API keys, tokens) | Config + secrets |
| **Auto-rotation** | Built-in (Lambda-based) | No native rotation |
| **Cost** | $0.40/secret/month + $0.05/10K API calls | Free (standard), $0.05/advanced |
| **Cross-account** | Resource-based policy | Not natively |
| **Max size** | 64 KB | 8 KB (advanced) |
| **Encryption** | Always encrypted (KMS) | Optional (SecureString) |

**Use Secrets Manager** for: Database credentials, API keys — anything needing rotation.
**Use Parameter Store** for: Feature flags, config values, non-sensitive parameters.

---

### 2. How does secret rotation work?

**Answer:**

```bash
# Enable auto-rotation for RDS credentials
aws secretsmanager rotate-secret \
  --secret-id prod/mydb/credentials \
  --rotation-lambda-arn arn:aws:lambda:...:SecretsManagerRotation \
  --rotation-rules AutomaticallyAfterDays=30
```

**Rotation steps (Lambda):**
1. **createSecret** — Generate new password, store as `AWSPENDING`
2. **setSecret** — Update the password in the database
3. **testSecret** — Verify new credentials work
4. **finishSecret** — Move `AWSPENDING` → `AWSCURRENT`

**AWS provides built-in rotation Lambdas** for RDS (MySQL, PostgreSQL, Oracle), Redshift, and DocumentDB.

---

### 3. How do you access secrets from EKS pods?

**Answer:**

```yaml
# Option 1: External Secrets Operator (recommended)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-creds
  data:
    - secretKey: username
      remoteRef:
        key: prod/mydb/credentials
        property: username
    - secretKey: password
      remoteRef:
        key: prod/mydb/credentials
        property: password

# Option 2: AWS Secrets Store CSI Driver (mount as volume)
# Option 3: Application SDK (boto3, AWS SDK) — fetch at runtime
```

---

### 4. Scenario: An application deployed to EKS cannot access a secret. How do you troubleshoot?

**Answer:**

| Check | Fix |
|-------|-----|
| **IAM permissions** | Pod's IRSA/Pod Identity role needs `secretsmanager:GetSecretValue` |
| **KMS permissions** | If secret uses CMK, role needs `kms:Decrypt` |
| **Secret exists?** | `aws secretsmanager describe-secret --secret-id prod/mydb/creds` |
| **Resource policy** | Secret's resource policy may restrict access |
| **VPC endpoint** | Private subnet needs Secrets Manager VPC endpoint |
| **External Secrets synced?** | `kubectl describe externalsecret db-creds` — check sync status |

---

### 5. What are best practices for managing secrets in AWS?

**Answer:**

- **Never hardcode** secrets in code, environment variables in task definitions, or Git
- **Enable rotation** — 30-90 day rotation for all credentials
- **Use resource policies** — Restrict which IAM principals can access each secret
- **KMS CMK** — Use customer-managed keys for audit trail and cross-account access
- **Audit** — CloudTrail logs all Secrets Manager API calls
- **Version stages** — `AWSCURRENT` and `AWSPREVIOUS` allow graceful rotation
- **Tag secrets** — `environment`, `team`, `application` for organization and cost tracking
- **Replicate** — `aws secretsmanager replicate-secret-to-regions` for multi-region DR

---

### 6. Scenario: During secret rotation, your application experiences brief downtime. How do you prevent this?

**Answer:**

**Root cause:** Application caches old credentials and doesn't pick up the new ones.

**Fixes:**
- **Use `AWSCURRENT` stage** — SDK always fetches current version
- **Implement retry logic** — On auth failure, re-fetch secret and retry
- **Cache with short TTL** — Cache secrets for 5-15 min, not indefinitely
- **Multi-user rotation** — Alternate between two DB users (user1 active while user2 rotates)
- **External Secrets Operator** — Auto-refreshes K8s Secret on interval

```python
# Retry pattern for rotation
import boto3, json, time

def get_secret(secret_name, retries=3):
    client = boto3.client('secretsmanager')
    for attempt in range(retries):
        try:
            response = client.get_secret_value(SecretId=secret_name)
            return json.loads(response['SecretString'])
        except client.exceptions.ResourceNotFoundException:
            time.sleep(2 ** attempt)
    raise Exception(f"Failed to get secret {secret_name}")
```
