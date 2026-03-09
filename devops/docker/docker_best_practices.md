# Docker Best Practices (FAANG-Level Engineering)

**Related:** [Docker Fundamentals](docker_fundamentals.md) | [Dockerfiles](dockerfiles.md) | [Docker Networking](docker_networking.md) | [Volumes & Storage](docker_volumes_and_storage.md) | [Docker Compose](docker_compose.md) | [GitHub Actions CI/CD](../github/github_actions_cicd.md)

---

## 1. Image Size Optimization

### Why Image Size Matters

- **Build speed:** Larger images = slower builds, pushes, pulls
- **Deploy speed:** Pulling a 2GB image on 100 nodes = 200GB network transfer
- **Attack surface:** More packages = more CVEs to patch
- **Cold start:** Larger images take longer for the runtime to start (serverless, auto-scaling)

### Base Image Selection

```
┌─────────────────────────────────────────────────────────────────┐
│  Base Image Sizes (approximate)                                 │
│                                                                 │
│  ubuntu:22.04          ~77MB    Full OS, apt, glibc            │
│  debian:bookworm-slim  ~52MB    Stripped Debian, glibc         │
│  alpine:3.19           ~7MB     Musl libc, apk, busybox       │
│  gcr.io/distroless/cc  ~2MB     No shell, no package manager  │
│  scratch               ~0MB     Empty filesystem               │
└─────────────────────────────────────────────────────────────────┘
```

```dockerfile
# For Go (static binary): use scratch
FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]

# For compiled languages needing libc: use distroless
FROM gcr.io/distroless/base-debian12
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]

# For interpreted languages: use alpine or slim variants
FROM python:3.12-slim
# FROM node:20-alpine
```

**Alpine gotcha:** Alpine uses musl libc instead of glibc. Some applications (especially Python C extensions, Java, .NET) may have compatibility issues or performance differences. Test thoroughly.

### Multi-Stage Build for Minimal Images

```dockerfile
# Stage 1: Build everything needed
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production image with only runtime artifacts
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./
USER 1000:1000
CMD ["node", "dist/index.js"]
```

### Layer Reduction Techniques

```dockerfile
# BAD: 4 layers, apt cache baked into layer 1
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN rm -rf /var/lib/apt/lists/*   # too late — already in previous layers

# GOOD: 1 layer, clean up in same RUN
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      wget && \
    rm -rf /var/lib/apt/lists/*
```

---

## 2. Security

### Run as Non-Root (Always)

```dockerfile
# Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Or use numeric UID (more portable, avoids /etc/passwd dependency)
USER 1000:1000

# Set file ownership during COPY
COPY --chown=1000:1000 . /app
```

**Why:** If a container is compromised and runs as root, the attacker has root inside the container. With user namespaces disabled (the default), that maps to root on the host.

```bash
# Verify at runtime
docker run --user 1000:1000 my-image
docker exec my-container whoami   # should NOT be root
```

### Read-Only Filesystem

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=50m \
  --tmpfs /var/run:rw,noexec,nosuid \
  -v app-logs:/var/log/app \
  my-app
```

**What's happening:** `--read-only` makes the overlay2 writable layer read-only. The container cannot create or modify any files except in explicitly mounted writable paths (tmpfs, volumes). An attacker who gains code execution cannot install tools, write scripts, or modify binaries.

### No Secrets in Images

```dockerfile
# TERRIBLE: secret baked into image, visible in docker history
ENV API_KEY=sk-1234567890
COPY .env /app/.env
RUN echo "password123" > /app/secret.txt

# BAD: ARG is better than ENV but still visible in build history
ARG DB_PASSWORD
RUN echo "Connecting to db with $DB_PASSWORD"

# GOOD: BuildKit secrets (never written to any layer)
RUN --mount=type=secret,id=db_password \
    DB_PASSWORD=$(cat /run/secrets/db_password) && \
    ./configure-db.sh "$DB_PASSWORD"

# AT RUNTIME: use Docker secrets or environment variables
docker run -e DB_PASSWORD="$(vault kv get -field=password secret/db)" my-app
```

```bash
# Build with BuildKit secret
docker build --secret id=db_password,src=./db_password.txt .

# Verify no secrets leaked
docker history my-image   # should show nothing sensitive
```

### Security Scanning

```bash
# Trivy — comprehensive vulnerability scanner
trivy image my-app:latest
trivy image --severity HIGH,CRITICAL my-app:latest

# Snyk (integrates with CI/CD)
snyk container test my-app:latest

# Docker Scout (built into Docker Desktop)
docker scout cves my-app:latest
docker scout recommendations my-app:latest

# Scan Dockerfile for misconfigurations
hadolint Dockerfile
```

### Drop Capabilities

```bash
# Drop ALL capabilities, add back only what's needed
docker run -d \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  my-app
```

| Capability | What It Allows |
|---|---|
| `NET_BIND_SERVICE` | Bind to ports below 1024 |
| `CHOWN` | Change file ownership |
| `SETUID/SETGID` | Change process UID/GID |
| `SYS_ADMIN` | Broad system admin (DANGEROUS — almost root) |

**Rule:** `--cap-drop ALL` then whitelist. Never use `--privileged` in production.

---

## 3. Layer Caching Optimization

```dockerfile
# OPTIMAL LAYER ORDER:
# 1. Base image (changes rarely)
FROM node:20-alpine

# 2. System dependencies (changes rarely)
RUN apk add --no-cache curl

# 3. Set up working directory
WORKDIR /app

# 4. Dependency manifest files (changes occasionally)
COPY package.json package-lock.json ./

# 5. Install dependencies (cached until package*.json changes)
RUN npm ci --only=production

# 6. Application source code (changes frequently)
COPY . .

# 7. Build step (depends on source code)
RUN npm run build

# 8. Runtime config (changes rarely)
USER 1000:1000
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

```
Cache hit rates with optimal ordering:
  Layer 1-3: ~99% cache hit (base + system deps)
  Layer 4-5: ~90% cache hit (deps don't change on every commit)
  Layer 6-7: ~0%  cache hit (source code changes on every commit)
  Layer 8:   ~99% cache hit (runtime config rarely changes)
```

### BuildKit Cache Mounts

```dockerfile
# Cache package manager downloads across builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

RUN --mount=type=cache,target=/root/.npm \
    npm ci

RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    go build -o /app/server .
```

**What's happening:** Cache mounts persist across builds. The package manager's download cache survives even if the layer is invalidated. This means `npm ci` only downloads changed packages, not everything.

---

## 4. Health Checks

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# For images without curl, use a built-in mechanism
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD ["./healthcheck"]   # custom binary that checks readiness

# PostgreSQL
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD pg_isready -U postgres || exit 1

# Redis
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD redis-cli ping || exit 1
```

**Why health checks matter:**
- Docker Compose `depends_on: condition: service_healthy` waits for readiness
- Kubernetes liveness/readiness probes prevent traffic to unhealthy pods
- Swarm mode replaces unhealthy containers automatically
- Load balancers use health checks to route traffic

---

## 5. Resource Limits

```bash
# Memory limits
docker run -d \
  --memory=512m \
  --memory-swap=1g \
  --memory-reservation=256m \
  my-app

# CPU limits
docker run -d \
  --cpus=1.5 \
  --cpu-shares=512 \
  my-app
```

**What happens when limits are exceeded:**
- **Memory:** Container is OOM-killed (exit code 137). No warning.
- **CPU:** Container is throttled (not killed). Requests take longer.

**FAANG practice:** Always set memory limits. A memory leak in one container shouldn't take down the host. Monitor with:

```bash
docker stats                    # real-time resource usage
docker stats --no-stream        # snapshot
```

---

## 6. Logging Strategy

```dockerfile
# Application should log to stdout/stderr, NOT to files
# Docker captures stdout/stderr via the logging driver

# BAD: logs written to a file inside the container
CMD ["./app", "--log-file=/var/log/app.log"]

# GOOD: logs to stdout (captured by Docker)
CMD ["./app"]

# If app only logs to files, redirect to stdout:
RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log
```

```bash
# Configure logging driver
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  my-app

# Production: use a centralized logging driver
docker run -d \
  --log-driver=fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="app.{{.Name}}" \
  my-app
```

**What's happening:** Docker's logging driver captures everything the container process writes to stdout/stderr. The `json-file` driver (default) stores logs as JSON in `/var/lib/docker/containers/<id>/<id>-json.log`. Without `max-size`, this file grows unbounded and will fill the disk.

---

## 7. 12-Factor App in Docker

The [12-Factor App](https://12factor.net) methodology maps perfectly to containerized applications:

| Factor | Docker Implementation |
|---|---|
| 1. Codebase | One repo, one Dockerfile, one image |
| 2. Dependencies | All deps in Dockerfile (RUN pip install, npm ci) |
| 3. Config | Environment variables (-e, --env-file), not baked in |
| 4. Backing services | Connect via env vars (DATABASE_URL, REDIS_URL) |
| 5. Build/release/run | Build: docker build. Release: tag + push. Run: docker run |
| 6. Processes | Stateless containers. Data in volumes/external stores |
| 7. Port binding | EXPOSE + -p. App binds to port, Docker maps it |
| 8. Concurrency | Scale by running more containers, not bigger containers |
| 9. Disposability | Fast startup, graceful shutdown (handle SIGTERM) |
| 10. Dev/prod parity | Same image in dev, staging, prod. Config via env vars |
| 11. Logs | stdout/stderr → Docker logging driver |
| 12. Admin processes | docker exec or docker compose run for one-off tasks |

---

## 8. CI/CD Integration

```yaml
# GitHub Actions: build, scan, push
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: 1   # fail build on high/critical CVEs
```

---

## 9. Debugging Containers

```bash
# Shell into a running container
docker exec -it my-container /bin/sh

# Shell into a stopped/crashed container's filesystem
docker commit my-container debug-image
docker run -it debug-image /bin/sh

# Debug a minimal image (no shell)
# Use a debug sidecar container sharing the network/pid namespace
docker run -it --rm \
  --network container:my-container \
  --pid container:my-container \
  nicolaka/netshoot \
  bash

# See container's stdout/stderr
docker logs my-container
docker logs --tail 100 -f my-container

# Inspect container details
docker inspect my-container | jq '.[0].State'
docker inspect my-container | jq '.[0].NetworkSettings'

# Resource usage
docker stats my-container

# Check why container exited
docker inspect --format '{{.State.ExitCode}}' my-container
docker inspect --format '{{.State.OOMKilled}}' my-container

# Filesystem changes (what the container modified)
docker diff my-container
# C /var/log           (changed)
# A /var/log/app.log   (added)
```

### Common Exit Codes

| Exit Code | Meaning |
|---|---|
| 0 | Normal exit |
| 1 | Application error |
| 126 | Command not executable |
| 127 | Command not found |
| 137 | SIGKILL (OOM-killed or docker kill) |
| 139 | SIGSEGV (segmentation fault) |
| 143 | SIGTERM (docker stop graceful shutdown) |

---

## 10. Production Checklist

```
Image:
  [ ] Pin base image versions (node:20.11-alpine, NOT node:latest)
  [ ] Multi-stage build (build stage + runtime stage)
  [ ] Minimal base image (alpine, slim, distroless, scratch)
  [ ] .dockerignore excludes .git, node_modules, .env, etc.
  [ ] No secrets in image (check with docker history)
  [ ] Image scanned for CVEs (trivy, snyk)
  [ ] Image size < 200MB for most applications

Security:
  [ ] Non-root user (USER 1000:1000)
  [ ] Read-only filesystem (--read-only)
  [ ] Capabilities dropped (--cap-drop ALL + whitelist)
  [ ] No --privileged flag
  [ ] no-new-privileges security option
  [ ] Secrets via runtime env vars or Docker secrets, not baked in

Runtime:
  [ ] Health check defined (HEALTHCHECK in Dockerfile or compose)
  [ ] Resource limits set (--memory, --cpus)
  [ ] Logging to stdout/stderr with log rotation (max-size, max-file)
  [ ] Graceful shutdown handles SIGTERM
  [ ] Restart policy set (unless-stopped or on-failure)

Storage:
  [ ] Database data on named volumes (never overlay2)
  [ ] Bind mounts read-only where possible (:ro)
  [ ] No anonymous volumes in production
```

---

## Key Takeaways

| Practice | What to Remember |
|---|---|
| Image size | alpine/distroless/scratch + multi-stage. Smaller = faster + safer |
| Non-root | Always USER 1000:1000. Never run production containers as root |
| Read-only | --read-only + tmpfs + volumes. Prevents attacker persistence |
| No secrets in images | BuildKit --mount=type=secret. Runtime env vars |
| Layer caching | Copy deps files first, then source code. Cache mounts for pkg managers |
| Health checks | Always define. Required for orchestrator health management |
| Resource limits | Always set --memory. OOM is better than taking down the host |
| Logging | stdout/stderr only. Docker handles collection. Set max-size |
| Scanning | trivy/snyk in CI. Block deploys on HIGH/CRITICAL CVEs |
| SIGTERM handling | PID 1 must handle SIGTERM. Use exec form in ENTRYPOINT |
