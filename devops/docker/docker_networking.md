# Docker Networking — Deep Dive

**Related:** [Docker Fundamentals](docker_fundamentals.md) | [Docker Compose](docker_compose.md) | [Docker Best Practices](docker_best_practices.md) | [Ansible Fundamentals](../ansible/ansible_fundamentals.md)

---

## Network Drivers Overview

Docker provides several network drivers. Each driver corresponds to a different networking model with different isolation and connectivity characteristics.

| Driver | Scope | Use Case |
|---|---|---|
| `bridge` | Single host | Default. Containers on the same host communicate via virtual bridge |
| `host` | Single host | Container shares host's network stack. No isolation, full performance |
| `overlay` | Multi-host | Containers across different hosts (Swarm / Kubernetes) |
| `macvlan` | Single host | Container gets its own MAC address, appears as physical device on LAN |
| `none` | Single host | No networking. Container is completely isolated |

```bash
# List networks
docker network ls

# Create a custom bridge network
docker network create --driver bridge my-network

# Run a container on a specific network
docker run -d --network my-network --name web nginx
```

---

## Bridge Network Internals

The bridge driver is the default and most common. Understanding its internals is critical.

### Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                          Host                                     │
│                                                                   │
│  ┌─────────────┐   veth pair   ┌──────────────────────┐          │
│  │ Container A │◄─────────────►│                      │          │
│  │ eth0:       │               │                      │          │
│  │ 172.17.0.2  │               │    docker0 bridge    │          │
│  └─────────────┘               │    172.17.0.1        │          │
│                                │                      │          │
│  ┌─────────────┐   veth pair   │                      │    eth0  │
│  │ Container B │◄─────────────►│                      │◄────────►│
│  │ eth0:       │               │                      │   Host   │
│  │ 172.17.0.3  │               └──────────────────────┘   NIC    │
│  └─────────────┘                                                  │
│                                                                   │
│  iptables NAT:                                                    │
│  PREROUTING: host:8080 → 172.17.0.2:80 (DNAT)                   │
│  POSTROUTING: 172.17.0.0/16 → MASQUERADE (SNAT for outbound)    │
└───────────────────────────────────────────────────────────────────┘
```

### Veth Pairs (Virtual Ethernet)

**What's happening:** When Docker creates a container, it creates a **veth pair** — two virtual network interfaces connected like a pipe. One end goes inside the container (appears as `eth0`), the other end is attached to the `docker0` bridge on the host.

```bash
# On the host: see veth interfaces attached to docker0
bridge link show docker0
# 5: veth1234@if4: ... master docker0

# Inside container: see the eth0 interface
docker exec my-container ip addr show eth0
# 4: eth0@if5: ... inet 172.17.0.2/16
```

Every packet from the container goes through the veth pair → docker0 bridge → host's network stack.

### docker0 Bridge

The `docker0` bridge is a Linux bridge (Layer 2 switch) that Docker creates at installation. It connects all containers on the default bridge network.

```bash
# See the docker0 bridge and connected interfaces
ip addr show docker0
# docker0: ... inet 172.17.0.1/16 ...

brctl show docker0
# bridge name    bridge id          STP enabled    interfaces
# docker0        8000.0242ac110001  no             veth1234
#                                                  veth5678
```

**What's happening:** The bridge operates at Layer 2 — it forwards Ethernet frames between connected veth interfaces. Containers on the same bridge can communicate directly via their IP addresses.

### iptables NAT Rules

Docker uses iptables for:
1. **Port mapping** (DNAT): Forward traffic from host port to container port
2. **Outbound NAT** (MASQUERADE): Allow containers to access the internet

```bash
# See Docker's iptables rules
sudo iptables -t nat -L -n -v

# Port mapping rule (-p 8080:80):
# Chain DOCKER (target DNAT)
#   DNAT tcp dpt:8080 to:172.17.0.2:80

# Outbound masquerade:
# Chain POSTROUTING (target MASQUERADE)
#   MASQUERADE all -- 172.17.0.0/16 !172.17.0.0/16
```

---

## Port Mapping (-p)

```bash
docker run -p 8080:80 nginx          # host:8080 → container:80
docker run -p 127.0.0.1:8080:80 nginx  # only accessible from localhost
docker run -p 8080:80/udp nginx      # UDP port mapping
docker run -P nginx                   # map all EXPOSEd ports to random host ports
docker run -p 8080-8090:80-90 nginx  # port range mapping
```

**What's happening under the hood:**
1. Docker creates an iptables DNAT rule in the `DOCKER` chain
2. Incoming packets to `host:8080` are rewritten to `container-ip:80`
3. The response is reverse-NATed back to the client

```
Client → host:8080 → iptables DNAT → docker0 bridge → veth → container:80
                                                                    │
Client ← host:8080 ← iptables reverse NAT ← docker0 ← veth ← response
```

---

## Default Bridge vs User-Defined Bridge

```bash
# Default bridge (docker0)
docker run -d --name web1 nginx
docker run -d --name web2 nginx

# User-defined bridge
docker network create app-network
docker run -d --network app-network --name web3 nginx
docker run -d --network app-network --name web4 nginx
```

| Feature | Default bridge | User-defined bridge |
|---|---|---|
| DNS resolution | NO (must use IP or `--link`) | YES (`ping web3` works from web4) |
| Automatic DNS | No | Container name = hostname |
| Isolation | All containers share docker0 | Only containers on same network see each other |
| Connect/disconnect live | No | `docker network connect/disconnect` |
| Recommended | No | Yes, always use user-defined bridges |

**Critical:** On the default bridge, containers can only reach each other by IP address. On user-defined bridges, Docker runs an embedded DNS server that resolves container names to IP addresses.

---

## DNS Resolution in Containers

### User-Defined Bridge Networks

Docker runs an embedded DNS server at `127.0.0.11` for user-defined networks.

```bash
# Create network and containers
docker network create backend
docker run -d --network backend --name api my-api
docker run -d --network backend --name db postgres

# From the api container:
docker exec api nslookup db
# Server: 127.0.0.11
# Name:   db
# Address: 172.18.0.3
```

**What's happening:**
1. Docker configures `/etc/resolv.conf` inside the container to point to `127.0.0.11`
2. The embedded DNS server intercepts queries for container names on the same network
3. For external domains, it forwards queries to the host's DNS resolver

### DNS Aliases and Network Aliases

```bash
# Network aliases (multiple names for the same container)
docker run -d --network backend --network-alias cache --name redis-primary redis

# From another container on the same network:
ping cache      # resolves to redis-primary's IP
ping redis-primary  # also works
```

---

## Container-to-Container Communication

### Same Network (Allowed by Default)

```bash
docker network create mynet
docker run -d --network mynet --name svc-a app-a
docker run -d --network mynet --name svc-b app-b

# svc-a can reach svc-b at http://svc-b:8080
```

### Different Networks (Isolated by Default)

```bash
docker network create frontend
docker network create backend
docker run -d --network frontend --name web nginx
docker run -d --network backend --name db postgres

# web CANNOT reach db — different networks are isolated
```

### Connecting a Container to Multiple Networks

```bash
# Connect an existing container to an additional network
docker network connect frontend db

# Now db is on both frontend and backend
# It gets a second IP address (one per network)
```

```
┌─────────────────────────────────────────────┐
│                                             │
│  frontend network                           │
│  ┌──────┐            ┌──────┐              │
│  │ web  │            │  db  │ ◄── eth0     │
│  │      │            │      │    172.20.0.3 │
│  └──────┘            └──┬───┘              │
│                          │                  │
├──────────────────────────┼──────────────────┤
│                          │                  │
│  backend network         │                  │
│                     ┌────┴─┐   ┌────────┐  │
│                     │  db  │   │ worker │  │
│                     │      │   │        │  │
│                eth1 │      │   │        │  │
│           172.21.0.2└──────┘   └────────┘  │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Host Network

```bash
docker run -d --network host --name web nginx
```

The container shares the host's network namespace entirely. No network isolation, no NAT overhead. The container binds directly to host ports.

**When to use:**
- Maximum network performance (no veth + bridge overhead)
- When the container needs to see all host network traffic
- Applications that bind to many random ports (e.g., some media servers)

**Trade-off:** No port mapping. If nginx listens on 80, it occupies host port 80. Two containers can't bind the same port.

---

## Overlay Networks (Multi-Host)

Overlay networks enable container-to-container communication across Docker hosts. Used by Docker Swarm and can be integrated with Kubernetes.

```
┌─────────────────────┐     VXLAN tunnel      ┌─────────────────────┐
│  Host A             │  ◄─────────────────►  │  Host B             │
│                     │   (UDP port 4789)      │                     │
│  ┌───────────────┐  │                        │  ┌───────────────┐  │
│  │ Container A1  │  │                        │  │ Container B1  │  │
│  │ 10.0.0.2      │  │                        │  │ 10.0.0.3      │  │
│  └───────┬───────┘  │                        │  └───────┬───────┘  │
│          │          │                        │          │          │
│  ┌───────▼───────┐  │                        │  ┌───────▼───────┐  │
│  │ overlay       │  │                        │  │ overlay       │  │
│  │ br0 bridge    │  │                        │  │ br0 bridge    │  │
│  └───────┬───────┘  │                        │  └───────┬───────┘  │
│          │ VXLAN    │                        │          │ VXLAN    │
│  ┌───────▼───────┐  │                        │  ┌───────▼───────┐  │
│  │ eth0 (host)   │  │                        │  │ eth0 (host)   │  │
│  │ 192.168.1.10  │  │                        │  │ 192.168.1.11  │  │
│  └───────────────┘  │                        │  └───────────────┘  │
└─────────────────────┘                        └─────────────────────┘
```

**What's happening:** VXLAN (Virtual Extensible LAN) encapsulates Layer 2 frames inside UDP packets. Container A1 sends a packet to 10.0.0.3 — the overlay driver wraps it in a VXLAN header and sends it via UDP to Host B, which unwraps it and delivers it to Container B1. The containers think they're on the same LAN.

```bash
# Create an overlay network (requires Swarm mode)
docker network create --driver overlay --attachable my-overlay

# Use with Swarm services
docker service create --network my-overlay --name web nginx
```

---

## Macvlan Networks

Assigns a real MAC address to the container, making it appear as a physical device on your network.

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

docker run -d --network my-macvlan --ip=192.168.1.50 --name web nginx
```

**Use case:** Legacy applications that expect to be directly on the LAN, or when you need the container to be addressable by other devices on the physical network without NAT.

---

## Network Troubleshooting

### Essential Commands

```bash
# Inspect a network (see connected containers, subnet, gateway)
docker network inspect bridge

# See container's IP address
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-container

# DNS resolution test
docker exec my-container nslookup some-service

# Connectivity test
docker exec my-container ping -c 3 other-container
docker exec my-container curl -v http://api:8080/health

# See what ports are mapped
docker port my-container

# Network namespaces from host
nsenter --net=/proc/$(docker inspect --format '{{.State.Pid}}' my-container)/ns/net ip addr

# Capture traffic on docker0 bridge
sudo tcpdump -i docker0 -n port 80

# See iptables NAT rules Docker created
sudo iptables -t nat -L DOCKER -n -v
```

### Common Problems and Solutions

```bash
# Problem: container can't resolve DNS
# Check: is /etc/resolv.conf correct?
docker exec my-container cat /etc/resolv.conf
# Fix: specify DNS server
docker run --dns 8.8.8.8 my-image

# Problem: containers on same network can't talk
# Check: are they actually on the same network?
docker inspect --format '{{json .NetworkSettings.Networks}}' container-a
docker inspect --format '{{json .NetworkSettings.Networks}}' container-b

# Problem: port already in use
# Error: "bind: address already in use"
# Check what's using the port
sudo lsof -i :8080
# Fix: use a different host port
docker run -p 8081:80 nginx

# Problem: container can't reach the internet
# Check: is IP forwarding enabled?
sysctl net.ipv4.ip_forward
# Fix: enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
```

---

## Practical Examples / Real-World Patterns

### Microservices Network Isolation

```bash
# Create separate networks for frontend and backend
docker network create frontend-net
docker network create backend-net

# Frontend services
docker run -d --network frontend-net --name nginx-proxy -p 80:80 nginx
docker run -d --network frontend-net --name web-app my-web-app

# Backend services (isolated from internet)
docker run -d --network backend-net --name api my-api
docker run -d --network backend-net --name db -e POSTGRES_PASSWORD=secret postgres

# API bridge: connect api to frontend so nginx can reach it
docker network connect frontend-net api

# Result: nginx-proxy can reach api, but NOT db
#         api can reach both nginx-proxy and db
#         db can only reach api
```

### Service Mesh Pattern with Docker Networks

```bash
# Each service pair gets its own network
docker network create orders-to-payments
docker network create orders-to-inventory
docker network create payments-to-notifications

# Services are connected only to networks they need
docker run -d --network orders-to-payments --name orders orders-svc
docker network connect orders-to-inventory orders

docker run -d --network orders-to-payments --name payments payments-svc
docker network connect payments-to-notifications payments

docker run -d --network orders-to-inventory --name inventory inventory-svc
docker run -d --network payments-to-notifications --name notifications notifications-svc
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Bridge (default) | veth pairs + linux bridge + iptables NAT. No DNS on default bridge |
| User-defined bridge | Always use this. Automatic DNS, container name resolution, isolation |
| Host network | No isolation, no NAT. Container uses host's network stack directly |
| Overlay | Multi-host via VXLAN encapsulation. Requires Swarm or external KV store |
| Port mapping | iptables DNAT rules. `-p host:container` |
| DNS | Embedded DNS at 127.0.0.11 on user-defined networks |
| Isolation | Different networks = no connectivity by default. Explicit `network connect` to bridge |
| Troubleshooting | `docker network inspect`, `nsenter`, `tcpdump -i docker0` |
