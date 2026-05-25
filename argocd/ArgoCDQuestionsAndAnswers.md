# Argo CD Interview Questions and Answers

A focused collection of Argo CD interview questions for DevOps engineers — covering GitOps fundamentals, application management, sync strategies, deployment patterns, security, multi-cluster management, and production troubleshooting.

---

## Table of Contents

- [Core Concepts & GitOps](#core-concepts--gitops)
- [Architecture & Components](#architecture--components)
- [Installation & Configuration](#installation--configuration)
- [Application Management](#application-management)
- [Sync Policies & Strategies](#sync-policies--strategies)
- [Helm & Kustomize Integration](#helm--kustomize-integration)
- [Security & Access Control](#security--access-control)
- [Multi-Cluster & Multi-Environment](#multi-cluster--multi-environment)
- [Deployment Strategies](#deployment-strategies)
- [Monitoring, Debugging & Troubleshooting](#monitoring-debugging--troubleshooting)
- [CI/CD Integration & Best Practices](#cicd-integration--best-practices)

---

## Core Concepts & GitOps

### 1. What is Argo CD and how does it work?

**Answer:** Argo CD is a **declarative, GitOps continuous delivery tool for Kubernetes**. It is a CNCF graduated project that automates the deployment of applications by continuously monitoring Git repositories and ensuring the live state of applications matches the desired state defined in Git.

**How it works:**
1. You define your desired application state (Kubernetes manifests, Helm charts, Kustomize, etc.) in a Git repository
2. Argo CD continuously monitors the Git repo for changes
3. It compares the desired state (Git) with the live state (cluster)
4. If there's a difference, it reports **OutOfSync** status
5. Based on sync policy (manual or auto), it reconciles the live state to match Git

```
Developer → Git Push → Git Repo ← Argo CD monitors
                                        ↓
                                  Compare desired vs live
                                        ↓
                                  Sync to Kubernetes Cluster
```

---

### 2. What is GitOps and how does Argo CD implement it?

**Answer:** **GitOps** is a practice of using Git repositories as the **single source of truth** for declarative infrastructure and application configuration. All changes go through Git (commits, PRs, reviews), and an automated agent ensures the target environment matches the state in Git.

**Argo CD implements GitOps by:**
- Using Git as the source of truth for Kubernetes manifests
- Continuously pulling and comparing the desired state from Git with the live cluster state
- Automatically or manually syncing changes to the cluster
- Providing rollback by reverting to previous Git commits
- Maintaining full audit trail through Git history

**GitOps principles:**
1. **Declarative** — Entire system described declaratively
2. **Versioned and immutable** — Desired state stored in Git
3. **Pulled automatically** — Agents pull changes (not push-based)
4. **Continuously reconciled** — Software agents ensure live matches desired

---

### 3. What are the benefits of Argo CD over traditional CI/CD tools?

**Answer:**

| Feature | Traditional CI/CD (push-based) | Argo CD (GitOps pull-based) |
|---------|-------------------------------|----------------------------|
| **Deployment trigger** | CI pipeline pushes to cluster | Argo CD pulls from Git |
| **Credentials** | CI needs cluster credentials | Only Argo CD needs cluster access |
| **Drift detection** | No — manual checks | Yes — continuous monitoring |
| **Rollback** | Rerun old pipeline | Revert Git commit or one-click UI |
| **Audit trail** | Pipeline logs | Full Git history |
| **Self-healing** | No | Yes — auto-corrects manual changes |
| **Visibility** | Pipeline status only | Live app state, health, diff view |

**Key benefits:**
- **Improved security** — CI pipeline doesn't need cluster credentials
- **Faster rollbacks** — Revert Git commit, Argo CD syncs automatically
- **Drift detection & self-healing** — Detects and corrects manual cluster changes
- **Declarative** — Desired state always in Git, auditable and reviewable
- **Multi-cluster** — Manage many clusters from one Argo CD instance

---

## Architecture & Components

### 4. What are the key components of Argo CD?

**Answer:**

| Component | Role |
|-----------|------|
| **API Server** | Exposes REST/gRPC API; serves the Web UI; handles authentication and RBAC |
| **Repository Server** | Clones Git repos, generates Kubernetes manifests (renders Helm/Kustomize/Jsonnet) |
| **Application Controller** | Kubernetes controller that continuously monitors live vs desired state and reconciles |
| **Redis** | Caching layer for Git repo data and app state |
| **Dex** | Optional OIDC/SSO identity provider for authentication |
| **ApplicationSet Controller** | Manages ApplicationSet CRDs for templated multi-app generation |

```
                ┌──────────────┐
                │   Web UI /   │
                │   CLI / API  │
                └──────┬───────┘
                       │
                ┌──────▼───────┐     ┌──────────────┐
                │  API Server  │────►│     Dex      │ (SSO/OIDC)
                └──────┬───────┘     └──────────────┘
                       │
          ┌────────────┼────────────┐
          ▼                         ▼
┌──────────────────┐    ┌───────────────────┐
│ Repo Server      │    │ App Controller    │
│ (Git clone,      │    │ (Reconcile loop,  │
│  manifest render)│    │  sync, health)    │
└────────┬─────────┘    └────────┬──────────┘
         │                       │
         ▼                       ▼
    Git Repos              K8s Clusters
```

---

### 5. What are Applications and Projects in Argo CD?

**Answer:**

**Application** — A CRD that defines:
- **Source:** Git repo, path, branch/tag, and tool (Helm/Kustomize/plain YAML)
- **Destination:** Target Kubernetes cluster and namespace
- **Sync policy:** Manual, auto-sync, self-heal, prune

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: apps/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**AppProject** — Groups applications and enforces policies:
- Restricts which Git repos can be used
- Restricts which clusters/namespaces can be targeted
- Limits which Kubernetes resource kinds can be deployed
- Controls RBAC (who can sync which apps)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-backend
  namespace: argocd
spec:
  description: Backend team project
  sourceRepos:
    - 'https://github.com/org/backend-*'
  destinations:
    - namespace: 'backend-*'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: ''
      kind: Service
```

---

### 6. What is ApplicationSet in Argo CD?

**Answer:** **ApplicationSet** is a CRD that allows you to create multiple Argo CD Applications from a single template using **generators**. It solves the problem of managing many similar applications across environments or clusters.

**Generators:**

| Generator | Use Case |
|-----------|----------|
| **List** | Static list of key-value pairs |
| **Cluster** | One app per registered cluster |
| **Git Directory** | One app per directory in a Git repo |
| **Git File** | One app per config file in a Git repo |
| **Matrix** | Combine two generators (e.g., cluster × environment) |
| **Merge** | Merge results from multiple generators |
| **Pull Request** | One app per open PR (for preview environments) |

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-all-envs
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: https://dev-cluster.example.com
          - env: staging
            cluster: https://staging-cluster.example.com
          - env: prod
            cluster: https://prod-cluster.example.com
  template:
    metadata:
      name: 'myapp-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/k8s-manifests.git
        targetRevision: main
        path: 'envs/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: myapp
```

---

## Installation & Configuration

### 7. How do you install and access Argo CD?

**Answer:**

```bash
# Create namespace
kubectl create namespace argocd

# Install Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# For HA installation (production)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/part-of=argocd -n argocd --timeout=300s

# Get initial admin password
argocd admin initial-password -n argocd

# Access via port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access via CLI
argocd login localhost:8080 --username admin --password <password>

# Change admin password
argocd account update-password

# Add external cluster
argocd cluster add my-cluster-context
```

---

### 8. What is the purpose of `argocd-cm` ConfigMap?

**Answer:** `argocd-cm` is the **main configuration ConfigMap** for Argo CD. It stores critical settings:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Repository credentials
  repositories: |
    - url: https://github.com/org/manifests.git
      passwordSecret:
        name: repo-creds
        key: password
      usernameSecret:
        name: repo-creds
        key: username

  # SSO/OIDC configuration
  oidc.config: |
    name: Okta
    issuer: https://org.okta.com
    clientID: argo-cd
    clientSecret: $oidc.okta.clientSecret

  # Custom health checks (Lua)
  resource.customizations.health.argoproj.io_Application: |
    hs = {}
    hs.status = "Healthy"
    if obj.status ~= nil then
      if obj.status.health ~= nil then
        hs.status = obj.status.health.status
      end
    end
    return hs

  # Kustomize build options
  kustomize.buildOptions: --load-restrictor LoadRestrictionsNone

  # Resource exclusions
  resource.exclusions: |
    - apiGroups: ["cilium.io"]
      kinds: ["CiliumIdentity"]
      clusters: ["*"]
```

**Other important ConfigMaps:**
- **`argocd-rbac-cm`** — RBAC policies
- **`argocd-cmd-params-cm`** — CLI/server parameters
- **`argocd-notifications-cm`** — Notification triggers and templates

---

## Application Management

### 9. How does Argo CD sync applications with Kubernetes?

**Answer:** Sync is the process of making the live cluster state match the desired state in Git.

**Sync flow:**
1. **Fetch** — Repo Server clones the Git repo at the specified revision
2. **Render** — Generates Kubernetes manifests (renders Helm/Kustomize if used)
3. **Compare** — App Controller compares rendered manifests with live resources
4. **Apply** — If OutOfSync, applies the diff to the cluster using `kubectl apply`
5. **Health check** — Waits for resources to become healthy
6. **Report** — Updates Application status (Synced/Healthy or Degraded)

```bash
# Manual sync via CLI
argocd app sync myapp

# Sync with prune (delete removed resources)
argocd app sync myapp --prune

# Sync specific resources only
argocd app sync myapp --resource ':Service:myapp-svc'

# Dry run (preview without applying)
argocd app sync myapp --dry-run

# Force sync (override immutable fields)
argocd app sync myapp --force

# Refresh (re-read Git without syncing)
argocd app get myapp --refresh
```

---

### 10. What is the difference between manual sync, auto sync, self-heal, and prune?

**Answer:**

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Manual sync** | User must click "Sync" in UI or run CLI | Production (requires human approval) |
| **Auto sync** | Syncs automatically when Git changes detected | Dev/staging environments |
| **Self-heal** | Auto-reverts manual changes to cluster resources | Prevent configuration drift |
| **Prune** | Deletes resources removed from Git | Clean up old resources |

```yaml
spec:
  syncPolicy:
    automated:
      prune: true        # Delete resources not in Git
      selfHeal: true     # Revert manual cluster changes
      allowEmpty: false   # Prevent accidental deletion of all resources
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Important:** Even with auto-sync, Argo CD will **not** auto-sync if the app is in an **error state** — this prevents infinite sync loops.

---

### 11. What are sync waves and hooks in Argo CD?

**Answer:**

**Sync Waves** control the **order** in which resources are applied during a sync. Resources with lower wave numbers are applied first.

```yaml
# Wave 0: Namespace and ConfigMaps (applied first)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"

# Wave 1: Database
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"

# Wave 2: Application (depends on DB)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"

# Wave 3: Ingress (applied last)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "3"
```

**Hooks** run at specific points during the sync lifecycle:

| Hook | When it runs |
|------|-------------|
| `PreSync` | Before sync (e.g., DB migration, backup) |
| `Sync` | During sync (alongside resources) |
| `PostSync` | After sync (e.g., integration tests, notifications) |
| `SyncFail` | When sync fails (e.g., cleanup, alerts) |
| `Skip` | Skip this resource during sync |

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:latest
          command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
```

**Hook delete policies:**
- `HookSucceeded` — Delete after successful execution
- `HookFailed` — Delete after failure
- `BeforeHookCreation` — Delete previous hook before creating new one

---

### 12. How do you handle rollback in Argo CD?

**Answer:**

```bash
# View application history
argocd app history myapp

# Rollback to specific revision
argocd app rollback myapp <revision-number>

# Or revert the Git commit (preferred GitOps approach)
git revert <commit-sha>
git push origin main
# Argo CD auto-syncs to the reverted state
```

**Via UI:** Application → History → Select revision → Rollback

**Important considerations:**
- With **auto-sync enabled**, `argocd app rollback` will be immediately overridden by the next sync from Git. The GitOps-correct approach is to **revert the Git commit**
- With **manual sync**, rollback via CLI/UI will persist until next manual sync
- Argo CD keeps a configurable number of revisions in history (default: 10)

---

## Helm & Kustomize Integration

### 13. How do you integrate Argo CD with Helm charts?

**Answer:**

```yaml
# Method 1: Helm chart from Git repo
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/org/charts.git
    path: charts/myapp
    targetRevision: main
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: "v1.5.0"
        - name: replicas
          value: "3"

# Method 2: Helm chart from Helm repository
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-ingress
spec:
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.8.3
    helm:
      releaseName: ingress-nginx
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
```

```bash
# Override Helm values via CLI
argocd app set myapp --helm-set image.tag=v2.0.0
argocd app set myapp --values values-staging.yaml
```

---

### 14. How do you integrate Argo CD with Kustomize?

**Answer:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    path: overlays/production
    targetRevision: main
    kustomize:
      namePrefix: prod-
      nameSuffix: -v2
      images:
        - myapp=registry.example.com/myapp:v1.5.0
      commonLabels:
        env: production
      commonAnnotations:
        team: backend
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

**Typical Kustomize directory structure:**
```
k8s-manifests/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

---

## Security & Access Control

### 15. How do you secure Argo CD access and authentication?

**Answer:**

**Authentication methods:**

| Method | Configuration |
|--------|--------------|
| **Local users** | `argocd-cm` ConfigMap |
| **SSO/OIDC** | Okta, Auth0, Keycloak, Google, Azure AD |
| **SAML** | Via Dex connector |
| **LDAP** | Via Dex connector |
| **GitHub/GitLab OAuth** | Via Dex connector |

```yaml
# argocd-cm — OIDC example (Okta)
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  oidc.config: |
    name: Okta
    issuer: https://org.okta.com
    clientID: argo-cd
    clientSecret: $oidc.okta.clientSecret
    requestedScopes:
      - openid
      - profile
      - email
      - groups
```

**Security best practices:**
- **Disable admin account** after configuring SSO: `admin.enabled: "false"` in `argocd-cm`
- **Enable HTTPS** with proper TLS certificates
- **Use RBAC** to restrict who can sync/delete apps
- **GPG signature verification** for Git commits
- **Network policies** to restrict access to Argo CD pods
- **Audit logging** — all operations are logged

---

### 16. How do you configure RBAC in Argo CD?

**Answer:** RBAC is configured in the `argocd-rbac-cm` ConfigMap using a CSV-based policy format.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Roles
    p, role:dev, applications, get, */*, allow
    p, role:dev, applications, sync, dev/*, allow

    p, role:ops, applications, *, */*, allow
    p, role:ops, clusters, get, *, allow

    p, role:admin, *, *, */*, allow

    # Group bindings (from SSO groups)
    g, dev-team, role:dev
    g, ops-team, role:ops
    g, platform-team, role:admin

  scopes: '[groups]'
```

**Policy format:** `p, <role>, <resource>, <action>, <project>/<object>, <allow/deny>`

**Resources:** `applications`, `clusters`, `repositories`, `projects`, `accounts`, `logs`, `exec`

**Actions:** `get`, `create`, `update`, `delete`, `sync`, `override`, `action`

---

### 17. How do you manage secrets securely in Argo CD?

**Answer:** Argo CD itself does **not** encrypt secrets in Git. You need external tools:

| Tool | How it works |
|------|-------------|
| **Sealed Secrets** | Encrypts secrets client-side; only the cluster can decrypt |
| **SOPS + age/KMS** | Encrypts YAML values in Git; decrypted at sync time via Argo CD plugin |
| **HashiCorp Vault** | Argo CD Vault Plugin fetches secrets at sync time |
| **External Secrets Operator** | K8s operator syncs secrets from Vault/AWS SM/GCP SM into K8s Secrets |
| **AWS Secrets Manager / GCP Secret Manager** | Via External Secrets Operator |

**Example: Sealed Secrets**
```bash
# Encrypt secret
kubeseal --controller-namespace kube-system \
  --controller-name sealed-secrets \
  --format yaml < secret.yaml > sealed-secret.yaml

# Commit sealed-secret.yaml to Git (safe — it's encrypted)
# Argo CD syncs it → SealedSecrets controller decrypts → creates K8s Secret
```

**Example: External Secrets Operator**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: password
      remoteRef:
        key: prod/database
        property: password
```

---

## Multi-Cluster & Multi-Environment

### 18. How do you manage multiple Kubernetes clusters in Argo CD?

**Answer:**

```bash
# Register a cluster
argocd cluster add staging-cluster-context
argocd cluster add prod-cluster-context

# List registered clusters
argocd cluster list

# Remove a cluster
argocd cluster rm https://staging-cluster.example.com
```

**Deploy to specific cluster:**
```yaml
spec:
  destination:
    server: https://prod-cluster.example.com   # Target cluster
    namespace: myapp
```

**ApplicationSet for multi-cluster:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-all-clusters
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: production
  template:
    metadata:
      name: 'myapp-{{name}}'
    spec:
      source:
        repoURL: https://github.com/org/manifests.git
        path: overlays/production
      destination:
        server: '{{server}}'
        namespace: myapp
```

**Architecture patterns:**
- **Hub-spoke** — Single Argo CD manages all clusters (most common)
- **Argo CD per cluster** — Each cluster has its own Argo CD (higher isolation)
- **Hybrid** — One Argo CD per environment tier (dev, staging, prod)

---

### 19. How do you promote applications across environments?

**Answer:**

**Pattern 1: Directory-based (recommended for Kustomize)**
```
repo/
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   ├── staging/
│   └── prod/
```
Promote by updating the overlay (e.g., change image tag in `staging/kustomization.yaml`).

**Pattern 2: Branch-based**
```
main branch     → production
staging branch  → staging
develop branch  → development
```
Promote by merging branches (e.g., `staging` → `main`).

**Pattern 3: Helm values per environment**
```yaml
# Application per environment
source:
  helm:
    valueFiles:
      - values.yaml
      - values-{{ env }}.yaml
```

**Best practice:** Use directory-based or values-based promotion with PRs for audit trail. Avoid branch-based for production — it's harder to track what's deployed.

---

## Deployment Strategies

### 20. How do you implement blue-green deployment with Argo CD?

**Answer:** Blue-green deployments require **Argo Rollouts** (a companion controller).

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: myapp-active       # Current live traffic
      previewService: myapp-preview     # New version for testing
      autoPromotionEnabled: false       # Manual promotion
      prePromotionAnalysis:
        templates:
          - templateName: smoke-test
        args:
          - name: service-name
            value: myapp-preview
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:v2.0.0
          ports:
            - containerPort: 8080
```

**Flow:** Deploy new version → Route preview traffic → Run tests → Promote (switch active) or Rollback.

---

### 21. How do you implement canary deployment with Argo CD?

**Answer:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10       # 10% traffic to canary
        - pause: { duration: 5m }
        - setWeight: 30       # 30% traffic
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 50       # 50% traffic
        - pause: { duration: 10m }
        - setWeight: 100      # Full rollout
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        istio:
          virtualService:
            name: myapp-vsvc
            routes:
              - primary
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:v2.0.0
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 60s
      successCondition: result[0] >= 0.95
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status=~"2.*", app="myapp-canary"}[5m]))
            /
            sum(rate(http_requests_total{app="myapp-canary"}[5m]))
```

**Supported traffic routers:** Istio, Nginx Ingress, ALB, SMI, Traefik, Ambassador.

---

## Monitoring, Debugging & Troubleshooting

### 22. What is application drift detection in Argo CD?

**Answer:** Drift detection is Argo CD's continuous comparison of the **desired state** (Git) with the **live state** (cluster). When someone makes a manual change directly to the cluster (bypassing Git), Argo CD detects this as **drift**.

**Drift scenarios:**
- Someone runs `kubectl edit deployment myapp` directly
- An admission webhook modifies resources
- A CRD controller adds labels or annotations

**How Argo CD handles drift:**
- Reports the app as **OutOfSync**
- Shows a **diff** in the UI between desired and live state
- With **self-heal enabled** → automatically reverts the drift
- Without self-heal → waits for manual sync

```bash
# View diff
argocd app diff myapp

# View detailed diff in UI: Application → App Details → Diff
```

---

### 23. How do you debug failed sync operations?

**Answer:**

```bash
# 1. Check application status
argocd app get myapp

# 2. View sync details and events
argocd app get myapp --show-operation

# 3. View application logs
argocd app logs myapp

# 4. View detailed diff
argocd app diff myapp

# 5. Check Argo CD controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=100

# 6. Check repo server logs (manifest rendering issues)
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=100

# 7. Check Kubernetes events
kubectl get events -n <app-namespace> --sort-by='.lastTimestamp'

# 8. Validate manifests locally
argocd app manifests myapp > /tmp/manifests.yaml
kubectl apply --dry-run=server -f /tmp/manifests.yaml
```

**Common causes of sync failures:**
- **Invalid manifests** — YAML syntax errors, missing required fields
- **RBAC** — Argo CD service account lacks permissions
- **Resource conflicts** — Immutable field changes (use `--force` or recreate)
- **Git access** — Repository credentials expired or SSH key issues
- **Helm/Kustomize errors** — Rendering failures, missing values
- **Webhook/admission controller** — Rejecting resources
- **Namespace** — Target namespace doesn't exist (add `CreateNamespace=true`)

---

### 24. How do you monitor Argo CD applications?

**Answer:**

**Built-in metrics (Prometheus):**
```yaml
# ServiceMonitor for Argo CD
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: argocd
  endpoints:
    - port: metrics
```

**Key metrics to monitor:**

| Metric | What it tells you |
|--------|-------------------|
| `argocd_app_info` | Application status (synced, healthy, degraded) |
| `argocd_app_sync_total` | Total sync operations (success/failure) |
| `argocd_app_reconcile_count` | Reconciliation loop count |
| `argocd_app_reconcile_bucket` | Reconciliation duration |
| `argocd_git_request_total` | Git fetch operations |
| `argocd_cluster_api_resources` | Cluster API resource count |

**Argo CD Notifications (alerts):**
```yaml
# argocd-notifications-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [slack-notification]
  template.slack-notification: |
    message: |
      Application {{.app.metadata.name}} sync {{.app.status.operationState.phase}}.
      {{.app.status.operationState.message}}
  service.slack: |
    token: $slack-token
    channel: argocd-alerts
```

---

### 25. How do you optimize Argo CD performance?

**Answer:**

**Problem areas and solutions:**

| Problem | Solution |
|---------|----------|
| **Slow Git cloning** | Enable shallow cloning, increase repo server cache TTL |
| **Too many reconciliations** | Increase `--app-resync` interval (default: 3 min) |
| **Large repos** | Split repos by team/environment, use directory generators |
| **Many applications** | Use ApplicationSets, increase controller replicas |
| **Resource-heavy manifests** | Use server-side apply, exclude unnecessary resources |
| **Redis memory** | Increase Redis memory limits, enable eviction policies |

```yaml
# argocd-cmd-params-cm — performance tuning
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  # Increase controller parallelism
  controller.status.processors: "50"    # Default: 20
  controller.operation.processors: "25"  # Default: 10

  # Reduce repo server load
  reposerver.parallelism.limit: "5"

  # Increase resync interval (seconds)
  timeout.reconciliation: "300"         # Default: 180
```

```bash
# Exclude resources from tracking
# In argocd-cm:
resource.exclusions: |
  - apiGroups: [""]
    kinds: ["Event"]
    clusters: ["*"]
  - apiGroups: ["cilium.io"]
    kinds: ["CiliumIdentity"]
    clusters: ["*"]
```

---

## CI/CD Integration & Best Practices

### 26. How do you integrate Argo CD with CI pipelines?

**Answer:**

**Pattern: CI builds image → updates Git → Argo CD syncs**

```yaml
# GitHub Actions example
name: CI/CD Pipeline
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        run: |
          docker build -t registry.example.com/myapp:${{ github.sha }} .
          docker push registry.example.com/myapp:${{ github.sha }}

      - name: Update K8s manifests
        run: |
          git clone https://github.com/org/k8s-manifests.git
          cd k8s-manifests
          # Update image tag
          sed -i "s|image: registry.example.com/myapp:.*|image: registry.example.com/myapp:${{ github.sha }}|" apps/myapp/deployment.yaml
          git commit -am "Update myapp to ${{ github.sha }}"
          git push
          # Argo CD detects Git change → syncs automatically
```

**Webhook-based (faster sync):**
```bash
# Configure webhook in Git repo settings
# Argo CD webhook URL: https://argocd.example.com/api/webhook

# Or trigger sync from CI pipeline
argocd app sync myapp --revision $GIT_SHA
argocd app wait myapp --timeout 300
```

**Image Updater (automated image tag updates):**
```yaml
# Argo CD Image Updater annotation
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=registry.example.com/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^v\d+\.\d+\.\d+$
```

---

### 27. What is the difference between `argocd app sync` and `argocd app refresh`?

**Answer:**

| Command | What it does |
|---------|-------------|
| `argocd app refresh` | Re-reads the Git repo and compares with live state. **Does not apply** any changes. Updates the sync status. |
| `argocd app sync` | Applies the desired state from Git to the cluster. **Actually deploys** changes. |

```bash
# Just check if app is in sync (no changes applied)
argocd app get myapp --refresh

# Actually deploy changes to cluster
argocd app sync myapp
```

**Analogy:** `refresh` is like checking for updates. `sync` is like installing updates.

---

### 28. How do you handle Argo CD in HA (High Availability) mode?

**Answer:**

```bash
# Install HA manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

**HA components:**
- **API Server** — Multiple replicas behind a load balancer
- **Repo Server** — Multiple replicas for parallel manifest rendering
- **Application Controller** — Uses leader election (only one active, others standby)
- **Redis** — HA Redis with sentinel or Redis cluster

**Production recommendations:**
- Minimum 3 replicas of API server and repo server
- Use external Redis cluster (not embedded)
- Place Argo CD on dedicated nodes with resource guarantees
- Configure pod disruption budgets
- Regular backups of Argo CD state (Applications, Projects, repos, clusters)

---

### 29. What are common Argo CD production issues and fixes?

**Answer:**

| Issue | Cause | Fix |
|-------|-------|-----|
| **OutOfSync but no diff** | Defaulting by admission webhooks | Add resource ignore rules in `argocd-cm` |
| **Sync stuck "Progressing"** | Health check never passes | Custom health check Lua script or increase timeout |
| **"rpc error: code = Unknown"** | Repo server OOM or crash | Increase repo server memory limits |
| **Slow UI** | Too many apps or large manifests | Increase API server resources, use projects to scope views |
| **Git "authentication failed"** | Expired token or SSH key | Rotate credentials in `argocd-cm` or Secret |
| **"ComparisonError"** | CRD not installed or schema mismatch | Install CRDs first (sync wave -1), or use `SkipDryRunOnMissingResource` |
| **Self-heal loop** | Admission controller constantly mutating resources | Use resource ignore differences annotation |

```yaml
# Ignore specific fields that drift due to controllers/webhooks
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas      # Ignore if HPA manages replicas
    - group: ""
      kind: Service
      jqPathExpressions:
        - .metadata.annotations."cloud.google.com/neg-status"
```

---

### 30. What are best practices for Argo CD in production?

**Answer:**

**Repository structure:**
- **Separate app code repo from K8s manifests repo** — allows independent deployment cycles
- Use **directory-per-environment** or **values-per-environment** structure
- Keep charts and manifests in a **dedicated GitOps repo**

**Security:**
- Disable admin account after SSO setup
- Use RBAC with least-privilege per team/project
- Never store plaintext secrets in Git — use Sealed Secrets, External Secrets, or Vault
- Enable GPG commit verification for production apps

**Operations:**
- Use **AppProjects** to isolate teams — restrict repos, clusters, namespaces
- Use **ApplicationSets** for managing similar apps at scale
- Enable **auto-sync + self-heal** for dev/staging; **manual sync** for production
- Configure **notifications** (Slack/Teams/email) for sync failures
- Set up **Prometheus + Grafana** dashboards for Argo CD metrics
- Use **sync waves** for ordered deployments (CRDs → namespaces → DB → app → ingress)
- Implement **retry policies** for transient failures
- **Backup Argo CD** — export Applications, Projects, repos, clusters regularly

```bash
# Export all applications (backup)
kubectl get applications -n argocd -o yaml > argocd-apps-backup.yaml
kubectl get appprojects -n argocd -o yaml > argocd-projects-backup.yaml
```
