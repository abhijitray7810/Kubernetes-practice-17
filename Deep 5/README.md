
# 🔐 Kubernetes: TLS, mTLS, Certificates, Ingress & Gateway API — Deep Dive

---

## Table of Contents

1. [K8s Architecture: The 8-Layer Stack](#1-k8s-architecture-the-8-layer-stack)
2. [Public Key, Private Key & Certificates — Fundamentals](#2-public-key-private-key--certificates--fundamentals)
3. [Certificate Authority (CA) — How It Works](#3-certificate-authority-ca--how-it-works)
4. [TLS — Transport Layer Security](#4-tls--transport-layer-security)
5. [mTLS — Mutual TLS](#5-mtls--mutual-tls)
6. [cert-manager — Certificate Lifecycle in Kubernetes](#6-cert-manager--certificate-lifecycle-in-kubernetes) 
7. [ClusterIssuer — In-Cluster CA Deployment](#7-clusterissuer--in-cluster-ca-deployment)
8. [Certificate.yml — Full Workflow](#8-certificateyml--full-workflow) 
9. [TLS Secret Store — tls.crt & tls.key](#9-tls-secret-store--tlscrt--tlskey)
10. [Ingress — TLS Termination & Routing](#10-ingress--tls-termination--routing) 
11. [Ingress Problem Use Cases](#11-ingress-problem-use-cases) 
12. [Gateway API — Modern Traffic Management](#12-gateway-api--modern-traffic-management)
13. [Gateway API Use Cases](#13-gateway-api-use-cases)
14. [Noisy Neighbor Problem & OOM](#14-noisy-neighbor-problem--oom)
15. [Interview Questions & Answers](#15-interview-questions--answers)

---

## 1. K8s Architecture: The 8-Layer Stack

### Why Do We Need This?

Kubernetes organizes workloads in a strict hierarchy. Understanding each layer is essential for debugging, scaling, and securing applications.

### Definition

The 8-layer model describes how Kubernetes encapsulates compute resources from the cluster level down to the container image.

### Deep Dive

```
1. Cluster       → The entire Kubernetes environment (control plane + worker nodes)
2. Node          → A physical or virtual machine running kubelet, kube-proxy
3. Namespace     → Logical isolation boundary (dev, staging, prod)
4. Deployment    → Declarative desired-state controller (manages ReplicaSets)
5. ReplicaSet    → Ensures N copies of a pod are always running
6. Pod           → Smallest deployable unit (1+ containers sharing network/storage)
7. Container     → Isolated process running inside a Pod (Docker, containerd)
8. Image         → Immutable snapshot of filesystem + app + dependencies
```

### Layer Relationship

```
Cluster
 └── Node (worker-1, worker-2)
      └── Namespace (default, kube-system, prod)
           └── Deployment (my-app)
                └── ReplicaSet (my-app-7d9f)
                     └── Pod (my-app-7d9f-xk2p)
                          └── Container (nginx, sidecar)
                               └── Image (nginx:1.25-alpine)
```

---

## 2. Public Key, Private Key & Certificates — Fundamentals

### Why Do We Need This?

Without cryptographic identity, any service could impersonate another. TLS certificates prove "I am who I say I am" and encrypt data in transit.

### Definition

- **Private Key** — Secret key kept only by the owner. Used to decrypt data or sign certificates. Never shared.
- **Public Key** — Derived from the private key. Shared openly. Used to encrypt data or verify signatures.
- **Certificate (x.509)** — A document binding a public key to an identity, signed by a trusted CA.

### Deep Dive — How Client Talks to Server

```
CLIENT                                   SERVER
  |                                         |
  |──── 1. Hello (supported TLS versions) ─►|
  |                                         |
  |◄─── 2. Server Certificate (Public Key) ─|
  |                                         |
  |  3. Client verifies cert via CA chain   |
  |                                         |
  |──── 4. Session Key (encrypted with      |
  |         server's public key) ──────────►|
  |                                         |
  |  5. Server decrypts session key         |
  |     with its Private Key                |
  |                                         |
  |◄═══ 6. Encrypted communication (AES) ══►|
```

### Key Commands

```bash
# Generate private key
openssl genrsa -out server.key 2048

# Generate Certificate Signing Request (CSR)
openssl req -new -key server.key -out server.csr \
  -subj "/CN=my-service/O=my-org"

# View certificate details
openssl x509 -in server.crt -text -noout
```

---

## 3. Certificate Authority (CA) — How It Works

### Why Do We Need This?

Without a CA, there is no trust chain. Any service could generate its own cert and claim any identity. A CA acts as a trusted third party that vouches for certificate validity.

### Definition

A **Certificate Authority (CA)** is an entity that issues, signs, and revokes digital certificates. It is the root of the PKI (Public Key Infrastructure) trust chain.

### CA Types

| Type | Examples | Usage |
|------|----------|-------|
| Public CA | Let's Encrypt, DigiCert, Comodo | Internet-facing services |
| Private / In-house CA | cert-manager, Vault, CFSSL | Internal Kubernetes services |
| Intermediate CA | Signed by Root CA | Issues end-entity certs |

### How CA Signs a Certificate

```
1. Service generates Private Key + CSR (Certificate Signing Request)
2. CSR sent to CA (contains: domain, org, public key)
3. CA verifies identity
4. CA signs CSR with its own Private Key → produces signed Certificate (.crt)
5. Service presents this signed Certificate to clients
6. Client verifies cert by checking CA's Public Key (in trust store)
```

### Let's Encrypt — Free Public CA

```
- ACME protocol (Automated Certificate Management Environment)
- Validates domain ownership via HTTP-01 or DNS-01 challenge
- Issues 90-day certificates
- Fully automated renewal via cert-manager
- Free and open source
```

### Other Public CAs

```
- DigiCert    → Enterprise-grade, extended validation (EV) certs
- Comodo/Sectigo → Low cost, wildcard certs
- GlobalSign  → Strong enterprise PKI
- GoDaddy     → Domain + cert bundles
```

---

## 4. TLS — Transport Layer Security

### Why Do We Need This?

Plain HTTP transmits data as cleartext. TLS encrypts data so that eavesdroppers cannot read or modify it in transit.

### Definition

**TLS (Transport Layer Security)** is a cryptographic protocol that provides privacy, integrity, and authenticity for data transmitted over a network. It is the successor to SSL.

### TLS Handshake Flow

```
Client                          Server
  |                               |
  |──── ClientHello ─────────────►|  (TLS version, cipher suites, random)
  |                               |
  |◄─── ServerHello ──────────────|  (chosen cipher, random)
  |◄─── Certificate ──────────────|  (server's signed cert)
  |◄─── ServerHelloDone ──────────|
  |                               |
  | [Client verifies cert via CA] |
  |                               |
  |──── ClientKeyExchange ───────►|  (pre-master secret, encrypted w/ server pubkey)
  |──── ChangeCipherSpec ────────►|
  |──── Finished ────────────────►|
  |                               |
  |◄─── ChangeCipherSpec ─────────|
  |◄─── Finished ─────────────────|
  |                               |
  |◄═══ Encrypted App Data ══════►|  (AES session key)
```

### Session Key

- **Session Key** — A temporary symmetric key (AES-128/256) generated per-connection.
- Derived from the handshake's pre-master secret.
- Faster than asymmetric encryption for bulk data.
- Discarded at session end.

---

## 5. mTLS — Mutual TLS

### Why Do We Need This?

Standard TLS only proves the **server's** identity to the client. In microservices, we also need the **client** to prove its identity to the server. mTLS enables zero-trust service-to-service authentication.

### Definition

**mTLS (Mutual TLS)** extends TLS by requiring **both** client and server to present and verify each other's certificates.

### mTLS Flow

```
CLIENT (with cert)                      SERVER (with cert)
  |                                         |
  |──── 1. ClientHello ────────────────────►|
  |◄─── 2. ServerHello + Server Cert ───────|
  |◄─── 3. CertificateRequest ──────────────|  ← Server asks for client cert
  |                                         |
  |──── 4. Client Certificate ─────────────►|  ← Client proves identity
  |──── 5. ClientKeyExchange ──────────────►|
  |──── 6. CertificateVerify ──────────────►|
  |                                         |
  |  [Both sides verify each other's cert   |
  |   against the same CA]                  |
  |                                         |
  |◄═══════ Encrypted App Data ════════════►|
```

### mTLS vs TLS

| Feature | TLS | mTLS |
|---------|-----|------|
| Server verified | ✅ | ✅ |
| Client verified | ❌ | ✅ |
| Use case | Browser → Website | Service → Service |
| Zero Trust | ❌ | ✅ |

### mTLS in Kubernetes — Istio / Linkerd

```yaml
# Istio PeerAuthentication — enforce mTLS in namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # All traffic must use mTLS
```

---

## 6. cert-manager — Certificate Lifecycle in Kubernetes

### Why Do We Need This?

Manually managing TLS certificates is error-prone and doesn't scale. cert-manager automates issuance, renewal, and storage of certificates inside Kubernetes.

### Definition

**cert-manager** is a Kubernetes add-on that automates the management of TLS certificates from various issuers (Let's Encrypt, Vault, self-signed, in-house CA).

### cert-manager Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                 │
│                                                     │
│  Certificate.yml ──► CertificateRequest ──► Order   │
│       ▼                                    ▼        │
│  cert-manager                        ACME Challenge  │
│  Controller                          (HTTP-01/DNS-01)│
│       ▼                                    ▼        │
│  ClusterIssuer ──────────────────► CA / Let's Encrypt│
│       ▼                                             │
│  Signed Certificate ──► TLS Secret (tls.crt+tls.key)│
└─────────────────────────────────────────────────────┘
```

### Install cert-manager

```bash
# Install via Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Verify
kubectl get pods -n cert-manager
kubectl get crds | grep cert-manager
```

---

## 7. ClusterIssuer — In-Cluster CA Deployment

### Why Do We Need This?

A ClusterIssuer defines **how** cert-manager should obtain certificates. It's a cluster-wide resource that allows any namespace to request certificates from the same issuer.

### Definition

**ClusterIssuer** is a cert-manager CRD (Custom Resource Definition) that configures a certificate issuing backend. Unlike `Issuer` (namespace-scoped), `ClusterIssuer` works across all namespaces.

### ClusterIssuer Types

```
1. SelfSigned   → Issues certs signed by themselves (bootstrap / dev)
2. CA           → Uses a secret containing a CA keypair (in-house CA)
3. ACME         → Let's Encrypt or compatible ACME servers
4. Vault        → HashiCorp Vault PKI backend
5. Venafi        → Venafi cloud/TPP
```

### ClusterIssuer.yml — In-House CA

```yaml
# Step 1: Create CA secret
kubectl create secret tls my-ca-secret \
  --cert=ca.crt \
  --key=ca.key \
  -n cert-manager

---
# Step 2: ClusterIssuer using in-house CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: my-ca-issuer
spec:
  ca:
    secretName: my-ca-secret   # Secret holding ca.crt + ca.key
```

### ClusterIssuer.yml — Let's Encrypt (ACME)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key   # ACME account key
    solvers:
    - http01:
        ingress:
          class: nginx    # Which ingress controller handles the challenge
```

```bash
# Verify ClusterIssuer is ready
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

---

## 8. Certificate.yml — Full Workflow

### Why Do We Need This?

A `Certificate` resource tells cert-manager what certificate to create, from which issuer, for which domains, and where to store it.

### Definition

**Certificate** is a cert-manager CRD that describes the desired state of a TLS certificate. cert-manager continuously reconciles it.

### Certificate.yml

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-app-tls
  namespace: production
spec:
  secretName: my-app-tls-secret    # Where to store tls.crt + tls.key
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: my-app.example.com
  dnsNames:
  - my-app.example.com
  - www.my-app.example.com
  duration: 2160h       # 90 days
  renewBefore: 360h     # Renew 15 days before expiry
```

### Full Certificate Generation Workflow

```
i.   Certificate.yml deployed
      ↓
ii.  cert-manager creates CertificateRequest
      ↓
iii. CertificateRequest → ACME Order (for Let's Encrypt)
      ↓
iv.  Proxy/Ingress serves ACME HTTP-01 challenge at:
     http://my-app.example.com/.well-known/acme-challenge/<token>
      ↓
v.   Let's Encrypt verifies domain ownership
      ↓
vi.  CA signs the certificate
      ↓
vii. cert-manager stores result in Kubernetes Secret:
     Secret: my-app-tls-secret
       tls.crt  →  signed certificate (PEM)
       tls.key  →  private key (PEM)
      ↓
viii. Ingress/Gateway reads Secret and terminates TLS
```

```bash
# Monitor certificate status
kubectl get certificate -n production
kubectl describe certificate my-app-tls -n production

# Check the resulting secret
kubectl get secret my-app-tls-secret -n production
kubectl describe secret my-app-tls-secret -n production
```

---

## 9. TLS Secret Store — tls.crt & tls.key

### Why Do We Need This?

Kubernetes stores TLS credentials in a Secret of type `kubernetes.io/tls`. Ingress controllers and Gateways reference this secret to terminate TLS.

### Definition

A **TLS Secret** is a Kubernetes Secret containing exactly two keys: `tls.crt` (certificate chain) and `tls.key` (private key), both base64-encoded.

### TLS Secret Structure

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate-chain>   # server cert + intermediate CA
  tls.key: <base64-encoded-private-key>         # server private key
```

### Manually Create TLS Secret

```bash
# From files
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n production

# View decoded cert
kubectl get secret my-tls-secret -n production \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout
```

---

## 10. Ingress — TLS Termination & Routing

### Why Do We Need This?

Without Ingress, every service would need its own LoadBalancer IP. Ingress provides a single entry point that routes HTTP/HTTPS traffic to multiple services based on hostname and path.

### Definition

**Ingress** is a Kubernetes API object that manages external access to services, typically HTTP/HTTPS. It provides load balancing, SSL termination, and name-based virtual hosting.

### Ingress Flow

```
Internet User types: https://my-app.example.com
        ↓
[DNS] my-app.example.com → Ingress Controller IP
        ↓
[Ingress Controller] (nginx, traefik, haproxy)
        ↓
  Reads Ingress rules:
  - host: my-app.example.com → backend: my-app-svc:80
  - TLS termination: secret: my-app-tls-secret
        ↓
[TLS Terminated] → plain HTTP to backend service
        ↓
[Service] → Pod
```

### Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # Auto-cert
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - my-app.example.com
    secretName: my-app-tls-secret    # TLS secret reference
  rules:
  - host: my-app.example.com        # Hostname check
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-svc        # Which service to route to
            port:
              number: 80
```

### cert-manager + Ingress Automated Flow

```
i.   Ingress has annotation: cert-manager.io/cluster-issuer
ii.  cert-manager detects Ingress → auto-creates Certificate CRD
iii. Certificate → CertificateRequest → ACME challenge via Ingress
iv.  CA signs → Secret stored (tls.crt + tls.key)
v.   Ingress reads Secret → TLS termination active
vi.  Already-running server behind proxy: TLS handled at edge, plain HTTP inside
```

---

## 11. Ingress Problem Use Cases

### Why Do We Need This?

Ingress has real-world limitations and failure modes that every Kubernetes engineer must understand.

### Common Ingress Issues

```
Problem 1: 404 / 503 Backend Not Found
─────────────────────────────────────
Cause:    Service name/port mismatch in Ingress rule
Fix:      kubectl get svc -n production
          Match: backend.service.name + backend.service.port

Problem 2: TLS Certificate Not Trusted
───────────────────────────────────────
Cause:    Self-signed cert, expired cert, missing CA chain
Fix:      kubectl describe certificate <name>
          kubectl get events -n cert-manager

Problem 3: ACME Challenge Failing
──────────────────────────────────
Cause:    HTTP-01 challenge: /.well-known/acme-challenge not reachable
Fix:      Check firewall rules (port 80 must be open)
          Check Ingress controller is handling challenge path

Problem 4: Ingress Class Conflict
──────────────────────────────────
Cause:    Multiple ingress controllers (nginx + traefik) without class annotation
Fix:      Set ingressClassName: nginx explicitly

Problem 5: Wildcard vs Specific Host
──────────────────────────────────────
Cause:    *.example.com cert used on sub.sub.example.com (doesn't match)
Fix:      Use SAN (Subject Alternative Names) with specific hostnames

Problem 6: Path Routing Issues
────────────────────────────────
Cause:    pathType: Exact vs Prefix mismatch
Fix:      Use pathType: Prefix for /api → /api/v1, /api/v2
```

```bash
# Debug Ingress
kubectl describe ingress my-app-ingress -n production
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
kubectl get events -n production --sort-by='.lastTimestamp'
```

---

## 12. Gateway API — Modern Traffic Management

### Why Do We Need This?

Ingress is limited: no traffic splitting, no header-based routing, no cross-namespace routes, and poor extensibility. Gateway API is the next-generation replacement with richer expressive power.

### Definition

**Gateway API** is a Kubernetes SIG-Network project that provides role-oriented, expressive, and extensible APIs for traffic management. It separates infrastructure (Gateway) from routing (HTTPRoute).

### Gateway API Resources

```
GatewayClass  → Defines which load balancer implementation to use
Gateway       → The actual load balancer instance (L4/L7)
HTTPRoute     → HTTP routing rules (host, path, headers, weights)
TCPRoute      → TCP routing
TLSRoute      → TLS passthrough routing
GRPCRoute     → gRPC routing
```

### kubectl Commands

```bash
# List GatewayClasses (which LB implementations are available)
kubectl get gatewayclass

# Example output:
# NAME              CONTROLLER                      ACCEPTED
# nginx             k8s.io/ingress-nginx            True
# istio             istio.io/gateway-controller     True
# cilium            io.cilium/gateway-controller    True

# List Gateways
kubectl get gateway -A

# List HTTPRoutes
kubectl get httproute -A
```

### GatewayClass — Which Load Balancer Runs on Your Backend

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway
spec:
  controllerName: k8s.io/ingress-nginx   # ← This defines which LB runs your traffic
```

```
gatewayClassName: nginx-gateway    → Nginx Ingress Controller
gatewayClassName: istio            → Istio (with mTLS + observability)
gatewayClassName: cilium           → Cilium eBPF-based gateway
gatewayClassName: kong             → Kong API Gateway
```

### Gateway.yml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: production
spec:
  gatewayClassName: nginx-gateway
  listeners:
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: my-app-tls-secret    # TLS secret
    hostname: my-app.example.com
```

### HTTPRoute.yml — Routing Rules

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
  namespace: production
spec:
  parentRefs:
  - name: my-gateway            # ← Which Gateway handles this route
    namespace: production
  hostnames:
  - "my-app.example.com"        # Hostname check
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
      weight: 100               # Traffic weight (for canary)
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
```

### Gateway API — Deployment Stages

```
Stage 1: Install Gateway API CRDs
────────────────────────────────
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

Stage 2: Deploy Load Balancer (GatewayClass implementation)
───────────────────────────────────────────────────────────
helm install nginx-gateway nginx-stable/nginx-gateway-fabric

Stage 3: Create GatewayClass
─────────────────────────────
kubectl apply -f gatewayclass.yml

Stage 4: Create Gateway (listener + TLS config)
────────────────────────────────────────────────
kubectl apply -f gateway.yml

Stage 5: Create HTTPRoute (routing rules)
──────────────────────────────────────────
kubectl apply -f httproute.yml

Stage 6: Traffic flows:
  Internet → GatewayClass LB → Gateway → HTTPRoute → Service → Pod
```

### Rollout / Canary with Gateway API

```yaml
# Split traffic: 90% stable, 10% canary
backendRefs:
- name: my-app-stable
  port: 80
  weight: 90
- name: my-app-canary
  port: 80
  weight: 10
```

---

## 13. Gateway API Use Cases

### Why Do We Need This?

Gateway API solves real-world traffic scenarios that Ingress cannot handle cleanly.

### Use Case Comparison

```
Use Case 1: Canary / Blue-Green Deployment
──────────────────────────────────────────
Ingress:     ❌ Requires annotations (vendor-specific, hacky)
Gateway API: ✅ weight field in backendRefs (native, standardized)

Use Case 2: Cross-Namespace Routing
─────────────────────────────────────
Ingress:     ❌ Routes and backends must be in same namespace
Gateway API: ✅ parentRefs can reference Gateway in another namespace

Use Case 3: Header-Based Routing
──────────────────────────────────
Ingress:     ❌ Not supported in standard spec
Gateway API: ✅ matches.headers supported natively

Use Case 4: Role Separation
────────────────────────────
Ingress:     ❌ Single resource for infra + routing
Gateway API: ✅ Infra team owns GatewayClass + Gateway
             ✅ Dev team owns HTTPRoute (per namespace)

Use Case 5: Multiple Protocols
────────────────────────────────
Ingress:     ❌ HTTP/HTTPS only
Gateway API: ✅ HTTP, HTTPS, TCP, TLS, gRPC, WebSocket

Use Case 6: End User Domain Routing Goal
─────────────────────────────────────────
User types: https://my-app.example.com
→ DNS → Gateway IP
→ Gateway listener matches hostname
→ HTTPRoute rules: hostname + path → backend service
→ Service → Pod
→ Response back to user
```

### parentRefs — Traffic Route to Gateway

```yaml
# parentRefs attaches HTTPRoute to a specific Gateway
spec:
  parentRefs:
  - name: my-gateway          # Gateway name
    namespace: production      # Gateway namespace
    sectionName: https        # Specific listener on the Gateway
```

---

## 14. Noisy Neighbor Problem & OOM

### Why Do We Need This?

In Kubernetes, multiple pods share the same node's CPU and memory. A misbehaving pod can starve others — this is the "noisy neighbor" problem. OOM (Out of Memory) kills are its most visible symptom.

### Definition

**Noisy Neighbor**: A pod consuming excessive CPU or memory on a shared node, degrading the performance of co-located pods.

**OOM (Out of Memory) Kill**: The Linux kernel OOM killer terminates processes when physical memory is exhausted. Kubernetes uses container memory limits to trigger this earlier.

### Detecting the Problem

```bash
# List all pods across all namespaces — find OOMKilled pods
kubectl get pods -A

# Look for pods with status OOMKilled or CrashLoopBackOff
kubectl describe pod <pod-name> -n <namespace>
# Check: "Last State: OOMKilled"

# Check node resource pressure
kubectl describe node <node-name>
# Look for: MemoryPressure, DiskPressure conditions

# Top resources
kubectl top pods -A --sort-by=memory
kubectl top nodes
```

### Solution — Resource Requests & Limits

```yaml
# Always set requests AND limits
resources:
  requests:
    memory: "128Mi"   # Guaranteed minimum (used for scheduling)
    cpu: "250m"       # 0.25 CPU cores
  limits:
    memory: "256Mi"   # Hard cap — OOMKilled if exceeded
    cpu: "500m"       # Throttled if exceeded (not killed)
```

```
requests: Scheduler uses this to place pod on a node with enough capacity
limits:   Kernel enforces this — memory limit breach = OOMKill
```

### LimitRange — Namespace Defaults

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      memory: "256Mi"
      cpu: "500m"
    defaultRequest:
      memory: "128Mi"
      cpu: "250m"
    max:
      memory: "1Gi"
      cpu: "2"
```

### ResourceQuota — Namespace Budget

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.memory: "8Gi"
    requests.cpu: "4"
    limits.memory: "16Gi"
    limits.cpu: "8"
    pods: "50"
```

---

## 15. Interview Questions & Answers

---

### 🔑 Section A — PKI & TLS

**Q1: What is the difference between a public key and a private key?**

> A private key is a secret mathematical value used by its owner to decrypt data or sign certificates. A public key is derived from the private key and shared openly; it encrypts data or verifies signatures. Data encrypted with one key can only be decrypted by the other.

**Q2: What is a CSR and what does it contain?**

> A CSR (Certificate Signing Request) is a message sent to a CA requesting a signed certificate. It contains the applicant's public key, identity information (CN, O, SAN), and is signed with the applicant's private key to prove key ownership.

**Q3: Explain the TLS handshake in simple terms.**

> The client says hello with its supported algorithms. The server responds with its certificate (public key). The client verifies the cert against a trusted CA. Both sides agree on a session key (using asymmetric encryption to exchange it securely). All subsequent communication uses this symmetric session key (AES), which is much faster.

**Q4: What is a session key? Why is it used instead of the public/private key pair for all communication?**

> A session key is a temporary symmetric AES key generated per-connection. Asymmetric encryption (RSA) is computationally expensive and slow for bulk data. Symmetric encryption (AES) is ~100x faster. TLS uses asymmetric crypto only to securely exchange the session key, then switches to AES for all data transfer.

**Q5: What is the CA chain / chain of trust?**

> Root CA → signs → Intermediate CA → signs → End-entity Certificate. Clients trust Root CAs (stored in OS/browser trust stores). They verify the chain from end-entity cert up to a trusted Root. This allows CAs to issue millions of certs without exposing the Root CA private key.

---

### 🔐 Section B — mTLS

**Q6: What is the difference between TLS and mTLS?**

> In TLS, only the server proves its identity to the client. In mTLS (Mutual TLS), both client and server present certificates and verify each other. mTLS is used for service-to-service authentication in microservices architectures.

**Q7: When would you use mTLS in Kubernetes?**

> mTLS is used when you need zero-trust security between services. Istio and Linkerd implement mTLS transparently as a sidecar, so services communicate securely without application code changes. It's essential in regulated industries (finance, healthcare) or when services span multiple teams/namespaces.

**Q8: How does Istio implement mTLS?**

> Istio injects an Envoy sidecar into each pod. The sidecar intercepts all inbound/outbound traffic. Istiod (control plane) distributes certificates to each sidecar via the xDS API. When pod A calls pod B, A's sidecar presents its cert and verifies B's sidecar cert — without the application knowing.

---

### 📜 Section C — cert-manager

**Q9: What is cert-manager and what problem does it solve?**

> cert-manager is a Kubernetes controller that automates TLS certificate lifecycle management — issuance, renewal, and storage. Without it, engineers manually create CSRs, submit to CAs, download certs, and create secrets. cert-manager eliminates all of that.

**Q10: What is the difference between Issuer and ClusterIssuer?**

> `Issuer` is namespace-scoped — it can only issue certs for `Certificate` resources in the same namespace. `ClusterIssuer` is cluster-scoped — any namespace can reference it. In production, use `ClusterIssuer` for shared issuers like Let's Encrypt.

**Q11: Walk me through how cert-manager gets a Let's Encrypt certificate.**

> 1. You deploy a `Certificate` resource referencing a `ClusterIssuer` pointing to Let's Encrypt ACME. 2. cert-manager creates a `CertificateRequest`. 3. cert-manager creates an `Order` with Let's Encrypt. 4. Let's Encrypt issues an ACME challenge (HTTP-01: serve a token at `/.well-known/acme-challenge/<token>`). 5. cert-manager creates a temporary Ingress or solves via DNS. 6. Let's Encrypt verifies. 7. CA signs and returns the certificate. 8. cert-manager stores it as a Kubernetes Secret with `tls.crt` and `tls.key`.

**Q12: What happens if a certificate expires?**

> cert-manager automatically renews certificates before expiry. The `renewBefore` field (e.g., 360h = 15 days) triggers renewal. The Secret is updated in-place, and any resource referencing it (Ingress, Gateway) picks up the new cert automatically.

---

### 🌐 Section D — Ingress

**Q13: What is an Ingress Controller and why is it needed?**

> An Ingress object is just a configuration — it does nothing without an Ingress Controller. The controller (nginx, traefik, haproxy) is a running pod that watches Ingress resources and actually configures the load balancer. Without a controller, Ingress rules are ignored.

**Q14: How does TLS termination work with Ingress?**

> The Ingress controller reads the `tls.secretName` field, loads the `tls.crt` and `tls.key` from the Kubernetes Secret, and configures its listener to terminate HTTPS. The connection from the client is decrypted at the Ingress controller, and traffic goes to the backend service as plain HTTP.

**Q15: A user types your domain but gets a 404. How do you debug?**

> Check: 1) DNS resolves to Ingress controller IP (`nslookup domain`). 2) Ingress rule has correct host and path (`kubectl describe ingress`). 3) Backend service exists and is running (`kubectl get svc`). 4) Service selector matches pod labels. 5) Pod is running and healthy (`kubectl get pods`). 6) Ingress controller logs for the error.

**Q16: What is the difference between `pathType: Exact` and `pathType: Prefix`?**

> `Exact` matches only the exact path string. `Prefix` matches any path starting with the prefix. For example, Prefix `/api` matches `/api`, `/api/v1`, `/api/users`. Exact `/api` only matches `/api`.

---

### 🚦 Section E — Gateway API

**Q17: Why was Gateway API created if Ingress already exists?**

> Ingress has serious limitations: no traffic splitting, no protocol support beyond HTTP/HTTPS, no cross-namespace routing, and extensibility only via vendor-specific annotations. Gateway API is role-oriented (infra team owns Gateway, dev team owns HTTPRoute), expressive (native canary, header routing), and extensible without annotations.

**Q18: What is GatewayClass and what does `gatewayClassName` mean?**

> GatewayClass defines the implementation (which load balancer). `gatewayClassName` in a Gateway resource says "use this implementation." For example, `gatewayClassName: nginx` means the Nginx Gateway Fabric controller manages this Gateway. The value directly maps to which load balancer runs on the backend.

**Q19: How is HTTPRoute different from Ingress rules?**

> HTTPRoute is a separate resource from the Gateway. It supports: native traffic splitting via `weight`, cross-namespace routing via `parentRefs`, header/query param matching, multiple parent Gateways, and is managed by app teams independently from infra teams. Ingress combines routing and infrastructure in one resource.

**Q20: How does `parentRefs` work in HTTPRoute?**

> `parentRefs` attaches an HTTPRoute to a specific Gateway (and optionally a specific listener on that Gateway). Traffic matching the HTTPRoute's hostnames and rules is processed by the referenced Gateway. Multiple HTTPRoutes can reference the same Gateway, and one HTTPRoute can reference multiple Gateways.

**Q21: How would you do a canary deployment using Gateway API?**

> Deploy two services (stable + canary). In HTTPRoute, add both as `backendRefs` with weights: stable at 90, canary at 10. Gateway API splits 10% of traffic to the canary. Gradually increase canary weight as confidence grows. No annotations, no Ingress controller patches — fully standardized.

---

### 🔇 Section F — Noisy Neighbor & OOM

**Q22: What is the noisy neighbor problem in Kubernetes?**

> When pods on the same node compete for CPU and memory without limits, a single pod can consume all available resources, degrading or crashing other pods. This is called the noisy neighbor problem — analogous to a loud neighbor affecting others in a shared apartment.

**Q23: How do you find OOMKilled pods?**

> Run `kubectl get pods -A` and look for `OOMKilled` or `CrashLoopBackOff` status. Then `kubectl describe pod <name>` shows "Last State: Terminated, Reason: OOMKilled." `kubectl top pods -A --sort-by=memory` shows current consumption. Check node pressure with `kubectl describe node`.

**Q24: What is the difference between resource requests and limits?**

> **Requests**: The minimum guaranteed resources. Used by the scheduler to place pods on nodes with sufficient capacity. **Limits**: The maximum a container can use. For memory, exceeding the limit causes OOMKill. For CPU, exceeding causes throttling (not kill). Always set both — requests without limits leaves nodes unprotected.

**Q25: A pod is getting OOMKilled repeatedly. What do you do?**

> 1. Check current memory usage: `kubectl top pod`. 2. Analyze application memory profile (heap dumps, profiling). 3. Increase memory limit if the app genuinely needs more. 4. Fix memory leak in application code if present. 5. Use Vertical Pod Autoscaler (VPA) to automatically right-size. 6. Check if JVM/Node.js heap settings align with container limits.

---

## Quick Reference — Key Commands

```bash
# Certificate management
kubectl get certificate -A
kubectl describe certificate <name> -n <ns>
kubectl get certificaterequest -A
kubectl get clusterissuer

# TLS secrets
kubectl get secret -n <ns> --field-selector type=kubernetes.io/tls
kubectl describe secret <tls-secret> -n <ns>

# Ingress
kubectl get ingress -A
kubectl describe ingress <name> -n <ns>
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

# Gateway API
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
kubectl describe gateway <name> -n <ns>

# Noisy neighbor / OOM
kubectl get pods -A
kubectl top pods -A --sort-by=memory
kubectl top nodes
kubectl describe node <node-name>
kubectl get events -A --sort-by='.lastTimestamp' | grep OOM

# cert-manager debug
kubectl get events -n cert-manager
kubectl logs -n cert-manager deploy/cert-manager
```

---

## Summary — End-to-End TLS Flow in Kubernetes

```
1. ClusterIssuer.yml     → Defines CA (Let's Encrypt / in-house)
2. Certificate.yml       → Requests cert for domain
3. CertificateRequest    → cert-manager submits CSR to CA
4. ACME Challenge        → Proxy/Ingress serves challenge
5. CA Signs              → Certificate issued
6. Secret Store          → tls.crt + tls.key stored in K8s Secret
7. Ingress/Gateway       → Reads Secret, terminates TLS
8. User request          → HTTPS → Ingress → HTTP → Pod → Response
```

---

*Generated for Kubernetes Deep Dive Study — TLS, mTLS, Gateway API, cert-manager, Ingress*
