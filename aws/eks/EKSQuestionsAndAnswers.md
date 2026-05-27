# AWS EKS Interview Questions and Answers

Comprehensive EKS questions for DevOps engineers — covering cluster architecture, networking, upgrades, add-ons, node management, security, and real-world scenario-based questions.

---

## Table of Contents

- [EKS Fundamentals](#eks-fundamentals)
- [Networking](#networking)
- [Node Groups & Compute](#node-groups--compute)
- [EKS Add-ons](#eks-add-ons)
- [EKS Upgrades](#eks-upgrades)
- [Security](#security)
- [Scenario-Based Questions](#scenario-based-questions)

---

## EKS Fundamentals

### 1. What is Amazon EKS and how does it differ from self-managed Kubernetes?

**Answer:** Amazon EKS is a **managed Kubernetes service** that runs the Kubernetes control plane across multiple AZs.

| Feature | EKS (managed) | Self-managed (kubeadm/kops) |
|---------|--------------|---------------------------|
| **Control plane** | AWS-managed, HA (3 AZs) | You manage etcd, API server, scheduler |
| **Upgrades** | AWS handles control plane upgrade | You handle everything |
| **etcd** | Managed, encrypted, backed up | You manage, backup, restore |
| **SLA** | 99.95% uptime | Your responsibility |
| **Patching** | Control plane auto-patched | You patch all components |
| **Cost** | $0.10/hr per cluster + nodes | Node cost only |
| **Integration** | Native AWS (IAM, ALB, EBS, VPC) | Manual integration |

**EKS Architecture:**
```
┌─────────────────────────────────────────────────┐
│           AWS-Managed Control Plane              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │API Server│  │API Server│  │API Server│ (3 AZs)│
│  └──────────┘  └──────────┘  └──────────┘      │
│  ┌──────────────────────────────────────┐       │
│  │           etcd (managed)             │       │
│  └──────────────────────────────────────┘       │
└──────────────────────┬──────────────────────────┘
                       │ (ENI in your VPC)
┌──────────────────────▼──────────────────────────┐
│              Your VPC / Data Plane               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │ Node    │  │ Node    │  │ Node    │          │
│  │ (EC2)   │  │ (EC2)   │  │(Fargate)│          │
│  │ AZ-a    │  │ AZ-b    │  │ AZ-c    │          │
│  └─────────┘  └─────────┘  └─────────┘         │
└─────────────────────────────────────────────────┘
```

---

### 2. What are the different compute options in EKS?

**Answer:**

| Option | Management | Use Case |
|--------|-----------|----------|
| **Managed Node Groups** | AWS manages ASG, AMI updates, draining | Most production workloads |
| **Self-managed Nodes** | You manage ASG, AMI, lifecycle | Custom AMIs, GPU, Windows |
| **Fargate** | Fully serverless, per-pod billing | Batch jobs, low-traffic services |
| **Karpenter** | Auto-provisions right-sized nodes | Dynamic workloads, cost optimization |

```bash
# Create managed node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name app-nodes \
  --node-role arn:aws:iam::xxx:role/eks-node-role \
  --subnets subnet-aaa subnet-bbb \
  --instance-types m5.xlarge m5a.xlarge m6i.xlarge \
  --scaling-config minSize=2,maxSize=20,desiredSize=3 \
  --capacity-type ON_DEMAND \
  --ami-type AL2_x86_64
```

---

### 3. What is the EKS API server endpoint and what access modes are available?

**Answer:**

| Mode | Public Endpoint | Private Endpoint | Use Case |
|------|----------------|-----------------|----------|
| **Public only** (default) | Enabled | Disabled | Dev/test, simple setup |
| **Public + Private** | Enabled | Enabled | Migration, external CI/CD |
| **Private only** | Disabled | Enabled | Production, max security |

```bash
# Update to private only
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true

# With public + restricted CIDRs
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config endpointPublicAccess=true,publicAccessCidrs=["203.0.113.0/24","10.0.0.0/8"]
```

**Private endpoint implications:**
- `kubectl` must run from within VPC (bastion, VPN, or Cloud9)
- CI/CD runners need VPC connectivity
- Nodes communicate with API server via private ENIs

---

## Networking

### 4. How does EKS networking work with the VPC CNI plugin?

**Answer:** The **Amazon VPC CNI** assigns real VPC IP addresses to each pod, unlike overlay networks (Calico, Flannel).

**How it works:**
1. Each node gets multiple ENIs (Elastic Network Interfaces)
2. Each ENI gets multiple secondary IP addresses
3. Each pod gets one of these secondary IPs
4. Pods can directly communicate using VPC routing — no encapsulation overhead

**Max pods per node** = (Number of ENIs × IPs per ENI) - reserved

| Instance Type | Max ENIs | IPs per ENI | Max Pods |
|--------------|---------|-------------|----------|
| t3.medium | 3 | 6 | 17 |
| m5.large | 3 | 10 | 29 |
| m5.xlarge | 4 | 15 | 58 |
| m5.2xlarge | 4 | 15 | 58 |

```bash
# Check max pods for an instance type
kubectl describe node <node-name> | grep -i "allocatable" -A 5

# Enable prefix delegation (more pods per node)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
# With prefix delegation: each ENI slot gets a /28 prefix (16 IPs) instead of 1 IP
# m5.xlarge: 4 ENIs × 15 slots × 16 IPs = ~960 pods (vs 58 without)
```

---

### 5. What is EKS custom networking and when do you need it?

**Answer:** Custom networking assigns pod IPs from a **different subnet/CIDR** than the node IPs.

**When you need it:**
- **Running out of IPs** in primary VPC CIDR
- **Security** — Pods in a different subnet with different NACLs/routing
- **IP separation** — Node IPs from primary CIDR, pod IPs from secondary CIDR

```bash
# 1. Add secondary CIDR to VPC
aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxx --cidr-block 100.64.0.0/16

# 2. Create subnets in secondary CIDR
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 100.64.1.0/24 --az us-east-1a

# 3. Enable custom networking on VPC CNI
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

# 4. Create ENIConfig per AZ
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a
spec:
  subnet: subnet-pod-az-a
  securityGroups:
    - sg-pod-sg
```

---

### 6. How does the AWS Load Balancer Controller work in EKS?

**Answer:** The **AWS Load Balancer Controller** provisions ALBs and NLBs for Kubernetes Ingress and Service resources.

| Kubernetes Resource | AWS Resource | Use Case |
|-------------------|-------------|----------|
| `Ingress` | ALB | HTTP/HTTPS routing, path-based |
| `Service (type: LoadBalancer)` | NLB | TCP/UDP, gRPC, static IPs |

```yaml
# Ingress → ALB
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip        # Pod IPs directly
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080

# Service → NLB
apiVersion: v1
kind: Service
metadata:
  name: myapp-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internal
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 8080
  selector:
    app: myapp
```

**target-type `ip` vs `instance`:**
- **ip** — Registers pod IPs directly (recommended, faster, no NodePort needed)
- **instance** — Registers EC2 instance IPs with NodePort

---

## Node Groups & Compute

### 7. What is Karpenter and how does it differ from Cluster Autoscaler?

**Answer:**

| Feature | Cluster Autoscaler | Karpenter |
|---------|-------------------|-----------|
| **Scaling unit** | Node group (ASG) | Individual nodes |
| **Instance selection** | Fixed per node group | Dynamic, right-sized per workload |
| **Speed** | Slower (ASG operations) | Fast (direct EC2 API) |
| **Configuration** | Multiple node groups for variety | Single NodePool with flexible constraints |
| **Consolidation** | Basic (scale-in) | Advanced (bin-packing, replacing underutilized nodes) |
| **Spot handling** | Per node group | Per NodePool, mixed |
| **AWS native** | Third-party (Kubernetes SIG) | AWS-built, K8s native |

```yaml
# Karpenter NodePool
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["m", "c", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "200"
    memory: 800Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

---

### 8. How do you manage EKS node AMI updates?

**Answer:**

**Managed Node Groups — AMI update:**
```bash
# Check current AMI version
aws eks describe-nodegroup --cluster-name my-cluster --nodegroup-name app-nodes \
  --query 'nodegroup.releaseVersion'

# Update to latest AMI
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name app-nodes \
  --launch-template name=my-lt,version='$Latest'

# The update performs a rolling replacement:
# 1. Launches new node with updated AMI
# 2. Cordons old node
# 3. Drains pods (respecting PDBs)
# 4. Terminates old node
```

**Karpenter — AMI update:**
```yaml
# EC2NodeClass — automatically uses latest EKS-optimized AMI
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
    - alias: al2023@latest   # Always picks latest AMI
  # Drift detection: Karpenter auto-replaces nodes when AMI changes
```

---

## EKS Add-ons

### 9. What are EKS add-ons and which ones are essential?

**Answer:** EKS add-ons are **operational software** managed by AWS that provide core cluster functionality.

| Add-on | Purpose | Required? |
|--------|---------|----------|
| **vpc-cni (aws-node)** | Pod networking — assigns VPC IPs to pods | Yes |
| **kube-proxy** | K8s Service networking (iptables/IPVS rules) | Yes |
| **CoreDNS** | Cluster DNS resolution | Yes |
| **EBS CSI Driver** | EBS volume provisioning for PVCs | Yes (if using EBS) |
| **EFS CSI Driver** | EFS filesystem mounting for shared PVCs | If using EFS |
| **AWS LB Controller** | Provisions ALB/NLB for Ingress/Services | Yes (for load balancers) |
| **Karpenter** | Node auto-provisioning | Recommended |
| **Cluster Autoscaler** | Node group scaling | If not using Karpenter |
| **Metrics Server** | Pod resource metrics (HPA) | Yes |
| **Pod Identity Agent** | IAM roles for pods (replaces IRSA) | Recommended (EKS 1.24+) |
| **Adot (OpenTelemetry)** | Observability data collection | Optional |
| **GuardDuty Agent** | Runtime threat detection | Recommended |
| **CloudWatch Observability** | Container Insights metrics/logs | Optional |

```bash
# List available add-ons
aws eks describe-addon-versions --kubernetes-version 1.30 \
  --query 'addons[*].addonName' --output table

# Install add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --addon-version v1.28.0-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::xxx:role/ebs-csi-role \
  --resolve-conflicts OVERWRITE

# Check add-on status
aws eks describe-addon --cluster-name my-cluster --addon-name aws-ebs-csi-driver

# Update add-on
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.16.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

---

### 10. How do you manage add-on versions and compatibility during upgrades?

**Answer:**

**Compatibility matrix:** Each EKS version supports specific add-on version ranges.

```bash
# Check compatible versions for an add-on + K8s version
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.30 \
  --query 'addons[0].addonVersions[*].{Version:addonVersion,Default:compatibilities[0].defaultVersion}' \
  --output table

# Check current add-on versions in cluster
aws eks list-addons --cluster-name my-cluster
for addon in $(aws eks list-addons --cluster-name my-cluster --query 'addons[]' --output text); do
  echo "$addon: $(aws eks describe-addon --cluster-name my-cluster --addon-name $addon --query 'addon.addonVersion' --output text)"
done
```

**Add-on update strategy during cluster upgrades:**
1. **Before upgrading control plane** — Verify add-on compatibility with target K8s version
2. **Upgrade control plane** first
3. **Update add-ons** one by one (vpc-cni → CoreDNS → kube-proxy → others)
4. **Update nodes** last

**Terraform management:**
```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "vpc-cni"
  addon_version               = "v1.16.0-eksbuild.1"
  resolve_conflicts_on_update = "OVERWRITE"
  service_account_role_arn    = aws_iam_role.vpc_cni.arn

  # Ensure control plane is upgraded before add-ons
  depends_on = [aws_eks_cluster.main]
}
```

---

### 11. What is the EBS CSI Driver and why is it needed as an add-on?

**Answer:** Since Kubernetes 1.23, the **in-tree EBS provisioner was deprecated**. The EBS CSI Driver (Container Storage Interface) is now required for dynamic EBS volume provisioning.

```yaml
# StorageClass using EBS CSI Driver
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com   # CSI driver (not kubernetes.io/aws-ebs)
parameters:
  type: gp3
  iops: "4000"
  throughput: "250"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:xxx:key/yyy
volumeBindingMode: WaitForFirstConsumer   # Bind to pod's AZ
allowVolumeExpansion: true
reclaimPolicy: Delete

# PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
```

**IAM requirement:** The EBS CSI Driver needs an IAM role (via IRSA or Pod Identity) with permissions to create/attach/delete EBS volumes.

---

### 12. What is the difference between IRSA (IAM Roles for Service Accounts) and EKS Pod Identity?

**Answer:**

| Feature | IRSA | EKS Pod Identity |
|---------|------|-----------------|
| **Mechanism** | OIDC provider + IAM trust policy per SA | Pod Identity Agent + IAM association |
| **Setup complexity** | High (OIDC provider, trust policy per role) | Low (one association per SA) |
| **Cross-account** | Complex trust policy | Simple |
| **Available since** | EKS 1.13 | EKS 1.24+ |
| **Session tags** | No | Yes (cluster name, namespace, SA) |
| **Recommended** | Legacy/existing setups | New setups |

```bash
# EKS Pod Identity — simpler setup
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace myapp \
  --service-account myapp-sa \
  --role-arn arn:aws:iam::xxx:role/myapp-role

# IAM role trust policy for Pod Identity is generic (same for all clusters):
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "pods.eks.amazonaws.com" },
    "Action": ["sts:AssumeRole", "sts:TagSession"]
  }]
}
```

---

## EKS Upgrades

### 13. How does the EKS upgrade process work end-to-end?

**Answer:** EKS clusters must be upgraded **one minor version at a time** (e.g., 1.28 → 1.29 → 1.30). AWS supports 4 versions concurrently and provides ~14 months of support per version.

**Upgrade order:**
```
1. Pre-upgrade checks
2. Upgrade control plane (API server, etcd, controllers)
3. Update EKS add-ons (vpc-cni, CoreDNS, kube-proxy, EBS CSI, etc.)
4. Upgrade nodes (managed node groups, self-managed, or Karpenter)
5. Post-upgrade validation
```

```bash
# Step 1: Pre-upgrade checks
# Check deprecated APIs
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Check PSP usage (removed in 1.25)
kubectl get psp

# Check add-on compatibility
aws eks describe-addon-versions --kubernetes-version 1.30

# Step 2: Upgrade control plane
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.30
# Takes 20-40 minutes, NO DOWNTIME for running pods

# Step 3: Update add-ons
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni --addon-version v1.16.0-eksbuild.1
aws eks update-addon --cluster-name my-cluster --addon-name coredns --addon-version v1.11.1-eksbuild.4
aws eks update-addon --cluster-name my-cluster --addon-name kube-proxy --addon-version v1.30.0-eksbuild.3

# Step 4: Upgrade nodes
aws eks update-nodegroup-version --cluster-name my-cluster --nodegroup-name app-nodes
# Rolling replacement: new nodes launch → old nodes drain → old nodes terminate
```

---

### 14. What are the critical pre-upgrade checks before upgrading EKS?

**Answer:**

| Check | Why | How |
|-------|-----|-----|
| **Deprecated APIs** | Removed APIs break workloads | `kubectl get --raw /metrics \| grep deprecated` or use `pluto` |
| **PSP → PSS migration** | PodSecurityPolicy removed in 1.25 | Migrate to Pod Security Standards |
| **Add-on compatibility** | Add-on may not support new K8s version | `aws eks describe-addon-versions` |
| **Helm chart compatibility** | Charts may use old API versions | `helm template` + `pluto` scan |
| **PDB configuration** | Aggressive PDBs block node draining | Check `PodDisruptionBudgets` |
| **Node AMI availability** | New AMI may not be available yet | Check EKS AMI release notes |
| **Third-party operators** | cert-manager, Istio, ArgoCD compatibility | Check operator docs |
| **Webhook compatibility** | Admission webhooks may block new resources | Review `ValidatingWebhookConfiguration` |
| **Storage classes** | In-tree → CSI migration | Ensure EBS/EFS CSI driver is installed |

```bash
# Use pluto to scan for deprecated APIs
pluto detect-all-in-cluster

# Check for PDBs that might block draining
kubectl get pdb -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,MIN:.spec.minAvailable,MAX:.spec.maxUnavailable'

# Verify webhook configurations
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

---

### 15. What are common issues during EKS upgrades and how do you handle them?

**Answer:**

| Issue | Cause | Fix |
|-------|-------|-----|
| **Control plane upgrade stuck** | Usually just slow (20-40 min) | Wait; check EKS console for status |
| **Nodes won't drain** | PDB blocking eviction | Adjust PDB `maxUnavailable`, check pending pods |
| **Pods stuck in Pending** | New node doesn't have enough resources or taints | Check node taints/labels, pod tolerations |
| **CoreDNS not resolving** | Version incompatible after upgrade | Update CoreDNS add-on |
| **Webhooks failing** | Webhook service not ready on new nodes | Add `failurePolicy: Ignore` temporarily, or upgrade webhook first |
| **ALB not routing** | AWS LB Controller incompatible | Update LB Controller before upgrading nodes |
| **CrashLoopBackOff** | Application incompatible with new K8s version | Check container runtime changes, API deprecations |
| **vpc-cni issues** | Plugin version mismatch | Update vpc-cni add-on first |
| **HPA not working** | Metrics Server version | Update Metrics Server |

**Rollback strategy:**
- **Control plane** — Cannot be rolled back; AWS only supports forward upgrades
- **Nodes** — Keep old node group running, drain new nodes if issues found
- **Add-ons** — Can be rolled back to previous version
- **Applications** — Use ArgoCD/Helm to rollback to previous working version

---

### 16. How do you plan an EKS upgrade from 1.27 to 1.30 across multiple clusters?

**Answer:**

**Phase 1: Planning (2-4 weeks before)**
```
1. Read K8s changelogs for 1.28, 1.29, 1.30
2. Run pluto/kubent against all clusters for deprecated APIs
3. Build add-on compatibility matrix
4. Test upgrade in dev cluster
5. Update Terraform/Helm charts for API changes
6. Notify teams of timeline and potential impact
```

**Phase 2: Upgrade per cluster (1.27 → 1.28 → 1.29 → 1.30)**
```
For each minor version:
  1. Upgrade control plane
  2. Wait for control plane to be ACTIVE
  3. Update add-ons (vpc-cni → CoreDNS → kube-proxy → EBS CSI → others)
  4. Upgrade node groups (rolling update)
  5. Validate:
     - kubectl get nodes (all Ready, correct version)
     - kubectl get pods -A (no CrashLoopBackOff)
     - Application health checks passing
     - Monitoring dashboards normal
  6. Wait 24-48 hours before next minor version
```

**Phase 3: Rollout order**
```
Dev cluster → Staging cluster → Prod (canary) → Prod (remaining)
     1 day gap      2 day gap        1 week gap
```

---

### 17. How do you handle the PSP to Pod Security Standards migration during upgrades?

**Answer:** PodSecurityPolicy (PSP) was removed in Kubernetes 1.25. Migration to **Pod Security Standards (PSS)** with **Pod Security Admission (PSA)** is required.

**PSS levels:**

| Level | What it allows |
|-------|---------------|
| **Privileged** | Unrestricted (equivalent to no PSP) |
| **Baseline** | Prevents known privilege escalations |
| **Restricted** | Heavily restricted, best practices |

```yaml
# Label namespaces with PSS enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted

# System namespaces that need privileged:
# kube-system, monitoring, logging (DaemonSets need host access)
```

**Migration steps:**
1. **Audit** — Label namespaces with `audit: restricted` to see violations in logs
2. **Warn** — Label with `warn: baseline` to surface warnings
3. **Fix** — Update workloads to comply
4. **Enforce** — Label with `enforce: baseline` or `enforce: restricted`
5. **Remove PSPs** after upgrading past 1.25

---

## Security

### 18. How does EKS authentication and authorization work?

**Answer:**

**Authentication:** Who are you?
```
kubectl request → EKS API Server → AWS IAM (via aws-auth ConfigMap or Access Entries)
```

**Authorization:** What can you do?
```
Authenticated identity → Kubernetes RBAC → Allow/Deny
```

**Two models:**

| Model | How it works | Recommended |
|-------|-------------|-------------|
| **aws-auth ConfigMap** | Maps IAM roles/users to K8s groups | Legacy |
| **EKS Access Entries** | AWS API to manage access (no ConfigMap) | New clusters |

```bash
# EKS Access Entries (recommended)
aws eks create-access-entry \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::xxx:role/developer-role \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name my-cluster \
  --principal-arn arn:aws:iam::xxx:role/developer-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=myapp
```

```yaml
# aws-auth ConfigMap (legacy)
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::xxx:role/eks-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups: ["system:bootstrappers", "system:nodes"]
    - rolearn: arn:aws:iam::xxx:role/developer-role
      username: developer
      groups: ["dev-group"]
```

---

### 19. How do you implement network policies in EKS?

**Answer:** EKS VPC CNI supports **native network policies** (since VPC CNI v1.14+) without requiring Calico.

```yaml
# Enable network policy in VPC CNI
# Set ENABLE_NETWORK_POLICY=true on aws-node DaemonSet

# Deny all ingress by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: myapp
spec:
  podSelector: {}
  policyTypes: ["Ingress"]

# Allow only from specific namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend
        - podSelector:
            matchLabels:
              app: web
      ports:
        - port: 8080
          protocol: TCP
```

---

## Scenario-Based Questions

### 20. Scenario: Pods are stuck in Pending state after deploying to EKS. How do you troubleshoot?

**Answer:**

```bash
# 1. Check pod events
kubectl describe pod <pod-name> -n <namespace>
# Look for: FailedScheduling, Insufficient cpu/memory, node affinity mismatch

# 2. Common causes and fixes:
```

| Cause | Symptom in events | Fix |
|-------|-------------------|-----|
| **Insufficient resources** | "Insufficient cpu" or "Insufficient memory" | Scale up nodes or add Karpenter |
| **Node selector/affinity** | "no nodes match" | Check labels, taints, tolerations |
| **PVC pending** | "waiting for volume" | Check StorageClass, EBS CSI driver |
| **IP exhaustion** | "failed to assign IP" | Enable prefix delegation or add subnets |
| **Max pods reached** | "Too many pods" | Larger instance type or prefix delegation |
| **Taints** | "node has taint that pod doesn't tolerate" | Add toleration or remove taint |

---

### 21. Scenario: Your EKS cluster nodes are frequently running out of IP addresses. How do you solve this?

**Answer:**

| Solution | Impact | Effort |
|----------|--------|--------|
| **Enable prefix delegation** | 16x more IPs per ENI slot | Low — env var change |
| **Custom networking** | Pods use secondary CIDR | Medium |
| **Add secondary CIDR to VPC** | More subnet space | Low |
| **Use smaller pod CIDR** | Careful subnet planning | Medium |
| **Larger instance types** | More ENIs + IPs | Cost increase |

```bash
# Enable prefix delegation (quickest fix)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1

# Verify
kubectl get nodes -o custom-columns='NAME:.metadata.name,PODS:.status.allocatable.pods'
```

---

### 22. Scenario: A deployment rollout is stuck and pods keep restarting (CrashLoopBackOff). How do you debug?

**Answer:**

```bash
# 1. Check pod status and restart count
kubectl get pods -n myapp -l app=myapp

# 2. Check pod events and last state
kubectl describe pod <pod> -n myapp

# 3. Check current logs
kubectl logs <pod> -n myapp --tail=100

# 4. Check previous container logs (crashed container)
kubectl logs <pod> -n myapp --previous

# 5. Common CrashLoopBackOff causes:
```

| Cause | How to identify | Fix |
|-------|----------------|-----|
| **Config error** | Logs show config parse error | Fix ConfigMap/Secret |
| **Missing env var** | "KeyError" or "undefined" in logs | Check env vars, Secrets |
| **DB connection failure** | Connection timeout in logs | Check SG, endpoint, credentials |
| **OOMKilled** | `kubectl describe` shows OOMKilled | Increase memory limit |
| **Readiness probe fail** | Pod killed before ready | Fix probe path/port/timeout |
| **Image pull error** | ImagePullBackOff | Check ECR permissions, image tag |
| **Permission denied** | File/port permission error | Check securityContext, runAsUser |

```bash
# 6. Debug with ephemeral container or exec
kubectl debug -it <pod> -n myapp --image=busybox -- sh
kubectl exec -it <pod> -n myapp -- sh
```

---

### 23. Scenario: How do you perform a zero-downtime node group migration (e.g., x86 to Graviton)?

**Answer:**

```bash
# Strategy: Create new node group → migrate workloads → delete old node group

# 1. Create new Graviton node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name app-nodes-graviton \
  --instance-types m7g.xlarge c7g.xlarge r7g.xlarge \
  --ami-type AL2_ARM_64 \
  ...

# 2. Wait for nodes to be Ready
kubectl get nodes -l eks.amazonaws.com/nodegroup=app-nodes-graviton

# 3. Cordon old nodes (prevent new pods from scheduling)
kubectl cordon <old-node-1>
kubectl cordon <old-node-2>

# 4. Drain old nodes (evict pods, respects PDBs)
kubectl drain <old-node-1> --ignore-daemonsets --delete-emptydir-data
kubectl drain <old-node-2> --ignore-daemonsets --delete-emptydir-data

# 5. Pods reschedule on new Graviton nodes
# 6. Verify all workloads healthy

# 7. Delete old node group
aws eks delete-nodegroup --cluster-name my-cluster --nodegroup-name app-nodes-old
```

**Important:** Ensure container images are **multi-arch** (amd64 + arm64) before migrating.

---

### 24. Scenario: EKS add-on update fails with conflict errors. How do you resolve it?

**Answer:**

```bash
# Check add-on status
aws eks describe-addon --cluster-name my-cluster --addon-name vpc-cni

# Common conflict: someone manually edited the add-on resources (kubectl edit)
# EKS add-on management conflicts with manual changes

# Resolution options:

# Option 1: OVERWRITE — EKS replaces manual changes
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.16.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE

# Option 2: PRESERVE — Keep manual changes, only update non-conflicting fields
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.16.0-eksbuild.1 \
  --resolve-conflicts PRESERVE

# Option 3: Use configuration-values for custom settings
aws eks update-addon \
  --cluster-name my-cluster \
  --addon-name vpc-cni \
  --addon-version v1.16.0-eksbuild.1 \
  --configuration-values '{"env":{"ENABLE_PREFIX_DELEGATION":"true","WARM_PREFIX_TARGET":"1"}}' \
  --resolve-conflicts OVERWRITE
```

**Best practice:** Never `kubectl edit` EKS-managed add-on resources. Use `--configuration-values` for customizations so EKS tracks them.

---

### 25. Scenario: How do you set up EKS for production with best practices?

**Answer:**

**Cluster config:**
- Private API endpoint (or public with restricted CIDRs)
- Envelope encryption for Secrets (KMS)
- Enable control plane logging (API, audit, authenticator)
- Use EKS Access Entries (not aws-auth)

**Networking:**
- Private subnets for nodes, public for ALB/NLB
- VPC endpoints for ECR, S3, STS, EC2, CloudWatch
- Network policies (VPC CNI native or Calico)
- Security Groups for Pods (granular pod-level SG)

**Compute:**
- Karpenter for auto-provisioning (or managed node groups)
- Graviton instances for cost savings
- Spot for non-critical workloads
- Pod Disruption Budgets on all production workloads

**Add-ons:**
- Managed add-ons via Terraform (vpc-cni, CoreDNS, kube-proxy, EBS CSI)
- AWS LB Controller for ingress
- External Secrets Operator or Pod Identity for secrets
- Metrics Server for HPA

**Security:**
- Pod Security Standards (baseline or restricted)
- IRSA or Pod Identity (no node-level IAM permissions for apps)
- GuardDuty EKS Runtime Monitoring
- Image scanning (ECR scan on push)
- OPA/Gatekeeper or Kyverno for policy enforcement

**Observability:**
- Prometheus + Grafana (or Datadog) for metrics
- Fluent Bit → Loki/OpenSearch for logs
- Distributed tracing (X-Ray, Tempo, or Datadog APM)
- Alerts on: pod restarts, node NotReady, HPA at max, error rate spikes
