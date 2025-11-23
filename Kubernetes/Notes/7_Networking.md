Perfect — **Kubernetes Networking** is one of the most important and most misunderstood parts of Kubernetes.
We’ll break it down deeply, clearly, and practically.

This guide will help you understand:

✅ Pod-to-Pod networking
✅ Pod-to-Service networking
✅ Node networking
✅ CNI (Container Network Interface)
✅ kube-proxy internals
✅ ClusterIP, NodePort, LoadBalancer
✅ IP assignment
✅ DNS
✅ Network policies
✅ Ingress (overview)

By the end, you’ll understand Kubernetes networking at a **production + interview-ready** level.

---

# 🚀 **1. Kubernetes Networking — The 4 Key Rules**

Kubernetes has **four fundamental networking rules**:

### 👇 **Rule 1: Every Pod has its own IP**

* Pods are assigned unique IPs.
* Pods do NOT share the node’s IP.
* This is called the **“IP-per-pod”** model.

### 👇 **Rule 2: Pods can reach all other Pods without NAT**

* All Pods in the cluster must communicate directly.
* No port-mapping required.

### 👇 **Rule 3: Agents on Node can reach Pods directly**

* kubelet, logs collector, metrics agent can talk to Pods.

### 👇 **Rule 4: Services provide stable virtual IPs**

* Pod IPs change when Pods restart.
* Services solve this by providing a stable virtual IP (ClusterIP).

These rules require a network plugin → **CNI**.

---

# 🧩 **2. CNI (Container Network Interface)**

CNI is responsible for:

* Assigning Pod IPs
* Creating virtual networks
* Enabling Pod-to-Pod communication
* NAT / Routing
* Enforcing network policies

Popular CNIs:

| CNI         | Features                         |
| ----------- | -------------------------------- |
| **Calico**  | Layer 3, NetworkPolicy, scalable |
| **Flannel** | Simple overlay network           |
| **Weave**   | Built-in encryption              |
| **Cilium**  | eBPF-based, fastest              |

**Kubernetes does NOT manage networking itself.
CNI does.**

---

# 🌐 **3. How Pods Get Their IP**

When a Pod is scheduled:

1️⃣ kubelet calls CNI plugin
2️⃣ CNI allocates an IP from the Pod CIDR range
3️⃣ Creates a virtual ethernet pair (veth)
4️⃣ Connects Pod to node’s network bridge (e.g., cni0)
5️⃣ Adds routes so Pod can reach others

```
[Pod] ↔ veth ↔ [cni0 bridge] ↔ [Node Network] ↔ [Other Nodes]
```

---

# 🔗 **4. Node-to-Node Networking**

Nodes communicate using:

* Routing tables
* Overlay networks
* VXLAN / IP-in-IP tunnels (Flannel, Calico)

Node A Pod → Node B Pod
traffic is routed using CNI routing rules.

---

# 🛰 **5. Pod-to-Pod Communication**

### Same Node:

Uses local veth + bridge
Very fast.

### Different Nodes:

Uses CNI routing
Possibly tunnels (IP-in-IP or VXLAN)

---

# 🎯 **6. Services — The Real Heart of Networking**

Pods have dynamic IPs.
Services give a **stable virtual endpoint**.

Types:

1. **ClusterIP**
2. **NodePort**
3. **LoadBalancer**
4. **Headless Service**

---

# 🟦 **ClusterIP** (default)

Used **inside** the cluster.

Example YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

Pod IPs change → Service IP stays constant.

**ClusterIP is not accessible outside the cluster.**

---

# 🟧 **NodePort**

Expose Service on each node using a port (30000–32767):

```
NodeIP:NodePort  → forwards to → Pod(s)
```

Used for:

* Testing
* On-prem
* Simple access

---

# 🟩 **LoadBalancer**

Cloud providers only:

AWS ELB, Azure LB, GCP LB.

```
Internet → Load Balancer → NodePort → Pod
```

---

# 🟣 **Headless Service**

Used for:

* Stateful apps
* Databases
* Direct Pod DNS

Provides **Pod IP** instead of ClusterIP.

```yaml
clusterIP: None
```

---

# 🔁 **7. kube-proxy — How Service Networking Actually Works**

kube-proxy watches:

* Services
* Endpoints (list of Pod IPs)

kube-proxy programs iptables/ipvs rules.

### Two routing modes:

1. **iptables mode** (default)
2. **IPVS mode** (faster)

### How it works:

When you curl:

```
curl <ClusterIP>:80
```

iptables rewrites the packet:

```
ClusterIP:80 → PodIP:8080
```

Load balancing happens at the node level.

---

# 🌍 **8. kube-dns / CoreDNS — DNS in Kubernetes**

DNS names for:

### **Services**

```
<service>.<namespace>.svc.cluster.local
```

Example:

```
myapp-svc.default.svc.cluster.local
```

### **Pods**

```
<pod-ip.format>.pod.<namespace>.cluster.local
```

---

# 🚧 **9. Network Policies (firewalls for pods)**

By default **everything can talk to everything**.

NetworkPolicies restrict:

* Which pods can talk to which
* Which namespaces can talk
* Ingress and Egress rules

Example policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

Only backend pods can talk to nginx.

---

# 🌐 **10. Ingress (Layer 7 Routing)**

Used for HTTP routing.

Ingress Controller required:

* NGINX Ingress
* Traefik
* Istio
* HAProxy

Allows:

* Path routing `/api`, `/admin`
* Host-based routing `app.example.com`
* TLS termination

Ingress examples later if you want.

---

# 🔍 **11. Deep-Dive Diagram — Complete Networking Flow**

```
Client (browser)
   ↓
Cloud Load Balancer (LoadBalancer)
   ↓
Node: NodePort
   ↓
kube-proxy rules (iptables/ipvs)
   ↓
ClusterIP service
   ↓
Pod IP (Pod selected by label selector)
   ↓
ContainerPort (8080)
   ↓
Container running in Pod
```

---

# 🧪 **12. Useful kubectl Networking Commands**

### Get Services:

```
kubectl get svc
```

### Describe Service:

```
kubectl describe svc myapp-svc
```

### Resolve DNS:

```
kubectl exec -it <pod> -- nslookup myapp-svc
```

### Debug networking inside Pod:

```
kubectl exec -it <pod> -- curl myapp-svc
kubectl exec -it <pod> -- ping google.com
kubectl exec -it <pod> -- netstat -ntlp
```

### Get endpoints:

```
kubectl get endpoints myapp-svc
```

---

# 🎯 Summary: You Now Know Kubernetes Networking Internals

You understand:

✔ CNI (IP assignment + routing)
✔ Pod-to-Pod communication
✔ Service types
✔ kube-proxy internals
✔ NodePort and LoadBalancer
✔ DNS
✔ Network Policies
✔ How pods actually reach each other

This is **production-grade networking knowledge**.

---

