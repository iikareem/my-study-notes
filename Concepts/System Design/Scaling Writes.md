---
tags:
  - mvcc
  - postgresql
  - scaling
  - sharding
  - system-design
  - wal
---

Scaling writes is fundamentally harder than scaling reads. Reads are inherently parallelizable — you can add replicas or cache nodes freely. Writes are not. Every write must land in exactly the right place, be durable, and remain consistent. The moment you spread writes across multiple nodes, coordination complexity explodes.

---

## Why Writes Are Hard

With reads, stale data is often acceptable. With writes, correctness is non-negotiable:

- **Every write must be routed to the right node** — a wrong route means data loss or corruption.
- **Writes cannot be freely duplicated** — unlike reads, duplicating a write creates inconsistency.
- **Contention is real** — multiple writers hitting the same record at the same time requires locking or conflict resolution.
- **Durability is required** — a write acknowledged to the client must survive crashes.

---

## The Full Write Path (PostgreSQL Model)

Every scaling decision is easier once you know what one write actually does. Nothing hits the “real” table files on the commit path — durability comes from the WAL; pages flush later.

```
CLIENT WRITE
     │
     ▼
[1] Buffer pool (RAM) — find/load 8KB pages
     │  miss → read from disk into RAM
     ▼
[2] XID + MVCC snapshot
     │
     ▼
[3] New row version on data page (RAM, dirty)
     │  old row stays; xmax set
     ▼
[4] Index pages updated (RAM, dirty)
     │  HOT may skip if column not indexed
     ▼
[5] Append WAL + fsync → client gets "committed"   ← durability happens here
     │
     ▼
[6] CHECKPOINT (periodic) — flush dirty pages to disk
     │
     ▼
[7] VACUUM (background) — reclaim dead row versions
```

### 1. Buffer pool (RAM)

Data and indexes live in **8KB pages**. The shared buffer pool caches hot pages in RAM.

- Page already in RAM → modify in place in memory.
- Page missing → **page fault**: read from disk into the buffer pool, then proceed.

> No write updates heap/index files on disk during the request. Everything goes through the buffer pool first.

### 2. MVCC snapshot

The transaction gets an **XID** and a snapshot of which transactions are visible. PostgreSQL **does not overwrite** the old row:

```
Old row:  xmin=100, xmax=NULL   ← visible before the update
New row:  xmin=201, xmax=NULL   ← visible after txn 201 commits
Old row:  xmin=100, xmax=201    ← dead version (deleted by txn 201)
```

- `xmin` = transaction that created this version
- `xmax` = transaction that deleted/updated it
- Readers use their snapshot — **reads don’t block writes** (and vice versa) for normal row versions

### 3. Modify data page in RAM

The new row version is written into the data page in the buffer pool. The page is marked **dirty**. Old and new versions coexist on the page until VACUUM.

### 4. Modify index pages in RAM

Every index covering changed columns must be updated — also in the buffer pool.

- **HOT (Heap Only Tuple):** if the updated columns are *not* in any index, Postgres can skip a new index entry and chain the new heap tuple from the old one — less index write amplification.
- If an indexed column changes, a new index entry is written; that index page is dirty too.

> After steps 3–4: data + index pages are dirty in RAM. Still nothing durable on disk.

### 5. WAL + fsync (commit)

Before the client hears “success,” the change is appended to the **Write-Ahead Log** and flushed:

- WAL is **sequential append-only** — cheapest disk write pattern.
- Records *what changed* (compact), not a full random page write.
- On commit: **`fsync()`** forces WAL to stable storage.
- Only after fsync returns is the write acknowledged.

```
Crash after WAL fsync, before checkpoint
  → replay WAL on restart → changes recovered

Dirty pages in RAM can be lost. WAL after fsync cannot.
```

### 6. Checkpoint (periodic)

Dirty pages eventually flush to their real locations on disk:

- Triggers: time (`checkpoint_timeout`) or WAL volume (`max_wal_size`)
- Flushes dirty heap + index pages
- WAL before the checkpoint can be recycled
- Spread via `checkpoint_completion_target` to avoid one giant I/O spike

```
Write → Write → Write → Write → CHECKPOINT
         ↑ WAL kept durability the whole time
```

### 7. VACUUM (background)

MVCC leaves dead row versions. **VACUUM** reclaims them:

- Frees tuples whose `xmax` is committed and invisible to all snapshots
- Updates the **visibility map** (enables index-only scans)
- **Autovacuum** runs from dead-tuple thresholds

> Default VACUUM marks space reusable inside the table file; it does **not** return disk to the OS. `VACUUM FULL` does, but rewrites and locks the table — rare in production.

---

## Cost of a Write (What Actually Matters)

Every step above is a durability vs performance trade-off. For interviews and capacity thinking, these are the costs that matter — bold = the ones that usually dominate under load.

| Mechanism | Protects against | Cost |
| --- | --- | --- |
| **WAL fsync on commit** | Data loss on crash | **Latency on every commit** — hard ceiling on durable writes/sec |
| **Index updates (per write)** | Fast lookups | **Write amplification** — one logical UPDATE → many page dirties |
| **Locks / hot rows** | Correct concurrent updates | **Serialization** — conflicting writers queue; spikes collapse throughput |
| **MVCC (no in-place overwrite)** | Readers not blocked by writers | **Dead-tuple bloat** → VACUUM CPU/IO; page bloat if autovacuum lags |
| Buffer pool (defer heap writes) | Random I/O on the commit path | RAM is volatile; relies on WAL; pressure → more evictions/faults |
| Checkpoint spreading | One huge flush stall | Longer recovery window; still causes **IO waves** under heavy write load |
| HOT optimization | Index write amplification | Only when updated columns are **not** indexed |

**Mental model:** commit latency ≈ WAL fsync + lock waits; sustained throughput ≈ how fast you can dirty pages, ship WAL, checkpoint, and vacuum without falling behind. That is where a single node spikes and dies.

---

## Why a Single Write Node Breaks (and Why That Creates Spikes)

Before sharding, almost every system has **one primary** that owns all writes. That design is simple and correct — until traffic grows.

### The single primary is a serial choke point

All of the important costs above share **one** machine:

| Resource | Tie-back to the write path |
| --- | --- |
| **WAL / redo log** | Step 5 — one sequential log + fsync bandwidth for *all* commits |
| **Locks & latches** | Hot rows serialize; lock waits stretch commit time |
| **Buffer pool / dirty pages** | Steps 3–4 + 6 — write spikes fill dirty buffers → checkpoint IO storm |
| **CPU (query + indexes)** | Steps 3–4 — secondary indexes multiply work per statement |
| **Replication fan-out** | Primary must ship WAL to every replica (helps reads, taxes writes) |
| **Connections & transactions** | Bursts hold snapshots/locks longer → vacuum and throughput both suffer |

Reads can be copied. **The write log and the lock domain cannot** — there is one ordered history of mutations for a given piece of data.

### How a traffic spike turns into an outage

A write spike (launch, viral post, flash sale, retry storm) compounds through the write path:

1. **Arrival rate > durable commit rate** — WAL fsync / commit queue grows; latency jumps.
2. **Lock wait amplification** — slow commits hold locks longer → more queueing → congestion collapse.
3. **Client retries** — timeouts retry the same write → *amplified* load on the same WAL/locks.
4. **Checkpoint / I/O storm** — dirty-page surge; even unrelated writes stall.
5. **VACUUM / bloat lag** — MVCC churn outpaces autovacuum; pages get fatter; every write gets more expensive.
6. **Replica lag / cascade** — WAL shipping falls behind; API pools exhaust; shared primary takes unrelated features down.

So the “spike” is a **feedback loop**: queueing → lock contention → retries → more queueing. One node has no escape hatch.

### Why vertical scaling and read replicas are not enough

- Bigger hardware raises the ceiling but you still have **one** WAL, **one** lock domain, **one** failure domain.
- You cannot buy out of **hot-row contention** — two updates to the same row stay serial.
- Peak can be 10–100× baseline; sizing for peak is wasteful and still may fail.
- **Replicas do not scale writes** — every INSERT/UPDATE/DELETE still hits the primary first; more replicas can add WAL fan-out cost.

When write QPS, data size, or contention exceeds one primary — **split the write domain** (shard) and/or **buffer/shed** so the peak never hits the commit path at full force.

---

## The Two Core Approaches

### 1. Sharding (Horizontal Partitioning)

Split the dataset across multiple database servers (shards). Each shard is its own primary with its own buffer pool, WAL, locks, checkpoints, and vacuum — so the expensive write-path costs are no longer global.

```
User ID 1–1M     → Shard A
User ID 1M–2M    → Shard B
User ID 2M–3M    → Shard C
```

**What sharding buys you:**

- **N× write throughput** (ideally) — N independent WAL/lock domains; spikes spread instead of concentrating
- **Smaller working set** — hotter buffers, cheaper checkpoints, healthier vacuum
- **Blast-radius isolation** — one sick shard ≠ whole product down
- **Independent scaling** — grow busy shards only (with a good key)

**What sharding costs:**

- Cross-shard queries / joins / transactions need 2PC, sagas, or app-level merge
- Re-sharding is painful
- Hot keys still melt a *single* shard — same write-path physics, smaller blast radius

#### How routing works

- **Range-based** — `user_id ∈ [1M, 2M)` → Shard B. Simple; uneven ranges recreate hot shards.
- **Hash-based** — `shard = hash(user_id) % N`. Even spread; changing `N` reshuffles almost every key.
- **Directory** — metadata `key → shard`. Flexible moves; directory is critical infra.

#### Consistent hashing

Plain `hash(key) % N` remaps most keys when N changes → massive data movement during scale-out.

**Consistent hashing** puts keys and shards on a ring (often with virtual nodes). Add a shard → only a neighboring arc moves (~`1/N` of data), not the whole keyspace. Virtual nodes keep load from clumping on one physical machine.

**Interview use:** shard by `user_id` with consistent hashing so growth does not reshuffle everything; pair with a migration plan (dual-write, shadow reads, bounded reshard windows).

**When:** one primary cannot sustain write throughput, storage, or spikes — even after vertical scaling and cutting unnecessary indexes/work — and you can choose a key that co-locates related writes.

---

### 2. Partitioning (Logical Separation)

Split by type, feature, or domain before full sharding:

- Separate DBs/tables for **users**, **orders**, **inventory**
- Hot vs cold write paths
- Event partitions by `topic` / `region` (e.g. Kafka)

**Why it helps the write path:** likes, analytics, and payments on one primary share WAL, locks, and checkpoints — an analytics spike can stall payments. Separate them so non-critical write storms cannot take the critical commit path.

---

## Picking a Shard / Partition Key

A bad key creates **hot spots** — one shard becomes the old single node again (same WAL/fsync/lock costs).

### Good key properties

| Property | Why it matters |
| --- | --- |
| High cardinality | More unique values → more even distribution |
| Uniform distribution | No single value dominates writes |
| Co-locates related data | Avoid cross-shard reads for common queries |
| Stable | Key changes imply expensive re-sharding |

### Good vs bad examples

| Domain | Good key | Bad key | Why bad fails |
| --- | --- | --- | --- |
| Social feeds | `user_id` | `created_at` | All “now” writes hit one shard — single-node spike again |
| Ride sharing | `geographic_region` | `driver_id` alone for matching | Regions co-locate nearby trips |
| E-commerce | `order_id` (hashed) | `product_category` | A few categories dominate |
| Messaging | `conversation_id` | `sender_id` | Keeps one conversation on one shard |

### Hot spots

Disproportionate writes on one shard/partition — other shards idle while one replays the full write-path collapse.

Causes: low-cardinality keys, celebrity/viral entities, time-only keys.

#### Mitigations

1. **Key salting** — `user_123_0..9`; writes spread; reads fan-out + merge (good for append-heavy celebrity timelines).
2. **VIP / dedicated shards** — isolate known hot tenants so they don’t crowd out others.
3. **Admission control** — per-key/per-tenant rate limits; `429` one viral key rather than exhaust the shard.
4. **Avoid monotonic shard keys** — use `hash(user_id) + timestamp` (or hash prefix), not bare `created_at`.
5. **Accept per-row serial ceiling** — one inventory row cannot be parallelized by sharding; use per-key queues, atomic ops, or external counters (Redis INCR).

---

## Handling Write Bursts

Even sharded systems get spikes. Use a **write buffer** so the commit path (WAL fsync + locks) never sees the raw burst shape.

### Queues as write buffers

```
Client → API → Message Queue (Kafka / SQS) → Workers → Database
```

- Ack when durable in the queue (define that clearly).
- Workers drain at a rate the DB’s WAL/checkpoints can sustain.
- Partition the queue by the **same shard key** — preserves per-key order; avoids stampeding the wrong shard.

**Trade-off:** DB state lags the ack. Fine for analytics/feeds/notifications; not for money/inventory without a sync path or read-your-writes from the buffer.

### Why queues dampen spikes

Dangerous shape: **tall, narrow burst** into one primary’s fsync/lock path. Queue → **wide, flat drain** at `workers × rate`. Queues smooth load; they do not invent capacity.

### Backlog risk

Sustained ingest > drain → disk/memory pressure, TTL loss, hours to catch up after a short outage, duplicates unless consumers are **idempotent**.

**Mitigations:** lag as a primary SLI; scale workers carefully per shard (don’t recreate a DB spike); API rate limits; idempotent consumers (dedupe/upserts).

### Load shedding

When backlog is dangerous, shed deliberately:

- Reject/defer non-critical writes (`503` + jittered retry — naive retry worsens spikes)
- Priority queues — payments/auth high; likes/views shed first
- Circuit breakers when downstream health dies

A partially degraded system beats a fully crashed one.

---

## Trade-offs Summary

| Approach | Benefit | Cost |
| --- | --- | --- |
| Bigger single primary | Simple; strong consistency | One WAL/lock domain; spike collapse |
| Sharding | Scale write domains; isolate blast radius | Routing; cross-shard work; re-shard pain; hot keys |
| Consistent hashing | Bounded key movement on scale-out | Rebalance/migration still required |
| Domain partitioning | Isolates write-path contention cheaply | One hot partition can still burn |
| Write queues | Absorbs bursts; protects fsync path | Eventual visibility; lag; need idempotency |
| Load shedding | Survive extreme load | Lost/delayed writes; needs priorities |

---

## Interview Signal

1. **Walk the write path briefly** — buffer pool → MVCC → dirty pages → **WAL fsync commit** → checkpoint → vacuum.
2. **Name the costly parts** — fsync latency, index amplification, hot-row locks, checkpoint/vacuum under churn.
3. **Why one node fails** — those costs share one machine; spikes → retries → congestion collapse. Replicas don’t fix writes.
4. **Why shard** — N WAL/lock domains; justify the shard key; consistent hashing for growth.
5. **Under pressure** — partition → shard → queue → shed; call out hot spots and backlog.

## See also

- Part of [[Scaling Reads vs Writes]] · [[Database Replication]] · [[WAL]] · [[System Design MOC]]
