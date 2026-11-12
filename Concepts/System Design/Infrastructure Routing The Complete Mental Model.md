# Infrastructure Routing — The Complete Mental Model

> **The core confusion:** Reverse proxy, load balancer, API gateway, and service discovery all touch traffic routing — so they _feel_ interchangeable. They're not. Each solves a different layer of the problem. This guide separates them cleanly, goes deep on what a reverse proxy actually does at the network level, then shows how they compose.

---

## Table of Contents

1. [The Mental Model Before the Details](#1-the-mental-model-before-the-details)
2. [Reverse Proxy — Deep Dive](#2-reverse-proxy--deep-dive)
    - Why it's one-to-one, not one-to-many
    - The Slow Client Problem — Kernel-Level Precision
    - Connection pooling
    - Payload compression
    - TLS termination
    - Why these concerns are offloaded
    - Does a reverse proxy do routing?
3. [Load Balancer — L4 vs L7](#3-load-balancer--l4-vs-l7)
4. [API Gateway](#4-api-gateway)
5. [Service Discovery](#5-service-discovery)
6. [How They All Work Together](#6-how-they-all-work-together)

---

## 1. The Mental Model Before the Details

**Definition:** Infrastructure routing is a stack of layers — each with one job — that sit between a client and your service code. Tools overlap in practice; the concepts do not.

Think of it as **layers of concern**, not competing tools:

```
Client
  │
  ▼
[ API Gateway ]        ← Business policy layer: auth, rate-limit, routing rules
  │
  ▼
[ Load Balancer ]      ← Distribution layer: spread traffic across N instances
  │
  ▼
[ Reverse Proxy ]      ← Network layer: handle the connection itself
  │
  ▼
[ Service ]            ← Your code
  │
  └──► [ Service Discovery ]  ← How services find each other (orthogonal to all above)
```

The reason they feel interchangeable: **Nginx can do all four**. But that's an implementation accident, not a conceptual truth. A hammer can be used as a paperweight — that doesn't mean hammers and paperweights are the same thing.

---

# 2. Reverse Proxy — Deep Dive

**Definition:** A reverse proxy sits in front of your servers, terminates the client connection, handles network mechanics (slow clients, TLS, compression, pooling), and forwards traffic to an upstream — so the client thinks it's talking directly to your service, but isn't.

---

## What "Reverse" Means

A **forward proxy** sits in front of clients — it represents clients to the internet (VPN, corporate proxy, Squid).

A **reverse proxy** sits in front of servers — it represents your servers to clients.

```
Client ──► Reverse Proxy ──► Your Service
           (nginx, caddy,
            haproxy, envoy)
```

---

## Why It's Fundamentally One-to-One

In its pure form, a reverse proxy maps one client connection to one upstream connection — it handles the connection mechanics itself, not which of N instances to route to.

Each client TCP connection terminates at the proxy. The proxy then opens (or reuses) a separate TCP connection to the upstream service:

```
Client TCP connection:  [Client]──────────[Proxy]
                                  ↕ terminated here
Upstream TCP connection:          [Proxy]──────────[Service]
```

This is the key: **two separate TCP connections with the proxy in the middle.** This decoupling is what enables every feature below.

> A reverse proxy in its purest form maps one client to one upstream. It's not inherently about picking between 10 instances — that's load balancing. The proxy's job is to handle the _mechanics of the connection_ and hand it off cleanly.

---

## The Slow Client Problem

### Definition

A slow client reads or writes bytes much slower than the service can produce or consume them. The connection stays open and holds resources while waiting.

The key insight here: **the primary cost is kernel resources, not CPU time.** To understand why, we need to look at what actually happens inside the OS.

---

### The OS Layer — `struct sock` and the Send Buffer

When your Node.js service calls `socket.write(data)`, the bytes are copied into the kernel's TCP send buffer. After that copy, **your app is done** — the OS owns those bytes.

Inside the kernel, every TCP connection is represented by a `struct sock`:

```c
struct sock {
    struct sk_buff_head  sk_write_queue;   // outgoing segments waiting to be sent
    int                  sk_wmem_queued;   // bytes currently in the send buffer
    int                  sk_sndbuf;        // max send buffer size (cap)
}
```

The TCP stack reads from `sk_write_queue` and sends segments — but only as fast as the **remote client's TCP receive window** allows. A slow client keeps its receive window small because it can't drain data fast enough. So the kernel stalls: segments can't go out, bytes keep piling up in `sk_write_queue`.

Once `sk_wmem_queued` hits `sk_sndbuf` (the buffer cap), the send buffer is full. The next `write()` from your app will either:

- **Block** (blocking socket) — the calling thread sleeps
- **Return `EAGAIN`** (non-blocking socket, which Node.js uses) — the write fails, retry later

So the first cost is **kernel memory**: one `struct sock` with a maxed-out send buffer, held open per slow client. At 10,000 slow clients, that's 10,000 structs sitting in kernel memory — all full, all waiting.

---

### Where Node.js / The Event Loop Gets Involved

Node.js uses non-blocking sockets, so a `write()` that can't complete yet won't freeze your app — it just returns `EAGAIN` instead of blocking. But something still has to watch that socket and retry the write once the client frees up some space. That's libuv's job:

```
socket.write(data)
  └─► kernel copies bytes to sk_write_queue
        └─► client's TCP window is full → kernel can't send
              └─► sk_wmem_queued hits sk_sndbuf → buffer is full
                    └─► write() returns EAGAIN

libuv (seeing EAGAIN):
  └─► registers the socket fd with epoll, watching for EPOLLOUT
        └─► this fd is now a parked watcher in the event loop
              └─► event loop moves on — no CPU spent waiting
                    └─► eventually: client drains a little
                          └─► kernel fires EPOLLOUT
                                └─► libuv retries write()
```

The important distinction: **the event loop is not burning CPU.** It's not spinning or polling. It's just holding a registered file descriptor — a passive record saying "tell me when this socket is writable again." Epoll does the actual waiting at the kernel level. The event loop only wakes up when there's something to do.

So with 10,000 slow clients, your Node.js process isn't overwhelmed in terms of CPU. What it's holding is:

| Resource                   | What's consumed                                | Where             |
| -------------------------- | ---------------------------------------------- | ----------------- |
| `struct sock` send buffers | Kernel memory (`sk_sndbuf` per socket)         | Kernel            |
| File descriptors           | One per open connection, capped by `ulimit -n` | Kernel            |
| epoll watchers             | One per `EAGAIN` socket registered in libuv    | Userspace (libuv) |
| CPU                        | Nearly zero — no active work                   | —                 |

The event loop ceiling you hit isn't CPU exhaustion — it's the **fd limit** (`ulimit -n`) and **kernel send buffer memory**. You run out of file descriptors or RAM long before the event loop thread gets busy.

---

### What the Kernel Is Actually Paying — No Proxy, No CPU, So What?

This is the part that trips people up. If slow clients don't cost CPU, what exactly is the cost?

**Three things accumulate per slow client, all in kernel space:**

**1. Kernel memory for the send buffer (`sk_sndbuf`)** Each socket has a send buffer. Default sizes vary by OS (often 4KB–128KB, sometimes auto-tuned higher). With a stuck client, that buffer fills and stays full. Memory can't be reclaimed until the connection closes. At scale, this is your first real pressure point.

**2. A file descriptor** Every open TCP connection consumes one fd. The OS enforces a hard cap via `ulimit -n` (commonly 1024 by default, tunable to millions but still a ceiling). A process that hits its fd limit can't accept new connections — not because it's busy, but because it's _full_.

**3. Kernel bookkeeping structures** Beyond `struct sock`, the kernel maintains routing tables, connection state machines (TIME_WAIT, ESTABLISHED), and retransmission timers per connection. Each slow client that just sits there doing nothing still occupies a slot in all of these.

**The shape of the problem:**

```
10,000 slow clients × (send buffer + 1 fd + kernel state)
= memory pressure + fd exhaustion
= new clients rejected
= your service appears down
```

And crucially — your service already did the work. It already ran the database query, rendered the response, paid the CPU cost. Now it's stuck holding all these kernel resources just to drip bytes to a client that's reading at 1KB/s. That's the mismatch: **expensive resources doing a cheap job.**

---

### The Asymmetry: Request vs Response

The response direction is more dangerous than the request direction:

||Request (client → service)|Response (service → client)|
|---|---|---|
|What is slow|Client uploading / sending headers|Client reading the response|
|Kernel buffer|`sk_rcvbuf` absorbs passively|`sk_wmem_queued` fills → `EAGAIN`|
|Severity|Real, but usually bounded|Genuinely dangerous|
|Why|Request eventually ends; OS cushions inbound|Client can read 1 byte/sec indefinitely|
|Under load|Limited window|fd limit + kernel send-buffer memory|

On the **request side**, the OS receive buffer (`sk_rcvbuf`) absorbs slow incoming bytes passively. The service isn't even reading yet. There's a natural cushion and a natural end — uploads finish.

On the **response side**, your service has already done the work, already paid the CPU cost, and now can't release the connection because the client hasn't finished reading. It's stuck holding kernel resources whose only remaining job is to wait.

---

### What the Proxy Does — At the Kernel Level

The proxy inserts a second TCP connection between itself and the service:

```
Client ←──(slow, internet)──► Proxy ←──(fast, LAN)──► Service
         struct sock A                  struct sock B
         sk_sndbuf filling              drains instantly
```

The service sends its response into **struct sock B** — the connection to the proxy, sitting on the same LAN or loopback interface. That connection drains at full speed. The service's `write()` never returns `EAGAIN`. No fd parks in libuv. **The service is free immediately.**

Now the proxy holds **struct sock A** — the slow client connection. But this is exactly what the proxy is built for.

**Why the proxy handles it cheaply:**

A reverse proxy like nginx or HAProxy has one job: hold connections and move bytes efficiently. It has no business logic, no database queries, nothing competing for its resources. This means:

- Its entire codebase is tuned around epoll — holding thousands of waiting fds is its normal operating mode, not a stress condition
- It uses `sendfile()` to move bytes directly from buffer to socket without copying through userspace
- Its kernel parameters (`sk_sndbuf`, fd limits, socket options) are tuned specifically for high connection counts
- When it's waiting on a slow client, it's not delaying anything important — waiting _is_ its job

**The proxy converts an expensive wait into a cheap wait.** The kernel mechanics are identical — same `struct sock`, same epoll, same fd. What changes is _who is holding it_. Your service, whose resources are meant for business logic, is replaced by a proxy whose resources exist specifically to wait.

---

### Corrected Summary

The common explanation ("event loop saturated with sockets") is directionally right but imprecise. The accurate picture:

- `socket.write()` hands bytes to the kernel — the app is decoupled from actual transmission
- A slow client fills `sk_write_queue` in `struct sock` — this is **kernel memory pressure**
- When `sk_sndbuf` is full, `write()` returns `EAGAIN` — libuv parks an epoll watcher passively
- The event loop accumulates passive watchers, not CPU load
- The real ceilings are **fd limit** (`ulimit -n`) and **kernel send buffer memory** — not event-loop throughput
- The proxy offloads all of this: the service's connection drains instantly, fd released, no `EAGAIN`, no parked watcher

> **Takeaway:** Never expose your service directly to the internet — even behind a load balancer. The slow-client problem is about kernel resource exhaustion, and it exists regardless of how many instances you have.

---

## Connection Pooling

**Definition:** Connection pooling keeps a set of warm, reusable TCP (and often TLS) connections from the proxy to upstream services, so each client request does not pay a full handshake cost.

Without a proxy, every HTTP/1.1 request may involve:

1. DNS resolution
2. TCP handshake (1 RTT)
3. TLS handshake (1–2 RTT)
4. HTTP request/response
5. TCP teardown

The proxy maintains a pool of persistent connections to upstream services instead:

```
[Client A] ──► [Proxy] ─────────────────────────────► [Service]
[Client B] ──► [Proxy]    (pool of 20 keep-alive       [Service]
[Client C] ──► [Proxy]     connections, reused          [Service]
[Client D] ──► [Proxy]     across thousands of          [Service]
                            client requests)
```

This:

- Eliminates per-request handshake latency
- Reduces OS fd churn on the service side
- **Controls concurrency seen by the service** — the pool size acts as a natural backpressure valve. A pool of 50 connections means your service will never see more than 50 concurrent requests from the proxy, regardless of how many clients hit it.

In Node.js terms, `http.globalAgent.maxSockets = 50` is a crude version of this — but a proxy does it better, with health checks, circuit breaking, and per-upstream tuning.

---

## Payload Compression

**Definition:** Payload compression shrinks HTTP response bodies (gzip, brotli, zstd) before they reach the client, trading CPU for bandwidth — best done at the proxy, not in service code.

Compression is CPU-intensive. If your Node.js service compresses every response, it burns CPU on work that has nothing to do with your business logic — and under load, that CPU contention directly cuts into request throughput.

```
Service ──► raw JSON (fast, no CPU cost) ──► Proxy ──► gzip/brotli ──► Client
```

The proxy handles content negotiation too — serving gzip to older clients and brotli to modern ones, without the service knowing or caring. It can also cache compressed versions of static responses so compression only happens once per unique body.

---

## TLS Termination

**Definition:** TLS termination ends the encrypted client session at the proxy, so upstream services speak plain HTTP (or internal mTLS) while certs, handshakes, and cipher policy stay centralized.

```
Client ──(TLS)──► Proxy ──(plain HTTP or mTLS)──► Service
                  ↑
            TLS terminates here
```

**What this buys you:**

- **Centralized certificate management** — update one cert in one place (or use Let's Encrypt auto-renewal via Caddy/cert-manager) instead of distributing certs to every service instance
- **Session resumption managed at the proxy** — session tickets and session IDs handled once, not per-service
- **Centralized cipher policy** — TLS 1.2 minimum, disabled weak ciphers, HSTS headers — all in one config
- **OCSP stapling** — the proxy pre-fetches certificate revocation status and staples it to the handshake; clients skip the extra round trip; your service has no awareness of any of this

---

## Why These Concerns Are Offloaded — The Decoupling Argument

All of the above (slow clients, connection pooling, compression, TLS) are **network-level concerns**. They have nothing to do with your business logic.

**Without a proxy:**

- Business logic is entangled with network mechanics
- Changing TLS policy means deploying every service
- A bug in compression code can crash a service that's responsible for charging customers
- Load testing your service also tests your TLS/compression stack, polluting benchmarks

**With a proxy:**

- Your service is a pure function: `request → business logic → response`
- Network policy changes are operational, not development changes
- You can test, tune, and upgrade the proxy independently of services
- A service becomes trivially portable — `node server.js` locally works identically, no TLS or compression needed

This is the same reason you don't put your database connection string in your HTML — **separation of concerns across layers.**

---

## Does a Reverse Proxy Do Routing?

**Short answer:** It can, but routing is not its core identity.

In its simplest form, a reverse proxy points at one upstream. It handles the connection; it doesn't choose between upstreams. However, most real proxy software (Nginx, Caddy, Traefik, HAProxy) includes routing because it's the next natural concern once you have a proxy in place:

```nginx
location /api/ {
    proxy_pass http://api-service;
}
location /static/ {
    root /var/www;
}
```

This is path-based routing — the proxy is now also doing API Gateway work.

|Concern|Belongs to|
|---|---|
|Handle TCP connection lifecycle|Reverse Proxy|
|Buffer slow clients|Reverse Proxy|
|TLS termination|Reverse Proxy|
|Connection pooling|Reverse Proxy|
|Compression|Reverse Proxy|
|Route /api to service A, /admin to service B|API Gateway (or proxy acting as one)|
|Spread traffic across 5 instances of service A|Load Balancer|
|Authenticate the request|API Gateway|
|Rate limit by user ID|API Gateway|

A proxy doing routing doesn't stop being a proxy — it's just also doing gateway work. The distinction matters when thinking about _where to put logic_, not which binary to run.
## 3. Load Balancer 

**Definition:** A load balancer distributes traffic across **multiple instances of the same service**. Its core question is: _which instance gets this request?_

### L4 — Transport Layer (TCP/UDP)

**Definition:** An L4 load balancer balances at the transport layer — it sees IP/port/TCP flags only, not HTTP URLs, headers, or bodies.

Operates at OSI Layer 4. It sees:

- Source IP, destination IP
- Source port, destination port
- TCP flags (SYN, ACK, etc.)

It **does not** see HTTP. It doesn't know what URL is being requested, what headers are present, or what's in the body.

```
Client TCP packet:
  src: 1.2.3.4:54321
  dst: 10.0.0.1:443
  
L4 LB sees: a TCP connection to port 443
L4 LB does: picks an upstream by IP hash or round-robin, forwards raw TCP stream
```

**How L4 balance works (NAT mode):**

```
Client ──► L4 LB (10.0.0.1:443) ──► rewrites dst to 10.0.1.5:443 (instance 3)
```

The L4 LB rewrites the destination IP (DNAT) and forwards the packet. It doesn't terminate TCP — the client's TCP connection goes all the way to the upstream instance.

**Algorithms at L4:**

- **Round-robin** — distribute evenly, sequentially
- **IP hash** — same source IP always goes to same instance (session stickiness)
- **Least connections** — pick the instance with fewest open connections

**Use L4 when:**

- You need raw TCP throughput (database connections, MQTT, custom protocols)
- You can't or don't want to terminate TLS at the LB
- You need the upstream to see the real client IP (though this can be solved with Proxy Protocol)
- Ultra-low latency is required (L4 adds ~microseconds vs L7's milliseconds)

**L4 blind spots:**

- Can't make routing decisions based on URL, headers, or cookies
- Can't do health checks beyond TCP connect (it doesn't know if your HTTP service is returning 500s)
- Can't do request-level stickiness (only connection-level)

---

### L7 — Application Layer (HTTP/HTTPS)

**Definition:** An L7 load balancer terminates TCP/TLS, parses HTTP, and can route or balance using method, URL, headers, cookies, and HTTP health checks.

Operates at OSI Layer 7. It **terminates** the TCP+TLS connection and parses the HTTP protocol. It sees everything: method, URL, headers, body (if it reads it).

```
Client ──(TLS)──► L7 LB ──(HTTP)──► Instance A
                    ↑                Instance B
              full HTTP parse        Instance C
```

**What L7 unlocks:**

**Content-based routing:**

```
GET /api/users  ──► user-service instances
GET /api/orders ──► order-service instances
```

**Header-based routing:**

```
X-Beta-User: true ──► canary instances (5% of fleet)
                  else ──► stable instances (95% of fleet)
```

**Sticky sessions by cookie:** The LB reads `Set-Cookie: SESSION=abc123` on the first response, then routes `Cookie: SESSION=abc123` to the same instance. L4 can't do this.

**HTTP health checks:** The LB sends `GET /health` every 10 seconds. If a service returns 500 or times out, it's removed from rotation. L4 can only check if TCP port is open.

**Retries:** If upstream returns a connection error, the L7 LB can retry on a different instance — transparently to the client. L4 can't retry because it doesn't buffer the request.

**Algorithms at L7:**

- **Round-robin** — still available
- **Least requests** — pick instance handling fewest in-flight HTTP requests (more accurate than least connections for HTTP/2)
- **Random with 2 choices (P2C)** — pick 2 random, take the less-loaded one. Better distribution than pure round-robin under variable request duration.
- **Weighted** — instance A gets 70%, instance B gets 30% (canary deployments)

**L7 costs:**

- Higher CPU (must parse HTTP)
- Higher latency (TLS termination + HTTP parsing)
- More memory (buffers the full request)

---

### L4 vs L7 Decision Table

| Scenario                                     | Use                           |
| -------------------------------------------- | ----------------------------- |
| HTTP microservices                           | L7                            |
| PostgreSQL, MySQL connection balancing       | L4                            |
| gRPC                                         | L7 (HTTP/2 aware)             |
| WebSockets (long-lived)                      | L4 or L7 with upgrade support |
| Blue/green deployments with weighted traffic | L7                            |
| Raw UDP (DNS, game servers)                  | L4                            |
| You need URL-based routing                   | L7                            |
| You need the upstream to see real client IP  | L4 (or L7 + X-Forwarded-For)  |

---

## 4. API Gateway

**Definition:** An API gateway is a **policy enforcement point**. It answers: _should this request be allowed, and if so, where does it go?_ It cares about request semantics — auth, rate limits, transforms, business routing — not connection mechanics.

Where a reverse proxy cares about the _connection_, and a load balancer cares about _which instance_, an API gateway cares about _the semantics of the request_.

### Core Responsibilities

**Authentication & Authorization:**

```
Request: GET /api/orders
Header: Authorization: Bearer eyJ...

Gateway:
  1. Validate JWT signature
  2. Extract user_id, scopes
  3. Check: does this user have read:orders scope?
  4. Inject X-User-Id: 42 header → forward to order-service
```

The service receives a pre-validated request with user context injected. It never handles JWTs, never calls an auth service — the gateway already did it.

**Rate Limiting:**

```
Rule: user_id X → max 100 req/min
Rule: /api/search → max 10 req/sec (expensive endpoint)
Rule: IP 1.2.3.4 → blocked (abuse)
```

Rate limiting at the gateway is centralized. Without it, each service must implement its own rate limiting, and they don't share state — so a user can hit 100 req/min on user-service AND 100 req/min on order-service simultaneously.

**Request/Response Transformation:**

- Add headers before forwarding (`X-Request-ID`, `X-Trace-ID`)
- Strip internal headers before returning to client
- Transform response shape (e.g., wrap in `{ data: ..., meta: ... }`)
- Protocol translation (REST → gRPC to internal services)

**Routing:**

```
/api/v1/users   ──► user-service v1
/api/v2/users   ──► user-service v2
/api/payments   ──► payment-service
/api/legacy     ──► monolith (during migration)
```

This is business-logic routing — the gateway knows what your services are and what they own. This is different from a reverse proxy's simple upstream mapping.

**Observability:** Every request passes through the gateway, making it the ideal place to emit unified access logs, traces, and metrics regardless of which service handles the request.

### API Gateway vs Reverse Proxy

||Reverse Proxy|API Gateway|
|---|---|---|
|Primary concern|Connection mechanics|Request policy|
|Knows about HTTP bodies?|Rarely|Often|
|Knows about your services?|No (just upstream URLs)|Yes (service registry, versions)|
|Auth logic|No|Yes|
|Rate limiting|No|Yes|
|Business routing rules|No|Yes|
|Examples|Nginx, Caddy, HAProxy|Kong, AWS API Gateway, Apigee, KrakenD|

### When NOT to put logic in the gateway

The gateway can become a **god layer** — a giant ball of business logic that's hard to test and deploy. The rule: if logic belongs to a specific service's domain, it belongs in that service, not the gateway.

Good in gateway: auth, rate limiting, tracing, cross-cutting routing Bad in gateway: "if user has premium plan, allow feature X" (that's your service's concern)

---

## 5. Service Discovery

**Definition:** Service discovery is **how services find each other** — a live map of which instances exist and where they are. It is orthogonal to the proxy / gateway / load-balancer stack.

### The Problem

You have `order-service` that needs to call `user-service`. Where is `user-service`?

- IP addresses of instances change when containers restart
- New instances spin up (scale-out), old ones die (scale-in)
- Hardcoding IPs is not an option in dynamic environments

### Two Models

**Client-side discovery:**

```
order-service ──► Service Registry ──► "user-service is at 10.0.1.5:3000, 10.0.1.8:3000"
order-service ──► picks one ──► 10.0.1.5:3000
```

The caller is responsible for looking up and picking an instance. Requires a discovery client library in every service. Examples: Netflix Eureka + Ribbon, Consul + client library.

**Server-side discovery:**

```
order-service ──► user-service.internal:3000 (DNS name)
                         ↓
               Load Balancer / Service Mesh
                         ↓
               10.0.1.5:3000 or 10.0.1.8:3000
```

The caller uses a stable DNS name. The infrastructure resolves it to a live instance. The service doesn't need a discovery library. Examples: Kubernetes Services (kube-proxy + CoreDNS), AWS Cloud Map + ALB.

### Registration

Services must **register** themselves when they start and **deregister** when they stop:

- **Self-registration:** the service calls the registry on startup. Problem: what if it crashes? Health checks handle deregistration.
- **Third-party registration:** the platform (Kubernetes, ECS) registers the service. The service is unaware of the registry.

### Health Checks and TTL

The registry actively probes registered instances:

```
Registry ──► GET http://10.0.1.5:3000/health every 10s
           ◄── 200 OK → keep in registry
           ◄── 503 / timeout → remove from registry after N failures
```

Without health checks, the registry becomes a list of dead addresses. Load balancers that integrate with service discovery pull the live address list and update routing tables automatically.

### DNS-Based Discovery

In Kubernetes, every Service gets a DNS entry:

```
user-service.default.svc.cluster.local → 10.96.0.42 (virtual ClusterIP)
```

kube-proxy programs iptables/ipvs rules to distribute traffic to pod IPs behind the ClusterIP. This is server-side discovery via DNS + kernel-level load balancing.

---

## 6. How They All Work Together

**Definition:** In production these pieces compose: gateway for policy, load balancer for instance choice, reverse proxy for connection mechanics, and service discovery for a live instance map.

Here's a realistic production architecture with all four components in play:

```
                          Internet
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │  ← Kong / AWS API GW / KrakenD
                    │                 │
                    │  • JWT auth     │
                    │  • Rate limit   │
                    │  • Route rules  │
                    │  • Tracing IDs  │
                    └────────┬────────┘
                             │
               ┌─────────────┼─────────────┐
               │             │             │
       ┌───────▼──────┐      │      ┌──────▼───────┐
       │  L7 Load     │      │      │  L7 Load     │
       │  Balancer    │      │      │  Balancer    │
       │  (user-svc)  │      │      │  (order-svc) │
       └──┬───┬───┬───┘      │      └──┬───┬───┬───┘
          │   │   │          │         │   │   │
       ┌──▼┐ ┌▼┐ ┌▼┐        │      ┌──▼┐ ┌▼┐ ┌▼┐
       │RP │ │RP│ │RP│       │      │RP │ │RP│ │RP│   ← Reverse Proxy (sidecar)
       │   │ │  │ │  │       │      │   │ │  │ │  │     per instance (Envoy)
       └──┬┘ └─┬┘ └─┬┘       │      └──┬┘ └─┬┘ └─┬┘
          │    │    │         │         │    │    │
       ┌──▼┐ ┌─▼┐ ┌─▼┐      │      ┌──▼┐ ┌─▼┐ ┌─▼┐
       │US1│ │US2│ │US3│     │      │OS1│ │OS2│ │OS3│  ← Service instances
       └───┘ └───┘ └───┘     │      └───┘ └───┘ └───┘
                              │
                    ┌─────────▼──────┐
                    │ Service        │
                    │ Discovery      │  ← Consul / Kubernetes DNS
                    │                │
                    │ user-svc →     │
                    │   [US1,US2,US3]│
                    │ order-svc →    │
                    │   [OS1,OS2,OS3]│
                    └────────────────┘
```

**Traffic flow, step by step:**

```
1. Client → API Gateway
   GET /api/v1/orders
   Authorization: Bearer eyJ...

2. API Gateway:
   a. Validates JWT → user_id=42, scope=read:orders ✓
   b. Checks rate limit for user_id=42 → 43/100 req/min ✓
   c. Looks up routing rule: /api/v1/orders → order-service
   d. Adds headers: X-User-Id: 42, X-Trace-Id: abc-123
   e. Forwards to order-service load balancer

3. L7 Load Balancer (order-service):
   a. Queries service discovery → [OS1:3000, OS2:3000, OS3:3000]
   b. Picks OS2 (least requests)
   c. Forwards HTTP request

4. Reverse Proxy (sidecar on OS2):
   a. Connection already pooled → no TCP handshake
   b. Forwards to localhost:3001 (order-service process)

5. order-service (OS2):
   a. Receives request with X-User-Id: 42 already injected
   b. No auth logic needed — trusted internal header
   c. Calls user-service to get user details:
      GET http://user-service.internal/users/42

6. Service Discovery (for internal call):
   a. user-service.internal resolves via DNS → ClusterIP
   b. kube-proxy routes to one of [US1, US2, US3]

7. Response travels back:
   Service → Sidecar Proxy → LB → API Gateway (strips internal headers) → Client
```

### The Sidecar Pattern (Service Mesh)

In modern deployments (Istio, Linkerd, AWS App Mesh), the reverse proxy lives as a **sidecar** — a container running alongside every service instance.

```
Pod:
  ├── order-service (port 3001, localhost only)
  └── envoy sidecar (port 3000, intercepts all traffic)
```

The sidecar intercepts all inbound and outbound traffic via iptables rules. The service never makes raw TCP connections — all traffic goes through the local proxy. This gives you:

- mTLS between every pair of services (zero-trust networking)
- Per-service circuit breaking and retries
- Distributed tracing without any service code changes
- Traffic shaping (canary rollouts) at the instance level

The trade-off: operational complexity. Every request now hops through two sidecars (sender's proxy and receiver's proxy). Worth it at scale; overkill for smaller deployments.

---

## Summary: One Sentence Each

|Component|One sentence|
|---|---|
|**Reverse Proxy**|Terminates the client TCP connection, handles network mechanics (slow clients, TLS, compression, connection pooling), and forwards to one upstream — decoupling network concerns from service code.|
|**L4 Load Balancer**|Distributes raw TCP streams across upstream instances without reading HTTP, based on IP/port information only.|
|**L7 Load Balancer**|Distributes HTTP requests across upstream instances, with full visibility into URLs, headers, and cookies enabling content-aware routing and HTTP-level health checks.|
|**API Gateway**|Enforces business policy on every request — authentication, authorization, rate limiting, and semantic routing — before it reaches any service.|
|**Service Discovery**|Maintains a live registry of which instances exist and where they are, enabling load balancers and services to find each other dynamically as the fleet changes.|

---

## Quick Reference: "Can X do Y?"

||Terminate TLS|Route by URL|Auth/Rate-limit|Balance across instances|Dynamic instance lookup|
|---|:-:|:-:|:-:|:-:|:-:|
|Reverse Proxy|✅|Partial*|❌|❌ (pure form)|❌|
|L4 LB|❌|❌|❌|✅|With discovery|
|L7 LB|✅|✅|❌|✅|With discovery|
|API Gateway|✅|✅|✅|Delegates to LB|Via discovery|
|Service Discovery|❌|❌|❌|❌|✅ (that's its job)|

*A proxy doing URL routing is crossing into gateway territory.
