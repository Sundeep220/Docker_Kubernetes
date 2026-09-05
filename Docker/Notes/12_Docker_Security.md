# Docker Security — Production Hardening

> **Prerequisites**: [10_Container_Internals.md](./10_Container_Internals.md), [11_Multi_Stage_Builds.md](./11_Multi_Stage_Builds.md)  
> **Next**: [13_Docker_Compose_Production.md](./13_Docker_Compose_Production.md)

Containers are isolated, but NOT fully secure by default. This doc covers the defense-in-depth layers you need for production.

---

## Security Layers Overview

```mermaid
graph TB
    subgraph Layers["Defense in Depth"]
        L1["1. Image Security<br/>Minimal base, scan for CVEs"]
        L2["2. Build Security<br/>No secrets in image, .dockerignore"]
        L3["3. Runtime Security<br/>Non-root, read-only FS, capabilities"]
        L4["4. Network Security<br/>Limit exposure, user-defined networks"]
        L5["5. Secret Management<br/>Docker secrets, env var alternatives"]
    end

    L1 --> L2 --> L3 --> L4 --> L5

    style Layers fill:#FFF3E0
```

---

# 1. Run as Non-Root (Most Important)

By default, containers run as **root**. If an attacker exploits your app, they have root inside the container — which can be escalated to host root.

### Bad (default)

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
CMD ["node", "server.js"]
# Runs as root! ❌
```

### Good

```dockerfile
FROM node:18-alpine

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser

WORKDIR /app
COPY --chown=appuser:appgroup . .

USER appuser
CMD ["node", "server.js"]
# Runs as UID 1001 ✅
```

```mermaid
graph LR
    subgraph Default["Default: Root"]
        Root["Container runs as root<br/>UID 0"]
        Risk["Attacker gets root<br/>Can modify system files<br/>Can escalate to host"]
        Root --> Risk
    end

    subgraph Hardened["Hardened: Non-Root"]
        NonRoot["Container runs as appuser<br/>UID 1001"]
        Safe["Attacker limited<br/>Cannot modify system files<br/>Cannot install packages"]
        NonRoot --> Safe
    end

    style Default fill:#FFCDD2
    style Hardened fill:#C8E6C9
```

---

# 2. Read-Only Root Filesystem

Prevent attackers from writing malware or modifying config:

```bash
docker run --read-only --tmpfs /tmp --tmpfs /var/run nginx
```

In Compose:

```yaml
services:
  web:
    image: nginx
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
      - /var/cache/nginx
```

The container can't write anywhere except the tmpfs mounts.

---

# 3. Drop Linux Capabilities

Docker gives containers a subset of root capabilities. You should drop all and add only what's needed:

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx
```

### Common Capabilities

| Capability | What It Allows | Keep? |
|-----------|----------------|-------|
| `NET_BIND_SERVICE` | Bind to ports < 1024 | ✅ Usually needed |
| `CHOWN` | Change file ownership | ❌ Drop if not needed |
| `DAC_OVERRIDE` | Bypass file permission checks | ❌ Drop |
| `SYS_ADMIN` | Mount, namespace ops | ❌ Almost never needed |
| `NET_RAW` | Raw sockets, ping | ❌ Drop in production |
| `SETUID/SETGID` | Change user/group | ❌ Drop if using fixed user |

---

# 4. Image Scanning

```mermaid
graph LR
    Build["Build Image"] --> Scan["Scan for CVEs"]
    Scan -->|"Critical CVEs"| Block["Block Deploy ❌"]
    Scan -->|"No Critical CVEs"| Push["Push to Registry ✅"]
    Push --> Deploy["Deploy to K8s"]

    style Block fill:#FFCDD2
    style Push fill:#C8E6C9
```

### Tools

```bash
# Docker Scout (built-in)
docker scout cves myapp:latest

# Trivy (most popular, free)
trivy image myapp:latest

# Snyk
snyk container test myapp:latest
```

### In CI/CD Pipeline (GitHub Actions example)

```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # Fail the build if critical CVEs found
```

---

# 5. No Secrets in Images

```mermaid
graph TB
    subgraph Bad["❌ Secrets in Image"]
        B1["ENV DB_PASSWORD=s3cret<br/>in Dockerfile"]
        B2["Anyone with image access<br/>can see the password"]
        B3["Password baked into<br/>every layer forever"]
        B1 --> B2 --> B3
    end

    subgraph Good["✅ Secrets at Runtime"]
        G1["Pass via environment<br/>at container start"]
        G2["Use Docker secrets<br/>or mounted files"]
        G3["Secret never in image<br/>never in layer history"]
        G1 --> G2 --> G3
    end

    style Bad fill:#FFCDD2
    style Good fill:#C8E6C9
```

### Bad Patterns

```dockerfile
# ❌ Secret in ENV
ENV DATABASE_URL=postgres://user:password@host:5432/db

# ❌ Secret in ARG (visible in docker history)
ARG API_KEY
RUN curl -H "Auth: $API_KEY" https://api.example.com/setup

# ❌ COPY-ing .env file
COPY .env /app/.env
```

### Good Patterns

```bash
# ✅ Pass at runtime
docker run -e DATABASE_URL="postgres://..." myapp

# ✅ Use .env file at runtime (not in image)
docker run --env-file .env myapp

# ✅ Docker secrets (Swarm / Compose)
# See section below
```

### Build-Time Secrets (Docker BuildKit)

If you need a secret DURING build (e.g., npm token for private registry):

```dockerfile
# syntax=docker/dockerfile:1
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
COPY . .
CMD ["node", "server.js"]
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

The secret is mounted during that RUN step only — it's NEVER stored in any image layer.

---

# 6. Docker Secrets (Compose)

```yaml
services:
  api:
    image: myapp
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

The secret file is mounted at `/run/secrets/db_password` inside the container as a read-only tmpfs mount. It never appears in any image layer.

---

# 7. Seccomp Profiles

Docker applies a default seccomp profile that blocks ~44 dangerous syscalls. You can make it stricter:

```bash
# See default profile
docker run --rm -it --security-opt seccomp=unconfined alpine  # NOT recommended

# Apply custom profile
docker run --security-opt seccomp=./custom-seccomp.json myapp
```

---

# 8. Security Checklist

```mermaid
graph TB
    subgraph Checklist["Production Security Checklist"]
        C1["✅ Non-root USER in Dockerfile"]
        C2["✅ Minimal base image (alpine/distroless)"]
        C3["✅ .dockerignore excludes .env, .git"]
        C4["✅ No secrets in Dockerfile or image layers"]
        C5["✅ Read-only root filesystem"]
        C6["✅ Drop all capabilities, add only needed"]
        C7["✅ Image scanning in CI/CD"]
        C8["✅ Pin image versions (no :latest)"]
        C9["✅ Use Docker secrets for sensitive data"]
        C10["✅ Separate build and runtime stages"]
    end

    style Checklist fill:#E8F5E9
```

### Quick Comparison

| Practice | Without | With |
|----------|---------|------|
| Non-root | Attacker gets root | Attacker is limited user |
| Read-only FS | Malware can be written | Writes blocked |
| Cap drop | 14 capabilities available | Only what's needed |
| Image scan | Unknown CVEs in production | CVEs caught before deploy |
| No secrets in image | Leaked via `docker history` | Secrets only at runtime |
| Pinned versions | `latest` can change unexpectedly | Reproducible builds |

---

> **Next**: [13_Docker_Compose_Production.md](./13_Docker_Compose_Production.md) — Production-grade Compose patterns
