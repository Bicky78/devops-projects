# AWS IAM Interview Questions and Answers

---

### 1. What is IAM and what are its core components?

**Answer:** IAM (Identity and Access Management) controls **who** can access **what** in AWS.

| Component | Description |
|-----------|-------------|
| **Users** | Individual identities (human or service) |
| **Groups** | Collection of users sharing permissions |
| **Roles** | Temporary credentials assumed by services, users, or external identities |
| **Policies** | JSON documents defining permissions (allow/deny) |
| **Identity Providers** | External identity sources (SAML, OIDC) for federation |

---

### 2. What is the difference between IAM Roles and IAM Users?

**Answer:**

| Feature | IAM User | IAM Role |
|---------|---------|---------|
| **Credentials** | Permanent (password, access keys) | Temporary (STS tokens, auto-rotated) |
| **Who uses it** | Humans, CI/CD services | EC2 instances, Lambda, EKS pods, cross-account |
| **Best practice** | Avoid for apps — use roles | Always prefer roles for services |

---

### 3. Explain IAM policy evaluation logic.

**Answer:**

```
Request → [1] Explicit Deny? → DENIED
        → [2] SCP (Organization) Allow? → If no → DENIED
        → [3] Resource Policy Allow? → Evaluate
        → [4] Identity Policy Allow? → If no → DENIED
        → [5] Permissions Boundary Allow? → If no → DENIED
        → ALLOWED
```

**Key rule:** An **explicit Deny** always wins, regardless of any Allow.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringNotEquals": { "aws:PrincipalTag/Team": "platform" }
      }
    }
  ]
}
```

---

### 4. What are AWS Managed Policies vs Customer Managed vs Inline Policies?

**Answer:**

| Type | Managed by | Reusable | Use case |
|------|-----------|----------|----------|
| **AWS Managed** | AWS | Yes | Common permissions (ReadOnlyAccess, PowerUserAccess) |
| **Customer Managed** | You | Yes | Organization-specific policies |
| **Inline** | Attached to single user/role/group | No | Strict 1:1 mapping (rare, avoid) |

**Best practice:** Use Customer Managed Policies for reuse. Avoid inline policies (hard to audit).

---

### 5. What is an IAM Permissions Boundary?

**Answer:** A permissions boundary **caps the maximum permissions** a user or role can have, even if their identity policy grants more.

```
Effective permissions = Identity Policy ∩ Permissions Boundary
```

**Use case:** Allow developers to create IAM roles but prevent them from creating overly permissive roles.

```json
// Permissions boundary — max allowed
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:*", "dynamodb:*", "logs:*", "cloudwatch:*"],
    "Resource": "*"
  }]
}
// Even if the role's policy grants "ec2:*", it's denied by the boundary.
```

---

### 6. Scenario: How do you set up cross-account access using IAM Roles?

**Answer:**

```
Account A (trusting) → Role with trust policy allowing Account B
Account B (trusted)  → User/Role assumes the role in Account A
```

```json
// Role in Account A — trust policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::ACCOUNT_B_ID:root" },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "unique-secret-id" }
    }
  }]
}
```

```bash
# From Account B — assume role in Account A
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_A_ID:role/CrossAccountRole \
  --role-session-name my-session \
  --external-id unique-secret-id
```

---

### 7. What are Service Control Policies (SCPs) and how do they work?

**Answer:** SCPs are **organization-level guardrails** in AWS Organizations that restrict what accounts can do.

- SCPs do **not grant** permissions — they only **limit** the maximum permissions
- Applied at OU or account level
- Even the account root user is restricted by SCPs

```json
// SCP: Deny all actions outside allowed regions
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
      },
      "ArnNotLike": {
        "aws:PrincipalARN": "arn:aws:iam::*:role/OrganizationAdmin"
      }
    }
  }]
}
```

---

### 8. What are IAM best practices for production?

**Answer:**

- **Never use root account** — Enable MFA, use only for billing/org tasks
- **Use Roles everywhere** — EC2, Lambda, EKS pods — never hardcode access keys
- **Least privilege** — Start with zero permissions, add as needed
- **Use IAM Identity Center (SSO)** — Centralized federation for human access
- **Enable MFA** — For all console users, especially privileged roles
- **Rotate credentials** — Access keys should be < 90 days old
- **Use Permissions Boundaries** — Prevent privilege escalation by delegated admins
- **Use SCPs** — Organization-level guardrails
- **Audit with Access Analyzer** — Find unused permissions and external access
- **Use Conditions** — Restrict by IP, MFA, region, tags

---

### 9. Scenario: An EC2 instance can't access S3 despite having an IAM role. How do you troubleshoot?

**Answer:**

| Check | Command/Action |
|-------|---------------|
| **Instance profile attached?** | `aws ec2 describe-instances --instance-id i-xxx` → check `IamInstanceProfile` |
| **Role policy correct?** | Check role's permissions policy allows `s3:GetObject` on the right bucket |
| **S3 bucket policy?** | Bucket policy may have explicit deny |
| **VPC endpoint policy?** | If using S3 gateway endpoint, its policy may restrict access |
| **SCP?** | Organization SCP may block S3 in this account/region |
| **KMS?** | If objects are SSE-KMS encrypted, role needs `kms:Decrypt` |
| **Test with CLI** | `aws sts get-caller-identity` from the instance to verify assumed role |

---

### 10. What is IAM Identity Center (AWS SSO)?

**Answer:** IAM Identity Center provides **centralized SSO** for all AWS accounts and cloud applications.

| Feature | IAM Users | IAM Identity Center |
|---------|----------|-------------------|
| **Scope** | Per account | All accounts in Organization |
| **Federation** | Manual SAML/OIDC per account | Built-in (Okta, Azure AD, Google) |
| **Credential type** | Long-term | Short-term (auto-rotated) |
| **Best for** | Service accounts | Human access to console/CLI |

**Best practice:** Use Identity Center for all human access. Use IAM Roles for all service/machine access.

---

### 11. Scenario: How do you prevent IAM privilege escalation in a multi-team environment?

**Answer:**

- **Permissions boundaries** — Cap max permissions when teams create roles
- **SCPs** — Deny `iam:CreateRole` without permissions boundary attached
- **Deny `iam:PassRole`** for roles outside the team's scope
- **Deny policy modification** — Teams can't edit their own boundaries
- **AWS Config rules** — Alert on roles without boundaries
- **IAM Access Analyzer** — Detect overly permissive policies

```json
// SCP: Require permissions boundary on all new roles
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": ["iam:CreateRole", "iam:AttachRolePolicy", "iam:PutRolePolicy"],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "iam:PermissionsBoundary": "arn:aws:iam::*:policy/TeamBoundary"
      }
    }
  }]
}
```
