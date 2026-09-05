# Kubernetes Debugging & Troubleshooting

> **Prerequisites**: All previous K8s notes  
> **Next**: Helm — [../../Helm/Notes/6_Advanced_Templating.md](../../Helm/Notes/6_Advanced_Templating.md)

When things break in K8s (and they will), you need a systematic approach. This doc covers the mental framework and tools for debugging pods, services, networking, and storage.

---

## The Debugging Mental Model

```mermaid
graph TB
    Problem["Something is broken 🔥"]
    
    Problem --> Q1{"Pod running?"}
    Q1 -->|"No (Pending/CrashLoop)"| PodDebug["Debug Pod Issues"]
    Q1 -->|"Yes"| Q2{"App responding?"}
    
    Q2 -->|"No"| AppDebug["Debug App / Logs"]
    Q2 -->|"Yes inside pod"| Q3{"Reachable via Service?"}
    
    Q3 -->|"No"| SvcDebug["Debug Service / Networking"]
    Q3 -->|"Yes via ClusterIP"| Q4{"Reachable from outside?"}
    
    Q4 -->|"No"| IngressDebug["Debug Ingress / LB"]
    Q4 -->|"Yes"| OK["Everything works ✅"]

    style Problem fill:#FFCDD2
    style PodDebug fill:#FFF3E0
    style AppDebug fill:#FFF3E0
    style SvcDebug fill:#FFF3E0
    style IngressDebug fill:#FFF3E0
    style OK fill:#C8E6C9
```

**Debug from the inside out**: Pod → App → Service → Ingress.

---

# 1. Debugging Pods

## Pod Status Reference

| Status | Meaning | Action |
|--------|---------|--------|
| `Pending` | Can't be scheduled | Check events, resources, taints |
| `ContainerCreating` | Image pulling or volume mounting | Check image name, pull secrets, PVC |
| `Running` | All containers started | Check logs if app misbehaving |
| `CrashLoopBackOff` | Container keeps crashing | Check logs, command, health probes |
| `ImagePullBackOff` | Can't pull image | Check image name, registry access, pull secrets |
| `Evicted` | Node under resource pressure | Check node resources |
| `OOMKilled` | Container exceeded memory limit | Increase memory limit |
| `Error` | Container exited with error | Check logs |

## Step-by-Step Pod Debug

```bash
# Step 1: What's the pod status?
kubectl get pods -o wide

# Step 2: WHY is it in that status?
kubectl describe pod <pod-name>
# Look at:
#   - Events section (bottom) — scheduling, pulling, failures
#   - Conditions section — Ready, Initialized, ContainersReady
#   - Container status — state, reason, exit code

# Step 3: Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous       # Logs from crashed container
kubectl logs <pod-name> -c <container>   # Specific container in multi-container pod

# Step 4: Get into the container
kubectl exec -it <pod-name> -- sh
# Check: can the app reach its dependencies?
# wget -qO- http://db-svc:5432
# nslookup db-svc
# env | grep DB

# Step 5: Run a debug container (if main container has no shell)
kubectl debug -it <pod-name> --image=busybox --target=<container-name>
```

## Common Pod Issues

### CrashLoopBackOff

```mermaid
graph TB
    CrashLoop["CrashLoopBackOff"]
    CrashLoop --> C1["Wrong command/entrypoint"]
    CrashLoop --> C2["App crash on startup<br/>(missing env var, config)"]
    CrashLoop --> C3["Liveness probe failing<br/>(too aggressive)"]
    CrashLoop --> C4["OOM Kill<br/>(memory limit too low)"]
    CrashLoop --> C5["Permission denied<br/>(non-root user, read-only FS)"]

    style CrashLoop fill:#FFCDD2
```

```bash
# Check why it crashed
kubectl describe pod <pod> | grep -A5 "Last State"
kubectl logs <pod> --previous

# Check if OOM killed
kubectl describe pod <pod> | grep OOMKilled
```

### ImagePullBackOff

```bash
# Check image name is correct
kubectl describe pod <pod> | grep Image

# Check pull secrets
kubectl get pod <pod> -o yaml | grep imagePullSecrets

# Test image pull manually
docker pull <image>

# For private registries, verify secret exists
kubectl get secret <pull-secret> -o yaml
```

### Pending Pod

```bash
# Check events
kubectl describe pod <pod>

# Common reasons:
# - Insufficient CPU/memory on any node
kubectl describe nodes | grep -A5 "Allocated resources"

# - PVC not bound
kubectl get pvc

# - Node taints without tolerations
kubectl describe nodes | grep Taint

# - Node selector / affinity mismatch
kubectl get nodes --show-labels
```

---

# 2. Debugging Services

```mermaid
graph TB
    SvcIssue["Service not working"]
    SvcIssue --> S1["Endpoints empty?<br/>kubectl get endpoints svc-name"]
    SvcIssue --> S2["Labels match?<br/>Compare pod labels vs selector"]
    SvcIssue --> S3["Port mapping correct?<br/>port → targetPort → containerPort"]
    SvcIssue --> S4["DNS resolving?<br/>nslookup svc-name from a pod"]

    style SvcIssue fill:#FFCDD2
```

```bash
# Step 1: Does the service have endpoints?
kubectl get endpoints <service-name>
# If EMPTY → labels don't match OR pods aren't ready

# Step 2: Check label matching
kubectl get pods --show-labels
kubectl describe svc <service-name>
# Compare "Selector" with pod labels

# Step 3: Check port mapping
kubectl describe svc <service-name>
# Port: 80 (service port)
# TargetPort: 8080 (container port)
# Are these correct?

# Step 4: Test from inside the cluster
kubectl run test-pod --image=busybox -it --rm -- sh
# Inside:
wget -qO- http://<service-name>:<port>
nslookup <service-name>
nslookup <service-name>.<namespace>.svc.cluster.local
```

### Port Mapping Confusion

```mermaid
graph LR
    Client["Client request<br/>:80"] -->|"Service port"| Svc["Service<br/>port: 80"]
    Svc -->|"targetPort"| Pod["Pod<br/>containerPort: 8080"]

    style Client fill:#F3E5F5
    style Svc fill:#E3F2FD
    style Pod fill:#E8F5E9
```

| Field | Where | What |
|-------|-------|------|
| `port` | Service | What clients connect to |
| `targetPort` | Service | Where traffic is forwarded to (container port) |
| `containerPort` | Pod | What the app listens on |

`port` → `targetPort` must be correct. `containerPort` is informational but should match `targetPort`.

---

# 3. Debugging Networking

```bash
# Test DNS resolution
kubectl exec -it <pod> -- nslookup <service-name>

# Test connectivity to a service
kubectl exec -it <pod> -- curl -v http://<service-name>:<port>

# Check kube-proxy is running
kubectl get pods -n kube-system | grep kube-proxy

# Check Network Policies blocking traffic
kubectl get networkpolicies -A

# Run a network debug pod
kubectl run netdebug --image=nicolaka/netshoot -it --rm -- bash
# Inside: dig, nslookup, curl, ping, traceroute, tcpdump all available
```

---

# 4. Debugging Storage

```bash
# PVC stuck in Pending?
kubectl describe pvc <pvc-name>
# Check events for:
# - StorageClass not found
# - No matching PV
# - Volume in wrong zone

# Check PV binding
kubectl get pv
kubectl get pvc

# Check if volume is mounted inside pod
kubectl exec -it <pod> -- df -h
kubectl exec -it <pod> -- mount | grep <mount-path>

# Check if pod can write
kubectl exec -it <pod> -- touch /data/test-file
```

---

# 5. Debugging Nodes

```bash
# Node status
kubectl get nodes
kubectl describe node <node-name>

# Check conditions
kubectl get nodes -o wide

# Check resource pressure
kubectl top nodes
kubectl describe node <node> | grep -A10 "Conditions"

# Check allocated vs capacity
kubectl describe node <node> | grep -A10 "Allocated resources"

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### Node Conditions

| Condition | True Means |
|-----------|-----------|
| `Ready` | Node is healthy |
| `MemoryPressure` | Low memory |
| `DiskPressure` | Low disk space |
| `PIDPressure` | Too many processes |
| `NetworkUnavailable` | Network not configured |

---

# 6. Essential Debug Toolkit

```bash
# Quick status check
kubectl get all                           # Everything in current namespace
kubectl get all -A                        # Everything in all namespaces
kubectl get events --sort-by=.metadata.creationTimestamp

# Resource usage
kubectl top pods
kubectl top nodes

# Watch resources live
kubectl get pods -w
kubectl get events -w

# Get YAML of any resource
kubectl get <resource> <name> -o yaml

# Diff between desired and actual
kubectl diff -f manifest.yaml

# Dry run (test without applying)
kubectl apply -f manifest.yaml --dry-run=client
kubectl apply -f manifest.yaml --dry-run=server

# Force delete stuck resources
kubectl delete pod <pod> --force --grace-period=0
```

---

# 7. Debugging Decision Tree

```mermaid
graph TB
    Start["Issue reported"]
    
    Start --> CheckPods["kubectl get pods -o wide"]
    CheckPods --> PodOK{"All pods Running<br/>and Ready?"}
    
    PodOK -->|"No"| DescPod["kubectl describe pod <pod><br/>kubectl logs <pod> --previous"]
    DescPod --> FixPod["Fix: image, env, resources,<br/>probes, permissions"]
    
    PodOK -->|"Yes"| CheckSvc["kubectl get endpoints <svc>"]
    CheckSvc --> SvcOK{"Endpoints<br/>populated?"}
    
    SvcOK -->|"No"| FixSvc["Fix: label selector,<br/>readiness probe,<br/>port mapping"]
    
    SvcOK -->|"Yes"| TestInternal["kubectl exec -- curl svc:port"]
    TestInternal --> InternalOK{"Response<br/>OK?"}
    
    InternalOK -->|"No"| FixNet["Fix: NetworkPolicy,<br/>kube-proxy, DNS"]
    
    InternalOK -->|"Yes"| TestExternal["curl external-ip/ingress"]
    TestExternal --> ExternalOK{"Response<br/>OK?"}
    
    ExternalOK -->|"No"| FixIngress["Fix: Ingress rules, TLS,<br/>LB config, DNS records"]
    ExternalOK -->|"Yes"| Done["Working ✅"]

    style Start fill:#FFCDD2
    style Done fill:#C8E6C9
```

---

## Quick Reference

| Problem | First Command |
|---------|--------------|
| Pod not starting | `kubectl describe pod <pod>` |
| App crashing | `kubectl logs <pod> --previous` |
| Service unreachable | `kubectl get endpoints <svc>` |
| DNS not resolving | `kubectl exec -- nslookup <svc>` |
| PVC stuck Pending | `kubectl describe pvc <pvc>` |
| Node issues | `kubectl describe node <node>` |
| Everything | `kubectl get events --sort-by=.metadata.creationTimestamp` |

---

> **Kubernetes section complete!** You've covered: intro → architecture → pods → YAML → ReplicaSets → Deployments → networking → services → configmaps/secrets → storage → workloads → scheduling → RBAC → ingress → debugging.
>
> **Next up**: Helm advanced topics starting with [Advanced Templating](../../Helm/Notes/6_Advanced_Templating.md)
