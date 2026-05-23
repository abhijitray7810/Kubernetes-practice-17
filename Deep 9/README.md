# ☸️ Kubernetes Deep Dive — Complete Reference Guide

> **Author:** Prerit | **Topic:** K8s ConfigMaps · Secrets · Service Mesh · Networking · DNS  
> **Docs:** [kubernetes.io](https://kubernetes.io/docs) | [external-secrets.io](https://external-secrets.io) | [istio.io](https://istio.io)

---

## 📋 Table of Contents

1. [ConfigMap](#1-configmap)
2. [Secrets](#2-secrets)
3. [External Secrets & Sealed Secrets](#3-external-secrets--sealed-secrets)
4. [TLS, mTLS & Zero Trust](#4-tls-mtls--zero-trust)
5. [Networking Fundamentals — OSI Model](#5-networking-fundamentals--osi-model)
6. [Kubernetes Services Deep Dive](#6-kubernetes-services-deep-dive)
7. [Service Mesh & Istio](#7-service-mesh--istio)
8. [What Happens When You Type google.com](#8-what-happens-when-you-type-googlecom)

---

## 1. ConfigMap

### 1.1 Definition

A **ConfigMap** is a Kubernetes API object used to store **non-confidential** configuration data as **key-value pairs**.  
It decouples environment-specific configuration from your container image — so the same image can run in `dev`, `staging`, and `prod` with different configs.

> ⚠️ **ConfigMaps are NOT for sensitive data** (passwords, tokens, keys). Use Secrets for those.

---

### 1.2 Why Do We Need ConfigMaps?

**Problem WITHOUT ConfigMaps:**

```
App needs DB_HOST=prod-db.example.com
→ You hardcode it in the Docker image
→ Now dev needs DB_HOST=dev-db.example.com
→ You build another image
→ Now you have 3 different images for the same app
→ Drift, confusion, and maintenance nightmare
```

**Solution WITH ConfigMaps:**

```
One image + different ConfigMap per environment
→ dev-configmap  → DB_HOST=dev-db
→ prod-configmap → DB_HOST=prod-db
→ Clean separation of code and config
```

---

### 1.3 Use Cases

| Use Case | Example |
|----------|---------|
| App configuration | `LOG_LEVEL=debug`, `MAX_CONNECTIONS=100` |
| Feature flags | `FEATURE_NEW_UI=true` |
| External service URLs | `DB_HOST=postgres.prod.svc.cluster.local` |
| Config files (nginx.conf, app.properties) | Mount entire config file as a volume |
| Command-line arguments | Pass as env var to container entrypoint |

---

### 1.4 Creating a ConfigMap

#### Method 1 — From Literal Values

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres \
  --from-literal=LOG_LEVEL=info \
  --from-literal=APP_PORT=8080
```

#### Method 2 — From a File

```bash
# Create a file first
cat > app.properties << EOF
db.host=postgres
log.level=info
app.port=8080
EOF

kubectl create configmap app-config --from-file=app.properties
```

#### Method 3 — YAML Manifest (Recommended)

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config          # ← ConfigMap name (used to reference it)
  namespace: default
data:
  # 1st: ConfigMap name (used above in metadata.name)
  # 2nd: Which keys we stored
  DB_HOST: "postgres.default.svc.cluster.local"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
  APP_ENV: "production"
  # You can also store multi-line config files
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }
```

```bash
kubectl apply -f configmap.yaml
kubectl get configmap app-config
kubectl describe configmap app-config
```

---

### 1.5 Consuming a ConfigMap

#### Way 1 — As Environment Variables (individual keys)

```yaml
# 1st: ConfigMap name → app-config
# 2nd: Which key we stored → DB_HOST, LOG_LEVEL
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:latest
    env:
    - name: DATABASE_HOST          # env var name inside container
      valueFrom:
        configMapKeyRef:
          name: app-config         # ← ConfigMap name (1st reference)
          key: DB_HOST             # ← Which key we stored (2nd reference)
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
```

#### Way 2 — All Keys as Environment Variables at Once

```yaml
spec:
  containers:
  - name: my-app
    image: my-app:latest
    envFrom:
    - configMapRef:
        name: app-config           # Injects ALL keys as env vars
```

#### Way 3 — Consumed in a Volume (Recommended for files)

```yaml
spec:
  containers:
  - name: my-app
    image: my-app:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config       # ← Files appear here inside container
  volumes:
  - name: config-volume
    configMap:
      name: app-config             # ← ConfigMap name
```

This mounts each key as a **file** inside `/etc/config/`.  
So `DB_HOST` → `/etc/config/DB_HOST` with content `postgres`.

---

### 1.6 ⚡ Critical: ConfigMap Update Behavior

| Consumption Method | Auto-Update When CM Changes? |
|-------------------|------------------------------|
| **Mounted as Volume** | ✅ **YES** — kubelet syncs changes (may take ~1-2 min) |
| **Environment Variables** | ❌ **NO** — Pod restart required |

> **Why?** Environment variables are set at container startup and baked into the process environment. They are not watched for changes. Volume mounts use a symlink mechanism and kubelet periodically syncs them.

```bash
# If consumed as env var, after updating ConfigMap you MUST restart:
kubectl rollout restart deployment my-app

# Verify the configmap was updated
kubectl get configmap app-config -o yaml
```

---

## 2. Secrets

### 2.1 Why Do We Need Secrets?

ConfigMaps store **non-confidential** data. But what about:

- Database passwords
- API keys (Stripe, AWS, GitHub tokens)
- TLS certificates
- SSH private keys
- Docker registry credentials

These **cannot** go into ConfigMaps because:
1. ConfigMaps are stored in plain text in etcd
2. Anyone with `kubectl get configmap` access sees them
3. They appear in logs, describe output, etc.

**Secrets provide a separate, restricted API object** with:
- Restricted RBAC (can deny access to secrets even if user has pod access)
- Stored separately in etcd (can be encrypted at rest)
- Not shown in `kubectl describe pod` by default

---

### 2.2 Use Cases

| Use Case | Secret Type |
|----------|-------------|
| DB password, API key, token | `Opaque` (generic) |
| Docker registry pull credentials | `kubernetes.io/dockerconfigjson` |
| TLS certificate + key | `kubernetes.io/tls` |
| SSH private key | `kubernetes.io/ssh-auth` |
| Service account token | `kubernetes.io/service-account-token` |

---

### 2.3 Creating Secrets

#### From Literal Values (Opaque/Generic)

```bash
# kubectl create secret generic <name> --from-literal=<key>=<value>
kubectl create secret generic db-credentials \
  --from-literal=username=prerit \
  --from-literal=password=MyS3cr3tP@ss
```

This creates a secret of type **Opaque** (default).

#### From a File

```bash
# Create a file with the secret content
echo -n "MyS3cr3tP@ss" > ./password.txt

kubectl create secret generic db-credentials \
  --from-file=password=./password.txt
```

#### From a YAML Manifest

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
type: Opaque                        # ← Secret type (Opaque is default/generic)
data:
  # Values MUST be base64 encoded
  username: cHJlcml0              # echo -n "prerit" | base64
  password: TXlTM2NyM3RQQHNz      # echo -n "MyS3cr3tP@ss" | base64
```

Or use `stringData` (Kubernetes auto-encodes):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:                         # ← Plain text, K8s auto base64 encodes
  username: prerit
  password: MyS3cr3tP@ss
```

---

### 2.4 Secret Types

| Type | Command Flag | Use Case |
|------|-------------|----------|
| `Opaque` | `generic` | Default, any arbitrary key-value data |
| `kubernetes.io/dockerconfigjson` | `docker-registry` | Docker Hub / ECR / GCR pull credentials |
| `kubernetes.io/tls` | `tls` | TLS certificate and private key |
| `kubernetes.io/ssh-auth` | `generic` | SSH private key |

#### Docker Registry Secret

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=prerit \
  --docker-password=mypassword \
  --docker-email=prerit@example.com
```

#### TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

---

### 2.5 Consuming Secrets

#### As Environment Variables

```yaml
spec:
  containers:
  - name: my-app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials     # Secret name
          key: password            # Key inside the secret
```

#### As Volume (more secure — not exposed in env)

```yaml
spec:
  containers:
  - name: my-app
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

Files appear as `/etc/secrets/username` and `/etc/secrets/password`.

---

### 2.6 ⚠️ Encoding ≠ Encryption

> **This is the most misunderstood thing about K8s Secrets.**

```bash
# What K8s stores:
echo -n "prerit" | base64
# → cHJlcml0

# Anyone can decode it in 1 second:
echo -n "cHJlcml0" | base64 -d
# → prerit
```

**Base64 is encoding, not encryption.**  
It is just a representation format — completely reversible with zero effort.

**What this means:**

| Claim | Reality |
|-------|---------|
| "Secrets are secure in K8s" | ❌ By default, NO. They're just base64 in etcd |
| "Only admins can read secrets" | Only if you properly configure RBAC |
| "Secrets are encrypted" | Only if you enable Encryption at Rest in the API server |

```bash
# ANYONE with kubectl access can do this:
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
# → MyS3cr3tP@ss  ← full plain text, instantly
```

---

### 2.7 Why Just Encoding? — The Real Problems

**Problem 1 — Secret Rotation**

- When a secret rotates (password changes), you must update the K8s secret manually
- Then restart all pods consuming it as env vars
- This is a manual, error-prone process
- At scale (50+ services), this becomes a nightmare

**Problem 2 — Not Scalable**

- 60%+ of companies (like Bitnami/VMware report) use external secret management
- K8s secrets stored in etcd don't integrate with:
  - Company password policies
  - Automatic rotation
  - Audit logging
  - Access controls beyond basic RBAC

**Problem 3 — Secret Sprawl**

- Secrets duplicated across namespaces and clusters
- No single source of truth
- Compliance nightmares (SOC2, PCI DSS, HIPAA)

```bash
# You can see secrets like this - which is a problem
kubectl get secret db-credentials -o yaml
# → Entire secret in base64, readable by anyone with get-secret permission
```

**That's why ~60% of companies use External Secrets solutions.**

---

## 3. External Secrets & Sealed Secrets

### 3.1 The Problem We're Solving

```
Native K8s Secrets:
  ✅ Simple to use
  ❌ Base64 only (not encrypted)
  ❌ No rotation
  ❌ No audit trail
  ❌ Cannot use corporate secret managers (Vault, AWS SSM, GCP Secret Manager)
  ❌ Checked into Git = exposed credentials
```

---

### 3.2 Sealed Secrets (Bitnami) — GitOps-Safe Secrets

#### What is it?

Sealed Secrets lets you **encrypt** your K8s secrets so you can safely commit them to Git.

#### How it works:

```
Developer → encrypts secret with public key → SealedSecret.yaml
SealedSecret.yaml → commit to Git (safe!)
→ Sealed Secrets Controller in cluster decrypts with private key
→ Creates real K8s Secret
```

#### Installation

```bash
# Install the controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# Install kubeseal CLI
brew install kubeseal  # or download binary
```

#### Usage

```bash
# Step 1: Create a regular secret manifest (dry-run, don't apply to cluster)
kubectl create secret generic name-sealed-secret \
  -n sealed-secret \
  --dry-run=client \
  --from-literal=name=prerit \
  -o yaml > secret.yaml

# secret.yaml contains base64 encoded data — NOT safe for git yet

# Step 2: Encrypt it with kubeseal
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# sealed-secret.yaml contains ENCRYPTED data — SAFE for git!
```

#### sealed-secret.yaml (what it looks like)

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: name-sealed-secret
  namespace: sealed-secret
spec:
  encryptedData:
    name: AgBy3i4OJSWK+PiTySYZZA==...  # ← RSA encrypted, not just base64!
```

```bash
# Step 3: Apply to cluster
kubectl apply -f sealed-secret.yaml

# Sealed Secrets controller automatically creates the real K8s Secret
kubectl get secret name-sealed-secret -n sealed-secret
```

---

### 3.3 External Secrets Operator (ESO) — The Preferred Solution (~60% companies)

#### What is it?

ESO bridges **external secret managers** (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault) with Kubernetes Secrets.

#### Architecture

```
┌─────────────────────────────────────────────┐
│                 K8s Cluster                  │
│                                              │
│  ┌──────────────┐    ┌──────────────────┐   │
│  │ SecretStore  │    │ ExternalSecret   │   │
│  │              │    │                  │   │
│  │ - Credentials│◄───│ - References     │   │
│  │   to access  │    │   SecretStore    │   │
│  │   external   │    │ - Specifies which│   │
│  │   vault      │    │   keys to pull   │   │
│  └──────┬───────┘    └────────┬─────────┘   │
│         │                     │              │
│         └──────────┬──────────┘              │
│                    ▼                          │
│          ┌──────────────────┐                │
│          │  ESO Controller  │                │
│          │  (watches both   │                │
│          │   objects)       │                │
│          └────────┬─────────┘                │
│                   │ Creates/Syncs             │
│                   ▼                          │
│          ┌──────────────────┐                │
│          │   K8s Secret     │ ← auto created │
│          │   (real secret)  │                │
│          └──────────────────┘                │
└─────────────────────┬───────────────────────┘
                      │ pulls from
                      ▼
         ┌────────────────────────┐
         │  External Secret Store │
         │  - AWS Secrets Manager │
         │  - GCP Secret Manager  │
         │  - HashiCorp Vault     │
         │  - Azure Key Vault     │
         └────────────────────────┘
```

**In the cluster, two objects are created:**
1. **SecretStore** — stores credentials and connection info to the external vault
2. **ExternalSecret** — references the SecretStore and specifies which secret keys to pull

#### Step-by-Step Workflow

**Step 1: IAM Setup (e.g., GCP)**

Create a GCP Service Account with the role **"Secret Manager Secret Accessor"**.  
This gives K8s permission to pull actual secrets from GCP Secret Manager.  
Download the JSON key file → store it as a K8s secret.

```bash
# Create the GCP service account key as a K8s secret
kubectl create secret generic gcpsm-secret \
  --from-file=secret-access-credentials=key.json
```

**The secret.yaml for the SA key:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gcpsm-secret
  labels:
    type: gcpsm
type: Opaque
stringData:
  secret-access-credentials: |
    {
      "type": "service_account",
      "project_id": "my-project",
      "private_key_id": "...",
      "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...",
      "client_email": "eso-sa@my-project.iam.gserviceaccount.com",
      ...
    }
```

**Step 2: Create SecretStore**

```yaml
# secretstore.yaml
# Follow: https://external-secrets.io/latest/provider/google-secrets-manager/
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secret-store
  namespace: default
spec:
  provider:
    gcpsm:                              # ← Provider: GCP Secret Manager
      auth:
        secretRef:
          secretAccessKeySecretRef:
            name: gcpsm-secret          # ← Reference to our SA key secret
            key: secret-access-credentials
      projectID: my-gcp-project
```

**Step 3: Create ExternalSecret**

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-external-secret
  namespace: default
spec:
  refreshInterval: 1h                   # How often to sync from external store
  secretStoreRef:
    name: gcp-secret-store              # ← Reference to SecretStore (Step 2)
    kind: SecretStore
  target:
    name: my-app-secret                 # ← Name of K8s Secret to create
    creationPolicy: Owner
  data:
  - secretKey: db-password              # Key in the K8s Secret
    remoteRef:
      key: projects/my-project/secrets/db-password  # Path in GCP Secret Manager
      version: latest
```

**Step 4: Automatic K8s Secret Creation**

ESO controller watches ExternalSecret → fetches from GCP → creates/syncs K8s Secret:

```bash
# This K8s Secret is automatically created and kept in sync
kubectl get secret my-app-secret
# → my-app-secret  Opaque  1  5s
```

#### Full Workflow Summary

```
1. IAM user/SA created with permission "Secret Manager Secret Accessor"
   ↓
2. SA JSON key stored as K8s Secret (gcpsm-secret)
   ↓
3. SecretStore created → references gcpsm-secret for auth
   ↓
4. ExternalSecret created → references SecretStore + specifies key paths
   ↓
5. ESO Controller watches both objects → copies/syncs secrets from external store
   ↓
6. K8s Secret automatically created and refreshed
```

---

### 3.4 HashiCorp Vault + CSI Driver

#### CSI Secrets Store Driver

Another approach: Mount secrets directly as files using the **Secrets Store CSI Driver**, bypassing K8s Secrets entirely.

```yaml
# Secret never touches etcd — mounted directly from Vault into pod filesystem
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.vault.svc.cluster.local:8200"
    roleName: "my-app-role"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/my-app"
        secretKey: "password"
```

```yaml
# Pod uses it like a volume
spec:
  containers:
  - name: my-app
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: vault-db-creds
```

---

### 3.5 Comparison: Which Secret Solution to Use?

| Solution | GitOps Safe | Auto-Rotation | External Vault | Best For |
|----------|-------------|---------------|----------------|----------|
| Native K8s Secret | ❌ | ❌ | ❌ | Dev/testing only |
| Sealed Secrets | ✅ | ❌ | ❌ | GitOps, small teams |
| ESO | ✅ | ✅ | ✅ AWS/GCP/Azure/Vault | Production, enterprise |
| Vault CSI Driver | ✅ | ✅ | ✅ Vault only | Vault-first organizations |

---

## 4. TLS, mTLS & Zero Trust

### 4.1 TLS (Transport Layer Security)

TLS is the protocol that provides:
- **Encryption** — data in transit is unreadable to eavesdroppers
- **Authentication** — server proves its identity via certificate
- **Integrity** — data hasn't been tampered with

```
Client                           Server
  │                                │
  │──── ClientHello ──────────────►│
  │◄─── ServerHello + Certificate──│
  │     [Client verifies cert      │
  │      against trusted CA]       │
  │──── Key Exchange ─────────────►│
  │◄─── Finished ──────────────────│
  │                                │
  │════ Encrypted Data ═══════════►│
```

In Kubernetes, TLS is used for:
- Ingress (HTTPS termination)
- API server communication
- etcd encryption
- Internal service-to-service communication (with mTLS)

---

### 4.2 mTLS (Mutual TLS) — Zero Trust

In regular TLS, **only the server** proves its identity.  
In **mTLS**, **both client AND server** prove their identities.

```
Client                           Server
  │──── I am "payment-service" ───►│
  │     [shows its certificate]    │
  │◄─── I am "order-service" ──────│
  │     [shows its certificate]    │
  │     [both verified by CA]      │
  │════ Encrypted + Authenticated═►│
```

**Why mTLS? — Zero Trust Network**

Traditional: "If you're inside the cluster, I trust you"  
Zero Trust: "I don't trust anyone by default — prove who you are every time"

```
Without mTLS:
  payment-service → order-service
  "Hey, I need to charge the customer"
  → order-service just responds (no verification)
  → A compromised pod can impersonate any service!

With mTLS:
  payment-service → order-service
  "I am payment-service, here's my cert signed by cluster CA"
  → order-service verifies cert, responds with its own cert
  → payment-service verifies order-service cert
  → Mutual trust established
  → A compromised pod can't impersonate — it doesn't have the right cert
```

**Zero Trust Principles:**

1. Never trust, always verify
2. Assume breach — even internal traffic is suspect
3. Least privilege access
4. Encrypt everything

mTLS is the implementation of Zero Trust at the network layer — every service-to-service call is authenticated and encrypted regardless of whether it's "inside" the cluster.

---

## 5. Networking Fundamentals — OSI Model

### 5.1 OSI Model Layers

```
┌─────┬──────────────────┬─────────────────────────────────────────┐
│  7  │   Application    │ HTTP, HTTPS, DNS, gRPC, WebSocket        │
│  6  │   Presentation   │ SSL/TLS, encoding, compression           │
│  5  │   Session        │ Session management, authentication        │
│  4  │   Transport      │ TCP, UDP — ports, flow control           │
│  3  │   Network        │ IP — routing between networks             │
│  2  │   Data Link      │ Ethernet, MAC addresses, switches         │
│  1  │   Physical       │ Cables, fiber, Wi-Fi signals              │
└─────┴──────────────────┴─────────────────────────────────────────┘
```

### 5.2 L4 vs L7 — The Most Important Distinction in K8s

#### L4 — Transport Layer

Works with **IP + Port** only. Has NO idea what the data contains.

```
Source: 10.0.0.1:54321
Dest:   10.0.0.5:80
Protocol: TCP

L4 sees: bytes flowing between ports
L4 does NOT know: "this is HTTP", "this is a GET request", "this is login"
```

**K8s L4 tools:**
- `kube-proxy` (NodePort, ClusterIP services)
- AWS NLB (Network Load Balancer)
- TCP-level load balancing

#### L7 — Application Layer

Works with **actual application protocol content**.

```
HTTP GET /api/v1/users
Host: my-app.example.com
Authorization: Bearer eyJ...
Content-Type: application/json

L7 sees: method, path, headers, body, cookies
L7 can: route by path, add auth, retry on 503, circuit break, inject headers
```

**K8s L7 tools:**
- Ingress controllers (nginx, traefik)
- AWS ALB (Application Load Balancer)
- Istio / Envoy proxy (service mesh)

### 5.3 TCP vs UDP

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, retransmits | Best-effort, no retransmit |
| Ordering | Ordered packets | Unordered |
| Speed | Slower (overhead) | Faster |
| Use cases | HTTP, HTTPS, DB connections, SSH | DNS, video streaming, gaming |
| K8s example | All API calls, service traffic | CoreDNS UDP queries |

**The 3-Way TCP Handshake:**

```
Client ──── SYN ──────────────────────► Server   "I want to connect"
Client ◄─── SYN-ACK ─────────────────── Server   "OK, I'm ready"
Client ──── ACK ──────────────────────► Server   "Great, let's talk"
           [Connection Established]
```

---

## 6. Kubernetes Services Deep Dive

### 6.1 Why Services Exist

Pods are **ephemeral** — they die, restart, and get new IPs constantly.

```
Without Service:
  my-app calls → postgres-pod (IP: 10.0.1.5)
  postgres-pod crashes → restarts with IP: 10.0.1.23
  my-app still calling 10.0.1.5 → connection refused!

With Service:
  my-app calls → postgres-service (IP: 10.96.0.50, NEVER changes)
  Service tracks current pod IPs via selectors
  Postgres pod restarts → service updates endpoints automatically
  my-app still calls 10.96.0.50 → works fine!
```

### 6.2 Service Types

#### ClusterIP (Default) — Internal Only

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: ClusterIP
  selector:
    app: postgres           # Routes to pods with this label
  ports:
  - port: 5432             # Service port
    targetPort: 5432       # Pod port
```

Access: `postgres-service.default.svc.cluster.local:5432`  
Only accessible **within the cluster**.

#### NodePort — Exposes on every node

```yaml
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080        # Port on every node (30000-32767)
```

Access: `<any-node-IP>:30080`  
Not for production (exposes on all nodes, no load balancing).

#### LoadBalancer — Production External Access

```yaml
spec:
  type: LoadBalancer       # Creates cloud load balancer (ALB/NLB/GCP LB)
  ports:
  - port: 80
    targetPort: 8080
```

Cloud provider creates external IP/LB automatically.

---

## 7. Service Mesh & Istio

### 7.1 The Monolith Problem

**Monolith Architecture:**

```
┌──────────────────────────────────────┐
│         E-Commerce Monolith          │
│  ┌────────────────────────────────┐  │
│  │ Auth + Users + Products +      │  │
│  │ Orders + Payments + Shipping + │  │
│  │ Notifications + Reviews        │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Problems:**
- One bug can bring down the entire app
- Must redeploy everything for a single change
- Can't scale individual components (payments needs 10x, auth needs 1x)
- One team blocks all others
- Technology lock-in (entire app must be in Java)
- Database shared by everything — hard to change schema

---

### 7.2 Microservices — The Solution

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Auth    │  │  Users   │  │ Products │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Orders  │  │ Payments │  │Shipping  │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘
```

**Benefits:**
- Independent deployment
- Independent scaling
- Technology freedom (Go for payments, Python for ML, Node for API)
- Team autonomy — each team owns their service
- Fault isolation — payment failure doesn't take down product catalog

---

### 7.3 The Microservices Problem — Why Service Mesh?

Now you have 50+ services calling each other. Every developer needs to implement:

```
Without Service Mesh — each dev implements manually:
  ❌ Retries (retry on 503, but not on 400)
  ❌ Circuit breaking (stop calling failing service)
  ❌ Timeouts (don't wait forever)
  ❌ Load balancing (round-robin, least-connection)
  ❌ mTLS (encrypt every call, verify identity)
  ❌ Distributed tracing (trace request across 15 services)
  ❌ Metrics (latency, error rate, request count per service)
  ❌ Rate limiting
  ❌ Canary deployments / traffic splitting
```

This logic gets duplicated in **every service**, in **every language**.  
Java SDK, Go client, Python library — all slightly different, all need updating.

**Service Mesh moves this to infrastructure — not application code.**

---

### 7.4 What Is a Service Mesh?

A service mesh is a **dedicated infrastructure layer** that handles service-to-service communication.

It works by injecting a **sidecar proxy** (Envoy) into every pod.

```
Without Service Mesh:
  [App Container] ──HTTP──► [Other App Container]

With Service Mesh:
  [App Container] → [Envoy Sidecar] ═══mTLS═══► [Envoy Sidecar] → [App Container]
                     ↑ intercepts all traffic      ↑ intercepts all traffic
                     ↑ applies policies             ↑ applies policies
```

The app **doesn't know** the sidecar exists. It still calls `http://order-service:8080`.  
The sidecar intercepts, upgrades to mTLS, applies retry/timeout/circuit-breaking policies, reports metrics.

---

### 7.5 Istio Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Control Plane                       │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │                    Istiod                        │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │    │
│  │  │  Pilot   │  │  Citadel │  │   Galley     │  │    │
│  │  │(traffic  │  │ (certs,  │  │ (config      │  │    │
│  │  │ mgmt)    │  │  mTLS)   │  │  validation) │  │    │
│  │  └──────────┘  └──────────┘  └──────────────┘  │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                            │ pushes config to
                            ▼
┌─────────────────────────────────────────────────────────┐
│                       Data Plane                         │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Pod: payment-service                            │   │
│  │  ┌──────────────────┐  ┌────────────────────┐   │   │
│  │  │   App Container  │  │  Envoy Sidecar     │   │   │
│  │  │   (payment app)  │◄─│  - mTLS            │   │   │
│  │  │                  │  │  - Retries         │   │   │
│  │  │                  │  │  - Circuit Break   │   │   │
│  │  │                  │  │  - Metrics         │   │   │
│  │  │                  │  │  - Tracing         │   │   │
│  │  └──────────────────┘  └────────────────────┘   │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**Istiod components:**
- **Pilot** — service discovery and traffic management rules
- **Citadel** — certificate management, mTLS
- **Galley** — configuration validation

---

### 7.6 Key Istio Features

#### Traffic Management

```yaml
# VirtualService — route 10% to v2, 90% to v1 (canary)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        x-user-beta:
          exact: "true"
    route:
    - destination:
        host: payment-service
        subset: v2
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90
    - destination:
        host: payment-service
        subset: v2
      weight: 10
```

#### Retry Policy

```yaml
http:
- route:
  - destination:
      host: payment-service
  retries:
    attempts: 3
    perTryTimeout: 2s
    retryOn: 503,gateway-error
```

#### Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-cb
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5      # After 5 errors
      interval: 10s                # in 10 seconds
      baseEjectionTime: 30s        # eject from pool for 30s
```

---

### 7.7 Why Not Every Company Uses Service Mesh

| Concern | Reality |
|---------|---------|
| Complexity | Sidecar injection, control plane — steep learning curve |
| Latency overhead | Each hop through Envoy adds ~1ms |
| Resource cost | Each sidecar uses ~50-100MB RAM |
| Debugging | Hard to trace issues through the mesh |
| Overkill | 5-service startup doesn't need circuit breaking |

**When to use Istio:**
- 20+ microservices
- Strict security requirements (mTLS, Zero Trust)
- Need canary deployments / traffic splitting
- Compliance requires audit trails of all service communication
- Multiple teams, need to enforce policies centrally

---

## 8. What Happens When You Type google.com

### 8.1 Full End-to-End Flow

```
User types: https://google.com
```

#### Step 1 — Browser Cache Check

Browser checks its own DNS cache.  
→ If `google.com` IP was resolved recently, use cached IP.  
→ If not cached → proceed to DNS resolution.

#### Step 2 — OS DNS Cache Check

Browser asks the OS resolver.  
OS checks `/etc/hosts` and its own DNS cache.  
→ Found? Use it.  
→ Not found? Ask the configured DNS server.

#### Step 3 — DNS Recursive Resolution

```
Your Machine
     │
     │ "What is the IP of google.com?"
     ▼
Recursive Resolver (your ISP / 8.8.8.8 / 1.1.1.1)
     │
     │ "I don't know, let me ask the root"
     ▼
Root Name Server (.)
     │ "I don't know google.com, but .com is handled by Verisign"
     ▼
TLD Name Server (.com)
     │ "I don't know google.com's IP, but google.com NS is ns1.google.com"
     ▼
Authoritative Name Server (ns1.google.com)
     │ "google.com → 142.250.80.46"
     ▼
Recursive Resolver caches + returns to you
     │
     ▼
Your Browser: IP = 142.250.80.46
```

**DNS Record Types:**
- `A` record: `google.com → 142.250.80.46` (IPv4)
- `AAAA` record: `google.com → 2607:f8b0:4004::200e` (IPv6)
- `CNAME` record: alias (`www.google.com → google.com`)
- `MX` record: mail server
- `TXT` record: verification, SPF

#### Step 4 — TCP Connection (L4)

```
Browser → TCP SYN → 142.250.80.46:443
Google  → SYN-ACK
Browser → ACK
[TCP Connection Established]
```

#### Step 5 — TLS Handshake (L6/L7)

```
Browser → ClientHello (supported cipher suites)
Google  → ServerHello + Certificate (signed by Google's CA)
Browser → Verifies cert against trusted CA store
Browser → Key Exchange (generate session keys)
Google  → Finished
[TLS Session Established — all data now encrypted]
```

#### Step 6 — HTTP/2 Request (L7)

```
GET / HTTP/2
Host: google.com
Accept: text/html
User-Agent: Chrome/126
```

#### Step 7 — Google's Infrastructure Handles the Request

In GCP (Google Cloud Platform):

```
Request arrives at edge PoP (Point of Presence)
     │
     ▼
Google Cloud CDN (checks if response is cached)
     │ Cache miss → forward to origin
     ▼
Google Cloud Load Balancer (GCP HTTP(S) Load Balancer — L7)
     │ Routes based on URL, headers, geo
     ▼
Google Frontend (GFE) — Google's internal L7 proxy
     │ Terminates TLS, routes to appropriate backend
     ▼
Backend Service (search, home page servers)
     │ Generates response
     ▼
Response flows back through same path
     │
     ▼
Browser receives HTML, CSS, JS
     │
     ▼
Browser renders google.com
```

#### Step 8 — DNS in Kubernetes (CoreDNS)

Inside a K8s cluster, DNS works similarly but with CoreDNS:

```
Pod calls: postgres-service.default.svc.cluster.local
     │
     ▼
CoreDNS (10.96.0.10 — the cluster DNS server)
     │
     ▼
Returns ClusterIP of postgres-service: 10.96.45.32
     │
     ▼
kube-proxy (iptables/IPVS rules) translates ClusterIP → actual pod IP
     │
     ▼
Packet routed to postgres pod
```

**K8s DNS search domains** (in /etc/resolv.conf inside pod):
```
search default.svc.cluster.local svc.cluster.local cluster.local
```

This is why you can call `postgres-service` (short name) instead of the full FQDN — the search domain appends `.default.svc.cluster.local` automatically.

#### GCP Services Involved in Serving google.com

| Step | GCP Service | Function |
|------|------------|----------|
| DNS | Google Cloud DNS | Authoritative DNS for google.com |
| Edge | Cloud CDN + PoPs | Cache static content globally |
| Load Balancing | Cloud HTTP(S) LB | L7 global load balancing |
| TLS | Google-managed SSL certs | TLS termination |
| Networking | Cloud VPC | Internal network routing |
| Backends | GCE / GKE | Actual compute running the app |
| DDoS | Google Cloud Armor | WAF and DDoS protection |

---

## 📌 Quick Reference — Key Commands

```bash
# ConfigMap
kubectl create configmap my-config --from-literal=key=value
kubectl get configmap my-config -o yaml
kubectl describe configmap my-config

# Secrets
kubectl create secret generic my-secret --from-literal=password=secret123
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
kubectl describe secret my-secret   # values NOT shown

# Sealed Secrets
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
kubectl apply -f sealed-secret.yaml

# External Secrets
kubectl get externalsecret
kubectl get secretstore
kubectl describe externalsecret my-external-secret

# Service Mesh (Istio)
kubectl get virtualservice
kubectl get destinationrule
kubectl get peerauthentication    # mTLS policies
istioctl analyze                  # check for misconfigurations
istioctl proxy-status             # sidecar sync status
```

---

## 🔗 Official Documentation

| Topic | URL |
|-------|-----|
| ConfigMaps | https://kubernetes.io/docs/concepts/configuration/configmap/ |
| Secrets | https://kubernetes.io/docs/concepts/configuration/secret/ |
| Encrypting Secrets at Rest | https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/ |
| Sealed Secrets | https://github.com/bitnami-labs/sealed-secrets |
| External Secrets Operator | https://external-secrets.io/latest/ |
| ESO GCP Provider | https://external-secrets.io/latest/provider/google-secrets-manager/ |
| ESO AWS Provider | https://external-secrets.io/latest/provider/aws-secrets-manager/ |
| Istio | https://istio.io/latest/docs/ |
| Istio Traffic Management | https://istio.io/latest/docs/concepts/traffic-management/ |
| CoreDNS | https://coredns.io/plugins/kubernetes/ |
| Secrets Store CSI Driver | https://secrets-store-csi-driver.sigs.k8s.io/ |

---

## 🎯 Interview Questions & Answers (30+)

> Covers all topics in this document. Questions range from L1 (basic) to L3 (senior/architect level).

---

### 🟢 ConfigMap Questions

---

**Q1. What is a ConfigMap and why do we use it instead of hardcoding values in the image?**

**Answer:**

A ConfigMap is a Kubernetes API object that stores non-confidential configuration data as key-value pairs, completely decoupled from the container image.

Without ConfigMaps, if you need different DB URLs for dev, staging, and prod, you'd build three different images — causing image drift and maintenance overhead. With ConfigMaps, you build one image and inject environment-specific config at runtime.

Key rule: ConfigMaps are for **non-sensitive** data only. Never store passwords, tokens, or keys in a ConfigMap.

---

**Q2. What are the two ways to consume a ConfigMap in a Pod, and what is the critical difference between them?**

**Answer:**

Two ways:

1. **Environment Variables** — via `env.valueFrom.configMapKeyRef` or `envFrom.configMapRef`
2. **Volume Mount** — via `volumes.configMap` and `volumeMounts`

**Critical difference — update behavior:**

| Method | Auto-updates when CM changes? |
|--------|-------------------------------|
| Environment Variable | ❌ NO — requires pod restart |
| Volume Mount | ✅ YES — kubelet syncs within ~1-2 minutes |

**Why?** Env vars are set at container start and baked into the process's environment. The kernel does not provide a mechanism to update a running process's env vars. Volume mounts use a symlink mechanism that kubelet periodically rotates.

```bash
# After updating a CM consumed as env var, you MUST:
kubectl rollout restart deployment my-app
```

---

**Q3. Can you store a full config file (like nginx.conf) in a ConfigMap? How?**

**Answer:**

Yes. Use the multi-line string syntax in the `data` field:

```yaml
data:
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }
```

Then mount it as a volume:

```yaml
volumeMounts:
- name: nginx-config
  mountPath: /etc/nginx/nginx.conf
  subPath: nginx.conf             # mount only this key as a file
volumes:
- name: nginx-config
  configMap:
    name: nginx-configmap
```

The `subPath` field is critical — without it, the entire directory is replaced, removing other files in `/etc/nginx/`.

---

**Q4. What happens if a ConfigMap referenced by a Pod does not exist?**

**Answer:**

The Pod fails to start with a `CreateContainerConfigError`. The kubelet cannot create the container because the required ConfigMap is not found.

To make a reference optional (pod starts even if CM missing):

```yaml
envFrom:
- configMapRef:
    name: optional-config
    optional: true          # Pod starts even if CM doesn't exist
```

---

### 🟡 Secrets Questions

---

**Q5. What is the difference between a ConfigMap and a Secret in Kubernetes?**

**Answer:**

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| Data type | Non-sensitive config | Sensitive credentials |
| Storage in etcd | Plain text | Base64 encoded (can be encrypted) |
| RBAC | Standard access | Can restrict independently |
| Display in `describe` | Values shown | Values hidden |
| Volume mount | World-readable | tmpfs (in-memory, not on disk) |

Both are key-value stores, but Secrets use base64 encoding and support additional security controls like encryption at rest and tighter RBAC policies.

---

**Q6. Explain the statement: "Kubernetes Secrets are encoded, not encrypted." Why is this important?**

**Answer:**

Base64 is a **reversible encoding format** — it is NOT encryption. It's just a way to represent binary data as ASCII text.

```bash
# Anyone can decode in 1 second:
echo "cHJlcml0" | base64 -d
# Output: prerit
```

**Why this matters:**
- Anyone with `kubectl get secret` permission can extract the full value instantly
- Secrets stored in etcd are readable by anyone with etcd access
- If etcd backups are stolen, all secrets are compromised

**The real security comes from:**
1. RBAC — restrict who can `get/list` secrets
2. Encryption at Rest — enable `EncryptionConfiguration` on the API server so etcd stores AES-encrypted data
3. External secret managers (Vault, AWS SM) — don't store secrets in etcd at all

---

**Q7. What are the different Secret types in Kubernetes? When do you use each?**

**Answer:**

| Type | kubectl flag | Use Case |
|------|-------------|----------|
| `Opaque` | `generic` | Default — any key-value data (DB passwords, API keys) |
| `kubernetes.io/dockerconfigjson` | `docker-registry` | Image pull credentials from private registries |
| `kubernetes.io/tls` | `tls` | TLS certificate + private key (for Ingress) |
| `kubernetes.io/ssh-auth` | `generic` | SSH private keys |
| `kubernetes.io/service-account-token` | auto-created | Service account JWT tokens |

**Opaque** is the most common — use it for any custom secret where no specific type exists.

---

**Q8. How do you create a secret and verify it was created correctly?**

**Answer:**

```bash
# Create
kubectl create secret generic db-creds \
  --from-literal=username=prerit \
  --from-literal=password=MyS3cr3t

# Verify (values are base64, not shown in describe)
kubectl get secret db-creds
kubectl describe secret db-creds     # shows keys but NOT values

# Decode a specific value
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

The `describe` command intentionally hides values. You must use `-o jsonpath` or `-o yaml` to see the encoded data, then decode manually.

---

**Q9. What are the problems with native Kubernetes Secrets at scale? Why do ~60% of companies use external solutions?**

**Answer:**

**Problem 1 — Secret Rotation:**
When a password rotates, you must manually update the K8s Secret and restart all pods using it as an env var. At 50+ services, this is error-prone and time-consuming.

**Problem 2 — No Audit Trail:**
Native K8s Secrets don't log who accessed what secret and when. This fails SOC2, PCI DSS, and HIPAA audit requirements.

**Problem 3 — Not GitOps-safe:**
You cannot commit a Secret YAML to Git because it contains base64-encoded (easily decoded) sensitive data.

**Problem 4 — Secret Sprawl:**
Secrets duplicated across namespaces and clusters with no single source of truth.

**Problem 5 — Encoding ≠ Encryption:**
Without explicit encryption at rest configuration, etcd stores secrets in base64 only.

**Solutions used in production:**
- Sealed Secrets (encrypt for Git)
- External Secrets Operator (sync from Vault/AWS SM/GCP SM)
- HashiCorp Vault + CSI Driver (bypass etcd entirely)

---

**Q10. You have a database password stored in AWS Secrets Manager. How do you make it available to a K8s pod without storing it in etcd?**

**Answer:**

Use the **Secrets Store CSI Driver** with the AWS provider. This mounts the secret directly from AWS Secrets Manager into the pod as a file, bypassing Kubernetes Secrets and etcd entirely.

```yaml
# SecretProviderClass
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-db-secret
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/db/password"
        objectType: "secretsmanager"
```

```yaml
# Pod consumes it
volumes:
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: aws-db-secret
```

The secret is mounted at `/mnt/secrets/password` — it never enters etcd, never gets base64-encoded in K8s, and is pulled fresh from AWS SM on every pod start.

---

### 🔵 Sealed Secrets & ESO Questions

---

**Q11. What is the problem that Sealed Secrets solves? How does it work?**

**Answer:**

**Problem:** You want to follow GitOps — store all K8s manifests in Git. But you can't commit Secrets to Git because they're only base64-encoded (not encrypted).

**Solution:** Sealed Secrets uses **asymmetric encryption (RSA)**. The controller in the cluster holds the private key. Developers use the public key to encrypt secrets before committing.

**Flow:**
```
Developer:
  kubectl create secret --dry-run -o yaml > secret.yaml
  kubeseal < secret.yaml > sealed-secret.yaml  ← RSA encrypted
  git commit sealed-secret.yaml                 ← SAFE to commit

Cluster:
  kubectl apply -f sealed-secret.yaml
  Sealed Secrets Controller decrypts with private key
  Creates real K8s Secret
```

The encrypted value is **cluster-specific** — a sealed secret encrypted for cluster A cannot be decrypted by cluster B (different private key). This prevents cross-environment secret leakage.

---

**Q12. What is the External Secrets Operator? Explain the SecretStore and ExternalSecret objects.**

**Answer:**

ESO is a K8s operator that syncs secrets from external secret managers (AWS SM, GCP SM, Vault, Azure Key Vault) into K8s Secrets automatically.

**Two core objects:**

**SecretStore** — defines HOW to connect to the external provider (credentials, region, project):
```yaml
kind: SecretStore
spec:
  provider:
    gcpsm:
      projectID: my-project
      auth:
        secretRef:
          secretAccessKeySecretRef:
            name: gcpsm-secret
            key: secret-access-credentials
```

**ExternalSecret** — defines WHAT to pull and WHERE to create the K8s Secret:
```yaml
kind: ExternalSecret
spec:
  secretStoreRef:
    name: gcp-secret-store
  target:
    name: my-k8s-secret      # K8s Secret that gets created
  data:
  - secretKey: db-password   # key in K8s Secret
    remoteRef:
      key: prod/db/password  # path in GCP Secret Manager
```

ESO controller watches ExternalSecrets, fetches from external store, and creates/updates K8s Secrets. The `refreshInterval` controls how often it syncs.

---

**Q13. What is the difference between Sealed Secrets and External Secrets Operator? When would you choose one over the other?**

**Answer:**

| | Sealed Secrets | External Secrets Operator |
|---|---|---|
| Source of truth | Git (encrypted) | External vault (Vault/AWS SM/GCP SM) |
| Encryption | RSA (kubeseal) | Handled by external provider |
| Auto-rotation | ❌ No | ✅ Yes (refreshInterval) |
| Audit trail | ❌ No | ✅ Yes (from external provider) |
| Offline capable | ✅ Yes (just apply YAML) | ❌ Needs external provider access |
| Setup complexity | Low | Medium-High |

**Choose Sealed Secrets when:**
- Small team, no existing secret manager
- GitOps workflow, want secrets in Git
- Simple use case, no rotation requirement

**Choose ESO when:**
- Company already uses Vault / AWS SM / GCP SM
- Need automatic rotation
- Compliance requires audit trails
- Multi-cluster environment with centralized secrets

---

### 🟠 TLS & mTLS Questions

---

**Q14. What is the difference between TLS and mTLS? Give a real K8s use case for each.**

**Answer:**

**TLS (one-way):** Only the **server** proves its identity. Client remains anonymous.
- Use case: Browser → Ingress (HTTPS). The browser verifies the server's cert. The server doesn't verify who the browser is.

**mTLS (mutual):** **Both** client and server prove their identities with certificates.
- Use case: payment-service → order-service inside the cluster. Both services exchange and verify certs. A compromised pod cannot impersonate another service.

```
TLS:    Client ──verifies server cert──► Server
mTLS:   Client ◄──verifies each other──► Server (both directions)
```

**In Istio**, mTLS is enforced via PeerAuthentication:

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT    # All traffic in this namespace MUST use mTLS
```

---

**Q15. What is Zero Trust networking? How is it implemented in Kubernetes?**

**Answer:**

Zero Trust is a security model based on: **"Never trust, always verify."** Unlike perimeter-based security (trust everything inside the network), Zero Trust assumes any component can be compromised.

**Principles:**
1. Every request must be authenticated (mTLS identity verification)
2. Every request must be authorized (RBAC/ABAC policies)
3. Least privilege — only allow what's explicitly needed
4. Encrypt everything — even internal east-west traffic

**Implementation in K8s:**
- **mTLS via Istio/Linkerd** — every service authenticates itself with a certificate
- **NetworkPolicy** — whitelist only necessary pod-to-pod communication
- **RBAC** — fine-grained access to K8s API resources
- **OPA/Gatekeeper** — admission policies for what can run in the cluster

```yaml
# NetworkPolicy: deny all by default, then whitelist
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}     # applies to ALL pods
  policyTypes:
  - Ingress
  - Egress
  # No rules = deny everything
```

---

### 🔴 Networking & OSI Questions

---

**Q16. What is the difference between L4 and L7 load balancing? Give examples of each in Kubernetes.**

**Answer:**

**L4 (Transport Layer):**
- Operates on IP + Port only
- Has NO knowledge of application protocol (HTTP, gRPC, etc.)
- Routes based on: source IP, destination IP, port, TCP/UDP
- Fast, low overhead
- **K8s examples:** `kube-proxy` (ClusterIP/NodePort services), AWS NLB

**L7 (Application Layer):**
- Operates on full HTTP/gRPC/WebSocket payload
- Can route based on: URL path, Host header, HTTP method, cookies, JWT claims
- Can do: SSL termination, retries, circuit breaking, header manipulation
- More overhead but much more powerful
- **K8s examples:** Ingress controllers (nginx, traefik), AWS ALB, Istio Envoy proxy

**Real example:**

```
L4: Route all traffic on port 443 → backend pods (no path awareness)

L7: 
  /api/v1/users → user-service
  /api/v1/orders → order-service
  /api/v1/payments → payment-service
  (same host, different paths → different services)
```

---

**Q17. Explain the difference between TCP and UDP. Which does DNS use and why?**

**Answer:**

| | TCP | UDP |
|---|---|---|
| Connection | 3-way handshake required | No handshake |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Packets arrive in order | May arrive out of order |
| Overhead | Higher (headers, ACKs) | Lower |

**DNS uses UDP (port 53) for most queries** because:
- DNS queries are small (< 512 bytes typically)
- Speed is critical — you need IP resolution fast
- The overhead of a TCP handshake is unnecessary for a request-response that fits in one packet
- If the response is too large (DNSSEC, many records), it falls back to TCP

**In K8s:** CoreDNS receives UDP queries from pods for service discovery. Pod queries `postgres-service` → CoreDNS responds with ClusterIP via UDP in microseconds.

---

**Q18. How does DNS resolution work inside a Kubernetes cluster? What is the role of CoreDNS?**

**Answer:**

Every Pod in K8s has `/etc/resolv.conf` injected by kubelet:

```
nameserver 10.96.0.10           # CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
```

**Resolution flow when a pod calls `postgres-service`:**

1. Pod sends DNS query to `10.96.0.10` (CoreDNS)
2. CoreDNS checks search domains — tries `postgres-service.default.svc.cluster.local`
3. CoreDNS has this record (because the Service exists) → returns ClusterIP `10.96.45.32`
4. Pod connects to `10.96.45.32`
5. `kube-proxy` (iptables/IPVS rules) translates ClusterIP → actual pod IP
6. Packet routed to the correct pod

**Full FQDN format:** `<service>.<namespace>.svc.<cluster-domain>`

```
postgres-service.default.svc.cluster.local
                 ^^^^^^^   ^^^
                 namespace cluster domain (default: cluster.local)
```

Cross-namespace calls must use full name: `postgres-service.database.svc.cluster.local`

---

### 🟣 Service Mesh & Istio Questions

---

**Q19. Why do microservices need a service mesh? What problem does it solve that application code can't?**

**Answer:**

In a microservices architecture, every service needs cross-cutting concerns: retries, timeouts, circuit breaking, mTLS, tracing, rate limiting. Without a service mesh, each team implements this in their service code.

**The problem:**
- Java team uses Resilience4j for circuit breaking
- Go team uses a different library
- Python team implements it slightly differently
- All three need to be updated when policies change
- You now have 3 implementations of the same logic, all slightly inconsistent

**What service mesh does:**
Moves all of this OUT of application code and INTO infrastructure (sidecar proxy).

```
Without mesh: App code = business logic + retry + circuit break + mTLS + tracing
With mesh:    App code = business logic ONLY
              Sidecar = retry + circuit break + mTLS + tracing
```

**Result:**
- Developers focus only on business logic
- Ops team manages all cross-cutting concerns centrally
- Policy changes apply to ALL services without code changes
- Consistent behavior regardless of language/framework

---

**Q20. Explain the Istio sidecar proxy pattern. How does Envoy intercept traffic without the application knowing?**

**Answer:**

When Istio injection is enabled on a namespace, Istiod's **MutatingWebhookConfiguration** intercepts every new pod creation and injects two containers:

1. **`istio-init`** (init container) — sets up iptables rules that redirect ALL inbound/outbound traffic to Envoy's port (15001/15006)
2. **`istio-proxy`** (sidecar) — Envoy proxy that handles the actual traffic

**iptables magic:**
```
Pod wants to connect to order-service:8080
  → iptables rule redirects to localhost:15001 (Envoy)
  → Envoy applies policies (mTLS, retry, timeout, tracing)
  → Envoy connects to actual order-service
  → Response comes back through Envoy
  → App receives response as if it connected directly
```

The application calls `http://order-service:8080` as normal. It has zero knowledge of Envoy. The iptables rules are transparent to the application process.

**Envoy gets its configuration from Istiod (xDS API):**
- LDS (Listener Discovery Service) — what ports to listen on
- RDS (Route Discovery Service) — routing rules
- CDS (Cluster Discovery Service) — upstream services
- EDS (Endpoint Discovery Service) — pod IPs

---

**Q21. What is a VirtualService in Istio? How would you do a canary deployment with 10% traffic to v2?**

**Answer:**

A VirtualService is an Istio resource that defines routing rules for traffic going to a service. It works alongside a DestinationRule (which defines the subsets/versions).

```yaml
# DestinationRule — defines subsets
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

```yaml
# VirtualService — 10% canary to v2
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90
    - destination:
        host: payment-service
        subset: v2
      weight: 10
```

This splits 10% of traffic to pods labeled `version: v2` and 90% to `v1`, without changing replica counts. You can gradually increase the weight as confidence grows.

---

**Q22. What is a circuit breaker pattern? How does Istio implement it?**

**Answer:**

**Circuit Breaker** is a pattern that stops sending requests to a failing service, giving it time to recover. Like an electrical circuit breaker — when current spikes, it trips to prevent damage.

**States:**
```
CLOSED (normal): requests flow through
    ↓ (too many failures)
OPEN (tripped): all requests fail fast, no calls to upstream
    ↓ (after timeout)
HALF-OPEN: allow one test request
    ↓ success → CLOSED
    ↓ failure → OPEN
```

**Istio implementation via DestinationRule:**

```yaml
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5    # After 5 consecutive 5xx errors
      interval: 10s              # measured in a 10s window
      baseEjectionTime: 30s      # eject the pod for 30 seconds
      maxEjectionPercent: 50     # eject at most 50% of pods
```

When a pod returns 5 consecutive 5xx responses in 10 seconds, Istio removes it from the load balancing pool for 30 seconds. This prevents cascading failures across services.

---

**Q23. Monolith vs Microservices — what are the trade-offs? When should you NOT use microservices?**

**Answer:**

**Monolith advantages:**
- Simple to develop, test, deploy
- Easy to debug (single process, single log stream)
- No network latency between components
- ACID transactions across all data
- Appropriate for small teams (< 10 engineers)

**Microservices advantages:**
- Independent deployment and scaling
- Technology freedom per service
- Fault isolation
- Team autonomy
- Can scale payment service 10x without touching user service

**When NOT to use microservices:**
- Team is < 5 engineers (operational overhead > benefit)
- Product is still finding product-market fit (frequent pivots)
- You don't yet know the domain boundaries (premature decomposition creates wrong boundaries)
- No DevOps maturity (can't manage 20+ deployments)

**Martin Fowler's rule:** "Don't start with microservices. Start with a well-structured monolith. Split when you feel the pain."

The cost of microservices: distributed tracing complexity, network latency, eventual consistency, service discovery, and the need for a service mesh — all of which add significant operational overhead.

---

### 🌐 DNS & "What Happens at google.com" Questions

---

**Q24. Walk me through every step that happens between typing google.com in your browser and seeing the page.**

**Answer:**

1. **Browser DNS cache** — checks if `google.com` was resolved recently
2. **OS DNS cache** — checks `/etc/hosts` and OS resolver cache
3. **Recursive DNS resolution:**
   - Query goes to your configured DNS resolver (ISP or 8.8.8.8)
   - Resolver asks Root Name Server (.) → "who handles .com?"
   - Root refers to Verisign TLD server (.com)
   - TLD server refers to Google's authoritative NS (ns1.google.com)
   - Authoritative NS returns A record: `google.com → 142.250.80.46`
   - Response cached at resolver and returned to browser
4. **TCP 3-way handshake** → SYN / SYN-ACK / ACK on port 443
5. **TLS handshake** → cipher negotiation, server cert verification, session key exchange
6. **HTTP/2 GET request** → `GET / HTTP/2 Host: google.com`
7. **Google infrastructure** → Cloud CDN → HTTP(S) Load Balancer → GFE → Backend servers
8. **HTML response** → browser parses, fetches sub-resources (JS/CSS/images), renders page

---

**Q25. What is the difference between an A record, CNAME record, and MX record?**

**Answer:**

| Record | Maps | Example |
|--------|------|---------|
| `A` | Domain → IPv4 address | `google.com → 142.250.80.46` |
| `AAAA` | Domain → IPv6 address | `google.com → 2607:f8b0::200e` |
| `CNAME` | Domain → another domain (alias) | `www.google.com → google.com` |
| `MX` | Domain → mail server | `google.com → aspmx.l.google.com` |
| `TXT` | Domain → arbitrary text | SPF records, domain verification |
| `NS` | Domain → name server | `google.com → ns1.google.com` |

**Important CNAME rule:** You cannot use a CNAME at the zone apex (root domain). `google.com` cannot be a CNAME. `www.google.com` can. This is why many providers offer `ALIAS` or `ANAME` records as a workaround.

---

**Q26. How does kube-proxy work? What happens when a Pod calls a ClusterIP Service?**

**Answer:**

`kube-proxy` runs on every node and maintains **iptables** (or IPVS) rules that implement Service load balancing.

**Flow when pod calls `ClusterIP 10.96.45.32:5432`:**

1. Pod sends packet destined for `10.96.45.32:5432`
2. Packet hits kernel networking stack
3. **iptables PREROUTING chain** — kube-proxy has written DNAT rules:
   ```
   -d 10.96.45.32/32 -p tcp --dport 5432 -j DNAT --to-destination 10.0.1.7:5432
   ```
4. Destination is rewritten to actual pod IP `10.0.1.7:5432` (randomly selected from endpoints)
5. Packet routes to the correct pod

**iptables vs IPVS:**
- iptables: O(n) rule evaluation — slow at 10,000+ services
- IPVS: hash table — O(1) regardless of service count, supports more LB algorithms

```bash
# View the rules kube-proxy creates
iptables -t nat -L KUBE-SERVICES
```

---

### ⚙️ Advanced / Architect-Level Questions

---

**Q27. A developer says "I updated the ConfigMap but my app still shows the old config." How do you diagnose and fix this?**

**Answer:**

**Step 1 — Determine how the CM is consumed:**

```bash
kubectl describe pod <pod-name> | grep -A5 "Environment\|Mounts"
```

**Case A — Consumed as Environment Variable:**

Env vars do NOT auto-update. This is expected behavior.

```bash
# Confirm CM was actually updated
kubectl get configmap my-config -o yaml

# Restart the deployment to pick up new values
kubectl rollout restart deployment my-app

# Verify new pods have correct env
kubectl exec -it <new-pod> -- env | grep MY_KEY
```

**Case B — Consumed as Volume Mount (but still showing old value):**

Volume mounts DO auto-update, but:
- Sync can take 1-2 minutes (kubelet's `syncFrequency`)
- The app may have already read the file at startup and cached it in memory
- Some apps need a reload signal (nginx needs `nginx -s reload`)

```bash
# Check current value in the mounted file
kubectl exec -it <pod> -- cat /etc/config/MY_KEY

# If file is updated but app shows old: app needs to re-read the file
# Send SIGHUP or use app-specific reload mechanism
```

---

**Q28. How would you design a secret management strategy for a production K8s cluster handling PCI DSS compliance?**

**Answer:**

PCI DSS requires: encryption at rest, access logging, rotation, least-privilege access.

**Architecture:**

```
AWS Secrets Manager / HashiCorp Vault (Centralized Secret Store)
    ↑ RBAC per secret per service account
    ↑ Full audit logging (who accessed what, when)
    ↑ Automatic rotation every 90 days

External Secrets Operator
    ↑ Syncs secrets into K8s with refreshInterval: 1h

K8s RBAC
    ↑ ServiceAccounts per microservice
    ↑ No service can read another service's secrets

etcd Encryption at Rest
    ↑ EncryptionConfiguration with AES-CBC or KMS provider
    ↑ Even if etcd is compromised, data is unreadable

Secrets Store CSI Driver (for highest-sensitivity secrets)
    ↑ Never enter etcd at all
    ↑ Mounted directly from Vault into pod

Network Policy
    ↑ Only pods with correct SA can reach secret endpoints
```

**Key decisions:**
- Never commit secrets to Git (even base64)
- Rotate all secrets every 90 days (PCI requirement)
- Audit log every secret access
- Use separate Vault paths per environment (dev/staging/prod)

---

**Q29. Explain what happens when an Istio-injected Pod starts. Walk through the sidecar injection process.**

**Answer:**

1. Developer runs `kubectl apply -f pod.yaml`
2. API server receives the request
3. **MutatingAdmissionWebhook** fires — Istiod's webhook (`istio-sidecar-injector`) intercepts
4. Istiod checks if the namespace has `istio-injection: enabled` label
5. If yes, Istiod **mutates** the Pod spec before it's persisted:
   - Adds `istio-init` init container (sets iptables rules)
   - Adds `istio-proxy` container (Envoy)
   - Mounts config volumes
6. Modified Pod spec is persisted to etcd
7. Scheduler assigns Pod to a node
8. kubelet on the node creates the containers:
   - `istio-init` runs → `iptables -t nat -A OUTPUT -p tcp -j ISTIO_OUTPUT` (redirects all traffic to Envoy port 15001)
   - `istio-proxy` starts → Envoy connects to Istiod and requests xDS config
   - App container starts → all its traffic is transparently intercepted by Envoy

```bash
# Verify injection worked
kubectl describe pod my-app | grep istio-proxy
# Should show istio-proxy container alongside your app
```

---

**Q30. What is the difference between a Kubernetes Ingress and an Istio VirtualService? Can they be used together?**

**Answer:**

| | Ingress | Istio VirtualService |
|---|---|---|
| Layer | L7 | L7 |
| Scope | North-south (external → cluster) | East-west (service → service) AND north-south |
| Traffic splitting | Basic (percentage by annotation) | Fine-grained (header, weight, mirror) |
| Requires | Ingress Controller (nginx) | Istio service mesh |
| Retry/timeout | Controller-dependent | Built-in, consistent |
| mTLS | TLS termination only | Full mTLS end-to-end |

**Can they be used together?**

Yes — common pattern:

```
Internet → Istio Ingress Gateway (replaces Ingress) → VirtualService → Services
```

Or hybrid:

```
Internet → nginx Ingress → cluster network → Istio VirtualService (east-west only)
```

**Istio Ingress Gateway** is preferred when you're already running Istio — it gives you full observability, traffic shaping, and mTLS all the way from edge to pod.

---

**Q31. How does Kubernetes handle DNS for headless services? When would you use one?**

**Answer:**

A headless service is created with `clusterIP: None`. Instead of creating a stable VirtualIP, it returns the **individual pod IPs** directly from DNS.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None         # ← headless
  selector:
    app: postgres
  ports:
  - port: 5432
```

**DNS behavior:**

```
Regular Service:
  nslookup postgres-service → 10.96.45.32 (single ClusterIP, kube-proxy does LB)

Headless Service:
  nslookup postgres-headless → 10.0.1.5, 10.0.1.7, 10.0.1.9 (all pod IPs)
```

**Use cases:**
- **StatefulSets** — each pod needs a stable DNS identity (`postgres-0.postgres-headless.default.svc.cluster.local`)
- **Custom client-side load balancing** — gRPC clients that maintain persistent connections need all pod IPs
- **Service discovery** — when the application itself handles routing logic (Cassandra, Elasticsearch cluster formation)
- **Databases in active-passive mode** — app needs to connect to the primary specifically, not a random pod

---

**Q32. What is the difference between `kubectl apply` and `kubectl create`? What happens on re-apply?**

**Answer:**

| | `kubectl create` | `kubectl apply` |
|---|---|---|
| Behavior | Creates the resource | Creates if not exists, updates if exists |
| If resource exists | **Fails** with AlreadyExists error | **Updates** the resource (3-way merge patch) |
| Use case | One-time creation | GitOps / CI-CD pipelines |
| Idempotent | ❌ No | ✅ Yes |

**`kubectl apply` 3-way merge:**

Compares:
1. **Last applied config** (stored in annotation `kubectl.kubernetes.io/last-applied-configuration`)
2. **Current live state** (what's in the cluster)
3. **New desired state** (your YAML file)

Calculates a patch and applies only the diff. This means fields modified by operators or autoscalers won't be overwritten if they're not in your YAML.

```bash
# In CI/CD always use apply (idempotent):
kubectl apply -f deployment.yaml

# For one-off operations:
kubectl create secret generic temp-secret --from-literal=key=value
```

---

**Q33. A pod can't connect to another service. Walk through your debugging steps.**

**Answer:**

**Step 1 — Verify the service exists and has endpoints:**

```bash
kubectl get service my-service
kubectl get endpoints my-service
# If endpoints is empty → service selector doesn't match pod labels
```

**Step 2 — Check pod labels vs service selector:**

```bash
kubectl describe service my-service | grep Selector
kubectl get pods --show-labels | grep my-app
# Labels must match exactly
```

**Step 3 — Test DNS resolution:**

```bash
kubectl exec -it debug-pod -- nslookup my-service
kubectl exec -it debug-pod -- nslookup my-service.default.svc.cluster.local
# If DNS fails → CoreDNS issue
```

**Step 4 — Test TCP connectivity:**

```bash
kubectl exec -it debug-pod -- curl http://my-service:8080/health
kubectl exec -it debug-pod -- nc -zv my-service 8080
```

**Step 5 — Check NetworkPolicy:**

```bash
kubectl get networkpolicy -n default
# NetworkPolicy may be blocking the connection
```

**Step 6 — Check if target pod is healthy:**

```bash
kubectl get pods -l app=my-service
kubectl logs <target-pod>
```

**Step 7 — Check kube-proxy and iptables:**

```bash
kubectl get pods -n kube-system | grep kube-proxy
iptables -t nat -L KUBE-SERVICES | grep my-service
```

**Step 8 — If Istio is present, check sidecar sync:**

```bash
istioctl proxy-status
istioctl analyze
```

---

*End of Interview Questions — 33 questions covering ConfigMaps, Secrets, Sealed/External Secrets, TLS/mTLS, OSI/L4/L7, DNS, Service Mesh, Istio, and Debugging.*

---

*This document covers: ConfigMap · Secrets · Encoding vs Encryption · Sealed Secrets · External Secrets Operator · HashiCorp Vault · CSI Driver · TLS · mTLS · Zero Trust · OSI Model · L4/L7 · TCP/UDP · Service Mesh · Istio · Microservices · DNS Resolution · What happens at google.com · 33 Interview Q&A*
