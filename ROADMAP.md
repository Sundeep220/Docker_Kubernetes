# Docker, Kubernetes & Helm — SDE-2 Level Roadmap

> **Goal**: Go from "I can use the commands" to "I understand *why* things work this way and can design, debug, and operate production systems confidently."
>
> **Your current state** (based on your notes):
> - Docker: Basics through networking, storage, compose, swarm ✅
> - Kubernetes: Intro through services (Pods, ReplicaSets, Deployments, Networking, Services) ✅
> - Helm: Intro through lifecycle (chart structure, templating, values) ✅
>
> **What's missing for SDE-2**: Deep internals, production patterns, debugging instincts, and the ability to *visualize the full request flow* end-to-end.

---

## How to Use This Roadmap

- Each section has a **Mental Model** block — read these first, they fix the "can't visualize it" problem
- **Hands-on** tasks are marked with 🔨 — do these, don't just read
- Estimated time: **8–10 weeks** at ~1 hour/day
- Priority: Items marked ⭐ are SDE-2 interview essentials

---

# Part 1: Docker — Intermediate Mastery (Week 1–2)

You already know how to run containers and write Dockerfiles. Now you need to understand *what's actually happening under the hood* and how to build production-grade images.

---

## 1.1 ⭐ Container Internals — What a Container Actually Is

### Mental Model
```
A container is NOT a lightweight VM. It's a regular Linux process with three isolation mechanisms:

┌─────────────────────────────────────────────────┐
│                  HOST KERNEL                     │
│                                                  │
│  ┌──────────────┐    ┌──────────────┐            │
│  │ Container A   │    │ Container B   │           │
│  │              │    │              │            │
│  │  Namespace   │    │  Namespace   │  ← Isolation│
│  │  (PID, Net,  │    │  (PID, Net,  │    (what    │
│  │   Mnt, UTS,  │    │   Mnt, UTS,  │    you can  │
│  │   IPC, User) │    │   IPC, User) │    see)     │
│  │              │    │              │            │
│  │  Cgroups     │    │  Cgroups     │  ← Limits   │
│  │  (CPU, Mem,  │    │  (CPU, Mem,  │    (what    │
│  │   IO, PIDs)  │    │   IO, PIDs)  │    you can  │
│  │              │    │              │    use)     │
│  │  UnionFS     │    │  UnionFS     │  ← Filesystem│
│  │  (Overlay2)  │    │  (Overlay2)  │    (what    │
│  │              │    │              │    you see  │
│  └──────────────┘    └──────────────┘    on disk) │
└─────────────────────────────────────────────────┘
```

**Think of it as**: Namespaces = walls (isolation), Cgroups = quotas (limits), UnionFS = layered filesystem.

### What to learn
- [ ] **Linux Namespaces**: PID (process isolation), NET (network stack), MNT (filesystem mounts), UTS (hostname), IPC (inter-process communication), USER (UID mapping)
- [ ] **Cgroups v2**: How CPU/memory limits actually work, what happens when a container exceeds limits (OOM killer)
- [ ] **UnionFS / OverlayFS**: How image layers stack, what the "thin writable layer" is, copy-on-write mechanics

### 🔨 Hands-on
```bash
# See namespaces of a running container
docker inspect --format '{{.State.Pid}}' <container>
ls -la /proc/<PID>/ns/

# See cgroup limits
cat /sys/fs/cgroup/docker/<container-id>/memory.max

# See overlay filesystem layers
docker inspect <image> --format '{{json .GraphDriver.Data}}' | jq
```

---

## 1.2 ⭐ Image Layer System — Deep Understanding

### Mental Model
```
Dockerfile Instruction    →    Image Layer (read-only, cached, shared)
─────────────────────────────────────────────────
FROM node:18-alpine       →    Layer 1: Base OS + Node  (200MB, shared across images)
COPY package.json .       →    Layer 2: package.json    (2KB)
RUN npm install           →    Layer 3: node_modules    (150MB, cached if package.json unchanged)
COPY . .                  →    Layer 4: App source      (5MB, changes every build)
CMD ["node", "index.js"]  →    Metadata only, no layer

Container runs:           →    Layer 5: Thin writable layer (copy-on-write)

KEY INSIGHT: Layers are cached top-to-bottom. The moment one layer changes,
ALL layers below it are rebuilt. This is why you COPY package.json before COPY . .
```

### What to learn
- [ ] **Layer caching strategy**: Order instructions from least-changing to most-changing
- [ ] **Multi-stage builds**: Why and how (separate build env from runtime env)
- [ ] **Image size optimization**: Alpine vs Distroless vs Scratch, why smaller = faster + more secure
- [ ] **.dockerignore**: Prevent leaking secrets and bloating context

### 🔨 Hands-on
```bash
# Analyze image layers and sizes
docker history <image> --no-trunc
dive <image>  # Install 'dive' tool — best way to visualize layers

# Multi-stage build example: build a Go binary
# Stage 1: Build (has compiler, SDK — large)
# Stage 2: Runtime (just the binary — tiny)
```

### Build a multi-stage Dockerfile
```dockerfile
# --- Stage 1: Build ---
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .

# --- Stage 2: Runtime ---
FROM alpine:3.18
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
# Final image: ~15MB instead of ~1GB
```

---

## 1.3 ⭐ Docker Networking — How Containers Talk

### Mental Model
```
HOST MACHINE
┌─────────────────────────────────────────────────────┐
│                                                      │
│  docker0 (bridge) ──── 172.17.0.1                    │
│       │                                              │
│       ├── veth123 ─── Container A (172.17.0.2)       │
│       ├── veth456 ─── Container B (172.17.0.3)       │
│       └── veth789 ─── Container C (172.17.0.4)       │
│                                                      │
│  my-network (user-defined bridge) ── 172.18.0.1      │
│       │                                              │
│       ├── veth_abc ── App (172.18.0.2)  ←──┐         │
│       └── veth_def ── DB  (172.18.0.3)  ←──┘ DNS!    │
│                                                      │
│  Port mapping: -p 8080:3000                          │
│  Host:8080 ──iptables NAT──→ Container:3000          │
│                                                      │
└─────────────────────────────────────────────────────┘

KEY INSIGHT: User-defined bridges give you DNS resolution by container name.
Default bridge does NOT. This is why docker-compose services can reach
each other by service name.
```

### What to learn
- [ ] **Bridge vs Host vs None**: When to use each
- [ ] **User-defined bridges**: DNS resolution, isolation between networks
- [ ] **Port mapping internals**: iptables NAT rules
- [ ] **Container-to-container communication**: Same network vs cross-network

### 🔨 Hands-on
```bash
# Inspect bridge network
docker network inspect bridge

# Create custom network and test DNS
docker network create mynet
docker run -d --name db --network mynet postgres
docker run --rm --network mynet alpine ping db  # Works! DNS resolution!

# See iptables rules created by Docker
sudo iptables -t nat -L -n
```

---

## 1.4 Docker Storage — Volumes vs Bind Mounts (Properly)

### Mental Model
```
┌────────────────────────────────────────────────────┐
│                   Container                         │
│                                                     │
│  /app/data  ←──── Named Volume (docker volume)      │
│                   Managed by Docker                 │
│                   Survives container removal         │
│                   Location: /var/lib/docker/volumes/ │
│                                                     │
│  /app/src   ←──── Bind Mount                        │
│                   YOUR host directory               │
│                   Great for development              │
│                   You control the path               │
│                                                     │
│  /tmp       ←──── tmpfs Mount                       │
│                   In-memory only                     │
│                   Fast, disappears on stop           │
│                   Good for secrets/temp files        │
│                                                     │
└────────────────────────────────────────────────────┘

RULE OF THUMB:
  - Production data (DB files): Named Volumes
  - Development (live reload): Bind Mounts
  - Sensitive temp data: tmpfs
```

---

## 1.5 Docker Compose — Production Patterns

### What to learn
- [ ] **Healthchecks in Compose**: Real `depends_on` with `condition: service_healthy`
- [ ] **Resource limits in Compose**: `deploy.resources.limits`
- [ ] **Profiles**: Run subsets of services (`docker compose --profile debug up`)
- [ ] **Compose Watch** (v2.22+): Auto-rebuild/sync for dev
- [ ] **Secrets in Compose**: Don't use env vars for passwords in production

### 🔨 Hands-on: Write a production-grade docker-compose.yml
```yaml
services:
  api:
    build: .
    ports: ["8080:3000"]
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  pgdata:
```

---

## 1.6 Docker Security Essentials

- [ ] **Run as non-root**: `USER 1001` in Dockerfile
- [ ] **Read-only filesystem**: `--read-only` flag
- [ ] **Drop capabilities**: `--cap-drop ALL --cap-add NET_BIND_SERVICE`
- [ ] **Scan images**: `docker scout cves <image>` or `trivy image <image>`
- [ ] **No secrets in images**: Use build args + multi-stage, or Docker secrets

---

# Part 2: Kubernetes — Intermediate Mastery (Week 3–6)

You know Pods, Deployments, Services. Now you need to understand the *control loop*, networking model, storage, and production patterns that SDE-2s are expected to know.

---

## 2.1 ⭐ Kubernetes Architecture — The Control Loop Mental Model

### Mental Model
```
Everything in Kubernetes is a CONTROL LOOP:

     ┌──────────────────────────────────────────┐
     │            DESIRED STATE                  │
     │   (what you wrote in YAML / API call)     │
     └──────────┬───────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │          etcd (brain/database)            │
     │   Stores ALL cluster state as key-value   │
     │   Only kube-apiserver talks to etcd       │
     └──────────┬───────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │     Controllers (watch + reconcile)       │
     │                                           │
     │   "Is current state == desired state?"    │
     │   No → Take action to fix it              │
     │   Yes → Do nothing, keep watching         │
     │                                           │
     │   Examples:                               │
     │   - Deployment Controller                 │
     │   - ReplicaSet Controller                 │
     │   - Node Controller                       │
     │   - Service Controller                    │
     │   - Endpoint Controller                   │
     └──────────┬───────────────────────────────┘
                │
                ▼
     ┌──────────────────────────────────────────┐
     │         CURRENT STATE                     │
     │   (what's actually running on nodes)      │
     └──────────────────────────────────────────┘

THIS IS THE SINGLE MOST IMPORTANT CONCEPT IN K8S.
Every feature is just another controller watching and reconciling.
```

### What happens when you run `kubectl apply -f deployment.yaml`
```
You (kubectl) ──→ kube-apiserver ──→ etcd (store desired state)
                       │
                       ▼
              Deployment Controller sees new Deployment
                       │
                       ▼
              Creates ReplicaSet object in etcd
                       │
                       ▼
              ReplicaSet Controller sees new ReplicaSet
                       │
                       ▼
              Creates Pod objects in etcd (status: Pending)
                       │
                       ▼
              kube-scheduler sees Pending pods
                       │
                       ▼
              Assigns pods to nodes (writes node binding to etcd)
                       │
                       ▼
              kubelet on assigned node sees pod assignment
                       │
                       ▼
              kubelet tells container runtime (containerd) to pull image & start container
                       │
                       ▼
              kubelet reports pod status back → etcd (status: Running)
```

### 🔨 Hands-on
```bash
# Watch the cascade in real-time (open 3 terminals):
# Terminal 1: Watch deployments
kubectl get deployments -w

# Terminal 2: Watch replicasets
kubectl get rs -w

# Terminal 3: Watch pods
kubectl get pods -w

# Now apply a deployment and see the cascade happen
kubectl apply -f deployment.yaml
```

---

## 2.2 ⭐ Kubernetes Networking — The Full Picture

### Mental Model: How a Request Reaches Your Pod
```
EXTERNAL REQUEST FLOW:
═══════════════════════════════════════════════════════════════

Internet User
    │
    ▼
┌──────────────────┐
│  Load Balancer    │  (Cloud LB: AWS ALB/NLB, GCP LB)
│  External IP      │
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  Ingress          │  (Layer 7 routing: path/host based)
│  Controller       │  e.g., nginx-ingress, traefik
│                   │  Rules: /api → api-svc, /web → web-svc
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  Service          │  (Stable virtual IP + DNS name)
│  (ClusterIP)      │  e.g., api-svc.default.svc.cluster.local
│                   │  iptables/IPVS rules on every node
└──────┬───────────┘
       │
       ▼ (round-robin to one of the endpoints)
┌──────────────────┐
│  Pod (Endpoint)   │  (Actual container running your code)
│  10.244.1.15:3000 │  Has its own IP (from CNI plugin)
└──────────────────┘


SERVICE TYPES — WHEN TO USE WHAT:
─────────────────────────────────
ClusterIP     → Internal only (default). Microservice-to-microservice.
NodePort      → Exposes on every node's IP:Port. Dev/testing only.
LoadBalancer  → Creates cloud LB. One per service (expensive!).
ExternalName  → DNS alias to external service (e.g., AWS RDS endpoint).
Ingress       → NOT a service type. L7 reverse proxy. Use this in production.
```

### Mental Model: Pod-to-Pod Networking
```
NODE 1 (10.0.1.10)                    NODE 2 (10.0.1.11)
┌────────────────────┐                ┌────────────────────┐
│  Pod A             │                │  Pod C             │
│  10.244.1.2        │                │  10.244.2.3        │
│       │            │                │       ▲            │
│  Pod B             │                │       │            │
│  10.244.1.3        │                │       │            │
│       │            │                │       │            │
│  ┌────┴────────┐   │                │  ┌────┴────────┐   │
│  │ cbr0 bridge │   │                │  │ cbr0 bridge │   │
│  └────┬────────┘   │                │  └────┬────────┘   │
│       │            │                │       │            │
└───────┼────────────┘                └───────┼────────────┘
        │                                     │
        └────────── Overlay Network ──────────┘
                  (VXLAN / Flannel / Calico / Cilium)

KEY RULES:
1. Every Pod gets its own IP (no NAT between pods)
2. All Pods can reach all other Pods without NAT
3. The IP a Pod sees for itself = the IP others see for it
```

### What to learn
- [ ] **Service discovery via DNS**: `<svc>.<namespace>.svc.cluster.local`
- [ ] **kube-proxy modes**: iptables vs IPVS (and why IPVS is better at scale)
- [ ] **CNI plugins**: What they do (Flannel = simple overlay, Calico = BGP + network policy, Cilium = eBPF)
- [ ] **Ingress deep dive**: Path-based routing, TLS termination, annotations
- [ ] **Network Policies**: Default deny, allow specific traffic (like firewall rules for pods)

### 🔨 Hands-on
```bash
# See how Services map to Endpoints (Pod IPs)
kubectl get endpoints <service-name>

# DNS resolution inside a pod
kubectl run tmp --rm -it --image=busybox -- nslookup my-service.default.svc.cluster.local

# See iptables rules created by kube-proxy
kubectl get svc  # note ClusterIP
sudo iptables -t nat -L -n | grep <ClusterIP>

# Apply a network policy to deny all ingress, then whitelist
```

---

## 2.3 ⭐ Workload Resources — Beyond Deployments

### Mental Model: Which Resource For Which Job?
```
┌─────────────────────────────────────────────────────────────┐
│                    WORKLOAD DECISION TREE                     │
│                                                              │
│  Is it a long-running process?                               │
│  ├── YES: Does it need stable identity/storage?              │
│  │   ├── YES → StatefulSet (databases, Kafka, Elasticsearch) │
│  │   └── NO  → Deployment (APIs, web servers, workers)       │
│  │                                                           │
│  └── NO: Is it a one-time task?                              │
│      ├── YES → Job (migrations, batch processing)            │
│      └── SCHEDULED → CronJob (nightly reports, cleanup)      │
│                                                              │
│  Need it on EVERY node?                                      │
│  └── YES → DaemonSet (log collectors, monitoring agents)     │
└─────────────────────────────────────────────────────────────┘
```

### What to learn
- [ ] **Deployments**: Rolling update strategy (`maxSurge`, `maxUnavailable`), rollback with `kubectl rollout undo`
- [ ] **StatefulSets**: Why ordinal naming matters, headless services, stable network identity
- [ ] **DaemonSets**: Node-level agents, tolerations to run on control-plane nodes
- [ ] **Jobs/CronJobs**: `backoffLimit`, `completions`, `parallelism`, `concurrencyPolicy`

### StatefulSet Mental Model
```
Deployment pods:    pod-abc12, pod-def34, pod-ghi56  (random names, interchangeable)
StatefulSet pods:   mysql-0, mysql-1, mysql-2         (ordered, stable identity)

StatefulSet guarantees:
1. Ordered creation:    mysql-0 → mysql-1 → mysql-2
2. Ordered deletion:    mysql-2 → mysql-1 → mysql-0
3. Stable DNS:          mysql-0.mysql-headless.default.svc.cluster.local
4. Stable storage:      Each pod gets its own PVC that persists across restarts
```

---

## 2.4 ⭐ Configuration & Secrets

### What to learn
- [ ] **ConfigMaps**: As env vars vs as mounted files, when to use each
- [ ] **Secrets**: Base64 ≠ encryption!, enable encryption at rest, use external secret managers
- [ ] **Immutable ConfigMaps/Secrets**: Performance benefit at scale
- [ ] **Config reload patterns**: Rolling restart via annotation change vs sidecar watchers

### Mental Model
```
ConfigMap / Secret mounting:

┌──────────────────────────────────────────┐
│              Pod                          │
│                                           │
│   env:                                    │
│     DB_HOST: {{ configmap.db-host }}      │  ← Injected at start. NO auto-update.
│     DB_PASS: {{ secret.db-pass }}         │
│                                           │
│   volumeMounts:                           │
│     /etc/config/app.yaml  ← configmap     │  ← Mounted as file. Auto-updates
│     /etc/secrets/creds    ← secret        │    (kubelet sync period ~60s)
│                                           │
│   GOTCHA: Env vars from ConfigMap are     │
│   set at pod creation. If you update the  │
│   ConfigMap, you MUST restart the pod     │
│   to pick up env var changes.             │
│   But file-mounted ConfigMaps update      │
│   automatically (with a delay).           │
└──────────────────────────────────────────┘
```

---

## 2.5 ⭐ Storage in Kubernetes

### Mental Model
```
┌────────────────────────────────────────────────────────────┐
│                   STORAGE HIERARCHY                         │
│                                                             │
│   StorageClass                                              │
│   (defines HOW to provision storage)                        │
│   e.g., "gp3", "ssd", "standard"                           │
│       │                                                     │
│       ▼                                                     │
│   PersistentVolume (PV)                                     │
│   (actual storage resource — disk, NFS share, EBS volume)   │
│   Can be pre-provisioned (static) or auto-created (dynamic) │
│       │                                                     │
│       ▼  ← Bound (1:1 relationship)                        │
│   PersistentVolumeClaim (PVC)                               │
│   (a REQUEST for storage by a pod)                          │
│   "I need 10Gi of SSD storage"                              │
│       │                                                     │
│       ▼  ← Mounted                                         │
│   Pod                                                       │
│   volumeMounts:                                             │
│     - mountPath: /data                                      │
│       name: my-storage                                      │
│                                                             │
│   ANALOGY:                                                  │
│   StorageClass = "type of storage available" (menu)         │
│   PV = "actual disk" (the kitchen made the dish)            │
│   PVC = "order for storage" (your order from the menu)      │
│   Pod = "consumer" (you eat the dish)                       │
│                                                             │
│   DYNAMIC PROVISIONING (most common in cloud):              │
│   Pod claims PVC → PVC references StorageClass →            │
│   StorageClass auto-creates PV (e.g., creates EBS volume)   │
└────────────────────────────────────────────────────────────┘
```

### What to learn
- [ ] **Access Modes**: ReadWriteOnce (RWO), ReadOnlyMany (ROX), ReadWriteMany (RWX)
- [ ] **Reclaim Policies**: Retain vs Delete — what happens to data when PVC is deleted
- [ ] **Dynamic Provisioning**: StorageClass + provisioner = automatic PV creation
- [ ] **StatefulSet + VolumeClaimTemplates**: Each pod gets its own PVC automatically

---

## 2.6 ⭐ Scheduling, Scaling & Resource Management

### Mental Model: How the Scheduler Works
```
New Pod (Pending) → Scheduler evaluates:

Step 1: FILTERING (which nodes CAN run this pod?)
  ├── Does node have enough CPU/memory? (resource requests)
  ├── Does node match nodeSelector/nodeAffinity?
  ├── Does node have taints that pod doesn't tolerate?
  └── Does pod have anti-affinity with pods on this node?

Step 2: SCORING (which node SHOULD run this pod?)
  ├── Least requested resources (spread load)
  ├── Node affinity preference weight
  └── Pod topology spread constraints

Step 3: BINDING
  └── Assign pod to highest-scoring node
```

### Resource Requests vs Limits
```
resources:
  requests:           # GUARANTEED minimum. Used for scheduling.
    cpu: "250m"       # 0.25 CPU cores — scheduler reserves this
    memory: "256Mi"   # 256 MiB — scheduler reserves this
  limits:             # MAXIMUM allowed. Enforced at runtime.
    cpu: "500m"       # Throttled if exceeded (NOT killed)
    memory: "512Mi"   # OOM-KILLED if exceeded

KEY INSIGHT:
  - Request too low  → pod gets scheduled but starves (slow)
  - Request too high → cluster can't schedule (wasted capacity)
  - No limits        → one pod can consume entire node
  - Memory limit hit → OOM kill (pod restart)
  - CPU limit hit    → throttling (pod slows down, NOT killed)
```

### What to learn
- [ ] **HPA (Horizontal Pod Autoscaler)**: Scale pods based on CPU/memory/custom metrics
- [ ] **VPA (Vertical Pod Autoscaler)**: Auto-tune resource requests
- [ ] **Cluster Autoscaler**: Add/remove nodes based on pending pods
- [ ] **Taints & Tolerations**: "Node says: only certain pods allowed here"
- [ ] **Node Affinity**: "Pod says: I prefer to run on certain nodes"
- [ ] **Pod Anti-Affinity**: "Don't put two replicas of me on the same node"
- [ ] **Topology Spread Constraints**: Distribute pods evenly across zones
- [ ] **Pod Disruption Budgets (PDB)**: "Keep at least N pods running during node drain"

### 🔨 Hands-on
```bash
# Set up HPA
kubectl autoscale deployment my-app --cpu-percent=50 --min=2 --max=10

# Watch scaling in action
kubectl get hpa -w

# Test with load
kubectl run -it --rm load-generator --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://my-app; done"

# Drain a node (see PDB in action)
kubectl drain <node-name> --ignore-daemonsets
```

---

## 2.7 ⭐ RBAC — Security That You'll Be Asked About

### Mental Model
```
WHO          can do    WHAT         on     WHERE
─────        ───────   ──────       ──     ─────
Subject      Verb      Resource     in     Namespace
(User/       (get,     (pods,       
 Group/       list,     deployments,
 ServiceAcc)  create,   secrets...)
              delete,
              watch)

┌────────────────────────────────────────────┐
│  Role (namespaced) / ClusterRole (global)  │
│  "What permissions exist"                  │
│                                            │
│  rules:                                    │
│  - apiGroups: [""]                         │
│    resources: ["pods"]                     │
│    verbs: ["get", "list", "watch"]         │
└──────────────┬─────────────────────────────┘
               │
    RoleBinding / ClusterRoleBinding
    "Who gets those permissions"
               │
┌──────────────┴─────────────────────────────┐
│  Subject                                    │
│  (User, Group, or ServiceAccount)           │
└─────────────────────────────────────────────┘

COMMON PATTERN:
  - Dev team gets Role: read pods/logs in their namespace
  - CI/CD ServiceAccount gets Role: create/update deployments
  - Admin gets ClusterRole: full access everywhere
```

---

## 2.8 Debugging & Troubleshooting — The SDE-2 Skill

### Debugging Decision Tree
```
Pod not running?
├── Status: Pending
│   ├── Check: kubectl describe pod → Events section
│   ├── Insufficient resources? → Check node capacity, adjust requests
│   ├── No matching node? → Check nodeSelector, affinity, taints
│   └── PVC not bound? → Check StorageClass, PV availability
│
├── Status: CrashLoopBackOff
│   ├── Check: kubectl logs <pod> --previous
│   ├── App crashing? → Fix app code, check config
│   ├── Readiness probe failing? → Check probe config, endpoints
│   └── OOMKilled? → Increase memory limit
│
├── Status: ImagePullBackOff
│   ├── Wrong image name/tag? → Fix image reference
│   ├── Private registry? → Create/fix imagePullSecret
│   └── Image doesn't exist? → Push image first
│
└── Status: Running but not working
    ├── Check: kubectl logs <pod> -f
    ├── Check: kubectl exec -it <pod> -- /bin/sh
    ├── Service not routing? → kubectl get endpoints
    ├── DNS not resolving? → kubectl run tmp --rm -it --image=busybox -- nslookup <svc>
    └── Network policy blocking? → Check network policies
```

### Essential Debug Commands
```bash
# The holy trinity of debugging
kubectl describe pod <pod>       # Events, conditions, node assignment
kubectl logs <pod> -f            # App logs (live)
kubectl logs <pod> --previous    # Logs from crashed container

# Network debugging
kubectl exec -it <pod> -- curl http://other-service:8080/health
kubectl exec -it <pod> -- nslookup other-service

# Resource pressure
kubectl top pods                 # CPU/memory usage per pod
kubectl top nodes                # CPU/memory usage per node

# Cluster-wide view
kubectl get events --sort-by='.lastTimestamp'
kubectl get pods --all-namespaces | grep -v Running
```

---

# Part 3: Helm — Intermediate Mastery (Week 6–7)

You know chart structure and templating. Now learn to build reusable, production-grade charts.

---

## 3.1 ⭐ Helm Architecture — What Actually Happens

### Mental Model
```
┌──────────────────────────────────────────────────────────┐
│                    HELM WORKFLOW                          │
│                                                          │
│   Chart (templates + values)                             │
│       │                                                  │
│       ▼                                                  │
│   helm template / helm install                           │
│       │                                                  │
│       ▼                                                  │
│   Template Engine (Go templates)                         │
│   Merges: values.yaml + overrides + built-in objects     │
│       │                                                  │
│       ▼                                                  │
│   Rendered K8s YAML manifests                            │
│       │                                                  │
│       ▼                                                  │
│   kubectl apply (via kube-apiserver)                     │
│       │                                                  │
│       ▼                                                  │
│   Release (stored as Secret in namespace)                │
│   Contains: chart, values, rendered templates, version   │
│                                                          │
│   KEY INSIGHT: Helm is PRIMARILY a templating engine +   │
│   release manager. It doesn't run anything itself.       │
│   It generates YAML and tracks versions.                 │
└──────────────────────────────────────────────────────────┘
```

### Release Lifecycle
```
helm install   →  Release v1 (deployed)
helm upgrade   →  Release v2 (deployed), v1 (superseded)
helm rollback  →  Release v3 (deployed, same as v1), v2 (superseded)

# See release history
helm history my-release

REVISION  STATUS       DESCRIPTION
1         superseded   Install complete
2         superseded   Upgrade complete
3         deployed     Rollback to 1
```

---

## 3.2 ⭐ Advanced Templating Patterns

### What to learn
- [ ] **Named templates** (`_helpers.tpl`): DRY up your charts
- [ ] **Flow control**: `if/else`, `range`, `with`
- [ ] **Template functions**: `default`, `quote`, `toYaml`, `nindent`, `include`
- [ ] **Lookups**: Query existing cluster resources from templates
- [ ] **Hooks**: Pre-install, post-install, pre-upgrade (DB migrations!)
- [ ] **Subcharts & Dependencies**: `Chart.yaml` dependencies, global values

### Common Patterns
```yaml
# _helpers.tpl — Reusable labels
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

# deployment.yaml — Using the helper
metadata:
  labels:
    {{- include "myapp.labels" . | nindent 4 }}

# Conditional resources
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# Iterating over values
{{- range .Values.extraEnvVars }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}
```

---

## 3.3 Helm in Production

### What to learn
- [ ] **values.yaml organization**: Base values, environment overrides (`values-prod.yaml`)
- [ ] **Helm diff plugin**: Preview changes before upgrade
- [ ] **Chart testing**: `helm lint`, `helm template`, `helm test`
- [ ] **Chart repositories**: Hosting your own charts (Chartmuseum, OCI registries)
- [ ] **Helmfile**: Manage multiple releases declaratively
- [ ] **Hooks for DB migrations**: Run migration Job before app upgrade

### 🔨 Hands-on
```bash
# Debug template rendering (see exactly what YAML will be applied)
helm template my-release ./my-chart -f values-prod.yaml --debug

# Diff before upgrade
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-release ./my-chart -f values-prod.yaml

# Lint your chart
helm lint ./my-chart --strict

# Package and push to OCI registry
helm package ./my-chart
helm push my-chart-1.0.0.tgz oci://my-registry.com/charts
```

---

## 3.4 Building a Production Chart — Structure

```
my-chart/
├── Chart.yaml              # Chart metadata + dependencies
├── Chart.lock              # Locked dependency versions
├── values.yaml             # Default values
├── values-dev.yaml         # Dev overrides
├── values-staging.yaml     # Staging overrides
├── values-prod.yaml        # Prod overrides
├── templates/
│   ├── _helpers.tpl        # Reusable template snippets
│   ├── deployment.yaml     # Main deployment
│   ├── service.yaml        # Service
│   ├── ingress.yaml        # Ingress (conditional)
│   ├── hpa.yaml            # HPA (conditional)
│   ├── configmap.yaml      # ConfigMap
│   ├── secret.yaml         # Secrets
│   ├── serviceaccount.yaml # ServiceAccount
│   ├── pdb.yaml            # Pod Disruption Budget
│   ├── networkpolicy.yaml  # Network Policy
│   └── tests/
│       └── test-connection.yaml  # helm test
└── charts/                 # Subcharts (dependencies)
```

---

# Part 4: System Design & Distributed Systems Networking (Week 8–9)

This is where Docker/K8s knowledge meets system design. In interviews, you're not just asked "what is a Service" — you're asked "design the networking for an e-commerce platform with 20 microservices." This section bridges that gap.

---

## 4.1 ⭐ How System Design Maps to Kubernetes

### Mental Model: The System Design → K8s Translation Table
```
SYSTEM DESIGN CONCEPT          →   KUBERNETES PRIMITIVE
══════════════════════════════════════════════════════════════
Service / Microservice          →   Deployment + Service
Database (stateful)             →   StatefulSet + PVC + Headless Service
Background worker               →   Deployment (no Service) or Job
Scheduled task (cron)           →   CronJob
API Gateway                     →   Ingress Controller / Gateway API
Load Balancer                   →   Service (LoadBalancer) / Ingress
Service Discovery               →   CoreDNS (built-in)
Config Management               →   ConfigMap + Secret
Secret Management               →   External Secrets Operator + Vault
Auto-scaling                    →   HPA / VPA / Cluster Autoscaler
Health Monitoring               →   Liveness + Readiness + Startup Probes
Circuit Breaker / Retry         →   Service Mesh (Istio/Linkerd) or app-level
Rate Limiting                   →   Ingress annotations / Service Mesh
Blue-Green Deployment           →   Two Deployments + Service selector switch
Canary Deployment               →   Argo Rollouts / Istio traffic splitting
Message Queue                   →   StatefulSet (Kafka/RabbitMQ) or managed service
Cache Layer                     →   Deployment (Redis) or managed (ElastiCache)
Log Aggregation                 →   DaemonSet (Fluentd/Fluent Bit) → ELK/Loki
Metrics Collection              →   DaemonSet (node-exporter) + Prometheus
Service Mesh                    →   Istio / Linkerd (sidecar proxies)
```

---

## 4.2 ⭐ Designing Networking for Distributed Systems

### Mental Model: The 4 Layers of Network Traffic
```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  LAYER 1: EXTERNAL → CLUSTER (North-South traffic)                   │
│  ══════════════════════════════════════════════════                   │
│  Internet → DNS → Cloud LB → Ingress Controller → Service → Pod     │
│                                                                      │
│  Design decisions:                                                   │
│  • Single Ingress vs multiple? (usually one per domain)              │
│  • TLS termination at Ingress or at Pod?                             │
│  • Rate limiting at Ingress level (protect backend)                  │
│  • WAF / DDoS protection before Ingress                              │
│                                                                      │
│  LAYER 2: SERVICE → SERVICE (East-West traffic)                      │
│  ══════════════════════════════════════════════════                   │
│  Pod A → ClusterIP Service → Pod B (via kube-proxy / IPVS)          │
│                                                                      │
│  Design decisions:                                                   │
│  • Sync (HTTP/gRPC) vs Async (message queue)?                        │
│  • Direct service call vs through API gateway?                       │
│  • Retry + timeout + circuit breaker (app-level or mesh)?            │
│  • mTLS between services? (service mesh provides this)               │
│                                                                      │
│  LAYER 3: SERVICE → DATA STORE (internal traffic)                    │
│  ════════════════════════════════════════════════════                 │
│  Pod → Headless Service → StatefulSet Pod (DB/Cache/Queue)           │
│                                                                      │
│  Design decisions:                                                   │
│  • In-cluster DB vs managed (RDS, Cloud SQL)?                        │
│  • Connection pooling (sidecar like PgBouncer)?                      │
│  • Read replicas via headless service DNS?                           │
│  • Network policy: only backend pods can reach DB                    │
│                                                                      │
│  LAYER 4: CLUSTER → EXTERNAL (egress traffic)                        │
│  ══════════════════════════════════════════════════                   │
│  Pod → NAT Gateway → External API / Third-party service              │
│                                                                      │
│  Design decisions:                                                   │
│  • Egress network policies (restrict what pods can call out)         │
│  • Egress gateway (Istio) for audit + control                        │
│  • Static egress IP (for IP whitelisting with partners)              │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4.3 ⭐ Sync vs Async Communication — The Core Design Decision

### Mental Model
```
SYNCHRONOUS (request-response)         ASYNCHRONOUS (event-driven)
══════════════════════════════         ══════════════════════════════

  Order-svc ──HTTP──→ Payment-svc       Order-svc ──msg──→ [Queue] ──→ Payment-svc
       │                                     │
       ├── Simple to understand               ├── Decoupled (services independent)
       ├── Tight coupling                     ├── Resilient (queue buffers failures)
       ├── Cascading failures                 ├── Eventual consistency
       ├── Latency adds up                    ├── Harder to debug/trace
       └── Use when: need immediate           └── Use when: can tolerate delay,
           response, simple flows                 high throughput, fan-out

IN KUBERNETES:
  Sync:  Service A → ClusterIP Service B (HTTP/gRPC)
  Async: Service A → Kafka/RabbitMQ (StatefulSet or managed) → Service B

HYBRID PATTERN (most production systems):
  ┌─────────┐  sync   ┌─────────────┐  async   ┌────────────┐
  │ API GW  │────────→│ Order Svc   │────────→│  [Kafka]    │
  │(Ingress)│         │(Deployment) │         │(StatefulSet)│
  └─────────┘         └─────────────┘         └─────┬──────┘
                                                     │
                              ┌───────────────────────┤
                              ▼                       ▼
                      ┌──────────────┐        ┌──────────────┐
                      │ Payment Svc  │        │Notification  │
                      │ (Deployment) │        │   Svc        │
                      └──────────────┘        └──────────────┘
```

### What to learn
- [ ] **When to use sync vs async**: Consistency requirements, latency budget, failure tolerance
- [ ] **Message queue patterns in K8s**: Kafka as StatefulSet, RabbitMQ operator, or managed services
- [ ] **Event-driven autoscaling**: KEDA (scale consumers based on queue depth)
- [ ] **Dead letter queues**: Handling failed async messages

---

## 4.4 ⭐ Service Mesh — When and Why

### Mental Model
```
WITHOUT SERVICE MESH:
  Every service must implement: retries, timeouts, circuit breakers,
  mTLS, tracing, metrics, rate limiting... IN APPLICATION CODE.

WITH SERVICE MESH (e.g., Istio):
  ┌──────────────────────────────────────┐
  │  Pod                                  │
  │  ┌────────────┐  ┌────────────────┐   │
  │  │ Your App   │──│ Sidecar Proxy  │   │  ← Envoy proxy injected
  │  │ (Container)│  │ (Envoy)        │   │    automatically
  │  └────────────┘  └───────┬────────┘   │
  └──────────────────────────┼────────────┘
                             │
                   All traffic goes through
                   the proxy, which handles:
                             │
              ┌──────────────┼──────────────┐
              │              │              │
        ┌─────┴─────┐ ┌─────┴─────┐ ┌──────┴──────┐
        │  mTLS     │ │ Retries & │ │ Observ-     │
        │  (auto    │ │ Circuit   │ │ ability     │
        │  encrypt) │ │ Breakers  │ │ (metrics,   │
        └───────────┘ └───────────┘ │  traces)    │
                                    └─────────────┘

WHEN TO USE SERVICE MESH:
  ✅ >10 microservices with complex communication
  ✅ Need mTLS everywhere (zero-trust security)
  ✅ Need fine-grained traffic control (canary, A/B, fault injection)
  ✅ Need distributed tracing without code changes
  ❌ <5 services (overkill, adds latency + complexity)
  ❌ Simple monolith or small apps
```

---

## 4.5 ⭐ Resilience Patterns for Distributed Systems

### Mental Model: What Fails and How to Handle It
```
┌─────────────────────────────────────────────────────────────────────┐
│            FAILURE MODE           →    K8S / DESIGN SOLUTION        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pod crashes                      →  Liveness probe + restart       │
│                                      policy (Always)                │
│                                                                     │
│  Pod overloaded                   →  HPA (scale out) +              │
│                                      resource limits                │
│                                                                     │
│  Downstream service down          →  Circuit breaker (Istio /       │
│                                      Resilience4j) + fallback       │
│                                                                     │
│  Downstream service slow          →  Timeout + retry with           │
│                                      exponential backoff            │
│                                                                     │
│  Network partition                →  Retry + idempotent APIs        │
│                                      + async with queue buffer      │
│                                                                     │
│  Node failure                     →  Pod rescheduled to another     │
│                                      node (controller reconcile)    │
│                                      + PDB to maintain quorum       │
│                                                                     │
│  AZ / Region failure              →  Multi-AZ pod spread            │
│                                      (topologySpreadConstraints)    │
│                                      + multi-region clusters        │
│                                                                     │
│  Deploy broke something           →  Rolling update (maxSurge)      │
│                                      + readiness probe gate         │
│                                      + helm rollback                │
│                                                                     │
│  Thundering herd / spike          →  Rate limiting (Ingress) +      │
│                                      HPA + queue buffering          │
│                                                                     │
│  Data corruption / bad deploy     →  Blue-green deployment          │
│                                      (instant rollback)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Retry + Timeout + Circuit Breaker — The Holy Trinity
```
Request flow with all three:

  Client ──→ [Timeout: 3s] ──→ [Retry: 3x, backoff] ──→ [Circuit Breaker] ──→ Server
                                                              │
                                                    ┌─────────┴──────────┐
                                                    │ Closed (normal)    │
                                                    │ If >50% fail in    │
                                                    │ last 10 calls:     │
                                                    │     → Open         │
                                                    │                    │
                                                    │ Open (blocking)    │
                                                    │ Fail fast, don't   │
                                                    │ even call server.  │
                                                    │ After 30s → Half   │
                                                    │                    │
                                                    │ Half-Open (testing)│
                                                    │ Let 1 request thru │
                                                    │ Success → Closed   │
                                                    │ Fail → Open again  │
                                                    └────────────────────┘

  WHERE TO IMPLEMENT:
    App-level:    Resilience4j (Java), Polly (.NET), go-circuit
    Infra-level:  Istio DestinationRule (no code changes!)
```

---

## 4.6 ⭐ Designing Network Security (Zero Trust)

### Mental Model
```
TRADITIONAL (perimeter security):    ZERO TRUST (K8s way):

  ┌─────────────────┐                ┌─────────────────────────────┐
  │   Firewall       │                │  Every pod-to-pod call:     │
  │   ┌───────────┐  │                │                             │
  │   │ Trust all │  │                │  1. Network Policy:         │
  │   │ inside    │  │                │     Who can talk to whom?   │
  │   └───────────┘  │                │                             │
  └─────────────────┘                │  2. mTLS (service mesh):    │
  "Castle and moat"                  │     Encrypt all traffic     │
  ONE breach = game over             │     Verify identity         │
                                     │                             │
                                     │  3. RBAC:                   │
                                     │     Who can do what?        │
                                     │                             │
                                     │  4. Pod Security Standards: │
                                     │     What can pods do?       │
                                     └─────────────────────────────┘
                                     "Never trust, always verify"
```

### Network Policy Design Pattern
```yaml
# DEFAULT DENY ALL (start here, then whitelist)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # Apply to ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress

---
# ALLOW: backend → database only
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
```

```
Visual result:
  frontend ──✅──→ backend ──✅──→ database
  frontend ──❌──→ database   (blocked by network policy)
  backend  ──❌──→ frontend   (blocked, no rule for it)
```

---

## 4.7 ⭐ System Design Example: E-Commerce Platform on K8s

### Full Architecture Mental Model
```
                         INTERNET
                            │
                      ┌─────┴─────┐
                      │  AWS ALB   │  ← Cloud Load Balancer
                      └─────┬─────┘
                            │
                   ┌────────┴────────┐
                   │ Ingress (NGINX) │  ← TLS termination, rate limiting
                   │ /api  → api-gw │     path-based routing
                   │ /     → web-ui │
                   └───┬────────┬───┘
                       │        │
              ┌────────┘        └────────┐
              ▼                          ▼
    ┌──────────────────┐      ┌──────────────────┐
    │ Web UI (React)   │      │ API Gateway      │  ← Auth, rate limit,
    │ Deployment: 3    │      │ Deployment: 3    │     request routing
    │ HPA: 3-10        │      │ HPA: 3-20        │
    └──────────────────┘      └────────┬─────────┘
                                       │
              ┌────────────┬───────────┼───────────┐
              ▼            ▼           ▼           ▼
    ┌──────────────┐ ┌───────────┐ ┌─────────┐ ┌──────────┐
    │ Order Svc    │ │ Product   │ │ User    │ │ Payment  │
    │ Dep: 3       │ │ Svc       │ │ Svc     │ │ Svc      │
    │ HPA: 3-15    │ │ Dep: 2    │ │ Dep: 2  │ │ Dep: 3   │
    └──────┬───────┘ └─────┬─────┘ └────┬────┘ └──────────┘
           │               │            │
    ┌──────┴───────┐       │       ┌────┴────────┐
    ▼              ▼       ▼       ▼             │
┌────────┐  ┌──────────┐ ┌──────────┐ ┌─────────┴──┐
│[Kafka] │  │ Order DB │ │Product DB│ │  User DB   │
│Stateful│  │ Postgres │ │ Postgres │ │ Postgres   │
│Set: 3  │  │ SS: 2    │ │ SS: 2    │ │ SS: 2      │
│        │  │ (primary+│ │          │ │            │
│        │  │  replica)│ │          │ │            │
└───┬────┘  └──────────┘ └──────────┘ └────────────┘
    │
    ├──→ Notification Svc (email/SMS) ── Deployment: 2
    ├──→ Inventory Svc (stock update) ── Deployment: 2
    └──→ Analytics Svc (tracking)     ── Deployment: 2


CROSS-CUTTING CONCERNS (system-wide):
┌────────────────────────────────────────────────────────────────┐
│ Observability:                                                 │
│   Prometheus (monitoring) ── DaemonSet: node-exporter          │
│   Grafana (dashboards)    ── Deployment: 1                     │
│   Fluent Bit (logs)       ── DaemonSet: every node             │
│   Jaeger (tracing)        ── Deployment: 1                     │
│                                                                │
│ Security:                                                      │
│   Network Policies (default deny + whitelist)                  │
│   RBAC per team namespace                                      │
│   External Secrets Operator → AWS Secrets Manager              │
│   Istio mTLS (if service mesh is adopted)                      │
│                                                                │
│ Reliability:                                                   │
│   PDB on all critical services (minAvailable: 1)               │
│   Pod anti-affinity (spread across AZs)                        │
│   topologySpreadConstraints: maxSkew: 1                        │
│   Resource requests/limits on everything                       │
└────────────────────────────────────────────────────────────────┘
```

---

## 4.8 ⭐ Deployment Strategies — How to Ship Without Breaking Things

### Mental Model
```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT STRATEGIES                             │
│                                                                     │
│  1. ROLLING UPDATE (K8s default)                                    │
│     Old: ████████  →  Old: ████░░░░  →  Old: ░░░░░░░░              │
│     New: ░░░░░░░░  →  New: ░░░░████  →  New: ████████              │
│     ✅ Zero downtime  ❌ Two versions run simultaneously            │
│     Config: maxSurge=25%, maxUnavailable=25%                        │
│                                                                     │
│  2. BLUE-GREEN                                                      │
│     Blue (live):  ████████ ──── Service selector: v1                │
│     Green (new):  ████████ ──── (idle, tested)                      │
│     Switch:       Service selector: v1 → v2  (instant)             │
│     Rollback:     Service selector: v2 → v1  (instant)             │
│     ✅ Instant rollback  ❌ 2x resources during deploy              │
│                                                                     │
│  3. CANARY                                                          │
│     Stable:  ████████████████ (90% traffic)                         │
│     Canary:  ██ (10% traffic)                                       │
│     Monitor metrics → gradually increase → 100%                     │
│     ✅ Low risk  ❌ Needs traffic splitting (Istio/Argo)            │
│                                                                     │
│  4. A/B TESTING                                                     │
│     Route by header/cookie/user-segment                             │
│     Premium users → v2, others → v1                                 │
│     ✅ Targeted testing  ❌ Complex routing rules                   │
│                                                                     │
│  DECISION GUIDE:                                                    │
│  Simple app, low risk       → Rolling Update                        │
│  Critical service, need instant rollback → Blue-Green               │
│  High-traffic, gradual validation → Canary                          │
│  Feature experimentation    → A/B Testing                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4.9 Designing for Observability

### The Three Pillars
```
┌──────────────────────────────────────────────────────────────────┐
│                  OBSERVABILITY IN K8S                             │
│                                                                  │
│  METRICS (what's happening now?)                                 │
│  ────────────────────────────────                                │
│  Prometheus ←─scrape─ Pod metrics (via /metrics endpoint)        │
│       │                                                          │
│       └──→ Grafana dashboards                                    │
│       └──→ Alertmanager (PagerDuty, Slack)                       │
│                                                                  │
│  Key metrics to monitor:                                         │
│  • Request rate (RPS)                                            │
│  • Error rate (5xx / total)          ← RED method                │
│  • Duration (p50, p95, p99 latency)                              │
│  • CPU / Memory utilization                                      │
│  • Pod restart count                                             │
│                                                                  │
│  LOGS (what happened?)                                           │
│  ─────────────────────                                           │
│  DaemonSet (Fluent Bit) ←─ stdout/stderr from all pods           │
│       │                                                          │
│       └──→ Elasticsearch / Loki → Kibana / Grafana               │
│                                                                  │
│  Best practice: structured JSON logs with correlation IDs        │
│                                                                  │
│  TRACES (why is it slow?)                                        │
│  ────────────────────────                                        │
│  OpenTelemetry SDK in app → Jaeger / Zipkin / Tempo              │
│                                                                  │
│  Shows the full journey of a request across services:            │
│  API-GW (2ms) → Order-Svc (15ms) → DB Query (45ms) ← SLOW!     │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4.10 Common System Design Interview Scenarios → K8s Design

| Scenario | Key K8s Design Decisions |
|----------|--------------------------|
| "Design a URL shortener" | Deployment + HPA, Redis (Deployment) for cache, Postgres (StatefulSet) for persistence, Ingress for routing |
| "Design a notification system" | Kafka (StatefulSet) for event bus, Consumer Deployments with KEDA autoscaling, CronJob for scheduled digests |
| "Design a file upload service" | Deployment with PVC for temp storage, S3 for permanent, Job for processing, Ingress with body-size annotation |
| "Design a chat application" | Deployment with WebSocket (sticky sessions via Ingress annotation), Redis (pub/sub) for cross-pod messaging |
| "Design a CI/CD pipeline" | Job for each build step, PVC for build cache, ServiceAccount with scoped RBAC, Network Policy for isolation |
| "Handle a traffic spike (10x)" | HPA + Cluster Autoscaler, rate limiting at Ingress, queue buffering with Kafka, PDB to survive node churn |

---

# Part 5: Putting It All Together (Week 9–10)

## 5.1 End-to-End Project

Build and deploy a microservices app that covers all the concepts:

### 🔨 Project: Deploy a 3-tier app to Kubernetes using Helm
```
┌─────────┐    ┌──────────┐    ┌──────────┐
│ Frontend │───→│ Backend  │───→│ Database │
│ (React)  │    │ (Node.js)│    │(Postgres)│
└─────────┘    └──────────┘    └──────────┘

Requirements:
├── Docker
│   ├── Multi-stage Dockerfiles for frontend & backend
│   ├── .dockerignore files
│   ├── Non-root users
│   └── Health check endpoints
│
├── Kubernetes
│   ├── Deployment for frontend & backend
│   ├── StatefulSet for database
│   ├── Services (ClusterIP for internal, Ingress for external)
│   ├── ConfigMaps for app config
│   ├── Secrets for DB credentials
│   ├── PVC for database storage
│   ├── HPA for backend
│   ├── Network Policies (frontend→backend, backend→db only)
│   ├── Resource requests & limits
│   ├── Liveness & readiness probes
│   └── PDB for backend
│
└── Helm
    ├── Umbrella chart with subcharts
    ├── Environment-specific values files
    ├── Pre-upgrade hook for DB migrations
    └── helm test for connectivity checks
```

---

## 5.2 ⭐ SDE-2 Interview Topics Checklist

These are the most commonly asked topics in SDE-2 interviews:

### Docker
- [ ] Explain container isolation (namespaces, cgroups, union filesystem)
- [ ] Multi-stage builds: why and how
- [ ] Docker networking: bridge, host, none, overlay
- [ ] Image layer caching: how to optimize Dockerfiles
- [ ] Docker Compose: production patterns (healthchecks, resource limits, secrets)

### Kubernetes
- [ ] Explain the K8s architecture and control loop
- [ ] What happens when you run `kubectl apply -f deployment.yaml`? (full flow)
- [ ] Service types and when to use each
- [ ] How does service discovery work? (DNS, endpoints)
- [ ] Deployment vs StatefulSet vs DaemonSet: when to use each
- [ ] Resource requests vs limits: what happens when exceeded
- [ ] How does HPA work? How to scale on custom metrics?
- [ ] RBAC: explain Role, ClusterRole, RoleBinding
- [ ] How to debug a pod that's not starting?
- [ ] Rolling updates: how do they work? How to rollback?
- [ ] Ingress vs LoadBalancer: when to use what?
- [ ] Network Policies: how to restrict pod-to-pod traffic
- [ ] PV, PVC, StorageClass: explain with an analogy

### Helm
- [ ] What problem does Helm solve?
- [ ] Chart structure and templating basics
- [ ] Release lifecycle (install, upgrade, rollback)
- [ ] How to manage multiple environments with Helm?
- [ ] Hooks: pre-install, post-install use cases

### System Design & Distributed Systems
- [ ] Sync vs Async communication: when to use each and why
- [ ] Design networking for microservices (the 4 layers: north-south, east-west, data, egress)
- [ ] Service Mesh: what problem does it solve, when is it overkill?
- [ ] Retry + Timeout + Circuit Breaker: explain the pattern and where to implement
- [ ] Blue-Green vs Canary vs Rolling update: trade-offs
- [ ] Zero trust networking in K8s: Network Policies + mTLS
- [ ] How would you design [X system] on Kubernetes? (map concepts to K8s primitives)
- [ ] Observability: RED method, structured logging, distributed tracing
- [ ] How to handle a 10x traffic spike with K8s primitives?
- [ ] Database per service vs shared database: how storage design maps to StatefulSets

---

## 5.3 Recommended Learning Resources

### Docker
- **Book**: "Docker Deep Dive" by Nigel Poulton
- **Hands-on**: Play with Docker (labs.play-with-docker.com)
- **Tool**: `dive` — for exploring image layers visually

### Kubernetes
- **Course**: KodeKloud CKA/CKAD courses (best for hands-on)
- **Book**: "Kubernetes in Action" by Marko Lukša
- **Interactive**: killer.sh (CKA/CKAD practice)
- **Visualization**: k9s (terminal UI for K8s — makes clusters visual)
- **Tool**: Lens (desktop K8s IDE — great for visualization)

### Helm
- **Official Docs**: helm.sh/docs (genuinely good)
- **Course**: KodeKloud Helm course
- **Practice**: Convert existing K8s YAML to a Helm chart

### System Design
- **Book**: "Designing Data-Intensive Applications" by Martin Kleppmann (the bible)
- **Book**: "Building Microservices" by Sam Newman (practical patterns)
- **Course**: ByteByteGo (system design visual explanations)
- **Practice**: Design a system → map every component to K8s resources

### Visualization Tools (install these!)
- **k9s**: Terminal-based K8s dashboard (`brew install k9s`)
- **Lens**: Desktop K8s IDE (free, visual cluster management)
- **dive**: Docker image layer explorer (`brew install dive`)
- **kubectx/kubens**: Fast context/namespace switching
- **Excalidraw**: Draw architecture diagrams (excalidraw.com)

---

## Weekly Schedule Summary

| Week | Focus | Key Outcome |
|------|-------|-------------|
| 1 | Docker internals (namespaces, cgroups, layers) | Understand what a container *is* |
| 2 | Docker production (multi-stage, security, compose) | Build production-grade images |
| 3 | K8s architecture & control loop | Visualize the full request flow |
| 4 | K8s networking & services deep dive | Debug network issues confidently |
| 5 | K8s storage, scheduling, scaling | Design stateful + scalable apps |
| 6 | K8s security (RBAC) & debugging | Handle production incidents |
| 7 | Helm advanced (templating, hooks, production) | Build reusable charts |
| 8 | System design: networking, communication patterns | Map design decisions to K8s |
| 9 | System design: resilience, observability, security | Design production-grade systems |
| 10 | End-to-end project + interview prep | Tie everything together |

---

> **Final tip**: The #1 thing that separates basics from intermediate is **understanding the WHY**. Don't just memorize commands — trace what happens at each step. Use `kubectl describe`, `kubectl get events`, and `kubectl logs` religiously. The ASCII diagrams in this roadmap are your mental models — redraw them from memory until they stick.
