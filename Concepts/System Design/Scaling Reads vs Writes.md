---
tags:
  - caching
  - postgresql
  - replication
  - scaling
  - sharding
  - system-design
---

# Scaling Reads vs Scaling Writes — A Complete Guide

A practical reference for learning, designing, and refreshing how systems handle traffic. Read it top-to-bottom once; later jump to the section you need.

**What you will walk away with:**
- Why reads and writes scale differently (physics, not slogans)
- The safe progression for each side — what to try first, what to add later
- The failure modes that actually show up in production and interviews
- Prerequisites explained *before* each technique, so nothing feels like jargon

---

## Table of contents

1. [Prerequisites — vocabulary you need](#1-prerequisites--vocabulary-you-need)
2. [The core difference at a glance](#2-the-core-difference-at-a-glance)
3. [Part A — Scaling reads](#3-part-a--scaling-reads)
4. [Part B — Scaling writes](#4-part-b--scaling-writes)
5. [Side-by-side comparison](#5-side-by-side-comparison)
6. [Decision framework](#6-decision-framework)
7. [Failure-mode cheatsheet](#7-failure-mode-cheatsheet)
8. [Interview / refresh checklist](#8-interview--refresh-checklist)

---

## 1. Prerequisites — vocabulary you need

Skip this only if these terms already feel boring. Every later section assumes them.

### 1.1 Primary and replica

| Term | Meaning |
| --- | --- |
| **Primary (leader)** | The database node that accepts writes. Owns the authoritative commit order. |
| **Replica (follower)** | A copy that receives changes from the primary. Usually serves reads. |
| **Async replication** | Primary commits, then ships the log. Fast writes; replicas can lag. |
| **Sync / semi-sync** | Primary waits for at least one replica ack. Safer, slower writes. |

**Prerequisite idea:** copying data helps *reads*. It does not remove work from the write path — every mutation still enters through a primary first (in classic primary–replica setups).

### 1.2 WAL, pages, buffer pool

| Term | Meaning |
| --- | --- |
| **Page** | Fixed-size disk block (Postgres: typically 8KB). Tables and indexes are sequences of pages. |
| **Buffer pool** | In-memory cache of pages. Almost all reads/writes touch RAM first. |
| **Dirty page** | Page modified in RAM but not yet flushed to its final place on disk. |
| **WAL (Write-Ahead Log)** | Sequential log of changes. Durability comes from appending + `fsync` here, not from immediately rewriting table files. |
| **Checkpoint** | Periodic flush of dirty pages to heap/index files so old WAL can be recycled. |
| **fsync** | OS call that forces buffered data onto stable storage. Expensive; usually on the commit critical path. |

**Prerequisite idea:** a “committed” write means “safe in the WAL,” not “already rewritten on the heap files.”

### 1.3 MVCC (Multi-Version Concurrency Control)

Databases like PostgreSQL often **don’t overwrite** a row in place. An update creates a new row version; the old one stays until no transaction needs it.

- `xmin` — transaction that created this version  
- `xmax` — transaction that deleted/updated it  
- Readers see a **snapshot** — which versions are visible to them  

**Why it matters:** readers don’t block writers the way old lock-everything designs did. The cost is **dead versions** (bloat) that **VACUUM** must reclaim later.

### 1.4 Consistency words (use them carefully)

| Term | Plain meaning |
| --- | --- |
| **Strong consistency** | After a write ack, everyone reads the new value (within the system’s guarantees). |
| **Eventual consistency** | Replicas/caches catch up later; brief staleness is allowed. |
| **Read-your-writes** | The user who just wrote must see their own write on the next read. |
| **Monotonic reads** | You never go backward in time across successive reads. |

**Prerequisite idea:** scaling almost always means choosing *where* you allow staleness — not pretending staleness doesn’t exist.

### 1.5 Cache, hit, miss, invalidation

| Term | Meaning |
| --- | --- |
| **Cache** | Fast store (Redis, Memcached, CDN, in-process) holding a copy of data. |
| **Hit / miss** | Found in cache vs must load from source of truth. |
| **TTL** | Time-to-live — entry expires after a window. |
| **Invalidation** | Deleting or updating cache when the source changes. |
| **Thundering herd** | Many clients miss the same key at once and stampede the DB. |
| **Singleflight / coalescing** | Only one request loads a missing key; others wait and share the result. |

### 1.6 Shard, partition, shard key

| Term | Meaning |
| --- | --- |
| **Partition (logical)** | Split by domain/type (users DB vs orders DB; hot vs cold). |
| **Shard (horizontal)** | Split *the same kind of data* across many databases by key. |
| **Shard key** | The field used to decide which shard owns a row (`user_id`, `order_id`, …). |
| **Hot spot** | One shard/key gets disproportionate traffic — becomes a mini single-node bottleneck. |
| **Consistent hashing** | Ring-based placement so adding a shard moves ~`1/N` of keys, not almost all. |

### 1.7 Vertical vs horizontal scaling

- **Vertical** — bigger machine (more CPU/RAM/disk). Simple ceiling; one failure domain.  
- **Horizontal** — more machines (replicas, shards, cache nodes). More moving parts; better escape hatch under growth.

---

## 2. The core difference at a glance

Most products are **read-heavy** (social apps often ~100:1 read:write). You usually hit a **read** bottleneck first. **Write** bottlenecks hurt more when they arrive — because writes are harder to parallelize.

```
READS                                      WRITES
─────                                      ──────
Can be copied freely                       Must land in the right place once
Staleness often OK                         Durability + correctness required
Scale by: tune → replicas → cache          Scale by: tune → partition → shard
                                           (+ queue / shed under bursts)

Failure themes:                            Failure themes:
  cache invalidation                         single WAL / lock domain
  replication lag                            hot rows + retry storms
  hot keys / thundering herd                 hot shards + backlog
```

**One sentence to remember:**

> Reads scale by *copying*. Writes scale by *splitting ownership* (and smoothing bursts) — because there is one ordered history of mutations per piece of data.

---

## 3. Part A — Scaling reads

### Prerequisites for this part

You should be comfortable with: primary/replica, async replication, cache hit/miss, TTL, and “eventual consistency is a product choice.”

### 3.1 The read progression (do this in order)

Work these layers in order. Each buys headroom before you need the next. **Do not jump to Redis first.**

#### Step 1 — Database optimizations

Fix the database before adding infrastructure.

| Technique | What it does | Prerequisite |
| --- | --- | --- |
| **Indexes** | Turn full scans into seeks on hot filters/joins | Know your real query patterns (`EXPLAIN` / slow query log) |
| **Query tuning** | Kill N+1s, bad joins, select-star waste | Ability to read query plans |
| **Extended statistics** | Help the planner when columns are correlated | Planner misestimates despite basic indexes |
| **Denormalization** | Store redundant data to avoid heavy joins at read time | Accept harder writes / sync rules |
| **Table partitioning** | Split large tables (often by time) for pruneable scans | Clear partition key; ops for attach/detach |

_Stop here if_ latency and CPU are fine after tuning. Extra replicas and caches are operational cost — don’t buy them early.

#### Step 2 — Read replicas

Writes → primary. SELECTs → replicas.

- Adds horizontal read capacity  
- Primary still does all writes + ships WAL/binlog  
- Use when the DB is CPU/IO bound on reads *after* query work  

**Prerequisite:** your app (or proxy) can route reads vs writes; you accept **replication lag**.

#### Step 3 — Caching

Put hot results in memory (Redis/Memcached), or at the edge (CDN) for public content.

- Best when read:write is high and some staleness is OK  
- Cache rows, query results, computed aggregates, or rendered objects  
- 90%+ hit rate can transform DB load  

**Prerequisite:** you have an invalidation story (even if it’s “TTL only”). A cache without a freshness policy is a time bomb.

---

### 3.2 Read-side problems (and mitigations that matter)

#### Cache invalidation

When the DB changes, cached copies must expire or update. Wrong handling → stale UX or a miss stampede.

**Strategies (pick from consistency need):**

| Strategy | Idea | Use when | Cost |
| --- | --- | --- | --- |
| **TTL** | Expire after a window | Approximate freshness OK | Bounded staleness until expiry |
| **Cache-aside** | Read: miss→DB→fill. Write: DB then **delete** key | Default for most apps | Miss storms if unguarded |
| **Write-through** | Update DB + cache on write | Need fresher cache, can pay write latency | Careful failure ordering |
| **Write-behind** | Cache first, DB async | Metrics/scratchpads | Durability risk — rare for core product data |

**Mitigations worth memorizing:**

1. **Delete-on-write (not update-on-write)** in cache-aside — races can’t leave a permanently wrong value; next read reloads truth.  
2. **Soft TTL / stale-while-revalidate** — serve slightly stale while one refresh runs; avoids expiry cliffs.  
3. **Jittered TTLs** — `300s ± 30s` so keys don’t expire in lockstep.  
4. **Probabilistic early recomputation (XFetch)** — near expiry, a fraction of requests refresh early; reduces synchronized misses under load (cousin of soft TTL).  
5. **Versioned keys** — `user:42:v3:profile`; bump version instead of mass flush.  
6. **Pub/sub or CDC invalidation** — writers publish “entity X changed”; many caches delete without hard-coding every consumer.  
7. **Singleflight on miss** — invalidating a hot key is a load event; coalesce refills.

#### Replication lag

Async replicas trail the primary. Classic bug: user writes, immediately reads a replica, doesn’t see their write (**read-your-writes** violation).

**Mitigations worth memorizing:**

1. **Primary routing for recent writers** — sticky window or “read primary until replica LSN ≥ write LSN.”  
2. **Sticky session to primary** — simple; can overload primary if overused.  
3. **Monotonic reads** — only use replicas at/above last seen position.  
4. **Sync path for critical data** — money, inventory, authz: primary reads or sync rep.  
5. **Accept lag in the product** — feeds, counts, recommendations; monitor lag SLO.  
6. **Fail closed on bad lag** — remove far-behind replicas from the pool.

#### Hot keys (and thundering herds)

One viral key hammers one Redis shard/thread or one DB row. Expiry/invalidation of that key → mass miss → DB stampede.

**Mitigations worth memorizing:**

1. **L1 in-process cache** (tens–hundreds of ms TTL) in front of Redis.  
2. **Singleflight** — 10k misses → 1 DB load.  
3. **Key sharding** — `post:42:0..N` copies; randomize reads; invalidate all (or short TTL).  
4. **CDN for public identical content** — never CDN personalized responses without correct `Vary`.  
5. **Pre-warm** predictable events; **rate-limit / degrade** pathological spikes.  
6. Reminder: **more replicas alone don’t fix one hot row.**

---

### 3.3 Read trade-offs (quick table)

| Approach | Benefit | Cost |
| --- | --- | --- |
| Indexes / tuning / stats | No new infra | Ceiling; write cost of indexes |
| Read replicas | Horizontal read scale | Lag; routing complexity |
| Caching | Huge latency + DB offload | Invalidation; staleness; herds |

**Read interview signal:** say **optimize → replicate → cache**, then name invalidation, lag, and hot keys *with mitigations*.

---

## 4. Part B — Scaling writes

### Prerequisites for this part

You should be comfortable with: buffer pool, WAL + fsync, dirty pages, checkpoint, MVCC + vacuum, and why replicas don’t remove write work.

### 4.1 Why writes are hard

- Every write must hit the **correct** owner node  
- Duplicating a write ≠ scaling; it creates conflict  
- Concurrent writers on one row need locks or conflict rules  
- An ack must mean **durable** (survive crash)

### 4.2 The full write path (PostgreSQL mental model)

Scaling writes is guessing until you know what one write *does*.

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
[5] Append WAL + fsync → client gets "committed"   ← durability here
     │
     ▼
[6] CHECKPOINT (periodic) — flush dirty pages to disk
     │
     ▼
[7] VACUUM (background) — reclaim dead row versions
```

| Step | What happens | Takeaway |
| --- | --- | --- |
| 1 Buffer pool | Touch pages in RAM; fault from disk if needed | Commit path avoids random heap writes |
| 2 MVCC | New versions, not in-place overwrite | Reads don’t block writes; bloat is the tax |
| 3 Data page dirty | New tuple in RAM | Old + new coexist until vacuum |
| 4 Indexes dirty | Secondary indexes amplify work | HOT helps only if indexed cols unchanged |
| **5 WAL + fsync** | Sequential log + force to disk | **This is the durability / latency ceiling** |
| 6 Checkpoint | Flush dirty pages later | IO waves under heavy write load |
| 7 Vacuum | Reclaim dead versions | Falling behind makes every write cost more |

### 4.3 Cost of a write (focus here)

| Mechanism | Protects against | Cost |
| --- | --- | --- |
| **WAL fsync on commit** | Crash loss | **Latency every commit** — durable WPS ceiling |
| **Index updates** | Fast lookups | **Write amplification** |
| **Locks / hot rows** | Correct concurrency | **Serialization** — spikes collapse throughput |
| **MVCC** | Reader/writer blocking | **Bloat → vacuum** CPU/IO |
| Buffer pool deferral | Random IO on commit | Volatility; depends on WAL |
| Checkpoint spreading | One giant stall | Recovery window; still IO waves |
| HOT | Index amplification | Only for non-indexed column updates |

**Mental model:**  
`commit latency ≈ fsync + lock waits`  
`sustained throughput ≈ dirty pages + WAL + checkpoint + vacuum keeping up`

### 4.4 Why a single write node spikes and dies

One primary means **one** WAL, **one** lock domain, **one** checkpoint/vacuum regime.

**Spike feedback loop:**

1. Arrival rate > durable commit rate → queues grow  
2. Lock waits stretch → congestion collapse  
3. Client timeouts **retry** → amplified write load  
4. Dirty-page surge → checkpoint IO storm  
5. Vacuum lags → bloat → every write more expensive  
6. WAL shipping / pools fail → cascade across features  

**Vertical scaling** raises the ceiling; it does not remove hot-row serialization or the single failure domain.  
**Read replicas do not scale writes** — they can add replication fan-out cost on the primary.

When one primary can’t absorb write QPS, data size, or spikes → **split write ownership** and/or **buffer/shed**.

### 4.5 Write progression

#### 1) Partition by domain (often first)

Separate users / orders / inventory; hot vs cold; critical vs analytics.  
Stops a likes-storm from sharing WAL/locks with payments.

#### 2) Shard horizontally

Each shard = independent primary (own WAL, locks, buffers, vacuum).

```
user_id 1–1M     → Shard A
user_id 1M–2M    → Shard B
user_id 2M–3M    → Shard C
```

**Buys:** ~N× write domains, smaller working sets, blast-radius isolation.  
**Costs:** cross-shard queries/transactions; painful re-shard; hot keys still melt one shard.

**Routing:**

- Range — simple; uneven growth → hot shard  
- `hash % N` — even; changing N reshuffles almost everything  
- Directory — flexible moves; metadata is critical path  
- **Consistent hashing** — add capacity moving ~`1/N` of keys; use virtual nodes for balance  

#### 3) Pick a shard key carefully

| Need | Why |
| --- | --- |
| High cardinality | Enough distinct values to spread |
| Uniform access | Avoid one value owning the world |
| Co-locate related data | Common queries shouldn’t fan out |
| Stable | Key changes imply migrations |

| Domain | Good | Bad | Why bad fails |
| --- | --- | --- | --- |
| Social | `user_id` | `created_at` | All “now” writes → one shard |
| Rides | geo region | naive single id for matching | Related trips scatter / hotspots |
| Commerce | hashed `order_id` | `category` | Few categories dominate |
| Chat | `conversation_id` | `sender_id` | Conversation splits across shards |

**Hot-spot mitigations:** salting (`user_123_0..9`), VIP shards, per-key rate limits, avoid time-only keys, accept that **one row stays serial** (queues / atomic counters).

#### 4) Buffer bursts with queues

```
Client → API → Queue (Kafka/SQS) → Workers → DB
```

Turns a tall spike into a flat drain. Partition the queue by the **same shard key**.  
Queues **smooth**; they don’t create capacity. Watch **consumer lag**; make consumers **idempotent**.

#### 5) Shed load deliberately

Priority queues, `503` + jittered retry, circuit breakers. Drop likes before payments. Partial degradation > total outage.

---

### 4.6 Write trade-offs (quick table)

| Approach | Benefit | Cost |
| --- | --- | --- |
| Bigger primary | Simple | One WAL/lock domain |
| Domain partition | Cheap isolation | One hot partition can still burn |
| Sharding | Scale write domains | Routing; cross-shard; re-shard |
| Consistent hashing | Bounded move on growth | Still need migration discipline |
| Queues | Protect commit path | Lag; idempotency; backlog risk |
| Load shedding | Survive extremes | Lost/delayed writes |

**Write interview signal:** walk **buffer → MVCC → dirty pages → WAL fsync → checkpoint → vacuum**, name the costly lines, explain single-node spike loops, then **partition → shard (key + consistent hashing) → queue → shed**.

---

## 5. Side-by-side comparison

| Dimension | Scaling reads | Scaling writes |
| --- | --- | --- |
| First instinct that works | Copy (replicas, cache) | Split ownership (partition/shard) |
| Safe order | Tune → replicas → cache | Tune → partition → shard → queue/shed |
| What you’re multiplying | Readers of the same data | Independent commit domains |
| Consistency tax | Lag + stale cache | Cross-shard transactions; eventual queues |
| Signature failure | Invalidation, lag, hot key herd | WAL/lock saturation, hot shard, retry storm |
| Replicas help? | Yes (SELECTs) | No (writes still on primary) |
| Cache helps? | Yes (offload reads) | Indirectly (fewer read-driven locks); doesn’t remove durable write cost |
| Hardest design choice | Freshness vs hit rate | Shard key + what may be async |

### How they interact (don’t design them in isolation)

- Caching reads **reduces** read load on primaries/replicas — good — but **invalidation is a write-adjacent event** (delete hot keys carefully + singleflight).  
- More replicas help reads but **add WAL shipping work** on the write primary.  
- Sharding helps writes **and** can help reads (smaller indexes) — but every read path must know the shard key or pay scatter-gather.  
- Queues protect write durability paths; your **read-your-writes** story must include “not in DB yet.”

---

## 6. Decision framework

Use this when designing or debugging.

```
Is the pain READ latency / READ CPU?
  ├─ Profile queries / EXPLAIN first
  ├─ Indexes, rewrite, stats, denorm/partition
  ├─ Still hot? → read replicas + lag strategy
  └─ Hot keys / repeated reads? → cache + invalidation + singleflight
      └─ Public identical content? → CDN

Is the pain WRITE latency / commit queue / lock waits?
  ├─ Cut unnecessary indexes/triggers; shorten transactions
  ├─ Separate critical vs best-effort write domains
  ├─ Single primary saturated? → shard with a real key
  ├─ Spiky ingest? → queue + idempotent consumers
  └─ Still melting? → shed non-critical; protect money/auth paths

Is it BOTH?
  └─ Usually: fix write ownership / spikes first if commits are failing,
     then scale reads — a wedged primary breaks everything.
```

### Product consistency cheat sheet

| Data | Typical stance |
| --- | --- |
| Balances, inventory, permissions | Strong / primary reads; minimal cache TTL or write-through |
| Profile, posts after edit | Read-your-writes for author; eventual for others |
| Feeds, counts, recommendations | Eventual; TTL cache fine |
| Analytics events | Queue + shed OK |

---

## 7. Failure-mode cheatsheet

| Symptom | Likely cause | First moves |
| --- | --- | --- |
| Reads slow, writes fine | Missing index, bad query, under-replicated reads | `EXPLAIN`, tune, add replicas |
| User doesn’t see own write | Read from lagging replica | Primary sticky window / LSN routing |
| Random stale data | Cache invalidation gap | Delete-on-write, TTL jitter, version keys |
| Sudden DB storm | Hot key expiry / invalidate | Singleflight, soft TTL, L1 cache |
| Commit latency climbs | WAL/fsync or lock contention | Find hot rows; shorten txns; consider shard/queue |
| One shard on fire | Bad key or celebrity entity | Salt, VIP shard, rate limit |
| Queue depth explodes | Drain < ingest | Scale workers carefully; rate limit; shed |
| Everything melts after timeout wave | Retry amplification | Jittered retry, idempotency, load shed |

---

## 8. Interview / refresh checklist

**Always**

- [ ] State read:write ratio and which side hurts  
- [ ] Give a progression (don’t lead with the fanciest tool)  
- [ ] Name one consistency relaxation you’re accepting  

**Reads**

- [ ] optimize → replicate → cache  
- [ ] Invalidation: delete-on-write, jitter, soft TTL / XFetch  
- [ ] Lag: read-your-writes routing  
- [ ] Hot keys: L1 + singleflight (+ CDN if public)  

**Writes**

- [ ] Write path: buffer → MVCC → WAL **fsync** → checkpoint → vacuum  
- [ ] Cost drivers: fsync, indexes, locks, bloat  
- [ ] Why one node spikes (feedback loop + retries)  
- [ ] partition → shard (justify key, consistent hashing) → queue → shed  

**Senior tell**

- [ ] Mitigations with *mechanism + when*  
- [ ] How read scaling and write scaling interact  
- [ ] What you monitor: replica lag, cache hit rate, commit latency, lock waits, consumer lag, per-key QPS  

---

## Quick reference card

| | Reads | Writes |
| --- | --- | --- |
| **Keyword** | Copy | Split |
| **Order** | Tune → replica → cache | Tune → partition → shard → queue |
| **Bottleneck physics** | CPU/IO on SELECTs; cache misses | WAL fsync + locks + amplification |
| **Watchouts** | Stale, lag, herds | Hot shard, retries, backlog |

---

*Companion deep-dives in this vault: [[Scaling Reads]] · [[Scaling Writes]]*

## See also

- [[Scaling Reads]] · [[Scaling Writes]] — focused companions
- [[Database Replication]] · [[Scaling PostgreSQL to 800 Million Users]]
- [[System Design MOC]]
