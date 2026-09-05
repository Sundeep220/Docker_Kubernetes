# Sync vs Async Communication — The Core Design Decision

> **Prerequisites**: [2_Distributed_Networking_Layers.md](./2_Distributed_Networking_Layers.md)  
> **Next**: [4_Service_Mesh.md](./4_Service_Mesh.md)

In a microservices architecture, how services talk to each other is the **single most impactful design decision**. Get it wrong, and you get cascading failures, tight coupling, and systems that can't scale.

---

## The Two Models

```mermaid
graph LR
    subgraph Sync["Synchronous (Request-Response)"]
        A1["Order Svc"] -->|"HTTP/gRPC<br/>waits for response"| B1["Payment Svc"]
        B1 -->|"response"| A1
    end

    subgraph Async["Asynchronous (Event-Driven)"]
        A2["Order Svc"] -->|"publish event"| Q["Message Queue<br/>(Kafka / RabbitMQ)"]
        Q -->|"consume"| B2["Payment Svc"]
        Q -->|"consume"| C2["Notification Svc"]
        Q -->|"consume"| D2["Analytics Svc"]
    end

    style Sync fill:#E3F2FD
    style Async fill:#FFF3E0
```

---

# 1. Synchronous Communication

Service A calls Service B and **waits** for the response before continuing.

## Protocols

| Protocol | Format | Use Case | K8s Service |
|----------|--------|----------|-------------|
| **HTTP/REST** | JSON | External APIs, simple internal calls | ClusterIP |
| **gRPC** | Protobuf (binary) | Internal high-performance calls | ClusterIP (HTTP/2) |
| **GraphQL** | JSON | Frontend → backend aggregation | ClusterIP + Ingress |

## In Kubernetes

```yaml
# Service A calls Service B
# Just use the DNS name!
# http://payment-svc:80/charge
# Or fully qualified: http://payment-svc.production.svc.cluster.local:80/charge

apiVersion: v1
kind: Service
metadata:
  name: payment-svc
spec:
  selector:
    app: payment
  ports:
    - port: 80
      targetPort: 8080
```

## Pros and Cons

```mermaid
graph TB
    subgraph Pros["✅ Pros"]
        P1["Simple to understand and debug"]
        P2["Immediate response / consistency"]
        P3["Easy error handling"]
        P4["Natural request-response pattern"]
    end

    subgraph Cons["❌ Cons"]
        C1["Tight coupling<br/>(both services must be up)"]
        C2["Cascading failures<br/>(B down → A down → C down)"]
        C3["Latency adds up<br/>(A→B→C = total of all latencies)"]
        C4["Hard to scale independently"]
    end

    style Pros fill:#C8E6C9
    style Cons fill:#FFCDD2
```

## The Cascading Failure Problem

```mermaid
sequenceDiagram
    participant API as API Gateway
    participant Order as Order Svc
    participant Payment as Payment Svc (DOWN!)
    participant Inventory as Inventory Svc

    API->>Order: POST /orders
    Order->>Payment: POST /charge
    Note over Payment: ❌ Timeout (30s)
    Payment-->>Order: Connection timeout
    Note over Order: ❌ Blocked waiting
    Order-->>API: 503 Service Unavailable
    Note over API: ❌ Error propagates up

    Note over API,Inventory: Payment going down<br/>took down the entire order flow!
```

**Solution**: Circuit breaker + timeout + retry (covered in [5_Resilience_Patterns.md](./5_Resilience_Patterns.md))

---

# 2. Asynchronous Communication

Service A sends a message to a queue/topic and **doesn't wait**. Service B processes it independently.

## Patterns

```mermaid
graph TB
    subgraph P2P["Point-to-Point (Queue)"]
        Producer1["Producer"] -->|"send"| Queue["Queue<br/>(RabbitMQ)"]
        Queue -->|"one consumer"| Consumer1["Consumer"]
    end

    subgraph PubSub["Pub/Sub (Topic)"]
        Producer2["Publisher"] -->|"publish"| Topic["Topic<br/>(Kafka)"]
        Topic -->|"subscribe"| SubA["Subscriber A"]
        Topic -->|"subscribe"| SubB["Subscriber B"]
        Topic -->|"subscribe"| SubC["Subscriber C"]
    end

    style P2P fill:#E3F2FD
    style PubSub fill:#FFF3E0
```

| Pattern | Tool | Use Case |
|---------|------|----------|
| **Queue** (one consumer) | RabbitMQ, SQS | Task processing, work distribution |
| **Pub/Sub** (many consumers) | Kafka, SNS/SQS | Event broadcasting, fan-out |
| **Event Streaming** | Kafka | Event sourcing, log aggregation |

## In Kubernetes

```yaml
# Kafka as StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-headless
  replicas: 3
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
        - name: kafka
          image: confluentinc/cp-kafka:7.5.0
          ports:
            - containerPort: 9092
          volumeMounts:
            - name: data
              mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: managed-ssd
        resources:
          requests:
            storage: 50Gi
---
# Headless service for stable DNS
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
    - port: 9092
# DNS: kafka-0.kafka-headless, kafka-1.kafka-headless, kafka-2.kafka-headless
```

## Pros and Cons

```mermaid
graph TB
    subgraph Pros["✅ Pros"]
        P1["Decoupled<br/>(services independent)"]
        P2["Resilient<br/>(queue buffers failures)"]
        P3["Scalable<br/>(add consumers freely)"]
        P4["Fan-out<br/>(one event, many consumers)"]
    end

    subgraph Cons["❌ Cons"]
        C1["Eventual consistency<br/>(not immediate)"]
        C2["Harder to debug<br/>(distributed flow)"]
        C3["Message ordering challenges"]
        C4["Infra complexity<br/>(need to manage queue)"]
    end

    style Pros fill:#C8E6C9
    style Cons fill:#FFCDD2
```

---

# 3. When to Use Which

```mermaid
graph TB
    Q1{"Need immediate<br/>response?"}
    Q1 -->|"Yes"| Sync["Use Sync (HTTP/gRPC)"]
    Q1 -->|"No"| Q2{"Multiple consumers<br/>for same event?"}

    Q2 -->|"Yes"| PubSub["Use Pub/Sub (Kafka)"]
    Q2 -->|"No"| Q3{"Need retry /<br/>buffer on failure?"}

    Q3 -->|"Yes"| Queue["Use Queue (RabbitMQ)"]
    Q3 -->|"No"| Sync2["Use Sync (simpler)"]

    style Sync fill:#E3F2FD
    style PubSub fill:#FFF3E0
    style Queue fill:#E8F5E9
    style Sync2 fill:#E3F2FD
```

| Scenario | Communication | Why |
|----------|--------------|-----|
| User places order → validate payment | **Sync** | User needs immediate confirmation |
| Order placed → send confirmation email | **Async** | Email can be delayed, user doesn't wait |
| Order placed → update inventory + analytics + notification | **Async (pub/sub)** | Fan-out to multiple consumers |
| Frontend → Backend API | **Sync (REST/gRPC)** | Direct request-response |
| Upload file → process (resize, scan) | **Async (queue)** | Processing is slow, don't block user |
| Batch data import | **Async (queue)** | Process in parallel with Job/CronJob consumers |

---

# 4. The Hybrid Pattern (Most Production Systems)

Real systems use **both** — sync for the critical path, async for everything else.

```mermaid
graph LR
    User["User"]
    Ingress["Ingress"]
    API["API Gateway<br/>(Deployment)"]
    Order["Order Svc<br/>(Deployment)"]
    Payment["Payment Svc<br/>(Deployment)"]
    Kafka["Kafka<br/>(StatefulSet)"]
    Notif["Notification Svc<br/>(Deployment)"]
    Inventory["Inventory Svc<br/>(Deployment)"]
    Analytics["Analytics Svc<br/>(Deployment)"]

    User -->|"HTTPS"| Ingress --> API
    API -->|"sync: need response"| Order
    Order -->|"sync: need confirmation"| Payment
    Order -->|"async: fire & forget"| Kafka
    Kafka --> Notif
    Kafka --> Inventory
    Kafka --> Analytics

    style User fill:#F3E5F5
    style Kafka fill:#FFF3E0
```

**Rule of thumb**: If the user is waiting → sync. If not → async.

---

# 5. Auto-Scaling Async Consumers (KEDA)

KEDA (Kubernetes Event-Driven Autoscaling) scales consumers based on queue depth — not CPU.

```mermaid
graph LR
    Kafka["Kafka Topic<br/>(lag: 10,000 msgs)"]
    KEDA["KEDA Controller"]
    Deploy["Consumer Deployment<br/>replicas: 1 → 10"]

    Kafka -->|"monitors lag"| KEDA
    KEDA -->|"scale up"| Deploy

    style Kafka fill:#FFF3E0
    style KEDA fill:#E3F2FD
    style Deploy fill:#E8F5E9
```

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-consumer-scaler
spec:
  scaleTargetRef:
    name: order-consumer          # Deployment name
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka-headless:9092
        consumerGroup: order-consumers
        topic: orders
        lagThreshold: "100"       # Scale when lag > 100
```

---

## Summary

| Aspect | Synchronous | Asynchronous |
|--------|------------|--------------|
| **Coupling** | Tight (both must be up) | Loose (independent) |
| **Consistency** | Strong (immediate) | Eventual |
| **Failure handling** | Cascading failures | Queue buffers failures |
| **Scaling** | HPA (CPU/memory) | KEDA (queue depth) |
| **Debugging** | Easier (request/response) | Harder (distributed tracing needed) |
| **K8s tools** | ClusterIP Service, gRPC | StatefulSet (Kafka), KEDA |
| **Best for** | User-facing requests | Background processing, fan-out |

---

> **Next**: [4_Service_Mesh.md](./4_Service_Mesh.md) — When and why to add a service mesh
