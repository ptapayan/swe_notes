# Docker Fundamentals — What Containers Actually Are

**Related:** [Dockerfiles](dockerfiles.md) | [Docker Networking](docker_networking.md) | [Volumes & Storage](docker_volumes_and_storage.md) | [Docker Compose](docker_compose.md) | [Docker Best Practices](docker_best_practices.md) | [GitHub Actions CI/CD](../github/github_actions_cicd.md)

---

## What Is a Container?

A container is **NOT a VM**. A container is a regular Linux process (or group of processes) that has been isolated using kernel features. There is no hypervisor, no guest OS, no hardware emulation.

**How to think about it:** A container is a process in a box. The "box" is built from three Linux kernel primitives:

```
Container = Namespaces (isolation) + Cgroups (resource limits) + Union Filesystem (layered fs)
```

```
┌──────────────────────────────────────────────────────┐
│                    Host Kernel                        │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Container A  │  │ Container B  │  │ Container C│ │
│  │ PID 1: nginx │  │ PID 1: node  │  │ PID 1: go  │ │
│  │              │  │              │  │            │ │
│  │ Namespaces:  │  │ Namespaces:  │  │ Namespaces:│ │
│  │  pid,net,mnt │  │  pid,net,mnt │  │ pid,net,mnt│ │
│  │  uts,ipc,user│  │  uts,ipc,user│  │ uts,ipc,usr│ │
│  │              │  │              │  │            │ │
│  │ Cgroups:     │  │ Cgroups:     │  │ Cgroups:   │ │
│  │  mem: 512MB  │  │  mem: 1GB    │  │ mem: 256MB │ │
│  │  cpu: 0.5    │  │  cpu: 1.0    │  │ cpu: 0.25  │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
│                                                      │
│  Shared kernel, shared host OS                       │
└──────────────────────────────────────────────────────┘
```

---

## Linux Namespaces (Isolation)

Namespaces partition kernel resources so each set of processes sees its own isolated instance of the global resource. There are 6 namespaces that Docker uses:

### PID Namespace

Each container gets its own PID tree. PID 1 inside the container is the entrypoint process. The container cannot see processes on the host or in other containers.

```bash
# Inside container: PID 1 is nginx
$ docker exec my-nginx ps aux
PID   USER     COMMAND
  1   root     nginx: master process
  5   nginx    nginx: worker process

# On host: same process has PID 28451
$ ps aux | grep nginx
root     28451  ... nginx: master process
```

**What's happening:** The kernel maintains separate PID number spaces. The process exists once in the kernel's task list but has different PID numbers depending on which namespace you observe it from.

### NET Namespace

Each container gets its own network stack — its own interfaces, IP addresses, routing table, iptables rules, and ports. Port 80 inside container A is completely independent from port 80 inside container B.

### MNT Namespace

Each container gets its own filesystem mount tree. The container sees a root filesystem (`/`) that is completely separate from the host's root. This is where the union filesystem (overlay2) is mounted.

### UTS Namespace

Each container gets its own hostname and domain name. `hostname` inside a container returns the container's hostname, not the host's.

### IPC Namespace

Isolates System V IPC objects (shared memory segments, message queues, semaphores). Processes in different IPC namespaces cannot communicate via shared memory.

### User Namespace

Maps container UIDs to different host UIDs. Root (UID 0) inside a container can be mapped to an unprivileged user (e.g., UID 100000) on the host. This is **rootless containers** — even if a container escape occurs, the attacker has no host privileges.

```bash
# Verify namespaces for a running container
$ ls -la /proc/$(docker inspect --format '{{.State.Pid}}' my-container)/ns/
lrwxrwxrwx 1 root root 0 ... cgroup -> cgroup:[4026532516]
lrwxrwxrwx 1 root root 0 ... ipc -> ipc:[4026532449]
lrwxrwxrwx 1 root root 0 ... mnt -> mnt:[4026532447]
lrwxrwxrwx 1 root root 0 ... net -> net:[4026532452]
lrwxrwxrwx 1 root root 0 ... pid -> pid:[4026532450]
lrwxrwxrwx 1 root root 0 ... user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 ... uts -> uts:[4026532448]
```

---

## Cgroups (Resource Limits)

Control Groups (cgroups) limit, account for, and isolate the resource usage of processes.

**What cgroups control:**

| Resource | Cgroup Controller | Example |
|---|---|---|
| Memory | `memory` | Max 512MB, OOM kill if exceeded |
| CPU | `cpu`, `cpuset` | 50% of one core, pin to cores 0-3 |
| Block I/O | `blkio` | Limit disk read/write bandwidth |
| Network | `net_cls`, `net_prio` | Traffic classification and priority |
| PIDs | `pids` | Max 100 processes (prevents fork bombs) |

```bash
# Set resource limits on a container
docker run -d \
  --memory=512m \
  --memory-swap=1g \
  --cpus=1.5 \
  --pids-limit=100 \
  --blkio-weight=300 \
  nginx

# Under the hood, Docker writes to cgroup files:
# /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
# /sys/fs/cgroup/cpu/docker/<container-id>/cpu.cfs_quota_us
```

**What's happening:** When a container tries to use more memory than its cgroup limit, the kernel's OOM killer terminates processes in that cgroup. The container exits with code 137 (128 + SIGKILL=9).

---

## Containers vs VMs

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│          VM Model           │     │       Container Model       │
│                             │     │                             │
│  ┌──────┐ ┌──────┐         │     │  ┌──────┐ ┌──────┐         │
│  │ App1 │ │ App2 │         │     │  │ App1 │ │ App2 │         │
│  ├──────┤ ├──────┤         │     │  ├──────┤ ├──────┤         │
│  │ Bins │ │ Bins │         │     │  │ Bins │ │ Bins │         │
│  │ Libs │ │ Libs │         │     │  │ Libs │ │ Libs │         │
│  ├──────┤ ├──────┤         │     │  └───┬──┘ └──┬───┘         │
│  │Guest │ │Guest │         │     │      │       │              │
│  │  OS  │ │  OS  │         │     │  ┌───┴───────┴───┐         │
│  └──┬───┘ └──┬───┘         │     │  │  Docker Engine │         │
│  ┌──┴────────┴───┐         │     │  ├───────────────┤         │
│  │  Hypervisor   │         │     │  │   Host Kernel  │         │
│  ├───────────────┤         │     │  ├───────────────┤         │
│  │  Host Kernel  │         │     │  │   Host OS      │         │
│  ├───────────────┤         │     │  └───────────────┘         │
│  │  Host OS      │         │     │                             │
│  └───────────────┘         │     │                             │
└─────────────────────────────┘     └─────────────────────────────┘
```

| Aspect | VM | Container |
|---|---|---|
| Isolation | Hardware-level (hypervisor) | OS-level (namespaces + cgroups) |
| Guest OS | Full guest OS per VM | Shared host kernel |
| Boot time | 30s-minutes | Milliseconds |
| Image size | GBs (full OS) | MBs (just app + libs) |
| Overhead | High (hypervisor, guest kernel) | Near-zero |
| Density | ~10-20 VMs per host | ~100-1000 containers per host |
| Security boundary | Stronger (separate kernels) | Weaker (shared kernel) |

**Key insight:** Because containers share the host kernel, a kernel exploit in a container can compromise the host. This is why VMs are still used for hard multi-tenant isolation (e.g., AWS runs each customer's containers in separate Firecracker microVMs).

---

## Docker Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                         Docker Host                            │
│                                                                │
│  ┌──────────┐    REST API     ┌──────────────┐                │
│  │  Docker   │ ◄────────────► │   dockerd     │                │
│  │  CLI      │  /var/run/     │  (daemon)     │                │
│  │ (client)  │  docker.sock   │               │                │
│  └──────────┘                 └───────┬───────┘                │
│                                       │                        │
│                               ┌───────▼───────┐               │
│                               │  containerd   │               │
│                               │  (runtime)    │               │
│                               └───────┬───────┘               │
│                                       │ gRPC                   │
│                          ┌────────────┼────────────┐          │
│                    ┌─────▼────┐ ┌─────▼────┐ ┌────▼─────┐    │
│                    │containerd│ │containerd│ │containerd│    │
│                    │ -shim   │ │ -shim   │ │ -shim   │    │
│                    └─────┬───┘ └─────┬───┘ └────┬────┘    │
│                    ┌─────▼───┐ ┌─────▼───┐ ┌────▼────┐    │
│                    │  runc   │ │  runc   │ │  runc   │    │
│                    │ (OCI)   │ │ (OCI)   │ │ (OCI)   │    │
│                    └─────────┘ └─────────┘ └─────────┘    │
│                                                                │
│  ┌──────────────────────────────┐                              │
│  │  Docker Registry (remote)    │                              │
│  │  (Docker Hub / ECR / GCR)    │                              │
│  └──────────────────────────────┘                              │
└────────────────────────────────────────────────────────────────┘
```

### Components

**Docker CLI (client):** Translates user commands (`docker run`, `docker build`) into REST API calls to the Docker daemon. Can talk to remote daemons.

**dockerd (daemon):** The main Docker daemon. Manages images, containers, networks, and volumes. Listens on a Unix socket (`/var/run/docker.sock`) or TCP.

**containerd:** A container runtime daemon that manages the complete container lifecycle — image transfer, storage, execution, supervision. containerd is a CNCF graduated project — it exists independently of Docker.

**containerd-shim:** An intermediary process between containerd and runc. It allows the container to keep running even if containerd restarts. One shim per container.

**runc:** The actual low-level OCI runtime that creates and runs containers. It calls the Linux kernel APIs (clone, unshare, setns) to set up namespaces and cgroups, then exec's the container's entrypoint. runc exits after starting the container — the shim takes over supervision.

---

## OCI Standard (Open Container Initiative)

The OCI defines two specifications:

1. **Runtime Specification:** How to run a container from an OCI bundle (a root filesystem + a `config.json` describing namespaces, cgroups, mounts, etc.)
2. **Image Specification:** How to package a container image (layers, manifests, config)

**Why it matters:** Any OCI-compliant image runs on any OCI-compliant runtime. You can build with Docker, run with Podman, store in any OCI registry. No vendor lock-in.

---

## How `docker run` Works Step by Step

```bash
docker run -d --name web -p 8080:80 nginx:1.25
```

**What happens (in order):**

```
1. CLI parses command → sends POST /containers/create to dockerd

2. dockerd checks if nginx:1.25 exists locally
   └── If not: pulls from registry (Docker Hub by default)
       ├── Resolves tag → manifest (SHA256 digest)
       ├── Downloads each layer (gzipped tar)
       └── Stores layers in /var/lib/docker/overlay2/

3. dockerd creates container metadata
   ├── Generates container ID
   ├── Creates overlay2 mount (union of image layers + writable layer)
   ├── Allocates network (creates veth pair, attaches to bridge)
   ├── Sets up port mapping (iptables NAT rule: host:8080 → container:80)
   └── Writes container config to /var/lib/docker/containers/<id>/

4. dockerd → containerd: create container (gRPC)
   └── containerd → containerd-shim: start shim process

5. containerd-shim → runc: create + start container
   ├── runc creates namespaces (clone/unshare)
   ├── runc configures cgroups (writes to /sys/fs/cgroup/)
   ├── runc pivots root to overlay mount
   ├── runc sets up /dev, /proc, /sys
   ├── runc execs the entrypoint (nginx)
   └── runc exits (shim supervises the container now)

6. Container is running
   └── nginx (PID 1 in container) listens on port 80
   └── Host iptables forwards :8080 → container's veth IP:80
```

---

## Container Lifecycle

```
Created ──► Running ──► Paused
   │            │           │
   │            ▼           │
   │        Stopped ◄──────┘
   │            │
   ▼            ▼
  Removed    Removed
```

```bash
docker create nginx         # Created (filesystem ready, not running)
docker start <id>           # Running
docker pause <id>           # Paused (SIGSTOP to all processes via cgroups freezer)
docker unpause <id>         # Running again
docker stop <id>            # Stopped (SIGTERM → 10s grace → SIGKILL)
docker kill <id>            # Stopped (immediate SIGKILL)
docker rm <id>              # Removed (filesystem cleaned up)
```

**What `docker stop` does:** Sends SIGTERM to PID 1, waits 10 seconds (configurable with `--time`), then sends SIGKILL. Your application must handle SIGTERM for graceful shutdown.

---

## Container Runtimes Comparison

| Runtime | Level | Description |
|---|---|---|
| runc | Low-level (OCI) | Reference implementation. Creates namespaces + cgroups |
| crun | Low-level (OCI) | Written in C, faster startup than runc |
| gVisor (runsc) | Low-level (OCI) | Intercepts syscalls in user-space. Stronger isolation |
| Kata Containers | Low-level (OCI) | Runs container in a lightweight VM. VM-level isolation |
| containerd | High-level | Manages container lifecycle, image operations |
| CRI-O | High-level | Kubernetes-specific runtime, lighter than containerd |

---

## Practical Examples / Real-World Patterns

### Inspect What Namespaces a Container Uses

```bash
# Get the container's PID on the host
HOST_PID=$(docker inspect --format '{{.State.Pid}}' my-container)

# Enter the container's namespaces manually (what docker exec does)
nsenter --target $HOST_PID --mount --uts --ipc --net --pid -- /bin/sh

# See cgroup resource limits
cat /sys/fs/cgroup/memory/docker/$(docker inspect --format '{{.Id}}' my-container)/memory.limit_in_bytes
```

### Check Why a Container Was OOM-Killed

```bash
docker inspect --format '{{.State.OOMKilled}}' my-container
# true → container exceeded its memory cgroup limit

docker inspect --format '{{.State.ExitCode}}' my-container
# 137 → killed by SIGKILL (128 + 9), typical OOM kill
```

### Run a Container with Reduced Privileges

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --user 1000:1000 \
  my-app
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Container | A regular Linux process isolated with namespaces + cgroups + union fs |
| Namespaces | 6 types: pid, net, mnt, uts, ipc, user. Provide isolation |
| Cgroups | Limit CPU, memory, I/O, PIDs. OOM kill on breach |
| vs VMs | Containers share the host kernel. Faster, lighter, weaker isolation |
| Docker architecture | CLI → dockerd → containerd → shim → runc → container process |
| OCI standard | Vendor-neutral spec for images and runtimes |
| `docker run` | Pull image → create overlay mount → setup network → runc creates namespaces → exec entrypoint |
| Security | Shared kernel = kernel exploit risk. Use user namespaces, drop caps, read-only fs |
