# Helm Interview Questions and Answers

A focused collection of Helm interview questions for DevOps engineers — covering charts, templates, releases, hooks, dependencies, security, and real-world scenarios.

---

## Table of Contents

- [Core Concepts](#core-concepts)
- [Chart Structure & Templates](#chart-structure--templates)
- [Commands & Operations](#commands--operations)
- [Advanced Topics](#advanced-topics)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Core Concepts

### 1. What is Helm and why is it used?

**Answer:** Helm is a **package manager for Kubernetes** — it simplifies defining, installing, and upgrading complex Kubernetes applications. Instead of managing dozens of individual YAML manifests, Helm packages them into a single unit called a **chart**.

**Problems Helm solves:**
- Writing and maintaining repetitive K8s YAML manifests
- Hardcoded values across environments
- No versioning or rollback for deployments
- No dependency management between services

**Three core concepts:**

| Concept | Description |
|---------|-------------|
| **Chart** | Package of pre-configured Kubernetes resources |
| **Release** | Instance of a chart deployed to a cluster |
| **Repository** | Location where charts are stored and shared |

---

### 2. What are the differences between Helm 2 and Helm 3?

**Answer:**

| Feature | Helm 2 | Helm 3 |
|---------|--------|--------|
| **Architecture** | Client + Tiller (server-side) | Client-only |
| **Security** | Tiller had cluster-wide access | Uses K8s RBAC natively |
| **Release storage** | Stored in `kube-system` namespace | Stored in release's namespace |
| **CRD support** | Limited | Native CRD support |
| **Chart dependencies** | `requirements.yaml` | `Chart.yaml` (dependencies section) |
| **Validation** | No schema validation | JSON Schema for `values.yaml` |
| **Three-way merge** | No (two-way) | Yes — detects manual changes |

Tiller was removed because it required cluster-admin access, making it a security risk. Helm 3 directly talks to the Kubernetes API server.

---

### 3. What is a Helm chart?

**Answer:** A Helm chart is a collection of files that describe a related set of Kubernetes resources. It's the packaging format for Kubernetes applications.

```bash
# Create a chart
helm create mychart

# Install a chart
helm install my-release bitnami/nginx

# Install from local directory
helm install my-release ./mychart
```

---

### 4. What is a Helm release?

**Answer:** A release is a running instance of a chart deployed to a Kubernetes cluster. Each release has a unique name and maintains its own revision history.

```bash
# Install (creates release)
helm install my-nginx bitnami/nginx

# You can install the same chart multiple times with different names
helm install nginx-prod bitnami/nginx -f values-prod.yaml
helm install nginx-staging bitnami/nginx -f values-staging.yaml

# List releases
helm list
helm list --all-namespaces
```

---

### 5. What is the purpose of `values.yaml`?

**Answer:** `values.yaml` provides default configuration values for the chart's templates. Users override these values at install/upgrade time to customize deployments.

```yaml
# values.yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  limits:
    cpu: 200m
    memory: 256Mi
```

**Override methods:**
```bash
# Using a custom values file
helm install my-release ./mychart -f values-prod.yaml

# Using --set flag
helm install my-release ./mychart --set replicaCount=3 --set image.tag=1.26

# Multiple values files (later files take precedence)
helm install my-release ./mychart -f values.yaml -f values-prod.yaml
```

---

### 6. What is a Helm repository?

**Answer:** A Helm repository is a location (HTTP server or OCI registry) where packaged charts are stored and shared.

```bash
# Add a repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable

# Update repo index
helm repo update

# Search for charts
helm search repo nginx
helm search hub nginx    # Search Artifact Hub

# Remove a repository
helm repo remove bitnami

# List repos
helm repo list
```

**OCI registry support (Helm 3.8+):**
```bash
# Push chart to OCI registry
helm push mychart-0.1.0.tgz oci://registry.example.com/charts

# Pull from OCI registry
helm pull oci://registry.example.com/charts/mychart --version 0.1.0
```

---

## Chart Structure & Templates

### 7. What is the folder structure of a Helm chart?

**Answer:**

```
mychart/
├── Chart.yaml          # Metadata (name, version, description, dependencies)
├── Chart.lock          # Lock file for dependencies
├── values.yaml         # Default configuration values
├── values.schema.json  # JSON Schema for validating values.yaml
├── charts/             # Dependency charts (subcharts)
├── crds/               # Custom Resource Definitions
├── templates/          # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── _helpers.tpl    # Template helper functions
│   ├── NOTES.txt       # Post-install usage notes
│   └── tests/
│       └── test-connection.yaml
├── .helmignore         # Files to ignore when packaging
└── README.md
```

**Key files:**
- **`Chart.yaml`** — Chart metadata, version, dependencies
- **`values.yaml`** — Default values for templates
- **`_helpers.tpl`** — Reusable template snippets (named templates)
- **`NOTES.txt`** — Displayed after install/upgrade

---

### 8. What is `_helpers.tpl` and why is it used?

**Answer:** `_helpers.tpl` contains reusable **named templates** (partials) that are included in other template files. Files starting with `_` are not rendered as Kubernetes manifests.

```yaml
# templates/_helpers.tpl
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

```yaml
# templates/deployment.yaml — using the helpers
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

---

### 9. What are Helm's built-in objects?

**Answer:** Helm provides several built-in objects accessible in templates:

| Object | Description |
|--------|-------------|
| `.Values` | Values from `values.yaml` and overrides |
| `.Release.Name` | Name of the release |
| `.Release.Namespace` | Namespace of the release |
| `.Release.Revision` | Current revision number |
| `.Release.IsUpgrade` | True if current operation is an upgrade |
| `.Chart.Name` | Chart name from `Chart.yaml` |
| `.Chart.Version` | Chart version |
| `.Chart.AppVersion` | Application version |
| `.Template.Name` | Current template file path |
| `.Capabilities` | K8s cluster capabilities (API versions) |

```yaml
# Example usage in template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
spec:
  replicas: {{ .Values.replicaCount }}
```

---

### 10. How does Helm templating work?

**Answer:** Helm uses the **Go templating engine**. Templates combine with values to generate valid Kubernetes YAML.

**Common template functions:**

```yaml
# Conditionals
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# Loops
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}

# Default values
image: {{ .Values.image.repository | default "nginx" }}

# Pipelines
name: {{ .Release.Name | upper | trunc 63 }}

# with — change scope
{{- with .Values.nodeSelector }}
nodeSelector:
  {{- toYaml . | nindent 8 }}
{{- end }}

# include named template
{{- include "mychart.labels" . | nindent 4 }}
```

---

## Commands & Operations

### 11. What are the essential Helm commands?

**Answer:**

| Command | Purpose |
|---------|---------|
| `helm create <name>` | Create a new chart |
| `helm install <release> <chart>` | Install a chart |
| `helm upgrade <release> <chart>` | Upgrade a release |
| `helm rollback <release> <revision>` | Rollback to previous version |
| `helm uninstall <release>` | Delete a release |
| `helm list` | List releases |
| `helm history <release>` | Show release history |
| `helm status <release>` | Show release status |
| `helm template <release> <chart>` | Render templates locally (no install) |
| `helm lint <chart>` | Validate chart syntax |
| `helm package <chart>` | Package chart into `.tgz` |
| `helm repo add/update/list` | Manage chart repositories |
| `helm search repo/hub <keyword>` | Search for charts |
| `helm dependency update` | Download chart dependencies |
| `helm test <release>` | Run chart tests |

---

### 12. What is the difference between `helm template` and `helm install`?

**Answer:**

| Feature | `helm template` | `helm install` |
|---------|----------------|----------------|
| **Cluster required** | No | Yes |
| **Output** | Rendered YAML to stdout | Deployed resources to cluster |
| **Release created** | No | Yes |
| **Use case** | Debug, review manifests, CI pipelines | Actual deployment |

```bash
# Render locally without deploying
helm template my-release ./mychart -f values-prod.yaml > manifests.yaml

# Useful for: code review, GitOps, debugging
# Install with dry-run (requires cluster but doesn't deploy)
helm install my-release ./mychart --dry-run --debug
```

---

### 13. How do you upgrade and rollback a Helm release?

**Answer:**

```bash
# Upgrade release
helm upgrade my-release ./mychart -f values-prod.yaml

# Upgrade with install (create if doesn't exist)
helm upgrade --install my-release ./mychart

# View release history
helm history my-release
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade complete
# 3         deployed    Upgrade complete

# Rollback to revision 2
helm rollback my-release 2

# Rollback to previous revision
helm rollback my-release
```

**Three-way merge (Helm 3):** During upgrade, Helm compares the old chart, new chart, and live cluster state. This means manual changes to resources are detected and preserved or merged correctly.

---

### 14. How do you validate and debug Helm charts?

**Answer:**

```bash
# 1. Lint — check chart for issues
helm lint ./mychart

# 2. Template — render and review output
helm template my-release ./mychart

# 3. Dry-run — simulate install against cluster
helm install my-release ./mychart --dry-run --debug

# 4. Get manifest — see what's actually deployed
helm get manifest my-release

# 5. Get values — see values used for a release
helm get values my-release

# 6. Test — run test pods defined in chart
helm test my-release
```

---

## Advanced Topics

### 15. What are Helm hooks?

**Answer:** Hooks allow you to run actions at specific points in the Helm release lifecycle.

**Hook types:**

| Annotation | When it runs |
|------------|-------------|
| `pre-install` | Before resources are installed |
| `post-install` | After all resources are installed |
| `pre-upgrade` | Before resources are upgraded |
| `post-upgrade` | After upgrade is complete |
| `pre-delete` | Before release is deleted |
| `post-delete` | After release is deleted |
| `pre-rollback` | Before rollback |
| `post-rollback` | After rollback |
| `test` | When `helm test` is called |

```yaml
# templates/db-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:{{ .Values.image.tag }}
          command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
```

**Use cases:** Database migrations, certificate provisioning, backup before upgrade, notifications.

---

### 16. How do you manage dependencies in a Helm chart?

**Answer:** Dependencies are defined in `Chart.yaml` (Helm 3):

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
# Download dependencies
helm dependency update ./myapp
# This creates charts/postgresql-12.1.0.tgz and charts/redis-17.0.0.tgz

# List dependencies
helm dependency list ./myapp
```

Override subchart values via parent's `values.yaml`:
```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    postgresPassword: "mypassword"
redis:
  enabled: false
```

---

### 17. What is an umbrella chart?

**Answer:** An umbrella chart is a chart that has **no templates of its own** but bundles multiple subcharts as dependencies. Used for deploying an entire application stack.

```yaml
# Chart.yaml (umbrella)
apiVersion: v2
name: my-platform
version: 1.0.0
dependencies:
  - name: frontend
    version: "1.2.0"
    repository: "file://../frontend-chart"
  - name: backend
    version: "2.0.0"
    repository: "file://../backend-chart"
  - name: postgresql
    version: "12.1.0"
    repository: "https://charts.bitnami.com/bitnami"
```

```bash
# Deploy entire stack in one command
helm install my-platform ./my-platform
```

---

### 18. How do you handle secrets in Helm charts?

**Answer:**

**Method 1: Kubernetes Secrets (base64 encoded)**
```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-secret
type: Opaque
data:
  password: {{ .Values.dbPassword | b64enc | quote }}
```

**Method 2: External Secrets (preferred for production)**
- **helm-secrets plugin** + SOPS for encrypted values files
- **External Secrets Operator** — syncs secrets from Vault, AWS SM, etc.
- **Sealed Secrets** — encrypt secrets that can be safely committed to Git

```bash
# Using helm-secrets plugin
helm secrets install my-release ./mychart -f secrets.yaml

# secrets.yaml is encrypted with SOPS
# Only decrypted at deploy time
```

**Never commit plaintext secrets** in `values.yaml` to Git.

---

### 19. How do you customize a Helm chart for different environments?

**Answer:** Use environment-specific values files:

```bash
# Directory structure
mychart/
├── values.yaml           # Defaults
├── values-dev.yaml       # Dev overrides
├── values-staging.yaml   # Staging overrides
└── values-prod.yaml      # Production overrides
```

```bash
# Deploy to dev
helm install myapp-dev ./mychart -f values-dev.yaml

# Deploy to prod
helm install myapp-prod ./mychart -f values-prod.yaml -n production

# Override specific value
helm install myapp ./mychart -f values-prod.yaml --set image.tag=v2.1.0
```

**Value precedence (last wins):**
`values.yaml` → `-f values-prod.yaml` → `--set image.tag=v2.1.0`

---

### 20. What is Helmfile?

**Answer:** Helmfile is a declarative tool for managing multiple Helm releases. It defines all releases in a single `helmfile.yaml`.

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

releases:
  - name: nginx
    namespace: web
    chart: bitnami/nginx
    values:
      - values/nginx.yaml

  - name: redis
    namespace: cache
    chart: bitnami/redis
    values:
      - values/redis.yaml

environments:
  production:
    values:
      - env/prod.yaml
  staging:
    values:
      - env/staging.yaml
```

```bash
# Deploy all releases
helmfile sync

# Deploy specific environment
helmfile -e production sync

# Show diff before applying
helmfile diff
```

---

### 21. What are some useful Helm plugins?

**Answer:**

| Plugin | Purpose |
|--------|---------|
| **helm-diff** | Shows diff between current and new release before upgrade |
| **helm-secrets** | Encrypt/decrypt values files using SOPS |
| **helm-unittest** | Unit test Helm chart templates |
| **helm-push** | Push charts to ChartMuseum or OCI registries |
| **helm-dashboard** | Web UI for managing Helm releases |

```bash
# Install a plugin
helm plugin install https://github.com/databus23/helm-diff

# Use helm-diff
helm diff upgrade my-release ./mychart -f values-prod.yaml
```

---

## Scenario-Based Questions

### 22. How do you use Helm in a CI/CD pipeline?

**Answer:**

```yaml
# Example: GitLab CI/CD
deploy:
  stage: deploy
  script:
    - helm repo add myrepo https://charts.example.com
    - helm repo update
    - helm upgrade --install my-release myrepo/myapp
        --namespace production
        --set image.tag=$CI_COMMIT_SHORT_SHA
        -f values-prod.yaml
        --wait
        --timeout 5m
  only:
    - main
```

**Key flags for CI/CD:**
- `--install` — Creates release if it doesn't exist
- `--wait` — Waits until pods are ready
- `--timeout` — Fails if not ready in time
- `--atomic` — Auto-rollback on failure

```bash
# Atomic deploy (auto-rollback on failure)
helm upgrade --install my-release ./mychart \
  --atomic \
  --timeout 5m \
  --set image.tag=${GIT_SHA}
```

---

### 23. A Helm upgrade failed. How do you troubleshoot?

**Answer:**

```bash
# 1. Check release status
helm status my-release

# 2. Check release history
helm history my-release

# 3. Check what was deployed
helm get manifest my-release

# 4. Check values used
helm get values my-release

# 5. Check Kubernetes events
kubectl get events --sort-by=.metadata.creationTimestamp -n <namespace>

# 6. Check pod logs
kubectl logs -l app.kubernetes.io/instance=my-release -n <namespace>

# 7. Compare with template output
helm template my-release ./mychart -f values-prod.yaml | kubectl diff -f -

# 8. Rollback if needed
helm rollback my-release
```

---

### 24. How do you integrate Helm with GitOps (ArgoCD/Flux)?

**Answer:**

**ArgoCD with Helm:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/org/helm-charts.git
    path: charts/myapp
    helm:
      valueFiles:
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

**Flux with Helm:**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: myapp
      version: "1.2.0"
      sourceRef:
        kind: HelmRepository
        name: my-repo
  values:
    replicaCount: 3
    image:
      tag: v1.2.3
```

---

### 25. What are best practices for writing Helm charts?

**Answer:**

| Practice | Description |
|----------|-------------|
| **Use `_helpers.tpl`** | Centralize labels, names, and reusable snippets |
| **Validate with JSON Schema** | Add `values.schema.json` to catch misconfiguration |
| **Use `helm lint`** | Run before every commit |
| **Default everything** | Provide sensible defaults in `values.yaml` |
| **Conditional resources** | Use `if .Values.x.enabled` for optional components |
| **Semantic versioning** | Version charts properly (MAJOR.MINOR.PATCH) |
| **Document values** | Comment every value in `values.yaml` |
| **Test charts** | Use `helm test` and `helm-unittest` |
| **Pin dependency versions** | Avoid `*` in dependency versions |
| **Avoid hardcoding** | Template everything configurable |
| **Use `--atomic`** | Auto-rollback failed deployments in CI/CD |
| **Never commit secrets** | Use helm-secrets, External Secrets, or Sealed Secrets |

---

### 26. How do you install Helm and set up a chart repository?

**Answer:**

```bash
# Install Helm (macOS)
brew install helm

# Install Helm (Linux)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install Helm (Windows)
choco install kubernetes-helm

# Verify installation
helm version

# Add chart repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Search available charts
helm search repo nginx
```

**Create a custom Helm repo (using GitHub Pages or ChartMuseum):**
```bash
# Package chart
helm package ./mychart

# Generate index file
helm repo index . --url https://my-org.github.io/helm-charts

# Push to GitHub Pages or upload to ChartMuseum
# Others can then: helm repo add myrepo https://my-org.github.io/helm-charts
```

---

### 27. How does `helm test` work and how do you write chart tests?

**Answer:** `helm test` runs test pods defined in `templates/tests/` to verify a release is working correctly.

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ .Release.Name }}-svc:{{ .Values.service.port }}']
  restartPolicy: Never
```

```bash
# Run tests after install
helm install my-release ./mychart
helm test my-release

# Output:
# Pod my-release-test-connection pending
# Pod my-release-test-connection succeeded
# NAME: my-release
# STATUS: deployed
# TEST SUITE:     my-release-test-connection
# Last Started:   ...
# Last Completed: ...
# Phase:          Succeeded
```

**Additional testing tools:**
- **`helm lint`** — Static analysis of chart
- **`helm template --dry-run`** — Validates rendered output
- **`helm-unittest`** — Unit testing for templates (no cluster needed)
- **`ct` (chart-testing)** — Lint and test charts in CI

---

### 28. How do you manage rolling updates with Helm?

**Answer:** Helm leverages Kubernetes' native rolling update strategy defined in Deployment specs.

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: {{ .Values.rollingUpdate.maxSurge | default "25%" }}
      maxUnavailable: {{ .Values.rollingUpdate.maxUnavailable | default "25%" }}
```

```yaml
# values.yaml
replicaCount: 4
rollingUpdate:
  maxSurge: "50%"
  maxUnavailable: 0    # Zero downtime
```

```bash
# Trigger rolling update by upgrading
helm upgrade my-release ./mychart --set image.tag=v2.0.0

# Monitor rollout
kubectl rollout status deployment/my-release -n production

# Rollback if issues
helm rollback my-release
```

**Other strategies via Helm:**
- **Blue-Green:** Deploy new release alongside old, then switch traffic
- **Canary:** Use `--set replicaCount=1` for canary, then scale up
- **Progressive delivery:** Integrate with Flagger or Argo Rollouts for automated canary/blue-green

---

### 29. How do you manage Helm charts in a multi-cluster environment?

**Answer:**

**Approach 1: Helmfile with multiple kubecontexts**
```yaml
# helmfile.yaml
releases:
  - name: myapp
    chart: ./mychart
    kubeContext: cluster-us-east
    values:
      - values-us-east.yaml

  - name: myapp
    chart: ./mychart
    kubeContext: cluster-eu-west
    values:
      - values-eu-west.yaml
```

**Approach 2: GitOps with ArgoCD ApplicationSets**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-multi-cluster
spec:
  generators:
    - list:
        elements:
          - cluster: cluster-us
            url: https://us-cluster.example.com
          - cluster: cluster-eu
            url: https://eu-cluster.example.com
  template:
    spec:
      source:
        repoURL: https://github.com/org/charts.git
        path: charts/myapp
        helm:
          valueFiles:
            - values-{{cluster}}.yaml
      destination:
        server: "{{url}}"
        namespace: production
```

**Best practices:**
- Store charts in a central repository (OCI registry or ChartMuseum)
- Use environment-specific values files per cluster
- Pin chart versions to prevent drift across clusters
- Use GitOps for consistent, auditable deployments

---

### 30. How do you handle chart versioning and packaging for sharing?

**Answer:**

**Versioning strategy (Semantic Versioning):**
```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.2.3       # Chart version (bump on any chart change)
appVersion: "4.5.6"   # Application version it deploys
```

| Version bump | When |
|-------------|------|
| **MAJOR** (2.0.0) | Breaking changes (renamed values, removed templates) |
| **MINOR** (1.3.0) | New features, new optional values |
| **PATCH** (1.2.4) | Bug fixes, documentation updates |

**Packaging and sharing:**
```bash
# 1. Lint before packaging
helm lint ./mychart

# 2. Package into .tgz
helm package ./mychart
# Creates: mychart-1.2.3.tgz

# 3. Push to OCI registry (preferred)
helm push mychart-1.2.3.tgz oci://registry.example.com/charts

# 4. Or host via ChartMuseum / GitHub Pages
helm repo index . --url https://charts.example.com
```

**Best practices for reusable charts:**
- Use `values.schema.json` for input validation
- Add `README.md` with all configurable values documented
- Include `NOTES.txt` for post-install instructions
- Use `_helpers.tpl` for DRY templates
- Pin all image tags — never use `latest`
- Run `helm lint` and `helm-unittest` in CI before publishing
- Sign charts with `helm package --sign` for integrity verification
