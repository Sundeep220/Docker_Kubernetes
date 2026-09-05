# Docker Compose — Production Patterns

> **Prerequisites**: [4_DockerCompose.md](./4_DockerCompose.md), [12_Docker_Security.md](./12_Docker_Security.md)  
> **Next**: Kubernetes — [../../../Kubernetes/Notes/9_ConfigMaps_and_Secrets.md](../../Kubernetes/Notes/9_ConfigMaps_and_Secrets.md)

You know Compose basics. This doc covers patterns you need for **production-grade** multi-container applications: real healthchecks, resource limits, secrets, profiles, and environment management.

---

## Compose V2 vs V1

```mermaid
graph LR
    V1["docker-compose (V1)<br/>Python-based<br/>Separate binary<br/>❌ Deprecated"] -->|"Replaced by"| V2["docker compose (V2)<br/>Go plugin<br/>Built into Docker CLI<br/>✅ Use this"]

    style V1 fill:#FFCDD2
    style V2 fill:#C8E6C9
```

Always use `docker compose` (space, not hyphen). The `version:` field in YAML is now **optional and ignored** in V2.

---

# 1. Healthchecks + Dependency Ordering (The Right Way)

### The Problem with `depends_on`

```yaml
# ❌ This only waits for the container to START, not be READY
services:
  api:
    depends_on:
      - db
  db:
    image: postgres
```

The API starts before Postgres is ready to accept connections → crash.

### The Solution: `condition: service_healthy`

```yaml
services:
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

```mermaid
sequenceDiagram
    participant Compose
    participant DB as Postgres
    participant Redis
    participant API

    Compose->>DB: Start container
    Compose->>Redis: Start container

    loop Healthcheck every 5s
        Compose->>DB: pg_isready?
        DB-->>Compose: Not ready yet...
    end

    DB-->>Compose: Ready! ✅

    loop Healthcheck every 5s
        Compose->>Redis: redis-cli ping?
    end

    Redis-->>Compose: PONG ✅

    Note over Compose: Both dependencies healthy
    Compose->>API: NOW start API container
```

### Common Healthcheck Commands

| Service | Healthcheck |
|---------|-------------|
| Postgres | `pg_isready -U postgres` |
| MySQL | `mysqladmin ping -h localhost` |
| Redis | `redis-cli ping` |
| MongoDB | `mongosh --eval "db.runCommand('ping')"` |
| Elasticsearch | `curl -f http://localhost:9200/_cluster/health` |
| HTTP API | `curl -f http://localhost:8080/health` |
| gRPC API | `grpc_health_probe -addr=:50051` |

---

# 2. Resource Limits

Without limits, a single container can consume ALL host CPU/memory. In production, always set limits.

```yaml
services:
  api:
    build: .
    deploy:
      resources:
        limits:
          cpus: "1.0"        # Max 1 CPU core
          memory: 512M       # Max 512 MB RAM
        reservations:
          cpus: "0.25"       # Guaranteed 0.25 CPU
          memory: 128M       # Guaranteed 128 MB RAM
```

```mermaid
graph TB
    subgraph Resources["Resource Management"]
        Reservation["Reservation (guaranteed)<br/>CPU: 0.25 cores<br/>Memory: 128 MB<br/>Always available for this container"]
        Limit["Limit (maximum)<br/>CPU: 1.0 cores<br/>Memory: 512 MB<br/>Cannot exceed this"]
    end

    Reservation -->|"Can burst up to"| Limit
    Limit -->|"CPU exceeded"| Throttle["Throttled (slows down)"]
    Limit -->|"Memory exceeded"| OOM["OOM Killed (container dies)"]

    style Reservation fill:#C8E6C9
    style Limit fill:#FFF9C4
    style Throttle fill:#FFE0B2
    style OOM fill:#FFCDD2
```

---

# 3. Secrets Management

Never put passwords in `environment:` in the compose file committed to git.

### Option 1: `.env` File (Simple, Dev/Staging)

```
# .env (gitignored!)
DB_PASSWORD=supersecret
REDIS_URL=redis://redis:6379
API_KEY=abc123
```

```yaml
services:
  api:
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
      - API_KEY=${API_KEY}
```

### Option 2: Docker Secrets (More Secure)

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt    # This file is gitignored
```

```mermaid
graph LR
    File["secrets/db_password.txt<br/>(on host, gitignored)"] -->|"Mounted as"| Mount["/run/secrets/db_password<br/>(inside container, read-only tmpfs)"]
    Mount -->|"Read by"| App["Application<br/>reads POSTGRES_PASSWORD_FILE"]

    style File fill:#FFF3E0
    style Mount fill:#E8F5E9
    style App fill:#E3F2FD
```

---

# 4. Profiles — Run Subsets of Services

```yaml
services:
  api:
    build: .
    ports: ["8080:3000"]

  db:
    image: postgres:16-alpine

  redis:
    image: redis:7-alpine

  # Debug tools — only run when needed
  adminer:
    image: adminer
    ports: ["8081:8080"]
    profiles: ["debug"]

  redis-commander:
    image: rediscommander/redis-commander
    ports: ["8082:8081"]
    profiles: ["debug"]

  # Monitoring — only in production
  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]
```

```bash
# Normal dev — only api, db, redis start
docker compose up

# With debug tools
docker compose --profile debug up

# With monitoring
docker compose --profile monitoring up

# Everything
docker compose --profile debug --profile monitoring up
```

---

# 5. Multi-Environment Management

```mermaid
graph TB
    Base["docker-compose.yml<br/>(base config)"]
    Dev["docker-compose.override.yml<br/>(auto-loaded for dev)"]
    Prod["docker-compose.prod.yml<br/>(explicit for production)"]
    Test["docker-compose.test.yml<br/>(explicit for testing)"]

    Base --> Dev
    Base --> Prod
    Base --> Test

    style Base fill:#E3F2FD
    style Dev fill:#E8F5E9
    style Prod fill:#FFF3E0
    style Test fill:#F3E5F5
```

### Base: `docker-compose.yml`

```yaml
services:
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      retries: 5
```

### Dev Override: `docker-compose.override.yml` (auto-loaded)

```yaml
services:
  api:
    volumes:
      - .:/app                    # Live reload
    ports:
      - "8080:3000"
      - "9229:9229"               # Debugger port
    environment:
      NODE_ENV: development
    command: ["npm", "run", "dev"]
```

### Production: `docker-compose.prod.yml`

```yaml
services:
  api:
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:
```

```bash
# Dev (auto-loads override)
docker compose up

# Production (explicit)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

# 6. Logging Configuration

```yaml
services:
  api:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"       # Max 10 MB per log file
        max-file: "3"         # Keep 3 rotated files
```

Without this, container logs grow unbounded and can fill your disk.

---

# 7. Complete Production Compose Example

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:3000"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      NODE_ENV: production
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: myapp
      DB_USER: postgres
      DB_PASSWORD_FILE: /run/secrets/db_password
      REDIS_URL: redis://redis:6379
    secrets:
      - db_password
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--maxmemory", "128mb", "--maxmemory-policy", "allkeys-lru"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  pgdata:
```

---

## Summary

| Pattern | What It Solves |
|---------|---------------|
| `condition: service_healthy` | Start order based on readiness, not just startup |
| `deploy.resources.limits` | Prevent runaway containers eating all resources |
| Docker secrets | Avoid plain-text passwords in compose files |
| Profiles | Run debug/monitoring tools only when needed |
| Override files | Same base config, different envs (dev/prod) |
| Logging limits | Prevent disk fill from unbounded logs |
| `read_only` + `tmpfs` | Prevent malware writes |
| `restart: unless-stopped` | Self-healing for production |

---

> **Docker section complete!** You've covered: basics → images → networking → storage → engine → compose → internals → multi-stage → security → production compose.
>
> **Next up**: Kubernetes intermediate topics starting with [ConfigMaps & Secrets](../../Kubernetes/Notes/9_ConfigMaps_and_Secrets.md)
