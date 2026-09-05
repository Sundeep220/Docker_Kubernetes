# System Design Interview Scenarios → K8s Design

> **Prerequisites**: All previous System Design notes  
> **Connects to**: [ROADMAP.md](../../ROADMAP.md) — Interview Topics Checklist

This doc covers common system design interview questions and shows how to map your design to Kubernetes primitives. For each scenario, we provide the **architecture, K8s resources, and key design decisions**.

---

## How to Approach K8s System Design Questions

```mermaid
graph TB
    Step1["1. Clarify Requirements<br/>Users, traffic, SLA, constraints"]
    Step2["2. Draw High-Level Architecture<br/>Services, data stores, queues"]
    Step3["3. Map to K8s Resources<br/>Deployment, StatefulSet, Ingress, etc."]
    Step4["4. Add Cross-Cutting Concerns<br/>Scaling, security, observability"]
    Step5["5. Handle Edge Cases<br/>Failures, traffic spikes, data consistency"]

    Step1 --> Step2 --> Step3 --> Step4 --> Step5

    style Step1 fill:#E3F2FD
    style Step2 fill:#FFF3E0
    style Step3 fill:#E8F5E9
    style Step4 fill:#F3E5F5
    style Step5 fill:#FCE4EC
```

---

# Scenario 1: URL Shortener

**"Design a URL shortener like bit.ly"**

```mermaid
graph LR
    User["User"]
    Ingress["Ingress<br/>(NGINX)"]
    API["URL Service<br/>Deployment: 3<br/>HPA: 3-10"]
    Redis["Redis Cache<br/>Deployment: 1"]
    PG["PostgreSQL<br/>StatefulSet: 2<br/>PVC: 20Gi"]

    User --> Ingress --> API
    API --> Redis
    API --> PG
```

| Component | K8s Resource | Details |
|-----------|-------------|---------|
| URL Service | Deployment + HPA | Scale on CPU (70%), 3-10 replicas |
| PostgreSQL | StatefulSet + PVC | Primary + read replica, 20Gi SSD |
| Redis | Deployment | Cache for hot URLs (read-heavy) |
| Ingress | Ingress (NGINX) | TLS, rate limiting (prevent abuse) |
| Config | ConfigMap | CACHE_TTL, DB_HOST |
| Secrets | Secret | DB_PASSWORD, REDIS_PASSWORD |

**Key decisions**:
- Read-heavy → Redis cache in front of DB
- Short URLs are immutable → high cache hit rate
- Rate limit at Ingress to prevent spam

---

# Scenario 2: Notification System

**"Design a system that sends emails, SMS, and push notifications"**

```mermaid
graph LR
    Producer["API Service<br/>Deployment"]
    Kafka["Kafka<br/>StatefulSet: 3"]
    Email["Email Worker<br/>Deployment: 2<br/>KEDA scaled"]
    SMS["SMS Worker<br/>Deployment: 2<br/>KEDA scaled"]
    Push["Push Worker<br/>Deployment: 2<br/>KEDA scaled"]
    DLQ["DLQ Consumer<br/>Deployment: 1"]
    Digest["Digest CronJob<br/>Daily at 6 AM"]

    Producer -->|"notification.send"| Kafka
    Kafka -->|"email topic"| Email
    Kafka -->|"sms topic"| SMS
    Kafka -->|"push topic"| Push
    Kafka -->|"dead letter"| DLQ
    Digest -->|"batch digest"| Email
```

| Component | K8s Resource | Details |
|-----------|-------------|---------|
| Kafka | StatefulSet (3) + PVC (50Gi) | Event bus, 3 brokers |
| Workers | Deployment + KEDA | Scale on Kafka lag, not CPU |
| Dead Letter Consumer | Deployment (1) | Process failed messages |
| Daily Digest | CronJob | `0 6 * * *` — sends daily summary |

**Key decisions**:
- Async (pub/sub) → decouple sender from delivery
- KEDA auto-scales consumers based on queue lag
- Dead letter queue for failed messages (retry later)
- CronJob for scheduled batch notifications

---

# Scenario 3: File Upload Service

**"Design a service for uploading and processing files (images, documents)"**

```mermaid
graph LR
    User["User"]
    Ingress["Ingress<br/>body-size: 50m"]
    Upload["Upload Service<br/>Deployment: 3"]
    TempPVC["Temp PVC<br/>10Gi"]
    Queue["RabbitMQ<br/>StatefulSet: 1"]
    Processor["Processor<br/>Job (per file)"]
    S3["S3 / Azure Blob<br/>(permanent storage)"]

    User -->|"upload"| Ingress --> Upload
    Upload --> TempPVC
    Upload -->|"enqueue"| Queue
    Queue -->|"process"| Processor
    Processor --> S3
```

| Component | K8s Resource | Details |
|-----------|-------------|---------|
| Upload Service | Deployment (3) | Receives files, stores temporarily |
| Temp Storage | PVC (ReadWriteMany) | NFS/Azure Files for shared access |
| RabbitMQ | StatefulSet (1) + PVC | Job queue |
| File Processor | Job | One Job per file (resize, scan, convert) |
| Permanent Storage | ExternalName Service → S3 | Not in K8s |
| Ingress | Ingress + annotation | `proxy-body-size: 50m` |

**Key decisions**:
- Async processing → user doesn't wait for resize/scan
- Job per file → parallel processing, auto-retry on failure
- RWX PVC for temp storage (shared between upload + processor pods)

---

# Scenario 4: Chat Application

**"Design a real-time chat application"**

```mermaid
graph LR
    User["User<br/>(WebSocket)"]
    Ingress["Ingress<br/>(sticky sessions)"]
    Chat["Chat Service<br/>Deployment: 5<br/>HPA: 5-20"]
    Redis["Redis Pub/Sub<br/>Deployment: 3"]
    MsgDB["Message DB<br/>StatefulSet: 2"]

    User -->|"WebSocket"| Ingress --> Chat
    Chat <-->|"cross-pod messaging"| Redis
    Chat --> MsgDB
```

| Component | K8s Resource | Details |
|-----------|-------------|---------|
| Chat Service | Deployment + HPA | WebSocket connections, scale on connections |
| Redis | Deployment (3, Sentinel) | Pub/Sub for cross-pod message delivery |
| Message DB | StatefulSet + PVC | Persist messages (Postgres/MongoDB) |
| Ingress | Ingress + annotations | `affinity: cookie` (sticky sessions for WebSocket) |

**Key decisions**:
- WebSocket → sticky sessions at Ingress
- Redis pub/sub → when user A is on pod-1 and user B is on pod-3, Redis broadcasts
- HPA scales on custom metric (active WebSocket connections)

---

# Scenario 5: CI/CD Pipeline

**"Design a CI/CD system that builds, tests, and deploys code"**

```mermaid
graph LR
    Git["Git Push"]
    Controller["Pipeline Controller<br/>Deployment: 1"]
    Build["Build Job<br/>(Kaniko)"]
    Test["Test Job"]
    Deploy["Deploy Job<br/>(Helm)"]
    Cache["Build Cache<br/>PVC"]
    Registry["Container Registry"]

    Git -->|"webhook"| Controller
    Controller --> Build
    Build --> Cache
    Build --> Registry
    Controller --> Test
    Controller --> Deploy
```

| Component | K8s Resource | Details |
|-----------|-------------|---------|
| Pipeline Controller | Deployment (1) | Tekton / Argo Workflows |
| Build | Job (Kaniko) | Build container image without Docker daemon |
| Test | Job | Run test suite |
| Deploy | Job (Helm) | `helm upgrade --install --atomic` |
| Build Cache | PVC (ReadWriteMany) | Cache layers across builds |
| ServiceAccount | RBAC-scoped | Only deploy to specific namespaces |
| NetworkPolicy | Isolate build pods | Build pods can't access production |

**Key decisions**:
- Jobs for each stage (auto-retry, parallelism)
- Kaniko for in-cluster builds (no Docker-in-Docker security risk)
- RBAC: CI/CD ServiceAccount can only create/update Deployments, not delete
- NetworkPolicy: build namespace isolated from production

---

# Scenario 6: Handle 10x Traffic Spike

**"Your e-commerce platform is getting 10x normal traffic during a flash sale. How do you handle it?"**

```mermaid
graph TB
    Spike["🔥 10x Traffic"]
    
    subgraph Immediate["Immediate (seconds)"]
        I1["HPA scales pods<br/>3 → 15-20 replicas"]
        I2["Rate limiting at Ingress<br/>protect backends"]
        I3["Redis cache absorbs<br/>read traffic"]
    end

    subgraph Medium["Medium (minutes)"]
        M1["Cluster Autoscaler<br/>adds nodes: 5 → 15"]
        M2["KEDA scales Kafka consumers<br/>based on lag"]
    end

    subgraph Design["Design-Level"]
        D1["Async for non-critical<br/>(notifications, analytics)"]
        D2["Circuit breaker on<br/>slow downstream services"]
        D3["PDB ensures minimum<br/>pods during node churn"]
    end

    Spike --> Immediate
    Spike --> Medium
    Spike --> Design

    style Immediate fill:#FFCDD2
    style Medium fill:#FFF3E0
    style Design fill:#E8F5E9
```

### What You'd Say in an Interview

1. **HPA** auto-scales pods based on CPU (responds in ~15-30s)
2. **Cluster Autoscaler** adds nodes when pods are Pending (~2-5 min)
3. **Rate limiting** at Ingress to protect from abuse
4. **Redis cache** to reduce DB load (most reads are cache hits)
5. **Kafka** buffers async work (emails, analytics) — prevents backpressure
6. **Circuit breaker** on downstream services (prevent cascade if payment API is slow)
7. **PDB** ensures minimum pods during the node scaling churn
8. **topologySpreadConstraints** spreads pods across AZs for resilience

---

## Interview Quick Reference

| Question | Key Points to Cover |
|----------|-------------------|
| "Design X on K8s" | Workload types + communication + scaling + security |
| "How to handle failures?" | Probes + circuit breaker + retry + PDB + multi-AZ |
| "How to deploy safely?" | Rolling update / canary + readiness probe + rollback |
| "How to secure it?" | NetworkPolicy + RBAC + mTLS + Pod Security + secrets |
| "How to monitor it?" | Prometheus (metrics) + Fluent Bit (logs) + Jaeger (traces) |
| "How to scale it?" | HPA (pods) + KEDA (queue) + CA (nodes) + cache + async |
| "Sync vs async?" | Sync for user-facing, async for background + fan-out |
| "Database design?" | StatefulSet + PVC for in-cluster, ExternalName for managed |

---

## Summary

The key to system design on K8s:

1. **Every box on the whiteboard** → K8s workload (Deployment, StatefulSet, Job, CronJob, DaemonSet)
2. **Every arrow** → Communication (ClusterIP sync, Kafka async)
3. **Every concern** → K8s primitive (HPA scaling, NetworkPolicy security, Prometheus observability)
4. **Always mention** → How it handles failure, how it scales, how it's secured

---

> **System Design section complete!**  
> Full learning path: Docker → Kubernetes → Helm → System Design  
> Refer to [ROADMAP.md](../../ROADMAP.md) for the complete schedule and interview checklist.
