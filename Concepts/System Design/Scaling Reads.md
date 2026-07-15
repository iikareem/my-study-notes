---
tags:
  - caching
  - hot-keys
  - postgresql
  - replication
  - scaling
  - system-design
---

Read traffic is usually the **first bottleneck** you hit. Systems like Instagram see read-to-write ratios of 100:1 — users scroll through hundreds of posts daily but upload only once. The pattern gives you a repeatable progression to handle this.

---

## The Optimization Progression

Work through these layers in order. Each one buys you more headroom before you need the next.

### 1. Database Optimizations

Fix the database itself before adding infrastructure.

- **Indexes** — add indexes on frequently queried columns so the database scans rows efficiently instead of doing full table scans.
- **Query tuning** — rewrite slow queries, avoid N+1 patterns, use EXPLAIN to find what the planner is doing wrong.
- **Denormalization** — store redundant data to avoid expensive JOINs at read time. Trade write complexity for read speed.
- Extended statistics 
- Partioning 

_When to stop here:_ if your read load is moderate and latency is acceptable after tuning, this is enough. Don't add replicas or caches you don't need yet.

### 2. Read Replicas

Spread read traffic across multiple database servers. Writes go to the primary; reads are distributed across replicas.

- One primary handles all writes and replication.
- Multiple replicas serve SELECT queries, reducing load on the primary.
- Adding replicas scales horizontally — you can keep adding them as traffic grows.

_When to use:_ when the database is the bottleneck and query tuning is already done.

### 3. Caching

Store frequently accessed data in memory (e.g. Redis, Memcached) so reads never hit the database at all.

- Cache query results, computed aggregates, or rendered objects.
- Hit rates of 90%+ can dramatically reduce database load.
- Works best when data is read far more often than it changes.

_When to use:_ when some staleness is acceptable and your hot data fits a clear access pattern.

---

## Problems to Know

These are the failure modes interviewers want to hear you anticipate. For each, understand the mechanism, then the mitigations in enough depth to defend a design choice.

### Cache Invalidation

The hardest part of caching. When data changes in the database, stale copies in the cache must be evicted or updated. Getting this wrong means users see old data — or worse, you stampede the database when a popular key expires.

#### Common strategies

- **TTL (time-to-live)** — every cache entry expires after a fixed window (e.g. 30s, 5m, 1h). Simple to implement and self-healing: even if you forget to invalidate, stale data dies eventually. The trade-off is a bounded staleness window — a write can sit invisible until expiry. Use when approximate freshness is fine (leaderboards, recommendation counts, public feed cards). Avoid for money, inventory, or permissions.
- ***** Probabilistic Early Recomputation (XFetch), cache stempede ******

- **Write-through** — on every write, update the database *and* the cache in the same path. Readers always get fresh data from cache. Cost: every write pays cache latency, and failed cache updates leave you inconsistent unless you carefully order the operations (usually write DB first, then cache; on cache failure, either retry or delete the key so the next read reloads).

- **Write-behind / write-back** — write to cache first and flush to the DB asynchronously. Great write throughput, dangerous durability — a crash can lose writes still sitting in memory. Rare for user-facing product data; more common for metrics/session scratchpads.

- **Cache-aside (lazy loading)** — the app reads cache first; on miss, loads from DB and fills the cache. Writes update the DB and *delete* (not update) the cache key, so the next read repopulates. Simple and the most common pattern, but cold starts and simultaneous misses create thundering herds.

#### Mitigations in depth

**1. Prefer delete-on-write over update-on-write (cache-aside)**

When the write path updates the DB, delete the related cache key instead of writing the new value into cache. Why: if two writers race, an "update cache with value A" after "update cache with value B" can leave the cache permanently wrong relative to the DB. Deleting forces the next read to reload from the source of truth. The window of a miss is cheaper than silent corruption.

**2. Soft TTL / early refresh (stale-while-revalidate)**

Store two deadlines: a soft TTL (serve stale, but trigger a background refresh) and a hard TTL (must revalidate). When soft TTL expires, one request refreshes while others keep getting the slightly stale value. This avoids the cliff where millions of keys expire at once and every request hits the DB. CDN and HTTP caches use the same idea (`stale-while-revalidate`).

**3. Jitter TTLs**

If you set every key to expire in exactly 300s, a restart or mass fill causes synchronized expiry. Add random jitter (e.g. TTL = 300 ± 30s) so expirations spread out and the DB sees a smooth miss curve instead of a spike.

**4. Versioned / namespaced keys**

Include a version or generation in the key (`user:42:v3:profile`). On a schema or semantics change, bump the version and old keys rot via TTL — no need for a mass `FLUSH`. For per-entity invalidation, bump an entity version counter and embed it in dependent keys so one write invalidates a whole related set.

**5. Pub/sub or change-data-capture invalidation**

For multi-service caches, have the write service publish "entity X changed" (Redis Pub/Sub, Kafka, DB CDC). Consumers delete local/remote cache entries. This decouples writers from knowing every cache that might hold a copy — critical once you have several services caching the same data.

**6. Guard the miss path (see Hot Keys)**

Invalidation that deletes a hot key is itself a load event. Pair invalidation with singleflight / request coalescing so only one DB load refills the key.

_Interview framing:_ say what consistency you need first, then pick TTL vs write-through vs cache-aside. Name at least delete-on-write, jitter, and stale-while-revalidate as concrete techniques.

---

### Replication Lag

Writes go to the primary, but replicas take time to catch up — typically milliseconds under healthy load, seconds (or worse) under backlog, network blips, or large transactions. If a user writes and immediately reads from a replica, they may not see their own write. That is the classic **read-your-writes** violation.

Replication is usually asynchronous: the primary commits, then streams the WAL / binlog to replicas. Sync replication exists (primary waits for replica ack) but hurts write latency and availability, so most read-scale setups accept async lag and design around it.

#### Mitigations in depth

**1. Read-your-writes routing (primary for recent writers)**

After a successful write, force that user's subsequent reads to the primary for a short window (e.g. 1–5 seconds), or until a replica reports a position ≥ the write's LSN / GTID / binlog coordinate.

How it works in practice:
- On write, record `last_write_at` (or the replication cursor) in the session cookie, Redis, or a sticky token.
- The read router checks: if `now - last_write_at < threshold`, send to primary; else send to a replica.
- Optionally compare replica lag metrics (`Seconds_Behind_Source`, LSN diff) and only send to replicas that are caught up past the user's write.

This preserves the UX users notice most ("I just posted — where is it?") without forcing *all* reads to the primary.

**2. Sticky sessions / session affinity to primary**

Pin a user (or a write-heavy session) to the primary for the whole request lifecycle or login session. Simpler than LSN tracking, but concentrates load: a burst of writers can overload the primary. Use when write volume is modest or the sticky window is short.

**3. Causal / monotonic reads via replica selection**

If the client (or gateway) remembers the last seen replication position, only route to replicas that have applied at least that position. You get monotonic reads without always hitting the primary. Needs replication metadata exposed to the app layer (Postgres LSNs, MySQL GTIDs, etc.).

**4. Synchronous or semi-sync replication for critical paths**

For money transfers, inventory decrements, or authz changes, use sync / semi-sync replication or read from the primary always for those tables. Accept higher write latency on a small, high-correctness subset instead of paying it everywhere.

**5. Quorum / dual-write patterns (when replicas aren't enough)**

Some systems write to a primary and a strongly consistent store (or use consensus DBs) for the fields that must be immediately visible. Heavier architecture — usually overkill unless you already have that store.

**6. Accept eventual consistency where the product allows**

Feeds, view counts, "people also bought," search indexes — users tolerate seconds of lag. Document the SLO ("replicas may lag up to N ms at p99") and monitor lag as a first-class alert. Designing the *product* for eventual consistency is a valid mitigation, not a dodge.

**7. Fail closed on lag spikes**

When lag exceeds a threshold, stop sending traffic to that replica (health check removes it from the pool) rather than serving arbitrarily stale data. Better to overload remaining healthy replicas / primary briefly than to show wrong balances.

_Interview framing:_ explain *why* lag exists (async WAL shipping), give the read-your-writes example, then walk through primary routing for recent writers vs accepting lag for public/aggregate reads.

---

### Hot Keys

When many clients request the same piece of data at once (a viral post, a celebrity profile, a flash-sale SKU), a single cache key or DB shard absorbs disproportionate traffic. Even a healthy Redis cluster can melt on one key: Redis is single-threaded per shard, and one hot key pins one shard's CPU/network while others sit idle.

Hot keys also amplify cache misses: if the hot key expires or is invalidated, thousands of concurrent requests miss together and stampede the database (**thundering herd**).

#### Mitigations in depth

**1. Local / in-process cache (L1) in front of Redis (L2)**

Each app instance keeps a tiny in-memory map (Caffeine, Guava, `sync.Map`, etc.) for the hottest keys with a very short TTL (e.g. 50–500ms). Most duplicate requests within one instance never leave the process. Trade-off: more staleness per instance, and each instance may briefly diverge — fine for public content, bad for per-user private state unless carefully scoped.

**2. Request coalescing / singleflight**

On a cache miss for key K, only **one** goroutine/thread/request is allowed to load K from the DB; others wait on the same future/promise and share the result. Languages often have this built-in (Go `singleflight`, similar patterns in Java/Netty). This turns "10,000 misses → 10,000 DB queries" into "10,000 misses → 1 DB query." Essential after invalidating a hot key.

**3. Cache key sharding (replicate the hot value)**

Instead of one key `post:42`, store `post:42:0` … `post:42:N` with the same payload. Readers pick a shard at random (`hash(request_id) % N` or random). Writes/invalidations must update or delete all N copies (or use a short TTL so they converge). You trade memory and invalidation complexity for spreading load across Redis shards/threads. Use when one key's QPS dominates a node.

**4. Probabilistic early expiry / soft TTL on hot keys**

Same idea as stale-while-revalidate, tuned for hot keys: as expiry approaches, a small fraction of requests refresh early so the key rarely truly expires under load. Avoids the synchronized miss cliff.

**5. CDN / edge cache for publicly shareable content**

If the hot object is the same for everyone (viral image, public article HTML, product page for anonymous users), push it to a CDN. Edge POPs absorb geographic traffic spikes; origin and Redis never see most of the load. Set short TTLs or use purge APIs on publish. Do **not** CDN-cache personalized responses without varying on the right headers — that leaks data across users.

**6. Read replicas + connection pooling won't fix a logical hot key alone**

Replicas help when load is spread across many rows. A single viral row still concentrates on one primary page/replica buffer. Combine replicas with caching + coalescing; don't assume "more replicas" fixes hot-key physics.

**7. Rate limit and degrade gracefully**

For pathological spikes, shed load: serve a slightly stale cached copy, a static fallback, or a "try again" for anonymous traffic while protecting authenticated critical paths. Pair with alerting on per-key QPS so ops sees the viral event.

**8. Pre-warm known hot keys**

For predictable events (product launch, game drop, celebrity AMA), pre-populate cache and CDN before the spike. Invalidation still matters, but you avoid cold-start herds at T0.

_Interview framing:_ define hot key vs thundering herd, then layer L1 cache → singleflight → key sharding → CDN, and say which apply to personalized vs public data.

---

## Trade-offs Summary

|Approach|Benefit|Cost|
|---|---|---|
|Indexes / query tuning|No new infrastructure|Limited ceiling; schema constraints|
|Read replicas|Linear horizontal scale for reads|Replication lag; eventual consistency|
|Caching|Massive latency reduction; DB offload|Cache invalidation complexity; stale data risk|

---

## Interview Signal

The progression — **optimize → replicate → cache** — is the key signal. Don't jump to caching first. Interviewers want to hear that you'd start with the simplest fix and add complexity only when needed.

Naming the problems is junior-level. Explaining mitigations with *mechanism + when to use* is senior-level:

- **Invalidation** — delete-on-write, jittered TTLs, stale-while-revalidate, versioned keys
- **Replication lag** — read-your-writes routing to primary, LSN-aware replica selection, accept lag where product allows
- **Hot keys** — L1 cache, singleflight, key sharding, CDN for public content

## See also

- Part of [[Scaling Reads vs Writes]] · [[Database Replication]] · [[System Design MOC]]
