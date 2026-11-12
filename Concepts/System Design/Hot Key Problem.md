---
tags: [interview, system-design, redis, caching]
aliases: [Hot key, Celebrity key problem]
---

# Hot Key Problem (Redis / Distributed Cache)

> [!summary] One-liner
> One key gets a huge share of traffic, so the single shard that owns it becomes the bottleneck while other shards sit idle. The right fix depends on **read vs write** and **how often the data changes**.

## Building blocks

- **Cache**: fast in-memory store (Redis) in front of a slow DB. Data lives under a **key**, e.g. `post:123`.
- **Single node limit**: ~100k requests/sec, then it queues and slows.
- **Sharding**: keys are split across nodes by hashing. Each key lives on exactly **one** shard.
- **Replica**: copy of a node. Serves reads; writes go to the primary.

## The problem

Sharding assumes traffic is spread evenly across keys. A celebrity tweet breaks that: millions hit `post:123`, which lives on **one** shard. Adding nodes doesn't help, because the hot key still maps to one node.

**Detect it from metrics**: one node's CPU or requests/sec far above the others.

## Two questions that decide the fix

1. Is the hot key mostly **read** or **written**?
2. How often does the data **change**?

## The 4 fixes

### 1. Read replicas

- Hot shard = 1 primary (writes) + N replicas (reads). Reads are spread across replicas.
- **Use when**: read-heavy, rarely updated (tweet content, product page). Covers most real cases.
- **Bonus**: a CDN does the same closer to users, with lower latency.
- **Who routes reads?** Redis does not do it automatically. Use a **client library** configured for replica reads (random / round-robin) or a **proxy / load balancer**. If all reads still hit the primary, replicas do nothing.
- **Catch: replication lag.** A replica can briefly return the old value after a write. Fine for most hot-key cases; if you need the latest value, read from the primary.
- **Catch: memory.** Replicas copy the _whole_ shard, not just the hot key.

### 2. Key copies with suffixes

- Store the same value under `post:123:0`, `post:123:1`, `post:123:2`. Clients pick a suffix at random.
- Different keys hash to **different shards**, so load spreads across separate nodes.
- **Use when**: data is basically static (cached HTML fragment, config value).
- **Catch**: on update you must write **every** copy, and they can briefly disagree.

> [!warning] Replicas vs suffixes
> - **Read replicas** = copies _inside one shard's group_.
> - **Suffix copies** = copies spread _across different shards_.
> - Suffixes are for **reads** of rarely-changing data, not for write-heavy data.

### 3. Write batching

- For hot **writes** (view/like counters). Each app server counts locally and flushes in bulk every ~100ms: "add 4,300 views" instead of 4,300 `INCR`s.
- Kafka can absorb the burst upstream before it reaches the cache.
- **Trade-off**: counts lag ~100ms, and buffered data is lost if the server crashes before flushing.
- **Use when**: write-heavy counters where slight staleness is fine.

### 4. Shard splitting

- The key space is cut into ranges, one per shard. If a shard is overloaded because **many popular keys** share its range, split the range and move half to a **new node**.

```
Before:  Shard B owns 34%-66%
After:   Shard B1 owns 34%-50%
         Shard B2 owns 51%-66%  (new node)
```

- **Why hard**: live data migration, routing updates, avoiding lost or wrong data during the move.
- **Why last resort**: it divides a _range of keys_, not a single key. One very hot key still lives on one node after the split.
- **Use when**: the node itself is the bottleneck at extreme scale.

## Decision table

| Situation | Reach for |
| --- | --- |
| Read-heavy, rarely updated | Read replicas (+ CDN) |
| Read-heavy, truly static | Key copies with suffixes |
| Write-heavy counters | Write batching |
| Node itself is the bottleneck, extreme scale | Shard splitting |

## How to answer in an interview

1. Name the problem: "one key is overloading one shard."
2. Say you'd confirm it from metrics.
3. Ask about read/write ratio and how often the data changes.
4. Start simple (replicas or key copies), add batching if writes are the issue.
5. Mention shard splitting only if pushed toward extreme scale. Don't jump straight to it.

> [!quote] Sample answer
> "A hot key means one key gets a disproportionate share of traffic, so the shard holding it becomes a bottleneck while others sit idle. I'd confirm it from metrics, then ask whether it's read-heavy or write-heavy and how often the data changes. If read-heavy, I'd add read replicas to the hot shard: the primary takes writes and replicas share reads, with the client or a proxy routing reads. If the data is basically static, I could also store suffixed copies like `post:123:0`, `post:123:1` so they land on different shards. If write-heavy, like a view counter, I'd batch writes in the app and flush to Redis every ~100ms, possibly with Kafka absorbing the burst. Only if a single node is still the bottleneck at extreme scale would I split the shard."

## Self-test

- [ ] Why doesn't adding more shards fix a hot key?
- [ ] Replicas vs suffix copies: what's the difference?
- [ ] Who routes reads to replicas?
- [ ] What's the downside of write batching?
- [ ] Why doesn't shard splitting fix a _single_ hot key?
