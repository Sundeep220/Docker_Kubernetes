# Container Internals — What a Container Actually Is

> **Prerequisites**: [6_Docker_Engine.md](./6_Docker_Engine.md)  
> **Next**: [11_Multi_Stage_Builds.md](./11_Multi_Stage_Builds.md)

You already know Docker runs containers. But what **is** a container at the OS level? This doc goes deep into the three Linux primitives that make containers possible: **Namespaces**, **Cgroups**, and **UnionFS**.

---

## The Big Misconception: Containers Are NOT Lightweight VMs

A VM runs a full guest OS with its own kernel. A container is just a **regular Linux process** with three types of isolation applied.

```mermaid
graph TB
    subgraph VM["Virtual Machine"]
        direction TB
        GuestOS["Guest OS (full kernel)"]
        AppVM["Application"]
        LibsVM["Libraries"]
        GuestOS --> AppVM
        GuestOS --> LibsVM
    end

    subgraph Container["Container"]
        direction TB
        AppC["Application"]
        LibsC["Libraries"]
    end

    subgraph Host["Host Machine"]
        HostKernel["Host Linux Kernel"]
    end

    VM --> Hypervisor["Hypervisor (VMware, KVM)"]
    Hypervisor --> HostKernel
    Container --> HostKernel

    style VM fill:#f9d0d0,stroke:#c00
    style Container fill:#d0f0d0,stroke:#0a0
    style Host fill:#d0d0f9,stroke:#00c
```

**Key Insight**: Containers share the host kernel. VMs don't. This is why containers start in milliseconds and VMs take minutes.

---

## The Three Pillars of Container Isolation

```mermaid
graph LR
    subgraph Container_Isolation["What Makes a Container"]
        N["Namespaces<br/>(what you can SEE)"]
        C["Cgroups<br/>(what you can USE)"]
        U["UnionFS<br/>(what you see ON DISK)"]
    end

    N -->|Isolation| Walls["Walls between processes"]
    C -->|Resource Limits| Quotas["CPU, Memory, IO limits"]
    U -->|Filesystem| Layers["Layered, copy-on-write FS"]

    style N fill:#FFE0B2
    style C fill:#B2DFDB
    style U fill:#E1BEE7
```

---

# 1. Linux Namespaces — Isolation (What You Can See)

Namespaces give each container its own **isolated view** of system resources. The container thinks it's the only process on the machine.

## The 7 Namespace Types

| Namespace | Flag | What It Isolates | Container Effect |
|-----------|------|------------------|------------------|
| **PID** | `CLONE_NEWPID` | Process IDs | Container sees its process as PID 1 |
| **NET** | `CLONE_NEWNET` | Network stack | Own IP, ports, routing table |
| **MNT** | `CLONE_NEWNS` | Filesystem mounts | Own root filesystem |
| **UTS** | `CLONE_NEWUTS` | Hostname | Own hostname |
| **IPC** | `CLONE_NEWIPC` | Inter-process communication | Isolated shared memory, semaphores |
| **USER** | `CLONE_NEWUSER` | User/Group IDs | Root inside container ≠ root on host |
| **CGROUP** | `CLONE_NEWCGROUP` | Cgroup root | Own cgroup hierarchy view |

### How Namespaces Work — Visual

```mermaid
graph TB
    subgraph Host["Host OS"]
        HP1["PID 1: systemd"]
        HP2["PID 435: dockerd"]
        HP3["PID 890: nginx (container A)"]
        HP4["PID 891: node (container B)"]
    end

    subgraph ContainerA["Container A (PID Namespace)"]
        CA1["PID 1: nginx"]
        CA2["PID 2: nginx worker"]
    end

    subgraph ContainerB["Container B (PID Namespace)"]
        CB1["PID 1: node"]
        CB2["PID 2: node worker"]
    end

    HP3 -.->|"same process,<br/>different view"| CA1
    HP4 -.->|"same process,<br/>different view"| CB1

    style Host fill:#E3F2FD
    style ContainerA fill:#E8F5E9
    style ContainerB fill:#FFF3E0
```

**PID 890 on the host IS PID 1 inside Container A** — it's the same process, just viewed through different namespace lenses.

### PID Namespace Deep Dive

- Container processes see themselves starting from PID 1
- PID 1 inside the container is the **init process** — if it dies, the container dies
- This is why signal handling matters (exec form vs shell form in Dockerfile)
- The host can see ALL container processes, but containers can't see each other

### NET Namespace Deep Dive

Each container gets its own:
- Network interfaces (`eth0`)
- IP address
- Routing table
- iptables rules
- Port space (two containers can both listen on port 80)

```mermaid
graph TB
    subgraph Host_Net["Host Network Namespace"]
        eth0_host["eth0: 192.168.1.10"]
        docker0["docker0 bridge: 172.17.0.1"]
        veth1["vethABC"]
        veth2["vethDEF"]
        
        eth0_host --- docker0
        docker0 --- veth1
        docker0 --- veth2
    end

    subgraph ContA_Net["Container A Network Namespace"]
        eth0_a["eth0: 172.17.0.2"]
    end

    subgraph ContB_Net["Container B Network Namespace"]
        eth0_b["eth0: 172.17.0.3"]
    end

    veth1 ---|"veth pair"| eth0_a
    veth2 ---|"veth pair"| eth0_b

    style Host_Net fill:#E3F2FD
    style ContA_Net fill:#E8F5E9
    style ContB_Net fill:#FFF3E0
```

### MNT Namespace

- Each container sees its own filesystem root (via `pivot_root`)
- The container's root (`/`) is actually an overlay mount point on the host
- Container can't see host files unless you bind-mount them

### USER Namespace (Rootless Containers)

- Maps UID 0 (root) inside container to a non-root UID on the host
- This is how **rootless Docker** works
- Even if an attacker breaks out of the container, they're not root on the host

## Hands-on: Inspecting Namespaces

```bash
# Get container's PID on the host
docker inspect --format '{{.State.Pid}}' <container>

# List all namespaces for that process
ls -la /proc/<PID>/ns/
# Output: cgroup, ipc, mnt, net, pid, pid_for_children, user, uts

# Enter a container's network namespace manually
nsenter -t <PID> -n ip addr

# Compare: host sees all processes
ps aux | grep nginx
# Container sees only its own
docker exec <container> ps aux
```

---

# 2. Cgroups — Resource Limits (What You Can Use)

Cgroups (Control Groups) **limit, account for, and isolate** resource usage. Without cgroups, one container could consume ALL host CPU/memory and starve everything else.

## What Cgroups Control

```mermaid
graph TB
    subgraph Cgroup_Controllers["Cgroup Controllers"]
        CPU["cpu<br/>CPU time allocation"]
        MEM["memory<br/>RAM limit + OOM kill"]
        IO["blkio<br/>Disk I/O bandwidth"]
        PIDS["pids<br/>Max number of processes"]
        NET_CLS["net_cls<br/>Network traffic class"]
    end

    subgraph Container["Container Limits"]
        C_CPU["--cpus=0.5<br/>(50% of one core)"]
        C_MEM["--memory=512m<br/>(512 MB max)"]
        C_PIDS["--pids-limit=100<br/>(max 100 processes)"]
    end

    CPU --> C_CPU
    MEM --> C_MEM
    PIDS --> C_PIDS

    style Cgroup_Controllers fill:#B2DFDB
    style Container fill:#E0F7FA
```

## CPU Limits vs Memory Limits — Critical Difference

| Resource | What Happens When Limit Hit | Effect |
|----------|---------------------------|--------|
| **CPU** | Process is **throttled** | Slows down, but keeps running |
| **Memory** | Process is **OOM-killed** | Container crashes and restarts |

This is one of the **most important things to understand** for production:
- CPU limit exceeded → your app gets slower (throttled), but stays alive
- Memory limit exceeded → Linux OOM killer terminates your process immediately

## How Docker Maps to Cgroups

```bash
# Run container with limits
docker run -d --name limited \
  --cpus="0.5" \
  --memory="256m" \
  --pids-limit=50 \
  nginx

# See the cgroup files created (cgroups v2)
# On the host:
cat /sys/fs/cgroup/docker/<container-id>/cpu.max
# Output: 50000 100000  (50% of one CPU)

cat /sys/fs/cgroup/docker/<container-id>/memory.max
# Output: 268435456  (256 MB in bytes)

# Monitor real-time resource usage
docker stats limited
```

## Cgroups v1 vs v2

| Feature | Cgroups v1 | Cgroups v2 |
|---------|-----------|-----------|
| Structure | Multiple hierarchies | Single unified hierarchy |
| Memory + CPU | Separate controllers | Combined, easier to manage |
| OOM handling | Basic | Improved (PSI support) |
| Used by | Older Docker/K8s | Modern Docker/K8s (default now) |

---

# 3. UnionFS / OverlayFS — Layered Filesystem (What You See on Disk)

You covered this in [2_Docker_images.md](./2_Docker_images.md) and [7_Docker_Storage.md](./7_Docker_Storage.md). Here's the visual mental model connecting it to container internals.

## How OverlayFS Merges Layers

```mermaid
graph TB
    subgraph Final_View["What Container Sees (merged view)"]
        MV["/bin /usr /lib /app /etc"]
    end

    subgraph Writable["Writable Layer (upperdir)"]
        WR["Modified files<br/>New files<br/>Whiteout files (deletions)"]
    end

    subgraph Layer3["Image Layer 3 (COPY . .)"]
        L3["/app/server.js<br/>/app/package.json"]
    end

    subgraph Layer2["Image Layer 2 (RUN npm install)"]
        L2["/app/node_modules/"]
    end

    subgraph Layer1["Image Layer 1 (FROM node:18-alpine)"]
        L1["/bin /usr /lib /etc"]
    end

    Layer1 --> Layer2 --> Layer3 --> Writable --> Final_View

    style Writable fill:#FFCDD2,stroke:#c00
    style Layer3 fill:#C8E6C9
    style Layer2 fill:#C8E6C9
    style Layer1 fill:#C8E6C9
    style Final_View fill:#E3F2FD
```

## Copy-on-Write in Action

```mermaid
sequenceDiagram
    participant App as Container App
    participant Upper as Writable Layer
    participant Lower as Image Layers (read-only)

    App->>Lower: Read /etc/nginx.conf
    Lower-->>App: Return file (fast, no copy)

    App->>Upper: Write /etc/nginx.conf (modified)
    Note over Upper,Lower: File copied from Lower to Upper<br/>(Copy-on-Write)
    Upper-->>App: Future reads come from Upper

    App->>Upper: Create /app/new-file.txt
    Note over Upper: New file only exists in Upper

    App->>Upper: Delete /usr/old-file.txt
    Note over Upper: Whiteout file created:<br/>.wh.old-file.txt<br/>(hides file from Lower)
```

## Why This Matters for Performance

- **Read-heavy workloads**: Fast — reads go directly to image layers (shared, cached)
- **Write-heavy workloads**: Slow — every first write to an existing file triggers a full copy
- **Databases**: MUST use volumes, not the writable layer (CoW is terrible for random writes)

---

# 4. Putting It All Together — Container Creation Flow

```mermaid
sequenceDiagram
    participant User as docker run
    participant Daemon as dockerd
    participant Ctrd as containerd
    participant Runc as runc
    participant Kernel as Linux Kernel

    User->>Daemon: POST /containers/create
    Daemon->>Ctrd: Create container

    Note over Ctrd: Prepare filesystem
    Ctrd->>Kernel: Mount OverlayFS<br/>(image layers + writable layer)

    Note over Ctrd: Create sandbox
    Ctrd->>Runc: Create container process

    Runc->>Kernel: clone() with namespace flags<br/>(CLONE_NEWPID | CLONE_NEWNET |<br/>CLONE_NEWNS | CLONE_NEWUTS |<br/>CLONE_NEWIPC)

    Runc->>Kernel: Set cgroup limits<br/>(CPU, memory, PIDs)

    Runc->>Kernel: pivot_root to container rootfs
    Runc->>Kernel: Apply seccomp filters
    Runc->>Kernel: Drop capabilities
    Runc->>Kernel: exec() the ENTRYPOINT

    Note over Kernel: Container process is now<br/>PID 1 in its own namespace

    Kernel-->>Daemon: Container running
    Daemon-->>User: Container ID
```

## What Each Component Does

```mermaid
graph LR
    CLI["docker CLI<br/>(sends API calls)"] -->|REST API| Daemon["dockerd<br/>(orchestrates)"]
    Daemon -->|gRPC| Ctrd["containerd<br/>(manages lifecycle)"]
    Ctrd -->|exec| Runc["runc<br/>(creates namespaces)"]
    Runc -->|syscalls| Kernel["Linux Kernel<br/>(actual isolation)"]

    style CLI fill:#E3F2FD
    style Daemon fill:#FFF3E0
    style Ctrd fill:#E8F5E9
    style Runc fill:#FCE4EC
    style Kernel fill:#F3E5F5
```

| Component | Role | Analogy |
|-----------|------|---------|
| `docker` CLI | Send commands | You calling a restaurant |
| `dockerd` | Coordinate everything | Restaurant manager |
| `containerd` | Manage container lifecycle | Kitchen manager |
| `runc` | Create the isolated process | Chef who actually cooks |
| Linux Kernel | Provide isolation primitives | Kitchen equipment |

---

# 5. Security Primitives (Beyond Namespaces & Cgroups)

Containers use additional kernel features for defense-in-depth:

### Seccomp (Secure Computing)

Filters which **system calls** a container can make. Docker's default seccomp profile blocks ~44 dangerous syscalls (like `reboot`, `mount`, `kexec_load`).

### Linux Capabilities

Instead of giving containers full root power, Docker drops most capabilities. A container by default can NOT:
- Mount filesystems (`CAP_SYS_ADMIN`)
- Change system time (`CAP_SYS_TIME`)
- Load kernel modules (`CAP_SYS_MODULE`)

```bash
# See capabilities of a container process
docker exec <container> cat /proc/1/status | grep Cap

# Run with all capabilities dropped, add only what you need
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

### AppArmor / SELinux

Mandatory Access Control — even if a process has the right UID, these policies can deny access based on profiles/labels.

---

# 6. Container vs VM — Complete Comparison

```mermaid
graph TB
    subgraph VM_Stack["VM Architecture"]
        direction TB
        VMApp1["App 1"] --> GOS1["Guest OS"]
        VMApp2["App 2"] --> GOS2["Guest OS"]
        GOS1 --> HV["Hypervisor"]
        GOS2 --> HV
        HV --> HW1["Hardware"]
    end

    subgraph Container_Stack["Container Architecture"]
        direction TB
        CApp1["App 1"] --> CR["Container Runtime"]
        CApp2["App 2"] --> CR
        CR --> HOS["Host OS (shared kernel)"]
        HOS --> HW2["Hardware"]
    end

    style VM_Stack fill:#FFEBEE
    style Container_Stack fill:#E8F5E9
```

| Aspect | Container | VM |
|--------|-----------|-----|
| **Startup time** | Milliseconds | Minutes |
| **Size** | MBs | GBs |
| **Kernel** | Shared with host | Own kernel |
| **Isolation** | Process-level (namespaces) | Hardware-level (hypervisor) |
| **Performance** | Near-native | ~5-10% overhead |
| **Security** | Weaker (shared kernel) | Stronger (separate kernel) |
| **Density** | 100s per host | 10s per host |
| **Use case** | Microservices, CI/CD | Full OS isolation, legacy apps |

---

# Summary

```mermaid
mindmap
  root((Container))
    Namespaces
      PID - process isolation
      NET - network isolation
      MNT - filesystem isolation
      UTS - hostname isolation
      IPC - IPC isolation
      USER - UID mapping
    Cgroups
      CPU throttling
      Memory limits + OOM
      I/O bandwidth
      PID limits
    UnionFS
      Image layers read-only
      Writable layer per container
      Copy-on-Write
      Whiteout files for deletion
    Security
      Seccomp syscall filter
      Capabilities drop
      AppArmor / SELinux
```

**A container is a Linux process with**: isolated view (namespaces) + resource limits (cgroups) + layered filesystem (UnionFS) + security hardening (seccomp/capabilities).

---

> **Next**: [11_Multi_Stage_Builds.md](./11_Multi_Stage_Builds.md) — Building production-grade Docker images
