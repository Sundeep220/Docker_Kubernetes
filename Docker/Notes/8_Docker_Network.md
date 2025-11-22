Awesome — let's now **dive deep into Docker Networking**, one of the most misunderstood but most powerful parts of Docker’s architecture.

We’ll go from basic to kernel-level internals:

---

# 🚀 **1. Docker Networking — High-Level Architecture**

Docker networking is built on top of:

* **Linux networking primitives** (veth pairs, namespaces, bridges, iptables, routing)
* **CNM (Container Network Model)**
* **Libnetwork** (Docker’s networking library)

Every container gets its own:

### ✔ **Network namespace**

Isolated network stack (its own eth0, routes, iptables, conntrack table).

### ✔ **Virtual Ethernet (veth) pair**

One end inside the container, other in the host.

### ✔ **IP address**

Managed by Docker’s built-in DHCP/allocator.

---

# 🎯 **Three main network types in Docker**

### **1️⃣ Bridge (default)**

```
docker0
```

NATed, private network → containers get IPs like:

```
172.17.0.2
```

### **2️⃣ Host**

Container shares the host network namespace.

### **3️⃣ None**

Container gets no network.

### **User-defined bridge**

Best for multi-container apps.

---

# 🔧 **2. What actually happens when you run a container?**

Run:

```
docker run -d nginx
```

Behind the scenes:

### **Step 1 — Create network namespace**

Linux creates:

```
/var/run/netns/<container-id>
```

This namespace has:

* empty `eth0`
* its own routing table
* its own iptables table

---

### **Step 2 — Create veth pair**

Docker creates:

```
vethXYZ -> container
vethABC -> host
```

Example:

```
veth0c12c4 (host)
eth0 (inside container)
```

---

### **Step 3 — Attach host end to docker0 bridge**

```
veth0c12c4 ----> docker0 bridge
```

Bridge (docker0) acts like a virtual switch.

---

### **Step 4 — Assign container IP**

Docker’s IPAM (IP Address Management) allocates:

```
172.17.0.2/16
```

Inside container:

```
eth0: 172.17.0.2
default route: 172.17.0.1 (docker0)
```

---

### **Step 5 — Setup NAT (MASQUERADE)**

Docker configures iptables:

```
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

This lets the container access the internet.

---

### **Step 6 — DNS configuration**

`/etc/resolv.conf` inside container points to Docker’s embedded DNS server.

---

# 🌐 **3. User-defined Bridge Networks**

Created by:

```
docker network create mynet
```

### Benefits:

* Built-in DNS between containers
* Cleaner IP allocation
* No NAT between containers
* Isolation from default networks

Example:

```
docker run --network mynet --name backend backend-image
docker run --network mynet --name frontend frontend-image
```

Now:

```
ping backend
```

works by DNS name.

---

# 🧱 **4. Host Network Mode**

```
docker run --network host nginx
```

Container uses host’s network namespace.

Meaning:

* Same IP as host
* No port mapping needed
* VERY fast (no NAT, no veth overhead)

Used for:

* High-performance networking
* Prometheus node exporter
* HAProxy, NGINX, Kafka, etc.

Downside: security isolation is weaker.

---

# 🏗 **5. None Network Mode**

```
docker run --network none ubuntu
```

Container gets:

* No network interface
* No DNS
* No routes

Used for:

* High-security containers
* Custom networking setups

---

# 📦 **6. Overlay Networks (Docker Swarm only)**

We won’t go deep unless you want, but summary:

Used for **cross-node networking**.

Technologies:

* VXLAN encapsulation
* Distributed control plane

**In Kubernetes, overlay is done by CNI (Calico, Flannel, Cilium)**, not Docker.

---

# 🔥 **7. Port Mapping Internals (VERY IMPORTANT)**

When you run:

```
docker run -p 8080:80 nginx
```

Docker adds iptables rule:

```
iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to 172.17.0.2:80
```

Also NAT for response packets:

```
MASQUERADE
```

This is how port forwarding works.

---

# 🔍 **8. Inspecting Docker Networking Internals**

### List networks

```
docker network ls
```

### Inspect a network

```
docker network inspect bridge
```

### See veth pairs

```
ip link
```

### See routing

```
ip route
```

### See iptables

```
iptables -t nat -L -n
```

---

# 🔬 **9. Docker DNS and Service Discovery**

User-defined bridge networks support:

* DNS-based service discovery
* Container names as hostnames

Example:

```
docker run --network app --name db mysql
docker run --network app --name api myapp
```

Inside `api`:

```
ping db
```

works!

Docker runs a small DNS server embedded in dockerd.

---

# 🧠 Summary — Docker Networking in One Table

| Concept             | Meaning                       |
| ------------------- | ----------------------------- |
| Network namespace   | Isolated network stack        |
| veth pair           | Virtual wire into a container |
| docker0 bridge      | Virtual switch                |
| NAT (iptables)      | External connectivity         |
| Host mode           | No isolation                  |
| None mode           | Full isolation                |
| User-defined bridge | DNS + cleaner networking      |
| Overlay (Swarm)     | Multi-node networking         |
| Port mapping        | DNAT + MASQUERADE             |

---

