# 🚀 Kubernetes Complete Deep Dive

> **A comprehensive guide covering every core concept, architecture, scheduling, resource management, multi-tenancy, observability, and interview Q&A**

---

## Table of Contents

1. [Cluster Architecture: The Full Stack](#1-cluster-architecture-the-full-stack)
2. [Namespaces: Auto-Created & Custom](#2-namespaces-auto-created--custom)
3. [Namespace Use Cases & Why You Need Them](#3-namespace-use-cases--why-you-need-them)
4. [When NOT to Use Namespaces (Problems You'll Face)](#4-when-not-to-use-namespaces-problems-youll-face)
5. [Load Balancer: 5 Algorithms](#5-load-balancer-5-algorithms)
6. [Multi-Tenancy](#6-multi-tenancy)
7. [CPU & Memory: Compressible vs Non-Compressible](#7-cpu--memory-compressible-vs-non-compressible)
8. [QoS (Quality of Service)](#8-qos-quality-of-service)
9. [Resource Quotas & LimitRanges](#9-resource-quotas--limitranges)
10. [Cost Optimization with OpenCost](#10-cost-optimization-with-opencost)
11. [Taints & Tolerations](#11-taints--tolerations)
12. [Node Selector, Affinity & Anti-Affinity](#12-node-selector-affinity--anti-affinity)
13. [Topology Spread Constraints](#13-topology-spread-constraints)
14. [New Pod Placement: Complete Picture](#14-new-pod-placement-complete-picture)
15. [Interview Questions & Answers](#15-interview-questions--answers)

---

## 1. Cluster Architecture: The Full Stack

### The Layered View (Bottom to Top)

```
┌─────────────────────────────────────────────────────────────────────┐
│  LAYER 12: Linux Kernel (cgroups, namespaces, network stack)        │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 11: cgroup v1 / v2 (CPU, Memory, I/O limits enforcement)     │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 10: Container Image (OCI: layers, base OS, app binaries)     │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 9:  container1  |  container2  (inside the same Pod)         │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 8:  Pod (smallest deployable unit; shared network+storage)   │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 7:  Endpoints / EndpointSlices (maps Service → Pod IPs)      │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 6:  ReplicaSet (ensures N replicas of a Pod template)        │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 5:  Deployment (rolling update, rollback over ReplicaSets)   │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 4:  Service (stable DNS + VIP + load balancing to Pods)      │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 3:  Namespace (virtual cluster; isolation boundary)          │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 2:  Node (VM or bare metal; Worker or Control Plane)         │
├─────────────────────────────────────────────────────────────────────┤
│  LAYER 1:  Cluster (1 Control Plane + N Worker Nodes)               │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Layer 1 — Cluster

**Definition:** A Kubernetes cluster is the top-level unit. It contains one or more control plane nodes and one or more worker nodes. Everything runs inside a single cluster's boundary, sharing the same API server, etcd, and cluster-level networking.

**Why needed:** Provides a single management boundary. Multiple teams, workloads, and environments can coexist.

**Example cluster with 2 nodes:**

```yaml
# kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
master-node    Ready    control-plane   30d   v1.29.0
worker-node1   Ready    <none>          30d   v1.29.0
```

---

### Layer 2 — Nodes (Worker + Control Plane)

**Definition:** A Node is a physical or virtual machine. Worker nodes run your workloads (Pods). The control plane node runs the API server, scheduler, controller-manager, and etcd.

**Control Plane Components:**

| Component | Role |
|---|---|
| `kube-apiserver` | The front door — all kubectl commands hit this |
| `etcd` | Distributed key-value store — the single source of truth |
| `kube-scheduler` | Watches for unscheduled Pods and assigns them to nodes |
| `kube-controller-manager` | Runs controllers: Node, ReplicaSet, Endpoint, etc. |
| `cloud-controller-manager` | Bridges K8s to cloud provider APIs (AWS, GCP, Azure) |

**Worker Node Components:**

| Component | Role |
|---|---|
| `kubelet` | The node agent — talks to API server, manages Pod lifecycle |
| `kube-proxy` | Manages iptables/IPVS rules for Service routing |
| `container runtime` | Actually runs containers (containerd, CRI-O) |

---

### Layer 3 — Namespaces

_(Covered in full in Section 2 and 3)_

---

### Layer 4 — Services

**Definition:** A Service is an abstraction that provides a stable IP address (ClusterIP), DNS name, and port to reach a set of Pods, even as Pods die and restart.

**Service Types:**

| Type | Reachable From | Use Case |
|---|---|---|
| `ClusterIP` | Only inside the cluster | Internal microservice communication |
| `NodePort` | External via NodeIP:Port | Dev/testing, not production |
| `LoadBalancer` | External via cloud LB | Production internet-facing apps |
| `ExternalName` | Maps to external DNS | Connecting to external databases |
| `Headless` | Direct Pod IPs (no VIP) | StatefulSets, direct pod discovery |

**YAML Example:**

```yaml
# Service YAML
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: production
spec:
  selector:
    app: my-app          # Targets Pods with this label
  ports:
    - protocol: TCP
      port: 80           # Service port
      targetPort: 8080   # Pod port
  type: ClusterIP
```

---

### Layer 5 — Deployments

**Definition:** A Deployment declares the desired state for a set of Pods. It manages a ReplicaSet under the hood and provides rolling updates, rollback, and pause/resume functionality.

**Why needed:** Without Deployment, if a Pod crashes, nobody restarts it. Deployment ensures your desired replica count is always maintained.

**YAML Example:**

```yaml
# Deployment YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # How many extra pods during update
      maxUnavailable: 1    # How many pods can be down during update
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: container1
          image: nginx:1.25
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
        - name: container2
          image: redis:7
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
```

**Useful Commands:**

```bash
kubectl rollout status deployment/my-app -n production
kubectl rollout history deployment/my-app -n production
kubectl rollout undo deployment/my-app -n production        # Rollback
kubectl rollout undo deployment/my-app --to-revision=2     # Rollback to specific version
```

---

### Layer 6 — ReplicaSet

**Definition:** A ReplicaSet ensures that a specified number of Pod replicas are running at any time. If a Pod dies, the ReplicaSet creates a new one.

**Why you rarely manage it directly:** Deployments manage ReplicaSets automatically. Direct ReplicaSet management means you lose rolling update + rollback capabilities.

**How Deployment uses ReplicaSets:**

```
Deployment (my-app)
  ├── ReplicaSet (my-app-7d9f8c6b4)   ← old version (0 replicas after update)
  └── ReplicaSet (my-app-5a3b2d1e9)   ← new version (3 replicas)
```

**Commands:**

```bash
kubectl get rs -n production
kubectl describe rs my-app-5a3b2d1e9 -n production
```

---

### Layer 7 — Endpoints & EndpointSlices

**Definition:** An Endpoint object is automatically created by Kubernetes for every Service. It holds the actual IP:Port pairs of the healthy Pods matching that Service's selector.

**Why it matters:** When you hit `my-service:80`, kube-proxy reads the Endpoints object to know which Pod IPs to route to.

```bash
kubectl get endpoints my-service -n production
# NAME         ENDPOINTS                         AGE
# my-service   10.0.1.5:8080,10.0.1.6:8080,...  2d
```

**EndpointSlices** (K8s 1.21+): More scalable alternative — splits large endpoint lists into slices of max 100 entries.

```bash
kubectl get endpointslices -n production
```

**Manual Endpoint (for external services):**

```yaml
# Service without selector (manual endpoints)
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  ports:
    - port: 5432
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-db
subsets:
  - addresses:
      - ip: 192.168.1.100    # External DB IP
    ports:
      - port: 5432
```

---

### Layer 8 — Pods

**Definition:** A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share the same network namespace (same IP, localhost), storage volumes, and lifecycle.

**Pod Lifecycle Phases:**

```
Pending → Running → Succeeded
                  ↘ Failed
                  ↘ Unknown
```

**YAML Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  namespace: production
  labels:
    app: my-app
spec:
  containers:
    - name: container1         # Main app
      image: nginx:1.25
      ports:
        - containerPort: 80
    - name: container2         # Sidecar: log forwarder
      image: fluentd:v1.16
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log
  volumes:
    - name: shared-logs
      emptyDir: {}
```

**Pod Design Patterns:**

| Pattern | Description | Example |
|---|---|---|
| Sidecar | Helper alongside main container | Log shipper next to app |
| Ambassador | Proxy local connections | DB proxy (e.g., ProxySQL) |
| Adapter | Normalize output from main container | Prometheus exporter |
| Init Container | Runs before main containers | DB migration, config fetch |

---

### Layer 9 — Containers (container1, container2)

**Definition:** Containers are OCI-compliant process bundles. In a Pod, they share:
- **Network namespace** (same IP, can communicate via localhost)
- **IPC namespace** (shared memory)
- **Volumes** (if explicitly mounted)

They do NOT share the filesystem by default.

```yaml
spec:
  initContainers:            # Runs first, must complete
    - name: init-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 1; done']
  containers:
    - name: container1       # Primary app
      image: myapp:v2
    - name: container2       # Sidecar
      image: envoy:v1.28
```

---

### Layer 10 — Container Images

**Definition:** A container image is an immutable, layered filesystem bundle (OCI format). It contains the app binary, dependencies, config, and a base OS layer.

**Image Naming:**

```
registry/repository:tag@digest
docker.io/library/nginx:1.25@sha256:abc123...
gcr.io/my-project/myapp:v2.1
```

**Image Pull Policies:**

| Policy | Behavior |
|---|---|
| `Always` | Always pull from registry |
| `IfNotPresent` | Use local cache if available (default for tagged) |
| `Never` | Only use local cache |

```yaml
containers:
  - name: app
    image: nginx:1.25
    imagePullPolicy: IfNotPresent
```

**Private Registry:**

```yaml
spec:
  imagePullSecrets:
    - name: registry-credentials
  containers:
    - name: app
      image: private.registry.io/myapp:v1
```

---

### Layer 11 — cgroups (Control Groups)

**Definition:** cgroups is a Linux kernel feature that limits, accounts for, and isolates resource usage (CPU, memory, disk I/O, network) of process groups. Kubernetes uses cgroups to enforce `requests` and `limits`.

**cgroup Hierarchy in Kubernetes:**

```
/sys/fs/cgroup/
  └── kubepods/
        ├── burstable/
        │     └── pod<pod-uid>/
        │           ├── container1/    ← cpu.shares, memory.limit_in_bytes
        │           └── container2/
        ├── besteffort/
        └── guaranteed/
```

**cgroup v1 vs v2:**

| Feature | cgroup v1 | cgroup v2 |
|---|---|---|
| Memory | Per-subsystem | Unified hierarchy |
| CPU | cpu + cpuacct subsystems | cpu controller |
| K8s support | Deprecated | Default K8s 1.25+ |

**How K8s maps Resources to cgroups:**

```
CPU request  → cpu.shares (proportional weight)
CPU limit    → cpu.cfs_quota_us (hard throttling)
Memory limit → memory.limit_in_bytes (triggers OOM kill if exceeded)
```

---

### Layer 12 — Linux Kernel

**Definition:** The foundation of everything. Kubernetes relies on several kernel features:

| Kernel Feature | Kubernetes Use |
|---|---|
| **Namespaces** (pid, net, mnt, uts, ipc, user) | Container isolation |
| **cgroups v2** | Resource limits |
| **iptables / nftables** | kube-proxy Service routing |
| **eBPF** | Modern networking (Cilium, Calico) |
| **overlayfs** | Container image layering |
| **seccomp** | System call filtering in pods |
| **AppArmor / SELinux** | Mandatory access control |
| **VXLAN / IPIP** | Pod overlay networking |

---

## 2. Namespaces: Auto-Created & Custom

### The 4 Automatic Namespaces

When you install Kubernetes, these namespaces are created automatically:

#### 1. `kube-system`

**Purpose:** Core Kubernetes system components live here.

**What runs here:**

```bash
kubectl get pods -n kube-system
# NAME                               READY   STATUS
# coredns-xxx                        1/1     Running    ← DNS
# etcd-master                        1/1     Running    ← State store
# kube-apiserver-master              1/1     Running    ← API
# kube-controller-manager-master     1/1     Running    ← Controllers
# kube-scheduler-master              1/1     Running    ← Scheduling
# kube-proxy-xxx                     1/1     Running    ← Networking
```

**YAML stored:** CoreDNS ConfigMap, kube-proxy ConfigMap, system RBAC.

**You should NEVER:** Delete or modify resources here unless you know exactly what you're doing.

---

#### 2. `kube-public`

**Purpose:** Public, readable by all users (even unauthenticated). Contains cluster info.

```bash
kubectl get configmap cluster-info -n kube-public -o yaml
# Contains: server address, CA certificate
```

**Use case:** `kubeadm join` reads from here to bootstrap new nodes.

---

#### 3. `kube-node-lease`

**Purpose:** Stores Lease objects — one per node. The kubelet on each node updates its Lease every few seconds (default: 10s) as a heartbeat signal.

```bash
kubectl get lease -n kube-node-lease
# NAME           HOLDER         AGE
# worker-node1   worker-node1   5d
# worker-node2   worker-node2   5d
```

**Why this exists (not in Node object):** Before Lease objects, the kubelet updated the Node object's `.status` for heartbeats. This caused heavy etcd write load. Lease objects are tiny and cheap to update.

**YAML stored:**

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: worker-node1
  namespace: kube-node-lease
spec:
  holderIdentity: worker-node1
  leaseDurationSeconds: 40
  renewTime: "2025-01-01T10:00:00.000000Z"
```

---

#### 4. `default`

**Purpose:** The fallback namespace. Any resource created without `-n <namespace>` lands here.

**Caution:** In production, you should almost never use `default`. It becomes a dumping ground.

---

### Namespaced vs Non-Namespaced Resources

```bash
kubectl api-resources --namespaced=true
# Pods, Services, Deployments, ConfigMaps, Secrets, ServiceAccounts...

kubectl api-resources --namespaced=false
# Nodes, PersistentVolumes, ClusterRoles, StorageClasses, Namespaces...
```

**Key insight:** Nodes, PersistentVolumes, and ClusterRoles are **cluster-scoped** — they exist at the cluster level, not inside any namespace.

---

## 3. Namespace Use Cases & Why You Need Them

### Definition

A Namespace is a virtual cluster inside a physical cluster. It provides a mechanism for isolating groups of resources. Names of resources only need to be unique within a namespace, not across namespaces.

### Core Use Case 1: Grouping by Team / Project

**Problem solved:** Without namespaces, all 500 microservices from 20 teams share one flat space. A `kubectl get pods` returns thousands of results. No team can work independently.

```bash
kubectl create namespace team-payments
kubectl create namespace team-auth
kubectl create namespace team-notifications

kubectl get pods -n team-payments
# Only payments team pods
```

---

### Core Use Case 2: Environment Isolation

**Scenario:** Dev, Staging, Production in the same cluster (cost saving for small teams).

```bash
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

**YAML for environment-specific deploy:**

```yaml
# production deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production    # ← namespace declared here
spec:
  replicas: 5
```

---

### Core Use Case 3: Customized Resource Allocation Per Namespace

**Problem solved:** Team A uses 90% of cluster CPU, starving Team B.

```yaml
# ResourceQuota for team-payments namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
```

---

### Core Use Case 4: RBAC Per Namespace (Security Boundary)

```yaml
# Allow dev team to only manage pods in their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-pod-manager
  namespace: team-payments
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

---

### Core Use Case 5: Per-Project Fine-Grained Billing

**Problem solved:** "Which project consumed how much cloud cost this month?"

- Tag all resources in namespace `team-payments` → cost report per namespace
- Tools: OpenCost, Kubecost — aggregate spend per namespace

---

### Core Use Case 6: Compliance & Assured Workloads

**Scenario:** PCI-DSS workloads (payment processing) must be isolated from general workloads.

```yaml
# NetworkPolicy: payments namespace cannot receive traffic from other namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: team-payments
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              compliance: pci-dss   # Only allow from same-compliance namespaces
```

---

### Core Use Case 7: Geo-Distributed Deployments

**Scenario:** EU cluster namespace `eu-west`, US cluster namespace `us-east` — different regions, different data residency rules.

Each cluster has its own namespaces mapping to regions, enabling region-specific policies, quotas, and data residency compliance.

---

### Core Use Case 8: Version Differentiation

**Scenario:** Running v1 and v2 of an API in parallel (Blue/Green or canary).

```bash
kubectl create namespace api-v1
kubectl create namespace api-v2

# Deploy v1 in api-v1 namespace
# Deploy v2 in api-v2 namespace
# Route 10% traffic to api-v2 via Ingress
```

---

## 4. When NOT to Use Namespaces (Problems You'll Face)

### Problem 1: Fine-Grained Billing Without Namespace Separation

If all teams share one namespace, you cannot separate costs per team. OpenCost/Kubecost aggregates by namespace — without it, billing is a black box.

**Solution:** One namespace per team or cost center.

---

### Problem 2: Cannot Differentiate Application Versions

If `api-v1` and `api-v2` are in the same namespace, label conflicts, selector collisions, and ConfigMap name clashes occur.

---

### Problem 3: No Compliance & Assurance Boundary

PCI-DSS, HIPAA, SOC2 require workload isolation. Namespaces enable NetworkPolicy, RBAC, and audit scoping. Without them, all workloads are reachable from each other.

---

### Problem 4: Multi-Tenancy Breaks Without Namespaces

Tenant A can accidentally (or maliciously) modify Tenant B's ConfigMaps, Secrets, or Services if they're in the same namespace.

---

### Problem 5: Observability Noise

Metrics, logs, and traces for 50 teams in one namespace = impossible to filter. Namespace-scoped dashboards in Grafana/Prometheus are only possible with namespace separation.

---

### Problem 6: Budget/Quota Total No. of Workloads

Without namespace-scoped ResourceQuotas, there's no mechanism to enforce "Team A can only run 20 pods max." One rogue deployment can consume all cluster resources.

---

### Problem 7: Compliance-Assured Workloads

Security scanning tools (Falco, OPA/Gatekeeper) apply policies at namespace level. Without namespaces, you cannot enforce different security postures for sensitive vs. non-sensitive workloads.

---

## 5. Load Balancer: 5 Algorithms

### When Kubernetes uses Load Balancing

A Kubernetes Service distributes traffic across healthy Pod replicas. The `kube-proxy` (or eBPF-based proxies like Cilium) implements the actual load balancing.

---

### Algorithm 1: Round Robin (Default)

**Definition:** Requests are distributed sequentially across all available Pods. Pod 1 gets request 1, Pod 2 gets request 2, Pod 3 gets request 3, back to Pod 1.

**How it works in K8s:** kube-proxy in iptables mode uses random selection (statistically equivalent to round robin at scale). IPVS mode implements true round-robin.

```bash
# Enable IPVS mode in kube-proxy
kubectl edit configmap kube-proxy -n kube-system
# Set mode: "ipvs"
```

**Use case:** Stateless services where every request is equal weight.

---

### Algorithm 2: Least Connections

**Definition:** New requests go to the Pod with the fewest active connections.

**Use case:** Long-lived connections (WebSockets, gRPC streaming) where some requests take much longer than others. Prevents one Pod from being overwhelmed.

```yaml
# Ingress NGINX annotation for least-conn
nginx.ingress.kubernetes.io/load-balance: "least_conn"
```

---

### Algorithm 3: IP Hash (Session Affinity / Sticky Sessions)

**Definition:** The client's IP address is hashed to always route to the same Pod. The same client always hits the same backend.

**Use case:** Shopping carts, session-based apps without a shared session store.

```yaml
# Service with session affinity
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600   # Session stickiness for 1 hour
```

---

### Algorithm 4: Weighted Round Robin

**Definition:** Pods are assigned different weights. Higher-weight Pods receive proportionally more requests. Used in canary deployments.

```yaml
# Canary: 90% to stable, 10% to canary using two Deployments
# stable: 9 replicas, canary: 1 replica → effectively 90/10 split
# (or use Istio VirtualService for explicit weights)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
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

---

### Algorithm 5: Random

**Definition:** Each request is routed to a randomly selected Pod. Simple and effective when all Pods are equally capable.

**Use case:** Stateless microservices with homogenous Pods and roughly equal request sizes. kube-proxy's iptables mode is effectively random selection.

---

## 6. Multi-Tenancy

### Definition

Multi-tenancy in Kubernetes means multiple independent teams, customers, or applications share a single Kubernetes cluster, with isolation between them.

### The Trust Question: "Do You Trust the Other Parties?"

This is the fundamental multi-tenancy design question:

| Trust Level | Model | Mechanism |
|---|---|---|
| **High trust** (internal teams) | Soft multi-tenancy | Namespaces + RBAC + ResourceQuota |
| **Low trust** (external customers) | Hard multi-tenancy | Separate clusters or vClusters |
| **Zero trust** (untrusted code) | Sandboxed | gVisor, Kata Containers |

---

### Soft Multi-Tenancy (Namespace-Based)

```yaml
# Per-tenant namespace with full isolation
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme
  labels:
    tenant: acme
    compliance: standard
---
# RBAC: Tenant only sees their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: tenant-acme
subjects:
  - kind: Group
    name: acme-developers
roleRef:
  kind: ClusterRole
  name: edit
---
# NetworkPolicy: Block cross-tenant traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  namespace: tenant-acme
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              tenant: acme
```

---

### Budget & Total No. of Workloads

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-acme-quota
  namespace: tenant-acme
spec:
  hard:
    pods: "50"                    # Max 50 pods
    requests.cpu: "10"            # Max 10 CPU cores requested
    requests.memory: "20Gi"       # Max 20Gi memory requested
    limits.cpu: "20"              # Max 20 CPU cores limit
    limits.memory: "40Gi"         # Max 40Gi memory limit
    persistentvolumeclaims: "10"  # Max 10 PVCs
    services.loadbalancers: "2"   # Max 2 LoadBalancer services
    services.nodeports: "0"       # No NodePort services
```

---

### Observability: Metrics, Tracing, Billing

**Prometheus with namespace labels:**

```yaml
# Scrape config: label metrics by namespace
- job_name: 'kubernetes-pods'
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_namespace]
      target_label: namespace
```

**Grafana dashboard:** Filter by `namespace=~"tenant-.*"` to get per-tenant CPU/memory graphs.

**Billing with OpenCost:**

```bash
# OpenCost API: cost per namespace
curl http://opencost.kube-system.svc:9003/allocation?window=7d&aggregate=namespace
```

---

## 7. CPU & Memory: Compressible vs Non-Compressible

### CPU — Compressible Resource

**Definition:** CPU is compressible because if a container exceeds its CPU limit, it gets **throttled** — it slows down but keeps running. No process is killed.

**Units:**

```
1 CPU     = 1000m (millicores) = 1 vCPU/Core/Hyperthread
100m      = 100 millicores = 0.1 CPU
500m      = 0.5 CPU

Math:
100m = 100/1000 = 0.1 CPU
10^3 = 1000m = 1 CPU
100 × 10 = 1000m → 100 × (1/1000) = 0.1 CPU
```

**CPU Request vs Limit:**

```yaml
resources:
  requests:
    cpu: "100m"    # Guaranteed: scheduler reserves 0.1 CPU
  limits:
    cpu: "500m"    # Ceiling: container throttled above 0.5 CPU
```

**What happens at the kernel level:**

```
cpu.shares = 102 (proportional, based on request)
cpu.cfs_quota_us = 50000 (50ms per 100ms period = 50% = 0.5 CPU)
cpu.cfs_period_us = 100000 (100ms)
```

---

### Memory — Non-Compressible Resource

**Definition:** Memory is non-compressible because if a container exceeds its memory limit, it is **OOM-killed** (killed immediately by the kernel). There is no throttling — the process dies.

**Units:**

| Suffix | Meaning | Bytes |
|---|---|---|
| `100M` | 100 Megabytes (SI) | 100,000,000 |
| `100Mi` | 100 Mebibytes (Binary) | 104,857,600 |
| `1G` | 1 Gigabyte | 1,000,000,000 |
| `1Gi` | 1 Gibibyte | 1,073,741,824 |

**Always use `Mi` and `Gi`** in Kubernetes — binary units match actual memory allocation.

```yaml
resources:
  requests:
    memory: "128Mi"   # Guaranteed: scheduler reserves 128MiB
  limits:
    memory: "256Mi"   # If container uses >256MiB → OOM Kill
```

---

### OOM Kill Explained

**OOM (Out of Memory) Kill** is triggered by the Linux kernel's OOM killer when the system (or cgroup) runs out of memory.

**Sequence:**

```
1. Container requests memory beyond limit
2. cgroup memory.limit_in_bytes exceeded
3. Linux kernel OOM killer fires
4. Container process killed with SIGKILL (exit code 137)
5. kubelet restarts the container (if restartPolicy: Always)
```

**Debugging:**

```bash
kubectl describe pod my-pod -n production
# Containers:
#   app:
#     Last State: Terminated
#       Reason: OOMKilled
#       Exit Code: 137

kubectl get events -n production --field-selector reason=OOMKilling
```

**Prevention:** Set memory limits higher than peak usage + set JVM heap explicitly for Java apps.

---

## 8. QoS (Quality of Service)

**Definition:** Kubernetes assigns one of three QoS classes to every Pod based on its resource requests and limits. This determines eviction priority when a node is under memory pressure.

### QoS Classes Table

| Class | Condition | Eviction Priority | Behavior |
|---|---|---|---|
| **Guaranteed** | `requests == limits` for ALL containers for BOTH cpu & memory | Last to be evicted | Most predictable; gets exactly what it asks for |
| **Burstable** | At least one container has `requests != limits` OR only requests set | Middle priority | Can use more than requested, up to limits |
| **BestEffort** | NO requests or limits set at all | First to be evicted | Uses whatever is available; zero guarantees |

### Guaranteed QoS (Best)

```yaml
containers:
  - name: app
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "500m"      # ← Must equal request
        memory: "256Mi"  # ← Must equal request
```

**Use case:** Production databases, payment services, anything that cannot be killed.

---

### Burstable QoS (Most Common)

```yaml
containers:
  - name: app
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"      # ← Higher than request = Burstable
        memory: "512Mi"
```

**Use case:** Web servers, APIs — normally use modest resources but can burst.

---

### BestEffort QoS (Avoid in Production)

```yaml
containers:
  - name: app
    # No resources section at all
```

**Use case:** Batch jobs, dev environments, throw-away tasks.

---

## 9. Resource Quotas & LimitRanges

### ResourceQuota — Namespace Level

**Definition:** ResourceQuota sets maximum aggregate resource consumption per namespace. It prevents a namespace from consuming more than its allocated share.

**`spec.hard` — Hard Limits (Enforced):**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: team-payments
spec:
  hard:
    # Compute
    requests.cpu: "8"           # Total CPU requests across all pods
    requests.memory: "16Gi"     # Total memory requests
    limits.cpu: "16"            # Total CPU limits
    limits.memory: "32Gi"       # Total memory limits
    # Objects
    pods: "30"
    services: "10"
    secrets: "20"
    configmaps: "20"
    persistentvolumeclaims: "10"
    # Service types
    services.loadbalancers: "2"
    services.nodeports: "0"     # Block NodePort services
```

**Note:** There is no `spec.soft` in ResourceQuota. The `hard` field IS the enforcement. (Soft limits are a concept in other systems like Linux ulimits, not K8s ResourceQuota.)

---

### LimitRange — Pod/Container Level

**Definition:** LimitRange sets default, minimum, and maximum resource constraints for individual Pods and Containers within a namespace. It operates at the container level, not the namespace aggregate level.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: team-payments
spec:
  limits:
    - type: Container
      default:              # Default LIMIT if not specified
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:       # Default REQUEST if not specified
        cpu: "100m"
        memory: "128Mi"
      max:                  # Maximum allowed limit
        cpu: "2"
        memory: "2Gi"
      min:                  # Minimum allowed request
        cpu: "50m"
        memory: "64Mi"

    - type: Pod             # Pod-level max (sum of all containers)
      max:
        cpu: "4"
        memory: "4Gi"

    - type: PersistentVolumeClaim
      max:
        storage: "50Gi"
      min:
        storage: "1Gi"
```

**ResourceQuota vs LimitRange:**

| Feature | ResourceQuota | LimitRange |
|---|---|---|
| Scope | Namespace aggregate | Per container/pod |
| Enforces | Total namespace consumption | Individual resource values |
| Default injection | No | Yes (injects defaults) |
| Used to | Prevent namespace over-use | Prevent containers with no limits |

---

## 10. Cost Optimization with OpenCost

### What is OpenCost?

OpenCost (https://www.opencost.io) is an open-source, CNCF sandbox project that provides real-time cost monitoring and allocation for Kubernetes workloads. It breaks down cloud spend by namespace, deployment, pod, and label.

### Installation

```bash
# Install OpenCost in kube-system
kubectl apply --server-side -f https://raw.githubusercontent.com/opencost/opencost/develop/kubernetes/opencost.yaml -n kube-system

# Access UI
kubectl port-forward -n kube-system svc/opencost 9090:9090
```

### Cost Allocation Query Examples

```bash
# Cost by namespace (last 7 days)
curl "http://localhost:9003/allocation?window=7d&aggregate=namespace"

# Cost by deployment
curl "http://localhost:9003/allocation?window=1d&aggregate=deployment"

# Cost by label (e.g., team label)
curl "http://localhost:9003/allocation?window=30d&aggregate=label:team"
```

### Optimization Strategies

1. **Right-size over-provisioned pods** — OpenCost shows CPU/memory efficiency %
2. **Identify idle namespaces** — dev namespaces with near-zero usage
3. **Spot/preemptible instances for BestEffort workloads** — batch jobs don't need on-demand VMs
4. **Consolidate small namespaces** — administrative overhead vs. cost savings trade-off
5. **Terminate unused PVCs** — storage cost often overlooked

---

## 11. Taints & Tolerations

### Definition

**Taint** (applied to Nodes): "I am a special node. Unless you explicitly tolerate my taint, you cannot schedule here."

**Toleration** (applied to Pods): "I can tolerate node taints. Schedule me even on tainted nodes."

Together they enable **dedicated node pools** — e.g., GPU nodes only for ML workloads, spot nodes only for batch jobs.

---

### Taint Effects

| Effect | Behavior |
|---|---|
| `NoSchedule` | New pods without toleration will NOT be scheduled here |
| `PreferNoSchedule` | Scheduler tries to avoid, but allows if no alternatives |
| `NoExecute` | Existing pods without toleration are EVICTED; new ones not scheduled |

---

### Taint Commands

```bash
# Add taint to a node
kubectl taint node worker-node1 gpu=true:NoSchedule

# Add NoExecute taint (evicts existing non-tolerating pods)
kubectl taint node worker-node1 maintenance=true:NoExecute

# Remove taint (note the trailing minus)
kubectl taint node worker-node1 gpu=true:NoSchedule-

# View taints on a node
kubectl describe node worker-node1 | grep Taints
```

---

### Toleration YAML

```yaml
# Pod that tolerates the gpu=true:NoSchedule taint
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training-job
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: trainer
      image: tensorflow/tensorflow:latest-gpu
```

---

### nodeName: Bypassing the Scheduler

**`nodeName`** is the most direct pod placement — it bypasses the scheduler entirely and places the pod on the exact named node.

```bash
# Generate pod YAML with dry-run
kubectl run mypod --image=redis --dry-run=client -o yaml > 1.yaml
```

```yaml
# 1.yaml — using nodeName to bypass scheduler
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  nodeName: worker-node1    # ← Bypasses scheduler, goes directly here
  containers:
    - name: redis
      image: redis
```

**Warning:** If the node doesn't exist or is full, the Pod stays `Pending` forever — no rescheduling happens.

---

### Real-World Taint Use Cases

| Scenario | Taint | Toleration |
|---|---|---|
| GPU nodes | `nvidia.com/gpu:NoSchedule` | ML training pods only |
| Spot/Preemptible nodes | `cloud.google.com/gke-spot:NoSchedule` | Batch jobs, stateless workers |
| Maintenance | `node.kubernetes.io/unschedulable:NoSchedule` | Auto-applied on drain |
| Dedicated master | `node-role.kubernetes.io/control-plane:NoSchedule` | System pods (CoreDNS) |
| High-memory nodes | `mem=high:NoSchedule` | Memory-intensive workloads |

---

## 12. Node Selector, Affinity & Anti-Affinity

### nodeSelector (Simple)

**Definition:** The simplest node selection mechanism. A Pod can only be scheduled on nodes with matching labels.

```bash
# Label a node
kubectl label node worker-node1 disktype=ssd
kubectl label node worker-node2 disktype=hdd
```

```yaml
spec:
  nodeSelector:
    disktype: ssd    # Pod only schedules on SSD nodes
  containers:
    - name: db
      image: postgres:15
```

**Limitation:** Only supports equality (key=value). Cannot express "prefer SSD but allow HDD if no SSD available."

---

### Node Affinity (Advanced nodeSelector)

**Definition:** More expressive version of nodeSelector. Supports `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` operators, and both hard (`required`) and soft (`preferred`) rules.

**Two types:**
- `requiredDuringSchedulingIgnoredDuringExecution` → Hard rule (pod won't schedule if not satisfied)
- `preferredDuringSchedulingIgnoredDuringExecution` → Soft rule (scheduler tries but won't block)

```yaml
spec:
  affinity:
    nodeAffinity:
      # HARD: Must be in us-east or us-west region
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/region
                operator: In
                values:
                  - us-east-1
                  - us-west-2
      # SOFT: Prefer SSD nodes (weight 1-100)
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
```

---

### Pod Affinity

**Definition:** Schedule a Pod **near** other Pods (same node or same zone). Useful for latency-sensitive co-location.

```yaml
spec:
  affinity:
    podAffinity:
      # HARD: Must run on same node as pods labeled app=cache
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - cache
          topologyKey: kubernetes.io/hostname  # "same node"
          # Use topology.kubernetes.io/zone for "same availability zone"
```

**Use case:** App pod next to its Redis cache pod on the same node = zero network latency.

---

### Pod Anti-Affinity

**Definition:** Schedule a Pod **away from** other Pods. Ensures Pods don't concentrate on the same node (HA spread).

```yaml
spec:
  affinity:
    podAntiAffinity:
      # HARD: No two replicas of this app on the same node
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - my-app
          topologyKey: kubernetes.io/hostname
```

**Use case:** If all 3 replicas land on 1 node and that node goes down, all 3 die. Anti-affinity spreads them across nodes.

---

## 13. Topology Spread Constraints

### Definition

Topology Spread Constraints control how Pods are distributed across **topology domains** (nodes, zones, regions). It's more powerful than anti-affinity for even distribution.

**Why needed:** Anti-affinity says "not on the same node" but doesn't ensure even distribution. Topology Spread ensures pods are spread as evenly as possible.

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                        # Max allowed imbalance
      topologyKey: kubernetes.io/hostname  # Spread across nodes
      whenUnsatisfiable: DoNotSchedule  # Hard (or ScheduleAnyway for soft)
      labelSelector:
        matchLabels:
          app: my-app
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone  # Also spread across AZs
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: my-app
```

**maxSkew:** Maximum difference in Pod count between any two topology domains. `maxSkew: 1` means at most a difference of 1 pod between the most-loaded and least-loaded node/zone.

---

### Container Failure + Topology Spread

When a container fails on one node:

1. Pod enters `CrashLoopBackOff`
2. If pod is deleted, ReplicaSet creates a new one
3. Topology Spread Constraints guide the scheduler to place it on the node with the fewest replicas
4. Result: auto-rebalancing after node failures

---

## 14. New Pod Placement: Complete Picture

When a new Pod is created, the scheduler follows this decision tree:

```
New Pod Created
      │
      ▼
1. FILTER Phase (Hard constraints — eliminate ineligible nodes)
   ├── Node has enough CPU/Memory? (based on requests)
   ├── Node taints tolerated by pod?
   ├── Node selector / required affinity matches?
   ├── Required topology spread satisfied?
   ├── Pod anti-affinity hard rules satisfied?
   └── Node is Ready/Schedulable?
      │
      ▼
2. SCORE Phase (Soft preferences — rank remaining nodes)
   ├── Preferred node affinity weight
   ├── Preferred pod affinity weight
   ├── Topology spread preferred
   ├── Least allocated resources (bin packing vs spreading)
   └── Image locality (is image already pulled?)
      │
      ▼
3. BIND Phase
   └── Highest score wins → kubelet on that node starts the pod
```

**Complete YAML with all placement mechanisms:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-ha-app
  namespace: production
spec:
  replicas: 6
  selector:
    matchLabels:
      app: my-ha-app
  template:
    metadata:
      labels:
        app: my-ha-app
    spec:
      # 1. Node Selector (simple label match)
      nodeSelector:
        disktype: ssd

      # 2. Node Affinity (complex node rules)
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: topology.kubernetes.io/region
                    operator: In
                    values: [us-east-1, eu-west-1]
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node-type
                    operator: In
                    values: [on-demand]

        # 3. Pod Anti-Affinity (spread across nodes)
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: my-ha-app
              topologyKey: kubernetes.io/hostname

      # 4. Topology Spread (even distribution across zones)
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-ha-app

      # 5. Tolerations (allow scheduling on tainted nodes)
      tolerations:
        - key: "environment"
          operator: "Equal"
          value: "production"
          effect: "NoSchedule"

      containers:
        - name: container1
          image: myapp:v2
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
```

---

## 15. Interview Questions & Answers

### Cluster & Architecture

**Q1: What is the difference between a Node and a Pod?**

> A Node is a physical or virtual machine (infrastructure). A Pod is a Kubernetes abstraction — the smallest deployable unit wrapping one or more containers. Many Pods run on one Node. A Node can exist without Pods, but a Pod must run on a Node.

---

**Q2: What happens when the kube-scheduler pod crashes?**

> Existing running Pods are unaffected — they keep running. However, no NEW Pods can be scheduled until the scheduler restarts. In a highly-available control plane (3 master nodes), another scheduler instance takes over immediately.

---

**Q3: What is etcd and what happens if it goes down?**

> etcd is the distributed key-value store that is Kubernetes' single source of truth. If etcd goes down: you cannot create/update/delete resources, kubectl commands fail, but running Pods continue running (kubelet works independently). This is why etcd is always deployed in odd numbers (3 or 5) for quorum.

---

### Namespaces

**Q4: Can two services in different namespaces have the same name?**

> Yes. `my-service` in `namespace-a` and `my-service` in `namespace-b` are completely separate resources. DNS resolves them as `my-service.namespace-a.svc.cluster.local` and `my-service.namespace-b.svc.cluster.local`.

---

**Q5: Can a Pod in namespace-a talk to a Pod in namespace-b?**

> Yes, by default. Kubernetes does NOT enforce network isolation between namespaces out of the box. You must create `NetworkPolicy` objects to restrict cross-namespace traffic. Without NetworkPolicy, all pods can communicate across namespaces.

---

**Q6: What is kube-node-lease used for?**

> It stores lightweight Lease objects — one per node — that the kubelet updates every ~10 seconds as a heartbeat. Before Lease objects (pre-K8s 1.14), heartbeats updated the Node object's status directly, causing heavy load on etcd. Lease objects are tiny (< 1KB) and reduce etcd I/O significantly.

---

### Resources

**Q7: What is the difference between CPU requests and CPU limits?**

> **Request** is the guaranteed minimum — the scheduler uses requests to decide if a node has enough capacity to place a Pod. **Limit** is the ceiling — if a container tries to use more CPU than its limit, it gets throttled (slowed down). The container keeps running but at reduced speed. For memory, exceeding the limit triggers an OOM kill.

---

**Q8: A pod is OOMKilled repeatedly. What do you do?**

> 1. `kubectl describe pod <name>` → confirm OOMKilled in Last State
> 2. Check actual memory usage: `kubectl top pod <name>`
> 3. Increase memory limit in the Deployment spec
> 4. For Java apps: set `-Xmx` JVM heap to 75% of the limit
> 5. Check for memory leaks with profiling if usage grows unboundedly

---

**Q9: What is QoS and which class gets evicted first?**

> QoS (Quality of Service) determines Pod eviction order when a node runs low on memory. **BestEffort** (no requests/limits) is evicted first. **Burstable** (requests < limits) is evicted second. **Guaranteed** (requests == limits) is evicted last. For critical production workloads, always set requests == limits for Guaranteed QoS.

---

**Q10: What is the difference between ResourceQuota and LimitRange?**

> **ResourceQuota** caps the total aggregate resource consumption across all Pods in a namespace (e.g., namespace total CPU ≤ 8 cores). **LimitRange** sets constraints on individual containers (e.g., each container must have a memory limit between 64Mi and 2Gi). LimitRange also injects default requests/limits for containers that don't specify them.

---

### Scheduling

**Q11: What is the difference between Taint+Toleration and Node Affinity?**

> **Taint+Toleration** is a "push" mechanism — the node pushes away pods that can't tolerate it. Used for dedicated node pools (GPU, spot). **Node Affinity** is a "pull" mechanism — the pod pulls toward specific nodes. Used when a pod has a preference or requirement for certain node characteristics. They are often used together.

---

**Q12: If I set `nodeName` in a Pod spec, what happens?**

> The Pod is bound directly to that node, bypassing the scheduler entirely. No filter or score phases run. If the node is full, NotReady, or tainted with NoExecute, the Pod stays Pending and kubelet will never retry on a different node. `nodeName` should be used only for debugging or system-level pods.

---

**Q13: What is Topology Spread Constraint and how is it better than Pod Anti-Affinity?**

> Pod Anti-Affinity with `requiredDuringScheduling` and `topologyKey: hostname` says "no two pods on the same node" — but doesn't prevent 5 pods on node-A and 1 on node-B. Topology Spread Constraint with `maxSkew: 1` enforces EVEN distribution, ensuring the difference between the most and least populated node/zone is at most 1.

---

### Load Balancing & Services

**Q14: How does a Kubernetes Service route traffic to Pods?**

> A Service has a stable ClusterIP and DNS name. When traffic arrives at the Service, kube-proxy (running on each node) maintains iptables/IPVS rules that DNAT (destination NAT) the traffic to one of the healthy Pod IPs listed in the Endpoints object. The Endpoints object is automatically updated by the Endpoint Controller as Pods come and go.

---

**Q15: What is the difference between ClusterIP, NodePort, and LoadBalancer?**

> **ClusterIP**: Virtual IP only reachable inside the cluster. Default type. For internal service-to-service communication.
> **NodePort**: Exposes service on a static port (30000-32767) on every node's IP. Reachable from outside. Not ideal for production (exposes all nodes).
> **LoadBalancer**: Provisions a cloud provider load balancer (AWS ALB/NLB, GCP LB) with a public IP. The recommended way to expose production services.

---

**Q16: When would you use a Headless Service?**

> A Headless Service (clusterIP: None) skips the VIP and instead DNS returns the actual Pod IPs directly. Used with StatefulSets where each Pod needs its own stable DNS identity (pod-0.service, pod-1.service), and for client-side load balancing where the client picks the Pod directly (Cassandra, Elasticsearch, Kafka).

---

### Multi-Tenancy

**Q17: What is the difference between soft and hard multi-tenancy in Kubernetes?**

> **Soft multi-tenancy**: Namespace isolation with RBAC, ResourceQuota, NetworkPolicy. Workloads share the same kernel and node. Trust the tenants (internal teams). Cost-efficient.
> **Hard multi-tenancy**: Separate clusters per tenant, or virtual clusters (vCluster), or sandboxed runtimes (gVisor/Kata). Workloads have strong isolation boundaries. For untrusted tenants (SaaS customers).

---

**Q18: How does OpenCost help with Kubernetes cost optimization?**

> OpenCost integrates with cloud billing APIs and Prometheus to calculate the actual cost of each Pod, Deployment, and Namespace in real-time. It allocates costs by CPU request, memory request, storage, and network usage. Teams can identify over-provisioned pods (high limit vs actual usage), idle namespaces, and expensive spot vs on-demand ratios, then right-size or consolidate to reduce cloud spend.

---

*This README covers the complete Kubernetes architecture from kernel to cluster, with production-grade YAML examples, use cases, and 18 interview Q&As. For the latest updates, refer to the [official Kubernetes documentation](https://kubernetes.io/docs/).*
