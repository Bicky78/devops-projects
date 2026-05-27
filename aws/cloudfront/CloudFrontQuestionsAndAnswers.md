# AWS CloudFront Interview Questions and Answers

Comprehensive CloudFront questions for DevOps engineers — covering CDN fundamentals, distributions, caching, security, Lambda@Edge, and real-world scenario-based questions.

---

## Table of Contents

- [CloudFront Fundamentals](#cloudfront-fundamentals)
- [Distributions & Origins](#distributions--origins)
- [Caching & Performance](#caching--performance)
- [Security](#security)
- [Lambda@Edge & CloudFront Functions](#lambdaedge--cloudfront-functions)
- [Scenario-Based Questions](#scenario-based-questions)

---

## CloudFront Fundamentals

### 1. What is CloudFront and how does it work?

**Answer:** Amazon CloudFront is a **Content Delivery Network (CDN)** that caches content at 400+ edge locations worldwide, reducing latency by serving content from the location closest to the user.

**How it works:**
```
User (Tokyo) → Edge Location (Tokyo) → [Cache HIT] → Return content (fast)
                                      → [Cache MISS] → Origin (us-east-1 S3/ALB)
                                                        → Cache at edge → Return
```

**Key concepts:**

| Concept | Description |
|---------|-------------|
| **Distribution** | CloudFront deployment configuration |
| **Origin** | Source of content (S3, ALB, custom HTTP server) |
| **Edge Location** | Cache endpoint closest to users |
| **Regional Edge Cache** | Larger cache between edge and origin |
| **Behavior** | Path-based routing rules + caching config |
| **Cache Policy** | What to include in cache key (headers, query strings, cookies) |
| **TTL** | How long content stays cached |

---

### 2. What types of content can CloudFront deliver?

**Answer:**

| Content Type | Example | Origin |
|-------------|---------|--------|
| **Static** | HTML, CSS, JS, images, fonts | S3 |
| **Dynamic** | API responses, personalized content | ALB, API Gateway, EC2 |
| **Streaming** | Video (HLS, DASH) | S3, MediaPackage |
| **WebSocket** | Real-time communication | ALB, EC2 |
| **Software downloads** | Packages, installers | S3 |

---

## Distributions & Origins

### 3. How do you create a CloudFront distribution with S3 and ALB origins?

**Answer:**

```bash
# Terraform — multi-origin distribution
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  default_root_object = "index.html"
  aliases             = ["www.example.com"]
  price_class         = "PriceClass_100"   # US, Canada, Europe only

  # Origin 1: S3 for static assets
  origin {
    domain_name              = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id                = "s3-static"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  # Origin 2: ALB for API
  origin {
    domain_name = aws_lb.api.dns_name
    origin_id   = "alb-api"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
    custom_header {
      name  = "X-Custom-Header"
      value = "secret-value"   # Verify requests come from CloudFront
    }
  }

  # Default behavior — S3 static content
  default_cache_behavior {
    target_origin_id       = "s3-static"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = aws_cloudfront_cache_policy.static.id
    compress               = true
  }

  # /api/* behavior — ALB
  ordered_cache_behavior {
    path_pattern           = "/api/*"
    target_origin_id       = "alb-api"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = aws_cloudfront_cache_policy.api.id
    origin_request_policy_id = aws_cloudfront_origin_request_policy.api.id
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }
}
```

---

### 4. What is Origin Access Control (OAC) and how does it differ from OAI?

**Answer:** Both restrict S3 access so only CloudFront can read from the bucket.

| Feature | OAI (Origin Access Identity) — Legacy | OAC (Origin Access Control) — Recommended |
|---------|--------------------------------------|------------------------------------------|
| **SSE-KMS** | Not supported | Supported |
| **Dynamic requests** | GET/HEAD only | All HTTP methods (PUT, POST, DELETE) |
| **Multiple distributions** | Complex | Simple |
| **AWS recommends** | No (legacy) | Yes |

```json
// S3 bucket policy with OAC
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/E1A2B3C4D5"
      }
    }
  }]
}
```

---

## Caching & Performance

### 5. How does CloudFront caching work and what is a cache key?

**Answer:** The **cache key** determines whether CloudFront serves from cache (HIT) or fetches from origin (MISS).

**Default cache key:** URL path only (e.g., `/images/logo.png`)

**Cache key can include:**
- URL path (always)
- Query strings (specific or all)
- HTTP headers (specific)
- Cookies (specific or all)

**Important:** More items in cache key = more cache misses = less effective caching.

```bash
# Cache policy — static assets (aggressive caching)
resource "aws_cloudfront_cache_policy" "static" {
  name        = "static-assets"
  default_ttl = 86400     # 1 day
  max_ttl     = 31536000  # 1 year
  min_ttl     = 3600      # 1 hour

  parameters_in_cache_key_and_forwarded_to_origin {
    headers_config {
      header_behavior = "none"   # Don't include headers in cache key
    }
    cookies_config {
      cookie_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "none"
    }
    enable_accept_encoding_gzip  = true
    enable_accept_encoding_brotli = true
  }
}

# Cache policy — API (respect query strings, minimal caching)
resource "aws_cloudfront_cache_policy" "api" {
  name        = "api-cache"
  default_ttl = 0         # Don't cache by default
  max_ttl     = 60
  min_ttl     = 0

  parameters_in_cache_key_and_forwarded_to_origin {
    headers_config {
      header_behavior = "whitelist"
      headers { items = ["Authorization"] }
    }
    cookies_config {
      cookie_behavior = "none"
    }
    query_strings_config {
      query_string_behavior = "all"
    }
  }
}
```

---

### 6. What are cache behaviors and how do you use path-based routing?

**Answer:** Cache behaviors define how CloudFront handles requests for different URL patterns.

```
www.example.com
├── /              → S3 (static website, cache 1 day)
├── /api/*         → ALB (dynamic, no cache or short TTL)
├── /static/*      → S3 (aggressive cache, 1 year)
├── /media/*       → S3 (media bucket, cache 7 days)
└── /ws/*          → ALB (WebSocket, no cache)
```

**Evaluation order:** Behaviors are evaluated in order (most specific first). The **default behavior** (`*`) catches everything not matched.

---

### 7. How do you invalidate CloudFront cache?

**Answer:**

```bash
# Invalidate specific paths
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5 \
  --paths "/index.html" "/css/*" "/js/*"

# Invalidate everything (expensive — use sparingly)
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5 \
  --paths "/*"
```

**Cost:** First 1,000 invalidation paths/month free, then $0.005 per path. A wildcard `/*` counts as 1 path.

**Better alternatives to invalidation:**
- **Versioned filenames:** `app.v2.3.js` instead of `app.js` — infinite cache, no invalidation needed
- **Cache busting query strings:** `style.css?v=abc123`
- **Short TTL** for frequently changing content

---

### 8. How do you optimize CloudFront performance?

**Answer:**

| Optimization | How |
|-------------|-----|
| **Compression** | Enable Brotli + Gzip (`compress = true`) |
| **HTTP/2 & HTTP/3** | Enabled by default (QUIC for HTTP/3) |
| **Price Class** | `PriceClass_All` for global, `PriceClass_100` for US/EU only |
| **Origin Shield** | Extra cache layer between edge and origin (reduces origin load) |
| **Regional Edge Cache** | Automatic — larger cache before going to origin |
| **Versioned URLs** | Long TTL + versioned filenames (`app.v2.js`) |
| **Connection reuse** | Persistent connections to origin |
| **Custom error pages** | Cache 404/503 pages to reduce origin hits |

```bash
# Enable Origin Shield (extra cache layer)
origin {
  domain_name = aws_s3_bucket.static.bucket_regional_domain_name
  origin_id   = "s3-static"
  origin_shield {
    enabled              = true
    origin_shield_region = "us-east-1"  # Region closest to origin
  }
}
```

---

## Security

### 9. How do you secure a CloudFront distribution?

**Answer:**

| Security Layer | How |
|---------------|-----|
| **HTTPS** | ACM certificate, `viewer-policy: redirect-to-https` |
| **TLS version** | Minimum TLSv1.2_2021 |
| **OAC** | Restrict S3 access to CloudFront only |
| **WAF** | Attach AWS WAF for SQL injection, XSS, rate limiting |
| **Geo restriction** | Block/allow specific countries |
| **Signed URLs/Cookies** | Time-limited access to private content |
| **Custom headers** | Origin verifies header to ensure request came from CF |
| **Field-level encryption** | Encrypt sensitive form fields at edge |
| **Response headers policy** | Security headers (HSTS, CSP, X-Frame-Options) |

```bash
# Security response headers policy
resource "aws_cloudfront_response_headers_policy" "security" {
  name = "security-headers"

  security_headers_config {
    strict_transport_security {
      access_control_max_age_sec = 31536000
      include_subdomains         = true
      preload                    = true
      override                   = true
    }
    content_security_policy {
      content_security_policy = "default-src 'self'; script-src 'self'"
      override                = true
    }
    content_type_options { override = true }  # X-Content-Type-Options: nosniff
    frame_options {
      frame_option = "DENY"
      override     = true
    }
    xss_protection {
      mode_block = true
      protection = true
      override   = true
    }
  }
}
```

---

### 10. How do CloudFront signed URLs and signed cookies work?

**Answer:** Used to restrict access to **private content** (paid content, time-limited downloads).

| Feature | Signed URLs | Signed Cookies |
|---------|------------|----------------|
| **Scope** | Single file/resource | Multiple files (whole site section) |
| **Use case** | Individual file download | Streaming, membership site |
| **URL change** | URL contains signature | URL unchanged, cookie carries signature |

```python
# Generate signed URL (Python)
from botocore.signers import CloudFrontSigner
import datetime, rsa

def rsa_signer(message):
    with open('private_key.pem', 'rb') as f:
        private_key = rsa.PrivateKey.load_pkcs1(f.read())
    return rsa.sign(message, private_key, 'SHA-1')

cf_signer = CloudFrontSigner('KEYPAIRID', rsa_signer)

signed_url = cf_signer.generate_presigned_url(
    url='https://cdn.example.com/premium/video.mp4',
    date_less_than=datetime.datetime.utcnow() + datetime.timedelta(hours=1)
)
```

---

### 11. How do you integrate AWS WAF with CloudFront?

**Answer:**

```bash
resource "aws_wafv2_web_acl" "cloudfront" {
  name  = "cloudfront-waf"
  scope = "CLOUDFRONT"  # Must be CLOUDFRONT (not REGIONAL)

  default_action { allow {} }

  # Rate limiting
  rule {
    name     = "rate-limit"
    priority = 1
    action { block {} }
    statement {
      rate_based_statement {
        limit              = 2000
        aggregate_key_type = "IP"
      }
    }
    visibility_config { ... }
  }

  # AWS Managed Rules — common attacks
  rule {
    name     = "aws-managed-common"
    priority = 2
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"
      }
    }
    visibility_config { ... }
  }

  # SQL injection protection
  rule {
    name     = "sql-injection"
    priority = 3
    override_action { none {} }
    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesSQLiRuleSet"
      }
    }
    visibility_config { ... }
  }
}

# Attach WAF to CloudFront
resource "aws_cloudfront_distribution" "main" {
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn
  ...
}
```

---

## Lambda@Edge & CloudFront Functions

### 12. What is the difference between Lambda@Edge and CloudFront Functions?

**Answer:**

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| **Runtime** | JavaScript only | Node.js, Python |
| **Execution location** | Edge locations (218+) | Regional edge caches (13) |
| **Triggers** | Viewer Request, Viewer Response | All 4 events |
| **Max execution time** | 1 ms | 5s (viewer) / 30s (origin) |
| **Max memory** | 2 MB | 128-10,240 MB |
| **Network access** | No | Yes |
| **Body access** | No | Yes |
| **Cost** | $0.10 / million | $0.60 / million + duration |
| **Use case** | URL rewrites, header manipulation, simple redirects | Auth, A/B testing, image resize, SSR |

**Event triggers:**
```
User → [Viewer Request] → CloudFront → [Origin Request] → Origin
User ← [Viewer Response] ← CloudFront ← [Origin Response] ← Origin
```

---

### 13. Give practical examples of Lambda@Edge and CloudFront Functions.

**Answer:**

```javascript
// CloudFront Function — URL rewrite (SPA routing)
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    // If URI doesn't have extension, serve index.html (SPA)
    if (!uri.includes('.')) {
        request.uri = '/index.html';
    }
    return request;
}

// CloudFront Function — Add security headers
function handler(event) {
    var response = event.response;
    var headers = response.headers;
    headers['strict-transport-security'] = { value: 'max-age=31536000; includeSubdomains; preload' };
    headers['x-content-type-options'] = { value: 'nosniff' };
    headers['x-frame-options'] = { value: 'DENY' };
    return response;
}

// CloudFront Function — Redirect www to non-www
function handler(event) {
    var request = event.request;
    var host = request.headers.host.value;
    if (host.startsWith('www.')) {
        return {
            statusCode: 301,
            statusDescription: 'Moved Permanently',
            headers: { location: { value: 'https://' + host.substring(4) + request.uri } }
        };
    }
    return request;
}
```

```python
# Lambda@Edge — Authorization at edge (Origin Request)
import json
import jwt  # PyJWT

def handler(event, context):
    request = event['Records'][0]['cf']['request']
    headers = request['headers']
    
    auth = headers.get('authorization', [{}])[0].get('value', '')
    if not auth.startswith('Bearer '):
        return {'status': '401', 'body': 'Unauthorized'}
    
    try:
        token = auth.split(' ')[1]
        decoded = jwt.decode(token, 'secret', algorithms=['HS256'])
        # Add user info as header for origin
        request['headers']['x-user-id'] = [{'value': decoded['sub']}]
        return request
    except jwt.InvalidTokenError:
        return {'status': '403', 'body': 'Forbidden'}
```

---

## Scenario-Based Questions

### 14. Scenario: Your CloudFront distribution is serving stale content after a deployment. How do you fix it?

**Answer:**

**Immediate fix:**
```bash
# Invalidate changed files
aws cloudfront create-invalidation \
  --distribution-id E1A2B3C4D5 \
  --paths "/index.html" "/app.js" "/styles.css"
```

**Long-term fix (prevent this from recurring):**
1. **Versioned filenames** — `app.v2.3.4.js` with max-age 1 year (never invalidate)
2. **Short TTL for index.html** — `max-age=0, s-maxage=60` (CloudFront caches 60s, browser always revalidates)
3. **CI/CD pipeline** — After deploy, auto-invalidate `/index.html`
4. **Cache busting** — Webpack/Vite generates content-hashed filenames automatically

```
# Ideal caching strategy:
index.html        → Cache-Control: no-cache                    (always revalidate)
app.a1b2c3.js     → Cache-Control: public, max-age=31536000   (1 year, content-hashed)
style.d4e5f6.css  → Cache-Control: public, max-age=31536000
logo.png          → Cache-Control: public, max-age=86400       (1 day)
```

---

### 15. Scenario: Users in Asia are experiencing slow page loads. Your origin is in us-east-1. How do you improve performance?

**Answer:**

| Optimization | Impact |
|-------------|--------|
| **Verify CloudFront is serving** | Check `X-Cache: Hit from cloudfront` header |
| **Enable Origin Shield** | Set to `ap-northeast-1` (Tokyo) — extra cache layer |
| **Check cache hit ratio** | Low ratio = content being fetched from origin too often |
| **Reduce cache key** | Remove unnecessary headers/cookies/query strings |
| **Enable compression** | Brotli + Gzip for text content |
| **Use PriceClass_All** | Ensure Asia edge locations are included |
| **Optimize origin** | Pre-warm cache, reduce origin response time |
| **Consider S3 replication** | Replicate static assets to ap-northeast-1 S3 bucket as failover origin |

```bash
# Check cache hit ratio
aws cloudwatch get-metric-data \
  --metric-data-queries '[{
    "Id": "hitratio",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/CloudFront",
        "MetricName": "CacheHitRate",
        "Dimensions": [{"Name":"DistributionId","Value":"E1A2B3C4D5"}]
      },
      "Period": 3600, "Stat": "Average"
    }
  }]' ...
```

---

### 16. Scenario: You need to serve a Single Page Application (SPA) with React/Vue on CloudFront + S3. How?

**Answer:**

```bash
resource "aws_cloudfront_distribution" "spa" {
  origin {
    domain_name              = aws_s3_bucket.spa.bucket_regional_domain_name
    origin_id                = "s3-spa"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  default_root_object = "index.html"

  default_cache_behavior {
    target_origin_id       = "s3-spa"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = data.aws_cloudfront_cache_policy.caching_optimized.id
    compress               = true

    # CloudFront Function for SPA routing
    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.spa_rewrite.arn
    }
  }

  # Custom error response — serve index.html for 404s (SPA routing)
  custom_error_response {
    error_code         = 403
    response_code      = 200
    response_page_path = "/index.html"
  }
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
  }
}
```

**Key:** S3 returns 404 for routes like `/about`, `/dashboard` since those files don't exist. The custom error response serves `index.html` and React Router handles the routing client-side.

---

### 17. Scenario: Your CloudFront origin (ALB) is overwhelmed during traffic spikes. How do you protect it?

**Answer:**

| Protection | How |
|-----------|-----|
| **Increase caching** | Cache API responses where possible (even 5-60 seconds helps) |
| **Origin Shield** | Extra cache layer reduces origin requests by 50%+ |
| **WAF rate limiting** | Block excessive requests per IP |
| **Custom error caching** | Cache 5xx responses briefly (10-30s) to stop retry storms |
| **Origin connection limits** | CloudFront limits concurrent connections to origin |
| **Auto Scaling** | Ensure ALB backend (ASG) can scale |

```bash
# Cache 503 errors for 10 seconds (prevent retry storms)
custom_error_response {
  error_code            = 503
  error_caching_min_ttl = 10
}

# Origin Shield
origin {
  origin_shield {
    enabled              = true
    origin_shield_region = "us-east-1"
  }
}
```

---

### 18. Scenario: You need to restrict CloudFront access so only users from specific countries can access content. How?

**Answer:**

```bash
# Geo restriction in CloudFront
restrictions {
  geo_restriction {
    restriction_type = "whitelist"       # or "blacklist"
    locations        = ["US", "CA", "GB", "DE", "FR"]
  }
}
```

**For more granular control (city, region, specific rules):**
- Use **CloudFront Functions** with `cloudfront-viewer-country` header
- Use **AWS WAF** geo-match conditions with custom rules

```javascript
// CloudFront Function — custom geo logic
function handler(event) {
    var country = event.request.headers['cloudfront-viewer-country']?.value || '';
    var blockedCountries = ['CN', 'RU'];
    
    if (blockedCountries.includes(country)) {
        return {
            statusCode: 403,
            statusDescription: 'Forbidden',
            body: { encoding: 'text', data: 'Access denied from your region' }
        };
    }
    return event.request;
}
```

---

### 19. Scenario: How do you set up CloudFront for a multi-region active-active architecture?

**Answer:**

```
                    ┌──────────────┐
                    │  CloudFront  │
                    │(Edge Caching)│
                    └──────┬───────┘
                           │
                   ┌───────▼───────┐
                   │ Origin Group  │
                   │ (failover)    │
                   └───┬───────┬───┘
                       │       │
               Primary │       │ Failover
                       ▼       ▼
                  ┌────────┐ ┌────────┐
                  │ALB     │ │ALB     │
                  │us-east │ │eu-west │
                  └────────┘ └────────┘
```

```bash
# Origin group with failover
origin_group {
  origin_id = "api-failover"

  failover_criteria {
    status_codes = [500, 502, 503, 504]
  }

  member { origin_id = "alb-primary" }
  member { origin_id = "alb-secondary" }
}
```

**For active-active (latency-based):**
- Use **Route 53 latency-based routing** as the origin domain
- Route 53 routes to nearest ALB
- CloudFront caches based on origin response

---

### 20. Scenario: How do you implement A/B testing using CloudFront?

**Answer:**

```python
# Lambda@Edge — A/B testing (Viewer Request)
import hashlib

def handler(event, context):
    request = event['Records'][0]['cf']['request']
    headers = request['headers']
    
    # Check for existing cookie
    cookies = headers.get('cookie', [{}])[0].get('value', '')
    
    if 'ab-group=' in cookies:
        # Existing user — extract group
        group = 'B' if 'ab-group=B' in cookies else 'A'
    else:
        # New user — assign group based on IP hash
        ip = event['Records'][0]['cf']['request']['clientIp']
        group = 'B' if int(hashlib.md5(ip.encode()).hexdigest(), 16) % 100 < 20 else 'A'
    
    # Route to different origin path
    if group == 'B':
        request['uri'] = request['uri'].replace('/app/', '/app-v2/')
    
    # Add header for origin to know the group
    request['headers']['x-ab-group'] = [{'value': group}]
    
    return request
```

---

### 21. What are CloudFront real-time logs and standard logs?

**Answer:**

| Feature | Standard Logs | Real-Time Logs |
|---------|-------------|----------------|
| **Delivery** | S3 bucket (delayed 10-30 min) | Kinesis Data Stream (seconds) |
| **Cost** | Free (S3 storage cost only) | Kinesis + per-request fee |
| **Sampling** | 100% of requests | Configurable (1-100%) |
| **Use case** | Historical analysis, compliance | Live monitoring, real-time dashboards |
| **Format** | Tab-delimited text | JSON |

```bash
# Enable standard logs
resource "aws_cloudfront_distribution" "main" {
  logging_config {
    bucket          = aws_s3_bucket.cf_logs.bucket_domain_name
    prefix          = "cloudfront/"
    include_cookies = false
  }
}
```

**Useful Athena query on CF logs:**
```sql
-- Top 10 most requested URLs with cache miss
SELECT uri, status, count(*) as requests,
       sum(CASE WHEN resulttype = 'Miss' THEN 1 ELSE 0 END) as misses
FROM cloudfront_logs
WHERE date = '2024-01-15'
GROUP BY uri, status
ORDER BY requests DESC
LIMIT 10;
```
