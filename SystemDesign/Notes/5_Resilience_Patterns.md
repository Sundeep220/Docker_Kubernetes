# Resilience Patterns for Distributed Systems

> **Prerequisites**: [3_Sync_vs_Async_Communication.md](./3_Sync_vs_Async_Communication.md), [4_Service_Mesh.md](./4_Service_Mesh.md)  
> **Next**: [6_Zero_Trust_Security.md](./6_Zero_Trust_Security.md)

In distributed systems, **failure is inevitable**. Networks fail, services crash, databases get overloaded. Resilience patterns ensure your system keeps working despite partial failures.

---

## Failure Modes → Solutions

```mermaid
graph TB
    subgraph Failures["What Can Go Wrong"]
        F1["Pod crashes"]
        F2["Pod overloaded"]
        F3["Downstream service down"]
        F4["Downstream service slow"]
        F5["Network partition"]
        F6["Node failure"]
        F7["Entire zone down"]
        F8["Bad deploy"]
        F9["Traffic spike (10x)"]
    end

    subgraph Solutions["K8s / Design Solutions"]
        S1["Liveness probe + restart"]
        S2["HPA + resource limits"]
        S3["Circuit breaker + fallback"]
        S4["Timeout + retry w/ backoff"]
        S5["Retry + idempotent APIs + queue buffer"]
        S6["Pod rescheduled + PDB"]
        S7["Multi-AZ spread + topologySpreadConstraints"]
        S8["Rolling update + readiness probe + rollback"]
        S9["HPA + rate limiting + queue buffering"]
    end

    F1 --> S1
    F2 --> S2
    F3 --> S3
    F4 --> S4
    F5 --> S5
    F6 --> S6
    F7 --> S7
    F8 --> S8
    F9 --> S9
```

---

# 1. Health Probes — Automatic Recovery

```mermaid
graph LR
    subgraph Probes["K8s Health Probes"]
        Liveness["Liveness Probe<br/>Is the process alive?<br/>Fail → restart container"]
        Readiness["Readiness Probe<br/>Can it handle traffic?<br/>Fail → remove from Service"]
        Startup["Startup Probe<br/>Has it finished booting?<br/>Fail → don't check liveness yet"]
    end

    style Liveness fill:#FFCDD2
    style Readiness fill:#FFF9C4
    style Startup fill:#E8F5E9
```

```yaml
spec:
  containers:
    - name: api
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3      # 3 failures → restart

      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 2      # 2 failures → remove from endpoints

      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 2          # 30 × 2s = 60s max startup time
```

### What Each Probe Should Check

| Probe | Check | Example |
|-------|-------|---------|
| **Liveness** | Is the process healthy? | Simple `/healthz` → 200 OK |
| **Readiness** | Can it handle requests? | DB connected? Cache warm? `/ready` |
| **Startup** | Has it finished booting? | Same as liveness, but generous timeout |

---

# 2. Retry + Timeout + Circuit Breaker — The Holy Trinity

```mermaid
sequenceDiagram
    participant Client as Client
    participant Timeout as Timeout (3s)
    participant Retry as Retry (3 attempts)
    participant CB as Circuit Breaker
    participant Server as Server

    Client->>Timeout: Request
    Timeout->>Retry: Forward
    
    Retry->>CB: Attempt 1
    CB->>Server: Forward
    Server-->>CB: ❌ 500 Error
    CB-->>Retry: Failed

    Note over Retry: Wait 100ms (backoff)

    Retry->>CB: Attempt 2
    CB->>Server: Forward
    Server-->>CB: ❌ 500 Error
    CB-->>Retry: Failed

    Note over Retry: Wait 200ms (backoff × 2)

    Retry->>CB: Attempt 3
    CB->>Server: Forward
    Server-->>CB: ✅ 200 OK
    CB-->>Retry: Success
    Retry-->>Timeout: Success
    Timeout-->>Client: 200 OK
```

### Timeout

**Never make a call without a timeout.** If the downstream is slow, your threads/connections pile up and your service dies too.

```yaml
# Istio VirtualService
timeout: 3s

# App-level (e.g., HTTP client)
# httpClient.setTimeout(3000)
```

### Retry with Exponential Backoff

Don't hammer a failing service. Wait longer between retries.

```
Attempt 1: wait 100ms
Attempt 2: wait 200ms
Attempt 3: wait 400ms
Attempt 4: wait 800ms + jitter
```

**Jitter** = random delay to prevent all clients retrying at the same time (thundering herd).

```yaml
# Istio VirtualService
retries:
  attempts: 3
  perTryTimeout: 1s
  retryOn: 5xx,reset,connect-failure
```

### Circuit Breaker

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open : Failure threshold reached<br/>(e.g., 5 consecutive 5xx)
    Open --> HalfOpen : After cooldown period<br/>(e.g., 30 seconds)
    HalfOpen --> Closed : Test request succeeds
    HalfOpen --> Open : Test request fails

    note right of Closed : Normal operation<br/>All requests pass through
    note right of Open : Fail fast<br/>Don't even call the server
    note right of HalfOpen : Test mode<br/>Let one request through
```

| State | Behavior |
|-------|----------|
| **Closed** | Normal — all requests go through |
| **Open** | Blocking — fail immediately, don't overload the failing service |
| **Half-Open** | Testing — let one request through; if it works, close again |

### Where to Implement

| Approach | Tool | When |
|----------|------|------|
| **App-level** | Resilience4j (Java), Polly (.NET), go-circuit | Few services, fine control |
| **Infra-level** | Istio DestinationRule | Many services, no code changes |

---

# 3. Pod Disruption Budgets (PDB) — Survive Node Drains

During node upgrades or scaling, K8s drains nodes (evicts pods). PDB ensures a minimum number of pods stay running.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2              # Always keep at least 2 pods running
  # OR: maxUnavailable: 1      # At most 1 pod can be down
  selector:
    matchLabels:
      app: api
```

```mermaid
graph TB
    subgraph NoPDB["Without PDB"]
        Drain1["kubectl drain node-1"]
        Kill1["All 3 api pods on node-1<br/>evicted at once ❌"]
        Down1["Service downtime!"]
        Drain1 --> Kill1 --> Down1
    end

    subgraph WithPDB["With PDB (minAvailable: 2)"]
        Drain2["kubectl drain node-1"]
        Kill2["Evict 1 pod from node-1"]
        Wait2["Wait for replacement<br/>pod on another node"]
        Kill3["Evict next pod"]
        Up2["At least 2 pods<br/>always running ✅"]
        Drain2 --> Kill2 --> Wait2 --> Kill3 --> Up2
    end

    style NoPDB fill:#FFCDD2
    style WithPDB fill:#C8E6C9
```

---

# 4. Pod Anti-Affinity — Spread Across Nodes/Zones

Don't put all replicas on the same node — if it fails, all pods die.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: api
          topologyKey: kubernetes.io/hostname     # Different nodes

  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone    # Spread across AZs
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: api
```

```mermaid
graph TB
    subgraph Bad["❌ All pods on one node"]
        N1["Node 1<br/>api-1, api-2, api-3"]
        N2["Node 2<br/>(empty)"]
        N3["Node 3<br/>(empty)"]
    end

    subgraph Good["✅ Spread across nodes + zones"]
        N4["Node 1 (Zone A)<br/>api-1"]
        N5["Node 2 (Zone B)<br/>api-2"]
        N6["Node 3 (Zone C)<br/>api-3"]
    end

    style Bad fill:#FFCDD2
    style Good fill:#C8E6C9
```

---

# 5. Rate Limiting — Protect from Overload

```yaml
# NGINX Ingress rate limiting
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "100"          # 100 requests/sec per IP
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"  # Allow burst of 500
    nginx.ingress.kubernetes.io/limit-connections: "10"      # 10 concurrent connections
```

---

# 6. Graceful Shutdown — Don't Drop In-Flight Requests

When K8s terminates a pod (during scale-down or deploy), it sends SIGTERM. Your app should finish in-flight requests before exiting.

```mermaid
sequenceDiagram
    participant K8s as Kubernetes
    participant Pod as Pod
    participant Svc as Service Endpoints

    K8s->>Svc: Remove pod from endpoints<br/>(no new traffic)
    K8s->>Pod: Send SIGTERM
    Note over Pod: App receives SIGTERM<br/>1. Stop accepting new requests<br/>2. Finish in-flight requests<br/>3. Close DB connections<br/>4. Exit gracefully
    Pod-->>K8s: Process exits (code 0)
    
    Note over K8s: If pod doesn't exit<br/>in terminationGracePeriodSeconds (30s)<br/>→ SIGKILL (forced)
```

```yaml
spec:
  terminationGracePeriodSeconds: 60     # Give app 60s to finish
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]  # Wait for endpoints to update
```

---

## Summary

| Pattern | Protects Against | K8s Implementation |
|---------|-----------------|-------------------|
| **Liveness Probe** | Stuck/crashed process | `livenessProbe` in pod spec |
| **Readiness Probe** | Serving when not ready | `readinessProbe` in pod spec |
| **Timeout** | Slow downstream calls | Istio VirtualService / app-level |
| **Retry + Backoff** | Transient failures | Istio / app-level |
| **Circuit Breaker** | Cascading failures | Istio DestinationRule / app-level |
| **PDB** | Node drain killing all pods | PodDisruptionBudget |
| **Anti-Affinity** | Node/zone failure | podAntiAffinity / topologySpreadConstraints |
| **HPA** | Traffic spikes | HorizontalPodAutoscaler |
| **Rate Limiting** | DDoS / overload | Ingress annotations |
| **Graceful Shutdown** | Dropped in-flight requests | preStop hook + terminationGracePeriodSeconds |

---

> **Next**: [6_Zero_Trust_Security.md](./6_Zero_Trust_Security.md) — Designing security for distributed systems
