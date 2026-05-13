# ☸️ Kubernetes Deep Dive: Authentication, Authorization & Volumes

> A complete reference guide covering AuthN, AuthZ (RBAC), Volumes, Dynamic Provisioning, error handling, and interview Q&A.

---

## Table of Contents

1. [Authentication (AuthN) Overview](#1-authentication-authn-overview)
2. [kube-apiserver Authentication Flow](#2-kube-apiserver-authentication-flow)
3. [K8s Manager vs Non-K8s Manager](#3-k8s-manager-vs-non-k8s-manager)
4. [K8s Manager — InCluster Apps & kubectl](#4-k8s-manager--incluster-apps--kubectl)
5. [Non-K8s Manager — External Users, SA at Runtime](#5-non-k8s-manager--external-users-sa-at-runtime)
6. [Authentication Methods](#6-authentication-methods)
7. [kubeconfig File & Contexts](#7-kubeconfig-file--contexts)
8. [kubectl ServiceAccount & Token Commands](#8-kubectl-serviceaccount--token-commands)
9. [Context Use Cases & Permissions](#9-context-use-cases--permissions)
10. [The Noisy Neighbor Problem](#10-the-noisy-neighbor-problem)
11. [Authorization (AuthZ) Overview](#11-authorization-authz-overview)
12. [AuthZ Modes & Static Manifest Paths](#12-authz-modes--static-manifest-paths)
13. [RBAC — 4 Core Objects](#13-rbac--4-core-objects)
14. [RBAC Best Practices](#14-rbac-best-practices)
15. [Subjects, Verbs & Resources](#15-subjects-verbs--resources)
16. [RBAC YAML Manifests](#16-rbac-yaml-manifests)
17. [Verifying Permissions](#17-verifying-permissions)
18. [HTTP Error Codes — Causes & Solutions](#18-http-error-codes--causes--solutions)
19. [Volumes — Why, Types & Categories](#19-volumes--why-types--categories)
20. [PV / PVC — Modes, Policies & Use Cases](#20-pv--pvc--modes-policies--use-cases)
21. [Dynamic Provisioning & StorageClass](#21-dynamic-provisioning--storageclass)
22. [Interview Questions & Answers](#22-interview-questions--answers)

---

## 1. Authentication (AuthN) Overview

### Why AuthN?

Every request to the Kubernetes API must prove **who** it is before anything else happens. Without authentication, any process on the network could create, delete, or read any resource in the cluster — a critical security gap.

### Definition

**Authentication (AuthN)** is the process of verifying the identity of an entity (user, service account, or system component) that is trying to access the Kubernetes API server.

### Use Cases

| Scenario | Identity Type |
|---|---|
| Developer running `kubectl` | Human user (x.509 cert or OIDC) |
| Pod calling the K8s API | ServiceAccount (JWT token) |
| CI/CD pipeline deploying apps | ServiceAccount token or kubeconfig |
| External operator/tool | Bearer token or x.509 cert |

---

## 2. kube-apiserver Authentication Flow

```
Client Request
      │
      ▼
┌─────────────────────────────────────────────────────┐
│                  kube-apiserver                     │
│                                                     │
│  1. TLS Termination (always HTTPS)                  │
│  2. Authentication Plugin Chain                     │
│     ├── x.509 Client Certificates                   │
│     ├── Static Token File                           │
│     ├── Bootstrap Tokens                            │
│     ├── ServiceAccount Tokens (JWT)                 │
│     ├── OpenID Connect (OIDC)                       │
│     └── Webhook Token Auth                          │
│  3. Identity resolved → UserInfo{name, groups, uid} │
└─────────────────────────────────────────────────────┘
      │
      ▼
   etcd (stores cluster state — NOT used for auth checks,
         only stores secrets/resources AFTER authn passes)
```

### kube-apiserver ↔ etcd Authentication

- **etcd** stores all Kubernetes object state (Pods, Secrets, ConfigMaps, etc.)
- Communication between `kube-apiserver` and `etcd` is secured via **mutual TLS (mTLS)**
- The kube-apiserver presents its own client certificate to etcd
- Flags used:
  ```
  --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
  --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
  --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
  ```
- etcd itself does NOT perform K8s AuthN — it only accepts connections from kube-apiserver

### kubeconfig Role in Authentication Checking

- `~/.kube/config` tells `kubectl` **which cluster, user, and namespace** to use
- The credentials inside (certificate, token) are sent as part of the HTTPS request
- kube-apiserver validates these credentials against its configured auth plugins
- kube-apiserver does **not** read kubeconfig directly — clients use it to build valid requests

---

## 3. K8s Manager vs Non-K8s Manager

```
                    kube-apiserver
                         │
          ┌──────────────┴──────────────┐
          │                             │
   K8s Manager                  Non-K8s Manager
  (in-cluster)                  (external / human)
          │                             │
  ┌───────┴────────┐           ┌────────┴────────┐
  │                │           │                 │
InCluster   kubectl         External          ServiceAccount
   App      cluster info     Users           injected at runtime
  (SA token   (kubeconfig)  (kubeconfig /     into pod via env
  auto-       used from     OIDC / x.509)     or volume mount
  mounted)    local machine)
```

### K8s Manager

Entities that are **part of the cluster** or run **inside the cluster**:

- InCluster applications (Pods with ServiceAccounts)
- Operators and controllers
- `kubectl` with a valid kubeconfig pointing to the cluster

### Non-K8s Manager

Entities that are **outside the cluster**:

- Human developers using `kubectl` or dashboards
- External automation tools (Terraform, ArgoCD server running outside)
- CI/CD systems with injected credentials

---

## 4. K8s Manager — InCluster Apps & kubectl

### InCluster Application (Auto-mounted ServiceAccount)

When a Pod runs inside the cluster, Kubernetes **automatically mounts** a ServiceAccount token:

```
/var/run/secrets/kubernetes.io/serviceaccount/
    ├── token        ← JWT token
    ├── ca.crt       ← cluster CA certificate
    └── namespace    ← current namespace
```

Application code uses this to call the API:

```python
# Python example using in-cluster config
from kubernetes import client, config
config.load_incluster_config()   # reads the auto-mounted token
v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod(namespace="default")
```

### kubectl cluster-info

```bash
# Show cluster endpoint and DNS info
kubectl cluster-info

# Example output:
# Kubernetes control plane is running at https://192.168.1.100:6443
# CoreDNS is running at https://192.168.1.100:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Dump full cluster debug info
kubectl cluster-info dump
```

### For Applications

```bash
# Check which ServiceAccount a pod uses
kubectl get pod <pod-name> -o jsonpath='{.spec.serviceAccountName}'

# List all ServiceAccounts
kubectl get serviceaccounts -n <namespace>

# Describe a ServiceAccount
kubectl describe serviceaccount <sa-name> -n <namespace>
```

---

## 5. Non-K8s Manager — External Users, SA at Runtime

### External Users

External users authenticate via:
- **kubeconfig** file with embedded certificates or tokens
- **OIDC** tokens from an identity provider (Google, Okta, Dex)
- **x.509 certificates** signed by the cluster CA

### ServiceAccount Injected into Pod at Runtime

Sometimes you need to attach a specific ServiceAccount or token to a pod dynamically:

```yaml
# pod-with-custom-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-custom-sa   # ← inject SA at pod definition
  automountServiceAccountToken: true  # default is true
  containers:
    - name: app
      image: myapp:latest
```

**Injecting a token as an environment variable at runtime:**

```yaml
env:
  - name: K8S_TOKEN
    valueFrom:
      secretKeyRef:
        name: my-sa-token-secret
        key: token
```

**Projected volume for short-lived tokens (recommended):**

```yaml
volumes:
  - name: token-vol
    projected:
      sources:
        - serviceAccountToken:
            audience: my-api
            expirationSeconds: 3600
            path: token
```

---

## 6. Authentication Methods

### 6.1 Webhook Token Authentication

- kube-apiserver sends the token to an **external HTTP endpoint**
- The webhook service validates the token and returns a `UserInfo` object
- Used for integrating custom identity providers

```
kubectl ──token──► kube-apiserver ──POST /webhook──► Auth Service
                                  ◄── {username, groups} ──────────
```

**Flag:**
```
--authentication-token-webhook-config-file=/etc/kubernetes/webhook-auth.yaml
```

**Webhook config:**
```yaml
apiVersion: v1
kind: Config
clusters:
  - name: webhook-auth
    cluster:
      server: https://auth.example.com/authenticate
users:
  - name: kube-apiserver
    user:
      client-certificate: /path/to/cert
      client-key: /path/to/key
contexts:
  - name: webhook
    context:
      cluster: webhook-auth
      user: kube-apiserver
current-context: webhook
```

---

### 6.2 Static Token File

- A simple CSV file with `token,user,uid,group` entries
- Loaded at kube-apiserver startup — **requires restart to update**
- Not recommended for production

**Flag:**
```
--token-auth-file=/etc/kubernetes/known_tokens.csv
```

**known_tokens.csv:**
```
mytoken123,alice,uid001,group1
devtoken456,bob,uid002,developers
```

**Usage:**
```bash
curl -H "Authorization: Bearer mytoken123" https://<apiserver>/api/v1/pods
```

---

### 6.3 x.509 Client Certificates

- Most common method for admin/cluster-component authentication
- Client presents a certificate signed by the cluster CA
- The **CommonName (CN)** becomes the username; **Organization (O)** becomes the group

**Generate a user certificate:**
```bash
# 1. Generate private key
openssl genrsa -out alice.key 2048

# 2. Create a Certificate Signing Request (CSR)
openssl req -new -key alice.key -out alice.csr \
  -subj "/CN=alice/O=developers"

# 3. Sign with the cluster CA
openssl x509 -req -in alice.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial -out alice.crt -days 365

# 4. Add to kubeconfig
kubectl config set-credentials alice \
  --client-certificate=alice.crt \
  --client-key=alice.key
```

---

### 6.4 OpenID Connect (OIDC)

- Enterprise-grade federation with external identity providers (Google, Azure AD, Okta, Keycloak)
- kube-apiserver validates the JWT token using the provider's public keys

```
User ──login──► OIDC Provider ──ID Token (JWT)──► kubectl
                                                     │
kubectl ──Authorization: Bearer <JWT>──► kube-apiserver
                                              │
                              validates JWT signature & claims
```

**kube-apiserver flags:**
```
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=kubernetes
--oidc-username-claim=email
--oidc-groups-claim=groups
--oidc-ca-file=/etc/kubernetes/oidc-ca.crt
```

---

### 6.5 Custom Authentication

- Implement a custom authenticator as a webhook or API server extension
- Use cases: internal SSO, legacy auth systems, hardware tokens
- Must return a `TokenReview` response

---

## 7. kubeconfig File & Contexts

### File Location

```
~/.kube/config              ← default location
$KUBECONFIG                 ← override with environment variable
/etc/kubernetes/admin.conf  ← cluster admin config (on control plane)
```

### Structure

```yaml
apiVersion: v1
kind: Config

# ─── CLUSTERS ──────────────────────────────────────────────
clusters:
  - name: production-cluster
    cluster:
      server: https://prod-api.example.com:6443
      certificate-authority-data: <base64-ca-cert>
  - name: staging-cluster
    cluster:
      server: https://staging-api.example.com:6443
      certificate-authority-data: <base64-ca-cert>

# ─── NAMESPACES (set per context) ──────────────────────────

# ─── USERS ─────────────────────────────────────────────────
users:
  - name: alice
    user:
      client-certificate-data: <base64-cert>
      client-key-data: <base64-key>
  - name: bob-token
    user:
      token: eyJhbGciOiJSUzI1NiIsImtpZCI...

# ─── CONTEXTS ──────────────────────────────────────────────
contexts:
  - name: alice-prod
    context:
      cluster: production-cluster
      namespace: team-a
      user: alice
  - name: bob-staging
    context:
      cluster: staging-cluster
      namespace: default
      user: bob-token

# ─── CURRENT CONTEXT ───────────────────────────────────────
current-context: alice-prod
```

### Context = Cluster + Namespace + User

| Component | Purpose |
|---|---|
| `cluster` | Which API server to connect to |
| `namespace` | Default namespace for commands |
| `user` | Which credentials to use |

---

## 8. kubectl ServiceAccount & Token Commands

### Create a ServiceAccount

```bash
# Create a ServiceAccount
kubectl create serviceaccount my-app-sa -n my-namespace

# Create with YAML
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-namespace
EOF
```

### Create a Token

```bash
# Create a short-lived token (expires in 1 hour by default)
TOKEN=$(kubectl create token my-app-sa -n my-namespace)

# Create a token with custom expiry (e.g., 24 hours)
TOKEN=$(kubectl create token my-app-sa -n my-namespace --duration=86400s)

# Verify the token content (JWT decode)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
```

### Get Current Context

```bash
# Show current context name
kubectl config current-context

# Show full current context details
kubectl config view --minify
```

### Set Credentials with Token

```bash
# Add token-based credentials to kubeconfig
kubectl config set-credentials my-app-user --token=$TOKEN
```

### Set Context

```bash
# Create a new context linking cluster + user + namespace
kubectl config set-context my-app-context \
  --cluster=my-cluster \
  --user=my-app-user \
  --namespace=my-namespace

# Switch to the new context
kubectl config use-context my-app-context

# Verify
kubectl config current-context
```

### Full Workflow Example

```bash
# Step 1: Create ServiceAccount
kubectl create serviceaccount ci-deployer -n production

# Step 2: Generate token
TOKEN=$(kubectl create token ci-deployer -n production --duration=3600s)

# Step 3: Set credentials
kubectl config set-credentials ci-deployer-user --token=$TOKEN

# Step 4: Set context
kubectl config set-context ci-prod-context \
  --cluster=production \
  --user=ci-deployer-user \
  --namespace=production

# Step 5: Switch context
kubectl config use-context ci-prod-context

# Step 6: Verify
kubectl get pods  # runs as ci-deployer in production namespace
```

---

## 9. Context Use Cases & Permissions

### Context-Based Permission Scenarios

| Context | Cluster | Namespace | User | Typical Permissions |
|---|---|---|---|---|
| `dev-admin` | dev-cluster | * | alice | Full access in dev |
| `prod-readonly` | prod-cluster | production | bob | `get, list, watch` only |
| `ci-deploy` | prod-cluster | apps | ci-sa | `get, create, update` on deployments |
| `monitoring` | prod-cluster | monitoring | prom-sa | `get, list` on metrics endpoints |

### Why Contexts Matter for Auth

- A context **does not grant permissions** — it only selects which identity to present
- The kube-apiserver uses the identity to look up **RBAC rules**
- Switching context = switching identity = different permissions
- This is the clean separation: kubeconfig handles **who you are**, RBAC handles **what you can do**

---

## 10. The Noisy Neighbor Problem

### What Is It?

The **Noisy Neighbor Problem** occurs when one pod in a shared cluster consumes a disproportionate amount of CPU, memory, network, or storage — degrading performance for other pods on the same node.

```
Node (8 CPU, 32Gi RAM)
├── Pod A: requests 1CPU/2Gi  → actually using 7CPU/28Gi  ← NOISY NEIGHBOR
├── Pod B: requests 1CPU/2Gi  → starved, throttled
└── Pod C: requests 1CPU/2Gi  → starved, throttled
```

### Causes

- Missing or incorrect resource `requests` and `limits`
- Memory leaks in application code
- Unexpected traffic spikes without autoscaling
- Batch jobs running alongside latency-sensitive workloads

### Solution: Resource Requests & Limits

```yaml
resources:
  requests:          # Guaranteed — used for scheduling
    memory: "256Mi"
    cpu: "250m"
  limits:            # Maximum — enforced at runtime
    memory: "512Mi"
    cpu: "500m"
```

### Solution: LimitRange (namespace-level defaults)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "1Gi"
```

### Solution: ResourceQuota (namespace-level cap)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
```

### Solution: Priority Classes

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: true
```

---

## 11. Authorization (AuthZ) Overview

### Why AuthZ?

AuthN tells us **who** someone is. AuthZ decides **what they are allowed to do**. Without authorization, every authenticated user would have unlimited access to every API endpoint.

### Definition

**Authorization (AuthZ)** is the process of determining whether an authenticated identity has permission to perform a specific action on a specific resource.

### How It Works

```
Authenticated Request
        │
        ▼
┌────────────────────────────────────────────────┐
│              kube-apiserver                    │
│                                                │
│  Authorization Modules (evaluated in order):  │
│  1. Node Authorizer     (for kubelet)          │
│  2. ABAC                (attribute-based)      │
│  3. RBAC                (role-based) ◄── most  │
│  4. Webhook             (external)             │
│  5. AlwaysAllow / AlwaysDeny (special modes)   │
│                                                │
│  If ANY module says ALLOW → request proceeds  │
│  If ALL modules say DENY  → 403 Forbidden      │
└────────────────────────────────────────────────┘
```

### Auth(Z) — Users Given Access to API Endpoints

Authorization controls access to API endpoints like:

```
GET    /api/v1/namespaces/default/pods
POST   /api/v1/namespaces/default/pods
DELETE /apis/apps/v1/namespaces/default/deployments/my-app
GET    /api/v1/nodes
```

---

## 12. AuthZ Modes & Static Manifest Paths

### Authorization Modes

| Mode | Description | Use Case |
|---|---|---|
| `AlwaysAllow` | Every request is permitted | Development/testing ONLY — never production |
| `AlwaysDeny` | Every request is denied | Disabling access completely |
| `ABAC` | Attribute-Based Access Control — uses policy files | Legacy, rarely used |
| `RBAC` | Role-Based Access Control — uses K8s API objects | **Standard for production** |
| `Webhook` | Delegates decision to external HTTP service | Custom policy engines (OPA, Styra) |
| `Node` | Special purpose for kubelet authorization | Always enabled in kubeadm clusters |

**Setting authorization modes on kube-apiserver:**
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
    - command:
        - kube-apiserver
        - --authorization-mode=Node,RBAC         # comma-separated, evaluated in order
        - --authorization-webhook-config-file=...
```

### Static Manifest Paths (kubeadm clusters)

```
/etc/kubernetes/manifests/
├── etcd.yaml                      ← etcd configuration
├── kube-apiserver.yaml            ← API server (auth modes set here)
├── kube-controller-manager.yaml   ← controller manager
└── kube-scheduler.yaml            ← scheduler
```

**Key etcd.yaml flags:**
```yaml
- etcd
- --data-dir=/var/lib/etcd
- --cert-file=/etc/kubernetes/pki/etcd/server.crt
- --key-file=/etc/kubernetes/pki/etcd/server.key
- --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
- --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
```

**Key kube-apiserver.yaml flags:**
```yaml
- kube-apiserver
- --authorization-mode=Node,RBAC
- --authentication-token-webhook-config-file=...
- --oidc-issuer-url=...
- --audit-log-path=/var/log/kubernetes/audit.log
```

> **Can you edit RoleRef?**
> **NO.** `roleRef` in a `RoleBinding` or `ClusterRoleBinding` is **immutable** after creation. If you need to change the role a binding points to, you must **delete and recreate** the binding. This is a deliberate design decision to prevent privilege escalation.

```bash
# Cannot patch roleRef — must delete and recreate
kubectl delete rolebinding my-binding -n my-namespace
kubectl create rolebinding my-binding \
  --role=new-role \
  --serviceaccount=my-namespace:my-sa \
  -n my-namespace
```

---

## 13. RBAC — 4 Core Objects

```
┌──────────────────────────────────────────────────────────────┐
│                    RBAC Object Model                         │
├─────────────────────┬────────────────────────────────────────┤
│ Object              │ Scope                                  │
├─────────────────────┼────────────────────────────────────────┤
│ Role                │ Namespace-scoped permissions           │
│ RoleBinding         │ Namespace-scoped binding               │
│ ClusterRole         │ Cluster-wide permissions               │
│ ClusterRoleBinding  │ Cluster-wide OR namespace binding      │
└─────────────────────┴────────────────────────────────────────┘
```

### 1. Role (Namespace-scoped)

Defines **what actions** are allowed on **which resources** within a **specific namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

### 2. RoleBinding (Namespace-scoped)

Binds a Role (or ClusterRole) to a subject **within a namespace**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. ClusterRole (Cluster-level)

Defines permissions that can apply **cluster-wide** or be reused across namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create"]
```

### 4. ClusterRoleBinding (Cluster-level OR cross-namespace)

Binds a ClusterRole to a subject cluster-wide **or** to a specific namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: global-node-reader
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Scope Comparison

```
ClusterRole + ClusterRoleBinding → cluster-wide access
ClusterRole + RoleBinding        → namespace-scoped access (reusing cluster role)
Role        + RoleBinding        → namespace-scoped access (namespace-local role)
Role        + ClusterRoleBinding → ❌ INVALID
```

---

## 14. RBAC Best Practices

### RoleBinding >>> ClusterRoleBinding

**Always prefer `RoleBinding` over `ClusterRoleBinding`** when access is only needed in specific namespaces.

```
❌ Bad:  ClusterRoleBinding grants access to ALL namespaces
✅ Good: RoleBinding grants access to only ONE namespace

Even if using a ClusterRole, bind it with RoleBinding per namespace
```

### Limit Sensitive Workloads

```yaml
# Use namespace isolation for sensitive workloads
apiVersion: v1
kind: Namespace
metadata:
  name: sensitive-data
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
```

```yaml
# Restrict what the sensitive namespace can mount
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sensitive-app-role
  namespace: sensitive-data
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["only-this-secret"]   # ← restrict to specific resource names
    verbs: ["get"]
```

### Use Namespaces for Isolation

```bash
# Create dedicated namespaces per team/environment
kubectl create namespace team-payments
kubectl create namespace team-identity
kubectl create namespace team-analytics

# Assign team leads as namespace admins
kubectl create rolebinding team-payments-admin \
  --clusterrole=admin \
  --user=payments-lead \
  --namespace=team-payments
```

### Principle of Least Privilege Checklist

- [ ] Use `Role` not `ClusterRole` when namespace-scoped is sufficient
- [ ] Use `RoleBinding` not `ClusterRoleBinding` unless cluster-wide access is truly needed
- [ ] Restrict `resourceNames` to specific resources where possible
- [ ] Avoid `*` (wildcard) verbs or resources in production
- [ ] Audit service accounts — disable automount if not needed: `automountServiceAccountToken: false`
- [ ] Rotate tokens regularly
- [ ] Never grant `escalate`, `bind`, `impersonate` without strong justification

---

## 15. Subjects, Verbs & Resources

### Subjects

```yaml
subjects:
  # 1. User (human, external identity)
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io

  # 2. ServiceAccount (pod/application identity)
  - kind: ServiceAccount
    name: my-app-sa
    namespace: production

  # 3. Group (collection of users or service accounts)
  - kind: Group
    name: system:authenticated         # all authenticated users
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: system:serviceaccounts:prod  # all SAs in prod namespace
    apiGroup: rbac.authorization.k8s.io
```

### Built-in Groups

| Group | Members |
|---|---|
| `system:masters` | Cluster admins (bypasses RBAC) |
| `system:authenticated` | All authenticated users |
| `system:unauthenticated` | Anonymous requests |
| `system:serviceaccounts` | All service accounts |
| `system:serviceaccounts:<ns>` | All SAs in a specific namespace |

### Verbs

| Verb | HTTP Method | Description |
|---|---|---|
| `get` | GET | Retrieve a single resource |
| `list` | GET | Retrieve a collection of resources |
| `watch` | GET + ?watch=true | Watch for changes (streaming) |
| `create` | POST | Create a resource |
| `update` | PUT | Replace a resource entirely |
| `patch` | PATCH | Partially update a resource |
| `delete` | DELETE | Delete a resource |
| `deletecollection` | DELETE | Delete a collection |
| `proxy` | various | Proxy to a service/pod |
| `connect` | various | Connect to a pod (exec, attach) |
| `use` | special | Use a PodSecurityPolicy |
| `bind` | special | Bind a role (escalation risk) |
| `escalate` | special | Grant higher privileges (escalation risk) |
| `impersonate` | special | Act as another user/group |

**Special Non-Resource URLs:**

```yaml
rules:
  - nonResourceURLs: ["/healthz", "/metrics", "/version"]
    verbs: ["get"]
```

### Full Permission Rule Structure

```yaml
rules:
  - verbs: ["get", "watch", "list"]        # what actions
    apiGroups: [""]                         # "" = core API group
    resources: ["secrets"]                  # what resources
    resourceNames: ["my-secret"]            # optional: specific resource names
  - nonResourceURLs: ["/metrics"]           # non-resource URL access
    verbs: ["get"]
```

**Common apiGroups:**

| apiGroup | Resources |
|---|---|
| `""` (core) | pods, services, secrets, configmaps, nodes, pv, pvc |
| `apps` | deployments, replicasets, daemonsets, statefulsets |
| `batch` | jobs, cronjobs |
| `rbac.authorization.k8s.io` | roles, rolebindings, clusterroles, clusterrolebindings |
| `networking.k8s.io` | ingresses, networkpolicies |
| `storage.k8s.io` | storageclasses, volumeattachments |

---

## 16. RBAC YAML Manifests

### role.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

### rolebinding.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-manager-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: app-deployer
    namespace: production
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
  # ⚠️ roleRef is IMMUTABLE after creation — delete and recreate to change
```

### clusterrole.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["production-db-password", "api-keys"]
    verbs: ["get", "watch", "list"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list"]
  - nonResourceURLs: ["/healthz", "/version"]
    verbs: ["get"]
```

### clusterrolebinding.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: secret-reader-global
subjects:
  - kind: Group
    name: security-auditors
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: monitoring-agent
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### Apply & Reconcile

```bash
# Apply RBAC manifests
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml

# Reconcile (idempotent, handles updates safely)
kubectl auth reconcile -f clusterrole.yaml
kubectl auth reconcile -f rolebinding.yaml

# Reconcile entire directory
kubectl auth reconcile -f ./rbac/

# Dry run first
kubectl auth reconcile --dry-run=client -f clusterrole.yaml
```

---

## 17. Verifying Permissions

### Check What You Can Do

```bash
# Can I create pods?
kubectl auth can-i create pods

# Can a specific user create pods?
kubectl auth can-i create pods --as=alice

# Can a service account create pods?
kubectl auth can-i create pods \
  --as=system:serviceaccount:production:app-deployer

# Can a user do something in a specific namespace?
kubectl auth can-i delete secrets --as=bob --namespace=production

# List ALL permissions for the current user
kubectl auth can-i --list

# List permissions for a specific user
kubectl auth can-i --list --as=alice --namespace=production
```

### Inspect Roles & Bindings

```bash
# Who can access secrets?
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o wide | grep secret

# What can a role do?
kubectl describe role pod-manager -n production
kubectl describe clusterrole secret-reader

# What bindings exist for a user?
kubectl get rolebindings -n production -o json | \
  jq '.items[] | select(.subjects[]?.name=="alice")'
```

### Audit Who Has Access to a Resource

```bash
# Find all subjects who can get secrets in production
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.roleRef.name | test("secret")) | 
      {name: .metadata.name, namespace: .metadata.namespace, subjects: .subjects}'
```

---

## 18. HTTP Error Codes — Causes & Solutions

### Quick Reference Table

| Code | Name | Kubernetes Context | Common Cause | Solution | First Action |
|---|---|---|---|---|---|
| **401** | Unauthorized | AuthN failed | Invalid/expired token, bad cert, wrong kubeconfig | Rotate token, check kubeconfig | `kubectl config view`, regenerate token |
| **403** | Forbidden | AuthZ failed | Missing RBAC permissions, wrong namespace | Add RoleBinding/ClusterRoleBinding | `kubectl auth can-i <verb> <resource> --as=<user>` |
| **404** | Not Found | Resource missing | Wrong name, wrong namespace, resource deleted | Check namespace and name | `kubectl get <resource> -A --show-labels` |
| **409** | Conflict | Resource already exists | `create` when it exists, resourceVersion mismatch | Use `apply` instead of `create`, fetch latest version | `kubectl get <resource> -o yaml` to get current resourceVersion |
| **422** | Unprocessable Entity | Validation failed | Invalid YAML, required field missing, type mismatch | Validate YAML | `kubectl apply --dry-run=server -f file.yaml` |
| **429** | Too Many Requests | Rate limited | Too many API calls (client-side throttling) | Add backoff/retry logic | Check `--qps` and `--burst` flags |
| **500** | Internal Server Error | API server error | Bug, misconfiguration, etcd issue | Check kube-apiserver logs | `kubectl logs -n kube-system kube-apiserver-<node>` |
| **503** | Service Unavailable | API server down | Control plane failure, network issue | Check node health | `kubectl get nodes`, check control plane pods |

### Detailed Error Analysis

#### 401 Unauthorized
```bash
# Cause: Token expired or invalid
# Diagnosis:
kubectl config view --minify
curl -k -H "Authorization: Bearer $TOKEN" https://<apiserver>/api/v1/pods
# Check response: {"kind":"Status","reason":"Unauthorized"}

# Solution 1: Regenerate token
kubectl create token my-sa --duration=3600s

# Solution 2: Check certificate validity
openssl x509 -in client.crt -noout -dates

# Solution 3: Re-login (OIDC)
kubectl oidc-login get-token --oidc-issuer-url=...
```

#### 403 Forbidden
```bash
# Cause: RBAC not configured properly
# Diagnosis:
kubectl auth can-i get pods --as=alice --namespace=production
# Output: no

# Solution: Add missing permissions
kubectl create rolebinding alice-pod-reader \
  --clusterrole=view \
  --user=alice \
  --namespace=production

# Verify:
kubectl auth can-i get pods --as=alice --namespace=production
# Output: yes
```

#### 404 Not Found
```bash
# Cause: Wrong namespace or resource name typo
# Diagnosis:
kubectl get pods -n wrong-namespace        # correct this
kubectl get pods --all-namespaces | grep myapp  # find it

# Solution: Use correct namespace
kubectl get pods -n correct-namespace
```

#### 503 Service Unavailable
```bash
# Cause: API server unreachable
# Diagnosis:
kubectl cluster-info
curl -k https://<apiserver>:6443/healthz

# Check control plane pods
kubectl get pods -n kube-system
kubectl describe pod kube-apiserver-<node> -n kube-system

# Check etcd health
kubectl exec -n kube-system etcd-<node> -- \
  etcdctl endpoint health \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Solution: Restart affected component
systemctl restart kubelet
```

#### 201 Created / 202 Accepted (Success codes to know)

| Code | Meaning |
|---|---|
| 200 | OK — GET/UPDATE succeeded |
| 201 | Created — POST succeeded, resource was created |
| 202 | Accepted — async operation accepted (e.g., delete in progress) |

---

## 19. Volumes — Why, Types & Categories

### Why Do We Need Volumes?

Container filesystems are **ephemeral** — when a container restarts, all data is lost. Volumes solve this by:

1. **Persisting data** beyond container lifecycle
2. **Sharing data** between containers in a pod
3. **Injecting configuration** (ConfigMaps, Secrets) as files
4. **Providing scratch space** for temporary processing

### Volume Lifecycle

```
Ephemeral:  tied to Pod lifecycle — data lost when pod is deleted
Persistent: outlives Pod lifecycle — data survives pod deletion/restart
```

### Volume Categories

```
Volumes
├── Ephemeral (in-pod lifetime)
│   ├── emptyDir          ← temp scratch space, shared between containers
│   ├── configMap         ← inject config files
│   ├── secret            ← inject secret files
│   ├── downwardAPI       ← inject pod metadata as files
│   └── projected         ← combine multiple sources
│
├── Persistent (survive pod deletion)
│   ├── hostPath          ← ⚠️ avoid in production (node-specific)
│   ├── local             ← node-local SSD, with node affinity
│   ├── nfs               ← Network File System
│   └── PersistentVolume  ← abstract storage backend (CSI-backed)
│
└── Cloud-Managed (via CSI drivers)
    ├── AWS EBS           ← awsElasticBlockStore / EBS CSI Driver
    ├── GCP PD            ← gcePersistentDisk / GCE CSI Driver
    ├── Azure Disk        ← azureDisk / Azure CSI Driver
    ├── Azure File        ← azureFile (supports RWX)
    └── Ceph/Rook, Longhorn, OpenEBS (self-hosted)
```

### Volume Types in Detail

#### emptyDir (Ephemeral)

```yaml
volumes:
  - name: cache-vol
    emptyDir:
      medium: ""          # "" = node disk, "Memory" = tmpfs (RAM-backed)
      sizeLimit: "500Mi"  # optional limit
```

Use cases: caching, temp processing, sharing data between init container and main container

#### hostPath (⚠️ Avoid in Production)

```yaml
volumes:
  - name: host-data
    hostPath:
      path: /var/log/myapp
      type: DirectoryOrCreate
```

Problems: breaks pod portability, security risk (pod accesses host FS), not HA.
Use only for: system-level DaemonSets (log collectors, metrics agents).

#### CSI (Container Storage Interface)

```yaml
volumes:
  - name: my-csi-vol
    csi:
      driver: ebs.csi.aws.com
      volumeAttributes:
        type: gp3
```

CSI is the **standard interface** between Kubernetes and storage providers. All major cloud providers and storage vendors implement CSI drivers.

### Pod Volume Lifecycle Alignment

```
initContainer starts  → emptyDir created
initContainer done   → data in emptyDir
main container starts → reads emptyDir data
pod deleted          → emptyDir destroyed, data GONE

                vs.

PVC bound to PV      → data exists on storage backend
pod deleted          → PVC/PV still exists
new pod mounts PVC   → data still available ✓
```

### Internal K8s Version & CSI Timeline

| K8s Version | CSI Milestone |
|---|---|
| 1.9 | CSI introduced (alpha) |
| 1.13 | CSI stable (GA) |
| 1.17 | In-tree volume plugins deprecated |
| 1.24+ | Most in-tree plugins removed, CSI mandatory |
| 1.26+ | `CSIMigration` flags enabled by default |

**EKS, GKE, AKS** — all use CSI drivers by default in modern versions.

---

## 20. PV / PVC — Modes, Policies & Use Cases

### PersistentVolume (PV)

A PV is a **cluster-level storage resource** provisioned by an admin or dynamically by a StorageClass.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem      # or Block
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0a1b2c3d4e5f
    fsType: ext4
```

### PersistentVolumeClaim (PVC)

A PVC is a **namespace-level request** for storage by a user/workload.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard   # must match PV's storageClassName
```

### Access Modes

| Mode | Short | Description | Use Case |
|---|---|---|---|
| `ReadWriteOnce` | RWO | One node can mount R/W | Databases, single-instance apps |
| `ReadOnlyMany` | ROX | Many nodes can mount read-only | Shared config, static assets |
| `ReadWriteMany` | RWX | Many nodes can mount R/W | Shared file storage (NFS, Azure File) |
| `ReadWriteOncePod` | RWOP | One **pod** can mount R/W (K8s 1.22+, beta) | Strict single-pod exclusivity |

> **Note:** `ReadWriteOncePod` (RWOP) is a beta feature (cbeta-1 / `--feature-gates=ReadWriteOncePod=true`). It was promoted to beta in K8s 1.27.

### Reclaim Policies

| Policy | Behavior After PVC Delete | Use Case |
|---|---|---|
| `Retain` | PV stays, data preserved, status = Released | Production data you want to keep |
| `Delete` | PV and underlying storage are deleted | Dev/test, cloud-backed dynamic PVs |
| `Recycle` | ⚠️ Deprecated — basic scrub then available again | Never use |

```bash
# Change reclaim policy on an existing PV
kubectl patch pv my-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### PV Binding Phases

```
Available → Bound → Released → (Retain: manual cleanup, Delete: auto-delete)
```

### Use Cases

| Use Case | Access Mode | Reclaim | StorageClass |
|---|---|---|---|
| MySQL database | RWO | Retain | gp3-ebs |
| Elasticsearch cluster | RWO (per node) | Delete | fast-ssd |
| Shared media assets | RWX | Retain | azure-file |
| Read-only config data | ROX | Retain | standard |
| CI/CD scratch space | RWO | Delete | standard |

### Can You Store Images in PVC/PV?

**Yes, but with caveats:**

```
✅ Store image files (JPG, PNG, etc.) in PVC — works fine
✅ Store OCI image layers in a registry-backed PVC (Harbor, Quay)
❌ Cannot store container images to be run directly from PVC
   — container runtime needs images in its own store (containerd/docker)
```

```yaml
# Example: Harbor registry using PVC for image storage
kind: PersistentVolumeClaim
metadata:
  name: harbor-registry-storage
spec:
  accessModes:
    - ReadWriteMany   # NFS or Azure File for HA
  resources:
    requests:
      storage: 100Gi
```

### Mount PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
spec:
  containers:
    - name: postgres
      image: postgres:15
      volumeMounts:
        - name: data-vol
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data-vol
      persistentVolumeClaim:
        claimName: my-pvc   # reference the PVC
```

---

## 21. Dynamic Provisioning & StorageClass

### Why Dynamic Provisioning?

Static provisioning requires a cluster admin to manually create PVs before users can claim storage. **Dynamic provisioning** removes this by automatically creating PVs when a PVC is submitted.

```
Static:   Admin creates PV → User creates PVC → K8s binds them
Dynamic:  User creates PVC → StorageClass creates PV + provisions storage
```

### StorageClass

A `StorageClass` defines how storage should be provisioned dynamically.

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # make this the default
provisioner: ebs.csi.aws.com          # ← which CSI driver handles provisioning
parameters:                            # ← driver-specific parameters
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/abc"
reclaimPolicy: Delete                  # Retain or Delete
volumeBindingMode: WaitForFirstConsumer  # or Immediate
allowVolumeExpansion: true
mountOptions:
  - debug
```

### volumeBindingMode

| Mode | Behavior | Use Case |
|---|---|---|
| `Immediate` | PV provisioned as soon as PVC is created | Simple setups, cloud storage |
| `WaitForFirstConsumer` | PV provisioned when Pod is scheduled | Node-local storage, zone-aware provisioning |

### Dynamic PVC with StorageClass

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  namespace: production
spec:
  storageClassName: fast-ssd    # ← references the StorageClass
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  # No volumeName needed — dynamically provisioned
```

### StorageClass Examples by Cloud

**AWS EKS:**
```yaml
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
```

**GCP GKE:**
```yaml
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
```

**Azure AKS:**
```yaml
# Block storage (RWO)
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS

# File storage (RWX)
provisioner: file.csi.azure.com
parameters:
  skuName: Standard_LRS
```

**On-premises (Longhorn):**
```yaml
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  diskSelector: "ssd"
```

### Default StorageClass

```bash
# List storage classes
kubectl get storageclass

# Mark a storage class as default
kubectl patch storageclass fast-ssd \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# PVC without storageClassName uses the default
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: auto-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
  # storageClassName omitted → uses default StorageClass
```

### Volume Expansion

```bash
# Resize a PVC (StorageClass must have allowVolumeExpansion: true)
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Check expansion status
kubectl describe pvc my-pvc | grep -A5 Conditions
```

---

## 22. Interview Questions & Answers

### Authentication (AuthN) Questions

**Q1: What is the difference between AuthN and AuthZ in Kubernetes?**

> **AuthN (Authentication)** verifies *who* the requester is — it establishes identity. **AuthZ (Authorization)** determines *what* that identity is allowed to do. AuthN happens first; if it fails, the request is rejected with 401. If AuthN passes but AuthZ fails, the response is 403. You need both: knowing who someone is doesn't mean they should have access to everything.

---

**Q2: How does a Pod authenticate to the Kubernetes API server?**

> Pods authenticate using a **ServiceAccount token**, which is automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This is a short-lived JWT token signed by the Kubernetes control plane. The pod presents this token as a Bearer token in API requests. Modern Kubernetes (1.22+) uses **projected service account tokens** with a configurable expiry (default 1 hour) instead of the old long-lived tokens stored as Secrets.

---

**Q3: Why are static token files not recommended in production?**

> Static token files have several problems: (1) they require an **API server restart** to add/remove tokens, (2) tokens never expire, (3) they're stored in plaintext on the control plane node, (4) there's no way to rotate without downtime. Better alternatives are short-lived OIDC tokens or projected ServiceAccount tokens.

---

**Q4: What happens if two authentication plugins are configured and the first one fails?**

> Kubernetes tries each configured authenticator in order. If the first plugin returns an error, it moves to the next plugin. If a plugin successfully identifies the user (or definitively says "I don't recognize this token"), processing stops. The request is only rejected with 401 if **all** plugins fail to authenticate it.

---

**Q5: How do you revoke a user's access immediately in Kubernetes?**

> Kubernetes does not have a built-in user database, so "revoking" depends on the auth method: (1) **x.509 certs**: add the cert to a Certificate Revocation List (CRL) or rotate the CA — certs cannot be truly revoked otherwise; (2) **OIDC tokens**: revoke at the identity provider level; (3) **Static tokens**: remove from file + restart API server; (4) **ServiceAccount tokens**: delete the ServiceAccount or its secret; projected tokens expire naturally.

---

### RBAC (AuthZ) Questions

**Q6: Can you edit the `roleRef` field in a RoleBinding?**

> **No.** `roleRef` is immutable after creation. This is a deliberate security decision — if `roleRef` were mutable, an attacker with `update` permission on RoleBindings could escalate their own privileges by changing which role a binding points to. To change the role, you must delete the binding and recreate it.

---

**Q7: What's the difference between ClusterRole and ClusterRoleBinding?**

> A **ClusterRole** defines *what* permissions exist (rules on resources/verbs) at the cluster level. A **ClusterRoleBinding** *grants* those permissions to subjects cluster-wide. Importantly, a ClusterRole can also be bound using a **RoleBinding** (namespace-scoped), which grants the ClusterRole's permissions but only within that namespace. This is a powerful pattern for reusing common permission sets (like "pod-reader") across namespaces.

---

**Q8: What is the principle of least privilege and how do you implement it in K8s?**

> Least privilege means giving an identity only the minimum permissions needed to do its job. In K8s: (1) use `Role` instead of `ClusterRole` where namespace-scope suffices; (2) use `RoleBinding` instead of `ClusterRoleBinding`; (3) restrict `resourceNames` to specific resources; (4) avoid wildcard `*` verbs; (5) set `automountServiceAccountToken: false` on pods that don't call the API; (6) use separate ServiceAccounts per workload rather than sharing.

---

**Q9: How do you check what permissions a specific user has?**

> ```bash
> kubectl auth can-i --list --as=alice --namespace=production
> kubectl auth can-i delete secrets --as=alice --namespace=production
> ```

---

**Q10: What are the 4 RBAC objects and their scopes?**

> (1) **Role** — namespace-scoped permission set; (2) **RoleBinding** — namespace-scoped binding of a Role or ClusterRole to subjects; (3) **ClusterRole** — cluster-scoped permission set, usable across namespaces; (4) **ClusterRoleBinding** — binds a ClusterRole to subjects cluster-wide. The key insight: ClusterRole + RoleBinding = namespace-restricted access using a cluster-defined role — a very useful pattern.

---

### Volumes Questions

**Q11: Why do we need PersistentVolumes when we already have emptyDir?**

> `emptyDir` only lives as long as the Pod. When the pod is deleted, all data is gone. **PersistentVolumes** outlive pods — the data persists across pod restarts, rescheduling, and deletion. This is essential for stateful workloads like databases (PostgreSQL, MySQL, MongoDB) where losing data on pod restart would be catastrophic.

---

**Q12: What is the difference between RWO, RWX, and ROX access modes?**

> **RWO (ReadWriteOnce)**: only one node can mount the volume with read-write access — used for single-instance databases. **RWX (ReadWriteMany)**: multiple nodes can mount with read-write access simultaneously — used for shared file storage in multi-replica deployments. **ROX (ReadOnlyMany)**: multiple nodes can mount read-only — used for shared static content. Most cloud block storage (EBS, Persistent Disk) only supports RWO. NFS and Azure File support RWX.

---

**Q13: What are the three reclaim policies and when would you use each?**

> **Retain**: PV is kept after PVC deletion with all data intact; admin must manually clean up — use for production databases where accidental deletion must be recoverable. **Delete**: PV and the underlying storage are automatically deleted when the PVC is deleted — use for ephemeral dev/test workloads or auto-managed cloud storage. **Recycle**: deprecated, never use.

---

**Q14: What is dynamic provisioning and how does a StorageClass enable it?**

> Dynamic provisioning automatically creates PersistentVolumes on demand when a PVC is submitted — no admin pre-provisioning required. A **StorageClass** defines the template: which CSI `provisioner` to call, what `parameters` to pass (disk type, IOPS, encryption), the `reclaimPolicy`, and `volumeBindingMode`. When a PVC references a StorageClass, the provisioner creates the real storage (e.g., an EBS volume in AWS) and a corresponding PV automatically.

---

**Q15: What is `WaitForFirstConsumer` binding mode and why does it matter?**

> With `Immediate` binding, a PV is provisioned as soon as the PVC is created — but this can create a PV in the wrong availability zone before the scheduler decides where to place the pod. `WaitForFirstConsumer` delays PV provisioning until a pod is actually scheduled, then provisions storage in the same zone as the pod. This is **critical for zone-aware storage** (like AWS EBS) in multi-AZ clusters.

---

**Q16: Can a single PVC be mounted by multiple pods?**

> It depends on the access mode. With **RWO**, only one node can mount the PVC — so multiple pods can use it only if they're on the same node. With **RWX** (e.g., backed by NFS or Azure File), multiple pods across multiple nodes can mount it simultaneously. Most use cases prefer separate PVCs per pod for isolation (especially StatefulSets which create PVCs per replica automatically).

---

**Q17: What is the noisy neighbor problem and how do you prevent it?**

> The noisy neighbor problem is when one pod consumes excessive CPU/memory/IOPS on a shared node, starving other pods. Prevention: (1) always set resource `requests` and `limits` on every container; (2) use `LimitRange` for namespace defaults; (3) use `ResourceQuota` for namespace caps; (4) use `PriorityClasses` to protect critical workloads; (5) use pod topology constraints to spread workloads; (6) use dedicated node pools for sensitive workloads with taints/tolerations.

---

**Q18: What is a CSI driver and why was it introduced?**

> **Container Storage Interface (CSI)** is a standardized API between Kubernetes and storage providers. Before CSI, storage plugins were compiled directly into the Kubernetes codebase ("in-tree plugins"), meaning storage vendors had to contribute code to the K8s repo and wait for release cycles. CSI allows vendors to ship storage drivers independently as containers, separate from K8s releases. This speeds up innovation, improves stability, and allows out-of-tree updates. All major cloud and storage providers now offer CSI drivers.

---

> **Last updated**: May 2026 | **Kubernetes version coverage**: 1.24 – 1.30+
