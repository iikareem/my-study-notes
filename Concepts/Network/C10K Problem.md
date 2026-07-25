---
tags:
  - network
  - scalability
  - sockets
  - io
---

# The C10K Problem: From Scratch to Mastery

> _"It's time for web servers to handle ten thousand clients simultaneously."_ — Dan Kegel, 1999

---

## Table of Contents

1. [What Is the C10K Problem?](#1-what-is-the-c10k-problem)
2. [Prerequisites: How Computers Handle I/O](#2-prerequisites-how-computers-handle-io)
    - 2.1 [Processes and Threads](#21-processes-and-threads)
    - 2.2 [File Descriptors](#22-file-descriptors)
    - 2.3 [The Kernel and System Calls](#23-the-kernel-and-system-calls)
    - 2.4 [Blocking vs Non-Blocking I/O](#24-blocking-vs-non-blocking-io)
    - 2.5 [The Socket and TCP Lifecycle](#25-the-socket-and-tcp-lifecycle)
3. [Why 10,000 Connections Is Hard](#3-why-10000-connections-is-hard)
    - 3.1 [The Thread-Per-Connection Model](#31-the-thread-per-connection-model)
    - 3.2 [Memory Cost of Threads](#32-memory-cost-of-threads)
    - 3.3 [Context Switching Overhead](#33-context-switching-overhead)
    - 3.4 [File Descriptor Limits](#34-file-descriptor-limits)
4. [I/O Multiplexing: The Core Solution](#4-io-multiplexing-the-core-solution)
    - 4.1 [select()](#41-select)
    - 4.2 [poll()](#42-poll)
    - 4.3 [epoll (Linux)](#43-epoll-linux)
    - 4.4 [kqueue (BSD/macOS)](#44-kqueue-bsdmacos)
    - 4.5 [IOCP (Windows)](#45-iocp-windows)
5. [Concurrency Models](#5-concurrency-models)
    - 5.1 [One Thread Per Connection](#51-one-thread-per-connection)
    - 5.2 [Thread Pool + Blocking I/O](#52-thread-pool--blocking-io)
    - 5.3 [Single-Threaded Event Loop](#53-single-threaded-event-loop)
    - 5.4 [Multi-Threaded Event Loop](#54-multi-threaded-event-loop)
    - 5.5 [Async/Await & Coroutines](#55-asyncawait--coroutines)
6. [The Event Loop Deep Dive](#6-the-event-loop-deep-dive)
7. [Reactor vs Proactor](#7-reactor-vs-proactor)
8. [Real-World Architectures](#8-real-world-architectures)
9. [OS-Level Tuning](#9-os-level-tuning)
10. [How Modern Systems Solved C10K](#10-how-modern-systems-solved-c10k)
11. [The C10M Problem](#11-the-c10m-problem)
12. [System Design Implications](#12-system-design-implications)
13. [Summary Cheat Sheet](#13-summary-cheat-sheet)

---

## 1. What Is the C10K Problem?

In **1999**, software engineer **Dan Kegel** wrote a landmark paper asking:

> Can a single server handle **10,000 simultaneous client connections**?

At the time, most web servers followed a simple model: for each incoming client, spawn a new process or thread. This worked fine at 100 or even 1,000 connections — but at 10,000, servers would collapse under the weight of memory exhaustion and CPU thrashing.

The problem was not about **bandwidth** or **raw CPU speed**. It was about **how the operating system and application manage thousands of concurrent I/O operations** efficiently.

Solving C10K unlocked the modern internet: it's why Node.js, Nginx, Redis, and event-driven architectures exist today.

---

## 2. Prerequisites: How Computers Handle I/O

Before understanding _why_ C10K is hard, you need to understand how your OS handles connections at all.

### 2.1 Processes and Threads

**Process**: An independent program in execution. Has its own memory space, file descriptors, and OS resources. Heavy to create (~1–10ms) and memory-expensive (~several MB).

**Thread**: A lighter execution unit _within_ a process. Shares memory with sibling threads. Cheaper to create than processes, but still not free.

```
Process A                     Process B
┌────────────────────┐        ┌────────────────────┐
│  Memory (private)  │        │  Memory (private)  │
│  Thread 1          │        │  Thread 1          │
│  Thread 2          │        │  Thread 2          │
│  FD table          │        │  FD table          │
└────────────────────┘        └────────────────────┘
        ↕ System Call                  ↕ System Call
    ┌──────────────────────────────────────┐
    │           OS Kernel                  │
    └──────────────────────────────────────┘
```

### 2.2 File Descriptors

In Unix/Linux, **everything is a file**: network sockets, disk files, pipes, and devices are all represented as **file descriptors (FDs)** — small integers your program uses to reference them.

```c
int fd = open("myfile.txt", O_RDONLY);       // fd might be 3
int sockfd = socket(AF_INET, SOCK_STREAM, 0); // fd might be 4
```

Each open connection to a client consumes **one file descriptor**. So 10,000 connections = 10,000 open file descriptors.

### 2.3 The Kernel and System Calls

Your application code runs in **user space**. To interact with the network or disk, it must ask the kernel via **system calls** (syscalls). This crossing of the user/kernel boundary has overhead.

```
User Space        │        Kernel Space
                  │
app calls read()  │
        ─────────→│   kernel checks if data is ready
                  │   if not: put process to sleep
                  │   when data arrives: wake process
        ←─────────│   return data to app
                  │
```

### 2.4 Blocking vs Non-Blocking I/O

This is the **central concept** for understanding C10K.

**Blocking I/O** (default): When you call `read()` on a socket with no data ready, the calling thread **sleeps** until data arrives. The thread does nothing while waiting.

```
Thread:  [working]──[BLOCKED waiting for data]──[working again]
Time:    ──────────────────────────────────────────────────────→
```

**Non-Blocking I/O**: You set the socket to non-blocking mode. When you call `read()` and no data is ready, it **returns immediately** with `EAGAIN` ("try again"). Your thread stays free to do other work.

```c
// Set socket to non-blocking
int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);
```

**The problem with naive non-blocking**: Your thread would have to _busy-poll_ — loop and keep calling `read()` over and over, wasting CPU. You need a smarter way to know _when_ data is ready. That smarter way is **I/O multiplexing** (covered in Section 4).

### 2.5 The Socket and TCP Lifecycle

Every client connection goes through:

```
Server                                Client
  │                                     │
  │  listen() on port 80                │
  │←────────── SYN ─────────────────────│  client initiates
  │─────────── SYN-ACK ─────────────────→│
  │←────────── ACK ─────────────────────│  3-way handshake done
  │                                     │
  │  accept() → new socket fd           │
  │←────────── HTTP GET ────────────────│
  │─────────── HTTP Response ───────────→│
  │←────────── FIN ─────────────────────│
  │─────────── FIN-ACK ─────────────────→│  connection closed
```

Key point: `accept()` returns a **new file descriptor** for each client. The server holds all of these open simultaneously for active connections.

---

## 3. Why 10,000 Connections Is Hard

### 3.1 The Thread-Per-Connection Model

The classic approach — and the root of the C10K problem:

```
Client 1 ─→ [Thread 1] (sleeping, waiting for I/O)
Client 2 ─→ [Thread 2] (sleeping, waiting for I/O)
Client 3 ─→ [Thread 3] (sleeping, waiting for I/O)
...
Client N ─→ [Thread N] (sleeping, waiting for I/O)
```

For most web workloads, threads spend the **majority of their time blocked** waiting for:

- Network data from the client
- A database query to finish
- Disk I/O to complete

This means most threads are asleep at any moment — yet they still consume memory.

### 3.2 Memory Cost of Threads

Each thread requires a **stack** — a chunk of memory reserved for its local variables and call frames.

|Resource|Typical Size|
|---|---|
|Thread stack|1 MB (Linux default)|
|Kernel thread struct|~8 KB|
|Per-thread overhead|~1–8 MB total|

**Math for 10,000 connections:**

```
10,000 threads × 1 MB stack = 10 GB of RAM
```

In 1999, servers had **256 MB to 1 GB** of RAM. Even today, 10 GB just for thread stacks is wasteful and unsustainable.

### 3.3 Context Switching Overhead

When you have more threads than CPU cores, the OS **context switches** between them — saving one thread's state and restoring another's.

- Each context switch takes ~1–10 microseconds
- With 10,000 threads, even if only 100 are runnable, the scheduler thrashes
- CPU caches get invalidated constantly → cache misses → slower execution

At high connection counts, the server spends more time **managing threads** than actually **serving requests**.

### 3.4 File Descriptor Limits

By default, Linux limits:

- **Per-process FDs**: 1,024 (the infamous `ulimit -n 1024`)
- **System-wide FDs**: ~100,000–500,000 (tunable)

Hitting 1,024 connections crashes a naive server. While this is tunable, it's a common foot-gun.

---

## 4. I/O Multiplexing: The Core Solution

The key insight: instead of one thread per connection, use **one thread watching many connections at once**, and act only when a connection is actually ready for I/O.

The OS provides syscalls to do this.

### 4.1 select()

The oldest (POSIX standard) I/O multiplexing call.

```c
fd_set read_fds;
FD_ZERO(&read_fds);
FD_SET(fd1, &read_fds);
FD_SET(fd2, &read_fds);

// Block until at least one fd is ready
select(max_fd + 1, &read_fds, NULL, NULL, &timeout);

// Check which fds are ready — must scan ALL of them
if (FD_ISSET(fd1, &read_fds)) { /* fd1 has data */ }
```

**Problems with select():**

- Hard limit of **1,024 file descriptors** (`FD_SETSIZE`)
- O(n) scan: every call must check _all_ watched FDs, even if only one is ready
- You must rebuild the FD set on every call
- Doesn't hand you the ready FDs — you must scan them all yourself

### 4.2 poll()

An improvement over `select()` — removes the 1,024 FD limit, uses an array instead of bitset.

```c
struct pollfd fds[NUM_CONNECTIONS];
fds[0].fd = fd1;
fds[0].events = POLLIN;

poll(fds, NUM_CONNECTIONS, timeout);

// Still must scan all to find ready ones
for (int i = 0; i < NUM_CONNECTIONS; i++) {
    if (fds[i].revents & POLLIN) { /* fd ready */ }
}
```

**Still O(n)**: must pass all FDs to the kernel on every call. For 10,000 connections, this is 10,000 FDs copied to/from kernel space each time — even if only 5 are active.

### 4.3 epoll (Linux)

`epoll` is the **modern Linux solution** and the foundation of high-performance servers like Nginx and Node.js.

**Key innovation**: You register interest in FDs _once_, and the kernel maintains an internal ready-list. On each call, it returns _only the FDs that are ready_ — O(1) per event regardless of total connections.

```c
// Create epoll instance
int epfd = epoll_create1(0);

// Register a socket — done ONCE
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// Event loop — kernel hands you ONLY ready FDs
struct epoll_event events[MAX_EVENTS];
while (1) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        handle(events[i].data.fd);  // only ready FDs, nothing to scan
    }
}
```

**Complexity comparison:**

```
10,000 total connections, 5 active right now:

select(): scan all 10,000 → find 5 ready    ← O(10,000)
epoll():  kernel hands you exactly 5         ← O(1) per event
```

**epoll trigger modes:**

- **Level-triggered (LT)** (default): notifies you as long as data remains unread. Safer, more forgiving.
- **Edge-triggered (ET)**: notifies only when _new_ data arrives. Higher performance but you must read all available data or you'll miss events.

### 4.4 kqueue (BSD/macOS)

The BSD/macOS equivalent of epoll. Same O(1) per event semantics.

```c
int kq = kqueue();

struct kevent change;
EV_SET(&change, sockfd, EVFILT_READ, EV_ADD, 0, 0, NULL);
kevent(kq, &change, 1, NULL, 0, NULL);  // register once

struct kevent events[MAX_EVENTS];
int n = kevent(kq, NULL, 0, events, MAX_EVENTS, NULL);  // wait
```

### 4.5 IOCP (Windows)

Windows uses **I/O Completion Ports** — a proactive model where you _post_ I/O operations and the kernel notifies you when they _complete_ (not when they're _ready to start_). More on this distinction in Section 7.

---

## 5. Concurrency Models

With I/O multiplexing available, how do you structure your server?

### 5.1 One Thread Per Connection

```
Pros:  Simple to write. Naturally handles blocking operations.
Cons:  Memory explosion. Context switch overhead. Does not scale past ~1K.
```

Apache's original `prefork` MPM used this. Still appropriate for low-connection, CPU-heavy workloads.

### 5.2 Thread Pool + Blocking I/O

A fixed number of threads (e.g., 4 × CPU cores) handle requests from a queue.

```
[Connection Queue] → Thread 1 (handling request)
                   → Thread 2 (blocked on DB)
                   → Thread 3 (handling request)
                   → Thread 4 (idle)
```

```
Pros:  Bounded memory. Simpler than event loops. Natural for CPU-bound work.
Cons:  Blocked threads waste pool slots. Queue grows under load.
       Still hits walls when all threads are I/O-blocked.
```

Used by: Java's traditional servlet containers, older Python frameworks.

### 5.3 Single-Threaded Event Loop

One thread, non-blocking I/O, epoll. The thread is **never blocked** — it always has useful work to do.

```
┌─────────────────────────────────────────────────┐
│              Single Thread                       │
│                                                  │
│  while(true) {                                   │
│      events = epoll_wait(...)  // get ready FDs  │
│      for each event:                             │
│          handle_event()        // fast, no block │
│  }                                               │
└─────────────────────────────────────────────────┘
```

**Rule**: callbacks/handlers must **never block**. Any blocking call (DNS, disk, DB) must be async or offloaded.

```
Pros:  No context switching. Minimal memory. Handles 100K+ connections.
Cons:  One slow handler blocks everything. CPU-bound tasks stall all I/O.
```

Used by: **Node.js**, Redis, Nginx (worker processes), Tornado.

### 5.4 Multi-Threaded Event Loop

Run multiple event loop threads — typically one per CPU core. Each thread handles its own epoll instance.

```
Core 0: Event Loop → epoll → handles connections 0–2499
Core 1: Event Loop → epoll → handles connections 2500–4999
Core 2: Event Loop → epoll → handles connections 5000–7499
Core 3: Event Loop → epoll → handles connections 7500–9999
                                    Total: ~100K connections
```

Used by: **Nginx** (worker_processes), **Node.js Cluster**, **Rust's Tokio**.

### 5.5 Async/Await & Coroutines

A higher-level abstraction over the event loop. You write code that _looks_ synchronous but is actually non-blocking under the hood.

```python
# Python asyncio — looks blocking, isn't
async def handle_client(reader, writer):
    data = await reader.read(1024)   # yields to event loop while waiting
    writer.write(process(data))
    await writer.drain()             # yields again
    writer.close()
```

The `await` keyword means: "pause this coroutine, let the event loop run other coroutines, resume me when this I/O is done."

Used by: Python asyncio, JavaScript async/await, Go goroutines, Rust async/await, C# async/await.

---

## 6. The Event Loop Deep Dive

Understanding the event loop is critical because it's the engine of modern high-performance servers.

There are **two levels** — keep them separate so they don't conflict:

1. **Generic Reactor loop** (any epoll server: Nginx, Redis, custom C) — the C10K idea
2. **Node.js / libuv phases** — the real scheduling order when you write Node code

> Full Node deep dive (poll queue, call stack timing, `setImmediate`): [[Node.js Event Loop & Internals]]

---

### Level 1 — Generic Reactor loop (C10K core)

One thread. Never block. Ask the kernel which FDs are ready, then handle them:

```
┌─────────────────────────────────────────┐
│           EVENT LOOP (Reactor)          │
│                                         │
│   while (true) {                        │
│     events = epoll_wait(...)            │
│     for each ready FD:                  │
│       read() / accept() / write()       │
│       run the handler for that FD       │
│   }                                     │
└─────────────────────────────────────────┘
```

That is enough to solve C10K. Nginx and Redis are this idea (with their own extras).

---

### Level 2 — Node.js / libuv phases (correct order)

Node wraps the Reactor inside **libuv** with a fixed phase order. This is the order to remember for Node:

```
┌──────────────────────────────────────────────────────────────┐
│                    Node.js EVENT LOOP (libuv)                │
│                                                              │
│  timers → pending → idle/prepare → poll → check → close      │
│                                                              │
│  ↕ Between phases: process.nextTick → Promise microtasks     │
└──────────────────────────────────────────────────────────────┘
```

| Phase | What runs |
|---|---|
| **Timers** | Expired `setTimeout` / `setInterval` |
| **Pending callbacks** | Deferred I/O errors (mostly internal) |
| **Idle / Prepare** | Internal libuv housekeeping |
| **Poll** | `epoll_wait` → ready FDs → libuv maps to callbacks → **run I/O callbacks** |
| **Check** | `setImmediate` |
| **Close** | `socket.on('close')`, etc. |

**One tick (simplified):**

```
1. Timers              → run expired timers
2. Pending callbacks   → deferred system I/O errors
3. Idle / Prepare      → internal
4. Poll                → epoll_wait() → ready FDs
                       → libuv maps FD → callback
                       → push callback onto V8 call stack → run now
                       → if idle + setImmediate pending → go to check
                       → if idle + no setImmediate     → wait on epoll
5. Check               → setImmediate
6. Close callbacks     → close handlers
   (microtasks drain between phases)
7. repeat
```

### Poll phase (correct — matches the Node note)

Poll does **two** jobs: wait on the OS, then run I/O callbacks.

```
Poll:

  epoll_wait()                    ← may block; returns only ready FDs
        ↓
  kernel returns [fd5, fd12, fd99]
        ↓
  for each FD:
    libuv read()/accept()         ← Reactor: you copy data from kernel
    map FD → JS callback
        ↓
  push callback onto call stack → execute immediately
```

- **epoll** returns **FDs**
- **poll queue** holds **callbacks** (not FDs)
- Pushing onto the call stack means **run now** (not deferred to another phase)
- There is **no** separate “I/O Callbacks” phase after poll in Node — I/O runs **inside poll**

**Key properties:**

- The loop runs as fast as the CPU allows
- Each callback must finish quickly (< ~1ms ideally)
- Long-running work (image processing, crypto, DB queries) is offloaded to a thread pool and results delivered back as events

---

## 7. Reactor vs Proactor

This is one of the most misunderstood distinctions. The confusion is natural because in both patterns the data _feels_ like it just arrives — but the difference is **where the data sits when you get notified**, and **who does the actual read()**.

### The Key Question: Who does read()?

**Reactor** — epoll/kqueue model:

The kernel tells you the FD is **ready**. But data is still in the **kernel buffer**. You must call `read()` yourself to copy it to your app.

```
Network → [Kernel Buffer] → "fd5 is READY" → app calls read() → [App Buffer] → callback
                                  ↑
                        notified here
                        (data still in kernel)
```

```c
// Reactor flow (inside libuv/Node.js):
epoll_wait() → "fd5 ready"     // notification: readiness
read(fd5, buf)                 // YOU copy data from kernel → app buffer
callback(buf)                  // now run handler
```

**Proactor** — IOCP/io_uring model:

You tell the kernel: "start a read from fd5 into _this_ buffer". The kernel does the I/O itself and notifies you only when the data is **already in your app buffer**.

```
Network → [Kernel Buffer] → kernel copies → [App Buffer] → "READ COMPLETE" → callback
                                                                  ↑
                                                        notified here
                                                        (data already in app)
```

```c
// Proactor flow (io_uring):
io_uring_submit_read(fd5, buf)  // tell kernel: do this read for me
// ... kernel does the work ...
completion_event()              // notification: operation complete
callback(buf)                   // data already in buf, just use it
```

### Why event loops feel like Proactor (but aren't)

In Node.js you never see the `read()` call — it's hidden inside libuv. So the callback feels like data just _appears_. But under the hood:

```
epoll_wait() → "fd5 ready"         ← Reactor notification (readiness)
  libuv calls read(fd5)            ← still happens, hidden from you
    data now in user space
      your JS callback runs        ← this is all you see
```

The event loop you know is **Reactor** — epoll notifies _readiness_, not _completion_.

### One-line summary

```
Reactor:   "the FD is READY"     → you do read()  → data moves kernel→app
Proactor:  "the read COMPLETED"  → data already in app buffer → just use it
```

### Real cost difference

Reactor has one extra step: the `read()` syscall crossing the kernel/user boundary. At normal scale this is negligible. At **C10M scale** — millions of events per second — eliminating that crossing matters, which is exactly why `io_uring` was invented.

||Reactor (epoll)|Proactor (io_uring/IOCP)|
|---|---|---|
|Notification|FD is ready|Operation complete|
|Data location on notify|Kernel buffer|App buffer|
|Who calls read()|Your app|The kernel|
|Syscall overhead|One extra per event|Eliminated|
|Complexity|Simpler|More complex|
|Used by|Nginx, Node.js, Redis|Windows IOCP, Linux io_uring|

---

## 8. Real-World Architectures

### Nginx Architecture

```
Master Process
    │
    ├── Worker Process 0  (CPU core 0, own epoll, ~25K connections)
    ├── Worker Process 1  (CPU core 1, own epoll, ~25K connections)
    ├── Worker Process 2  (CPU core 2, own epoll, ~25K connections)
    └── Worker Process 3  (CPU core 3, own epoll, ~25K connections)
                                    Total: ~100K connections
```

Each worker is single-threaded with a non-blocking event loop. Nginx handles **100K+ concurrent connections** on modest hardware.

### Node.js Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Node.js Process                    │
│                                                      │
│  JavaScript (single-threaded event loop)             │
│       ↕  via libuv                                   │
│  ┌──────────────────────────────────────────────┐   │
│  │  libuv event loop (epoll/kqueue/IOCP)         │   │
│  │  Thread Pool (4 threads, for fs/crypto/DNS)  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

Node.js is single-threaded in JavaScript land, but libuv uses a thread pool for operations the OS doesn't support asynchronously (like file I/O, DNS).

---

## 9. OS-Level Tuning

Even with the right architecture, you need to tune the OS to allow 10K+ connections.

```bash
# 1. File descriptor limits (per process)
ulimit -n 65536
# Or permanently in /etc/security/limits.conf:
# * soft nofile 65536
# * hard nofile 65536

# 2. System-wide file descriptor limit
sysctl -w fs.file-max=2000000

# 3. TCP tuning — allow more pending connections
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# 4. Port reuse
sysctl -w net.ipv4.tcp_tw_reuse=1

# 5. Increase ephemeral port range
sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# 6. Kernel receive/send buffer sizes
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
```

---

## 10. How Modern Systems Solved C10K

|Technology|Year|Approach|
|---|---|---|
|Apache httpd|1995|Thread/process per connection (the problem)|
|Nginx|2004|Async event loop, multiple workers|
|Node.js|2009|Single-threaded async JS event loop (libuv)|
|Tornado|2009|Python async I/O|
|Go|2009|Goroutines (M:N threading, non-blocking)|
|Redis|2009|Single-threaded event loop|
|Netty (Java)|2004|NIO event-driven Java framework|
|Rust/Tokio|2019|Async/await on top of epoll|
|io_uring|2019|Linux kernel async I/O (proactor model)|

---

## 11. The C10M Problem

C10K was solved. By the 2010s, the new challenge was **C10M — 10 million simultaneous connections**.

At that scale, even the OS kernel becomes a bottleneck:

- Syscall overhead for every packet
- Network interrupt handling burns CPU
- TCP/IP stack in the kernel is too general-purpose

**Solutions:**

- **DPDK (Data Plane Development Kit)**: bypass the kernel entirely, handle packets in user space
- **AF_XDP**: Linux kernel's express data path, zero-copy networking
- **io_uring**: shared memory ring buffers, avoids most syscalls, bridges Reactor and Proactor
- **Kernel bypass networking**: the NIC DMA's data directly to user-space memory

At C10M scale, systems like financial trading platforms and DDoS protection services operate. This is the domain of specialized hardware and custom kernel modules.

---

## 12. System Design Implications

Understanding C10K shapes how you design and reason about systems.

### Load Balancing

```
Clients → [Load Balancer] → Server 1 (10K connections)
                          → Server 2 (10K connections)
                          → Server 3 (10K connections)
```

Even with efficient servers, horizontal scaling distributes connection load.

### Long-Polling and WebSockets

HTTP long-polling holds connections open — this is _exactly_ the C10K scenario. WebSockets are persistent connections. Both require efficient I/O multiplexing on the server.

### Database Connection Pools

Databases can't handle 10K threads either. Connection poolers (PgBouncer, ProxySQL) solve the "C10K for databases" problem by multiplexing many application connections onto a few DB connections.

### Choosing Your Stack

|Use Case|Recommended|Why|
|---|---|---|
|High-concurrency API|Node.js, Go, Rust|Async-native, low overhead|
|CPU-heavy processing|Thread pool/workers|Don't block event loop|
|Microservices gateway|Nginx, Envoy|Event-driven, battle-tested|
|Real-time messaging|Go, Erlang/Elixir|Lightweight concurrency primitives|
|General web app|Go, Elixir, Rust|Good default concurrency models|

### The Golden Rule

> **Never block the event loop.** Long-running CPU work belongs in a worker/thread pool. The event loop should only orchestrate I/O.

---

## 13. Summary Cheat Sheet

```
THE C10K PROBLEM
────────────────
Problem:     Handle 10,000 concurrent connections on one server
Root cause:  Thread-per-connection: too much memory + context switching overhead
             Most threads are idle — sleeping waiting for I/O

THE MINDSET SHIFT
─────────────────
Old:  threads wait on connections
New:  one thread, connections notify it when ready (event loop)

THE NUMBERS
───────────
Thread cost:      ~1 MB stack
10K threads:      ~10 GB RAM → server dies
Event loop:       handles 10K+ connections with 1 thread + ~KB per connection

KEY CONCEPTS
────────────
Blocking I/O:       thread sleeps waiting → wasted resource
Non-blocking I/O:   returns EAGAIN immediately, thread stays free
I/O Multiplexing:   one call watches many FDs, returns only ready ones

SELECT → POLL → EPOLL EVOLUTION
────────────────────────────────
select():   O(n) scan of ALL FDs, 1024 FD hard limit
poll():     O(n) scan of ALL FDs, no FD limit
epoll():    O(1) per event — kernel hands you ONLY ready FDs
            registers FDs once, maintains internal ready-list

REACTOR vs PROACTOR
───────────────────
Reactor (epoll):    "fd is READY"     → data in kernel buffer → you call read()
Proactor (io_uring):"read COMPLETE"   → data in app buffer   → just use it

EVENT LOOP
──────────
Generic (Reactor):  while { epoll_wait → handle ready FDs }

Node.js / libuv phases (correct order):
  timers → pending → idle/prepare → poll → check → close
  ↕ Between phases: nextTick → Promise microtasks

Poll (Node):
  epoll_wait → ready FDs → libuv maps to callbacks → run on call stack (now)
  (no separate "I/O Callbacks" phase — I/O runs inside poll)

Details: [[Node.js Event Loop & Internals]]

WHO SOLVED C10K
───────────────
Nginx:   multi-process, each = single-threaded event loop (epoll)
Node.js: single-process event loop via libuv → epoll
Go:      goroutines (thousands of lightweight coroutines over epoll)
Redis:   single-threaded event loop (commands are always fast)

SYSTEM DESIGN LESSONS
─────────────────────
→ Use async-native servers for high-concurrency workloads
→ Use connection poolers (PgBouncer) between app and database
→ Never block the event loop — offload CPU work to worker queues
→ Tune OS limits (ulimit, somaxconn) before deploying at scale
→ WebSockets and long-polling are C10K problems in disguise
→ Horizontal scaling distributes connection load across servers
```

---

## Further Reading

- **Dan Kegel's original C10K paper**: http://www.kegel.com/c10k.html
- **Linux epoll man page**: `man 7 epoll`
- **libuv documentation** (Node.js's async I/O layer): https://libuv.org
- **io_uring introduction**: https://kernel.dk/io_uring.pdf
- **"The Architecture of Open Source Applications" — Nginx chapter**

---

## Related

- [[TCP & Connection Lifecycle]]
- [[HTTP Connections]]
- [[Worker]]

← [[Concepts MOC]]
