# Latency and Throughput: The Two Numbers That Run System Design

[[Latency and throughout review]]

Every performance conversation in system design eventually collapses into these two words. They sound related — and they are — but they measure **different things**, get optimized by **different techniques**, and very often **trade off against each other**. Confusing them is one of the fastest ways to "fix" a system and make it worse.

This is a from-scratch walkthrough: what each metric actually means, why they aren't the same axis, and the concrete levers you pull to move each one.

---

## Table of Contents

1. [The Definitions](#1-the-definitions)
    - [Latency](#11-latency)
    - [Throughput](#12-throughput)
2. [Why They Aren't the Same Axis](#2-why-they-arent-the-same-axis)
3. [Percentiles: Why "Average Latency" Lies to You](#3-percentiles-why-average-latency-lies-to-you)
4. [Optimizing Latency](#4-optimizing-latency)
5. [Optimizing Throughput](#5-optimizing-throughput)
6. [The Tradeoff, Made Explicit](#6-the-tradeoff-made-explicit)
7. [Summary](#summary)

---

## 1. The Definitions

### 1.1 Latency

**Latency** is the elapsed time to complete a single unit of work, end to end — one request, one query, one message. It is a property of _o**ne operation's journey through the system_**, measured in units of time (ms, μs, s).

> **Latency** = time from request sent → response received, for **one** operation.

Latency is never really "one number" internally — it's the **sum of every stage** that operation passes through:

```
Total latency = network transit time   (client → server)
              + queuing/wait time      (waiting for a free thread, connection, or CPU slot)
              + processing time        (CPU work: parsing, business logic, serialization)
              + I/O wait time          (disk read, DB round trip, downstream service call)
              + network transit time   (server → client)
```

This decomposition matters because **"latency is high" is not a diagnosis — it's a symptom.** The fix is completely different depending on which term in that sum is dominant:

|Dominant term|Likely cause|Wrong instinct|Right fix|
|---|---|---|---|
|Queuing / wait time|Under-provisioned relative to load|Add a cache|Add workers / scale out|
|I/O wait time|Slow downstream dependency (DB, cache, service)|Optimize business logic|Fix or cache the slow dependency|
|Processing time|Too much CPU work per request|Add more servers|Optimize the code itself|

Measuring latency without breaking it into these components is why "just add more servers" so often fails to fix a latency problem that was actually a downstream-database problem.

---

### 1.2 Throughput

**Throughput** is the rate at which the _system as a whole_ completes units of work — not one operation's speed, but how many operations the system disposes of per unit of time. It's measured in **units-per-time**: requests/sec (RPS), transactions/sec (TPS), messages/sec, MB/sec.

> **Throughput** = number of operations completed / unit of time.

Throughput isn't a property of a single request — it's an **emergent property** of how much work the system can perform _concurrently_, given its resources. The clearest way to build intuition for it is to derive it step by step, from a single worker up to a full system.

#### Step 1: One worker, no concurrency

Say you have one worker (one thread, one process, one connection) and every request takes exactly **W = 100ms** from start to finish. Because this worker can only do one thing at a time, it must fully finish request 1 before starting request 2.

In 1000ms (1 second), how many 100ms requests can it finish?

```
requests per second = 1000ms / 100ms per request = 10 req/sec
```

That's the entire throughput of a single, non-concurrent worker: **10 req/sec**. This number came directly from `1 / W`. With only one worker, throughput is just the inverse of latency — this is the special case that makes people think latency and throughput are "the same thing." They only look the same when concurrency = 1.

#### Step 2: Add concurrency

Now put **C = 10** of these workers side by side, each still taking the same **W = 100ms** per request. They're not making any single request faster. What's changed is that 10 requests can now be _in flight at the same time_.

In that same 1000ms window:

```
10 workers × 10 req/sec each = 100 req/sec total
```

The system's throughput went from 10 → 100 req/sec — **a 10× jump** — purely by adding concurrency. Latency per request is still 100ms, completely unchanged.

> **Key insight:** You just multiplied throughput by 10 without making anything faster. You made more things happen _at the same time_.

#### Step 3: The general formula

Generalizing: if you have `C` workers each handling one request in `W` seconds, all running in parallel:

```
throughput = C / W
```

Where:

- `C` = **concurrency** — how many requests the system is actively working on at once (thread pool size, connection pool size, worker count, etc.)
- `W` = **latency** — how long one request takes, start to finish
- `throughput` = requests completed per second, system-wide

Plug in step 2's numbers to verify: `C / W = 10 / 0.1s = 100 req/sec` ✓

This formula reveals the **two — and only two — levers** that move throughput:

1. **Raise `C` (add concurrency)** — more workers, threads, connections, or server instances. This is the primary throughput lever and why "scale horizontally" is the default answer to a throughput problem.
2. **Lower `W` (reduce latency)** — make each request finish faster so each worker frees up sooner. Latency improvements _do_ help throughput, but indirectly, through this formula — not as the main lever.

**Why throughput has a ceiling:** `C` can't grow forever. Each additional worker eventually competes for a finite underlying resource — CPU cores, DB connections, disk I/O bandwidth. The moment that resource is fully busy, adding another worker doesn't add real parallel capacity. It just adds another worker waiting in line for the same CPU core or disk head, which makes `W` go _up_ (more contention, more context switching) instead of adding throughput. That's the **saturation point** — the knee in the curve where more concurrency stops paying off.

---

> **To summarize:**
> 
> - **Latency (`W`)** is about one request's time.
> - **Throughput** is about how many requests are simultaneously "in progress" (`C`) divided by how long each one takes (`W`). It is a property of the whole system's parallel capacity, not of any individual request's speed.
> 
> You can have **low latency with low throughput** (a system that responds in 2ms, but only has one worker fed one request every few seconds). You can have **high throughput with high latency** (a system doing 50,000 req/sec, but each individual request sits in queue for 800ms). These are genuinely independent axes.

---

## 2. Why They Aren't the Same Axis

The core reason latency and throughput diverge is **concurrency and queuing**.

A single-threaded, single-request system has a simple relationship: `throughput ≈ 1 / latency`. If one request takes 10ms, you can serve at most 100 req/sec, serially.

But almost no real system works that way. Real systems handle many requests concurrently — via threads, event loops, connection pools, or worker processes. Once you introduce concurrency, throughput stops being a simple inverse of latency and instead becomes a function of:

- how much work can happen in parallel
- how much of that work **contends** for the same resource (CPU core, DB connection, lock, disk head)
- how the system behaves once demand exceeds capacity (queuing)

This is where **Little's Law** becomes the load-bearing formula of the whole topic:

```
L = λ × W
```

|Variable|Meaning|
|---|---|
|`L`|Average number of requests in the system (in-flight)|
|`λ` (lambda)|Arrival rate / throughput|
|`W`|Average time a request spends in the system (latency)|

Rearranged: **latency = concurrency / throughput.**

This single equation explains almost every throughput/latency tradeoff you'll encounter. If you hold concurrency fixed and push more throughput through the system, latency _must_ rise — requests start queuing behind each other. The only way to raise throughput without raising latency is to raise concurrency, up to the point where you saturate the underlying resource.

### The Three Regions of a System Under Load

As you increase load, a system moves through three distinct regions:

```
Throughput
    │                          ●●● (plateau / collapse)
    │                     ●●●
    │               ●●●
    │         ●●●      ← knee (saturation point)
    │   ●●●
    │●●●
    └────────────────────────────────────► Request Rate
         ↑              ↑            ↑
     Linear          Approaching   Past
     scaling         saturation    saturation
```

1. **Low load** — throughput scales linearly with request rate, latency stays flat. Plenty of spare capacity.
2. **Approaching saturation** — latency starts climbing as queues form, throughput growth slows.
3. **Past saturation** — throughput plateaus or _drops_ (context-switching overhead, retries, timeouts, cascading failures), while latency spikes. Sometimes called a **latency cliff** or "the hockey stick."

Every capacity-planning exercise is really about finding where you sit on that curve and building enough headroom that traffic spikes don't push you past the knee.

---

## 3. Percentiles: Why "Average Latency" Lies to You

A single average latency number hides the shape of the distribution — and the **shape is what actually matters** for user experience.

| Percentile       | What it tells you                                |
| ---------------- | ------------------------------------------------ |
| **p50 (median)** | Half of requests are faster than this            |
| **p95**          | The slowest 5% of requests                       |
| **p99**          | The slowest 1% of requests                       |
| **p99.9**        | The "tail" — rare, but at scale, not rare at all |

### The Tail Latency Math You Can't Ignore

Here's the part that trips people up: at high request volume, "rare" tail latency stops being rare in absolute terms.

> If you serve **1,000,000 requests/day** and your p99.9 is 2 seconds, that's **1,000 requests a day** taking 2+ seconds — every single day.

It gets worse. If a single user page load fans out into **20 backend calls**, the probability that _at least one_ of those 20 hits the tail is much higher than 0.1%. This compounding effect is called **tail latency amplification**, and it's why large-scale systems obsess over p99 and p99.9 rather than the average.

> **Practical implication:** Optimize the tail, not the mean. Averages are dragged down by the fast majority and can look great while a meaningful slice of real users have a bad time.

---

## 4. Optimizing Latency

Latency optimization is about **shrinking the critical path** of a single request — removing steps, doing steps in parallel instead of sequentially, or moving the work closer to where it's needed.

### Reduce the Number of Network Hops

Every hop adds round-trip time. Chatty service-to-service calls (A calls B calls C, sequentially) stack latencies additively. Batching, GraphQL-style field selection, or a **BFF layer** that aggregates multiple backend calls into one client-facing response are common fixes.

### Parallelize Independent Work

If a request needs data from three unrelated services, calling them concurrently (`Promise.all` in Node, fan-out/fan-in pattern) means total latency is bounded by the **slowest** call, not the **sum** of all three.

```
Sequential:   [A: 50ms] → [B: 80ms] → [C: 40ms] = 170ms total
Parallel:     [A: 50ms]
              [B: 80ms]  ← all in flight at once
              [C: 40ms]
                                                  = 80ms total
```

### Cache Aggressively at the Right Layer

CDN edge caching for static/semi-static content, in-memory caches (Redis, Memcached) for hot data, and application-level memoization all **remove round trips to slower systems entirely**. The tradeoff is staleness — cache invalidation strategy matters as much as the caching itself.

### Move Data Physically Closer to the Request

Geo-distributed deployments, edge computing, and read replicas in the requester's region cut the irreducible **speed-of-light cost** of long-haul network trips.

### Reduce Serialization/Deserialization Overhead

Protobuf/gRPC over JSON/REST for internal service calls, avoiding unnecessary data transformation layers, and trimming oversized payloads all shave real time off the critical path.

### Use Connection Reuse and Keep-Alives

TCP/TLS handshakes are expensive relative to the request itself. HTTP keep-alive, connection pooling, and HTTP/2 multiplexing **amortize that cost** across many requests instead of paying it every time.

### Pick the Right Consistency Model

Reads that don't need to observe the very latest write can be served from a nearby replica instead of waiting on a round trip to a primary in another region. This is a direct **latency-for-consistency** tradeoff — worth making explicitly rather than defaulting to strong consistency everywhere.

---

## 5. Optimizing Throughput

Throughput optimization is about maximizing the _rate_ of completed work — which usually means **increasing usable concurrency** and removing bottlenecks that force serialization.

### Scale Horizontally

Add more instances behind a load balancer. This directly increases the number of requests that can be processed in parallel — but only as far as the shared resources behind those instances (database, downstream service) can also scale.

### Use Asynchronous, Non-Blocking I/O

A thread blocked waiting on a DB call or disk read is a thread not doing other work. Event-loop architectures (Node.js), async/await patterns, and non-blocking I/O (epoll/io_uring underneath) let a single process handle **far more concurrent in-flight requests** than a thread-per-request blocking model.

### Batch Operations

Grouping many small writes into one bulk database insert, or many small Kafka messages into one batch, **amortizes fixed per-operation overhead** (network round trip, transaction commit cost) across many units of work — dramatically raising throughput at the cost of a small amount of added latency per item (you wait for the batch to fill or flush).

### Use Message Queues to Decouple Producers from Consumers

A queue (Kafka, SQS, RabbitMQ) lets producers keep emitting work at their own rate while consumers process at a sustainable rate, **smoothing out traffic spikes** instead of letting them cascade into timeouts. This is a throughput-_stabilizing_ pattern more than a raw-throughput-increasing one — it protects throughput under bursty load.

### Shard/Partition the Data Layer

A single database instance has a throughput ceiling. Horizontal sharding (hash or range-based) spreads write and read load across multiple database nodes, **multiplying achievable throughput roughly linearly with shard count** — at the cost of cross-shard query complexity.

### Apply Backpressure and Rate Limiting Deliberately

Counterintuitively, throttling _protects_ overall throughput. Without backpressure, a system pushed past its saturation point spends increasing resources on context switching, retries, and half-finished work, and total useful throughput can actually **collapse**. A circuit breaker or rate limiter that rejects excess load fast keeps the system operating at its actual sustainable throughput instead of falling off the cliff.

### Reduce Lock Contention

Fine-grained locking, optimistic concurrency control (compare-and-swap, version columns) instead of pessimistic locks, and lock-free data structures where possible all **reduce the time threads spend waiting** instead of working — directly increasing the rate of completed operations.

---

## 6. The Tradeoff, Made Explicit

Because of Little's Law, most "throughput" optimizations either cost some latency per request (batching, queuing) or are latency-neutral (horizontal scaling, better indexing). Most "latency" optimizations either cost some resource efficiency (caching uses memory, parallel fan-out uses more simultaneous connections) or are throughput-neutral (removing a network hop helps both).

A few concrete tension points worth knowing by name:

|Technique|Throughput effect|Latency effect|Why|
|---|---|---|---|
|**Batching**|↑ Higher|↓ Worse (per item)|You wait for the batch window to fill|
|**Strong consistency**|↓ Lower|↓ Worse|Waiting for quorum/replica acknowledgment adds round trips|
|**Relaxed consistency**|↑ Higher|↑ Better|Serve from a nearby replica; risk staleness|
|**Connection pool too small**|↓ Caps throughput|—|Requests queue waiting for a connection|
|**Connection pool too large**|↓ Can hurt throughput|↓ Worse|Too many concurrent DB ops → contention and context switching|
|**Sync → async processing**|↑ Higher (batch/schedule)|↑ Better (user-facing)|Removes work from the user-facing critical path|

### When to optimize which

- **Optimize latency** when user-facing responsiveness is the goal — checkout flows, search-as-you-type, interactive UIs.
- **Optimize throughput** when total capacity under load is the goal — batch ETL, log ingestion, bulk notification sending.

Most real systems have both kinds of paths and need different strategies for each. **Knowing _which_ path you're optimizing, before reaching for a fix, matters more than any individual technique on this list.**

---

## Summary

||Latency|Throughput|
|---|---|---|
|**Measures**|Time per operation|Operations per unit time|
|**Unit**|ms, μs, s|req/s, MB/s, TPS|
|**Analogy**|Time for one car to cross the highway|Cars passing a checkpoint per hour|
|**Linked by**|Little's Law: `L = λ × W` (latency = concurrency / throughput)||
|**Primary optimization lever**|Shrink the critical path — parallelize, cache, reduce hops|Increase usable concurrency — scale out, batch, async I/O|
|**Common cost**|Resource usage (memory, connections)|Per-item latency (batching, queuing)|
|**What to watch**|p99 / p99.9, **not** the average|Sustained rate under saturation, **not** peak burst|