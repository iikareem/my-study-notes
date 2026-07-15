---
tags:
  - companion
  - debugging
  - interview
  - latency
  - observability
  - problem-6
  - problems
  - system-design
---

# Companion — Production Debugging Playbook

**Domain:** Observability & Debugging in Production  
**Topics:** debugging funnel · metrics · traces · logs

**Pairs with:** [[Our API is slow but only for some users and we can't reproduce it]] · [[Problems MOC]]

A complete mental model for diagnosing latency and performance issues in production — from first symptom to root cause.

---

## The Debugging Funnel

The core instinct is correct: narrow from wide to narrow. Never jump straight to traces — that's like reading every book in a library instead of checking the index first.

```
METRICS     → something is wrong, which layer?
    ↓
LOGS        → which users, what pattern?
    ↓
TRACES      → which span, which operation?
    ↓
PROFILER    → which line of code, which function?
```

Each layer filters the problem space before you go deeper. Each hands off to the next when it hits its limit.

---

## Layer 1 — Metrics: Detect & Locate the Symptom

**Questions metrics answer:**

- Is this affecting everyone or some users?
- Is it one endpoint or all endpoints?
- Is it the app or the database?
- When exactly did it start?

**Signals to look at:**

```
├── p99 latency per endpoint         → which endpoint is the culprit
├── p99 DB query time                → is it the app or the DB?
├── p99 external service call time   → is it a third-party?
├── error rate                       → is it slow or failing?
└── saturation metrics               → connection pool, queue depth, thread count
```

**The critical split — app latency vs DB latency:**

|Reading|Interpretation|
|---|---|
|App p99 = 4200ms, DB p99 = 80ms|Problem is in the app (N+1? serialization? external call?)|
|App p99 = 4200ms, DB p99 = 4100ms|Problem is in the DB (bad query? hot shard? lock?)|

Metrics give you _which layer is slow_ before you touch a single log line.

**Special case — Event Loop (async runtimes):**

Add a canary probe that measures how long the loop takes to respond. This gives you a metric-level signal for event loop blocking so you catch it here in Layer 1 instead of needing a profiler.

```javascript
// Node.js
setInterval(() => {
    const start = Date.now()
    setImmediate(() => {
        const lag = Date.now() - start
        metrics.gauge('event_loop.lag_ms', lag)  // alert if > 100ms
    })
}, 500)
```

```python
# Python asyncio
async def monitor_event_loop():
    while True:
        start = asyncio.get_event_loop().time()
        await asyncio.sleep(0)
        lag = asyncio.get_event_loop().time() - start
        metrics.gauge('event_loop.lag_ms', lag * 1000)
```

---

## Layer 2 — Logs: Find the Pattern

**Questions logs answer:**

- Which specific users are slow?
- What do slow users have in common?
- Which instance handled the slow requests?
- Is there a data pattern (large accounts, specific region, specific plan)?

**Queries to run in your log aggregator:**

```sql
-- Find slow requests
SELECT user_id, duration_ms, instance_id, shard_id, trace_id
WHERE endpoint = '/api/orders'
  AND duration_ms > 1000
ORDER BY duration_ms DESC
LIMIT 100

-- Find the pattern
SELECT shard_id, COUNT(*), AVG(duration_ms)
WHERE duration_ms > 1000
GROUP BY shard_id
-- → shard_3 has 97% of slow requests ← hot partition found

-- Or find data volume pattern
SELECT user_id, row_count, duration_ms
WHERE duration_ms > 1000
ORDER BY row_count DESC
-- → all slow users have row_count > 10,000 ← data volume problem
```

Logs give you _who_ and _what pattern_ — and critically the `trace_id` to jump directly to the right trace.

---

## Layer 3 — Traces: Find the Exact Span

**Questions traces answer:**

- Which operation inside the request is slow?
- Is it one DB call or many small ones?
- Is the external service call timing out?
- Is there a sequential chain that should be parallel?

**What you see:**

```
POST /api/orders  [4318ms]
├── auth                    [12ms]   ✅
├── fetch_user              [8ms]    ✅
├── fetch_order_history     [4271ms] ⚠️ ← go here
│   ├── db.query            [4265ms] ⚠️ ← DB span slow
│   │     shard=3
│   │     rows_returned=30000        ← data volume
│   │     query="SELECT * FROM orders WHERE user_id=?"
│   └── serialize           [6ms]    ✅
└── create_order            [24ms]   ✅
```

Traces give you _exactly where time was spent_ with full context — no guessing.

**Trace attributes are as important as span timing.** A span that says `db.query` took 4200ms is not enough. A span that says `db.query` took 4200ms, `shard=3`, `rows=30000` tells you the root cause immediately.

```python
with tracer.start_as_current_span("db.query") as span:
    span.set_attribute("db.shard_id", shard_id)       # → HOT SHARD
    span.set_attribute("db.rows_returned", row_count)  # → DATA VOLUME
    span.set_attribute("db.query", sql)                # → WHICH QUERY
    span.set_attribute("cache.hit", cache_hit)         # → CACHE MISS
    result = db.query(sql, params)
```

> Traces without rich attributes are just timing data. Traces with attributes are a diagnosis.

---

## Layer 4 — Profiler: Find the Exact Line of Code

Traces stop when you reach an app span with no child spans — no DB call, no HTTP call, just slow application code. This is where profiling takes over.

```
POST /api/orders [4200ms]
└── process_order_logic [4190ms] ⚠️
      ← no DB call, no external call
      ← just... slow
```

### Three App-Level Problems That Need Profiling

**Problem A — CPU Bound**

```python
def process_order(order):
    for item in order.items:
        price = calculate_price(item)   # what's inside here?
        tax   = calculate_tax(price)
```

CPU profiler reveals:

```
calculate_price         → 0.3ms  ✅
  └── apply_discounts   → 0.2ms  ✅
        └── regex.match → 3800ms ⚠️  ← regex compiled fresh on every call
```

Invisible from traces. Visible immediately in a CPU flame graph.

**Problem B — Event Loop Blocking (Node.js / Python asyncio)**

```javascript
// Looks async, but crypto is CPU-bound
async function processOrder(order) {
    const hash = crypto
        .createHash('sha256')
        .update(JSON.stringify(largePayload))  // blocks event loop
        .digest('hex')
}
```

One synchronous CPU operation blocks the entire event loop. From the outside this looks like random users being slow:

```
Request A  ████████████████████████████  4200ms  ← blocking
Request B          ░░░░░░░░░░░░░░░░░░░░  waiting
Request C                  ░░░░░░░░░░░░  waiting
```

Signal: CPU usage normal, but event loop lag metric spikes.

**Problem C — Memory Pressure / GC Pauses**

```python
def process_order(order):
    all_history = [serialize(o) for o in user.orders]   # 30,000 objects
    filtered    = [o for o in all_history if o.active]  # another 30,000
    sorted_list = sorted(filtered, key=lambda x: x.date) # another copy
```

Three full copies of 30,000 objects triggers GC pauses:

```
GC pause: 800ms  ← entire process freezes
GC pause: 750ms
GC pause: 1200ms
```

From traces this looks like the function is randomly slow. Only visible in a memory profiler or GC log.

### Profiling Tools by Runtime

```
Python
├── CPU:    cProfile, py-spy (production safe, attaches to live process)
├── Memory: memory_profiler, tracemalloc, memray
└── Async:  austin, pyinstrument

Node.js
├── CPU:    --prof flag, clinic.js flame, 0x
├── Memory: --inspect + Chrome DevTools heap snapshot
└── Event loop lag: clinic.js doctor, custom setImmediate probe

Go
├── CPU:    pprof (built-in HTTP endpoint)
├── Memory: pprof heap profile
└── Goroutine: pprof goroutine dump

Java / Kotlin
├── CPU:    async-profiler, JFR (Java Flight Recorder)
├── Memory: heap dump + Eclipse MAT
└── Threads: thread dump (jstack)
```

### Production-Safe Profiling

Most profilers have two modes:

|Mode|Overhead|Safe for production?|
|---|---|---|
|Development (instrument everything)|10–50% CPU|No|
|Sampling (sample call stack every N ms)|1–3% CPU|Yes|

**Example — py-spy in production (Python):**

```bash
# Attach to running process with no code changes
py-spy record -o profile.svg --pid 12345 --duration 30

# Or live top-like view
py-spy top --pid 12345
```

**Output — a flame graph:**

```
▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  process_order (95%)
▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓            apply_discounts (71%)
▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓                re.match (63%)  ← the culprit
▓▓▓▓                                     json.loads (9%)
▓▓                                       db.query (5%)
```

Width of each bar = percentage of time spent. `re.match` consuming 63% of all CPU is something no metric or trace would show you.

---

## Root Cause Decision Tree

Once you know which span is slow, apply domain knowledge:

```
DB span is slow
      ├── High row_count?
      │     → Data volume problem (missing LIMIT, unbounded query)
      │
      ├── Same shard for all slow requests?
      │     → Hot partition (redistribute data, add read replica)
      │
      ├── Lock wait time visible in DB metrics?
      │     → Lock contention (find who holds the lock, batch the writer)
      │
      ├── Query plan changed? (check with EXPLAIN)
      │     → Missing index, wrong plan, stale statistics
      │
      └── Specific time of day?
            → Competing background job

External service span is slow
      ├── All requests slow?
      │     → Provider outage (check status page, add circuit breaker)
      │
      └── Some requests slow?
            → Timeout misconfigured, or specific payload triggers slowness

App span is slow (no DB or external call)
      ├── High memory allocation?
      │     → Serializing too much data, large object in memory → profiler (memory)
      │
      ├── CPU spike?
      │     → Expensive computation, regex on large input, N+1 in app code → profiler (CPU)
      │
      └── Event loop lag metric spiking?
            → Blocking call in async runtime → profiler (event loop)
```

---

## The Handoff Conditions

```
Traces show DB span slow          → go to DB (EXPLAIN, shard analysis)
Traces show external span slow    → go to provider (status page, circuit breaker)
Traces show app span slow
    └── with no child spans       → go to profiler
          ├── CPU flame graph     → regex, serialization, CPU-bound work
          ├── Memory profiler     → GC pressure, large allocations, leaks
          └── Event loop monitor  → blocking call in async runtime
```

---

## Complete Tool Reference

|Layer|Tools|Answers|Blind Spot|
|---|---|---|---|
|Metrics|Prometheus, Datadog|What, when, which layer|Not why|
|Logs|Loki, Splunk, Datadog|Who, what pattern|Not where in code|
|Traces|Jaeger, Tempo, OTEL|Which operation, which span|Inside pure app code|
|Profiler|py-spy, pprof, async-profiler|Which function, which line|Requires process access|

---

## The Full Cycle

```
SYMPTOM        p99 spike, user complaints
    ↓
METRICS        which endpoint? app slow or DB slow? since when?
    ↓
LOGS           which users? what pattern? shard? plan? data size?
    ↓
TRACES         which span? what attributes? rows? shard? cache hit?
    ↓
PROFILER       which function? which line? CPU / memory / event loop?
    ↓
ROOT CAUSE
    ├── DB span slow + high rows    → unbounded query, add LIMIT
    ├── DB span slow + same shard   → hot partition, redistribute
    ├── DB span slow + lock wait    → lock contention, find the writer
    ├── External span slow          → provider issue, circuit breaker
    ├── App span slow + CPU         → regex, N+1, CPU-bound computation
    ├── App span slow + memory      → GC pressure, large allocations
    └── App span slow + event loop  → blocking call in async runtime
    ↓
ACTION
    ├── Immediate  → rollback, feature flag, kill the query
    └── Proper fix → fix root cause + add targeted alert at the metric level
```

The loop closes back to metrics: after fixing, add a targeted alert so next time you catch it in Layer 1 in minutes instead of discovering it from user complaints.

## Next

- Scenario: [[Our API is slow but only for some users and we can't reproduce it]]
- Back to hub: [[Problems MOC]]
