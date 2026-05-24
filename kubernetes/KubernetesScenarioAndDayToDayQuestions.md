# Kubernetes Scenario-Based & Day-to-Day Interview Questions

Real-world troubleshooting scenarios, day-to-day operational tasks, and practical problem-solving questions for Kubernetes interviews — focused on what DevOps engineers actually face in production.

---

## Table of Contents

- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [Day-to-Day Operations Questions](#day-to-day-operations-questions)
- [Deployment & Release Scenarios](#deployment--release-scenarios)
- [Networking & Service Scenarios](#networking--service-scenarios)
- [Security & Access Control Scenarios](#security--access-control-scenarios)
- [Scaling & Performance Scenarios](#scaling--performance-scenarios)
- [Cluster Administration Scenarios](#cluster-administration-scenarios)

---

## Troubleshooting Scenarios

### 1. A Pod is stuck in CrashLoopBackOff. How do you troubleshoot?

**Answer:** CrashLoopBackOff means the container is repeatedly crashing and restarting.

**Step-by-step troubleshooting:**

```bash
# 1. Describe the Pod — check Events, exit code, last state
kubectl describe pod <pod-name>

# 2. Check logs from the previous (crashed) container instance
kubectl logs <pod-name> --previous

# 3. Check cluster events sorted by time
kubectl get events --sort-by='.lastTimestamp' -n <namespace>

# 4. Check the exit code
#    Exit code 1   → Application error (bad config, missing file, runtime error)
#    Exit code 137  → OOMKilled (memory limit exceeded)
#    Exit code 139  → Segmentation fault
#    Exit code 143  → SIGTERM (graceful shutdown failed)
```

**Common causes and fixes:**

| Cause | Symptom | Fix |
|-------|---------|-----|
| Missing env vars / config | App startup error in logs | Fix ConfigMap/Secret references |
| Memory limit too low | Exit code 137 (OOMKilled) | Increase `resources.limits.memory` |
| Wrong command/args | Container exits immediately | Fix `command` and `args` in spec |
| Failed liveness probe | Healthy app gets killed | Adjust `initialDelaySeconds`, `timeoutSeconds` |
| Image issues | Wrong entrypoint or missing binary | Verify image and tag |
| Dependency not available | Connection refused / timeout | Add init container to wait for dependency |

---

### 2. A Pod is stuck in Pending state. What are the possible reasons?

**Answer:**

```bash
# Check why the Pod is pending
kubectl describe pod <pod-name>
# Look at the "Events" section for scheduling failure messages
```

**Common causes:**

| Cause | Event Message | Fix |
|-------|--------------|-----|
| Insufficient CPU/memory | "Insufficient cpu" or "Insufficient memory" | Reduce requests, add nodes, or use cluster autoscaler |
| No matching node selector | "didn't match Pod's node affinity/selector" | Fix `nodeSelector` labels or add labels to nodes |
| PVC not bound | "persistentvolumeclaim not found" | Create PV or fix StorageClass |
| Taint not tolerated | "node(s) had taint that the pod didn't tolerate" | Add toleration to Pod spec |
| Image pull issues | "ImagePullBackOff" | Fix image name, add `imagePullSecret` |
| Scheduler not running | No events at all | Check kube-scheduler health |
| Resource quota exceeded | "exceeded quota" | Request quota increase or free resources |

---

### 3. You're getting 503 errors when hitting a LoadBalancer URL. How do you troubleshoot?

**Answer:** Follow a systematic approach from the load balancer down to the Pod:

```bash
# 1. Check if service has endpoints (Pods backing it)
kubectl get endpoints <service-name>
# Empty endpoints = service selector doesn't match any pod labels

# 2. Check Pod health
kubectl get pods -l app=<app-label>
# Are Pods Running? Are they Ready (1/1)?

# 3. Check service port mapping
kubectl describe service <service-name>
# Verify: Port, TargetPort, Selector match Pod labels and container port

# 4. Test internal connectivity
kubectl exec -it <pod-name> -- curl localhost:<container-port>
# If this fails, the app itself is down

# 5. Test service connectivity from within cluster
kubectl run test --rm -it --image=busybox -- wget -qO- http://<service-name>:<port>

# 6. Check Ingress / LB configuration
kubectl describe ingress <ingress-name>
# Verify backend service name and port

# 7. Check readiness probes — failing probes remove Pods from endpoints
kubectl describe pod <pod-name> | grep -A5 "Readiness"
```

---

### 4. How do you handle an ImagePullBackOff error?

**Answer:**

```bash
# 1. Describe the Pod and check Events section
kubectl describe pod <pod-name>

# 2. Verify image name — check for typos in image name or tag
spec:
  containers:
    - image: my-registry.com/myapp:v1.2.3   # Correct?

# 3. Check if node can reach the registry
kubectl exec -it <another-running-pod> -- nslookup my-registry.com

# 4. For private registries — verify imagePullSecret exists and is correct
kubectl get secret my-registry-secret -o yaml
kubectl get pods <pod-name> -o jsonpath='{.spec.imagePullSecrets}'

# 5. Create imagePullSecret if missing
kubectl create secret docker-registry my-registry-secret \
  --docker-server=my-registry.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com
```

---

### 5. A Pod is running but the application is slow. How do you debug?

**Answer:**

```bash
# 1. Check Pod resource utilization
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# 2. Describe the Pod — look for resource throttling, restarts, probe failures
kubectl describe pod <pod-name>

# 3. Check container logs for errors, timeouts, connection failures
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>   # Multi-container Pod

# 4. Test network latency from inside the Pod
kubectl exec -it <pod-name> -- ping my-database
kubectl exec -it <pod-name> -- curl -w "Total: %{time_total}s\n" http://my-service

# 5. Check node health — resource exhaustion on the node
kubectl get nodes
kubectl describe node <node-name>
kubectl top nodes

# 6. Check if HPA is struggling to scale
kubectl get hpa
# If current replicas = max replicas, the app may need more headroom

# 7. Check for noisy neighbors on the same node
kubectl get pods -o wide --field-selector spec.nodeName=<node-name>
```

---

### 6. Pods are being OOMKilled frequently. What do you do?

**Answer:**

```bash
# 1. Confirm OOMKill
kubectl describe pod <pod-name>
# Look for: "Last State: Terminated, Reason: OOMKilled, Exit Code: 137"

# 2. Check current memory usage vs limits
kubectl top pod <pod-name>
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].resources}'

# 3. Options:
```

| Option | When to Use |
|--------|-------------|
| Increase memory limit | App genuinely needs more memory |
| Fix memory leak in app | Usage grows unbounded over time |
| Use VPA (Vertical Pod Autoscaler) | Auto-tune requests/limits based on usage |
| Profile the application | Identify which component consumes memory |
| Add JVM flags (Java apps) | `-Xmx` to cap heap: `java -Xmx512m` |

```yaml
# Increase memory limit
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "1Gi"    # Give headroom above request
```

---

### 7. CoreDNS is not resolving service names. How do you troubleshoot?

**Answer:**

```bash
# 1. Check if CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# 3. Test DNS from inside a pod
kubectl exec -it <pod-name> -- nslookup kubernetes.default
kubectl exec -it <pod-name> -- nslookup <service-name>.<namespace>.svc.cluster.local

# 4. Check if the Service exists and has endpoints
kubectl get svc <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>

# 5. Check CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml

# 6. Check resolv.conf inside the Pod
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
# Should point to CoreDNS ClusterIP (usually 10.96.0.10)
```

---

### 8. Nodes are showing NotReady status. What do you check?

**Answer:**

```bash
# 1. Get node status
kubectl get nodes
kubectl describe node <node-name>
# Check "Conditions" section — MemoryPressure, DiskPressure, PIDPressure

# 2. Check kubelet status on the node (SSH in)
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100

# 3. Check container runtime
sudo systemctl status containerd   # or docker/cri-o
sudo crictl ps

# 4. Check disk space
df -h

# 5. Check system resources
free -h
top

# 6. Common causes:
```

| Cause | Fix |
|-------|-----|
| Kubelet crashed | `sudo systemctl restart kubelet` |
| Certificate expired | Renew certs: `kubeadm certs renew all` |
| Disk full | Clean up logs, images: `crictl rmi --prune` |
| Network issue | Check CNI plugin, firewall rules |
| OOM on node | Free memory, set resource limits on Pods |

---

## Day-to-Day Operations Questions

### 9. How do you check resource usage across the cluster?

**Answer:**

```bash
# Node-level resource usage
kubectl top nodes

# Pod-level resource usage (sorted)
kubectl top pods --sort-by=cpu --all-namespaces
kubectl top pods --sort-by=memory -n <namespace>

# Check resource requests vs capacity
kubectl describe nodes | grep -A5 "Allocated resources"

# List pods consuming the most resources
kubectl top pods -A --sort-by=cpu | head -20

# Check resource quotas
kubectl get resourcequotas --all-namespaces
```

---

### 10. How do you view logs for a Pod, including crashed containers?

**Answer:**

```bash
# Current container logs
kubectl logs <pod-name>

# Previous (crashed) container logs
kubectl logs <pod-name> --previous

# Follow logs in real-time
kubectl logs <pod-name> -f

# Specific container in a multi-container Pod
kubectl logs <pod-name> -c <container-name>

# Last N lines
kubectl logs <pod-name> --tail=100

# Logs since a time duration
kubectl logs <pod-name> --since=1h

# All pods matching a label
kubectl logs -l app=my-app --all-containers

# Init container logs
kubectl logs <pod-name> -c <init-container-name>
```

---

### 11. How do you exec into a running container for debugging?

**Answer:**

```bash
# Shell into a container
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash

# Specific container in multi-container Pod
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Run a single command
kubectl exec <pod-name> -- env
kubectl exec <pod-name> -- cat /etc/config/app.conf

# If no shell in the image, use ephemeral debug containers (K8s 1.23+)
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```

---

### 12. How do you scale a Deployment?

**Answer:**

```bash
# Manual scaling
kubectl scale deployment my-app --replicas=5

# Auto-scaling with HPA
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=70

# Check current scale
kubectl get deployment my-app
kubectl get hpa my-app

# Scale to zero (stop all pods)
kubectl scale deployment my-app --replicas=0
```

---

### 13. How do you perform a rolling update and rollback?

**Answer:**

```bash
# Trigger rolling update by changing image
kubectl set image deployment/my-app my-app=my-app:v2.0

# Or edit the deployment directly
kubectl edit deployment my-app

# Monitor rollout progress
kubectl rollout status deployment/my-app

# View rollout history
kubectl rollout history deployment/my-app
kubectl rollout history deployment/my-app --revision=2

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=3

# Pause/resume a rollout (for canary-style)
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app
```

---

### 14. How do you drain a node for maintenance?

**Answer:**

```bash
# 1. Cordon the node (prevent new pods from being scheduled)
kubectl cordon <node-name>

# 2. Drain the node (evict existing pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 3. Perform maintenance (upgrade, patching, etc.)

# 4. Uncordon the node (allow scheduling again)
kubectl uncordon <node-name>

# 5. Verify node status
kubectl get nodes
```

**Important flags:**
- `--ignore-daemonsets` — DaemonSet pods can't be evicted, so ignore them
- `--delete-emptydir-data` — Delete pods using emptyDir volumes (data will be lost)
- `--force` — Force eviction of unmanaged pods
- `--grace-period=30` — Override default grace period

---

### 15. How do you manage Secrets in day-to-day operations?

**Answer:**

```bash
# Create a Secret from literal values
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=secret123

# Create from file
kubectl create secret generic tls-cert \
  --from-file=cert.pem \
  --from-file=key.pem

# View Secret (base64 encoded)
kubectl get secret db-creds -o yaml

# Decode a Secret value
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d

# Edit a Secret
kubectl edit secret db-creds

# Use in a Pod as env variable
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: password

# Use as mounted file
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
    readOnly: true
volumes:
  - name: secret-volume
    secret:
      secretName: db-creds
```

---

### 16. How do you troubleshoot inter-pod networking issues?

**Answer:**

```bash
# 1. Check if pods can reach each other
kubectl exec -it <pod-a> -- ping <pod-b-ip>
kubectl exec -it <pod-a> -- curl http://<service-name>:<port>

# 2. Check DNS resolution
kubectl exec -it <pod-a> -- nslookup <service-name>
kubectl exec -it <pod-a> -- nslookup <service-name>.<namespace>.svc.cluster.local

# 3. Check Network Policies
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <policy-name>

# 4. Check CNI plugin
kubectl get pods -n kube-system | grep -i calico  # or flannel, cilium

# 5. Check service endpoints
kubectl get endpoints <service-name>

# 6. Deploy a debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -- /bin/bash
# Inside: tcpdump, dig, nslookup, curl, ping, traceroute all available
```

---

### 17. How do you check what's consuming resources in a namespace?

**Answer:**

```bash
# Pod resource usage
kubectl top pods -n <namespace> --sort-by=cpu
kubectl top pods -n <namespace> --sort-by=memory

# Resource quota usage
kubectl describe resourcequota -n <namespace>

# List all resources in a namespace
kubectl get all -n <namespace>

# Count objects
kubectl get pods -n <namespace> --no-headers | wc -l
kubectl get deployments -n <namespace> --no-headers | wc -l

# PVC storage usage
kubectl get pvc -n <namespace>

# Events (recent activity)
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
```

---

### 18. How do you copy files to/from a Pod?

**Answer:**

```bash
# Copy file from local to Pod
kubectl cp ./local-file.txt <pod-name>:/tmp/remote-file.txt

# Copy file from Pod to local
kubectl cp <pod-name>:/var/log/app.log ./app.log

# Specific container in multi-container Pod
kubectl cp <pod-name>:/tmp/file.txt ./file.txt -c <container-name>

# Copy entire directory
kubectl cp ./local-dir <pod-name>:/tmp/remote-dir

# Specific namespace
kubectl cp <namespace>/<pod-name>:/tmp/file.txt ./file.txt
```

---

## Deployment & Release Scenarios

### 19. Your team deployed a new version and it's failing. Users are experiencing downtime. What do you do?

**Answer:**

```bash
# 1. IMMEDIATE: Rollback to the previous working version
kubectl rollout undo deployment my-app

# 2. Verify rollback succeeded
kubectl rollout status deployment/my-app
kubectl get pods -l app=my-app

# 3. Investigate what went wrong
kubectl rollout history deployment my-app
kubectl logs -l app=my-app --previous

# 4. Check the new image for issues
kubectl describe deployment my-app
# Look at the image tag, env vars, config changes

# 5. Fix the issue in the new version
# 6. Test in staging first
# 7. Re-deploy with fix
```

**Prevention:**
- Always use readiness probes
- Set `maxUnavailable: 0` in rolling update strategy
- Use Pod Disruption Budgets
- Test in staging before production
- Use canary deployments for critical services

---

### 20. How would you implement a canary deployment in Kubernetes?

**Answer:**

**Method 1: Using multiple Deployments (native)**
```yaml
# Stable deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      version: stable
  template:
    metadata:
      labels:
        app: my-app
        version: stable
    spec:
      containers:
        - name: my-app
          image: my-app:v1.0

---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: canary
  template:
    metadata:
      labels:
        app: my-app
        version: canary
    spec:
      containers:
        - name: my-app
          image: my-app:v2.0

---
# Service selects both (app: my-app) — traffic split by replica ratio
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app     # Matches both stable and canary
  ports:
    - port: 80
      targetPort: 8080
```

**Method 2: Using Istio / Service Mesh (precise traffic splitting)**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - my-app
  http:
    - route:
        - destination:
            host: my-app
            subset: stable
          weight: 90
        - destination:
            host: my-app
            subset: canary
          weight: 10
```

**Method 3: Using Argo Rollouts (purpose-built)**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 30
        - pause: {duration: 5m}
        - setWeight: 60
        - pause: {duration: 5m}
```

---

### 21. How do you run a one-time database migration before your application starts?

**Answer:** Use **init containers**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      initContainers:
        - name: run-migration
          image: myapp:migration
          command: ['sh', '-c', 'run-migration.sh']
          env:
            - name: DB_HOST
              value: "postgres-service"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
      containers:
        - name: app
          image: myapp:latest
```

The init container runs and completes **before** the main container starts. If the migration fails, the Pod stays in `Init:Error` state and the app never starts — preventing a broken app from serving traffic.

---

### 22. How do you ensure a specific number of Pods are always available during a rolling update?

**Answer:** Use a combination of rolling update strategy and Pod Disruption Budget:

```yaml
# Deployment strategy
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # At most 1 extra Pod during update
      maxUnavailable: 0    # Zero downtime — never remove a Pod before new one is Ready

---
# Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 3          # At least 3 Pods must always be available
  selector:
    matchLabels:
      app: my-app
```

---

## Networking & Service Scenarios

### 23. An Nginx web server is running, but the exposed URL fails to connect. How do you troubleshoot?

**Answer:**

```bash
# 1. Verify Pod is running and healthy
kubectl get pods -o wide
kubectl describe pod nginx-web

# 2. Check Service port mapping — does TargetPort match container port?
kubectl describe service nginx-service
# Verify: Port, TargetPort, Selector

# 3. Check endpoints — does the Service find the Pods?
kubectl get endpoints nginx-service
# Empty = selector labels don't match Pod labels

# 4. Check Network Policies blocking ingress traffic
kubectl get networkpolicies
kubectl describe networkpolicy <policy-name>

# 5. Verify Ingress configuration
kubectl describe ingress nginx-ingress
# Check: host, path, backend service name and port

# 6. Test connectivity from inside the cluster
kubectl run test --rm -it --image=busybox -- wget -qO- http://nginx-service

# 7. Check if the container is actually listening
kubectl exec -it <nginx-pod> -- curl localhost:80
```

---

### 24. How can you ensure only specific Pods can communicate with your backend service?

**Answer:** Use a **Network Policy**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-access-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access-backend: "true"
      ports:
        - protocol: TCP
          port: 8080
```

Only Pods with label `access-backend: "true"` can reach the backend Pods on port 8080. All other ingress traffic is denied.

**Important:** Network Policies require a CNI that supports them (Calico, Cilium). Flannel does NOT support Network Policies.

---

### 25. Application can't connect to an external database outside the cluster. How do you fix it?

**Answer:**

```bash
# 1. Test connectivity from inside a Pod
kubectl exec -it <pod-name> -- curl http://my-database.example.com:5432

# 2. Check DNS resolution
kubectl exec -it <pod-name> -- nslookup my-database.example.com
# If DNS fails → CoreDNS misconfiguration

# 3. Check Network Policies blocking egress
kubectl get networkpolicies -n <namespace>
# A deny-all egress policy would block external connections

# 4. Check if ExternalName Service is configured correctly
kubectl get svc -n <namespace>

# 5. Verify firewall/security groups allow outbound from nodes
# Cloud provider → check VPC/subnet route tables and security groups

# 6. Use ExternalName service for clean access
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: my-database.example.com
```

---

### 26. What happens if the firewall between master and worker nodes breaks?

**Answer:**

**Impact:**
- API server becomes inaccessible to kubelets
- New Pod scheduling stops
- Service discovery fails
- Existing Pods may continue running but can't be managed
- No new deployments, scaling, or updates possible

**Recovery steps:**

```bash
# 1. Restore firewall rules for required ports:
#    - 6443:  API server
#    - 10250: Kubelet
#    - 2379-2380: etcd
#    - 30000-32767: NodePort range

# 2. Check component health
kubectl get componentstatuses    # Deprecated but quick check
kubectl get nodes

# 3. Restart kubelet on affected nodes
sudo systemctl restart kubelet

# 4. Verify API server is accessible
curl -k https://<master-ip>:6443/healthz

# 5. Test Pod creation and service connectivity
kubectl run test --rm -it --image=busybox -- echo "Cluster is healthy"
```

---

## Security & Access Control Scenarios

### 27. You need to give a developer read-only access to view logs only. How do you set up RBAC?

**Answer:**

```yaml
# ClusterRole — read-only log access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]

---
# ClusterRoleBinding — bind to user
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: log-reader-binding
subjects:
  - kind: User
    name: developer1
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: log-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Verify permissions
kubectl auth can-i get pods --as developer1
kubectl auth can-i get pods/log --as developer1
kubectl auth can-i delete pods --as developer1   # Should return "no"
```

For namespace-scoped access, use `Role` and `RoleBinding` instead of `ClusterRole` and `ClusterRoleBinding`.

---

### 28. How do you prevent Pods from running as root in your cluster?

**Answer:** Use **Pod Security Admission** (PSA) — the replacement for Pod Security Policies:

```yaml
# Label the namespace to enforce restricted security standard
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

Or use a **Kyverno** policy for more flexibility:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-non-root
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Running as root is not allowed."
        pattern:
          spec:
            containers:
              - securityContext:
                  runAsNonRoot: true
```

---

### 29. A service account token was leaked. What do you do?

**Answer:**

```bash
# 1. IMMEDIATE: Delete the compromised service account
kubectl delete serviceaccount <compromised-sa> -n <namespace>

# 2. Create a new service account
kubectl create serviceaccount <new-sa> -n <namespace>

# 3. Check what the SA had access to
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.subjects[]?.name == "<compromised-sa>")'

# 4. Check audit logs for unauthorized activity
# Review API server audit logs for requests using the compromised token

# 5. Recreate any secrets associated with the SA
kubectl delete secrets -l kubernetes.io/service-account.name=<compromised-sa>

# 6. Update deployments to use new SA
kubectl patch deployment <name> -n <namespace> \
  -p '{"spec":{"template":{"spec":{"serviceAccountName":"<new-sa>"}}}}'

# 7. Review and tighten RBAC — principle of least privilege
```

---

### 30. How do you manage resource quotas in a multi-tenant cluster?

**Answer:**

```yaml
# Resource Quota per team namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "10"
    persistentvolumeclaims: "20"

---
# LimitRange — default limits for pods without explicit resource definitions
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "250m"
        memory: "256Mi"
      type: Container
```

```bash
# Check quota usage
kubectl describe resourcequota team-a-quota -n team-a
```

---

## Scaling & Performance Scenarios

### 31. Cluster resources are exhausted. New Pods remain in Pending state. What do you do?

**Answer:**

```bash
# 1. Check which pods are pending and why
kubectl get pods --field-selector=status.phase=Pending -A
kubectl describe pod <pending-pod>

# 2. Check node capacity vs allocated
kubectl describe nodes | grep -A10 "Allocated resources"

# 3. Options:
```

| Solution | When to Use |
|----------|-------------|
| **Scale up nodes** | Cloud: add nodes to node pool / ASG |
| **Enable Cluster Autoscaler** | Automatically add/remove nodes based on demand |
| **Optimize resource requests** | Pods may be over-requesting CPU/memory |
| **Clean up unused resources** | Delete completed Jobs, old ReplicaSets, unused PVCs |
| **Use priorities** | Set PriorityClasses so critical Pods preempt less important ones |
| **Bin-pack better** | Use `LimitRange` to set defaults, prevent over-requesting |

```yaml
# PriorityClass for critical workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000
globalDefault: false
description: "For critical production workloads"
```

---

### 32. How do you ensure high availability for your application in Kubernetes?

**Answer:**

```yaml
# 1. Multiple replicas
spec:
  replicas: 3

# 2. Pod Anti-Affinity — spread across nodes/zones
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: ["my-app"]
        topologyKey: "topology.kubernetes.io/zone"

# 3. Pod Disruption Budget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app

# 4. Health checks
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080

# 5. HPA for auto-scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

### 33. How do you handle Pod scheduling for mixed workloads (latency-sensitive services + batch jobs)?

**Answer:**

```yaml
# Node pool 1: Dedicated to latency-sensitive workloads
# Label nodes: workload-type=realtime
# Taint: kubectl taint nodes <node> workload=realtime:NoSchedule

# Latency-sensitive Pod
spec:
  nodeSelector:
    workload-type: realtime
  tolerations:
    - key: "workload"
      value: "realtime"
      effect: "NoSchedule"
  priorityClassName: high-priority
  resources:
    requests:
      cpu: "1"
      memory: "2Gi"
    limits:
      cpu: "2"
      memory: "4Gi"

---
# Batch job Pod — no node selector, runs on general nodes
spec:
  priorityClassName: low-priority
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
```

Use **PriorityClasses** so latency-sensitive Pods can preempt batch jobs if resources are scarce.

---

## Cluster Administration Scenarios

### 34. How do you upgrade a Kubernetes cluster?

**Answer:**

```bash
# 1. BACKUP: Snapshot etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-pre-upgrade.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=<ca-cert> --cert=<cert> --key=<key>

# 2. Check release notes for breaking changes
# Always upgrade one minor version at a time (e.g., 1.27 → 1.28, NOT 1.27 → 1.29)

# 3. Upgrade control plane (one node at a time)
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.28.0
sudo apt-get update && sudo apt-get install -y kubelet=1.28.0-00 kubectl=1.28.0-00
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# 4. Upgrade worker nodes (one at a time)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
# SSH to node:
sudo kubeadm upgrade node
sudo apt-get update && sudo apt-get install -y kubelet=1.28.0-00
sudo systemctl daemon-reload && sudo systemctl restart kubelet
# From control plane:
kubectl uncordon <node>

# 5. Upgrade add-ons (CNI, CoreDNS, Ingress controllers)

# 6. Verify
kubectl get nodes
kubectl get pods -A
```

---

### 35. How do you handle a scenario where etcd is corrupted?

**Answer:**

```bash
# 1. Stop the API server to prevent further writes
sudo systemctl stop kube-apiserver   # or move static pod manifest

# 2. Check for available snapshots
ls -la /backup/etcd-*.db

# 3. Restore from snapshot
ETCDCTL_API=3 etcdutl \
  --data-dir /var/lib/etcd-restored \
  snapshot restore /backup/etcd-snapshot.db

# 4. Update etcd configuration to use restored data directory
# Edit etcd static pod manifest or systemd unit

# 5. Restart etcd and API server
sudo systemctl start etcd
sudo systemctl start kube-apiserver

# 6. Verify cluster health
kubectl get nodes
kubectl get pods -A
etcdctl endpoint health
```

**Prevention:** Schedule regular etcd backups (e.g., CronJob every 6 hours).

---

### 36. All pods in a StatefulSet are connecting to the same storage volume. What's wrong?

**Answer:**

The issue is an incorrect `volumeClaimTemplates` configuration. StatefulSets should create **unique PVCs per Pod**.

**Wrong (shared volume):**
```yaml
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: shared-pvc    # All pods share this PVC
```

**Correct (per-pod PVC via volumeClaimTemplates):**
```yaml
spec:
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
        storageClassName: fast-ssd
```

Each StatefulSet Pod gets its own PVC with naming pattern: `<claim-name>-<statefulset-name>-<ordinal>` (e.g., `data-mysql-0`, `data-mysql-1`, `data-mysql-2`).

---

### 37. How do you handle a node that has disk pressure?

**Answer:**

```bash
# 1. Check node conditions
kubectl describe node <node-name> | grep -A5 "Conditions"

# 2. SSH to the node and check disk usage
df -h
du -sh /var/lib/kubelet/* | sort -hr | head -10
du -sh /var/lib/containerd/* | sort -hr | head -10
du -sh /var/log/* | sort -hr | head -10

# 3. Clean up container images
crictl rmi --prune          # Remove unused images
docker system prune -a -f   # If using Docker

# 4. Clean up old logs
journalctl --vacuum-size=500M
find /var/log -name "*.gz" -mtime +7 -delete

# 5. Clean up completed pods
kubectl delete pods --field-selector=status.phase=Succeeded -A
kubectl delete pods --field-selector=status.phase=Failed -A

# 6. Set image garbage collection thresholds in kubelet config
# --image-gc-high-threshold=85
# --image-gc-low-threshold=80
```

---

### 38. How do you debug a Pod that can't mount its PVC?

**Answer:**

```bash
# 1. Check Pod events
kubectl describe pod <pod-name>
# Look for: "FailedMount", "FailedAttachVolume"

# 2. Check PVC status
kubectl get pvc <pvc-name>
# Status should be "Bound", not "Pending"

# 3. If PVC is Pending — check PV availability
kubectl get pv
kubectl describe pvc <pvc-name>

# 4. Check StorageClass exists
kubectl get storageclass

# 5. Common issues:
```

| Issue | Fix |
|-------|-----|
| PVC Pending, no PV available | Create PV or check StorageClass provisioner |
| Access mode mismatch | PVC requests RWX but PV only supports RWO |
| Volume still attached to old node | Wait for old pod to terminate, or force detach |
| Storage quota exceeded | Check resource quotas |
| CSI driver not installed | Install the correct CSI driver for your storage backend |

---

### 39. How do you set up monitoring alerts for your cluster?

**Answer:**

**Prometheus + Alertmanager setup:**

```yaml
# PrometheusRule — define alert conditions
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-alerts
spec:
  groups:
    - name: pod-alerts
      rules:
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"

        - alert: HighMemoryUsage
          expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Container {{ $labels.container }} memory usage > 90%"

        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready", status="true"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} is not ready"
```

**Key metrics to monitor:**
- Pod restart count, CrashLoopBackOff events
- CPU/memory usage vs limits
- Node health and disk pressure
- API server latency and error rate
- etcd health and disk usage
- PVC usage percentage

---

### 40. How do you migrate workloads from one cluster to another?

**Answer:**

```bash
# Method 1: Velero (recommended for full migration)
# Backup from source cluster
velero backup create cluster-backup --include-namespaces production

# Restore to target cluster
velero restore create --from-backup cluster-backup

# Method 2: Export and apply manifests
# Export all resources from a namespace
kubectl get all,configmaps,secrets,pvc -n production -o yaml > resources.yaml
# Edit resources.yaml (remove status, resourceVersion, uid, etc.)
# Apply to new cluster
kubectl apply -f resources.yaml

# Method 3: GitOps (cleanest)
# If using Argo CD / Flux — point the new cluster to the same Git repo
# The GitOps agent will automatically reconcile the desired state

# For stateful data:
# 1. Backup PV data (database dumps, file copies)
# 2. Restore data to new cluster's PVs
# 3. Verify application state
```

---

### 41. You need to schedule a Pod on a specific node. How can you achieve this?

**Answer:** Three approaches, from flexible to rigid:

```yaml
# Method 1: nodeSelector (simplest)
spec:
  nodeSelector:
    disktype: ssd

# Method 2: Node Affinity (more flexible)
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                  - "node-1"

# Method 3: nodeName (bypasses scheduler entirely — NOT recommended)
spec:
  nodeName: node-1
```

```bash
# Label a node first
kubectl label nodes <node-name> disktype=ssd
```

**Best practice:** Use `nodeSelector` or Node Affinity. Avoid `nodeName` as it bypasses the scheduler and doesn't respect taints, affinity, or resource checks.

---

### 42. How do you handle a situation where a team is consuming all cluster resources?

**Answer:**

```yaml
# 1. Set Resource Quotas per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "100"

---
# 2. Set LimitRange to enforce defaults and max per pod
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: team-a
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "4Gi"
      type: Container
```

```bash
# 3. Monitor usage
kubectl describe resourcequota -n team-a
kubectl top pods -n team-a --sort-by=cpu

# 4. Implement priority classes so critical workloads always get resources
# 5. Enable Cluster Autoscaler for dynamic node scaling
# 6. Review and right-size resource requests/limits regularly
```

---

### 43. How do you handle log aggregation for a large Kubernetes cluster?

**Answer:**

**Architecture:**
```
Pods (stdout/stderr) → DaemonSet (Fluent Bit/Fluentd) → Aggregator → Storage → Dashboard
```

**Option 1: Loki Stack (lightweight)**
```bash
# Deploy with Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --set fluent-bit.enabled=true \
  --set grafana.enabled=true
```

**Option 2: ELK Stack (enterprise)**
```bash
# Elasticsearch + Logstash + Kibana
# Deploy Filebeat as DaemonSet on all nodes
```

**Best practices:**
- Deploy log collectors as **DaemonSets** (one per node)
- Apps log to **stdout/stderr** (not files)
- Set **log rotation** on nodes
- Use **structured logging** (JSON) for easier parsing
- Add **labels** for filtering (service, namespace, environment)
- Set **retention policies** (delete logs older than 30 days)

---

### 44. How do you test a Kubernetes manifest before applying it to production?

**Answer:**

```bash
# 1. Dry run — validate without applying
kubectl apply -f manifest.yaml --dry-run=client
kubectl apply -f manifest.yaml --dry-run=server   # Server-side validation

# 2. Diff — see what would change
kubectl diff -f manifest.yaml

# 3. Validate YAML syntax
kubectl apply -f manifest.yaml --validate=true

# 4. Lint with kubeval or kubeconform
kubeval manifest.yaml
kubeconform -strict manifest.yaml

# 5. Policy checks with kyverno / OPA
kyverno apply policy.yaml --resource manifest.yaml

# 6. Apply to staging namespace first
kubectl apply -f manifest.yaml -n staging

# 7. Test with a temporary namespace
kubectl create namespace test-deploy
kubectl apply -f manifest.yaml -n test-deploy
# Verify, then clean up
kubectl delete namespace test-deploy
```

---

### 45. How do you handle certificate rotation in a Kubernetes cluster?

**Answer:**

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Renew all certificates
sudo kubeadm certs renew all

# Restart control plane components to pick up new certs
sudo systemctl restart kubelet

# For auto-rotation (kubelet certificates):
# In kubelet config:
# rotateCertificates: true
# serverTLSBootstrap: true

# Verify
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

**Best practice:** Set up monitoring alerts for certificate expiration (30 days before).
