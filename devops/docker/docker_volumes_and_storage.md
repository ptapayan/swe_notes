# Docker Volumes and Storage — Deep Dive

**Related:** [Docker Fundamentals](docker_fundamentals.md) | [Dockerfiles](dockerfiles.md) | [Docker Compose](docker_compose.md) | [Docker Best Practices](docker_best_practices.md)

---

## Union Filesystem (How Container Filesystems Work)

A Docker image is built from **read-only layers** stacked on top of each other. When you run a container, Docker adds a thin **writable layer** on top. This is implemented using a **union filesystem** — specifically **overlay2** on modern Linux.

### Overlay2 Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Container View                         │
│                  (merged directory)                       │
│  /app/server  /etc/nginx.conf  /var/log/app.log (new)   │
│                                                          │
│  What the container process sees — a unified filesystem  │
└────────────────────────┬─────────────────────────────────┘
                         │ overlay2 mount
┌────────────────────────┼─────────────────────────────────┐
│                        │                                  │
│  ┌─────────────────────▼──────────────────────┐          │
│  │  UPPER (writable layer / container layer)   │          │
│  │  /var/lib/docker/overlay2/<id>/diff/        │          │
│  │  Contains: new files, modified files,       │          │
│  │            deleted files (whiteout markers)  │          │
│  └─────────────────────────────────────────────┘          │
│                                                           │
│  ┌─────────────────────────────────────────────┐          │
│  │  LOWER (read-only image layers, stacked)    │          │
│  │                                              │          │
│  │  Layer 3: COPY . /app     (app source)      │          │
│  │  Layer 2: RUN npm install (dependencies)    │          │
│  │  Layer 1: FROM node:20    (base OS + node)  │          │
│  └─────────────────────────────────────────────┘          │
│                                                           │
│  ┌─────────────────────────────────────────────┐          │
│  │  WORKDIR                                     │          │
│  │  /var/lib/docker/overlay2/<id>/work/         │          │
│  │  (internal overlay2 bookkeeping, not visible)│          │
│  └─────────────────────────────────────────────┘          │
└───────────────────────────────────────────────────────────┘
```

### How Reads and Writes Work (Copy-on-Write)

**Reading a file:** overlay2 looks in the upper (writable) layer first. If not found, it searches down through the lower layers. The first match wins.

**Writing to an existing file (Copy-on-Write):**
1. The file is found in a lower (read-only) layer
2. overlay2 **copies** the entire file to the upper (writable) layer
3. The write happens on the copy in the upper layer
4. All subsequent reads hit the upper layer copy

**Deleting a file:** overlay2 creates a **whiteout file** in the upper layer. This is a special character device that tells overlay2 to hide the file in the lower layers. The original file still exists in the lower layer (which is why image layers never shrink when you delete files in a later layer).

**Creating a new file:** Written directly to the upper layer. No copy needed.

```bash
# Inspect a container's overlay2 layers
docker inspect --format '{{json .GraphDriver.Data}}' my-container | jq .
# {
#   "LowerDir": "/var/lib/docker/overlay2/abc123/diff:...",
#   "UpperDir": "/var/lib/docker/overlay2/xyz789/diff",
#   "MergedDir": "/var/lib/docker/overlay2/xyz789/merged",
#   "WorkDir": "/var/lib/docker/overlay2/xyz789/work"
# }

# See what the container has written (files in the writable layer)
ls /var/lib/docker/overlay2/xyz789/diff/
```

---

## Storage Drivers

The storage driver is the component that manages the layered filesystem. overlay2 is the default and recommended driver.

| Driver | Status | Use Case |
|---|---|---|
| `overlay2` | **Default, recommended** | All modern Linux. Best performance |
| `btrfs` | Supported | Hosts running Btrfs filesystem |
| `zfs` | Supported | Hosts running ZFS |
| `vfs` | Debug only | No copy-on-write. Copies entire layer for each container. Very slow |
| `devicemapper` | Deprecated | Older RHEL/CentOS. Don't use for new deployments |
| `aufs` | Deprecated | Original Docker storage driver. Replaced by overlay2 |

```bash
# Check which storage driver is in use
docker info | grep "Storage Driver"
# Storage Driver: overlay2
```

---

## Three Types of Persistent Storage

```
┌──────────────────────────────────────────────────────────────┐
│                        Docker Host                           │
│                                                              │
│  ┌────────────┐                                              │
│  │ Container  │                                              │
│  │            │                                              │
│  │ /app/data ─┼──── Volume ──────► /var/lib/docker/volumes/  │
│  │            │     (Docker-managed)     my-vol/_data/        │
│  │            │                                              │
│  │ /app/src ──┼──── Bind Mount ──► /home/user/project/src/   │
│  │            │     (host path)                              │
│  │            │                                              │
│  │ /tmp ──────┼──── tmpfs ───────► RAM (never written to disk)│
│  │            │                                              │
│  └────────────┘                                              │
└──────────────────────────────────────────────────────────────┘
```

### Volumes (Docker-Managed)

Volumes are the **preferred mechanism** for persisting data. Docker manages the storage location on the host.

```bash
# Create a named volume
docker volume create my-data

# Run with a named volume
docker run -d -v my-data:/var/lib/postgresql/data postgres

# Equivalent long-form (more explicit, preferred)
docker run -d --mount type=volume,source=my-data,target=/var/lib/postgresql/data postgres

# Anonymous volume (Docker generates a random name)
docker run -d -v /var/lib/postgresql/data postgres

# List volumes
docker volume ls

# Inspect volume (see mount point on host)
docker volume inspect my-data
# [{ "Mountpoint": "/var/lib/docker/volumes/my-data/_data", ... }]
```

**What's happening:** Docker creates a directory at `/var/lib/docker/volumes/<name>/_data/` and bind-mounts it into the container. The volume bypasses the overlay2 filesystem entirely — writes go directly to the host filesystem.

**Volume advantages:**
- Managed by Docker (easy backup, migration)
- Works on Linux and Docker Desktop
- Can be pre-populated from container's image content
- Support volume drivers for remote storage (NFS, cloud storage)
- Better performance than overlay2 writes

### Bind Mounts (Host Path)

Bind mounts map a specific host directory into the container. You control the exact path.

```bash
# Bind mount current directory into container
docker run -v $(pwd):/app my-image

# Read-only bind mount (container can't write)
docker run -v $(pwd)/config:/etc/app/config:ro my-image

# Long-form
docker run --mount type=bind,source=$(pwd),target=/app my-image
```

**Use cases:**
- **Development:** Mount source code for live reloading
- **Configuration:** Mount config files into containers
- **Sharing data:** Between host and container

**Dangers:**
- Container can modify host files (if not `:ro`)
- Host path must exist (unlike volumes, not auto-created)
- Not portable (paths differ between hosts)
- Security risk if mounting sensitive host directories

### tmpfs Mounts (RAM Only)

tmpfs mounts data in memory. Nothing is written to disk. Data is lost when the container stops.

```bash
docker run --tmpfs /tmp:rw,noexec,nosuid,size=100m my-image

# Long-form
docker run --mount type=tmpfs,target=/tmp,tmpfs-size=104857600 my-image
```

**Use cases:**
- Sensitive data that shouldn't persist (tokens, temp credentials)
- High-performance scratch space
- When using `--read-only` filesystem but app needs writable /tmp

---

## Volume Lifecycle

```bash
# Create
docker volume create app-data

# Use (multiple containers can mount the same volume)
docker run -d -v app-data:/data --name writer my-writer
docker run -d -v app-data:/data:ro --name reader my-reader

# Inspect
docker volume inspect app-data

# Remove (only works if no container is using it)
docker volume rm app-data

# Remove all unused volumes (DANGEROUS in production)
docker volume prune

# Remove container but keep volume
docker rm writer    # volume survives

# Remove container AND its anonymous volumes
docker rm -v writer  # anonymous volumes deleted, named volumes survive
```

**Critical:** Named volumes persist until explicitly deleted. Anonymous volumes (created without a name) are deleted when you `docker rm -v` the container or `docker volume prune`.

---

## Data Persistence Patterns

### Database Persistence

```bash
# PostgreSQL with named volume
docker run -d \
  --name postgres \
  -v pg-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# MySQL with named volume
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8
```

**What's happening:** The database writes its data files to `/var/lib/postgresql/data` inside the container. Because this path is a volume mount, the writes go directly to `/var/lib/docker/volumes/pg-data/_data/` on the host, bypassing overlay2. When the container is recreated, the data is still there.

### Application Uploads / User Data

```bash
docker run -d \
  --name app \
  -v uploads:/app/uploads \
  -v logs:/app/logs \
  my-app
```

### Development Hot-Reload Pattern

```bash
# Mount source code for live reloading
docker run -d \
  -v $(pwd)/src:/app/src \
  -v node_modules:/app/node_modules \
  -p 3000:3000 \
  my-dev-image npm run dev
```

**Key insight:** Mount `node_modules` as a named volume (not a bind mount) so that `npm install` from the Dockerfile populates it, and the host's `node_modules` (which may not exist or have different platform binaries) doesn't overwrite it.

---

## Performance Considerations

### overlay2 Write Performance

Every write to a file that exists in a lower layer triggers a **full copy** of that file to the upper layer before the write (copy-on-write). For large files (databases, logs), this is catastrophic.

```
Write to 1GB database file in overlay2:
1. Copy 1GB from lower → upper layer (copy-on-write)
2. Perform the actual write on the upper copy
3. Every subsequent write modifies the upper copy (no more CoW)

Write to 1GB database file on a volume:
1. Perform the write directly on the host filesystem
   (no copy, no overlay2 involvement)
```

### Volume vs Bind Mount vs overlay2 Performance

| Operation | overlay2 (container layer) | Volume / Bind Mount |
|---|---|---|
| Read existing file | Fast (direct from layer) | Fast (direct from host fs) |
| First write to existing file | Slow (copy-on-write) | Fast (no CoW) |
| Subsequent writes | Fast (upper layer) | Fast |
| Create new file | Fast | Fast |
| Heavy I/O workloads | Poor (CoW overhead) | Native host fs performance |

**Rule of thumb:** Any data that is written frequently (databases, logs, uploads) must be on a volume. The container's writable layer should only be used for ephemeral runtime state.

### Direct Volume I/O Optimization

```bash
# Use delegated consistency for macOS Docker Desktop (significant speedup)
docker run -v $(pwd):/app:delegated my-image

# For Linux, volumes are already native performance
# But you can choose the volume driver for specific storage backends
docker volume create --driver local \
  --opt type=none \
  --opt o=bind \
  --opt device=/mnt/fast-ssd/app-data \
  fast-volume
```

---

## Backup Strategies

### Volume Backup with tar

```bash
# Backup a named volume to a tar file
docker run --rm \
  -v pg-data:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/pg-data-backup.tar.gz -C /source .

# Restore from tar
docker run --rm \
  -v pg-data:/target \
  -v $(pwd):/backup:ro \
  alpine sh -c "cd /target && tar xzf /backup/pg-data-backup.tar.gz"
```

### Database-Specific Backups

```bash
# PostgreSQL dump (consistent, application-aware)
docker exec postgres pg_dumpall -U postgres > backup.sql

# Restore
docker exec -i postgres psql -U postgres < backup.sql

# MySQL dump
docker exec mysql mysqldump -u root -psecret --all-databases > backup.sql
```

**What's happening:** Filesystem-level backups (tar) capture raw files — they may be inconsistent if the database is actively writing. Application-level backups (pg_dump) create consistent snapshots. Always prefer application-level backups for databases.

### Volume Copy Between Hosts

```bash
# Export volume data
docker run --rm -v my-vol:/data -v $(pwd):/backup alpine \
  tar czf /backup/my-vol.tar.gz -C /data .

# Transfer to another host
scp my-vol.tar.gz user@other-host:/tmp/

# Import on other host
docker volume create my-vol
docker run --rm -v my-vol:/data -v /tmp:/backup alpine \
  tar xzf /backup/my-vol.tar.gz -C /data
```

---

## Practical Examples / Real-World Patterns

### Docker Compose with Volumes

```yaml
services:
  app:
    image: my-app
    volumes:
      - app-uploads:/app/uploads
      - ./config:/app/config:ro

  db:
    image: postgres:16
    volumes:
      - pg-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes

volumes:
  app-uploads:
  pg-data:
  redis-data:
```

### Read-Only Container with Specific Writable Paths

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=50m \
  --tmpfs /var/run:rw,noexec,nosuid \
  -v app-logs:/var/log/app \
  my-app
```

**What's happening:** `--read-only` makes the entire overlay2 filesystem read-only. `--tmpfs` provides writable directories in RAM for temp files. The volume provides persistent writable storage for logs. This is a strong security posture — an attacker who compromises the app cannot write to the filesystem.

### NFS Volume for Shared Storage

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.5,rw,nfsvers=4 \
  --opt device=:/shared/data \
  nfs-shared

docker run -d -v nfs-shared:/data my-app
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| overlay2 | Union fs: read-only lower layers + writable upper layer. Copy-on-write |
| Whiteout files | Deletions in upper layer create markers hiding lower layer files |
| Volumes | Docker-managed, best for persistent data. Bypass overlay2 |
| Bind mounts | Host path → container path. For dev and config. Not portable |
| tmpfs | RAM-only. For secrets and scratch space. Lost on container stop |
| CoW performance | First write to existing file copies entire file. Use volumes for heavy I/O |
| Database storage | Always use named volumes. Never store DB data in overlay2 |
| Backups | Use application-level dumps for databases. tar for file volumes |
| Read-only containers | `--read-only` + tmpfs + volumes. Strong security posture |
