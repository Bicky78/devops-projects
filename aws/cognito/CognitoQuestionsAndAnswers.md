# AWS Cognito Interview Questions and Answers

---

### 1. What is AWS Cognito and what are its components?

**Answer:** Cognito provides **authentication, authorization, and user management** for web/mobile apps.

| Component | Purpose |
|-----------|---------|
| **User Pools** | User directory — sign-up, sign-in, MFA, password policies, JWT tokens |
| **Identity Pools** | Federated identities — exchange tokens for temporary AWS credentials |

```
User → Sign in → Cognito User Pool → JWT Token → API Gateway (Cognito Authorizer)
                                          ↓
                      Cognito Identity Pool → AWS Credentials → S3, DynamoDB
```

---

### 2. What is the difference between User Pools and Identity Pools?

**Answer:**

| Feature | User Pool | Identity Pool |
|---------|----------|---------------|
| **Purpose** | Authentication (who are you?) | Authorization (what AWS resources?) |
| **Output** | JWT tokens (ID, Access, Refresh) | Temporary AWS credentials (STS) |
| **Use case** | App login, user management | Direct AWS access from client |
| **Federation** | Social login (Google, Facebook), SAML, OIDC | Cognito User Pool, social, SAML |

---

### 3. How do you integrate Cognito with API Gateway?

**Answer:**

```yaml
# Cognito Authorizer on API Gateway
Resources:
  CognitoAuthorizer:
    Type: AWS::ApiGateway::Authorizer
    Properties:
      Name: CognitoAuth
      Type: COGNITO_USER_POOLS
      RestApiId: !Ref MyApi
      ProviderARNs:
        - !GetAtt UserPool.Arn
      IdentitySource: method.request.header.Authorization
```

Client sends: `Authorization: Bearer <JWT_TOKEN>` → API Gateway validates with Cognito → Allows/Denies.

---

### 4. What are Cognito triggers (Lambda triggers)?

**Answer:**

| Trigger | When | Use Case |
|---------|------|----------|
| **Pre Sign-up** | Before user creation | Validate email domain, auto-confirm |
| **Post Confirmation** | After user confirms | Send welcome email, create DB record |
| **Pre Authentication** | Before sign-in | Check if user is banned |
| **Post Authentication** | After successful sign-in | Log analytics, update last login |
| **Pre Token Generation** | Before JWT issued | Add custom claims to token |
| **Custom Message** | When sending verification code | Custom email/SMS templates |
| **Migrate User** | When user doesn't exist | Migrate from legacy auth system |

---

### 5. Scenario: You need to migrate users from a custom auth system to Cognito without resetting passwords. How?

**Answer:** Use the **User Migration Lambda Trigger**.

```python
def handler(event, context):
    if event['triggerSource'] == 'UserMigration_Authentication':
        # Verify against old auth system
        user = old_auth.authenticate(event['userName'], event['request']['password'])
        if user:
            event['response']['userAttributes'] = {
                'email': user['email'],
                'email_verified': 'true'
            }
            event['response']['finalUserStatus'] = 'CONFIRMED'
            event['response']['messageAction'] = 'SUPPRESS'
    return event
```

When a user signs in and doesn't exist in Cognito, the Lambda checks the old system and migrates them transparently.

---

### 6. Scenario: How do you implement multi-tenant authentication with Cognito?

**Answer:**

| Approach | How | Best for |
|----------|-----|----------|
| **One User Pool per tenant** | Complete isolation | Strict compliance, large tenants |
| **One User Pool, custom attributes** | `custom:tenant_id` attribute | Simple, small-medium scale |
| **One User Pool, groups per tenant** | Cognito Groups = tenants | Role-based access per tenant |

**Recommendation:** Single User Pool with `custom:tenant_id` for most cases. Use Pre Token Generation trigger to inject tenant claims into JWT.
