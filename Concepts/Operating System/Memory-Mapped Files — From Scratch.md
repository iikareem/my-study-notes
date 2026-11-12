# Memory-Mapped Files — Full Recap

---

## 1. The Foundation: Virtual Memory & Page Table

Every process **never** accesses physical RAM directly. It always goes through a layer of indirection:

```
Your Process          Page Table (middleware)       Physical RAM
┌────────────┐        ┌──────────────────────┐      ┌─────────────┐
│ virtual    │ ─────► │ 0x7FFF_1000          │ ───► │ 0x3A2_0000  │
│ address    │        │   → 0x3A2_0000       │      │ [ bytes ]   │
└────────────┘        └──────────────────────┘      └─────────────┘
```

- **Virtual address** = what your code works with (lives in your heap, stack, etc.)
- **Page table** = the OS-managed map that translates virtual → physical
- **Physical RAM** = where bytes actually live
- **Page** = the unit of mapping = **4 KB chunk**
- The CPU does this translation **automatically and invisibly** on every single memory access

> Your variables, buffers, code — everything in your process is virtual. The page table is always the middleman.

---

## 2. Normal File I/O — The Double Copy Problem

When you call `fs.readFile()` in Node.js:

```
Disk
 │
 │  DMA (hardware transfer)
 ▼
Kernel Page Cache (RAM)       ← COPY 1 — file loaded here first
 │
 │  memcpy() across protection boundary
 ▼
Your Heap Buffer (RAM)        ← COPY 2 — now you can use it
 │
 │  page table lookup
 ▼
Your Code reads buf[0] ✅
```

**Two copies of the same data exist in RAM simultaneously:**

```
Physical RAM
┌──────────────────────────────────────────────┐
│  0x5C2_0000  [ file bytes ]  kernel cache    │  ← copy 1
│  0x3A2_0000  [ file bytes ]  your buffer     │  ← copy 2
└──────────────────────────────────────────────┘
```

**Why the copy?** Because your process cannot touch kernel space memory. The CPU enforces a hard protection boundary (Ring 0 vs Ring 3). The kernel must physically copy bytes across that boundary into your process's own RAM pages.

---

## 3. Kernel Space vs User Space — Same RAM, Different Protection

This is a critical point many people get wrong:

- Kernel space and user space are **not different hardware**
- They are **different regions of the same physical RAM**
- The CPU simply **forbids** your process from reading kernel space addresses
- Violation = instant crash (segfault)

```
Physical RAM (one pool)
┌─────────────────────────────────────────────────────────┐
│  Kernel region  [ page cache, kernel code, kernel stack ]│  ← your process: ❌
│  User region    [ your heap, your stack, your code ]     │  ← your process: ✅
└─────────────────────────────────────────────────────────┘
```

---

## 4. Memory-Mapped Files — The Fix

`mmap()` tells the OS:

> "Don't copy anything. Just point my page table directly at your page cache page."

```
With normal read():

Page table:  0x7FFF_1000  →  0x3A2_0000  (your heap buffer)
                                 ↑
                           kernel did memcpy() here from 0x5C2_0000

With mmap():

Page table:  0x7FFF_1000  →  0x5C2_0000  (kernel page cache directly)
                                 ↑
                           NO memcpy. same physical page. done.
```

**Result:**

```
Physical RAM
┌──────────────────────────────────────────────┐
│  0x5C2_0000  [ file bytes ]  ONE copy only   │  ← both kernel and your
└──────────────────────────────────────────────┘     process point here
```

---

## 5. The Lazy Loading Detail (Page Fault)

`mmap()` does NOT load the file into RAM immediately. It only reserves virtual address space.

```
mmap() called
  → OS reserves virtual addresses 0x7FFF_0000 to 0x8FFF_0000
  → NO data loaded yet. Zero RAM used.

You access buf[0]  (virtual 0x7FFF_0000)
  → CPU checks page table → page not loaded → PAGE FAULT
  → OS intercepts, loads 4KB from disk into page cache
  → Updates page table: 0x7FFF_0000 → 0x5C2_0000
  → Your code continues, sees the byte ✅

You access buf[4096]  (next page)
  → if already in page cache → instant, no disk
  → if not → another page fault, loads that 4KB
```

**You only pay for pages you actually touch.** A 10GB file mapped with mmap costs 0 RAM until you access a byte.

---

## 6. The Complete Comparison

```
Normal read():
Disk → kernel page cache → memcpy() → your heap buffer → page table → you
                           ^^^^^^^^
                           expensive, duplicate RAM

mmap():
Disk → kernel page cache → page table points here → you
                           ^^^^^^^^^^^^^^^^^^^^
                           no copy, lazy, single RAM page
```

||Normal `read()`|`mmap()`|
|---|---|---|
|Copies|2|0 (after page fault)|
|RAM usage (10GB file)|up to 20GB|only pages touched|
|Random access|seek() + read() per chunk|just dereference pointer|
|Process sharing|needs IPC/socket/pipe|point two processes at same page|
|Disk access|on every read() if not cached|only on first access (page fault)|
|Lazy loading|❌|✅|

---

## 7. Real-World Use Cases

---

### 7.1 SQLite

**What it does:** SQLite is an embedded database that stores everything in a single `.db` file. It uses a B-tree structure internally, meaning it needs to jump around the file randomly to traverse nodes.

**Without mmap:**

```
SQLite wants page 482 of the B-tree
  → pread() syscall
  → kernel copies page into SQLite's internal buffer
  → B-tree code reads from buffer
  → repeat for every single page traversal
```

Every node traversal = 1 syscall + 1 copy.

**With mmap (`PRAGMA mmap_size = 268435456`):**

```
SQLite wants page 482
  → dereference pointer at mapped address
  → OS loads page if not in RAM (page fault, first time only)
  → B-tree code reads directly
  → next traversal of same page = pure memory speed
```

**Why they use it:** Read-heavy workloads (SELECT queries traversing the B-tree) see **2–5x throughput improvement** because hot pages stay in the page cache and are accessed at memory speed with zero syscalls after the first load.

---

### 7.2 LMDB (Lightning Memory-Mapped Database)

**What it is:** The database engine used by OpenLDAP, and also used by Node.js via the `lmdb` npm package. It's literally named after mmap.

**How it works:** LMDB maps its **entire database file** into virtual address space from the start. There is no internal buffer pool or page cache in LMDB itself — it fully delegates that job to the OS page cache via mmap.

```
LMDB architecture:

Database file on disk
       │
       │  mmap() on startup
       ▼
Virtual address space of process
  [  entire DB  ]  ← just pointers, OS handles the rest
       │
       │  access any record
       ▼
OS page cache (lazy loaded 4KB at a time)
```

**Why they use it:**

- **Zero-copy reads** — reads are pointer dereferences, not function calls
- **OS manages caching** — no need to write their own buffer pool
- **Multiple readers, zero locks** — multiple processes can mmap the same file read-only, sharing the exact same physical RAM pages
- **Crash safety** — LMDB uses Copy-on-Write + OS-level page flushing for ACID guarantees

**Result:** LMDB is consistently one of the fastest key-value stores in benchmarks for read-heavy workloads.

---

### 7.3 Redis (AOF and RDB persistence)

**What it does:** Redis is in-memory but needs to persist data to disk (either RDB snapshots or AOF append-only log).

**RDB (snapshot) — uses `fork()` + mmap behavior:**

```
Redis forks a child process to write the snapshot
  → fork() copies the page table (not the actual RAM pages)
  → Child and parent share the same physical pages (Copy-on-Write)
  → Child writes snapshot by reading pages (all via page table)
  → Parent continues serving requests
  → If parent modifies a page → OS makes a private copy for parent
     (this is Copy-on-Write, which is mmap's sibling concept)
```

Redis doesn't call `mmap()` explicitly here, but `fork()` uses the exact same page table sharing mechanism under the hood.

**AOF (append-only file) — uses mmap for reads:** When Redis restarts and replays the AOF file to reconstruct state, it can use mmap to read the potentially huge log file efficiently without loading it all into memory.

**Why they use it:** Fork-based snapshotting with Copy-on-Write means Redis can snapshot gigabytes of data with almost zero RAM overhead at fork time — the child shares all physical pages until it needs to read them.

---

### 7.4 PostgreSQL

**What it does:** PostgreSQL manages its own **shared buffer pool** — a region of shared memory that all PostgreSQL worker processes share to cache disk pages.

**How mmap fits in:**

```
PostgreSQL startup
  → allocates shared memory region (via mmap with MAP_SHARED + MAP_ANONYMOUS)
  → all worker processes (connections) map this same region
  → when worker 1 loads a table page → it goes into shared buffer
  → worker 2 querying the same table → finds it already in shared buffer
  → zero disk I/O, zero copy between workers
```

**Why they use it:** Every connection is a separate OS process in PostgreSQL. Without shared mmap memory, each process would need its own copy of cached pages. With shared mmap, all processes share one pool — dramatically reducing RAM usage and eliminating inter-process copies.

---

### 7.5 RocksDB / LevelDB (used by many Node.js apps)

**What it is:** RocksDB is the storage engine under Kafka, TiKV, MyRocks (MySQL), and many others. LevelDB is its predecessor and is used by Node.js's default `localStorage` in some environments and many npm packages.

**How mmap fits in:** RocksDB stores data in SST (Sorted String Table) files. When reading, it memory-maps these files.

```
Query comes in for key "user:123"
  → RocksDB checks bloom filter (in memory)
  → identifies which SST file contains the key
  → SST file is mmap'd
  → binary search through mmap'd index → pointer dereference
  → read value directly from mapped address
  → OS loads only the 4KB pages needed
```

**Why they use it:**

- SST files are immutable (written once, read many times) — perfect for mmap
- Random point lookups are just pointer arithmetic + page fault on first access
- Hot SST files stay in OS page cache automatically — no manual caching needed
- Multiple RocksDB instances can share the same SST page cache via the OS

---

### 7.6 Nginx (serving static files)

**What it does:** Nginx uses `sendfile()` on Linux which is conceptually the mmap idea applied to network I/O — it tells the kernel to send file pages directly to the network socket **without copying into user space at all**.

```
Normal file serving:
Disk → kernel page cache → memcpy() → Nginx buffer → write() → kernel socket buffer → network

sendfile() (zero-copy):
Disk → kernel page cache → kernel socket buffer → network
                           (Nginx user space never touched)
```

**Why they use it:** Serving a 100MB video file with normal I/O copies 100MB into Nginx's process RAM on every request. With sendfile/mmap-style zero-copy, the file bytes go straight from page cache to network — Nginx RAM usage stays flat regardless of file size or traffic.

---

## 8. The One Mental Model to Remember

```
Normal I/O:
  File data exists in TWO places in RAM
  (kernel page cache + your buffer)
  You got there via: syscall → memcpy → page table lookup

mmap:
  File data exists in ONE place in RAM
  (kernel page cache = your address space, same physical page)
  You got there via: page table lookup only

The page table is always the mechanism.
mmap just makes your page table skip the middleman.
```