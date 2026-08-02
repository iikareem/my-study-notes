[[00 OLAP MOC|↑ Table of Contents]] · [[02 Module 2 — Columnar Execution Internals|← Module 2]] · [[04 Module 3 — Parquet & File Formats|Next: Module 3 →]]

# Volcano vs Vectorized Execution Models

> For PostgreSQL users who know `EXPLAIN ANALYZE` but haven't studied query engine internals.

---

## 1 — What Is a Query Execution Model?

When you run a SQL query, PostgreSQL breaks it into a tree of **operators** — each one doing a single job. You see this tree in `EXPLAIN ANALYZE`:

```
Sort
  └─ HashAggregate
       └─ Hash Join
            ├─ Seq Scan on orders
            └─ Hash
                 └─ Seq Scan on customers
```

### The tree is built top-down, but data flows bottom-up

The **planner** builds this tree before any data moves. It decides which operators to use and how to connect them — purely as a plan, no execution yet.

Then the **executor** starts at the top node and begins pulling. The request travels down the tree until it hits a leaf node — always a scan (`Seq Scan`, `Index Scan`) — because that is the only operator that actually touches real data. The data then travels back up, passing through each operator on the way.

```
Planner builds the tree (no data yet, top-down)
          ↓
Executor starts at the TOP and pulls downward
          ↓
Request reaches the leaf (Seq Scan) — hits disk/buffer cache
          ↑
Data propagates back UP through each operator
```

So the correct mental model: **the request travels down, the data travels up.** Each operator in between just receives, transforms, and passes upward. That pull chain is the entire foundation of how PostgreSQL executes queries.

The **execution model** answers one question: _how much data travels up at a time?_

There are two answers: Volcano (one row) and Vectorized (a batch of rows).

---

## 2 — The Volcano Model (what PostgreSQL uses)

### The idea

Every operator has one method: `next()`. When an operator needs data, it calls `next()` on its child. The child calls `next()` on its child, and so on down the tree. One row travels back up. Then the cycle repeats.

This is called a **pull model** — the top operator pulls rows upward, one at a time.

### How it works — step by step

Take this query:

```sql
SELECT customer_id, SUM(amount)
FROM orders
WHERE amount > 100
GROUP BY customer_id;
```

Plan:

```
Aggregate
  └─ Filter (amount > 100)
       └─ Seq Scan on orders
```

At runtime:

1. `Aggregate` calls `Filter.next()`
2. `Filter` calls `SeqScan.next()`
3. `SeqScan` reads row #1 from disk (via the buffer cache — explained in Part 5) → returns it to `Filter`
4. `Filter` checks: `amount > 100`? Yes → passes row up to `Aggregate`
5. `Aggregate` adds it to its SUM bucket
6. `Aggregate` calls `Filter.next()` again — the whole cycle repeats
7. When `SeqScan` has no more rows, it returns `NULL` → `Aggregate` emits the final result

One row. One full trip up the tree. Then again. For every row in the table.

### Why PostgreSQL uses it

- Each operator is self-contained — easy to implement and extend
- Very low memory — only one row lives between operators at a time
- Works perfectly for OLTP: you rarely touch more than a few thousand rows

### Why it struggles at scale

**Too many function calls.** A 100M-row table with a 5-level plan means 500 million `next()` calls just for iteration. Each call has overhead: stack setup, register saves, branch prediction misses.

**CPU cache goes cold.** Processing one row at a time means jumping between operators constantly. The CPU cache never gets a chance to stay warm with useful data.

**SIMD is impossible.** Modern CPUs can compare 8–16 integers in a single clock cycle (SIMD). But you can only exploit that when you have an array of values — not one row at a time.

Volcano is fast for OLTP. For analytics on billions of rows, it becomes the bottleneck.

---

## 3 — The Vectorized Model (what DuckDB, Snowflake, ClickHouse use)

### The idea

Same operator tree. Same `next()` concept. But instead of returning one row, each operator returns a **batch** — typically 1,024 rows packed together in columnar format.

### What a batch looks like

Volcano passes one row (all columns together):

```
{ order_id: 1001, customer_id: 42, amount: 99.50 }
```

Vectorized passes a batch (each column as its own array):

```
order_id:     [1001, 1002, 1003, ...]   ← 1024 values
customer_id:  [  42,   42,   17, ...]   ← 1024 values
amount:       [99.5, 150.0, 200.0, ...]  ← 1024 values
```

Each column is a contiguous block in memory. That matters because the CPU can load the entire `amount` array into its cache and process all 1,024 values without a single cache miss.

### Same query, different flow

```
Aggregate
  └─ Filter (amount > 100)
       └─ Seq Scan on orders
```

1. `SeqScan` reads 1,024 rows → returns them as a batch
2. `Filter` runs `amount > 100` across all 1,024 amounts at once using SIMD → returns only passing rows (say 612)
3. `Aggregate` accumulates all 612 rows into SUM buckets in one tight loop
4. Repeat: next batch of 1,024 rows

### Why it is faster

100M rows ÷ 1,024 = ~97,000 `next_batch()` calls instead of 100 million `next()` calls — a 1,000× reduction in function call overhead. The tight column loops are predictable, cache-friendly, and SIMD-eligible.

The tradeoff: more memory per operator (one full batch at a time), and more implementation complexity. For analytics, the performance gain is worth it by a large margin.

---

## 4 — Streaming vs Blocking Operators

This concept applies to both models. It answers: **can an operator return its first row immediately, or must it consume ALL input before returning anything?**

### Streaming operators

A streaming operator emits rows as they arrive — it does not wait for the full input. The pipeline keeps flowing.

|Operator|Why it streams|
|---|---|
|Seq Scan|Reads one page, emits rows, reads next page|
|Index Scan|Finds one key, emits row, finds next key|
|Filter (WHERE)|Checks each row inline, passes it immediately|
|Nested Loop Join (probe side)|Emits a match the moment it finds one|
|Limit|Cuts the pipeline after N rows|
|Append|Passes rows from each child in sequence|

Because of streaming, a `Seq Scan → Filter → Limit 10` pipeline returns the first row almost instantly — even on a 100M-row table.

### Blocking operators (pipeline breakers)

A blocking operator must consume its **entire input** before it can emit even one output row. It breaks the pipeline — nothing downstream can start until it finishes.

|Operator|Why it blocks|
|---|---|
|Sort|Cannot return row #1 without seeing all rows first|
|Hash (build side of Hash Join)|Must load the full inner table into a hash table|
|HashAggregate|Must accumulate all rows before finalizing any group|
|Materialize|Buffers everything into memory (or disk)|

This is why `ORDER BY` on a large table is expensive even if you only want the first 10 results — the Sort node still reads and sorts everything before `LIMIT` can do anything.

> **One exception:** PostgreSQL may use `top-N heapsort` when it detects a `SORT + LIMIT` pattern. You'll see it in `EXPLAIN` as `Sort Method: top-N heapsort` — it avoids sorting the entire input.

### When blocking operators run out of memory

Each blocking operator gets a slice of `work_mem`. If the data exceeds that, PostgreSQL spills to disk — a significant performance hit. You see it in `EXPLAIN ANALYZE` as:

```
Sort Method: external merge  Disk: 348216kB
HashAggregate  Batches: 8  Memory Usage: 4096kB
```

`Batches: 1` means it fit in memory. `Batches: 8` means it had to make 8 passes over the data writing to disk. Fix: `SET work_mem = '256MB'` before the query.

### Blocking in vectorized engines

Blocking operators still exist in vectorized engines — Sort still needs all rows before it can sort. The difference is that the accumulation phase is faster because each batch is processed with SIMD. The wall still exists; the work behind it is just cheaper.

---

## 5 — How PostgreSQL Fetches Data: Disk vs Buffer Cache

Before a `Seq Scan` or `Index Scan` can hand a row to the next operator, it needs to get that row from somewhere. The answer is never "directly from disk" — it always goes through the **shared buffer cache** first.

### The shared buffer cache

PostgreSQL stores data on disk in **8KB pages** (also called blocks). Every table and index is a sequence of these pages.

Between disk and your query sits a memory pool called the **shared buffer cache** — configured by `shared_buffers` in `postgresql.conf` (default 128MB, typically set to 25% of RAM in production). It holds recently accessed pages in RAM.

When an operator needs a page:

```
Does the page exist in shared buffer cache?
        │
        ├── YES (cache hit)  → read rows from RAM — nanoseconds
        │
        └── NO  (cache miss) → ask the OS
                    │
                    ├── OS page cache has it → read from kernel RAM — microseconds
                    │
                    └── Not cached → read from disk (SSD ~0.1ms, HDD ~10ms)
                        → page is loaded into shared buffer cache for next time
```

The key point: **PostgreSQL fetches whole pages, not individual rows.** Once a page is in the buffer cache, all rows on that page are available at RAM speed.

### Cold cache vs warm cache

**First run (cold cache):** The buffer cache is empty. Every page must be fetched from disk or the OS. This is slow — you'll see high `actual time` values.

**Second run (warm cache):** Pages from the first run are still in the buffer cache. The same query reads everything from RAM. Same rows, same plan — but dramatically faster. This is why benchmarking a query only once is misleading.

**Eviction:** PostgreSQL uses a clock-sweep algorithm (similar to LRU) to decide which pages to evict when the cache fills up. If `shared_buffers` is too small to hold your working set, pages get evicted and re-fetched constantly — called **cache thrashing**.

### Seeing it in EXPLAIN ANALYZE

Add `BUFFERS` to expose exactly which layer served each page:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE amount > 100;
```

Cold cache:

```
Seq Scan on orders  (actual time=45.221..312.883 rows=4701 loops=1)
  Buffers: shared hit=0 read=1600
```

Warm cache (same query, run again):

```
Seq Scan on orders  (actual time=0.022..18.441 rows=4701 loops=1)
  Buffers: shared hit=1600 read=0
```

Same 4,701 rows. 17× faster. The only difference is cache state.

### Buffer fields reference

|Field|Meaning|
|---|---|
|`shared hit=N`|Pages read from shared buffer cache (RAM) — no disk involved|
|`shared read=N`|Pages not in cache — fetched from OS or disk|
|`shared dirtied=N`|Pages modified during this query (writes)|
|`temp written / read`|Disk spill from Sort or HashAggregate|

**`shared read` is high** → query is I/O-bound. The table is not cached. Add an index to avoid full scans, or increase `shared_buffers`.

**All `shared hit`** → query is CPU-bound. Data is in RAM. Focus on the plan itself — join types, row reduction.

**`temp written` is high** → a blocking operator spilled to disk. Increase `work_mem`.

---

## 6 — Reading EXPLAIN ANALYZE

Now everything connects. Each node in an `EXPLAIN ANALYZE` plan is a Volcano operator. The output tells you how each operator behaved at runtime.

### Anatomy of a node

```
Hash Join  (cost=120.00..980.50 rows=4823 width=32)
           (actual time=14.231..38.042 rows=4701 loops=1)
```

|Field|Meaning|
|---|---|
|`cost=120.00..980.50`|Planner estimate. Start cost = before first row emitted. Total = after last row.|
|`rows=4823`|Planner's estimate of rows this operator will emit.|
|`actual time=14.231..38.042`|`14.231ms` = when first row was emitted. `38.042ms` = when last row was emitted (operator finished).|
|`actual rows=4701`|Real number of rows emitted by this operator.|
|`loops=N`|How many times this operator was re-executed from scratch.|

**The start time tells you whether the operator is streaming or blocking:**

- Small start time → streaming (first row came quickly)
- Start time ≈ end time → blocking (operator accumulated everything before emitting anything)

### A full annotated example

```sql
EXPLAIN ANALYZE
SELECT c.name, SUM(o.amount)
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.amount > 100
GROUP BY c.name
ORDER BY SUM(o.amount) DESC;
```

```
Sort  (actual time=85.231..85.419 rows=1000 loops=1)
  Sort Key: (sum(o.amount)) DESC
  Sort Method: quicksort  Memory: 102kB
  -> HashAggregate  (actual time=78.100..80.212 rows=1000 loops=1)
       Group Key: c.name
       Batches: 1  Memory Usage: 245kB
       -> Hash Join  (actual time=0.812..55.331 rows=4701 loops=1)
            Hash Cond: (o.customer_id = c.id)
            -> Seq Scan on orders o  (actual time=0.022..28.441 rows=4701 loops=1)
                 Filter: (amount > 100)
                 Rows Removed by Filter: 5299
            -> Hash  (actual time=0.741..0.741 rows=600 loops=1)
                 Buckets: 1024  Batches: 1  Memory Usage: 38kB
                 -> Seq Scan on customers c  (actual time=0.009..0.301 rows=600 loops=1)
```

**`Seq Scan on customers`** — streaming. Reads 600 rows in 0.301ms. Done.

**`Hash` (build)** — blocking. Reads all 600 rows and builds a hash table in memory (38kB). Notice `actual time=0.741..0.741` — start equals end. Nothing was emitted until the hash table was fully built. Textbook blocking behavior.

**`Seq Scan on orders`** — streaming. Reads the full table. Filter runs inline — no separate node. 10,000 rows read, 5,299 dropped, 4,701 passed up. This is the Volcano pipeline: each `next()` on Filter loops internally over multiple `next()` calls on SeqScan until it finds a row that passes.

**`Hash Join`** — probe side is streaming. For each row from orders, it probes the hash table and emits a match immediately. Started at 0.812ms (right after the hash table was ready), finished at 55.331ms.

**`HashAggregate`** — blocking. Must consume all 4,701 rows before emitting any group result. It spent 78ms silently accumulating. Only the final 2ms produced output. `Batches: 1` = fit in `work_mem`, no spill.

**`Sort`** — blocking. Gets all 1,000 groups and sorts them. `Memory: 102kB` = fit in memory. `actual time=85.231..85.419` — the 85ms start time is nearly all wait time for HashAggregate to finish. Only 0.2ms of actual sorting.

### The `loops` trap

`loops` on an inner node of a Nested Loop join means the operator was re-run once per outer row. The `actual rows` shown is a **per-loop average**, not the total.

```
Nested Loop  (actual rows=3120 loops=1)
  -> Seq Scan on customers  (actual rows=10 loops=1)
  -> Index Scan on orders   (actual rows=312 loops=10)
```

The index scan ran 10 times (once per matching customer). It returned 312 rows on average each time. Real total: **312 × 10 = 3,120 rows**. Always multiply `actual rows × loops`.

### Spotting planner mistakes

Compare `rows` (estimate) to `actual rows` (reality):

```
Seq Scan on orders  (cost=... rows=500 ...)      ← planner expected 500
                    (actual ... rows=50000 ...)   ← reality: 50,000
```

A 100× underestimate causes the planner to choose wrong join strategies. Fix: run `ANALYZE orders;` to refresh statistics. If the problem persists, the columns may be correlated — use `CREATE STATISTICS` to teach the planner about that correlation.

### How to read any plan — in order

1. **Find the blocking operators** (`Sort`, `Hash` build, `HashAggregate`, `Materialize`). Check if they spilled (`external merge`, `Batches > 1`).
2. **Check `loops`** on any inner node. Multiply `rows × loops` for real volume.
3. **Find where estimates diverge** from actuals — that is where the planner went wrong.
4. **Check `Buffers`** (if enabled). High `read` = I/O-bound. All `hit` = CPU-bound.
5. **Total query time** = the top node's end time. Work down to find which operator owns the most of it.

---

## 7 — Comparison at a Glance

||Volcano|Vectorized|
|---|---|---|
|Unit moved between operators|1 row|~1,024 rows (a batch)|
|Data layout inside a batch|Row-oriented|Column-oriented|
|Function call overhead|1 call per row per operator|1 call per batch per operator|
|SIMD utilization|None|High|
|CPU cache efficiency|Low|High|
|Memory per operator|Minimal|One batch (~tens of KB)|
|Best for|OLTP — small row counts|OLAP — millions/billions of rows|
|Used by|PostgreSQL, MySQL|DuckDB, Snowflake, ClickHouse|

Blocking operators exist in both models. The "wall" is the same — vectorized just builds it faster.

---

## 8 — Does PostgreSQL Have Any Vectorization?

PostgreSQL is Volcano-based, but has two partial optimizations worth knowing:

**JIT compilation** (`SET jit = on`) — PostgreSQL compiles hot parts of a query to native machine code at runtime. It reduces interpreter overhead but is not true vectorization. Helps mostly on CPU-heavy expressions.

**Parallel query** — multiple worker processes each run their own Volcano pipeline over a portion of the data. Results are merged through a `Gather` node at the top. Still one-row-at-a-time per worker, but parallelism helps.

For serious analytics workloads alongside PostgreSQL, **DuckDB** is commonly used. It is fully vectorized and can read PostgreSQL tables directly via its `postgres` extension.

---

## 9 — Why Vectorized Engines Are Better for Analytics and the CPU

This is worth understanding deeply, because the performance gap between Volcano and Vectorized is not small — it is often 10x to 100x on analytical queries. The reason comes down to how modern CPUs actually work.

### The CPU is not waiting for your logic — it is waiting for your data

A modern CPU can execute billions of instructions per second. The bottleneck in most database workloads is not the computation — it is **getting data to the CPU fast enough**. The CPU has a hierarchy of memory, each level faster but smaller than the one below:

```
L1 cache   ~  32KB  — 1 ns   (fastest, closest to the CPU core)
L2 cache   ~ 256KB  — 4 ns
L3 cache   ~  16MB  — 20 ns
RAM        ~  32GB  — 80 ns
SSD        ~   1TB  — 100,000 ns  (100x slower than RAM)
```

Every time the CPU needs data that is not in L1 or L2, it stalls and waits. This is called a **cache miss**. In Volcano, processing one row at a time means the CPU constantly jumps between different operators, different structs, different memory locations — the cache goes cold on every jump. The CPU spends most of its time stalling, not computing.

In Vectorized, an operator processes 1,024 values of the same column in a tight loop. Those 1,024 values are contiguous in memory — they load into L1/L2 cache together and stay there for the entire loop. The CPU stalls once at the start, then runs at full speed for the rest of the batch.

### SIMD — one instruction, many values

Modern CPUs have a set of special instructions called **SIMD** (Single Instruction, Multiple Data). Instead of comparing two integers, a single SIMD instruction can compare 8, 16, or even 32 integers simultaneously in one clock cycle.

For example, filtering `amount > 100` across 1,024 rows:

- **Volcano:** compare one `amount` value, call `next()`, compare the next, call `next()` — 1,024 individual comparisons with 1,024 function calls
- **Vectorized:** the 1,024 `amount` values are a contiguous float array → one SIMD loop, ~64 instructions total

The vectorized version does the same logical work in a fraction of the CPU cycles.

### Why this matters specifically for analytics

OLTP queries (web requests, single-row lookups, small inserts) typically touch tens to hundreds of rows. The Volcano overhead is invisible — the query finishes in microseconds regardless.

Analytical queries aggregate millions or billions of rows — things like `GROUP BY`, `SUM`, `COUNT DISTINCT`, window functions, large joins. These are exactly the workloads where:

- Every row passes through multiple operators (high function call cost in Volcano)
- The same column is scanned repeatedly (perfect for SIMD in Vectorized)
- CPU cache efficiency determines whether a query takes 2 seconds or 200 seconds

This is why DuckDB, Snowflake, ClickHouse, and BigQuery all converged on vectorized execution. It is not a new idea — the MonetDB/X100 paper proved it in 2005. But it took a decade for the industry to rebuild engines around it.

### The columnar storage connection

Vectorized execution and **columnar storage** (like Parquet, ORC) are designed for each other. In columnar storage, each column is stored as a contiguous block on disk — so reading a batch of 1,024 `amount` values means one sequential disk read, which lands directly into a format the vectorized engine can process with SIMD. No row unpacking needed.

Volcano's row-oriented processing pairs naturally with row-oriented storage (PostgreSQL's heap). Vectorized processing pairs naturally with columnar storage. This is why analytical databases almost always combine both.

### Summary

|What the CPU needs|Volcano gives it|Vectorized gives it|
|---|---|---|
|Data in L1/L2 cache|One row at a time — cache goes cold|1,024 contiguous values — cache stays hot|
|Predictable loops|Jumps between operators — unpredictable|Tight inner loops — fully predictable|
|SIMD-eligible arrays|Impossible — one value at a time|Natural — each column is already an array|
|Few function calls|1 per row per operator|1 per batch per operator|

The CPU in a vectorized engine is doing real work almost every cycle. The CPU in a Volcano engine spends most of its time waiting — for data, for branch predictions to resolve, for function call overhead to clear. That gap is why DuckDB can outperform PostgreSQL by 10–100x on the same analytical query over the same data.

---

## Further Reading

- [Volcano paper — Graefe, 1994](https://dl.acm.org/doi/10.1109/69.273032)
- [MonetDB/X100 — Boncz et al., 2005](https://www.cidrdb.org/cidr2005/papers/P19.pdf)
- [Why DuckDB — duckdb.org](https://duckdb.org/why_duckdb)
- PostgreSQL docs: `EXPLAIN`, `work_mem`, `shared_buffers`, JIT compilation