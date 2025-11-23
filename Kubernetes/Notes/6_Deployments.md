Perfect — **Deployments** are one of the most critical concepts in Kubernetes.
We’ll go *deep* into architecture, internals, how rolling updates work, how rollbacks work, how Deployments interact with ReplicaSets and Pods, YAML structure, and commands.

This will give you **production-level Kubernetes knowledge**.

---

# 🚀 **1. What is a Deployment?**

A **Deployment** is a higher-level Kubernetes object that manages:

* **ReplicaSets**
* **Pods**
* **Rolling updates**
* **Rollbacks**
* **Versioning**
* **Declarative updates**
* **Scaling**

Deployment = “Controller” that ensures your application runs correctly *across its entire lifecycle*.

---

# 🏗 **2. Deployment Architecture — Deep Internal Flow**

### Flow inside Kubernetes:

```
Deployment Controller
     ↓
Creates / Manages ReplicaSets
     ↓
ReplicaSet creates Pods
```

### Why this intermediate RS layer?

Because:

✔ Rolling updates need multiple ReplicaSets
✔ Each Deployment version = a new ReplicaSet
✔ Easy rollback (just switch to the previous RS)

---

# 🧱 **3. Deployment Internals in Detail**

### When you create a Deployment:

1️⃣ Deployment creates a **ReplicaSet (RS v1)**
2️⃣ RS creates the **pods**
3️⃣ If you update image/version:

* Deployment creates **RS v2**
* Scales **RS v1 down**, **RS v2 up**
* Controlled using deployment strategy

This mechanism is what makes rolling updates smooth.

---

# 🧩 **4. Deployment Strategies**

Kubernetes supports **two strategies**:

---

## 🟦 **1. RollingUpdate (default)**

This allows:

* Zero-downtime deployment
* Controlled rollout
* Gradual replacement

Controlled using:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

### Meaning:

* **maxSurge: 1** → at most 1 extra pod above desired
* **maxUnavailable: 1** → at most 1 pod down during rollout

---

## 🟥 **2. Recreate**

All old pods deleted → new pods created.

```yaml
strategy:
  type: Recreate
```

Use when:

* No stateful compatibility
* Database migrations incompatible
* Single-instance apps

---

# 📄 **5. Deep Dive into Deployment YAML**

Let’s break the YAML into key sections.

---

## ✔️ **Full Deployment YAML Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

---

# 🎯 **6. Key Fields You MUST Understand**

### ✔️ **spec.selector**

Links Deployment → ReplicaSet → Pods
MUST match template labels.

### ✔️ **spec.replicas**

Number of desired pod replicas.

### ✔️ **spec.strategy**

Controls how updates happen.

### ✔️ **spec.template**

The Pod template that ReplicaSet clones.

---

# 🧪 **7. Important kubectl Commands for Deployments**

---

## ➤ **Create Deployment from YAML**

```
kubectl apply -f deployment.yaml
```

---

## ➤ **List Deployments**

```
kubectl get deployments
```

---

## ➤ **Describe Deployment**

```
kubectl describe deployment nginx-deployment
```

---

## ➤ **Get ReplicaSets created by Deployment**

```
kubectl get rs
```

---

## ➤ **Get Pods**

```
kubectl get pods -l app=nginx
```

---

## ➤ **Update Deployment image**

```
kubectl set image deployment/nginx-deployment nginx=nginx:1.28
```

This automatically triggers a **rolling update**.

---

## ➤ **See rollout status**

```
kubectl rollout status deployment/nginx-deployment
```

---

## ➤ **Rollback Deployment**

```
kubectl rollout undo deployment/nginx-deployment
```

---

## ➤ **Rollback to a specific revision**

```
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

---

## ➤ **Pause a rollout**

```
kubectl rollout pause deployment/nginx-deployment
```

---

## ➤ **Resume a rollout**

```
kubectl rollout resume deployment/nginx-deployment
```

---

# 🔍 **8. Deployment Rollout Versions Explained**

Every rollout creates a new ReplicaSet:

```
nginx-deployment-748c8cdf8     (old version)
nginx-deployment-6fb7ccd9db    (current version)
```

Kubernetes maintains rollout history:

```
kubectl rollout history deployment/nginx-deployment
```

Useful for:

* Debugging broken releases
* Quickly rolling back

---

# 🤖 **9. Self-Healing with Deployment**

If a pod crashes:

* ReplicaSet recreates the pod
* Deployment monitors state
* Ensures correct number of replicas

If the entire RS gets corrupted:

* Deployment itself repairs it
* Or re-creates a RS

---

# 🧠 **10. Deployment Conditions**

Deployment tracks conditions like:

* **Progressing**
* **Available**
* **ReplicaFailure**

Check via:

```
kubectl describe deployment nginx-deployment
```

---

# 📌 **11. Deployment Best Practices**

### ✔ Always use labels consistently

```yaml
selector:
  matchLabels:
    app: myapp
template:
  metadata:
    labels:
      app: myapp
```

### ✔ Never change the selector after creation

This breaks the Deployment → RS mapping.

### ✔ Use resource limits

Deployments must define:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
```

### ✔ Use readiness & liveness probes

For robust rollouts:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
readinessProbe:
  httpGet:
    path: /
    port: 80
```

### ✔ Use RollingUpdate for production

Default behavior is safest.

---

# 🎉 Summary

### ✔ Deployment manages:

* **ReplicaSets**
* **Pods**
* **Rollouts**
* **Rollbacks**
* **Scaling**
* **Zero-downtime updates**

### ✔ Deployment workflow:

```
Deployment → ReplicaSets → Pods
```

### ✔ Deployment is the *recommended* way to run stateless applications.

---

### 📌 **Create Deployment**

```bash
kubectl create deployment my-deployment --image=nginx
```

---

### 📌 **List Deployments**

```bash
kubectl get deployments
```

---

### 📌 **Describe a Deployment (detailed info)**

```bash
kubectl describe deployment my-deployment
```

---

### 📌 **Edit Deployment**

```bash
kubectl edit deployment my-deployment
```

---

### 📌 **Update Image in a Deployment**

```bash
kubectl set image deployment/my-deployment nginx=nginx:1.21
```

---

### 📌 **Scale Deployment (change replicas)**

```bash
kubectl scale deployment my-deployment --replicas=5
```

---

### 📌 **Check Rollout Status**

```bash
kubectl rollout status deployment/my-deployment
```

---

### 📌 **History of Rollouts**

```bash
kubectl rollout history deployment/my-deployment
```

---

### 📌 **Undo Last Rollout**

```bash
kubectl rollout undo deployment/my-deployment
```

---

### 📌 **Expose Deployment as Service**

```bash
kubectl expose deployment my-deployment --type=NodePort --port=80
```

---

### 📌 **Delete Deployment**

```bash
kubectl delete deployment my-deployment
```

---

### 📌 Bonus — YAML Output of Deployment

```bash
kubectl get deployment my-deployment -o yaml
```

---
