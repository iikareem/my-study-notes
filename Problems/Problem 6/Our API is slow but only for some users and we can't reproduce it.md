---
tags:
  - debugging
  - interview
  - latency
  - observability
  - performance
  - problem-6
  - problems
  - system-design
  - tail-latency
---

# Problem 6 — "Our API is slow but only for some users and we can't reproduce it"

**Domain:** Observability & Debugging in Production  
**Topics:** observability · latency · tracing · metrics · debugging

**In this folder:** scenario (this note) · [[Problem 6 Companion]]

## 🧩 The Scenario

You're on-call. At 2pm you get paged — P1 alert, API p99 latency spiked from 80ms to **4200ms.** Average latency looks fine at 95ms. The spike started 47 minutes ago.

You check the usual suspects:

```
CPU usage:        42%  → normal
Memory:           61%  → normal  
DB connections:   45/100 → normal
Error rate:       0.3% → normal
Active instances: 8    → normal
```

Everything looks healthy. But users are screaming. You dig into logs:

```
[INFO]  POST /api/orders  user=USR-001  duration=67ms   ✅
[INFO]  POST /api/orders  user=USR-002  duration=71ms   ✅
[INFO]  POST /api/orders  user=USR-884  duration=4318ms ⚠️
[INFO]  POST /api/orders  user=USR-003  duration=58ms   ✅
[INFO]  POST /api/orders  user=USR-991  duration=3987ms ⚠️
[INFO]  POST /api/orders  user=USR-004  duration=82ms   ✅
```

Slow requests are **random, intermittent, and only affect specific users.** You can't reproduce it locally. Your manager is standing behind you asking for updates every 5 minutes.

---

## 🤔 Think About It First

Why does average latency look normal while p99 is exploding? What does "only specific users" tell you architecturally? And what's the systematic way to debug something you can't reproduce?

---

## 🔍 Root Cause Analysis

### Why Average Hides the Problem — Percentiles vs Mean

```
Request latencies in one minute:
[65, 70, 68, 72, 4200, 69, 71, 4318, 67, 73]

Average: (65+70+68+72+4200+69+71+4318+67+73) / 10 = 897ms
                                                      ← looks bad but diluted

p50 (median): 70ms   ← most users fine
p95:          4200ms ← 5% of users suffering
p99:          4318ms ← worst 1% of users
```

**The mean lies.** A small percentage of catastrophically slow requests gets averaged away by the majority of fast ones. This is why production monitoring must track **percentiles, not averages.**

```
Average = 95ms  → "everything is fine"
p99     = 4200ms → "1 in 100 users waits 4 seconds"
```

Both statements are simultaneously true. Only one tells you there's a problem.

---

### What "Only Specific Users" Tells You

Random users being slow while others are fast is a strong signal. It rules out:

```
❌ Global DB slowdown      → would affect everyone
❌ Network issue           → would affect everyone
❌ CPU/memory pressure     → would affect everyone proportionally
❌ Bad code deploy         → would affect everyone
```

It points toward:

```
✅ Data-specific slowness   → certain users have more data / complex queries
✅ Hot shard / partition    → certain users route to a slow DB shard
✅ External service call    → some requests hit a slow third-party API
✅ Specific instance issue  → some app instances are degraded
✅ Cache miss pattern       → certain users bypass cache
✅ Lock contention          → certain users' data is being locked
```

"Random specific users" is your most important clue — it narrows the problem space dramatically.

---

### The Systematic Debug Framework

Most engineers jump to guesses. The right move is **structured elimination:**

```
1. WHEN    → exactly when did it start? what changed?
2. WHO     → which users? any pattern?
3. WHERE   → which instance? which DB shard? which region?
4. WHAT    → which endpoint? which operation inside it?
5. WHY     → root cause
```

---

## ✅ The Investigation — Step by Step

### Step 1 — Correlate the Start Time with Deployments / Changes

```bash
# When did p99 spike?
# Answer: 2:00pm exactly

# What happened at 2:00pm?
git log --since="1:55pm" --until="2:05pm"
# → deployment at 1:58pm: "optimize order history query"

# Check deploy history
kubectl rollout history deployment/order-service
# → revision 47 deployed at 13:58:22
```

**Always check what changed.** Most production incidents have a cause that deployed in the last hour.

---

### Step 2 — Find the Pattern in Slow Users

```python
# Pull slow requests from logs
slow_users = [USR-884, USR-991, USR-445, USR-1203, USR-778 ...]

# Query: what do these users have in common?
SELECT
    u.id,
    u.plan,
    u.created_at,
    COUNT(o.id) as order_count,
    u.shard_id
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.id IN (slow_users)
GROUP BY u.id
```

```
Results:
USR-884   plan=enterprise  orders=15,847  shard=3
USR-991   plan=enterprise  orders=22,103  shard=3
USR-445   plan=enterprise  orders=18,442  shard=3
USR-1203  plan=enterprise  orders=31,205  shard=3
USR-778   plan=enterprise  orders=19,887  shard=3
```

**Every slow user is on shard 3 with a large order history.** Two signals at once — a bad shard AND a data volume problem.

---

### Step 3 — Add Distributed Tracing (What Should Already Exist)

This is where the real problem surfaces — you can't debug production latency without traces. A trace breaks one request into its component spans:

```
POST /api/orders  [total: 4318ms]
├── auth middleware         [12ms]   ✅
├── validate request        [3ms]    ✅
├── fetch user              [8ms]    ✅
├── fetch order history  [4271ms]   ⚠️  ← HERE
│   ├── db query            [4265ms] ⚠️
│   └── serialize           [6ms]    ✅
└── create order            [24ms]   ✅
```

Without tracing you know the request is slow. With tracing you know **exactly which line is slow** in under 2 minutes.

```python
# What tracing looks like in code
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def complete_order(user_id, items):
    with tracer.start_as_current_span("complete_order") as span:
        span.set_attribute("user.id", user_id)

        with tracer.start_as_current_span("fetch_order_history"):
            history = db.query("""
                SELECT * FROM orders
                WHERE user_id = ?
                ORDER BY created_at DESC
            """, user_id)      # ← no LIMIT — fetches all 30,000 orders

        with tracer.start_as_current_span("create_order"):
            order = db.execute("INSERT INTO orders ...")

        return order
```

---

### Step 4 — The Actual Bug

```sql
SELECT * FROM orders
WHERE user_id = ?
ORDER BY created_at DESC
-- ← no LIMIT
```

The "optimize order history" deploy changed this query to fetch **all orders for sorting** — removing a `LIMIT 50` that was there before. For regular users with 200 orders this is fine — 200 rows serialize fast. For enterprise users with 30,000 orders:

```
30,000 rows × avg row size 2KB = 60MB transferred from DB
+ serialize 30,000 objects to JSON
+ allocate memory for 30,000 Python dicts
= 4 seconds
```

The query itself is fast. The **data volume** is the problem.

---

### Step 5 — Immediate Mitigation vs Proper Fix

**Immediate — rollback the deploy:**

```bash
kubectl rollout undo deployment/order-service
# p99 drops back to 80ms within 60 seconds
```

**Proper fix — paginate and limit:**

```python
def fetch_order_history(user_id, limit=50, cursor=None):
    query = """
        SELECT id, status, amount, created_at
        FROM orders
        WHERE user_id = ?
        {}
        ORDER BY created_at DESC
        LIMIT ?
    """.format("AND id < ?" if cursor else "")

    params = [user_id, cursor, limit] if cursor else [user_id, limit]
    return db.query(query, *params)
```

Never fetch unbounded data. Always `LIMIT`. Always paginate.

---

## 🧠 The Observability Stack You Need

This incident was hard to debug because the right tools weren't in place. Here's what production systems need:

```
METRICS  → what is happening (Prometheus, Datadog)
  ├── p50, p95, p99 latency per endpoint
  ├── error rate per endpoint
  └── saturation (queue depth, connection pool usage)

LOGS     → what happened for a specific request (structured JSON logs)
  ├── request_id, user_id, duration, status on every log line
  └── never log plain strings — always structured key=value

TRACES   → why it happened and where time was spent (OpenTelemetry, Jaeger)
  ├── span per operation (DB query, external call, cache lookup)
  ├── attributes: user_id, query, row_count, cache_hit
  └── link trace_id across services
```

These are called the **three pillars of observability.** Missing any one of them makes incidents like this take hours instead of minutes.

---

### Structured Logging — The Difference

```python
# Bad — unsearchable, unqueryable
logger.info(f"Order completed for user {user_id} in {duration}ms")

# Good — every field queryable, aggregatable, alertable
logger.info("order.completed", extra={
    "user_id":    user_id,
    "duration_ms": duration,
    "order_id":   order_id,
    "shard_id":   shard_id,
    "row_count":  row_count,
    "trace_id":   trace_id,   # ← links log to trace
})
```

With structured logs you can run:

```sql
-- In your log aggregator (Datadog, Loki, Splunk)
SELECT user_id, AVG(duration_ms), MAX(row_count)
WHERE endpoint = '/api/orders'
  AND duration_ms > 1000
GROUP BY user_id
ORDER BY AVG(duration_ms) DESC
```

This query would have found the pattern in **2 minutes** instead of 47.

---

## 📊 Debug Framework Summary

```
INCIDENT STARTS
      ↓
Check percentiles (p99 not average)
      ↓
Identify who is affected → find the pattern
      ↓
Correlate with recent changes → deployments, config, data growth
      ↓
Use traces → find which span is slow
      ↓
Use structured logs → query for the pattern
      ↓
Immediate: rollback or feature flag
Long term: fix root cause + add alerting so you catch it earlier next time
```

---

## 🔑 Key Takeaways

|Lesson|Rule|
|---|---|
|Average vs percentiles|Average lies — always monitor p50/p95/p99|
|Specific users slow|Rules out global issues — look for data, shard, or routing patterns|
|Three pillars|Metrics tell you _what_, logs tell you _what happened_, traces tell you _why_|
|Structured logging|Every log line needs request_id, user_id, duration — never plain strings|
|Unbounded queries|Always `LIMIT` — data volume kills performance silently|
|Rollback first|Restore service, then fix — never debug while users are down|
|Trace IDs|Link logs to traces — one ID that follows a request everywhere|

---

## Next

- Companion deep dive: [[Problem 6 Companion]]
- Back to hub: [[Problems MOC]]
