# AWS Certificate Manager (ACM) Interview Questions and Answers

---

### 1. What is AWS Certificate Manager?

**Answer:** ACM provisions, manages, and deploys **SSL/TLS certificates** for AWS services. Public certificates are **free**.

**Supported services:** CloudFront, ALB/NLB, API Gateway, Elastic Beanstalk, CloudFormation.

**Not supported directly:** EC2 instances (install certificate manually or use ACM Private CA).

---

### 2. How do you create and validate an ACM certificate?

**Answer:**

```bash
# Request certificate
aws acm request-certificate \
  --domain-name example.com \
  --subject-alternative-names "*.example.com" "api.example.com" \
  --validation-method DNS

# DNS validation — add CNAME record to Route 53
# ACM provides: _abc123.example.com → _def456.acm-validations.aws
# Auto-renews as long as CNAME exists
```

| Validation | How | Auto-renew | Best for |
|-----------|-----|-----------|----------|
| **DNS** | Add CNAME record | Yes (if CNAME stays) | Production (recommended) |
| **Email** | Click approval link | No (manual) | Quick tests |

---

### 3. What is the difference between ACM public and ACM Private CA?

**Answer:**

| Feature | ACM Public | ACM Private CA |
|---------|-----------|---------------|
| **Cost** | Free | $400/month per CA |
| **Trust** | Publicly trusted (browsers) | Internal only |
| **Use case** | Public websites, APIs | Internal services, mTLS, IoT |
| **Renewal** | Auto (DNS validated) | Auto |

---

### 4. Scenario: ACM certificate is not renewing automatically. Why?

**Answer:**

| Cause | Fix |
|-------|-----|
| **DNS CNAME removed** | Re-add the validation CNAME record |
| **Email validation used** | Switch to DNS validation |
| **Domain no longer resolves** | Fix DNS for the domain |
| **Certificate imported (not ACM-issued)** | Imported certs don't auto-renew — renew and re-import manually |
| **Certificate not in use** | ACM only auto-renews certificates attached to AWS resources |

---

### 5. Scenario: You need the same certificate for CloudFront and ALB in different regions. How?

**Answer:**
- **CloudFront** requires certificates in **us-east-1** only
- **ALB** requires certificates in the **same region** as the ALB
- Solution: Request **two certificates** — one in us-east-1 (for CF), one in the ALB's region

```hcl
# us-east-1 — for CloudFront
provider "aws" { alias = "us_east_1"; region = "us-east-1" }
resource "aws_acm_certificate" "cf" {
  provider = aws.us_east_1
  domain_name = "example.com"
  validation_method = "DNS"
}

# eu-west-1 — for ALB
resource "aws_acm_certificate" "alb" {
  domain_name = "example.com"
  validation_method = "DNS"
}
```

---

### 6. How do you implement certificate pinning and mTLS with ACM?

**Answer:**

**mTLS (Mutual TLS)** — Both client and server verify each other's certificates.

- **API Gateway** supports mTLS with custom truststore (S3 bundle of CA certificates)
- **ALB** does not natively support mTLS (terminate at NLB or use custom solution)
- Use **ACM Private CA** to issue client certificates

```bash
# API Gateway mTLS
aws apigatewayv2 update-api --api-id abc123 \
  --mutual-tls-authentication truststoreUri=s3://my-bucket/truststore.pem
```
