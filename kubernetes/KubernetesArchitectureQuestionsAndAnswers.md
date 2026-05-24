# Kubernetes Architecture Interview Questions and Answers

A comprehensive collection of Kubernetes architecture interview questions and answers — from beginner to advanced level. Covers core components, networking, storage, security, workload management, and cluster operations.

---

## Table of Contents

- [Basic / Core Architecture Questions](#basic--core-architecture-questions)
- [Networking Questions](#networking-questions)
- [Workload & Scheduling Questions](#workload--scheduling-questions)
- [Storage Questions](#storage-questions)
- [Security Questions](#security-questions)
- [Advanced Architecture Questions](#advanced-architecture-questions)

---

## Basic / Core Architecture Questions

### 1. Explain Kubernetes Architecture.

**Answer:** Kubernetes follows a master-worker (control plane / data plane) architecture:

**Control Plane (Cluster Management):**
- **API Server (kube-apiserver):** Entry point for all REST commands. Validates and processes requests, then updates cluster state in etcd. Every component — including `kubectl` — communicates through this API.
- **etcd:** Distributed key-value store that acts as the single source of truth for all cluster data — desired state, current state, metadata, RBAC policies, and runtime configuration.
- **Scheduler (kube-scheduler):** Watches for newly created Pods with no assigned node and selects a suitable node based on resource availability, affinity/anti-affinity rules, taints/tolerations, and other constraints.
- **Controller Manager (kube-controller-manager):** Runs a collection of controllers bundled into a single process. Each controller watches for a specific resource type and reconciles desired state vs. actual state (e.g., Deployment controller, ReplicaSet controller, Node controller, Job controller).

**Data Plane (Worker Nodes):**
- **Kubelet:** Agent running on each node. Receives instructions from the API server and ensures containers are running as specified. Reports node and Pod status back to the control plane.
- **Kube-proxy:** Manages network rules on every node for Pod communication. Handles service discovery, load balancing, and protocol routing.
- **Container Runtime:** Software that actually executes containers (e.g., containerd, CRI-O). Kubernetes removed direct Docker support (dockershim) in v1.24 and moved to CRI-based runtimes.

```
┌─────────────────────────────────────────────────────────────┐
│                    CONTROL PLANE                            │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌──────────┐  │
│  │API Server│  │   etcd   │  │ Scheduler │  │Controller│  │
│  │          │  │          │  │           │  │ Manager  │  │
│  └────┬─────┘  └──────────┘  └───────────┘  └──────────┘  │
│       │                                                     │
└───────┼─────────────────────────────────────────────────────┘
        │
┌───────┼─────────────────────────────────────────────────────┐
│       ▼            WORKER NODES                             │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐             │
│  │ Kubelet  │  │Kube-proxy│  │Container      │             │
│  │          │  │          │  │Runtime        │             │
│  └──────────┘  └──────────┘  └───────────────┘             │
│                                                             │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                       │
│  │Pod 1│  │Pod 2│  │Pod 3│  │Pod N│                       │
│  └─────┘  └─────┘  └─────┘  └─────┘                       │
└─────────────────────────────────────────────────────────────┘
```

---

### 2. What is container orchestration?

**Answer:** Container orchestration is the automated process of managing the entire lifecycle of software containers. It handles provisioning, deployment, scaling (up/down), networking, load balancing, and health monitoring of containers across a cluster of machines. Without orchestration, you'd manually decide which server runs which container, handle failures yourself, and manage networking and scaling by hand. Kubernetes is the most widely used orchestrator that automates all of this.

---

### 3. What problems does Kubernetes solve, and why not just run containers directly?

**Answer:** Running `docker run` works fine on a single machine, but at scale you hit problems that a container runtime alone doesn't solve:

- **Scheduling:** Deciding which node a container should run on based on CPU, memory, and constraints
- **Self-healing:** If a container crashes or a node goes down, Kubernetes detects it and reschedules pods automatically
- **Service discovery & load balancing:** Assigns DNS names and IPs to groups of pods, distributes traffic across them
- **Storage orchestration:** Automatically mounts the right storage system (cloud disk, NFS, local)
- **Secrets management:** Stores and manages sensitive data separately from application code
- **Resource efficiency:** Bin-packs containers onto nodes based on resource requirements
- **Rolling updates:** Updates applications with zero downtime by gradually replacing old pods

---

### 4. What is the difference between a Pod and a container?

**Answer:** A container is a single, isolated process running with its own filesystem, network stack, and process space. A Pod is the smallest deployable unit in Kubernetes — a logical host for one or more tightly coupled containers that share:

- **Network namespace:** Containers in the same Pod communicate via `localhost`
- **Storage volumes:** Shared volumes accessible by all containers in the Pod
- **Lifecycle:** All containers in a Pod are co-scheduled and co-located on the same node

While a Pod can contain a single container (most common), the Pod itself is what Kubernetes manages, schedules, and scales — not the individual container.

---

### 5. What is the purpose of the Kubernetes API server?

**Answer:** The API server (`kube-apiserver`) is the front end of the Kubernetes control plane. It:

- Serves as the central hub through which **all** components communicate
- Handles all REST operations — create, read, update, delete resources
- Validates incoming requests before persisting them to etcd
- Handles authentication, authorization (RBAC), and admission control
- Exposes the Kubernetes API that `kubectl`, client libraries, dashboards, and CI/CD tools use

Every interaction with the cluster — whether from a human, a controller, or an external system — goes through the API server.

---

### 6. Describe the role of etcd in Kubernetes.

**Answer:** etcd is a distributed, consistent, and highly available key-value store that serves as the central database for the entire Kubernetes cluster.

**What etcd stores:**
- **Desired state:** Definitions of Deployments, Services, ConfigMaps, Secrets, etc.
- **Current state:** Real-time status of Pods, Nodes, and other components
- **Metadata:** Cluster-wide configuration, RBAC policies, and runtime data

**Key characteristics:**
- Uses the Raft consensus algorithm for distributed consistency
- All reads/writes go through the API server (never accessed directly by other components)
- Regular snapshots are critical for disaster recovery
- Loss of etcd data = loss of the entire cluster state

---

### 7. What is kubectl?

**Answer:** `kubectl` is the primary command-line tool for communicating with a Kubernetes cluster. Under the hood, every `kubectl` command translates into an HTTP request to the Kubernetes API server. You use it to create and manage resources, inspect cluster state, view logs, and debug workloads.

Other ways to interact with Kubernetes:
- **Kubernetes API directly:** Using `curl` or any HTTP client for automation and CI/CD pipelines
- **Client libraries:** Official SDKs for Go, Python, Java, etc.
- **Kubernetes Dashboard:** Web-based UI for visual cluster overview
- **IaC tools:** Helm, Kustomize, Terraform for declarative resource management

---

### 8. What is a Kubernetes cluster?

**Answer:** A Kubernetes cluster is a set of nodes (physical or virtual machines) that work together to run containerized applications. It consists of:

- **Control plane nodes:** Run the API server, etcd, scheduler, and controller manager
- **Worker nodes:** Run the kubelet, kube-proxy, container runtime, and the actual application Pods

A cluster provides a unified platform where you declare what you want (desired state), and Kubernetes continuously works to make the actual state match.

---

### 9. What are Namespaces in Kubernetes?

**Answer:** Namespaces divide a single physical cluster into virtual sub-clusters, each with its own scope for resources like Pods, Services, and ConfigMaps.

**Benefits:**
- **Isolation:** Each team/project gets its own namespace, preventing accidental interference
- **Naming conflicts:** Resources like `web-service` can exist in multiple namespaces without clashing
- **Access control:** RBAC policies can be applied per namespace
- **Resource quotas:** Limits on CPU, memory, and object counts per namespace

**Default namespaces:** `default`, `kube-system`, `kube-public`, `kube-node-lease`

```bash
kubectl get namespaces
kubectl create namespace dev-team
kubectl get pods -n dev-team
```

---

### 10. What is the relationship between a Deployment, ReplicaSet, and Pod?

**Answer:** These form a management hierarchy:

- **Deployment:** Defines the desired state of an application (replicas, container image, update strategy). It's the object you create and manage.
- **ReplicaSet:** Ensures the specified number of Pod replicas are running at any time. Created and managed automatically by the Deployment.
- **Pod:** The smallest deployable unit — runs one or more containers.

**Update flow:**
1. You update a Deployment (e.g., new image)
2. Deployment creates a **new ReplicaSet** with updated config
3. New ReplicaSet scales up while old one scales down (rolling update)
4. Old ReplicaSet is retained (scaled to zero) for rollback

```
Deployment
   └── ReplicaSet (current)
   │      ├── Pod-1
   │      ├── Pod-2
   │      └── Pod-3
   └── ReplicaSet (previous, scaled to 0 — for rollback)
```

---

### 11. What are the various Kubernetes service types?

**Answer:** Kubernetes supports four service types:

| Service Type | Description | Use Case |
|-------------|-------------|----------|
| **ClusterIP** | Exposes service on a cluster-internal IP. Only reachable within the cluster. | Internal microservice communication |
| **NodePort** | Exposes service on a static port on every node's IP. Accessible externally via `<NodeIP>:<NodePort>`. | Development/testing, simple external access |
| **LoadBalancer** | Provisions an external load balancer (cloud provider). Gets its own external IP. | Production external access |
| **ExternalName** | Maps a service to an external DNS name. No proxying. | Accessing external services from within cluster |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

---

### 12. What are ConfigMaps and Secrets in Kubernetes, and how do they differ?

**Answer:** Both externalize configuration from application code:

**ConfigMap:**
- Stores **non-sensitive** configuration data as key-value pairs
- Used for environment variables, command-line args, or config files
- Data stored in plain text

**Secret:**
- Stores **sensitive** data like passwords, tokens, SSH keys
- Data is base64-encoded by default (not encrypted unless you configure encryption at rest)
- Supports fine-grained access control via RBAC
- Can be mounted as files or exposed as environment variables

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  LOG_LEVEL: "info"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64 encoded
```

---

### 13. Explain the use of Labels and Selectors in Kubernetes.

**Answer:** Labels are key-value pairs attached to Kubernetes objects (Pods, Services, Deployments) that identify them. Selectors are queries that match objects based on their labels.

**How they work together:**
- A **Deployment** assigns labels to Pods it creates
- A **Service** uses selectors to find which Pods to route traffic to
- **ReplicaSets** use selectors to know which Pods they manage

```yaml
# Deployment labels its Pods
metadata:
  labels:
    app: web
    tier: frontend

# Service selects Pods with matching labels
spec:
  selector:
    app: web
    tier: frontend
```

Labels and selectors are the fundamental mechanism for connecting Deployments, Pods, and Services in Kubernetes.

---

### 14. What are Liveness, Readiness, and Startup Probes?

**Answer:** Probes are health checks performed by the Kubelet to monitor container status:

| Probe | Purpose | On Failure |
|-------|---------|------------|
| **Liveness** | Checks if container is still running | Container is killed and restarted |
| **Readiness** | Checks if container is ready to serve traffic | Pod removed from Service endpoints (no traffic sent) |
| **Startup** | Verifies if application has started successfully | Disables liveness/readiness until it passes |

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**Startup probes** are crucial for slow-starting apps — they prevent premature restarts by disabling liveness/readiness checks until the app is ready.

---

### 15. How do rolling updates work in a Deployment?

**Answer:**
1. The Deployment controller creates a **new ReplicaSet** with the updated configuration (e.g., new container image)
2. It incrementally **scales up** the new ReplicaSet while **scaling down** the old one
3. This continues until the desired number of updated Pods is running and old Pods are terminated
4. The old ReplicaSet is retained (scaled to zero) for rollback capability

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max extra pods during update
      maxUnavailable: 0   # Zero downtime
```

```bash
# Trigger rolling update
kubectl set image deployment/my-app my-app=my-app:v2

# Check rollout status
kubectl rollout status deployment/my-app

# Rollback if needed
kubectl rollout undo deployment/my-app
```

---

## Networking Questions

### 16. How does Kubernetes enforce communication boundaries between Pods?

**Answer:** By default, all Pods can communicate freely with each other. **Network Policies** act as firewalls at Layer 3/4 (IP and port level) to restrict traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: allowed-app
```

**Layered networking architecture:**
- **Services:** Handle L4 connectivity and internal load balancing
- **Ingress:** Manages L7 routing for external HTTP/S traffic
- **Network Policies:** Enforce security filtering at the IP/port level

Network Policies require a CNI plugin that supports them (Calico, Cilium, etc.).

---

### 17. Explain the concept of Ingress in Kubernetes.

**Answer:** Ingress is a Kubernetes API object that defines **Layer 7 routing rules** for managing external HTTP/HTTPS traffic into the cluster.

**Ingress** (resource):
- Routes traffic based on hostnames (e.g., `api.example.com`) or paths (e.g., `/users`)
- Enables centralized, declarative management of external access
- Supports TLS termination

**Ingress Controller** (runtime):
- Enforces the rules defined in the Ingress object
- Acts as a reverse proxy / load balancer (e.g., NGINX, Traefik, cloud-native options)
- Must be deployed separately — Ingress rules alone don't work without it

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 80
```

---

### 18. What is the Gateway API, and how is it different from Ingress?

**Answer:** The Gateway API is a newer Kubernetes standard for managing traffic routing. It addresses Ingress limitations by providing a more expressive, extensible, and role-oriented model.

**Key differences:**

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Resource model | Single resource | Split across GatewayClass, Gateway, HTTPRoute |
| Role separation | No | Platform team manages Gateway, app team manages Routes |
| Traffic splitting | Annotations only | Native support |
| Header-based routing | Annotations only | Native support |
| Cross-namespace routing | Limited | Native support |
| Protocol support | HTTP/HTTPS | HTTP, TCP, gRPC, UDP |

**Gateway API resources:**
- **GatewayClass:** Defines the type of infrastructure (like StorageClass for storage)
- **Gateway:** Represents the entry point (load balancer) — managed by platform team
- **HTTPRoute / TCPRoute / GRPCRoute:** Defines routing rules — managed by app teams

---

### 19. Describe the role of kube-proxy in the cluster.

**Answer:** kube-proxy is a network component running on every node. Its primary job is enabling communication between Services and Pods:

- **Service discovery & routing:** Translates Service virtual IPs into actual Pod IPs
- **Load balancing:** Distributes traffic across healthy Pod endpoints
- **Protocol handling:** Supports TCP, UDP, and SCTP
- **Implementation modes:** Can use iptables rules (default), IPVS (for large clusters), or userspace proxying

When you create a Service, kube-proxy on every node updates network rules so that traffic destined for the Service's ClusterIP is forwarded to one of the backing Pods.

---

### 20. How do containers within a Pod communicate with each other?

**Answer:** Containers within the same Pod share the same network namespace, so they communicate via `localhost` on different ports. They can also share storage volumes.

**Pod-to-Pod communication** across nodes is handled by a **CNI (Container Network Interface)** plugin. Common CNI plugins:
- **Calico:** Widely used, supports Network Policies
- **Flannel:** Simpler option, basic overlay networking
- **Cilium:** Uses eBPF for high-performance networking and advanced observability

The Kubernetes networking model requires that every Pod gets its own IP and can reach every other Pod directly without NAT. The CNI plugin implements this.

---

### 21. What is a LoadBalancer Service in Kubernetes?

**Answer:** A `LoadBalancer` Service type provisions an external load balancer through your cloud provider (AWS, GCP, Azure, Civo, etc.). When created, Kubernetes requests the cloud to spin up a load balancer that routes external traffic to the correct set of Pods.

**Characteristics:**
- Each LoadBalancer Service gets its own external IP
- Works well for a small number of externally-exposed services
- Becomes expensive at scale (each service = separate cloud load balancer)
- For many services, use **Ingress** instead (single load balancer + routing rules)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

---

## Workload & Scheduling Questions

### 22. What is the difference between a Deployment, a StatefulSet, and a DaemonSet?

**Answer:**

| Feature | Deployment | StatefulSet | DaemonSet |
|---------|-----------|-------------|-----------|
| **Use case** | Stateless apps | Stateful apps (databases, queues) | Node-level agents |
| **Pod identity** | Interchangeable | Stable, unique (pod-0, pod-1) | One per node |
| **Storage** | Shared or none | Dedicated PVC per pod | Typically host-path |
| **Scaling** | Any order | Ordered (sequential) | Auto (one per node) |
| **Examples** | Web servers, APIs | PostgreSQL, Kafka, Redis | Log collectors, monitoring agents, CNI plugins |

---

### 23. What are Taints and Tolerations?

**Answer:** Taints and Tolerations control which Pods can run on which Nodes:

- **Taint (Node-level):** Applied to a Node. Tells Kubernetes: "Do not schedule Pods here unless they explicitly tolerate this."
- **Toleration (Pod-level):** Added to a Pod. Allows it to ignore specific taints and be scheduled on matching Nodes. Does NOT guarantee placement — just permits it.

```bash
# Taint a node
kubectl taint nodes node1 key1=value1:NoSchedule
```

```yaml
# Pod tolerates the taint
spec:
  tolerations:
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoSchedule"
```

**Taint effects:**
- `NoSchedule` — Don't schedule new pods
- `PreferNoSchedule` — Try to avoid, but allow if necessary
- `NoExecute` — Evict existing pods that don't tolerate

---

### 24. What are Affinity and Anti-Affinity?

**Answer:**

- **Node Affinity:** Attracts Pods to specific nodes based on node labels. More expressive than `nodeSelector` — supports `In`, `NotIn`, `Exists`, `DoesNotExist` operators.
- **Pod Affinity:** Attracts Pods to be co-located with other Pods (e.g., same zone for low latency).
- **Pod Anti-Affinity:** Repels Pods from being co-located (e.g., spread replicas across zones for HA).

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: ["web"]
        topologyKey: "kubernetes.io/hostname"
```

This ensures no two `web` Pods run on the same node.

---

### 25. How does the Kubernetes Scheduler assign Pods to Nodes?

**Answer:** The scheduler (`kube-scheduler`) follows a two-phase process:

1. **Filtering (Predicates):** Eliminates nodes that can't host the Pod based on:
   - Insufficient CPU/memory
   - Node taints the Pod can't tolerate
   - Node selectors or affinity rules that don't match
   - Volume or topology constraints

2. **Scoring (Priorities):** Ranks remaining eligible nodes using priority functions:
   - Resource balance
   - Image locality (node already has the image)
   - Inter-pod affinity preferences

3. **Binding:** Pod is bound to the highest-scoring node, and the API server updates cluster state.

---

### 26. What is a Horizontal Pod Autoscaler (HPA)?

**Answer:** HPA automatically scales the number of Pod replicas based on observed metrics (CPU, memory, or custom metrics). It increases Pods to handle load and decreases them when load subsides.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

HPA checks metrics every 15 seconds (default) and adjusts replicas accordingly.

---

### 27. What are requests and limits?

**Answer:**

- **Requests:** The guaranteed amount of CPU/memory a container receives. The scheduler uses requests to decide node placement. If a node doesn't have enough unreserved resources, the Pod won't be scheduled there.
- **Limits:** The maximum amount a container is allowed to consume. If it exceeds memory limit, it's OOMKilled. If it exceeds CPU limit, it's throttled.

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

---

### 28. What are QoS (Quality of Service) classes?

**Answer:** Kubernetes automatically assigns a QoS class to each Pod based on requests/limits configuration:

| QoS Class | Condition | Eviction Priority |
|-----------|-----------|-------------------|
| **Guaranteed** | Every container has equal requests and limits | Last to be evicted |
| **Burstable** | At least one container has requests or limits set, but not equal | Middle |
| **BestEffort** | No requests or limits set | First to be evicted |

During node memory pressure, Kubernetes evicts BestEffort first, then Burstable, then Guaranteed. Understanding this helps you configure requests/limits for workloads of different importance.

---

### 29. What is a Pod Disruption Budget (PDB)?

**Answer:** A PDB tells Kubernetes the minimum number of Pods in a group that must remain available during **voluntary disruptions** (node drains, cluster upgrades, autoscaler scale-downs). It does NOT cover involuntary disruptions (hardware failures, kernel crashes).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 3       # OR maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

If you have 5 replicas and set `minAvailable: 3`, Kubernetes blocks any voluntary disruption that would bring available count below 3.

---

### 30. What is a DaemonSet?

**Answer:** A DaemonSet ensures that a copy of a specific Pod runs on **every node** (or a defined subset) in the cluster. When a new node is added, the DaemonSet automatically schedules a Pod on it. When a node is removed, the Pod is cleaned up.

**Common use cases:**
- Log collectors (Fluentd, Filebeat)
- Monitoring agents (Prometheus node exporter)
- Network plugins (CNI components)
- Storage daemons

**Key difference from Deployment:** A Deployment says "run N replicas somewhere." A DaemonSet says "run exactly one replica on every eligible node."

---

### 31. Describe the use of init containers.

**Answer:** Init containers are specialized containers that run **before** application containers in a Pod. They run sequentially and must complete successfully before the main containers start.

**Use cases:**
- Run database migrations before the app starts
- Wait for a dependency service to be available
- Download configuration files or secrets
- Set up filesystem permissions

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nslookup postgres-service; do sleep 2; done']
    - name: run-migration
      image: myapp:migration
      command: ['sh', '-c', 'run-migration.sh']
  containers:
    - name: app
      image: myapp:latest
```

---

### 32. What is a sidecar container?

**Answer:** A sidecar container runs alongside the main application container within the same Pod. It enhances or supports the primary container without being part of the core application logic.

**Common use cases:**
- Log forwarding (Fluentd/Filebeat reads logs from shared volume)
- Metrics collection (Prometheus exporter)
- Service mesh proxies (Envoy/Istio)
- TLS termination
- Data synchronization

**Native Sidecars (Kubernetes v1.28+):** Defined in `initContainers` with `restartPolicy: Always`. Kubernetes guarantees they start before the main container, solving the race condition where a sidecar might not be ready when the app starts.

---

### 33. What are Kubernetes Jobs and CronJobs?

**Answer:**

- **Job:** Creates one or more Pods and ensures they run to completion. If a Pod fails, the Job creates a new one. Used for batch processing, data migration, report generation.
- **CronJob:** Creates Jobs on a recurring schedule (cron syntax). Used for periodic tasks like backups, cleanup, report emails.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"    # Every day at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: backup-tool:latest
              command: ["/bin/sh", "-c", "backup.sh"]
          restartPolicy: OnFailure
```

---

## Storage Questions

### 34. What are Persistent Volumes (PVs) and PersistentVolumeClaims (PVCs)?

**Answer:** A two-part abstraction for managing storage:

- **PersistentVolume (PV):** A piece of storage in the cluster — provisioned manually by an admin or dynamically via a StorageClass. It's a cluster-level resource that exists independently of any Pod.
- **PersistentVolumeClaim (PVC):** A request for storage by a user/application. Specifies size and access mode. Kubernetes matches the claim to an available PV.

This separation decouples infrastructure (what storage exists) from consumption (what the app needs).

```yaml
# PVC — application requests storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2
```

**Access modes:**
- `ReadWriteOnce (RWO)` — Mounted read-write by a single node
- `ReadOnlyMany (ROX)` — Mounted read-only by many nodes
- `ReadWriteMany (RWX)` — Mounted read-write by many nodes

---

### 35. How does Kubernetes manage storage orchestration?

**Answer:** Kubernetes uses the **Container Storage Interface (CSI)** standard to interact with storage systems.

**Key components:**
- **PersistentVolume (PV):** Represents provisioned storage
- **PersistentVolumeClaim (PVC):** A request for storage
- **StorageClass:** Defines the type of storage (SSD, HDD, encrypted) and links to a CSI driver for **dynamic provisioning**

In practice, most teams use StorageClasses so that when a PVC is created, the StorageClass automatically provisions a matching PV — no manual volume creation needed.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iopsPerGB: "50"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

---

## Security Questions

### 36. Explain Role-Based Access Control (RBAC) in Kubernetes.

**Answer:** RBAC controls who can do what within the cluster using four key resources:

| Resource | Scope | Purpose |
|----------|-------|---------|
| **Role** | Namespace | Defines permissions within a specific namespace |
| **RoleBinding** | Namespace | Grants a Role to a user/group/service account in that namespace |
| **ClusterRole** | Cluster-wide | Defines permissions across the entire cluster |
| **ClusterRoleBinding** | Cluster-wide | Grants a ClusterRole at the cluster level |

```yaml
# Role — allows reading pods in "dev" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
# RoleBinding — binds role to user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
  - kind: User
    name: developer1
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Check permissions
kubectl auth can-i create deployments --namespace production
kubectl auth can-i get secrets --as system:serviceaccount:default:my-app
```

---

### 37. How can you secure a Kubernetes cluster?

**Answer:** Follow the **4C security model:**

1. **Cloud provider security:** Use IAM roles, firewall rules, private API endpoints
2. **Cluster security:** Enable RBAC, audit logging, API server encryption, rotate certificates
3. **Container security:** Scan images for vulnerabilities, use non-root users, read-only filesystems
4. **Code security:** Secrets management (Vault, AWS Secrets Manager), network policies, pod security admission

**Additional best practices:**
- Enable encryption at rest for etcd Secrets
- Use Pod Security Admission to enforce Pod Security Standards
- Enforce Network Policies (deny all by default, then allow explicitly)
- Regularly rotate API tokens and certificates
- Use namespaces and resource quotas for multi-tenancy

---

### 38. What are Admission Controllers?

**Answer:** Admission controllers sit in the request path between the API server receiving a request and the resource being persisted to etcd. They intercept create/update/delete requests and can modify or reject them.

**Two types:**
- **Mutating admission controllers:** Modify incoming requests (e.g., inject sidecar containers, add default resource limits, attach required labels). Run first.
- **Validating admission controllers:** Check if requests meet criteria and reject if not (e.g., block pods running as root, require specific annotations). Run second.

Tools like **OPA/Gatekeeper** or **Kyverno** let you write custom validation/mutation rules without building webhooks from scratch.

---

### 39. How do you encrypt Kubernetes Secrets in etcd?

**Answer:** By default, Secrets are stored as base64-encoded (not encrypted) in etcd. To enable encryption at rest:

1. Create an encryption configuration file:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

2. Pass it to the API server via `--encryption-provider-config` flag
3. Re-encrypt existing Secrets: `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`

For production, use a KMS provider (AWS KMS, HashiCorp Vault) instead of local keys.

---

## Advanced Architecture Questions

### 40. What are Custom Resource Definitions (CRDs)?

**Answer:** CRDs extend the Kubernetes API with your own resource types. Out of the box, Kubernetes knows about Pods, Services, Deployments, etc. A CRD lets you define something entirely new (e.g., `Database`, `Certificate`) that Kubernetes treats like any native resource.

On its own, a CRD just stores data. It becomes useful when paired with a **controller** that watches for those resources and acts on them.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                version:
                  type: string
                replicas:
                  type: integer
```

```bash
kubectl get databases
kubectl describe database my-postgres
```

---

### 41. What is a Kubernetes Operator?

**Answer:** An Operator is a controller paired with one or more CRDs that encodes **domain-specific operational knowledge** into software. It automates complex application tasks — deployment, scaling, backup, upgrade, failure recovery — using Kubernetes-native APIs.

**Key components:**
- **CRD:** Defines the application-level resource (e.g., `PostgresCluster`)
- **Controller:** Contains logic for managing the full lifecycle
- **Reconciliation loop:** Continuously ensures the application state matches the desired state

**Difference from a plain controller:** Every Operator is a controller, but not every controller is an Operator. An Operator embeds application-specific logic that would otherwise require a human admin.

**Examples:** PostgreSQL Operator, Prometheus Operator, Elasticsearch Operator, Strimzi (Kafka)

---

### 42. What is a mutating admission webhook?

**Answer:** A mutating admission webhook is a dynamic admission controller that intercepts API requests before objects are persisted to etcd and **modifies** the request payload.

**Common uses:**
- Injecting sidecar containers (e.g., Istio proxy)
- Setting default resource limits for Pods
- Adding required labels/annotations for audit tracking
- Enforcing security configurations

Mutating webhooks run **before** validating webhooks, so validation checks the final modified version.

---

### 43. What is Helm, and how is it used?

**Answer:** Helm is a package manager for Kubernetes. A Helm **chart** contains all the Kubernetes manifests needed to run an application (Deployments, Services, ConfigMaps, etc.) plus a `values.yaml` for customization.

**Benefits:**
- Deploy complex apps with a single command
- Customize via values (replicas, image tag, resource limits) per environment
- Version and share charts through repositories
- Rollback to previous chart versions

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-postgres bitnami/postgresql --set auth.postgresPassword=secret

helm upgrade my-postgres bitnami/postgresql --set replicaCount=3
helm rollback my-postgres 1
```

---

### 44. How does Kustomize differ from Helm?

**Answer:**

| Feature | Helm | Kustomize |
|---------|------|-----------|
| **Approach** | Templating (Go templates with values) | Patching (base manifests + overlays) |
| **Configuration** | `values.yaml` | `kustomization.yaml` |
| **Manifests** | Templates (not valid YAML until rendered) | Always valid Kubernetes YAML |
| **Complexity** | More powerful, steeper learning curve | Simpler, native to kubectl |
| **Best for** | Sharing packages, complex apps | Environment-specific overrides |

**Choose Helm** when you need to package and distribute reusable application charts.
**Choose Kustomize** when you want lightweight environment-specific overlays on plain YAML.

---

### 45. What is GitOps, and how does it relate to Kubernetes?

**Answer:** GitOps is a methodology where Git is the single source of truth for the desired state of infrastructure and applications. An agent in the cluster continuously compares desired state (in Git) with actual state (in cluster) and reconciles any drift.

**Core principles:**
- Entire desired state is declared in Git
- All changes go through Git (PRs) — full audit trail + easy rollback
- An in-cluster agent detects drift and self-corrects

**Key difference from CI/CD:** In GitOps, the cluster **pulls** its own state from Git. In traditional CI/CD, an external system **pushes** updates.

**Tools:**
- **Argo CD:** Web UI, application-level management, Git sync with drift detection
- **Flux:** Lightweight, composable controllers, integrates with Helm/Kustomize

---

### 46. What is a service mesh?

**Answer:** A service mesh is a dedicated infrastructure layer that manages service-to-service communication within the cluster. It deploys a proxy (typically Envoy) as a sidecar alongside each Pod.

**Capabilities:**
- **Mutual TLS (mTLS):** Encrypts all service-to-service traffic, zero-trust networking
- **Observability:** Detailed metrics, distributed tracing, traffic logs — without code changes
- **Traffic management:** Canary deployments, A/B testing, retries, timeouts, circuit breaking
- **Rate limiting & access control:** Policy enforcement at the network level

**Popular implementations:** Istio, Linkerd, Consul Connect

**When to use:** When you need mTLS enforcement, deep observability, or fine-grained traffic control. The Gateway API now covers some traffic management features, reducing the need for a mesh in simpler cases.

---

### 47. How do you implement zero-downtime Deployments?

**Answer:** Kubernetes provides several strategies:

- **Rolling updates (default):** Gradually replace old Pods with new ones. Configure `maxSurge` and `maxUnavailable` for control.
- **Canary deployments:** Route a small percentage of traffic to new version, monitor, then scale up.
- **Blue-green deployments:** Run two full environments, switch traffic from old to new.
- **Readiness probes:** Ensure Kubernetes only sends traffic to Pods that are actually ready.
- **Pod Disruption Budgets:** Maintain minimum available Pods during updates.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
```

---

### 48. How do you back up and restore a Kubernetes cluster?

**Answer:** Two main components to back up:

**1. Cluster State (etcd):**
```bash
# Snapshot
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<ca-cert> --cert=<cert> --key=<key> \
  snapshot save /backup/etcd-snapshot.db

# Restore
etcdutl --data-dir /var/lib/etcd-restored snapshot restore /backup/etcd-snapshot.db
```

**2. Application Data (Persistent Volumes):**
- Use **Velero** to snapshot cluster resources and persistent volumes
- Velero can back up and restore entire namespaces or specific resources
- Supports scheduled backups and cross-cluster migration

---

### 49. How does Kubernetes handle node failures?

**Answer:** Kubernetes provides multiple resilience mechanisms:

- **Node health monitoring:** The controller manager detects unresponsive nodes (default: 40s timeout) and marks them as `NotReady`
- **Pod rescheduling:** After a grace period (default: 5 minutes), Pods on failed nodes are rescheduled to healthy nodes
- **Replication:** Deployments/ReplicaSets ensure the desired number of replicas are always running
- **Pod restart policies:** Containers can be automatically restarted on failure
- **Pod Disruption Budgets:** Maintain minimum availability during disruptions
- **Multi-zone deployments:** Spread nodes across availability zones for fault tolerance

---

### 50. How do you implement multi-tenancy in Kubernetes?

**Answer:** Two approaches:

**Soft multi-tenancy:**
- Namespaces for logical isolation
- RBAC for access control per namespace
- Network Policies to restrict cross-namespace traffic
- Resource Quotas to limit CPU/memory per namespace
- Limit Ranges for per-pod defaults

**Hard multi-tenancy:**
- Virtual clusters (e.g., vCluster) — separate control planes on shared infrastructure
- Dedicated clusters per tenant (most isolated, highest cost)

```yaml
# Resource quota per namespace
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
    pods: "50"
```

---

### 51. How do you monitor a Kubernetes cluster?

**Answer:** Recommended monitoring stack:

| Tool | Purpose |
|------|---------|
| **Prometheus + Grafana** | Collect and visualize cluster metrics, create alerts |
| **Loki + Fluentd/Fluent Bit** | Collect and analyze logs |
| **Jaeger / OpenTelemetry** | Distributed tracing |
| **Kubernetes Dashboard** | UI-based cluster overview |
| **kubectl top** | Quick resource usage check |

```bash
# Quick resource check
kubectl top nodes
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

---

### 52. How do you set up Kubernetes logging?

**Answer:** Kubernetes discourages writing logs to disk inside Pods. Instead:

1. **Applications log to stdout/stderr**
2. **Kubelet** collects these logs on each node
3. **View logs:** `kubectl logs <pod-name>` or `kubectl logs <pod-name> --previous` (crashed container)
4. **Centralized logging** via:
   - **Loki + Fluentd + Grafana** (lightweight, fast)
   - **ELK Stack** (Elastic, Logstash, Kibana — scalable, enterprise-grade)

Log collectors (Fluentd, Filebeat) are typically deployed as **DaemonSets** to run on every node.

---

### 53. How is Kubernetes related to Docker?

**Answer:** Docker was the default container runtime in Kubernetes for years. However, Kubernetes **removed direct Docker support (dockershim)** in version 1.24 and moved fully to CRI-based runtimes like containerd and CRI-O.

**Key distinction:**
- **Docker:** Focuses on building and running containers, packaging images
- **Kubernetes:** Orchestrates containers at scale

Docker images follow the OCI standard, so images built with Docker run on any CRI-compliant runtime. You can still use Docker for building images and Kubernetes for orchestrating them — they address different layers of the stack.
