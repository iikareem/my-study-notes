---
tags:
  - operating-system
  - performance
  - backend
---

# Memory for Backend Engineers — From Zero

> Distilled from Ulrich Drepper, *What Every Programmer Should Know About Memory* (2007), rewritten so you can start with no hardware background and still leave with a complete mental model for backend work.
>
> Related notes: [[Memory Hierarchy Closest to CPU]], [[Virtual Memory]], [[Cache Line IO Block Read]], [[Mechanical Sympathy]]

---

## How to use this note

Read top to bottom once. Each section assumes only what came before.

| Part | What you get |
|------|----------------|
| **0** Prerequisites | Vocabulary + time scale |
| **1** The problem | Why memory dominates performance |
| **2** Caches | How CPUs hide RAM latency |
| **3** Virtual memory & TLB | Why addresses aren't free |
| **4** Multi-core & false sharing | Why "parallel" can be slower |
| **5** NUMA | Why big servers are uneven |
| **6** What to do in code | Layout, access patterns, threads |
| **7** How to measure | Don't guess |
| **8** Checklist | What changes how you think |

Skip deep DRAM chip timing. That is interesting hardware history, not day-to-day backend leverage.

---

## Part 0 — Prerequisites (start here if anything feels fuzzy)

### 0.1 What a program actually is

Your code becomes machine instructions. The CPU executes them. Almost every useful instruction eventually **reads or writes memory**.

```
Your code  →  compiler/runtime  →  machine instructions  →  CPU
                                                      ↕
                                                   memory
```

In a backend service (Node, Go, JVM, Python, etc.) you rarely touch physical addresses. The **OS and hardware** still do. Your data layout and access patterns decide whether the machine stays fast or stalls.

### 0.2 Units you need

| Term | Meaning |
|------|---------|
| **Bit** | 0 or 1 |
| **Byte** | 8 bits |
| **Word** | Natural chunk the CPU likes (often 8 bytes on 64-bit) |
| **Address** | Number that names a byte in memory |
| **Pointer** | Value that stores an address (and then you follow it) |

**Cache line** ≈ **64 bytes** on almost all modern x86/ARM servers.  
**Page** ≈ **4 KB** by default (sometimes 2 MB / 1 GB huge pages).

Remember those two sizes. Most of this note is about them.

### 0.3 Process memory layout (what "RAM for my app" means)

Each process gets a **private virtual address space**. Rough map (low → high address, simplified):

```
High addresses
┌─────────────┐
│   Stack     │  local vars, call frames  (grows down)
├─────────────┤
│   (gap)     │
├─────────────┤
│   Heap      │  malloc / new / JS objects  (grows up)
├─────────────┤
│   Data/BSS  │  globals / statics
├─────────────┤
│   Code      │  instructions
└─────────────┘
Low addresses
```

Important: **heap objects are often scattered**. Contiguous arrays are friendlier to caches than linked lists / object graphs.

### 0.4 Virtual vs physical (one-sentence version)

- Your program uses **virtual addresses**.
- Hardware (MMU) + OS page tables map them to **physical RAM**.
- This gives isolation between processes and lets the OS oversubscribe / swap.

→ Details in [[Virtual Memory]]. Here we only need: translation has a cost, and that cost is cached by the **TLB**.

### 0.5 Time: the only table that must stick

Approximate latencies (order of magnitude):

```
Register access              ~0.3 ns
L1 cache hit                 ~1–4 ns
L2 cache hit                 ~4–10 ns
L3 cache hit                 ~30–40 ns
Main RAM                     ~100 ns
SSD read                     ~100,000 ns   (0.1 ms)
Network same datacenter      ~500,000+ ns  (0.5+ ms)
```

**Rule of thumb:** RAM is ~100× slower than L1.  
If your hot path keeps missing cache, no clever Big-O tweak fully saves you.

### 0.6 Locality (the law of the land)

Hardware is built around two bets:

1. **Temporal locality** — if you used `x` just now, you'll use `x` again soon.
2. **Spatial locality** — if you used address `A`, you'll soon use `A+1`, `A+2`, …

Caches, prefetchers, and pages all exist because of these bets.  
**Backend consequence:** sequential scans over compact data win; pointer-chasing over huge heaps lose.

---

## Part 1 — The one mental model

```
CPU wants data in ~1 ns
RAM delivers data in ~100 ns
```

That gap is the problem. Everything else — L1/L2/L3, prefetchers, TLB, NUMA — is a workaround.

**Old question:** "Is my algorithm O(n)?"  
**Better question:** "Does my **working set** fit in L2/L3, and do I walk it sequentially?"

Working set = the bytes you actually touch in a hot loop / request path.

---

## Part 2 — CPU caches (§3)

### 2.1 The hierarchy

```
CPU core
  ├─ L1  ~32–64 KB     private, tiny, fastest
  ├─ L2  ~256 KB–1 MB  usually private
  └─ L3  ~4–64 MB      shared across cores on the chip
         │
         ▼
       RAM (GBs)
```

You never "call the cache." You only choose **what data is hot** and **in what order** you touch it. The hardware fills/evicts cache for you.

> Perspective: you don't control RAM access directly. You control **which data stays in cache**.

### 2.2 Cache lines — the real unit of transfer

When the CPU needs one `int64` (8 bytes) and it isn't in cache (**cache miss**), it does **not** fetch 8 bytes from RAM. It fetches a whole **cache line** (~64 bytes) around that address.

Why: spatial locality. Nearby bytes are often needed next.

```
You asked for 1 field (8B)
Hardware brought 64B
```

**Backend implications:**

- Fields packed next to each other travel together for free.
- Unused fields in the same object still waste cache space.
- The opposite of packing: two unrelated hot fields in the **same** 64B line can hurt under multi-core (false sharing — Part 4).

### 2.3 Hits, misses, and cliffs

Drepper-style intuition (numbers vary by CPU; shape is stable):

```
Working set fits in L1 → ~few cycles / element
Fits in L2             → slower step
Fits in L3             → another step
Larger than L3         → RAM latency (~100 ns) dominates
```

These are **cliffs**, not gentle slopes. Crossing a cache size boundary can make the same O(n) loop many times slower.

### 2.4 Associativity (why "fits in size" isn't enough)

A cache is not a free-for-all bag of 64B lines. Addresses map into **sets**. Each set holds only a few lines (**ways**, e.g. 8-way).

```
Address bits:  [ tag | set index | offset within line ]
                         ↑
                   picks which set
```

If many hot addresses map to the **same set**, they evict each other even when total data "should fit." That is **conflict miss** / thrashing.

> Practical rare case: two large power-of-two buffers with the same alignment fighting for the same sets. A small padding/offset between them can fix it. Don't optimize this until measured — but know it exists.

### 2.5 Writes: write-back (what modern CPUs do)

- **Write-through:** every write also goes to lower levels / RAM — simple, slow.
- **Write-back (normal):** write updates the cache line; RAM is updated later on eviction.

So repeated writes to the **same** hot object can stay in L1 and feel almost free.  
Scattered writes across many different addresses pay eviction / bandwidth costs.

### 2.6 Critical word (field order matters a little)

On a miss, the CPU needs the **exact word it asked for** first ("critical word"), then the rest of the line.

> Put frequently read fields **first** in hot structs/objects when you control layout (C/C++/Rust/Go structs; less control in JS objects, but splitting hot/cold objects still helps).

### 2.7 Bandwidth vs latency

Latency = how long one miss waits.  
Bandwidth = how much data the memory path can move per second.

Many cores missing at once **contend for memory bandwidth**. Sequential access by fewer cores often beats random access by many.

---

## Part 3 — Virtual memory & the TLB (§4)

### 3.1 Why this is a performance topic

Every load/store uses a **virtual** address. Translating to physical needs a **page-table walk** (multiple memory reads) unless the translation is cached.

The cache for translations is the **TLB** (Translation Lookaside Buffer). It is **small**.

### 3.2 Pages and TLB coverage

Default page = **4 KB**.

Example:

```
64 TLB entries × 4 KB  ≈ 256 KB of address space "easy"
```

If your hot data touches many scattered pages, TLB misses stack up. Each miss can cost on the order of a RAM access again.

### 3.3 How to help the TLB

1. **Huge pages** (2 MB / 1 GB)  
   Same TLB entry covers much more memory → fewer misses.  
   Used by Redis, Postgres, JVMs, databases when tuned.  
   Linux: transparent huge pages / `madvise(MADV_HUGEPAGE)` / explicit huge pages.

2. **Compact, contiguous allocations**  
   Pool allocators, arenas, arrays beat millions of tiny scattered allocations for TLB (and cache).

3. **Don't ignore page faults**  
   First touch / cold pages / swapping are separate from cache misses but show up as latency spikes. Measure major/minor faults under load.

> TLB problems are often invisible in "CPU %" alone. You need hardware counters or specialized profiling.

---

## Part 4 — Multi-core coherence & false sharing (§3.3.4, §6.4)

### 4.1 Each core has private L1 (usually)

Caches must stay **coherent**: two cores cannot disagree about the same memory forever. Protocols like **MESI** (Modified / Exclusive / Shared / Invalid) coordinate ownership of cache lines.

Key rule for intuition:

> Only one core can hold a line in a writable ("Modified") state. Other writers force the line to move.

### 4.2 False sharing (critical for backend concurrency)

```
Core 0 writes field a  ─┐
Core 1 writes field b  ─┴─ a and b live in the SAME 64-byte cache line
```

Logically independent. Hardware treats them as one line. The line **ping-pongs** between cores → parallel code runs like serial (or worse).

**Fix:** give each frequently written counter / worker slot its **own cache line** (pad to 64 bytes).

```c
// Bad — shares a line
struct { int64_t a; int64_t b; };

// Good — separate lines
struct {
  int64_t a;
  char pad0[56];
  int64_t b;
  char pad1[56];
};
```

Node `SharedArrayBuffer` pattern:

```js
// Worker i only touches offset i * 64
const buf = new SharedArrayBuffer(64 * NUM_WORKERS);
```

Same idea as sharded counters in Prometheus client libraries.

### 4.3 Atomics and locks are memory traffic

Uncontended atomics ≈ cheap.  
Contended atomics / mutexes ≈ cache-line bouncing.

Design:

- Prefer **partitioned** state (per-worker, per-shard) + merge later.
- Prefer message passing / queues with clear ownership over shared mutable hot fields.
- "More threads" on one shared counter often **reduces** throughput.

---

## Part 5 — NUMA (§5, §6.5)

On multi-socket machines, RAM is attached to sockets:

```
Socket 0 CPU  ←fast→  RAM 0
     ↕ interconnect (slower)
Socket 1 CPU  ←fast→  RAM 1
```

- Local memory: faster  
- Remote memory: often ~2×+ slower (varies by machine)

OS usually allocates memory near the CPU that **first touched** it. If threads later run on the other socket, you pay remote latency silently.

```bash
# Pin process to one NUMA node (example)
numactl --cpubind=0 --membind=0 node server.js
```

When it matters: high-QPS services, in-memory DBs, large caches on 2+ socket hosts.  
When it often doesn't: small single-socket cloud VMs.

---

## Part 6 — What programmers can do (§6)

This is the payoff chapter. Four rules cover most backend wins.

### Rule 1 — Keep hot data compact

Less bytes touched → more of the working set fits in L1/L2/L3.

- Drop unused fields from hot objects.
- Prefer smaller types when safe.
- Don't carry large strings/blobs inside records you scan millions of times.

### Rule 2 — Access sequentially when you can

Hardware **prefetchers** detect strides and fetch the next lines early.

```
Sequential array scan  → prefetcher hides much of the latency
Random pointer chasing → miss after miss (~100 ns each)
```

Classic example: matrix multiply loop order. Same Big-O; different access order; large speed difference because one walks memory row-wise and the other thrashes columns.

Backend analogues:

- Scan a typed array / buffer, not a linked list of heap objects.
- Prefer contiguous columns for aggregations (see SoA below).
- Cursor/pagination over packed rows beats random primary-key hops when you control the format.

### Rule 3 — Split hot and cold fields

```js
// Mixed: every session scan pulls bio/preferences into cache
const user = { id, lastSeen, name, bio, preferences };

// Split: scan only what the hot path needs
const hot  = { id, lastSeen };
const cold = { name, bio, preferences };
```

When you iterate 100k sessions for expiry, hot-only structs pack more records per cache line → fewer misses.

### Rule 4 — AoS vs SoA

**Array of Structures (AoS)** — good for "fetch one record":

```js
[{ id, price, qty, notes }, { id, price, qty, notes }, ...]
```

**Structure of Arrays (SoA)** — good for "aggregate one field across all":

```js
{ ids: [...], prices: [...], qtys: [...], notes: [...] }
```

Summing `prices` over 100k orders with SoA touches mostly the prices array — not notes.

| Access pattern | Prefer |
|----------------|--------|
| One full entity | AoS |
| One field across many | SoA / columnar |

### Working set vs L3 (in-process cache myth)

```
10M entries × 200 bytes ≈ 2 GB  →  far beyond L3
```

"In-process = always faster than Redis" is false once the working set thrashes RAM and TLB. Hit rate + access pattern decide. Measure.

### Prefetching (know the idea; don't hand-tune first)

- **Hardware prefetch:** automatic for predictable strides.
- **Software prefetch:** explicit hints; easy to misuse; use after profiling.

For backend: make patterns **predictable** first. That is usually enough.

---

## Part 7 — How to measure (§7)

Without measurement, memory advice becomes cargo cult.

| Goal | Approaches (Linux / typical) |
|------|------------------------------|
| CPU + cache behavior | `perf stat`, `perf record` (cache-misses, LLC misses) |
| Allocations / RSS growth | language allocators, `massif`-style tools, runtime metrics |
| Page faults | `/proc/<pid>/stat`, `perf`, language GC / fault metrics |
| Teaching / simulation | Cachegrind (simulation — directionally useful, not exact HW) |

What to watch on a hot endpoint:

1. Cache miss rate / LLC misses  
2. Allocation rate & GC pauses (managed runtimes)  
3. Minor/major page faults  
4. Lock / atomic contention  
5. Working set size vs machine cache sizes  

> Optimize the path the counters point at — not the story you prefer.

---

## Part 8 — Perspective shifts (keep this table)

| Old thinking | New thinking |
|--------------|--------------|
| Fits in RAM → fast | Fits in **L3** (and TLB) → fast |
| Big-O is enough | Same Big-O can differ **10–50×** by layout/access |
| Random indexing is fine | Random = miss-heavy |
| More threads = more speed | Shared hot lines = bounce / stall |
| In-process cache always beats Redis | Depends on working set + hit pattern |
| Fat objects are convenient | Hot/cold mix wastes cache lines |
| Atomics are "just correctness" | Contended atomics are **interconnect traffic** |
| NUMA is only for DBAs | Any hot service on multi-socket can feel it |

---

## Study path (backend-focused)

1. Part 0–1 until latencies + locality feel obvious  
2. Part 2 (caches + cache lines) — highest ROI  
3. Part 3 (TLB) — short but important  
4. Part 4 (false sharing / atomics) — if you use threads / shared memory  
5. Part 6 rules — apply to one real hot path at work  
6. Part 7 — measure that path  
7. Part 5 — only when you deploy on multi-socket / large hosts  

Optional later from the original paper: deep DRAM protocols (§2), upcoming-2007 tech (§8).

---

## One-page cheat sheet

```
Unit of cache transfer     = 64B cache line
Unit of address translation= 4KB page (or huge page)
Speed cliff                = working set vs L1/L2/L3
Win pattern                = compact + sequential + partitioned
Lose pattern               = scattered pointers + shared hot lines
Multi-core tax             = false sharing / contended atomics
Big machine tax            = remote NUMA memory
Always                     = measure misses, faults, contention
```

---

_Source: Ulrich Drepper, "What Every Programmer Should Know About Memory", 2007_  
_Emphasis: §§3, 4.3, 5, 6.2, 6.4, 7 — rewritten from zero for backend engineers_
