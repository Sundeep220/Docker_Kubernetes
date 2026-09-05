# Observability — Metrics, Logs & Traces in Kubernetes

> **Prerequisites**: [7_System_Design_Example.md](./7_System_Design_Example.md)  
> **Next**: [10_Interview_Scenarios.md](./10_Interview_Scenarios.md)

You can't fix what you can't see. Observability gives you visibility into your distributed system's health, performance, and behavior through **three pillars**: metrics, logs, and traces.

---

## The Three Pillars

```mermaid
graph TB
    subgraph Metrics["📊 Metrics (What's happening NOW?)"]
        M1["Prometheus scrapes /metrics"]
        M2["Grafana dashboards"]
        M3["Alertmanager → PagerDuty/Slack"]
    end

    subgraph Logs["📝 Logs (What HAPPENED?)"]
        L1["Fluent Bit (DaemonSet)<br/>collects stdout/stderr"]
        L2["Elasticsearch / Loki"]
        L3["Kibana / Grafana"]
    end

    subgraph Traces["🔍 Traces (WHY is it slow?)"]
        T1["OpenTelemetry SDK in app"]
        T2["Jaeger / Tempo"]
        T3["Request journey across services"]
    end

    style Metrics fill:#E3F2FD
    style Logs fill:#FFF3E0
    style Traces fill:#E8F5E9
```

---

# Pillar 1: Metrics — Prometheus + Grafana

## How Prometheus Works in K8s

```mermaid
graph LR
    subgraph Pods["Application Pods"]
        App1["Order Svc<br/>/metrics endpoint"]
        App2["Payment Svc<br/>/metrics endpoint"]
        App3["User Svc<br/>/metrics endpoint"]
    end

    subgraph Infra["Infrastructure"]
        NE["Node Exporter<br/>(DaemonSet)<br/>CPU, memory, disk"]
        KSM["kube-state-metrics<br/>(Deployment)<br/>Pod status, replica count"]
    end

    Prom["Prometheus<br/>(Deployment)"]
    Grafana["Grafana<br/>(Dashboards)"]
    Alert["Alertmanager<br/>(PagerDuty, Slack)"]

    Prom -->|"scrape every 15s"| App1
    Prom -->|"scrape"| App2
    Prom -->|"scrape"| App3
    Prom -->|"scrape"| NE
    Prom -->|"scrape"| KSM
    Prom --> Grafana
    Prom --> Alert

    style Pods fill:#E3F2FD
    style Infra fill:#E8F5E9
```

### Key Metrics: The RED Method

| Metric | What it Measures | Alert When |
|--------|-----------------|------------|
| **R**ate | Requests per second | Sudden drop or spike |
| **E**rror | Error rate (5xx / total) | > 1% error rate |
| **D**uration | Latency (p50, p95, p99) | p99 > 500ms |

### Additional Metrics to Monitor

| Metric | Source | Why |
|--------|--------|-----|
| CPU utilization | Node Exporter | Scaling decisions |
| Memory utilization | Node Exporter | OOM risk |
| Pod restart count | kube-state-metrics | Crash loops |
| Pod pending count | kube-state-metrics | Scheduling issues |
| PVC usage | kubelet metrics | Disk full risk |
| HPA current vs desired | kube-state-metrics | Scaling working? |

### Prometheus in K8s (via kube-prometheus-stack)

```bash
# Install the full stack (Prometheus + Grafana + Alertmanager + exporters)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

### ServiceMonitor — Tell Prometheus to Scrape Your App

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service-metrics
spec:
  selector:
    matchLabels:
      app: order-service
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

### Example Alert Rule

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
spec:
  groups:
    - name: order-service
      rules:
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{app="order-service",status=~"5.."}[5m])
            / rate(http_requests_total{app="order-service"}[5m]) > 0.01
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Order service error rate > 1%"

        - alert: HighLatency
          expr: |
            histogram_quantile(0.99,
              rate(http_request_duration_seconds_bucket{app="order-service"}[5m])
            ) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Order service p99 latency > 500ms"
```

---

# Pillar 2: Logs — Fluent Bit + Loki/ELK

## Log Collection Architecture

```mermaid
graph LR
    subgraph Nodes["Every Node"]
        Pod1["Pod A<br/>stdout/stderr"]
        Pod2["Pod B<br/>stdout/stderr"]
        FB["Fluent Bit<br/>(DaemonSet)"]
        Pod1 -->|"/var/log/containers/"| FB
        Pod2 -->|"/var/log/containers/"| FB
    end

    subgraph Storage["Log Storage"]
        Loki["Loki<br/>(lightweight)"]
        ELK["Elasticsearch<br/>(full-featured)"]
    end

    subgraph UI["Visualization"]
        Grafana["Grafana<br/>(for Loki)"]
        Kibana["Kibana<br/>(for ELK)"]
    end

    FB --> Loki --> Grafana
    FB --> ELK --> Kibana

    style Nodes fill:#E3F2FD
    style Storage fill:#FFF3E0
    style UI fill:#E8F5E9
```

### Best Practices for Logging

| Practice | Why |
|----------|-----|
| **Structured JSON logs** | Machine-parseable, searchable |
| **Correlation IDs** | Trace a request across services |
| **Log to stdout/stderr** | K8s captures automatically |
| **Don't log sensitive data** | Passwords, tokens, PII |
| **Include context** | service, method, user_id, request_id |

### Structured Log Example

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "service": "order-service",
  "method": "POST /orders",
  "request_id": "abc-123-def",
  "user_id": "user-456",
  "error": "payment declined",
  "duration_ms": 245
}
```

### Fluent Bit DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: containerlog
              mountPath: /var/log/containers
              readOnly: true
          resources:
            limits:
              cpu: 200m
              memory: 128Mi
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containerlog
          hostPath:
            path: /var/log/containers
```

---

# Pillar 3: Distributed Tracing — Jaeger / Tempo

## Why Traces Matter

In a microservice architecture, a single request passes through many services. When it's slow, you need to know WHERE the bottleneck is.

```mermaid
graph LR
    subgraph Trace["Request Trace: POST /orders"]
        Span1["API Gateway<br/>2ms"]
        Span2["Order Service<br/>15ms"]
        Span3["Inventory Check<br/>8ms"]
        Span4["Payment Service<br/>120ms ⚠️"]
        Span5["DB Query<br/>95ms 🔥"]

        Span1 --> Span2
        Span2 --> Span3
        Span2 --> Span4
        Span4 --> Span5
    end

    style Span5 fill:#FFCDD2
    style Span4 fill:#FFF9C4
```

Without tracing, you'd only see "POST /orders took 150ms" and have no idea where the time went.

## How It Works

```mermaid
sequenceDiagram
    participant GW as API Gateway
    participant Order as Order Service
    participant Pay as Payment Service
    participant DB as Database

    Note over GW: Generate trace_id: abc-123
    GW->>Order: trace_id: abc-123, span_id: 1
    Order->>Pay: trace_id: abc-123, span_id: 2
    Pay->>DB: trace_id: abc-123, span_id: 3
    DB-->>Pay: Response (95ms)
    Pay-->>Order: Response (120ms)
    Order-->>GW: Response (150ms)

    Note over GW,DB: All spans with trace_id: abc-123<br/>are stitched together in Jaeger/Tempo
```

### OpenTelemetry Setup

```yaml
# OpenTelemetry Collector (Deployment)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel
          image: otel/opentelemetry-collector:latest
          ports:
            - containerPort: 4317   # gRPC receiver
            - containerPort: 4318   # HTTP receiver
```

Your app sends traces to the collector, which forwards to Jaeger/Tempo.

---

## Putting It All Together

```mermaid
graph TB
    subgraph App["Application"]
        Pod["Your Pod"]
    end

    subgraph Observability["Observability Stack"]
        subgraph MetricsStack["Metrics"]
            Prom["Prometheus<br/>(scrapes /metrics)"]
            Grafana["Grafana<br/>(dashboards)"]
            Alert["Alertmanager<br/>(alerts)"]
        end

        subgraph LogStack["Logs"]
            FB["Fluent Bit<br/>(DaemonSet)"]
            Loki["Loki"]
        end

        subgraph TraceStack["Traces"]
            OTel["OpenTelemetry<br/>Collector"]
            Jaeger["Jaeger / Tempo"]
        end
    end

    Pod -->|"/metrics"| Prom --> Grafana
    Prom --> Alert
    Pod -->|"stdout"| FB --> Loki --> Grafana
    Pod -->|"trace spans"| OTel --> Jaeger --> Grafana

    style MetricsStack fill:#E3F2FD
    style LogStack fill:#FFF3E0
    style TraceStack fill:#E8F5E9
```

**Grafana** becomes the single pane of glass — metrics, logs, and traces all in one UI.

---

## Summary

| Pillar | Tool | Deployed As | What It Answers |
|--------|------|-------------|-----------------|
| **Metrics** | Prometheus + Grafana | Deployment + ServiceMonitor | "What's happening now?" |
| **Logs** | Fluent Bit + Loki | DaemonSet + Deployment | "What happened?" |
| **Traces** | OpenTelemetry + Jaeger | SDK in app + Deployment | "Why is it slow?" |
| **Alerts** | Alertmanager | Deployment + PrometheusRule | "Is something wrong?" |

---

> **Next**: [10_Interview_Scenarios.md](./10_Interview_Scenarios.md) — Common system design interview scenarios mapped to K8s
