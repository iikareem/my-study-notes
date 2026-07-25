---
tags:
  - postgresql
  - database
  - performance
---

# Prepared Statements — The Full Optimization Guide

---

## 1. What Is a Driver

Before anything else — the driver is just a **library installed inside your app** that handles all communication with the database. Your code has no idea how to speak PostgreSQL's wire protocol, open TCP sockets, or serialize data. The driver does all of that.

```
Your App Process (memory)
┌─────────────────────────────────────────┐
│  Your Code                              │
│                                         │
│  Driver (e.g. psycopg2, node-postgres)  │
│  ┌───────────────────────────────────┐  │
│  │ - opens TCP socket to DB          │  │
│  │ - speaks the wire protocol        │  │
│  │ - serializes your params          │  │
│  │ - caches prepared handles         │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
          │
          │  TCP
          ▼
   Database Server
```

Common drivers by language:

|Language|PostgreSQL|MySQL|
|---|---|---|
|Python|psycopg2, asyncpg|mysql-connector-python|
|Node.js|pg (node-postgres)|mysql2|
|Java|JDBC|JDBC|
|Go|pgx, lib/pq|go-sql-driver|

When you `pip install psycopg2` — that's your driver.

---

## 2. What Happens Without a Prepared Statement

Every time you send a raw query, the database runs the full pipeline from scratch:

```
Raw SQL String
     │
     ▼
┌─────────────┐
│   Parse     │  tokenize and validate SQL syntax
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Analyze   │  resolve table names, column names, types
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Rewrite   │  expand views, CTEs, apply rewrite rules
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Plan      │  pick indexes, join strategies, scan types
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Execute   │  actually run against data
└─────────────┘
```

Run the same query 10,000 times with different values — all five steps run **every single time**. That's the waste prepared statements eliminate.

---

## 3. The Driver Cache — No Extra Round Trip

A common misconception: people assume the driver sends a "does this exist?" check to the server before each execute. It doesn't. That would be an extra round trip and would defeat the purpose entirely.

The driver maintains a **local in-memory map** of what it has already prepared:

```
Driver memory (inside your app process):
{
  "SELECT * FROM users WHERE id = $1":        "handle_0x4A",
  "SELECT * FROM orders WHERE country = $1":  "handle_0x4B",
}
```

Before any call it checks **this local map** — zero network involved.

```
First call:
  driver checks cache → not there
  → sends PREPARE (SQL text) to server
  → server parses + plans, stores in session
  → driver stores the handle in local cache
  → sends EXECUTE (handle + params)

Every call after:
  driver checks cache → found
  → sends EXECUTE (handle + params) only
  → no prepare, no SQL text over the wire
```

---

## 4. What Actually Goes Over the Wire

### Without prepared statement — every call

```
Client                              Server
  │                                    │
  │── "SELECT * FROM orders            │
  │    WHERE country = 'DE'" ────────► │  parse, plan, execute
  │ ◄──────────────────── results ────│
```

Full SQL text sent every time, server does full pipeline every time.

### With prepared statement — first call

```
Client                              Server
  │                                    │
  │── PREPARE (SQL text) ───────────► │  parse + plan, stored in session
  │ ◄── handle 0x4A ─────────────────│
  │                                    │
  │── EXECUTE 0x4A + ['DE'] ────────► │  execute only
  │ ◄──────────────────── results ────│
```

### With prepared statement — every subsequent call

```
Client                              Server
  │                                    │
  │── EXECUTE 0x4A + ['DE'] ────────► │  straight to execute
  │ ◄──────────────────── results ────│
```

No SQL text. No parsing. No planning. Just a handle and binary params.

---

## 5. Binary Protocol — The Other Wire Saving

In the execute phase, parameters travel as **typed binary values**, not text:

```
Text protocol:    WHERE id = 1000000   →  "1000000" = 7 bytes of ASCII + quoting
Binary protocol:  integer 1000000      →  4 bytes, no parsing needed
```

The server also skips type coercion entirely — it already knows from the prepare phase that `$1` maps to an `INT` column. For high-frequency queries with floats, timestamps, or BLOBs, this compounds fast.

---

## 6. Plan Caching — What the Server Actually Stores

When the server receives PREPARE, it runs Parse → Analyze → Rewrite → Plan and stores the result.

| Stage           | Cached?    | Notes                                |
| --------------- | ---------- | ------------------------------------ |
| Parse tree      | ✅ Yes      | syntax validated once                |
| Name resolution | ✅ Yes      | table/column lookups done once       |
| Query rewrite   | ✅ Yes      | view expansions resolved once        |
| Query plan      | ⚠️ Depends | PostgreSQL has special behavior here |
| Execution       | ❌ No       | always runs fresh against data       |

---

## 7. PostgreSQL's Custom vs Generic Plan

This is where it gets interesting. PostgreSQL faces a real dilemma at prepare time — it doesn't know what value `$1` will be, and the best plan depends heavily on that value.

### The problem with skewed data

```sql
PREPARE get_orders FROM 'SELECT * FROM orders WHERE country = $1';
```

Imagine your data distribution:

```
country = 'US'  →  8,500,000 rows  (85% of table)
country = 'DE'  →     50,000 rows  (0.5% of table)
country = 'JP'  →      1,000 rows  (0.01% of table)
```

The optimal plan is completely different per value:

- `'US'` → **Sequential scan** — index is useless when touching 85% of the table
- `'DE'` → **Index scan** — selective enough to be worth it
- `'JP'` → **Index scan** — highly selective, index is very fast

### Calls 1–5: Custom Plan

PostgreSQL re-plans with the **actual value** on every execution:

```
EXECUTE get_orders('US')  → planner sees 'US' → picks seq scan   ✅
EXECUTE get_orders('DE')  → planner sees 'DE' → picks index scan ✅
EXECUTE get_orders('JP')  → planner sees 'JP' → picks index scan ✅
EXECUTE get_orders('US')  → planner sees 'US' → picks seq scan   ✅
EXECUTE get_orders('DE')  → planner sees 'DE' → picks index scan ✅
```

Each call pays the planning cost, but gets the optimal plan for that value. PostgreSQL is also **secretly recording the average cost** of these custom plans during this window.

### Call 6: The Decision Point

PostgreSQL builds a **generic plan** — one plan made without knowing the value, using average column statistics:

```
Generic plan assumption: average selectivity across all values
→ probably picks index scan as a middle-ground guess
```

Then it compares:

```
avg cost of custom plans (from calls 1–5)
           vs
estimated cost of generic plan

if generic ≤ custom × 1.1  →  switch to generic permanently
if generic >  custom × 1.1  →  keep doing custom plans forever
```

The 1.1 threshold means it only switches if the generic plan is within 10% as good — giving custom plans a slight bias.

### Where it goes wrong

If calls 1–5 were all selective values (`'DE'`, `'JP'`), PostgreSQL records a low average cost. If it then switches to a generic plan and you execute with `'US'`:

```
Generic plan: index scan (assumed average selectivity)
Actual call:  EXECUTE get_orders('US')  ← 85% of the table

Result: index scan on 8.5M rows, thrashing between
        index and heap pages = catastrophically slow
```

PostgreSQL has no way to detect this per-value regression once it commits to the generic plan.

### The controls you have

```sql
-- Always re-plan with actual values (safest for skewed data)
SET plan_cache_mode = force_custom_plan;

-- Always use generic plan (for very uniform distributions)
SET plan_cache_mode = force_generic_plan;

-- Default adaptive behavior
SET plan_cache_mode = auto;
```

---

## 8. Batch Execution — Multiplying the Gains

Prepared statements are the prerequisite for efficient batching:

```python
# Without — N separate round trips
for user_id in user_ids:
    db.execute("UPDATE users SET last_seen = NOW() WHERE id = %s", user_id)

# With batch — 1 prepare + 1 network write for all values
stmt = db.prepare("UPDATE users SET last_seen = NOW() WHERE id = ?")
stmt.executemany([(id,) for id in user_ids])
```

The batch collapses N execute round trips into a single network write. The server processes them sequentially against the cached plan.

---

## 9. The Connection Pooling Problem (and the Fix)

Prepared statements are **session-scoped** — they live on a specific server connection. This is a problem with connection poolers in transaction mode, because the pooler hands you a different backend connection per transaction, so the prepared statement from the previous connection is gone.

```
Connection Pool (before PgBouncer 1.21)
┌──────────────────────────────────────────────┐
│  Conn A  →  "get_orders" prepared ✅         │
│  Conn B  →  "get_orders" NOT prepared ❌     │
│  Conn C  →  "get_orders" prepared ✅         │
└──────────────────────────────────────────────┘
```

When your app gets `Conn B`, the driver's local cache says the statement is prepared — but the server session on `Conn B` has never seen it. This caused errors like `"prepared statement does not exist"`.

### PgBouncer 1.21 fixed this

Released in 2023, <cite index="1-0">PgBouncer 1.21 added prepared statement support for transaction mode</cite>. The way it works: <cite index="1-0">PgBouncer can now prepare statements correctly on the server side if needed, and it is transparent to the client</cite>. So if your transaction lands on `Conn B` and the statement isn't there, PgBouncer re-prepares it automatically without your app knowing.

The new setting that enables this:

```ini
# pgbouncer.ini
max_prepared_statements = 10
```

<cite index="1-0">You wouldn't want to set this to a very high value because it causes higher memory utilization on the database side — 10 is a good starting point.</cite>

### Caveats that still exist

<cite index="1-0">You can only use this with protocol-level prepared statements. Depending on your client library, it may not work if it uses raw statements like `EXECUTE` and `PREPARE` instead of the protocol-level functions.</cite>

<cite index="1-0">There is currently no way to do a protocol-level `DEALLOCATE` — that support only arrives in PostgreSQL 17. This means drivers that issue `DEALLOCATE` themselves can produce unexpected errors. Some drivers now offer a way to skip deallocations and trust PgBouncer to handle cleanup instead.</cite>

### If you can't upgrade PgBouncer

- Run PgBouncer in **session mode** — one connection per client, session state is preserved, but you lose the connection multiplexing benefit
- Use **driver-side emulation** — driver interpolates params client-side and sends plain SQL, no server-side caching but no session problem either
- Use **unnamed statements** — deallocated after each execute, pooler-safe but no plan caching

---

## 10. When Prepared Statements Hurt

They're not always a win.

**One-shot queries** — if a query runs exactly once, you pay the prepare round trip with zero reuse:

```
Without: 1 round trip (parse + execute together)
With:    2 round trips (prepare, then execute)
```

**DDL changes** — adding a column, dropping an index, or rebuilding a table can invalidate cached plans. The database re-plans silently or returns an error, forcing a re-prepare.

**Dynamic WHERE clauses** — if query structure changes based on optional filters, each shape needs its own prepared handle:

```sql
SELECT * FROM products WHERE category = ?
SELECT * FROM products WHERE category = ? AND price < ?
SELECT * FROM products WHERE category = ? AND price < ? AND in_stock = ?
```

These are three different statements. Managing all their handles adds complexity that may not be worth it.

**Skewed data + generic plan** — as covered in §7, the generic plan can be catastrophically wrong for non-uniform distributions.

---

## 11. Measuring the Actual Gain

```sql
-- PostgreSQL: see active prepared statements and usage
SELECT name, calls, mean_exec_time
FROM pg_prepared_statements;

-- MySQL: check execute vs prepare ratio
SHOW STATUS LIKE 'Com_stmt_execute';
SHOW STATUS LIKE 'Com_stmt_prepare';
```

A healthy system has `Com_stmt_execute` >> `Com_stmt_prepare` — many executes per prepare. If they're close to equal, you're not reusing prepared statements and you're paying the prepare cost for nothing.

---

## Summary

|Concept|What it means|
|---|---|
|**Driver**|Library in your app that speaks the DB wire protocol and holds the local prepare cache|
|**No check round trip**|Driver knows what's prepared from its own memory — no network lookup|
|**Plan caching**|Server skips parse/analyze/plan on every execute after the first|
|**Binary protocol**|Params sent as typed bytes, not SQL text — smaller and faster|
|**Custom vs generic plan**|PostgreSQL re-plans for real values 5 times, then decides if a generic cached plan is good enough|
|**Session scope**|Prepared handles live on one connection — connection poolers can break this|
|**Batch execution**|N values in one network write instead of N round trips|

The core idea: **pay once to prepare, collect savings on every execute**. The higher your execute-to-prepare ratio, the more you gain.

---

## Related

- [[Explain Analyze]]
- [[pg_stat_statements]]
- [[Isolation Level and MVCC]]

← [[Concepts MOC]]
