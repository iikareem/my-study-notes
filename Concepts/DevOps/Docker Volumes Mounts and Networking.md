---
tags:
  - containers
  - devops
  - docker
  - networking
  - volumes
---

# Docker: Volumes, Mounts & Networking — Complete Guide

---

## Table of Contents

1. [Storage Overview](#storage-overview)
2. [Volumes](#1-volumes)
3. [Bind Mounts](#2-bind-mounts)
4. [tmpfs Mounts](#3-tmpfs-mounts)
5. [Networking Overview](#networking-overview)
6. [Bridge Network](#1-bridge-network-default)
7. [Host Network](#2-host-network)
8. [None Network](#3-none-network)
9. [Overlay Network](#4-overlay-network)
10. [Macvlan Network](#5-macvlan-network)
11. [Quick Comparison Tables](#quick-comparison-tables)

---

## Storage Overview

Docker containers have an **ephemeral filesystem** — when a container dies, its data dies with it. To persist or share data, Docker offers three storage mechanisms:

```
┌─────────────────────────────────────────────────────┐
│                   Docker Host                        │
│                                                      │
│  ┌─────────────┐   ┌─────────────┐   ┌───────────┐ │
│  │   Volume    │   │ Bind Mount  │   │   tmpfs   │ │
│  │ (managed by │   │ (any host   │   │ (RAM only │ │
│  │   Docker)   │   │   path)     │   │ no disk)  │ │
│  └──────┬──────┘   └──────┬──────┘   └─────┬─────┘ │
│         │                 │                 │        │
│         └────────────┬────┘                 │        │
│                      ▼                      ▼        │
│               ┌─────────────────────────────────┐   │
│               │         Container               │   │
│               │    /app/data   /tmp   /run      │   │
│               └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## 1. Volumes

### What is it?

A **Volume** is storage that Docker itself creates and manages, stored under `/var/lib/docker/volumes/` on the host. Containers don't need to know where this is — Docker handles it.

### Prerequisites

- Docker installed (no special setup needed)
- Optionally: volume driver plugins for remote/cloud storage (NFS, AWS EBS, etc.)

### Default Behavior

- A volume persists **even after the container is removed**
- Multiple containers can share the same volume
- Docker initializes volume contents from the container image if the volume is empty on first mount

### Key Characteristics

|Property|Detail|
|---|---|
|Managed by|Docker|
|Location on host|`/var/lib/docker/volumes/<name>/_data`|
|Persists after container removal|✅ Yes|
|Shareable across containers|✅ Yes|
|Works on Linux, Mac, Windows|✅ Yes|
|Best for production|✅ Yes|
|Performance|Excellent (native filesystem)|

---

### Commands

```bash
# Create a named volume
docker volume create mydata

# List all volumes
docker volume ls

# Inspect a volume (see where it lives)
docker volume inspect mydata

# Remove a volume
docker volume rm mydata

# Remove ALL unused volumes (dangerous!)
docker volume prune
```

---

### Examples

**Example 1 — Basic named volume (database persistence)**

```bash
# Run a PostgreSQL container with a named volume
docker run -d \
  --name postgres_db \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \   # <-- volume:container_path
  postgres:16
```

> Even if you do `docker rm postgres_db`, your database files survive in the `pgdata` volume.

---

**Example 2 — Share a volume between two containers**

```bash
# Writer container
docker run -d \
  --name writer \
  -v shared_logs:/app/logs \
  myapp:latest

# Reader container (same volume!)
docker run -d \
  --name log_viewer \
  -v shared_logs:/logs:ro \   # :ro = read-only
  busybox tail -f /logs/app.log
```

---

**Example 3 — Using Docker Compose**

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    volumes:
      - appdata:/app/data

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  appdata:   # Docker manages this
  pgdata:    # Docker manages this
```

---

**Example 4 — Anonymous volume (one-time, auto-named)**

```bash
docker run -d \
  -v /app/cache \   # No name — Docker auto-generates one
  myapp:latest
```

> Anonymous volumes are harder to reference later. Use named volumes for anything important.

---

## 2. Bind Mounts

### What is it?

A **Bind Mount** maps a **specific path on your host machine** directly into the container. You control where the data lives — Docker doesn't manage it.

### Prerequisites

- The host path **must exist** before running the container (or Docker will create it as a directory)
- You need read/write permissions on the host path
- Absolute paths required (no `~/` shorthand in `docker run`; use `$(pwd)` instead)

### Default Behavior

- The host directory content **overrides** whatever was in the container at that path
- Changes in the container are instantly visible on the host (and vice versa)
- Deleting the container does **not** delete host files

### Key Characteristics

|Property|Detail|
|---|---|
|Managed by|You (the developer)|
|Location on host|Anywhere you specify|
|Persists after container removal|✅ Yes (it's just your host files)|
|Shareable across containers|✅ Yes|
|Great for local development|✅ Yes|
|Security risk if misconfigured|⚠️ Yes (can expose sensitive host paths)|
|Performance on Mac/Windows|⚠️ Slower (due to filesystem translation)|

---

### Examples

**Example 1 — Mount source code for live development**

```bash
docker run -d \
  --name dev_server \
  -p 3000:3000 \
  -v $(pwd)/src:/app/src \   # host path : container path
  node:20 npm run dev
```

> Edit files in `./src` on your machine → the container sees changes instantly. No rebuild needed.

---

**Example 2 — Mount a config file (read-only)**

```bash
docker run -d \
  --name nginx \
  -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \   # :ro = read-only
  nginx:latest
```

---

**Example 3 — Mount multiple paths**

```bash
docker run -d \
  -v $(pwd)/app:/app \          # source code
  -v $(pwd)/config:/etc/app \   # config files
  -v /var/log/myapp:/app/logs \ # log output
  myapp:latest
```

---

**Example 4 — Docker Compose bind mount**

```yaml
services:
  app:
    image: myapp:latest
    volumes:
      - type: bind
        source: ./src          # relative to docker-compose.yml
        target: /app/src
      - ./config:/etc/app:ro   # shorthand syntax
```

---

### Volume vs Bind Mount — When to Use Which

|Situation|Use|
|---|---|
|Persist database data|**Volume**|
|Share config between containers|**Volume** or **Bind Mount**|
|Live code reload in development|**Bind Mount**|
|CI/CD pipeline artifacts|**Bind Mount**|
|Production app data|**Volume**|
|Sensitive host files you control|**Bind Mount** with `:ro`|

---

## 3. tmpfs Mounts

### What is it?

A **tmpfs** mount stores data **in the host's RAM** only — it is never written to disk. The data vanishes when the container stops.

### Prerequisites

- Linux host only (not supported on Docker Desktop for Mac/Windows)
- Sufficient RAM available

### Default Behavior

- Data is lost when the container stops or restarts
- Not shareable between containers

### Examples

```bash
# Store secrets or session data in RAM only
docker run -d \
  --name secure_app \
  --tmpfs /run/secrets:size=64m,mode=0700 \
  myapp:latest
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    tmpfs:
      - /run/secrets
      - /tmp
```

> **Use case:** Storing API tokens, session caches, or any sensitive data that must never touch disk.

---

---

## Networking Overview

Docker containers are isolated by default. Networking controls **how containers talk to each other and to the outside world**.

```
┌───────────────────────────────────────────────────────────┐
│                        Docker Host                         │
│                                                            │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐ │
│  │  Container  │   │  Container  │   │    Container    │ │
│  │     A       │   │     B       │   │       C         │ │
│  └──────┬──────┘   └──────┬──────┘   └────────┬────────┘ │
│         │                 │                    │           │
│         └────────┬────────┘                   none         │
│                  │                                         │
│           ┌──────┴──────┐                                  │
│           │docker0 bridge│  (172.17.0.0/16 by default)    │
│           └──────┬───────┘                                 │
│                  │                                         │
│           ┌──────┴───────┐                                 │
│           │  Host Network │                                │
│           │  Interface    │                                │
│           └──────┬────────┘                                │
└──────────────────┼─────────────────────────────────────────┘
                   │
              The Internet
```

---

## 1. Bridge Network (Default)

### What is it?

Bridge is Docker's **default network**. Containers on the same bridge can talk to each other by container name. Containers are isolated from the host network.

### Prerequisites

- Nothing special — it's automatic
- For custom bridges: `docker network create`

### Default Behavior

- Every container you start without `--network` gets placed on `bridge` (the default bridge)
- On the **default bridge**, containers communicate by IP only (no DNS)
- On a **custom bridge**, containers communicate by container name (DNS works ✅)

> **Best practice:** Always create a custom bridge network. Never rely on the default `bridge`.

---

### Examples

**Example 1 — Default bridge (not recommended for production)**

```bash
docker run -d --name app1 myapp:latest
docker run -d --name app2 myapp:latest

# app1 and app2 can reach each other only by IP
# docker inspect app1 | grep IPAddress  → e.g., 172.17.0.2
```

---

**Example 2 — Custom bridge (recommended)**

```bash
# Create a custom bridge network
docker network create mynet

# Run containers on it
docker run -d --name backend --network mynet mybackend:latest
docker run -d --name frontend --network mynet myfrontend:latest

# frontend can reach backend by name:
# curl http://backend:8080/api
```

---

**Example 3 — Docker Compose (auto-creates a custom bridge)**

```yaml
services:
  frontend:
    image: myfrontend:latest
    ports:
      - "3000:3000"

  backend:
    image: mybackend:latest
    ports:
      - "8080:8080"

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret

# Compose auto-creates a bridge network named "myproject_default"
# All three services can reach each other by service name
# backend can call: postgres://db:5432
```

---

**Example 4 — Expose a port to the host**

```bash
# -p host_port:container_port
docker run -d -p 8080:80 nginx:latest

# Access at http://localhost:8080
```

---

## 2. Host Network

### What is it?

The container **shares the host's network stack directly** — no isolation, no NAT, no port mapping needed.

### Prerequisites

- Linux only (not supported on Docker Desktop for Mac/Windows)
- Use with caution — container can bind to any host port

### Default Behavior

- Container uses the host's IP address
- Port conflicts are possible if multiple containers need the same port
- Best performance (no NAT overhead)

### Example

```bash
docker run -d \
  --network host \
  --name fast_server \
  nginx:latest

# nginx now listens on host's port 80 directly
# No -p needed — and -p would be ignored anyway
```

> **Use case:** High-performance networking tools, monitoring agents (Prometheus node exporter), or when you need the container to see the host's real network interfaces.

---

## 3. None Network

### What is it?

The container has **no network access at all** — completely isolated.

### Prerequisites

- Nothing special

### Example

```bash
docker run -d \
  --network none \
  --name isolated_job \
  myprocessor:latest

# Container cannot reach the internet or any other container
# Useful for data processing that must be air-gapped
```

> **Use case:** Security-sensitive batch jobs, offline data transformation, or sandboxing untrusted code.

---

## 4. Overlay Network

### What is it?

Overlay networks **span multiple Docker hosts** (machines), letting containers on different servers communicate as if they were on the same local network. Used with **Docker Swarm** or **Kubernetes**.

### Prerequisites

- Docker Swarm mode initialized (`docker swarm init`)
- Open ports between hosts: TCP 2377, TCP/UDP 7946, UDP 4789
- All hosts must be Swarm nodes

### Example

```bash
# Initialize swarm on manager node
docker swarm init

# Create an overlay network
docker network create \
  --driver overlay \
  --attachable \
  myoverlay

# Deploy a service across the swarm
docker service create \
  --name webapp \
  --replicas 3 \
  --network myoverlay \
  myapp:latest
```

```yaml
# docker-compose.yml for Swarm (deploy with docker stack deploy)
services:
  app:
    image: myapp:latest
    networks:
      - myoverlay
    deploy:
      replicas: 3

networks:
  myoverlay:
    driver: overlay
```

> **Use case:** Multi-server deployments, microservices spread across a cluster, Docker Swarm production environments.

---

## 5. Macvlan Network

### What is it?

Assigns a **real MAC address** to a container, making it appear as a physical device on your LAN. The container gets its own IP from your router — just like a real machine.

### Prerequisites

- Linux host with a network interface that supports promiscuous mode
- Your network must allow multiple MAC addresses per port (check switch settings)
- Usually requires root/admin access

### Example

```bash
# Create a macvlan network mapped to your host interface
docker network create \
  --driver macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --opt parent=eth0 \
  mymacvlan

# Run a container with its own LAN IP
docker run -d \
  --network mymacvlan \
  --ip 192.168.1.50 \
  --name lan_device \
  myapp:latest
```

> **Use case:** IoT devices, legacy apps that expect a real LAN IP, network monitoring tools that need to see raw traffic.

---

## Quick Comparison Tables

### Storage

|Type|Managed by|Persists?|Shareable?|Best for|
|---|---|---|---|---|
|**Volume**|Docker|✅ Yes|✅ Yes|Production data, databases|
|**Bind Mount**|You|✅ Yes|✅ Yes|Local dev, config injection|
|**tmpfs**|OS RAM|❌ No|❌ No|Secrets, temp/session data|

---

### Networking

|Driver|Scope|Isolation|Port Mapping|Best for|
|---|---|---|---|---|
|**bridge** (default)|Single host|Partial|Required|Most apps, local dev|
|**host**|Single host|None|Not needed|High-perf, Linux tools|
|**none**|Single host|Full|N/A|Sandboxing, batch jobs|
|**overlay**|Multi-host|Partial|Optional|Swarm, microservices clusters|
|**macvlan**|Single host|None (LAN)|Not needed|IoT, legacy LAN apps|

---

### Default Summary

|Concept|Default|
|---|---|
|Container storage|Ephemeral (in-container layer, lost on delete)|
|Volume location|`/var/lib/docker/volumes/`|
|Network for new containers|`bridge` (the default bridge network)|
|DNS between containers|Only on **custom** bridge networks|
|Port exposure|Nothing exposed unless you use `-p`|
|Network on `docker compose up`|Custom bridge auto-created per project|

---

## Full Real-World Example (App + DB + Reverse Proxy)

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"           # Expose to host
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro   # Bind mount (config)
    networks:
      - frontend

  app:
    image: myapp:latest
    networks:
      - frontend          # Can receive from nginx
      - backend           # Can reach db
    volumes:
      - uploads:/app/uploads   # Named volume (user uploads)

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data   # Named volume (DB files)
    networks:
      - backend           # Isolated — nginx can't directly reach DB

volumes:
  pgdata:     # Persists DB across restarts
  uploads:    # Persists user files

networks:
  frontend:   # nginx <-> app
  backend:    # app <-> db (db not reachable from nginx)
```

This setup:

- Uses **named volumes** for all persistent data
- Uses a **bind mount** for the nginx config
- Uses **two custom bridge networks** for network segmentation (nginx can't reach the DB directly)
- Exposes **only port 80** to the outside world

## See also

- [[Docker]] · [[VMs vs Containers]] · [[Virtualization vs Containerization]]
