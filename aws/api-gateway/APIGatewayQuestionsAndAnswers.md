# AWS API Gateway Interview Questions and Answers

---

### 1. What is API Gateway and what types are available?

**Answer:**

| Type | Protocol | Use Case | Cost |
|------|---------|----------|------|
| **REST API** | HTTP (REST) | Full-featured: auth, throttling, caching, WAF | $3.50/million |
| **HTTP API** | HTTP | Low-latency, simpler, cheaper | $1.00/million |
| **WebSocket API** | WebSocket | Real-time: chat, notifications, streaming | $1.00/million + connection |

**Choose HTTP API** when you need a simple proxy to Lambda/ALB.
**Choose REST API** when you need caching, WAF, request validation, or API keys.

---

### 2. How does API Gateway integrate with Lambda?

**Answer:**

| Integration | Behavior |
|------------|----------|
| **Lambda Proxy** | Passes entire request to Lambda; Lambda returns full HTTP response |
| **Lambda Non-Proxy** | API Gateway transforms request/response via mapping templates |

```yaml
# SAM template — API Gateway + Lambda
Resources:
  ApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      Runtime: python3.12
      Events:
        Api:
          Type: Api
          Properties:
            Path: /users/{id}
            Method: GET
```

---

### 3. How do you implement authentication in API Gateway?

**Answer:**

| Method | How | Use Case |
|--------|-----|----------|
| **IAM** | Sigv4 signed requests | Internal AWS services |
| **Cognito Authorizer** | JWT token from Cognito User Pool | User-facing apps |
| **Lambda Authorizer** | Custom auth function (token or request-based) | Custom auth logic |
| **API Keys + Usage Plans** | API key in header | Third-party developers, rate limiting |

---

### 4. What are stages and stage variables?

**Answer:** Stages represent different environments (`dev`, `staging`, `prod`) of the same API.

```bash
# Deploy to stage
aws apigateway create-deployment --rest-api-id abc123 --stage-name prod

# Stage variables — pass environment-specific config
aws apigateway update-stage --rest-api-id abc123 --stage-name prod \
  --patch-operations op=replace,path=/variables/lambdaAlias,value=PROD
# Lambda integration: arn:aws:lambda:...:my-function:${stageVariables.lambdaAlias}
```

---

### 5. How do you implement throttling and rate limiting?

**Answer:**

```bash
# Account-level: 10,000 RPS, 5,000 burst (default)
# Stage-level: Custom per stage
# Method-level: Custom per endpoint

# Usage plan with API key
aws apigateway create-usage-plan --name "BasicPlan" \
  --throttle burstLimit=100,rateLimit=50 \
  --quota limit=10000,period=MONTH
```

---

### 6. Scenario: API Gateway returns 502 Bad Gateway. How do you troubleshoot?

**Answer:**

| Cause | Fix |
|-------|-----|
| **Lambda error** | Lambda threw unhandled exception — check Lambda logs |
| **Lambda timeout** | API GW timeout (29s max) < Lambda timeout — reduce Lambda duration |
| **Malformed response** | Lambda must return `{ statusCode, body, headers }` for proxy integration |
| **Integration error** | Backend (ALB, HTTP) unreachable — check VPC link, SG, URL |
| **Payload too large** | Response > 10 MB — paginate or compress |

```bash
# Enable CloudWatch logging for API Gateway
aws apigateway update-stage --rest-api-id abc123 --stage-name prod \
  --patch-operations op=replace,path=/accessLogSetting/destinationArn,value=arn:aws:logs:...
```

---

### 7. What are API Gateway caching and how does it work?

**Answer:** REST API supports **response caching** at the stage level (0.5 GB - 237 GB, $0.02-$4.68/hr).

- Cache key: method + resource path + query strings (configurable)
- TTL: 0-3600 seconds (default 300)
- Invalidation: `Cache-Control: max-age=0` header (must authorize)

**Best for:** GET requests with stable responses, reducing Lambda/backend invocations.

---

### 8. What is a VPC Link and when do you use it?

**Answer:** VPC Link enables API Gateway to reach **private resources** (NLB, ALB, ECS) inside your VPC.

```
Client → API Gateway → VPC Link → NLB (private) → ECS/EC2
```

- **REST API** — VPC Link to NLB
- **HTTP API** — VPC Link to ALB, NLB, or Cloud Map

---

### 9. How do you implement CORS in API Gateway?

**Answer:**

```bash
# HTTP API — CORS built-in
aws apigatewayv2 update-api --api-id abc123 \
  --cors-configuration AllowOrigins='["https://example.com"]',AllowMethods='["GET","POST"]',AllowHeaders='["Content-Type","Authorization"]'

# REST API — must configure OPTIONS method + headers on each resource
# Or use SAM/Serverless framework to auto-generate CORS config
```

---

### 10. Scenario: Your API is receiving sudden traffic spikes and returning 429 errors. How do you handle it?

**Answer:**

- **Increase throttling limits** — Request account-level increase via AWS Support
- **Enable caching** — Reduce backend calls for repeated GET requests
- **Usage plans** — Separate limits per customer/API key
- **SQS buffering** — Queue requests and process asynchronously
- **WAF rate limiting** — Block abusive clients before hitting API Gateway
- **Client-side retry with backoff** — Handle 429 gracefully
