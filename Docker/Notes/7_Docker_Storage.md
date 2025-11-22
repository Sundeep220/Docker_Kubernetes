Perfect — Docker **Storage** is one of the MOST important parts to understand if you want to master Docker performance, image building, caching, debugging, and Kubernetes container behavior.

We’ll go deep into:

1. Storage Architecture
2. Image Storage (layers, manifests, digests)
3. Container Storage (writable layer, snapshotters)
4. OverlayFS / overlay2 (deep internals)
5. Volume types & lifecycle
6. `/var/lib/docker` directory layout
7. Copy-on-write behavior & performance implications
8. Advanced snapshotters (containerd, btrfs, zfs)

---

# 🚀 1. High-Level Docker Storage Architecture

Docker storage has TWO big worlds:

### **(A) Image Storage (Read-only layers)**

Managed by **containerd** + **snapshotter**
Used to compose images.

### **(B) Container Storage (Writable layer)**

Union-mount on top of image layers (copy-on-write).

### **(C) Volumes / Bind mounts (Persistent Storage)**

Used for databases, logs, data that must survive container deletion.

---

# 📦 2. Docker Images — How They Are Stored

Every Docker image consists of:

1. **Image layers** (rootfs layers) — tar files
2. **Manifest** — JSON that lists layers + config
3. **Image config (JSON)** — ENV, ENTRYPOINT, USER, etc.
4. **Content-addressable storage** (layers stored by digest SHA256, not tag)

### Key idea:

The *tag* points to a manifest, the manifest points to *layer digests*.

Example:

```
ubuntu:latest -> manifest -> [
    layer1 sha256:a12f...,
    layer2 sha256:b98d...,
    ...
]
```

All stored in:

```
/var/lib/docker/
└── overlay2/
└── image/
```

Different images reuse identical layers → huge space efficiency.

---

# 🔥 3. Container Storage — Writable Layer

When you run a container:

```
docker run ubuntu
```

Docker does:

```
container writable layer
    +
read-only image layers
```

This is stacked using **OverlayFS** (overlay2 driver).

Writable layer is:

* Ephemeral
* Stores only changed files
* Cleared when container is deleted (unless you commit it)

---

# 🔬 4. OverlayFS / overlay2 — Deep Internals

This is the core of Docker storage.

OverlayFS merges:

```
lowerdir (image layers read-only)
upperdir (container writes)
workdir (internal metadata workspace)
```

Diagram:

```
     +-------------------------+
     | Writable layer (upper) |
     +-------------------------+
     | Lower layers (image)   |
     +-------------------------+
     | Lower layers (image)   |
     +-------------------------+
     | Lower layers (image)   |
     +-------------------------+
```

### When container writes a file:

* Docker checks if file exists in lowerdir
* If yes → copy it to upperdir (Copy-on-Write)
* Modify only upperdir
* Lower layers remain immutable

### When container deletes a file:

Instead of really deleting it, Docker creates a **whiteout file** named:

```
.wh.<filename>
```

This hides the original file in lower layers.

### Performance impact:

* Copy-on-write is slower for large files
* Metadata-heavy workloads (like unpacking node_modules or compiling) can be slow because overlayfs metadata is expensive.

---

# 🧬 5. What lives in `/var/lib/docker`?

Let's expand this clearly:

```
/var/lib/docker/
├── overlay2/             # main storage for layers (image & container writable layer)
│   ├── <layerid>/       
│   │   ├── diff/         # actual filesystem of this layer
│   │   ├── link          # hardlink
│   │   ├── lower         # which lower dirs exist below
│   │   └── work/         # workdir for overlayfs
│
├── containers/           # metadata + logs for each running container
│   └── <container-id>/
│       ├── config.v2.json
│       └── hostconfig.json
│
├── image/
│   └── overlay2/
│       ├── imagedb/      # manifest & config metadata
│       └── layerdb/      # metadata for each layer
│
├── volumes/              # named volumes
│   └── <volume-name>/data
│
└── plugins/              # storage, network, log plugins
```

This is where **all Docker state** lives.

---

# 📚 6. How Image Layers Are Stored

Each layer is stored in:

```
/var/lib/docker/overlay2/<layerId>/diff
```

These contain files like:

```
/bin/
/usr/
/lib/
/etc/
```

Layers are **content-addressable**:

* The SHA256 digest ensures no duplicates
* Deduplication across images saves space

So ubuntu/nginx/redis might share base Ubuntu layers.

---

# 🧱 7. Volumes — The Real Persistent Storage

Containers are ephemeral.
Volumes survive container deletion.

Three options:

### 1️⃣ Named volumes (preferred)

```
docker volume create mydata
docker run -v mydata:/var/lib/mysql mysql
```

Stored under:

```
/var/lib/docker/volumes/mydata/_data/
```

### 2️⃣ Bind mounts (map host folder)

```
docker run -v /home/user/code:/app
```

Directly ties host paths → no isolation.

### 3️⃣ tmpfs mounts (RAM-only)

```
docker run --tmpfs /cache
```

Fast, lives only in memory.

---

# 🚀 8. containerd snapshotters (advanced concept)

Docker uses containerd internally.

containerd has "snapshotters":

* overlayfs (default)
* btrfs
* zfs
* aufs (old)
* devicemapper (deprecated)
* stargz (lazy loading image)

Snapshotters manage:

* Layer composition
* File diffs
* Performance characteristics

---

# 💡 9. Copy-on-Write Performance Implications

### ✔ Great for:

* Starting containers very quickly
* Small layers
* Reusing layers between images
* Immutable infrastructure

### ❌ Slow for:

* Writing many small files (npm, pip install)
* Heavy I/O like databases (use volumes instead!)
* Renaming or modifying large files (copy before write)

This is why:
**Databases MUST use volumes, not container writable layer.**

---

# 🧪 10. Debugging Docker Storage

### Check storage driver:

```
docker info | grep Storage
```

### Check disk usage:

```
docker system df
```

### Check layers:

```
docker inspect <image>
```

### Clean everything:

```
docker system prune -a --volumes
```

---

# 🏁 Summary — Docker Storage in One Table

| Component       | Purpose                                   |
| --------------- | ----------------------------------------- |
| image layers    | read-only filesystem shared across images |
| writable layer  | temporary container-specific disk         |
| overlay2        | union filesystem implementing CoW         |
| volumes         | persistent storage                        |
| snapshotters    | manage layers and diffs                   |
| /var/lib/docker | the root of all Docker state              |

---
