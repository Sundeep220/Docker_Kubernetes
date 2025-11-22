# Docker Engine — deep dive (architecture, components, and how containerization actually works)

Nice — let’s unpack the Docker Engine layer-by-layer. I’ll explain the **components**, the **control flow** when you run containers, and the **OS primitives** that make containerization possible. After this, we can move to Storage and Networks (one at a time) exactly as you asked.

---

# 1. High-level picture

`docker` (CLI) ⇄ **Docker Engine / daemon (dockerd)** ⇄ Container runtime & kernel (containerd → runc → kernel)

* **Docker CLI / client**: `docker ...` (talks to daemon via REST API / unix socket)
* **Docker daemon (dockerd)**: orchestrates images, containers, networks, volumes — exposes the Engine API
* **containerd**: long-running service that manages images, snapshots, and container lifecycle (spawn/stop). Implements high-level container primitives.
* **runc** (an OCI runtime): creates and runs the container process (it uses Linux kernel primitives).
* **Kernel**: Linux namespaces, cgroups, SELinux/AppArmor, seccomp, overlayfs, network stack — provides isolation and resource control.

Think of Docker Engine as a coordinator: it translates your high-level commands into image management, snapshotting, and kernel-level process isolation through containerd/runc.

---

# 2. Main components of Docker Engine

## 2.1 `dockerd` (Docker daemon)

* Central server process (usually systemd service).
* Responsibilities:

  * Expose REST API (`/var/run/docker.sock`).
  * Manage images (pull/push, cache).
  * Manage containers (create/start/stop/remove).
  * Manage networks and volumes.
  * Talk to `containerd` and storage/network plugins.
* Plugins and drivers are often loaded here.

Inspect it:

```sh
systemctl status docker   # on systemd systems
docker info               # shows engine, storage driver, containerd version, etc.
```

## 2.2 Docker CLI (`docker`)

* Client that talks to `dockerd` over unix socket or TCP.
* Sends API requests; can run remotely if configured.

## 2.3 containerd

* A CNCF project used by Docker, Kubernetes, etc.
* Handles:

  * Pulling images
  * Managing local image store and layers/blobs
  * Snapshotters (provide filesystem layers)
  * Container lifecycle (create, start, exec)
  * Container metrics & events
* `dockerd` delegates to `containerd` rather than doing all tasks itself (separation of concerns).

You’ll see containerd running as a process: `ps aux | grep containerd`.

## 2.4 runc (OCI runtime)

* Low-level runtime that creates namespaces & launches the container process.
* Implements the OCI runtime spec.
* `containerd` calls `runc` (or other runtimes like `crun`) to actually exec the container process.

## 2.5 Storage / graphdrivers (overlay2, etc.)

(We'll go deep later — but Docker Engine selects a *graphdriver* which reads/writes image layers and provides copy-on-write snapshots to containers.)

## 2.6 Networking (bridge, macvlan, ipvlan, host)

(Docker Engine creates and manages networks and DNS for services; more later in Networks section.)

## 2.7 API & plugins

* Engine API is HTTP/JSON — used by CLI, Compose, Kubernetes CRI shims.
* Plugins: volume drivers, network drivers, authorization plugins, log drivers.

---

# 3. Container lifecycle under the hood (what happens when you `docker run`)

1. **CLI sends request** to daemon (`POST /containers/create`, then `POST /containers/<id>/start`).
2. **Image resolution**:

   * Daemon checks local image store for the image manifest and layers; if missing, `containerd` pulls blobs from the registry.
3. **Snapshot & filesystem preparation**:

   * Docker creates a read-only stack of image layers (lowerdirs) and a writable layer (upperdir) for the container using a snapshotter (overlay2 usually).
4. **Create container( metadata )**:

   * Config structure with env, mounts, network settings, uid/gid, capabilities, seccomp profile, cgroup settings, entrypoint/CMD.
5. **Network namespace**:

   * Engine creates network namespace or attaches container to an existing network; veth pair + bridge for default networks.
6. **cgroups & resource limits**:

   * Engine applies cgroup settings (CPU, memory, blkio limits) to the container.
7. **runc invoked**:

   * `containerd` calls `runc create` then `runc start`, which:

     * creates namespaces (`CLONE_NEWNS`, `CLONE_NEWPID`, `CLONE_NEWNET`, `CLONE_NEWUTS`, `CLONE_NEWIPC`, `CLONE_NEWUSER` if used)
     * mounts proc/sysfs and stitched overlay filesystem
     * performs `pivot_root` or `chroot` to make rootfs
     * applies seccomp/apparmor/SELinux contexts
     * forks/execs the user process (PID 1 inside the container)
8. **Container running**:

   * runc’s child process is the container’s init (PID 1 in namespace). Dockerd/containerd track it and attach logs.
9. **Stop & cleanup**:

   * `docker stop` sends SIGTERM/SIGKILL to PID 1; runc exits; daemon removes network endpoints and the writable layer (unless preserved).

---

# 4. Linux primitives that make containerization possible

## 4.1 Namespaces (isolation)

Each of these namespaces gives a separate view to the process:

* **PID** — process space (PID 1 inside container)
* **NET** — network interfaces, routing tables
* **MNT** — separate mount table (mount/proc)
* **UTS** — hostname and domainname
* **IPC** — System V IPC, POSIX message queues
* **USER** — user and group IDs mapping (user namespaces)
* **CGROUP** — control group namespace (in some kernels)

Namespaces isolate resources without full VMs.

## 4.2 cgroups (resource control)

* Limit CPU, memory, blkio, pids, devices
* Provide accounting and limits so a runaway container cannot starve host

## 4.3 union filesystems & copy-on-write

* overlayfs / aufs / btrfs / zfs provide unioned view of stacked read-only layers + writable layer
* Only modified files are copied into writable layer (copy-on-write) → efficiency

## 4.4 chroot / pivot_root

* `pivot_root` switches process root to container's rootfs
* Combined with mount namespace to limit view of filesystem

## 4.5 seccomp / capabilities / LSMs

* **seccomp** filters syscalls to harden containers
* **capabilities** allow dropping Linux capabilities (e.g., CAP_SYS_ADMIN)
* **AppArmor/SELinux** apply mandatory access controls

---

# 5. Images vs Containers (quick conceptual distinction)

* **Image**: immutable set of read-only layers + metadata (manifest, config). Stored as blobs/digests and referenced by tag/digest.
* **Container**: runtime instance of image + thin writable layer + namespaces & cgroups — a live process on the host.

---

# 6. Important runtime flow commands & places to inspect

* `docker info` — shows engine info, storage driver, containerd version.
* `ps aux | grep dockerd` / `containerd` / `runc` — see processes.
* `docker inspect <container>` — see config and mount points.
* Look at socket: `/var/run/docker.sock` — the daemon API entrypoint.
* Engine logs:

  * `journalctl -u docker` (systemd)
  * docker daemon log files on system

---

# 7. Common advanced pieces and how they fit

* **CRI integration**: Kubernetes uses container runtimes via CRI — containerd has CRI shim.
* **Alternative runtimes**: `crun` (lighter), `kata-containers` (VM-based isolation), `gVisor` (user-space kernel).
* **Daemonless tools**: podman aims to be daemonless and rootless; still uses same kernel primitives.
* **Rootless mode**: containers run without root by using user namespaces and other tricks (good security practice).

---

# 8. Performance & security considerations

* CPU/Memory isolation via cgroups ensures better multi-tenant behavior.
* Overlayfs is efficient for startup and disk usage but has limits (e.g., metadata-heavy workloads).
* Running long-running PID 1 processes: choose a proper init (signal handling) or use `tini`/`dumb-init`.
* Use seccomp, capabilities, read-only rootfs, and minimal base images to reduce attack surface.

---

# 9. Visual summary (text diagram)

```
[ docker CLI ]  --HTTP-->  [ dockerd ]
                            |
                            +--> containerd  -- snapshotter --> storage driver (overlay2)
                            |                  |
                            |                  +--> stores blobs under /var/lib/docker or containerd store
                            |
                            +--> networking (bridge, iptables rules, libnetwork)
                            |
                            +--> volumes, plugins, API
                            |
                            +--> invokes runc/crun --> kernel namespaces + cgroups --> process in container
```

--- 