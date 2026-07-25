# TCP Connections, Ephemeral Ports, File Descriptors & Connection Pooling


---

## 1. What Identifies a TCP Connection?

A TCP connection is **not** identified by an IP or a port alone. It is identified by a **4-tuple**:

```
(Source IP, Source Port, Destination IP, Destination Port)
```

Every unique combination of these four values is a **unique connection** in the kernel's eyes.

Example — your app talking to a database:

```
Source IP:   10.0.0.5      ← your app server
Source Port: 49201         ← ephemeral port (OS picked it)
Dest IP:     10.0.0.9      ← your database server
Dest Port:   5432          ← PostgreSQL (always fixed)
```

---

## 2. What Are Ephemeral Ports?

When your application opens a TCP connection without binding to a specific port, the OS automatically assigns a free port from the **ephemeral port range**.

- Linux default range: `32768 – 60999` → ~28,000 available ports
- Check yours: `cat /proc/sys/net/ipv4/ip_local_port_range`

This assigned port is the **Source Port** in the 4-tuple. It lives only for the duration of the connection — hence "ephemeral" (temporary).

**Important:** The OS picks a new, different ephemeral port on every `connect()` call — even if the destination IP and port are identical. Two connections with the same 4-tuple would be indistinguishable to the kernel, so it never allows it.

```
Request 1:  (10.0.0.5 : 49201)  →  (10.0.0.9 : 5432)   ← unique connection
Request 2:  (10.0.0.5 : 49202)  →  (10.0.0.9 : 5432)   ← unique connection
Request 3:  (10.0.0.5 : 49203)  →  (10.0.0.9 : 5432)   ← unique connection
```

---

## 3. File Descriptors (FD)

### Everything is a file in Unix/Linux

The kernel treats every open resource — files, sockets, pipes, devices — as a "file" and represents each one with a **file descriptor**: a small integer that indexes into a per-process table.

```
fd=0  →  stdin
fd=1  →  stdout
fd=2  →  stderr
fd=3  →  a real file       (open("/tmp/data.txt"))
fd=4  →  a socket          (connect() to DB)
fd=5  →  a pipe            (IPC between processes)
fd=6  →  a device          (/dev/null, /dev/urandom)
```

Your app doesn't care what's underneath — it just calls `read(fd)` or `write(fd)` and the kernel figures out what to actually do.

### The three-layer picture

```
Your Process
┌──────────────────────────┐
│  Per-Process FD Table    │
│  fd=0  → stdin           │
│  fd=1  → stdout          │
│  fd=4  → socket ─────────────────────────────┐
└──────────────────────────┘                   │
                                               ▼
Kernel                                Kernel Socket Entry
                             ┌──────────────────────────────────┐
                             │ type: SOCKET                     │
                             │ protocol: TCP                    │
                             │ 4-tuple: (app:49201 → db:5432)   │
                             │ recv buffer: [...]               │
                             │ send buffer: [...]               │
                             └──────────────────────────────────┘
                                               │
                                               ▼
                                      Network Stack
                                   (routes packets in/out)
```

### The complete chain

```
App → FD → Socket → Ephemeral Port → TCP Connection → Database
```

Note: not every FD has a port — only socket FDs do.

```
FD type  │ Has Port?
─────────┼───────────────────────────────────
File     │ No
Pipe     │ No
Device   │ No
Socket   │ Yes (ephemeral if client, fixed if server)
```

---

## 4. How Responses Find Their Way Back to Your App

When your app calls `connect()`, the OS does two things:

1. Picks an ephemeral port (e.g. `49201`) and assigns it to that socket
2. Registers an entry in the **kernel socket table** — mapping the 4-tuple to an FD

```
Kernel Socket Table
┌──────────────────────────────────────────────┬────────────┐
│ 4-Tuple                                      │ Socket FD  │
├──────────────────────────────────────────────┼────────────┤
│ (10.0.0.5:49201) → (10.0.0.9:5432)          │ fd=4       │
│ (10.0.0.5:49202) → (10.0.0.9:5432)          │ fd=5       │
│ (10.0.0.5:49203) → (10.0.0.9:5432)          │ fd=6       │
└──────────────────────────────────────────────┴────────────┘
```

When the DB sends a response, it puts the ephemeral port in the **destination field** of the packet (it learned it from your SYN packet during the handshake):

```
Incoming packet:
  Src IP:   10.0.0.9   ← DB
  Src Port: 5432
  Dst IP:   10.0.0.5   ← your app
  Dst Port: 49201      ← the ephemeral port OS assigned
```

The OS looks up `Dst Port 49201` in the socket table → finds `fd=4` → delivers data to your app's `read()` call on that fd. Your app never touches ports directly — it just reads from a file descriptor.

```
DB response for query 1  →  Dst Port 49201  →  OS maps to fd=4  →  your thread 1
DB response for query 2  →  Dst Port 49202  →  OS maps to fd=5  →  your thread 2
DB response for query 3  →  Dst Port 49203  →  OS maps to fd=6  →  your thread 3
```

---

## 5. The Life Cycle of One TCP Connection

```
App                         OS Kernel                      Database
 |                               |                              |
 |-- connect(db_url) ----------->|                              |
 |                     picks port 49201                         |
 |                               |--- SYN --------------------->|
 |                               |<-- SYN-ACK ------------------|
 |                               |--- ACK --------------------->|
 |                               |   (TCP Handshake complete)   |
 |<-- connection ready (fd=4) ---|                              |
 |                                                              |
 |-- SQL query (write to fd=4) --------------------------------->|
 |<-- result   (read  from fd=4) -------------------------------|
 |                                                              |
 |-- close(fd=4) ----------->|                                  |
 |                           |--- FIN --------------------->    |
 |                           |<-- FIN-ACK ------------------|   |
 |                           |--- ACK --------------------->    |
 |                           port 49201 enters TIME_WAIT (~60s) |
 |                           port 49201 BLOCKED for reuse       |
```

---

## 6. TIME_WAIT

When a TCP connection is closed, the side that sent the first FIN enters `TIME_WAIT` state.

- Duration: **2 × MSL** (Maximum Segment Lifetime) = typically **60 seconds** on Linux
- During this time: the port **cannot be reused** for the same 4-tuple
- Purpose: ensures stale packets from the dead connection don't corrupt a new one

### Why same destination makes TIME_WAIT worse

TIME_WAIT is scoped to the **full 4-tuple**. If you always connect to the same DB IP and port, your ephemeral port stays blocked for the full 60 seconds. Connecting to a different destination IP would free the port sooner (different 4-tuple = no conflict). Same destination = stricter contention.

---

## 7. The Problem: New Connection Per Request

```
Time 0.0s  → Request 1  → port 49201 used → closed → TIME_WAIT 60s
Time 0.1s  → Request 2  → port 49202 used → closed → TIME_WAIT 60s
Time 0.2s  → Request 3  → port 49203 used → closed → TIME_WAIT 60s
...
```

At 500 req/sec you burn through ~28,000 ports in under **60 seconds**. After that:

```
connect() → EADDRNOTAVAIL: Cannot assign requested address
```

The OS has no free ephemeral port to assign and new connections fail or hang.

---

## 8. The Limits Stack

There are three independent walls you can hit:

```
Your Process
┌────────────────────────────────────────┐
│  Per-Process FD Limit                  │
│  default: 1024 (soft) / 4096 (hard)   │  ← ulimit -n
│  error:   EMFILE (too many open files) │
└────────────────────────────────────────┘
                    ↓
Kernel
┌────────────────────────────────────────┐
│  System-Wide FD Limit                  │
│  /proc/sys/fs/file-max                 │  ← e.g. 1,000,000
│  error:   ENFILE                       │
└────────────────────────────────────────┘
                    ↓
Network Stack
┌────────────────────────────────────────┐
│  Ephemeral Port Range                  │
│  32768 – 60999 = ~28,000 ports         │  ← ip_local_port_range
│  error:   EADDRNOTAVAIL                │
└────────────────────────────────────────┘
```

|Limit|Default|Error When Hit|
|---|---|---|
|Per-process FD limit|~1,024|`EMFILE: too many open files`|
|System-wide FD limit|~1,000,000|`ENFILE`|
|Ephemeral port range|~28,000|`EADDRNOTAVAIL`|

---

## 9. The Solution: Connection Pooling

A connection pool opens N connections **once at startup** and reuses them forever.

```
Pool opens at startup (N=3):
  fd=4 → socket → port 49201 → db:5432   (stays OPEN)
  fd=5 → socket → port 49202 → db:5432   (stays OPEN)
  fd=6 → socket → port 49203 → db:5432   (stays OPEN)

Request 1 → borrows fd=4 → query → returns fd=4 to pool
Request 2 → borrows fd=5 → query → returns fd=5 to pool
Request 3 → borrows fd=4 (reused!) → query → returns fd=4 to pool
```

Because connections are **never closed**, TIME_WAIT never triggers. The same 3 FDs and 3 ephemeral ports handle unlimited requests with zero new TCP connections.

---

## 10. Full Comparison

||New Connection Per Request|Connection Pool|
|---|---|---|
|FDs consumed|1 per request, constantly opening/closing|Fixed N, open forever|
|Ephemeral ports consumed|1 per request, burn through ~28k in <60s|Fixed N, never released|
|TIME_WAIT triggered|Yes, on every close|Never|
|TCP handshake cost|Every request (3 packets)|Once at startup|
|Port exhaustion risk|High under load|None|
|DB connections at a time|Unbounded (until failure)|Controlled, bounded|

---

## 11. The Mental Model in One Diagram

```
Your App Process
│
├── fd=0  → stdin                          (no port)
├── fd=1  → stdout                         (no port)
├── fd=2  → stderr                         (no port)
├── fd=3  → log file                       (no port)
│
├── fd=4  → socket → src port: 49201  →  db:5432
├── fd=5  → socket → src port: 49202  →  db:5432
└── fd=6  → socket → src port: 49203  →  db:5432

Full chain:
App → FD → Socket → Ephemeral Port → TCP Connection → Database
```

---

## 12. Key Takeaways

- A TCP connection = **(src IP, src port, dst IP, dst port)** — all four matter
- Every `connect()` gets a **new ephemeral port**, regardless of destination
- Same DB URL does **not** mean same source port — ever
- Same destination = **stricter** TIME_WAIT contention, not less
- Each connection = one **FD** in your process
- FDs represent any open resource (file, socket, pipe, device) — only socket FDs have ports
- The OS routes responses back using the destination port → kernel socket table → FD → your app
- You have three independent limits: per-process FDs, system FDs, ephemeral ports
- Connection pooling solves all three by keeping a fixed number of connections open permanently

---

## Related

- [[TCP & Connection Lifecycle]]
- [[HTTP Connections]]
- [[TCP TIME_WAIT — Recap]]
- [[C10K Problem]]

← [[Concepts MOC]]
