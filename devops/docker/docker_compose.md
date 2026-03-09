# Docker Compose — Deep Dive

**Related:** [Docker Fundamentals](docker_fundamentals.md) | [Dockerfiles](dockerfiles.md) | [Docker Networking](docker_networking.md) | [Volumes & Storage](docker_volumes_and_storage.md) | [Docker Best Practices](docker_best_practices.md) | [GitHub Actions CI/CD](../github/github_actions_cicd.md)

---

## What Is Docker Compose?

Docker Compose is a tool for defining and running **multi-container applications** using a single YAML file. Instead of running multiple `docker run` commands with complex flags, you declaratively describe the entire application stack.

```bash
docker compose up -d      # start everything
docker compose down        # stop and remove everything
docker compose logs -f     # tail all logs
docker compose ps          # list running services
```

**What's happening:** `docker compose up` reads `docker-compose.yml`, creates a default bridge network, pulls/builds images, creates containers, sets up volumes, and starts everything in dependency order.

---

## docker-compose.yml Structure

```yaml
# Compose file reference (no version field needed for Compose V2+)
name: my-application    # optional: project name (default: directory name)

services:               # container definitions
  web:
    image: nginx:1.25
    ports:
      - "8080:80"

  api:
    build: ./api
    ports:
      - "3000:3000"

networks:               # custom network definitions
  frontend:
  backend:

volumes:                # named volume definitions
  db-data:
  redis-data:

configs:                # external configuration files
  nginx-conf:
    file: ./nginx.conf

secrets:                # sensitive data
  db-password:
    file: ./db_password.txt
```

---

## Service Configuration

### Build vs Image

```yaml
services:
  # Pull a pre-built image
  redis:
    image: redis:7-alpine

  # Build from a Dockerfile
  api:
    build: ./api                    # uses ./api/Dockerfile
    # OR with more control:
    build:
      context: ./api               # build context directory
      dockerfile: Dockerfile.prod  # specific Dockerfile
      args:                        # build-time arguments
        NODE_ENV: production
      target: production           # multi-stage build target
      cache_from:
        - my-registry/api:latest
```

### Ports

```yaml
services:
  web:
    ports:
      - "8080:80"                # host:container
      - "127.0.0.1:8443:443"    # bind to localhost only
      - "9090:9090/udp"         # UDP
      - "3000-3005:3000-3005"   # port range

    expose:
      - "9090"                   # expose to other containers (not host)
```

### Environment Variables

```yaml
services:
  api:
    environment:
      NODE_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      API_KEY: ${API_KEY}         # from host environment or .env file

    # OR load from a file
    env_file:
      - .env
      - .env.production
```

### depends_on and Healthchecks

```yaml
services:
  api:
    depends_on:
      postgres:
        condition: service_healthy    # wait for postgres to be healthy
      redis:
        condition: service_started    # just wait for container to start

  postgres:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s             # grace period before health checks start

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

**What's happening:** `depends_on` with `condition: service_healthy` blocks the dependent service from starting until the dependency's healthcheck passes. Without `condition`, it only waits for the container to be created (not necessarily ready).

### Restart Policies

```yaml
services:
  api:
    restart: unless-stopped     # restart unless explicitly stopped

  # Options:
  # "no"              → never restart (default)
  # "always"          → always restart
  # "on-failure"      → restart only on non-zero exit code
  # "unless-stopped"  → restart unless docker stop was called
```

### Deploy (Resource Limits)

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: "1.0"           # max 1 CPU core
          memory: 512M          # max 512MB RAM
        reservations:
          cpus: "0.25"          # guaranteed 0.25 CPU
          memory: 128M          # guaranteed 128MB RAM
      replicas: 3               # run 3 instances (Swarm mode)
```

**Note:** `deploy.resources` works with `docker compose up` since Compose V2. In Compose V1, resource limits required Swarm mode.

### Volumes in Services

```yaml
services:
  api:
    volumes:
      - app-data:/app/data                    # named volume
      - ./src:/app/src                        # bind mount (relative path)
      - /var/log/api:/app/logs                # bind mount (absolute path)
      - ./config/app.conf:/app/config:ro      # read-only bind mount
      - type: tmpfs                           # tmpfs mount
        target: /tmp
        tmpfs:
          size: 100000000                     # 100MB
```

---

## Networking in Compose

### Default Network Behavior

```yaml
# docker-compose.yml in directory "myapp"
services:
  web:
    image: nginx
  api:
    image: my-api
  db:
    image: postgres
```

**What Compose creates automatically:**

```
Network: myapp_default (bridge)
  ├── web   (hostname: web,   IP: 172.18.0.2)
  ├── api   (hostname: api,   IP: 172.18.0.3)
  └── db    (hostname: db,    IP: 172.18.0.4)

All three can reach each other by service name:
  api → http://db:5432 ✓
  web → http://api:3000 ✓
```

**What's happening:** Compose creates a single bridge network named `<project>_default`. Every service gets a DNS entry matching its service name. The embedded DNS server (127.0.0.11) resolves these names.

### Custom Networks for Isolation

```yaml
services:
  nginx:
    image: nginx
    networks:
      - frontend
    ports:
      - "80:80"

  api:
    image: my-api
    networks:
      - frontend       # nginx can reach api
      - backend         # api can reach db

  db:
    image: postgres
    networks:
      - backend         # only api can reach db

networks:
  frontend:
  backend:
```

```
┌───────────────────────────────────────┐
│  frontend network                     │
│  ┌───────┐         ┌──────┐         │
│  │ nginx │ ◄──────► │ api  │         │
│  └───────┘         └──┬───┘         │
│                        │              │
├────────────────────────┼──────────────┤
│                        │              │
│  backend network       │              │
│                   ┌────▼──┐          │
│                   │  api  │          │
│                   │       │          │
│                   └───┬───┘          │
│                       │               │
│                   ┌───▼───┐          │
│                   │  db   │          │
│                   └───────┘          │
└───────────────────────────────────────┘

nginx → db: BLOCKED (different networks)
api → db: ALLOWED (same backend network)
nginx → api: ALLOWED (same frontend network)
```

### Service Discovery by Name

```yaml
services:
  api:
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
      REDIS_URL: redis://cache:6379
      # "db" and "cache" resolve via Docker's embedded DNS

  db:
    image: postgres:16

  cache:
    image: redis:7-alpine
```

---

## Environment Variables and .env Files

### .env File (Auto-Loaded)

Docker Compose automatically loads a `.env` file from the project directory:

```bash
# .env
POSTGRES_VERSION=16
APP_PORT=3000
DB_PASSWORD=supersecret
COMPOSE_PROJECT_NAME=myapp
```

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:${POSTGRES_VERSION}    # resolves to postgres:16
  api:
    ports:
      - "${APP_PORT}:3000"                  # resolves to 3000:3000
    environment:
      DB_PASSWORD: ${DB_PASSWORD}           # resolves to supersecret
```

### Variable Precedence (highest to lowest)

```
1. Shell environment variables (export VAR=value)
2. .env file in project directory
3. Default values in compose file: ${VAR:-default}
```

```yaml
services:
  api:
    image: my-api:${VERSION:-latest}   # uses $VERSION if set, else "latest"
    environment:
      LOG_LEVEL: ${LOG_LEVEL:-info}    # default to "info"
```

---

## Compose Profiles

Profiles let you selectively start services. Services without a profile always start.

```yaml
services:
  api:
    image: my-api              # always starts (no profile)

  db:
    image: postgres:16         # always starts

  debug-tools:
    image: debug-image
    profiles: ["debug"]        # only starts with --profile debug

  monitoring:
    image: prometheus
    profiles: ["monitoring"]   # only starts with --profile monitoring

  seed-db:
    image: my-api
    command: npm run seed
    profiles: ["setup"]        # one-time setup tasks
```

```bash
docker compose up -d                          # starts: api, db
docker compose --profile debug up -d          # starts: api, db, debug-tools
docker compose --profile debug --profile monitoring up -d  # all except seed-db
docker compose run --rm seed-db               # run seed task once
```

---

## Common Commands

```bash
# Lifecycle
docker compose up -d                    # start all services (detached)
docker compose down                     # stop and remove containers + networks
docker compose down -v                  # also remove volumes (DATA LOSS)
docker compose down --rmi all           # also remove images

# Monitoring
docker compose ps                       # list services and their status
docker compose logs -f api              # follow logs for api service
docker compose logs --tail=100 api db   # last 100 lines from api and db
docker compose top                      # show running processes

# Operations
docker compose exec api sh              # shell into running api container
docker compose run --rm api npm test    # run one-off command in new container
docker compose restart api              # restart a service
docker compose pull                     # pull latest images
docker compose build --no-cache         # rebuild all images from scratch

# Scaling
docker compose up -d --scale api=3     # run 3 instances of api
```

---

## Practical Examples / Real-World Patterns

### Full-Stack Application (App + DB + Cache)

```yaml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      api:
        condition: service_healthy
    networks:
      - frontend
    restart: unless-stopped

  api:
    build:
      context: ./api
      target: production
    environment:
      DATABASE_URL: postgresql://app:${DB_PASSWORD}@db:5432/mydb
      REDIS_URL: redis://cache:6379
      NODE_ENV: production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"
    networks:
      - frontend
      - backend
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 1G
    networks:
      - backend
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    networks:
      - backend
    restart: unless-stopped

networks:
  frontend:
  backend:

volumes:
  pg-data:
  redis-data:
```

### Development vs Production Configs (Override Files)

```yaml
# docker-compose.yml (base / production)
services:
  api:
    image: my-registry/api:${VERSION:-latest}
    environment:
      NODE_ENV: production
    restart: unless-stopped
```

```yaml
# docker-compose.override.yml (auto-loaded in development)
services:
  api:
    build: ./api
    volumes:
      - ./api/src:/app/src           # live reload
    environment:
      NODE_ENV: development
      DEBUG: "true"
    ports:
      - "9229:9229"                  # debugger port
    command: npm run dev
```

```bash
# Development (auto-loads override):
docker compose up

# Production (explicit file, skips override):
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### CI/CD Testing with Compose

```yaml
# docker-compose.test.yml
services:
  test:
    build:
      context: .
      target: test
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://test:test@db:5432/testdb
    command: npm run test:integration

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 3s
      retries: 5
    tmpfs:
      - /var/lib/postgresql/data     # RAM-only for speed
```

```bash
# Run in CI pipeline
docker compose -f docker-compose.test.yml up --build --abort-on-container-exit --exit-code-from test
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Default network | Compose creates `<project>_default` bridge. Services resolve by name |
| Custom networks | Use for isolation. Frontend/backend separation |
| depends_on | Use `condition: service_healthy` for real readiness checks |
| .env file | Auto-loaded. For variable substitution in compose file |
| Profiles | Selectively start services. For debug tools, setup tasks |
| Override files | `docker-compose.override.yml` auto-loaded for dev config |
| Healthchecks | Always define them. Critical for depends_on and orchestrator health |
| Volumes in compose | Define top-level named volumes. Use bind mounts for dev only |
| `down -v` | Removes volumes too. Data loss. Be careful in production |
| Resource limits | `deploy.resources.limits` works in Compose V2 without Swarm |
