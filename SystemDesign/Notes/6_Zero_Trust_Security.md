# Zero Trust Security in Kubernetes

> **Prerequisites**: [K8s RBAC](../../Kubernetes/Notes/13_RBAC_Security.md), [4_Service_Mesh.md](./4_Service_Mesh.md)  
> **Next**: [7_System_Design_Example.md](./7_System_Design_Example.md)

Traditional security uses a "castle and moat" approach — one perimeter, trust everything inside. Zero trust says: **never trust, always verify** — even between services in the same cluster.

---

## Castle & Moat vs Zero Trust

```mermaid
graph TB
    subgraph Traditional["Traditional: Castle & Moat"]
        Firewall["Firewall<br/>(perimeter)"]
        subgraph Inside["Inside = Trusted ❌"]
            T1["Service A"] <-->|"unencrypted"| T2["Service B"]
            T2 <-->|"unencrypted"| T3["Database"]
        end
        Firewall --> Inside
        Note1["One breach = game over"]
    end

    subgraph ZeroTrust["Zero Trust"]
        subgraph Verified["Every call verified ✅"]
            Z1["Service A"] -->|"mTLS + RBAC"| Z2["Service B"]
            Z2 -->|"mTLS + NetworkPolicy"| Z3["Database"]
            Z1 -.->|"❌ blocked"| Z3
        end
        Note2["Breach contained,<br/>lateral movement blocked"]
    end

    style Traditional fill:#FFCDD2
    style ZeroTrust fill:#C8E6C9
```

---

## The 5 Layers of K8s Security

```mermaid
graph TB
    L1["Layer 1: Network Policies<br/>Who can talk to whom?"]
    L2["Layer 2: mTLS (Service Mesh)<br/>Encrypt + verify identity"]
    L3["Layer 3: RBAC<br/>Who can do what in the cluster?"]
    L4["Layer 4: Pod Security Standards<br/>What can pods do?"]
    L5["Layer 5: Secret Management<br/>How are credentials stored/accessed?"]

    L1 --> L2 --> L3 --> L4 --> L5

    style L1 fill:#E3F2FD
    style L2 fill:#FFF3E0
    style L3 fill:#E8F5E9
    style L4 fill:#F3E5F5
    style L5 fill:#FCE4EC
```

---

# Layer 1: Network Policies — Pod-Level Firewall

By default, all pods can talk to all pods. **Start with deny-all, then whitelist.**

### Default Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}          # All pods
  policyTypes:
    - Ingress
    - Egress
```

### Allow Specific Traffic

```yaml
# Frontend → Backend (only on port 8080)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
---
# Backend → Database (only on port 5432)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - port: 5432
---
# Allow DNS for all pods (required!)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

### Visual Result

```mermaid
graph LR
    Frontend["frontend"] -->|"✅ port 8080"| Backend["backend"]
    Backend -->|"✅ port 5432"| DB["database"]
    Frontend -.->|"❌ blocked"| DB
    Backend -.->|"❌ blocked"| Frontend
    DB -.->|"❌ blocked"| Frontend
    DB -.->|"❌ blocked"| Backend

    style Frontend fill:#E3F2FD
    style Backend fill:#FFF3E0
    style DB fill:#E8F5E9
```

---

# Layer 2: mTLS — Encrypt All Internal Traffic

Without mTLS, traffic between pods is **plain text** on the cluster network. Anyone who compromises a node can sniff all traffic.

```mermaid
graph LR
    subgraph NoMTLS["Without mTLS"]
        A1["Pod A"] -->|"HTTP (plain text)"| B1["Pod B"]
        Attacker1["🔓 Attacker can<br/>sniff traffic"]
    end

    subgraph WithMTLS["With mTLS"]
        A2["Pod A<br/>(cert: identity-a)"] -->|"HTTPS (encrypted)<br/>mutual authentication"| B2["Pod B<br/>(cert: identity-b)"]
        Attacker2["🔒 Attacker sees<br/>encrypted gibberish"]
    end

    style NoMTLS fill:#FFCDD2
    style WithMTLS fill:#C8E6C9
```

### Enable with Istio

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT      # All traffic must be mTLS
```

With Istio, each pod gets an auto-rotated X.509 certificate. The Envoy sidecar handles TLS handshake — **zero code changes**.

---

# Layer 3: RBAC — Least Privilege

Every workload should use its own ServiceAccount with minimal permissions.

```yaml
# Dedicated ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
---
# Minimal Role — only what it needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
    resourceNames: ["order-config"]   # Only specific ConfigMap
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames: ["order-secrets"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: order-service
roleRef:
  kind: Role
  name: order-role
  apiGroup: rbac.authorization.k8s.io
---
# Pod spec
spec:
  serviceAccountName: order-service
  automountServiceAccountToken: false   # Don't mount unless needed
```

---

# Layer 4: Pod Security Standards

Control what pods are allowed to do at the OS level.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### What "Restricted" Enforces

| Restriction | Why |
|------------|-----|
| Must run as non-root | Prevents container breakout |
| Must drop all capabilities | No kernel-level privileges |
| Read-only root filesystem | Prevents writing malware |
| No hostNetwork/hostPID/hostIPC | Prevents accessing host |
| No privileged containers | Prevents full host access |
| Seccomp profile required | Limits system calls |

### Pod Security Context

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: api
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp            # Writable tmp dir
  volumes:
    - name: tmp
      emptyDir: {}
```

---

# Layer 5: Secret Management

```mermaid
graph TB
    subgraph Bad["❌ Bad Practices"]
        B1["Secrets in git"]
        B2["Secrets in env vars<br/>(visible in crash dumps)"]
        B3["Base64 in Secret YAML<br/>(not encrypted!)"]
    end

    subgraph Good["✅ Good Practices"]
        G1["External Secrets Operator<br/>→ Vault / AWS SM / Azure KV"]
        G2["Mount secrets as files<br/>(not env vars)"]
        G3["Enable encryption at rest<br/>for etcd"]
        G4["RBAC: restrict Secret access"]
        G5["Rotate secrets regularly"]
    end

    style Bad fill:#FFCDD2
    style Good fill:#C8E6C9
```

### External Secrets Operator

Syncs secrets from Vault/AWS/Azure into K8s Secrets automatically:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: ClusterSecretStore
  target:
    name: db-credentials          # K8s Secret name
  data:
    - secretKey: password
      remoteRef:
        key: production-db-password
```

---

## Complete Zero Trust Architecture

```mermaid
graph TB
    Internet["🌐 Internet"]
    WAF["WAF / DDoS Protection"]
    Ingress["Ingress Controller<br/>+ TLS + Rate Limit"]

    subgraph Cluster["K8s Cluster (Zero Trust)"]
        subgraph NS["Namespace: production"]
            FE["Frontend<br/>🔒 restricted PSS<br/>🔑 fe-sa"]
            BE["Backend<br/>🔒 restricted PSS<br/>🔑 be-sa"]
            DB["Database<br/>🔒 restricted PSS<br/>🔑 db-sa"]

            FE -->|"mTLS ✅<br/>NetworkPolicy ✅"| BE
            BE -->|"mTLS ✅<br/>NetworkPolicy ✅"| DB
            FE -.->|"❌ blocked"| DB
        end
    end

    Vault["HashiCorp Vault<br/>(External Secrets)"]

    Internet --> WAF --> Ingress --> FE
    Vault -->|"auto-sync"| NS

    style Cluster fill:#E8F5E9
```

---

## Security Checklist

| Layer | Action | K8s Tool |
|-------|--------|----------|
| **Network** | Default deny + whitelist | NetworkPolicy |
| **Encryption** | Encrypt all pod-to-pod traffic | Service Mesh (mTLS) |
| **Identity** | Dedicated ServiceAccount per workload | RBAC + ServiceAccount |
| **Authorization** | Least-privilege roles | Role + RoleBinding |
| **Pod hardening** | Non-root, read-only FS, drop caps | Pod Security Standards |
| **Secrets** | External manager, mounted as files | External Secrets Operator |
| **Audit** | Log all API access | K8s audit logging |
| **Scanning** | Scan images for CVEs | Trivy / Docker Scout in CI |

---

> **Next**: [7_System_Design_Example.md](./7_System_Design_Example.md) — Full e-commerce platform design on K8s
