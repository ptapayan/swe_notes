# Dockerfiles — Deep Dive

**Related:** [Docker Fundamentals](docker_fundamentals.md) | [Volumes & Storage](docker_volumes_and_storage.md) | [Docker Best Practices](docker_best_practices.md) | [GitHub Actions CI/CD](../github/github_actions_cicd.md)

---

## What Is a Dockerfile?

A Dockerfile is a text file containing instructions that Docker reads to **build an image**. Each instruction creates a **layer** in the image. Layers are cached and reused to speed up subsequent builds.

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates
COPY --from=builder /app/server /usr/local/bin/server
USER 1000:1000
EXPOSE 8080
ENTRYPOINT ["server"]
```

---

## Instructions Deep Dive

### FROM — Base Image

Every Dockerfile must start with `FROM`. It sets the base image that subsequent layers build upon.

```dockerfile
FROM ubuntu:22.04            # specific tag (always pin versions in production)
FROM golang:1.22-alpine      # alpine variant (smaller, musl libc)
FROM scratch                 # empty image (for fully static binaries)
FROM node:20 AS builder      # named build stage (for multi-stage builds)
```

**What's happening:** `FROM` tells Docker which image layers to start with. `scratch` is literally an empty filesystem — your binary must be completely static.

### RUN — Execute Commands

Executes a command in a new layer. The result is committed as a new image layer.

```dockerfile
# Shell form (runs in /bin/sh -c)
RUN apt-get update && apt-get install -y curl

# Exec form (no shell interpretation — preferred for predictability)
RUN ["apt-get", "update"]

# IMPORTANT: chain commands to reduce layers and avoid stale apt cache
# BAD: two layers, apt cache in first layer is useless
RUN apt-get update
RUN apt-get install -y curl

# GOOD: single layer, clean up in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

**What's happening:** Each `RUN` creates a new layer via overlay2. Even if you `rm` files in a later layer, they still exist in the previous layer (the image gets bigger). Always clean up in the same `RUN` command.

### COPY vs ADD

```dockerfile
COPY app.py /opt/app/           # copies from build context to image
COPY --chown=1000:1000 . /app/  # copy with ownership change
COPY --from=builder /app/bin /usr/local/bin/  # copy from another build stage

ADD app.tar.gz /opt/app/        # like COPY but auto-extracts tar archives
ADD https://example.com/f.txt /opt/  # can fetch URLs (NOT recommended)
```

**Best practice:** Always use `COPY` unless you specifically need tar auto-extraction. `ADD` is too "magic" — it does too many things implicitly.

### CMD vs ENTRYPOINT

```dockerfile
# CMD: default command — can be overridden by docker run arguments
CMD ["nginx", "-g", "daemon off;"]

# ENTRYPOINT: the executable — docker run arguments are appended
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]   # default args to ENTRYPOINT
```

**How they interact:**

```
┌─────────────────────────────┬───────────────────────────────────┐
│ Dockerfile                  │ docker run my-image               │
├─────────────────────────────┼───────────────────────────────────┤
│ CMD ["nginx"]               │ Runs: nginx                       │
│                             │ docker run my-image curl → curl   │
├─────────────────────────────┼───────────────────────────────────┤
│ ENTRYPOINT ["nginx"]        │ Runs: nginx                       │
│                             │ docker run my-image -v → nginx -v │
├─────────────────────────────┼───────────────────────────────────┤
│ ENTRYPOINT ["nginx"]        │ Runs: nginx -g daemon off;        │
│ CMD ["-g", "daemon off;"]   │ docker run my-image -v → nginx -v │
└─────────────────────────────┴───────────────────────────────────┘
```

**Shell form vs exec form:**

```dockerfile
# Exec form (preferred): PID 1 is your process, receives signals
ENTRYPOINT ["nginx", "-g", "daemon off;"]

# Shell form: PID 1 is /bin/sh, your process is a child, misses SIGTERM
ENTRYPOINT nginx -g "daemon off;"
```

**Critical:** If you use shell form, `docker stop` sends SIGTERM to `/bin/sh`, not your app. Your app won't shut down gracefully — Docker will SIGKILL it after the timeout.

### ENV and ARG

```dockerfile
# ARG: build-time only. Not available in running container.
ARG GO_VERSION=1.22
FROM golang:${GO_VERSION}-alpine

# ENV: persists in running container. Also available during build.
ENV APP_PORT=8080
ENV NODE_ENV=production

# ARG can feed ENV
ARG VERSION
ENV APP_VERSION=${VERSION}
```

```bash
docker build --build-arg GO_VERSION=1.21 --build-arg VERSION=2.0.0 .
```

**Security:** `ARG` values are visible in `docker history`. Never pass secrets as `ARG`. Use BuildKit secrets instead:

```dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm install
```

```bash
docker build --secret id=npm_token,src=.npmrc .
```

### Other Important Instructions

```dockerfile
WORKDIR /app          # set working directory (creates if doesn't exist)
                      # ALWAYS use WORKDIR, never RUN cd /app && ...

USER 1000:1000        # switch to non-root user (by UID, not name — portable)

EXPOSE 8080           # documentation only — doesn't actually publish ports
                      # you still need -p 8080:8080 at runtime

VOLUME /data          # creates a mount point. Docker creates anonymous volume
                      # at runtime. Prefer named volumes via docker run -v

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

---

## Build Context

When you run `docker build .`, the `.` is the **build context** — the directory sent to the Docker daemon. Docker tars up the entire context and sends it to the daemon.

```bash
# SLOW: sending 2GB node_modules to daemon
docker build .

# FAST: .dockerignore excludes unnecessary files
```

### .dockerignore

```
.git
node_modules
*.md
.env
docker-compose*.yml
.DS_Store
__pycache__
*.pyc
coverage/
dist/
.terraform/
```

**What's happening:** Like `.gitignore`, but for Docker build context. Reduces the context size sent to the daemon, speeds up builds, and prevents accidentally copying secrets or large files into images.

---

## Layer Caching — How It Works

```
Build instruction → SHA256 of (parent layer + instruction + files)
                          │
                    Cache hit? ──► YES → reuse layer
                          │
                          NO → execute instruction → create new layer
```

**Cache invalidation rules:**
1. If any previous layer is invalidated, ALL subsequent layers are rebuilt
2. `RUN` instructions are cached based on the command string, not the output
3. `COPY`/`ADD` are cached based on file checksums in the build context
4. `ARG` changes invalidate from the point they're used

### Layer Ordering for Cache Efficiency

```dockerfile
# BAD: any code change invalidates the dependency install
COPY . /app
RUN npm install        # re-runs every time ANY file changes

# GOOD: dependencies cached separately from code
COPY package.json package-lock.json ./
RUN npm install                          # only re-runs if package*.json changes
COPY . .                                 # code changes don't bust dep cache
```

```
Layer 1: FROM node:20-alpine         ← cached (rarely changes)
Layer 2: COPY package*.json ./       ← cached (deps don't change often)
Layer 3: RUN npm install             ← cached (same deps = same layer)
Layer 4: COPY . .                    ← rebuilt (code changed)
Layer 5: RUN npm run build           ← rebuilt (dependency on layer 4)
```

---

## Multi-Stage Builds

Multi-stage builds use multiple `FROM` statements. Only the final stage ends up in the output image. This dramatically reduces image size.

```dockerfile
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

# Stage 2: Runtime
FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
ENTRYPOINT ["/server"]
```

**Size comparison:**

```
golang:1.22-alpine (builder)  →  ~250MB
scratch + static binary       →    ~8MB  (97% reduction)
```

**What's happening:** The build stage contains the Go toolchain, source code, and all build artifacts. `COPY --from=builder` extracts only the compiled binary. The final image has nothing but the binary and CA certificates.

### Multi-Stage for Testing

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM deps AS test
COPY . .
RUN npm test

FROM deps AS build
COPY . .
RUN npm run build

FROM nginx:alpine AS production
COPY --from=build /app/dist /usr/share/nginx/html
```

```bash
# Run only the test stage (CI validation)
docker build --target test .

# Build the full production image
docker build --target production .
```

---

## Practical Examples / Real-World Patterns

### Python Production Dockerfile

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir poetry
COPY pyproject.toml poetry.lock ./
RUN poetry export -f requirements.txt -o requirements.txt --without-hashes
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
RUN useradd -r -s /bin/false appuser
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
CMD ["gunicorn", "app:create_app()", "-b", "0.0.0.0:8000", "-w", "4"]
```

### Java Spring Boot Dockerfile (Layered)

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew build.gradle settings.gradle ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon
COPY src ./src
RUN ./gradlew bootJar --no-daemon
RUN java -Djarmode=layertools -jar build/libs/*.jar extract

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
USER spring:spring
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

### Node.js with Build Caching

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production && \
    cp -R node_modules /prod_modules && \
    npm ci

FROM deps AS builder
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /prod_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./
USER 1000:1000
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## Build Performance Tips

```bash
# Enable BuildKit (default in Docker 23.0+, explicit in older)
DOCKER_BUILDKIT=1 docker build .

# Use cache mounts for package managers (BuildKit feature)
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/root/.npm npm ci

# Parallel multi-stage builds (BuildKit builds independent stages concurrently)
# Stage "test" and "build" can run in parallel if both only depend on "deps"
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| FROM | Always pin versions. Use `scratch` for static binaries |
| RUN | Chain commands with `&&`. Clean up in the same layer |
| COPY vs ADD | Always use COPY unless you need tar extraction |
| CMD vs ENTRYPOINT | ENTRYPOINT = the executable, CMD = default args |
| Exec vs shell form | Always exec form `["cmd"]`. Shell form breaks signal handling |
| Layer caching | Copy dependency files first, then source code |
| Multi-stage | Build in one stage, run in another. Massive size reduction |
| .dockerignore | Always create one. Exclude .git, node_modules, .env |
| Secrets | Never ARG for secrets. Use `--mount=type=secret` with BuildKit |
| HEALTHCHECK | Always define one. Enables orchestrator health monitoring |
