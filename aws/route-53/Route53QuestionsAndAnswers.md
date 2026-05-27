# AWS Route 53 Interview Questions and Answers

---

### 1. What is Route 53 and what are its key functions?

**Answer:** Route 53 is AWS's **DNS service** providing domain registration, DNS routing, and health checking.

**Functions:** Domain registration, DNS hosting (authoritative), traffic routing (latency, geo, failover, weighted), and health checks.

---

### 2. What are the Route 53 routing policies?

**Answer:**

| Policy | How it works | Use Case |
|--------|-------------|----------|
| **Simple** | Single resource, no health checks | Single server |
| **Weighted** | Distribute by percentage (e.g., 70/30) | A/B testing, canary deployments |
| **Latency** | Route to lowest-latency region | Multi-region active-active |
| **Failover** | Primary/secondary with health checks | Disaster recovery |
| **Geolocation** | Route based on user's country/continent | Regulatory compliance, localized content |
| **Geoproximity** | Route based on physical distance (with bias) | Fine-tuned geographic routing |
| **Multi-value** | Up to 8 healthy records returned | Simple load balancing |
| **IP-based** | Route based on client IP ranges (CIDR) | ISP-specific routing |

---

### 3. What is the difference between Alias and CNAME records?

**Answer:**

| Feature | CNAME | Alias |
|---------|-------|-------|
| **Zone apex** | Cannot point root domain (example.com) | Can point root domain |
| **Target** | Any DNS name | AWS resources only (ALB, CF, S3, etc.) |
| **DNS query cost** | Charged | Free for AWS resources |
| **Health checks** | Not integrated | Integrated |

**Always use Alias** for AWS resources (CloudFront, ALB, S3, API Gateway).

---

### 4. How do Route 53 health checks work?

**Answer:**

```bash
# Health check types:
# 1. Endpoint — HTTP/HTTPS/TCP to an IP or domain
# 2. Calculated — Based on other health checks (AND/OR logic)
# 3. CloudWatch alarm — Based on CloudWatch metric

aws route53 create-health-check --caller-reference $(date +%s) \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "api.example.com",
    "Port": 443,
    "ResourcePath": "/health",
    "RequestInterval": 10,
    "FailureThreshold": 3
  }'
```

---

### 5. Scenario: How do you implement DNS failover between two regions?

**Answer:**

```
Route 53 Failover Record: api.example.com
├── Primary: ALB in us-east-1 (health check attached)
└── Secondary: ALB in eu-west-1 (failover if primary unhealthy)
```

```hcl
resource "aws_route53_record" "primary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "A"
  set_identifier = "primary"
  failover_routing_policy { type = "PRIMARY" }
  alias {
    name    = aws_lb.primary.dns_name
    zone_id = aws_lb.primary.zone_id
  }
  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "secondary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "A"
  set_identifier = "secondary"
  failover_routing_policy { type = "SECONDARY" }
  alias {
    name    = aws_lb.secondary.dns_name
    zone_id = aws_lb.secondary.zone_id
  }
}
```

---

### 6. Scenario: How do you do blue-green deployment with Route 53?

**Answer:** Use **weighted routing** to shift traffic gradually.

```bash
# Start: 100% → blue, 0% → green
# Then:  90%  → blue, 10% → green (canary)
# Then:  50%  → blue, 50% → green
# Then:  0%   → blue, 100% → green (cutover)

aws route53 change-resource-record-sets --hosted-zone-id Z1234 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "green",
        "Weight": 100,
        "AliasTarget": { "HostedZoneId": "Z5678", "DNSName": "green-alb.us-east-1.elb.amazonaws.com" }
      }
    }]
  }'
```

---

### 7. What is a Route 53 private hosted zone?

**Answer:** A private hosted zone resolves DNS names **within your VPC(s)** — not accessible from the internet.

**Use cases:** Internal service discovery (`db.internal.example.com`), split-horizon DNS (different answers for internal vs external).

---

### 8. Scenario: DNS changes are not propagating. How do you troubleshoot?

**Answer:**

| Check | How |
|-------|-----|
| **TTL** | High TTL means old records cached by resolvers — wait or lower TTL before next change |
| **NS records** | Verify domain's NS records point to Route 53's name servers |
| **Record correct?** | `aws route53 list-resource-record-sets --hosted-zone-id Z1234` |
| **dig/nslookup** | `dig api.example.com @ns-xxx.awsdns-xx.com` — query Route 53 directly |
| **Health check** | If using failover/weighted, record may be hidden because health check is failing |

---

### 9. What is Route 53 Resolver and when do you use it?

**Answer:** Route 53 Resolver handles DNS resolution between **VPC and on-premises** networks.

| Component | Direction | Use Case |
|-----------|----------|----------|
| **Inbound Endpoint** | On-prem → VPC | On-prem resolves AWS private DNS |
| **Outbound Endpoint** | VPC → On-prem | AWS resolves on-prem DNS names |
| **Resolver Rules** | Conditional forwarding | Route specific domains to specific DNS servers |

---

### 10. Scenario: How do you migrate DNS from GoDaddy/external provider to Route 53 without downtime?

**Answer:**

1. **Create hosted zone** in Route 53 for your domain
2. **Replicate all records** from old provider to Route 53
3. **Lower TTL** on old provider to 60-300 seconds (do this 48h before migration)
4. **Test** — Query Route 53 name servers directly: `dig @ns-xxx.awsdns-xx.com example.com`
5. **Update NS records** at domain registrar to Route 53's name servers
6. **Monitor** — Some resolvers will cache old NS for up to the old TTL
7. **Verify** propagation: `dig example.com NS +trace`
