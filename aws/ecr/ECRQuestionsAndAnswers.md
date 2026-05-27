# AWS ECR Interview Questions and Answers

---

### 1. What is Amazon ECR?

**Answer:** ECR (Elastic Container Registry) is a fully managed Docker container registry for storing, managing, and deploying container images.

**Key features:** Integrated with ECS/EKS/Lambda, image scanning, lifecycle policies, cross-region/cross-account replication, OCI artifact support, and encryption at rest.

---

### 2. How do you push and pull images from ECR?

**Answer:**

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp --image-scanning-configuration scanOnPush=true

# Build, tag, push
docker build -t myapp:v1.0 .
docker tag myapp:v1.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0

# Pull
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
```

---

### 3. What are ECR lifecycle policies and why are they important?

**Answer:** Lifecycle policies automatically **clean up old images** to save storage costs.

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep only 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Remove untagged images older than 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": { "type": "expire" }
    }
  ]
}
```

---

### 4. How does ECR image scanning work?

**Answer:**

| Type | When | Engine |
|------|------|--------|
| **Basic scanning** | On push or manual | Clair (CVE database) |
| **Enhanced scanning** | Continuous (real-time) | Amazon Inspector |

```bash
# Enable scan on push
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Get scan findings
aws ecr describe-image-scan-findings --repository-name myapp --image-id imageTag=v1.0
```

---

### 5. How do you set up cross-account and cross-region ECR access?

**Answer:**

```json
// Repository policy — allow cross-account pull
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCrossAccountPull",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::OTHER_ACCOUNT:root" },
    "Action": ["ecr:GetDownloadUrlForLayer", "ecr:BatchGetImage", "ecr:BatchCheckLayerAvailability"]
  }]
}
```

```bash
# Cross-region replication
aws ecr put-replication-configuration --replication-configuration '{
  "rules": [{"destinations": [{"region": "eu-west-1", "registryId": "123456789"}]}]
}'
```

---

### 6. Scenario: EKS pods fail with ImagePullBackOff for ECR images. How do you troubleshoot?

**Answer:**

| Check | Fix |
|-------|-----|
| **ECR auth token expired** | Token valid for 12h — ensure node refreshes it |
| **IAM permissions** | Node role needs `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer` |
| **Image tag exists?** | `aws ecr describe-images --repository-name myapp` |
| **VPC Endpoint** | Private nodes need ECR VPC endpoints (`ecr.api`, `ecr.dkr`, S3 gateway) |
| **Repository policy** | Cross-account? Check repository policy allows pull |
| **Disk full on node** | Old images consuming disk — kubelet image GC |

---

### 7. What is ECR pull-through cache?

**Answer:** Pull-through cache automatically caches images from public registries (Docker Hub, GitHub, Quay) in your private ECR.

```bash
# Create pull-through cache rule for Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix docker-hub \
  --upstream-registry-url registry-1.docker.io

# Now pull Docker Hub images via ECR
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/docker-hub/nginx:latest
# First pull caches; subsequent pulls served from ECR
```

**Benefits:** Avoids Docker Hub rate limits, faster pulls from ECR, works with private EKS nodes.

---

### 8. How do you secure ECR?

**Answer:**

- **Encryption** — SSE-KMS for images at rest
- **Immutable tags** — Prevent tag overwrite (`imageTagMutability: IMMUTABLE`)
- **Image scanning** — Enhanced scanning with Inspector for continuous CVE detection
- **Repository policies** — Restrict who can push/pull
- **VPC endpoints** — Private access from EKS/ECS without internet
- **IAM** — Least privilege — separate push (CI/CD) and pull (runtime) permissions
- **Lifecycle policies** — Remove old/vulnerable images automatically

---

### 9. Scenario: Your CI/CD pipeline is slow because Docker builds push large images. How do you optimize?

**Answer:**

- **Multi-stage builds** — Separate build and runtime stages
- **Smaller base images** — `alpine`, `distroless`, `scratch`
- **Layer caching** — Order Dockerfile for max cache reuse (deps before code)
- **BuildKit** — `DOCKER_BUILDKIT=1` for parallel builds and better caching
- **ECR cache** — Push/pull cache layers using `--cache-from`

```dockerfile
# Multi-stage build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

---

### 10. What is the difference between ECR Public and ECR Private?

**Answer:**

| Feature | ECR Private | ECR Public |
|---------|------------|-----------|
| **Access** | IAM-authenticated | Public (anyone can pull) |
| **Use case** | Internal app images | Open-source projects, shared tools |
| **Registry URL** | `<account>.dkr.ecr.<region>.amazonaws.com` | `public.ecr.aws/<alias>` |
| **Cost** | Storage + transfer | 50 GB free, then standard pricing |
| **Region** | Any | us-east-1 only |
