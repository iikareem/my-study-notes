---
tags:
  - caching
  - interview
  - invalidation
  - problem-8
  - problems
  - system-design
  - thundering-herd
---

# Problem 8 — "We added a cache and our system got slower and more broken than before"

**Domain:** Caching & Memory Architecture  
**Topics:** caching · thundering herd · invalidation · hot keys

**In this folder:** scenario (this note) · [[Problem 8 Companion]]

## 🧩 The Scenario

Your product dashboard loads slowly — it aggregates data from several tables. A senior engineer says _"just add Redis cache"_ and leaves for vacation. You implement it:

```python
CACHE_TTL = 3600  # 1 hour

def get_dashboard(user_id):
    cache_key = f"dashboard:{user_id}"

    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    data = db.query("""
        SELECT
            COUNT(o.id)          as total_orders,
            SUM(o.amount)        as total_revenue,
            AVG(o.amount)        as avg_order_value,
            COUNT(DISTINCT o.customer_id) as unique_customers
        FROM orders o
        WHERE o.user_id = ?
          AND o.created_at >= NOW() - INTERVAL '30 days'
    """, user_id)

    redis.setex(cache_key, CACHE_TTL, json.dumps(data))
    return data
```

You deploy. Three things go wrong immediately:

- **Problem A** — At 9am Monday every user hits the dashboard simultaneously. The site goes down for 4 minutes even though Redis is running fine.
- **Problem B** — A user updates their order data, refreshes the dashboard, sees stale numbers. They call support convinced your platform has a bug. Your TTL is 1 hour so they'll see wrong data for up to 60 minutes.
- **Problem C** — Someone deletes Redis to "clear the cache." Every single dashboard query hits the DB simultaneously. DB CPU spikes to 100%, connection pool exhausts, entire platform goes down for 8 minutes.

Your manager says: _"The cache made things worse. Remove it."_ You know the cache is right — the implementation is wrong.

---

## 🤔 Think About It First

Each problem has a name in distributed systems. Problem A and Problem C sound similar but are caused by different things and have different fixes. What are they called and what's the structural difference between them?

---

## 🔍 Root Cause Analysis

### Problem A — Cache Stampede (Thundering Herd)

At 9am Monday, 10,000 users open their dashboards simultaneously. All their cache keys expired overnight (TTL = 1 hour, set at 9pm Sunday):

```
t=9:00:00am
  User 1:  GET dashboard:USR-001 → MISS → query DB → set cache
  User 2:  GET dashboard:USR-002 → MISS → query DB → set cache
  User 3:  GET dashboard:USR-003 → MISS → query DB → set cache
  ...
  User 10,000: GET dashboard:USR-10000 → MISS → query DB → set cache

= 10,000 simultaneous DB queries
= DB overwhelmed, connections exhausted, site down
```

This is a **cache stampede** — many cache misses happening simultaneously, all rushing to the DB at once like a herd of stampeding animals.

The key characteristic: **all keys expire at the same time** because they were all set at the same time with the same TTL.

---

### Problem C — Cache Avalanche

Someone deletes all Redis keys at once. Now **every key is missing simultaneously** for every user, every endpoint, every cached object:

```
Redis flushed at t=0

t=0.001s: dashboard requests  → all miss → all hit DB
t=0.001s: user profile requests → all miss → all hit DB
t=0.001s: product page requests → all miss → all hit DB
t=0.001s: search requests → all miss → all hit DB

= entire platform's cached data gone simultaneously
= DB receives every query it was shielded from
= instant overload
```

This is a **cache avalanche** — the entire cache layer collapses at once, exposing the DB to full traffic suddenly.

---

### The Structural Difference

```
Cache Stampede:  one key, many concurrent misses, all racing to rebuild it
Cache Avalanche: all keys, all missing simultaneously, entire cache layer gone
```

Same symptom (DB overload), different cause, different fix.

---

### Problem B — Cache Invalidation

The hardest problem in caching. Your data changed in the DB but the cache still holds the old value:

```
t=0:    user places order → DB updated ✅
t=0:    cache still has old dashboard data ← stale
t=3600: cache expires → fresh data loaded ✅

Window of staleness: up to 60 minutes
```

The famous quote in computer science:

> _"There are only two hard things in computer science: cache invalidation and naming things."_ — Phil Karlton

---

## ✅ Fix for Problem A — Cache Stampede

### Fix 1 — Jitter on TTL (Simplest)

Instead of every key expiring at exactly the same time, add randomness:

```python
import random

def get_dashboard(user_id):
    cache_key = f"dashboard:{user_id}"
    cached = redis.get(cache_key)

    if cached:
        return json.loads(cached)

    data = compute_dashboard(user_id)

    # Add random jitter — keys expire between 50-70 minutes
    # not all at exactly 60 minutes
    ttl = 3600 + random.randint(-600, 600)
    redis.setex(cache_key, ttl, json.dumps(data))

    return data
```

```
User 1 key expires at: 54 min
User 2 key expires at: 67 min
User 3 key expires at: 61 min
User 4 key expires at: 48 min

→ misses spread out over time → DB load spread out → no stampede
```

Simple and effective for user-specific keys. Not enough for shared keys (one key, many readers).

---

### Fix 2 — Probabilistic Early Recomputation (XFetch)

Instead of waiting for expiry, **proactively recompute** the cache before it expires — with probability increasing as expiry approaches:

```python
import math, random, time

def get_dashboard_xfetch(user_id, beta=1.0):
    cache_key   = f"dashboard:{user_id}"
    delta_key   = f"dashboard:delta:{user_id}"  # stores compute time

    cached    = redis.get(cache_key)
    ttl       = redis.ttl(cache_key)
    delta     = float(redis.get(delta_key) or 1)  # time to compute

    if cached:
        # XFetch formula — recompute early with increasing probability
        # as TTL approaches zero
        should_recompute = (-delta * beta * math.log(random.random())) >= ttl

        if not should_recompute:
            return json.loads(cached)
        # else fall through and recompute proactively

    start = time.time()
    data  = compute_dashboard(user_id)
    delta = time.time() - start   # record how long computation takes

    redis.setex(cache_key, 3600, json.dumps(data))
    redis.setex(delta_key, 7200, delta)

    return data
```

A query that takes 2 seconds to compute starts recomputing **early** — before expiry, while the old value is still being served. Zero cold miss window.

---

### Fix 3 — Mutex Lock (For Shared / Global Keys)

For keys shared by many users (e.g. a global leaderboard, site-wide stats), use a **distributed lock** so only one worker recomputes while others wait or serve stale:

```python
def get_global_stats():
    cache_key = "global:stats"
    lock_key  = "global:stats:lock"

    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # Try to acquire lock — only one process wins
    lock_acquired = redis.set(lock_key, "1", nx=True, ex=30)

    if lock_acquired:
        try:
            data = compute_global_stats()   # expensive
            redis.setex(cache_key, 3600, json.dumps(data))
            return data
        finally:
            redis.delete(lock_key)
    else:
        # Lost the lock race — wait briefly and retry
        # or serve a slightly stale value
        time.sleep(0.1)
        cached = redis.get(cache_key)
        if cached:
            return json.loads(cached)
        raise ServiceUnavailableError("Stats temporarily unavailable")
```

```
10,000 requests all miss:
→ one acquires lock and recomputes
→ 9,999 wait 100ms and get fresh value from cache
→ DB receives exactly 1 query instead of 10,000
```

---

## ✅ Fix for Problem C — Cache Avalanche

### Fix 1 — Staggered Warming After Flush

Never flush production cache without a warming strategy:

```python
def safe_cache_flush():
    # Step 1 — get all keys before flushing
    all_keys = redis.keys("dashboard:*")

    # Step 2 — flush
    redis.flushdb()

    # Step 3 — warm critical keys immediately, staggered
    for i, key in enumerate(all_keys):
        user_id = key.split(":")[1]

        # Stagger the warming — don't hit DB with all at once
        time.sleep(0.01 * (i % 10))

        warm_cache.delay(user_id)   # background job
```

Or better — **never flush in production.** Use key expiry and targeted invalidation instead.

### Fix 2 — Circuit Breaker on DB

If cache is gone and DB is overwhelmed, a circuit breaker stops the cascade:

```python
from circuit_breaker import CircuitBreaker

db_breaker = CircuitBreaker(
    failure_threshold=10,    # open after 10 failures
    recovery_timeout=30,     # try again after 30s
    expected_exception=DBTimeoutError
)

def get_dashboard(user_id):
    cached = redis.get(f"dashboard:{user_id}")
    if cached:
        return json.loads(cached)

    try:
        with db_breaker:
            data = compute_dashboard(user_id)
            redis.setex(f"dashboard:{user_id}", 3600, json.dumps(data))
            return data
    except CircuitOpenError:
        # Circuit is open — DB is struggling
        # Return degraded response rather than timing out
        return {"error": "dashboard_temporarily_unavailable", "retry_after": 30}
```

During an avalanche the circuit opens after 10 failures — remaining requests get a degraded response immediately instead of waiting 30 seconds to timeout and exhausting connection pools.

---

## ✅ Fix for Problem B — Cache Invalidation

### Strategy 1 — Active Invalidation on Write

When data changes, immediately delete or update the cache:

```python
def update_order(order_id, user_id, data):
    db.execute("UPDATE orders SET ... WHERE id = ?", order_id)

    # Immediately invalidate — next read will be fresh
    redis.delete(f"dashboard:{user_id}")
```

Simple and effective. The next dashboard read rebuilds the cache from fresh DB data. Zero staleness window.

**Tradeoff:** You must remember to invalidate in every code path that writes. Miss one → stale data. This gets hard in large codebases.

---

### Strategy 2 — Write-Through Cache

Write to cache **at the same time** as the DB — cache is always current:

```python
def update_order(order_id, user_id, new_amount):
    db.execute("UPDATE orders SET amount = ? WHERE id = ?", new_amount, order_id)

    # Recompute and update cache immediately
    fresh_data = compute_dashboard(user_id)
    redis.setex(f"dashboard:{user_id}", 3600, json.dumps(fresh_data))
```

**Tradeoff:** Every write triggers a cache recompute — expensive if writes are frequent. Good when reads vastly outnumber writes.

---

### Strategy 3 — Event-Driven Invalidation (Most Robust)

From Problem 3 — publish an event when data changes, a cache invalidator subscribes:

```python
# Order Service — on any order change
def update_order(order_id, user_id, data):
    db.execute("UPDATE orders ...")
    event_bus.publish("order.updated", {"user_id": user_id, "order_id": order_id})

# Cache Invalidation Service — subscribes to order events
def on_order_updated(event):
    redis.delete(f"dashboard:{event.user_id}")
    # optionally pre-warm:
    warm_dashboard_cache.delay(event.user_id)
```

**Tradeoff:** Small propagation delay (milliseconds). But decoupled — the order service doesn't need to know about caching. New caches can be added by subscribing to existing events.

---

## 🧠 Caching Strategies — The Full Mental Model

```
READ strategies:
├── Cache-aside (lazy loading)   → check cache, miss → load DB → fill cache
│                                   your original implementation
│                                   simple, only caches what's needed
│
└── Read-through                 → cache sits in front, loads DB automatically
                                    on miss, transparent to app code

WRITE strategies:
├── Write-around                 → write to DB only, cache filled on next read
│                                   simple, but cold cache after writes
│
├── Write-through                → write to DB + cache simultaneously
│                                   fresh cache always, expensive on write
│
└── Write-behind (write-back)    → write to cache, async flush to DB
                                    fastest writes, risk of data loss
```

---

## 📊 Problem → Fix Summary

|Problem|Name|Root Cause|Fix|
|---|---|---|---|
|9am spike crashes DB|Cache Stampede|Many keys expire simultaneously|TTL jitter, XFetch, mutex lock|
|Redis flush crashes DB|Cache Avalanche|Entire cache wiped at once|Staggered warming, circuit breaker|
|Stale data after writes|Cache Invalidation|Cache not updated on write|Active invalidation, write-through, event-driven|

---

## 🔑 Key Takeaways

|Lesson|Rule|
|---|---|
|TTL jitter|Never use fixed TTL for many keys — add randomness to spread expiry|
|Stampede vs avalanche|Stampede = one key, many readers. Avalanche = all keys gone at once|
|Cache invalidation|Hardest problem — prefer event-driven invalidation for decoupling|
|Circuit breaker|Protect DB from cache failures — fail fast, return degraded response|
|Never flush production|Use targeted invalidation or key expiry — never `FLUSHDB` on live system|
|Write strategy|Match to your read/write ratio — write-through for read-heavy, write-around for write-heavy|
|Cache is not source of truth|DB is always truth — cache is a performance optimization, not storage|

---

## Next

- Companion deep dive: [[Problem 8 Companion]]
- Back to hub: [[Problems MOC]]
