Great — let's master **Pod YAML** from the ground up.
This will also help you later with Deployments, Services, StatefulSets, etc.

We’ll cover:

1️⃣ **Structure of any Kubernetes YAML**
2️⃣ **Required fields in every YAML**
3️⃣ **Deep explanation of Pod YAML fields**
4️⃣ **Example Pod YAMLs (basic → advanced)**
5️⃣ **Commands to create/manage/delete YAML Pods**

---

# 🧩 **1. The Structure of ANY Kubernetes YAML**

A Kubernetes manifest ALWAYS has these **4 top-level mandatory fields**:

```yaml
apiVersion: <API group/version>
kind: <Resource Type>
metadata:
  <Object metadata: name, labels, namespace>
spec:
  <Resource specification>
```

If you understand these 4, you understand 80% of Kubernetes YAML.

---

# 🧱 **2. Required Fields Explained**

Let’s break them down.

---

## 🧩 **2.1 apiVersion**

Defines WHICH API version this object uses.

For Pods:

```yaml
apiVersion: v1
```

Different objects have different versions:

* Deployments → `apps/v1`
* Ingress → `networking.k8s.io/v1`
* StatefulSets → `apps/v1`

---

## 🧩 **2.2 kind**

What object you are creating.

Examples:

* `Pod`
* `Deployment`
* `Service`
* `StatefulSet`
* `ConfigMap`

For now:

```yaml
kind: Pod
```

---

## 🧩 **2.3 metadata**

This section identifies the object.

Most common fields:

```yaml
metadata:
  name: my-pod
  namespace: dev
  labels:
    app: myapp
    tier: backend
  annotations:
    owner: "MBRDI team"
```

### Purpose:

✔ Name & namespace: uniquely identify resources
✔ Labels: used by Services, Deployments, selectors
✔ Annotations: add extra metadata (ignored by Kubernetes)

---

## 🧩 **2.4 spec**

This is the *heart* of the object.

For Pods, you define:

```yaml
spec:
  containers:
  initContainers:
  volumes:
  restartPolicy:
  nodeSelector:
  affinity:
  tolerations:
  serviceAccountName:
  dnsPolicy:
  hostNetwork:
  securityContext:
```

But for a **basic Pod**, only one field is mandatory:

* `containers`

---

# 🧱 **3. Deep-Dive Into Pod YAML Fields**

Let's explore the important fields.

---

## 🍱 **3.1 containers (MANDATORY)**

Minimum required:

```yaml
containers:
  - name: app
    image: nginx
```

Optional features:

* `ports`
* `command`
* `args`
* `env`
* `resources`
* `volumeMounts`
* `livenessProbe`
* `readinessProbe`
* `startupProbe`
* `securityContext`

Example:

```yaml
containers:
  - name: app
    image: nginx:latest
    ports:
      - containerPort: 80
    env:
      - name: APP_COLOR
        value: blue
```

---

## 🔁 **3.2 restartPolicy**

Default: `Always`

Options:

* `Always`
* `OnFailure`
* `Never`

Example:

```yaml
restartPolicy: OnFailure
```

---

## 🧪 **3.3 Probes (Health Checks)**

### Liveness probe

Restart container if unhealthy.

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
```

### Readiness probe

Control when Pod receives traffic.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

---

## 🔐 **3.4 Environment variables**

```yaml
env:
  - name: DB_HOST
    value: "mysql"
```

From secret:

```yaml
env:
  - name: PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

---

## 🗂 **3.5 Volumes + Mounts**

Volume definition:

```yaml
volumes:
  - name: html
    emptyDir: {}
```

Mount volume into container:

```yaml
volumeMounts:
  - name: html
    mountPath: /usr/share/nginx/html
```

---

## 🛠 **3.6 Init Containers**

Init containers run **before** main containers.

```yaml
initContainers:
  - name: setup
    image: busybox
    command: ["sh", "-c", "echo Setup done"]
```

---

## 🌎 **3.7 Node scheduling**

### Node selectors:

```yaml
nodeSelector:
  disktype: ssd
```

### Tolerations:

```yaml
tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"
```

---

# 🚀 **4. Putting it Together — Different Pod YAML Examples**

---

# ✔️ **4.1 Basic Pod YAML**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

---

# ✔️ **4.2 Pod with Environment Variables**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: color-app
spec:
  containers:
    - name: app
      image: kodekloud/simple-webapp
      env:
        - name: APP_COLOR
          value: "blue"
```

---

# ✔️ **4.3 Multi-Container Pod (Sidecar Pattern)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
    - name: main-app
      image: nginx

    - name: logger
      image: busybox
      command: ["sh", "-c", "while true; do echo log; sleep 5; done"]
```

---

# ✔️ **4.4 Pod With Init Container**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-example
spec:
  initContainers:
    - name: prep
      image: busybox
      command: ["sh", "-c", "echo initializing...; sleep 5"]

  containers:
    - name: app
      image: nginx
```

---

# ✔️ **4.5 Pod with Volume**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  volumes:
    - name: data
      emptyDir: {}

  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /var/www/html
          name: data
```

---

# 📌 **Commands to Apply and Manage Pod YAML**

### Apply configuration

```bash
kubectl apply -f nginx-pod.yaml
```

### Get pod

```bash
kubectl get pod
```

### Describe pod

```bash
kubectl describe pod nginx-pod
```

### Get YAML running configuration

```bash
kubectl get pod nginx-pod -o yaml
```

### Delete pod

```bash
kubectl delete -f nginx-pod.yaml
```

---

# 🎯 Summary (What You Must Always Think in YAML)

For any object:

```
apiVersion   
kind         
metadata     
spec         
```

For Pods, MUST include:

```
spec:
  containers:
```

Everything else (probes, env, volumes, init containers, security, resources)
is optional — but important for production.

---
