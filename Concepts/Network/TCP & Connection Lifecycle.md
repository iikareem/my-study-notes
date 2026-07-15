---
tags:
  - http
  - network
  - sockets
  - tcp
---

# TCP & Connection Lifecycle

> Merged reference: handshake, kernel queues, connection management, TIME_WAIT, HTTP connection reuse, and congestion control.

## Related deep-dives

- [[HTTP Connections]] — HTTP/1.0 → 1.1 → 2/3 connection reuse & multiplexing
- [[TCP TIME_WAIT — Recap]] — why TIME_WAIT exists and how to handle it
- [[Congestion Window]] — flow control vs congestion control, cwnd
- [[HOL]] — head-of-line blocking
- [[WebSocket vs Server-Sent Events (SSE)]] — long-lived alternatives

---

# Part A — Backend Connection Management & TCP Lifecycle

> A complete reference covering TCP internals, kernel socket architecture, security vulnerabilities, and reverse proxy optimization.

---

## Table of Contents

1. [The TCP Three-Way Handshake](#1-the-tcp-three-way-handshake)
2. [Kernel Connection Queues](#2-kernel-connection-queues)
3. [Connections, Sockets & File Descriptors](#3-connections-sockets--file-descriptors)
4. [Listening vs. Connected Sockets](#4-listening-vs-connected-sockets)
5. [SYN Flood Attacks & SYN Cookies](#5-syn-flood-attacks--syn-cookies)
6. [Reverse Proxy Optimizations](#6-reverse-proxy-optimizations)
7. [Compression Offloading](#7-compression-offloading)

---

## 1. The TCP Three-Way Handshake

TCP is a **connection-oriented** protocol — before any data flows, both endpoints must agree to open a channel. This is done via the **Three-Way Handshake**.

```
Client                         Server
  |                               |
  |──── SYN (ISN_c) ────────────>|   Step 1: Client initiates
  |                               |
  |<─── SYN-ACK (ISN_s, ACK_c) ──|   Step 2: Server acknowledges & responds
  |                               |
  |──── ACK (ACK_s) ────────────>|   Step 3: Client confirms
  |                               |
  |         [ESTABLISHED]         |   Connection is live
```

### Step-by-Step Breakdown

|Step|Sender|Flag(s)|What Happens|
|---|---|---|---|
|1|Client → Server|`SYN`|Client picks a random **Initial Sequence Number** (`ISN_c`) and sends it|
|2|Server → Client|`SYN + ACK`|Server acknowledges (`ISN_c + 1`), picks its own `ISN_s`, and replies|
|3|Client → Server|`ACK`|Client acknowledges (`ISN_s + 1`). Both sides enter `ESTABLISHED` state|

> **Why sequence numbers?** They allow both sides to detect lost or out-of-order packets and request retransmission — this is what makes TCP _reliable_.

---

## 2. Kernel Connection Queues

The OS kernel manages incoming connections using **two separate queues** per listening socket. Understanding these is critical for tuning backend performance.

### A. SYN Queue (Incomplete Connection Queue)

- Holds connections that have sent `SYN` and received `SYN-ACK`, but **haven't sent the final ACK yet**
- Connection state: `SYN_RCVD`
- Each entry stores a **TCB (Transmission Control Block)** — the kernel's memory structure for tracking the connection

### B. Accept Queue (Complete Connection Queue)

- Holds **fully established** connections (`ESTABLISHED` state) waiting for the application to call `accept()`
- Once `accept()` is called, the kernel dequeues the connection and hands it to the application as a **File Descriptor**

```
Internet ──> [SYN Queue] ──(handshake complete)──> [Accept Queue] ──(accept())──> Application
               SYN_RCVD                               ESTABLISHED
```

### Key Tuning Parameters (Linux)

```bash
# SYN Queue size (per socket backlog)
net.ipv4.tcp_max_syn_backlog

# Accept Queue size (set in listen() call or via somaxconn)
net.core.somaxconn
```

> **What happens when these queues fill up?** Incoming `SYN` packets are **dropped**, causing connection timeouts for clients.

---

## 3. Connections, Sockets & File Descriptors

These three terms are related but distinct. Confusing them is one of the most common backend misconceptions.

### Connection

An **abstract logical state** — the concept that two endpoints are in bidirectional communication. Not a concrete OS object.

---

### Socket

The **concrete OS-level object** representing one endpoint of a connection. A socket is uniquely identified by a **4-Tuple**:

```
{ Local IP : Local Port, Remote IP : Remote Port }
```

**Key insight — Port Multiplexing:**  
A single server port (e.g., `8080`) can hold **thousands of simultaneous sockets** because each socket has a _unique client IP/port combination_, keeping every 4-tuple distinct.

```
Server 0.0.0.0:8080 ←── Client 1.2.3.4:50001   (Socket A, unique 4-tuple)
Server 0.0.0.0:8080 ←── Client 1.2.3.4:50002   (Socket B, unique 4-tuple)
Server 0.0.0.0:8080 ←── Client 5.6.7.8:41000   (Socket C, unique 4-tuple)
```

---

### File Descriptor (FD)
**open resource** _used by_ a process to communicate and perform I/O (Input/Output).

Unix treats **everything as a file** — including network sockets. When a socket enters the Accept Queue, the kernel:

1. Creates a socket structure in memory
2. Allocates an integer index (e.g., `3`, `4`, `5`...) in the process's **File Descriptor Table**
3. Returns that integer to the application

Your application reads/writes network data by referencing this integer handle.

```c
int fd = accept(listen_fd, ...);  // fd = 5
read(fd, buffer, size);           // Read data from client via FD 5
write(fd, response, size);        // Send response to client via FD 5
```

### Resource Bottleneck: FD Limits

```bash
ulimit -n          # View current FD limit (default often 1024)
ulimit -n 65536    # Increase limit for high-traffic servers
```

> If this limit is hit, the kernel **drops new connections** even if the server has spare CPU and RAM — FD exhaustion is a common and nasty production incident.

---

## 4. Listening vs. Connected Sockets

A backend server always has **two structurally different categories** of sockets running simultaneously.

### Listening Socket

| Property           | Detail                                                |
| ------------------ | ----------------------------------------------------- |
| Count              | **One per port** (permanent, never changes)           |
| Bound to           | `0.0.0.0:8080` (local IP + port only, no client info) |
| Job                | Accept incoming `SYN` packets, feed the kernel queues |
| Can transfer data? | **No** — it never reads/writes application payload    |

### Connected Socket

|Property|Detail|
|---|---|
|Count|**One per connected client** (ephemeral, created/destroyed dynamically)|
|Bound to|Full 4-tuple `{server IP, server port, client IP, client port}`|
|Job|Bidirectional data exchange with a specific client|
|Can transfer data?|**Yes** — this is where HTTP bodies, DB queries, etc. flow|

### Concurrency Model

```
500 clients connected to :8080
│
├── 1 × Listening Socket (permanent, still watching for new SYNs)
│
└── 500 × Connected Sockets (one per client, each with unique 4-tuple)
```

When you call `accept()`, the kernel hands you a File Descriptor pointing to one of those **Connected Sockets** — the Listening Socket continues accepting new clients uninterrupted.

---

## 5. SYN Flood Attacks & SYN Cookies

### The Attack

A **SYN Flood** is a Denial-of-Service attack that exploits the finite size of the SYN Queue:

```
Attacker (spoofed IPs) ───> SYN ──> Server
                            SYN ──> Server   (SYN Queue fills up)
                            SYN ──> Server
                            ...

Legitimate Client ─────> SYN ──> Server    ← DROPPED (queue full)
```

**Why it works:**

1. Attacker sends thousands of `SYN` packets with **spoofed (fake) source IPs**
2. Server allocates a TCB per `SYN`, replies with `SYN-ACK` to the fake IPs
3. The fake IPs never send the final `ACK` — slots stay occupied
4. SYN Queue fills → legitimate connections are silently dropped

### The Defense: SYN Cookies

SYN Cookies eliminate the need to store state during the handshake. When the SYN Queue passes a threshold:

```
Normal mode:      SYN ──> allocate TCB in SYN Queue ──> SYN-ACK
SYN Cookie mode:  SYN ──> [NO allocation] ──> SYN-ACK with encoded ISN_s
```

**How the encoding works:**

```
ISN_s = HASH(client_IP, client_port, server_IP, server_port, timestamp, secret_key)
```

**Verification on ACK:**

```
Client returns ACK with ACK_number = ISN_s + 1
Kernel recomputes hash, subtracts 1, compares ──> match? Allocate socket. No match? Drop.
```

**Result:** The server never stores anything for unverified connections. Spoofed SYNs consume **zero memory**.

> Enable check in Linux: `cat /proc/sys/net/ipv4/tcp_syncookies` (1 = enabled)

---

## 6. Reverse Proxy Optimizations

A **Reverse Proxy** (NGINX, HAProxy, Envoy, Caddy) sits between the internet and your backend, leveraging the socket architecture above to dramatically reduce backend load.

```
[Internet Clients]
       │  (hundreds of dynamic connections)
       ▼
[Reverse Proxy]  ◄─── handles TCP, TLS, buffering, compression
       │  (small pool of persistent backend connections)
       ▼
[Backend Server]  ◄─── only sees clean, fast, local traffic
```

### Optimization 1: Connection Buffering (Slowloris Protection)

**The problem:** Slow clients (mobile, bad networks) hold a Connected Socket open for seconds while trickling in their HTTP request. This exhausts the backend's thread pool.

**The fix:**

```
Slow Client ──(slow stream)──> Proxy buffer ──(full request, instant)──> Backend
                               [proxy absorbs the wait]
```

The backend socket is occupied for **milliseconds**, not seconds.

---

### Optimization 2: TCP Connection Pooling (Keep-Alives)

Every new TCP connection costs: 1 RTT for the handshake + kernel memory for queues + FD allocation.

**Without a proxy:** Every user → new handshake with backend  
**With a proxy:** Proxy maintains a **pre-warmed pool** of persistent connections to the backend

```
[1000 clients] ──> [Proxy] ──(10 keep-alive connections)──> [Backend]
                                  ↑
                         Proxy multiplexes all
                         client requests over
                         this small stable pool
```

The backend is completely isolated from connection churn.

---

### Optimization 3: TLS Termination

TLS handshakes require expensive asymmetric cryptography (RSA/ECDH key exchange, certificate verification) before any data flows.

```
Client ──(HTTPS/TLS)──> Proxy ──(HTTP/plaintext)──> Backend
                          │
                    Proxy decrypts here.
                    Backend never touches crypto.
```

**Result:** Backend CPU is freed for business logic and database work, not cryptographic math.

---

### Optimization 4: Load Balancing

Because the proxy controls all inbound listening sockets, it can monitor backend health in real time:

- Accept Queue filling up on Instance A? → Route new connections to Instance B
- Instance A returns 5xx errors? → Mark it unhealthy, stop sending traffic
- Instance C comes online? → Gradually introduce it to the pool

---

## 7. Compression Offloading

Compression (Gzip / Brotli) is another expensive CPU task that belongs at the proxy layer.

### The Request Flow

```
1. Client sends:   Accept-Encoding: gzip, br

2. Proxy forwards the request to backend (uncompressed internally)

3. Backend returns: raw 5MB JSON payload

4. Proxy compresses: 5MB → ~800KB, adds Content-Encoding: gzip header

5. Proxy sends: compressed payload over the public internet
```

### Why Offload to the Proxy?

|Benefit|Explanation|
|---|---|
|**CPU Isolation**|Compression is CPU-intensive. Keeping it out of your app server preserves cycles for request handling|
|**Reduced Bandwidth**|Payloads shrink 70–80%, meaning fewer TCP round trips and faster delivery|
|**Static File Pre-compression**|Proxy compresses static assets once, caches the `.gz` version in memory, and serves it directly without ever hitting the backend again|

### The Full Optimized Pipeline

```
[Client]
   │ compressed (Brotli/Gzip)
   │ encrypted (TLS)
   │ slow public internet
   ▼
[Reverse Proxy]   ← decrypts TLS, decompresses (for routing), recompresses outbound
   │ raw plaintext
   │ uncompressed
   │ fast local network
   │ persistent keep-alive connection
   ▼
[Backend Server]  ← only sees clean, fast, local HTTP
```

> The backend lives in a sanitized environment: no TLS math, no compression overhead, no slow client management — just your application logic.

---

## Quick Reference: Key Concepts

|Concept|One-Line Summary|
|---|---|
|**SYN Queue**|Holds half-open connections (`SYN_RCVD`) during handshake|
|**Accept Queue**|Holds fully established connections waiting for `accept()`|
|**Socket 4-Tuple**|`{local IP, local port, remote IP, remote port}` — uniquely identifies every connection|
|**File Descriptor**|Integer handle the kernel gives your app to read/write a socket|
|**SYN Cookie**|Stateless handshake technique; encodes connection info in the sequence number|
|**Connection Pooling**|Proxy reuses a small set of backend connections for all clients|
|**TLS Termination**|Proxy handles encryption; backend gets plain HTTP|
|**Compression Offload**|Proxy compresses responses; backend returns raw data|

**Listening socket** — there is exactly one, it lives forever, and it has one job: receive SYN packets and kick off handshakes. It only knows `0.0.0.0:8080` — it has no idea who the client is. It cannot read or write any HTTP data. Think of it as a **receptionist** whose only job is to open the door for new visitors.

**Connected socket** — the kernel creates a brand new one the moment a handshake completes. It knows the full 4-tuple (your IP + port + client IP + client port), so it can route data to exactly one client. This is where `read()` and `write()` actually happen. Think of it as a **dedicated phone line** opened just for that one client.

The key insight: the listening socket never stops doing its job. While your app is reading HTTP from connected socket A, the listening socket is simultaneously receiving a new SYN from client D. They don't block each other at all.

- **Connection** → the abstract fact that two machines agreed to talk (`ESTABLISHED` state)
- **Socket** → the concrete OS object on each machine representing its endpoint (identified by the 4-tuple)
- **File Descriptor** → the integer your code uses to reference the socket (`FD = 3`). Your app never touches the socket directly — it just says "read from FD 3" and the kernel handles it

ANOTHER THING

That is a really smart question. They both deal with incoming data from a client, but they are actually **completely different** and live at different stages of a network connection.

The short answer is: **No, the receiver buffer is not the SYN Queue.** To understand the difference, imagine a popular restaurant. There are three completely different "queues" or waiting areas a customer goes through:

1. **The Hostess Desk (SYN Queue):** Checking if you have a reservation and doing the handshake.
    
2. **The Waiting Line (Accept Queue):** Handshake is done, you are inside the building, but waiting for a table to open up.
    
3. **Your Actual Table (Receiver Buffer):** You are seated, and the kitchen is bringing you plates of food (data) faster than you can eat them.
    

Here is exactly how they break down in networking and Node.js.

## 1. The SYN Queue (The Handshake Phase)

When a client wants to connect to your Node.js or Express server, it must perform the TCP 3-way handshake (SYN $\rightarrow$ SYN-ACK $\rightarrow$ ACK).

- **What it holds:** It holds "half-open" connections. A client sent a `SYN`, your server replied with `SYN-ACK`, and now the server is waiting for the final `ACK`.
    
- **Data inside:** There is **zero application data** (no HTTP requests, no JSON) here yet. It only contains network metadata needed to establish the connection.
    
- **Security connection:** This is what gets overwhelmed during a **SYN Flood DDoS attack**.
    

## 2. The Accept Queue (The Connection Phase)

Once the client sends that final `ACK`, the handshake is complete. The connection is now fully established. It moves out of the SYN Queue and into the **Accept Queue**.

- **What it holds:** Fully established TCP connections that are waiting for your application code (like Node.js, Express, or Netty) to call `server.listen()` and actively `.accept()` them into the event loop.
    
- **If it fills up:** Your server will stop accepting new connections entirely, and users will get "Connection Refused" errors.
    

## 3. The Receiver Buffer (The Data Phase)

Now, the connection has been accepted by Node.js. The pipe is wide open, and the client starts blasting data—like uploading a 50MB file or sending a massive JSON payload.

**This is where the Receiver Buffer (`SO_RCVBUF`) lives.**

- **What it holds:** It holds the **actual raw bytes of the request data** that have physically arrived at the server's network card but haven't been processed by your code yet.
    
- **How it works:** As bytes arrive over the network, the operating system puts them into this memory buffer. When you write code like `req.on('data', chunk => ...)` in Node.js, you are actively pulling bytes _out_ of this Receiver Buffer.
    
- **Flow Control Link:** This is the buffer that dictates the **Receiver Window ($rwnd$)** we talked about earlier. If your Node.js code gets blocked and stops reading data, this buffer fills up. Once it's full, the OS tells the client to stop sending data.
    

## Summary Comparison

|**Metric**|**SYN Queue**|**Accept Queue**|**Receiver Buffer**|
|---|---|---|---|
|**Stage**|Before connection is made.|Connection made, waiting for app.|Connection active, holding raw data bytes.|
|**What's inside?**|Connection requests (Handshake).|Fully open network sockets.|The actual HTTP payload / bytes.|
|**Controlled by...**|Operating System kernel.|Operating System kernel.|Application reading speed (Node.js event loop).|
|**Related TCP Window**|None.|None.|**Flow Control ($rwnd$)**|

When building your **adaptive rate limiter**, you are primarily trying to protect the **Receiver Buffer** and the **Event Loop**. If your event loop slows down, you stop reading from the Receiver Buffer, it fills up, and the whole system grinds to a halt!
---

# Part B — TCP Connection Lifecycle (Practical Recap)

---

## 1. The Big Picture

When a client connects to your server, the connection goes through 3 stages before your app can use it:

```
Client                         Server
  │                              │
  │──── SYN ───────────────────► │  [SYN Queue]
  │◄─── SYN-ACK ────────────────│
  │──── ACK ───────────────────► │  [Accept Queue]
  │                              │
  │                         App calls accept()
  │                              │
  │◄══════ connection ready ════►│  [Your App]
```

|Stage|What's Happening|
|---|---|
|**SYN Queue**|Client is mid-handshake, not ready yet|
|**Accept Queue**|Handshake done, waiting for your app to pick it up|
|**Your App**|You called `accept()`, connection is yours to use|

---

## 2. After Your App Calls `accept()`

### Where does the connection actually live?

Your app gets back a **file descriptor (fd)** — just a number like `fd = 5`.

That number is a **reference** to the real connection, which lives in **OS kernel memory**.

```
Your App Process
┌─────────────────┐
│   fd = 5   ──────────────────► OS Kernel Memory
│                 │               (real socket, buffers,
│                 │                connection state)
└─────────────────┘
```

### Simple analogy

Think of it like a **coat check at a restaurant**:

- The **coat** (real connection) is stored by the coat check (OS kernel)
- You just hold a **ticket number** (file descriptor)
- When you want to use it, you show your ticket
- When you're done, you return the ticket → coat gets discarded

---

## 3. The Kernel Data Structure — `struct sock`

The real connection lives in a kernel structure called **`struct sock`**:

```
struct sock  (in kernel memory)
┌─────────────────────────┐
│  Send Buffer            │  ← your app writes data here
│  Recv Buffer            │  ← incoming data lands here
│  State (ESTABLISHED)    │
│  IP / Port info         │
│  Timers                 │
└─────────────────────────┘
```

### How fd connects to it

```
Your App          Kernel
fd = 5   ──►   file table entry  ──►  struct sock
```

Just a chain of pointers until it reaches the real struct.

### How the kernel tracks ALL connections globally

The kernel keeps a **hash table** of all active connections:

```
Hash Table (ehash)
├── connection 1  →  struct sock
├── connection 2  →  struct sock
├── connection 3  →  struct sock
└── ...
```

When a packet arrives, the kernel does an **O(1) lookup** by `(src IP, src port, dst IP, dst port)` to find the right struct instantly.

### Summary

||Where|
|---|---|
|**Real connection data**|OS Kernel memory (managed by OS)|
|**Your app's reference to it**|File descriptor (just a number)|
|**Global index of all connections**|Kernel hash table (ehash)|

> Your fd → points to → `struct sock` → stored in → kernel heap → indexed in → hash table

---

## 4. How Your App Interacts With the Connection

|Your App Does|What Actually Happens|
|---|---|
|`write(fd, data)`|Data goes into **send buffer**, OS sends it over the network|
|`read(fd)`|OS moves data from **recv buffer** to your app|
|`close(fd)`|OS starts closing the connection, frees memory|

---

## 5. Each Client Gets Its Own `struct sock`

```
Client 1  ──►  fd = 5  ──►  struct sock  (client 1 buffers)
Client 2  ──►  fd = 6  ──►  struct sock  (client 2 buffers)
Client 3  ──►  fd = 7  ──►  struct sock  (client 3 buffers)
```

Each one is **completely separate** — different buffers, different state.

### But who reads/writes them?

|Server Model|How It Works|
|---|---|
|**Thread per connection** (Java, traditional)|Each thread holds its own fd → its own struct sock|
|**Event loop** (Node.js)|One thread handles all fds, switches between them|
|**Thread pool** (Nginx)|Multiple threads share access carefully|

---

## 6. Why Slow Clients Hurt Your Server

### The slow-down chain

```
Client has slow internet
        │
        ▼
Recv Buffer fills up (struct sock)
        │
        ▼
App can't read fast enough
        │
        ▼
Accept Queue fills up
        │
        ▼
SYN Queue fills up
        │
        ▼
New clients get rejected ❌
```

### Each bottleneck

| Where            | Problem                                                        |
| ---------------- | -------------------------------------------------------------- |
| **Recv Buffer**  | Slow client → data sits too long → buffer fills                |
| **Accept Queue** | App busy with slow clients → can't call `accept()` fast enough |
| **SYN Queue**    | Everything backed up → new handshakes have no room             |

### What "app blocked" really means

For **thread-per-connection** servers:

```
Thread 1  waiting for client 1 data  ← STUCK (doing nothing)
Thread 2  waiting for client 2 data  ← STUCK (doing nothing)
Thread 3  waiting for client 3 data  ← STUCK (doing nothing)

New client arrives... no free thread ❌
```

For **event loop** servers (Node.js), the thread isn't stuck — but the slow connection still:

- Holds a `struct sock` in memory
- Holds an fd slot
- Consumes recv buffer space

### Restaurant analogy

> A slow client is like a customer sitting at a table for 3 hours ordering nothing. The waiter isn't stuck but the **table is taken**, so new customers can't sit.
> 
> The table = `struct sock` + its buffers.

---

## 7. The Solution — Reverse Proxy

### Without a reverse proxy

```
Slow client (bad internet)
        │
        │  slow / partial data
        ▼
Your App Server  ← holding struct sock open for a long time
                   wasting memory, fd slots, buffers
```

### With a reverse proxy (Nginx, Caddy, etc.)

```
Slow Client          Reverse Proxy          Your App Server
    │                     │                      │
    │  slow connection     │   fast connection    │
    │ ───────────────────► │ ────────────────────►│
    │                      │                      │
    │                  buffers here           gets clean
    │                  handles slow           fast requests
    │                  client pain            only ✅
```

The reverse proxy:

- **Absorbs** the slow client connection
- **Buffers** the full request internally
- **Then** sends it to your app in one fast local shot

### Comparison

||Without Proxy|With Proxy|
|---|---|---|
|**Connection speed**|Slow internet speed|Localhost speed|
|**Request delivery**|Slow / partial|Complete and instant|
|**struct sock held**|Long time|Very short time|
|**Buffers**|Fill up slowly|Filled instantly|
|**Thread / fd slots**|Wasted on slow clients|Freed up quickly|

### What reverse proxy also handles

- **SSL termination** — your app doesn't deal with encryption overhead
- **Compression** — gzip handled before your app sees the data
- **Keep-alive management** — proxy manages slow clients, app gets clean connections
- **Load balancing** — spreads requests across multiple app instances

---

## 8. Key Takeaways for Backend Devs

|Concept|What to Remember|
|---|---|
|**fd**|Just a ticket/handle your app holds|
|**struct sock**|The real connection, lives in kernel memory|
|**Send / Recv Buffer**|Where data flows through — can fill up|
|**Slow client**|Holds a struct sock hostage, backs up the whole pipeline|
|**Reverse proxy**|Shields your app from slow clients, gives it clean fast connections|
|**Connection pooling**|Reuse open connections instead of open/close per request|
|**Timeouts**|Don't wait forever for a slow client|

> A live connection = an open socket your app holds in memory. The OS tracks it. Your framework pools it. It dies when closed or timed out.