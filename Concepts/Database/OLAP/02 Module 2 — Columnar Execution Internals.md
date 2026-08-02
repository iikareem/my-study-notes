# Module 2 — Columnar Execution Internals

[[01 Module 1 — OLTP vs OLAP Mental Model|← Module 1]] · [[00 OLAP MOC|↑ Table of Contents]] · [[03 Volcano vs Vectorized Execution Models|Volcano comparison]] · [[04 Module 3 — Parquet & File Formats|Next: Module 3 →]]

---

## Module 2 Summary

> **One-liner:** Columnar engines win by processing batches of typed arrays, skipping data early, and compressing same-type columns — not by “smarter SQL.”

**Most important takeaways:**

1. **Vectorized vs volcano** — Postgres passes one tuple per `next()` call; columnar engines pass batches (~1024 values) as plain arrays → fewer calls + SIMD + better cache use.
2. **Late materialization** — Filter on one column at a time using row positions; only fetch output columns for rows that survive. Selective WHERE = bigger win.
3. **Predicate pushdown (3 layers)** — (1) skip whole partitions/files → (2) skip row groups via min/max stats → (3) filter values while scanning. Prefer plain comparisons over functions in WHERE (`UPPER(col)` blocks stats pushdown).
4. **Column compression** — Dictionary, RLE, and delta work because a column is one type with repeating patterns; sort order at write time makes RLE/stats much better.
5. **DuckDB EXPLAIN focus** — Projected columns at scan, filters inside `PARQUET_SCAN` (good) vs separate `FILTER` above it (weaker), and row groups scanned vs skipped.

**Jump to:** [[#2.1 Vectorized execution vs volcano/tuple-at-a-time|2.1]] · [[#2.2 Late materialization|2.2]] · [[#2.3 Predicate pushdown|2.3]] · [[#2.4 Compression that works because of columns|2.4]] · [[#2.5 How to read an analytical EXPLAIN differently from Postgres EXPLAIN|2.5]] · [[#2.7 Postgres → OLAP Translation Table|2.7 Translation]]

---

## 2.1 Vectorized execution vs volcano/tuple-at-a-time

To understand why columnar engines are fast, you need to understand how Postgres processes queries — and then see how columnar engines do it differently.

### The Postgres execution model: volcano (tuple-at-a-time)

Postgres uses an execution model called the **volcano model** (also called the iterator model). Every operator in the query plan — Seq Scan, Filter, Hash Join, Aggregate — implements a `next()` function. The top-level operator calls `next()` on its child, which calls `next()` on its child, and so on down the tree. Each call returns **one tuple**.

```
Aggregate.next()
  └─► Hash Join.next()
        ├─► Index Scan.next() → returns row 1
        └─► Seq Scan.next()   → returns row 1
  └─► Hash Join.next()
        ├─► Index Scan.next() → returns row 2
        └─► Seq Scan.next()   → returns row 2
  ... (once per row, millions of times)
```

This model has a well-known overhead: **one function call per row per operator**. If you have 10 million rows and 5 operators in the plan, that's 50 million function calls just for the coordination overhead — before doing any actual computation. Modern CPUs execute billions of instructions per second, but poorly-predicted function calls and pointer chasing are expensive relative to arithmetic.

Additionally, the volcano model passes **one row at a time** between operators. That row is a generic tuple — a C struct with a type tag, null flag, and a value pointer. The CPU has no opportunity to use SIMD (single instruction, multiple data) instructions, because the data isn't laid out as a plain array.

### The columnar execution model: vectorized

Columnar engines use **vectorized execution**. Instead of passing one tuple at a time, each `next()` call returns a **batch of values** — a vector — typically 1,024 or 2,048 values at once, stored as a typed, contiguous array in memory.

```
Aggregate.next_batch()           → receives 1024 floats
  └─► Filter.next_batch()        → produces 1024 filtered floats
        └─► Column Scan.next_batch() → reads 1024 floats from disk
```

The same 10M row scan now involves ~10,000 batch calls instead of 10 million single-row calls. But more importantly:

- Each batch is a **plain typed array** — `float64[1024]`, `int32[1024]`. The CPU can use SIMD to process 4–16 values per clock cycle.
- The filter `revenue > 100` applied to 1,024 floats at once is a tight loop the compiler can auto-vectorize.
- Data stays in L1/L2 cache across the batch (1,024 × 8 bytes = 8 KB — fits in L1 cache on most CPUs).

**The result:** Columnar vectorized engines can process hundreds of millions of rows per second per core. Postgres, on its best day with a sequential scan, processes tens of millions of rows per second per core.

**Analogy:** The volcano model is a factory assembly line where each worker hands one part to the next worker, one at a time. Vectorized execution is the same assembly line, but each worker passes a crate of 1,024 parts at once. The number of handoffs drops by 1,000x.

### What this means for you as a developer

You don't configure vectorized execution — it's automatic in DuckDB and other columnar engines. But you should understand it because:

1. **Explain plans look different.** You won't see `Hash Join (cost=... rows=1 width=8)` with per-row costs. You'll see batch-oriented operators.
2. **Query tuning intuition changes.** In Postgres, reducing rows early (with selective WHERE clauses) is critical because every row costs one function call per operator. In columnar engines, the primary lever is **column pruning**(reduce columns) and **partition pruning** (reduce files read), not just row filtering.
3. **Small tables don't benefit.** Vectorized execution pays off at millions of rows. For small result sets, the overhead of batch coordination exists without the payoff. DuckDB still works on small tables, but don't expect 10x speedups vs Postgres on a 10,000-row table.

---

## 2.2 Late materialization

**Materialization** means assembling the full tuple — all columns needed — for processing. In Postgres, this happens early: when a row is fetched from the heap, all its columns come together immediately.

In columnar execution, **late materialization** means: work with column IDs (row positions) as long as possible, and only fetch the actual column values at the last possible moment.

### How it works

Suppose your query is:

```sql
SELECT order_id, revenue
FROM orders
WHERE category = 'Electronics' AND revenue > 500
```

**Early materialization (Postgres style):**

1. Read full row 1: `[order_id=1, customer='Alice', category='Books', revenue=79, ...]`
2. Check `category = 'Electronics'` → false → discard
3. Read full row 2: `[order_id=2, customer='Bob', category='Electronics', revenue=29, ...]`
4. Check `category = 'Electronics'` → true. Check `revenue > 500` → false → discard
5. ... repeat for all rows

You loaded all columns of every row, including ones you discarded.

**Late materialization (columnar style):**

1. Scan only the `category` column. Build a list of row positions where `category = 'Electronics'` → positions `{2, 7, 11, 15, ...}`(2% of rows)
2. Scan only the `revenue` column at those positions. Filter `revenue > 500` → positions `{7, 15, ...}` (0.1% of rows)
3. Now — and only now — fetch `order_id` and `revenue` values for positions `{7, 15, ...}`.

**What this saves:** You only materialize the output columns for the tiny fraction of rows that survive all filters. All the intermediate filtering happened on one column at a time, never loading the other columns at all.

**The rule of thumb:** The more selective your WHERE clause, the bigger the win from late materialization. A filter that passes 0.1% of rows means you only fully materialize 0.1% of rows. A filter that passes 99% of rows gives almost no benefit — you end up materializing everything anyway.

---

## 2.3 Predicate pushdown

**Predicate** is the formal term for a filter condition — the expression in your WHERE clause.

**Predicate pushdown** means moving the filter as early (as deep) in the execution plan as possible, so that it eliminates data before expensive operations happen. This exists in Postgres too, but in columnar systems it extends to the file format itself.

### Pushdown layers in a columnar system

There are three places a predicate can be applied, from cheapest to most expensive:

**Layer 1 — Partition pruning (file level)**

Before reading any file, the engine looks at your WHERE clause and eliminates entire files or directories.

```sql
SELECT * FROM read_parquet('data/events/date=*/part.parquet', hive_partitioning=true)
WHERE date = '2024-01-15'
```

The engine only opens the `date=2024-01-15/` directory. Every other date's files are never touched — not even opened. This is the cheapest skip possible.

**Layer 2 — Row group statistics (page level)**

Inside a Parquet file, data is divided into row groups (explained in Module 3). Each row group has metadata: min value, max value, null count for each column. The engine checks your predicate against these statistics before reading the row group's data.

```
Row group 1: revenue min=5.00, max=89.00
Row group 2: revenue min=91.00, max=450.00
Row group 3: revenue min=12.00, max=1200.00

Query: WHERE revenue > 500
→ Skip row group 1 (max=89, cannot contain revenue > 500)
→ Skip row group 2 (max=450, cannot contain revenue > 500)
→ Read row group 3 (max=1200, may contain revenue > 500)
```

**Layer 3 — Value-level filtering (during column scan)**

Within a row group, the engine reads the column values and applies the filter. This is the most expensive layer — actual data is being read and compared — but late materialization ensures you only load other columns for rows that pass.

### Why this matters for your SQL

Predicates that use columns in ways the engine can't push down are expensive. Examples:

```sql
-- Good: DuckDB can push this to row group statistics
WHERE revenue > 500

-- Bad: function call prevents statistics-level pushdown
WHERE UPPER(category) = 'ELECTRONICS'
-- DuckDB has to read and transform all category values before filtering

-- Good: literal value comparison
WHERE event_date = '2024-01-15'

-- Bad: non-deterministic; can't push down
WHERE event_date = CURRENT_DATE - INTERVAL '30 days'
-- (DuckDB is actually getting better at evaluating these at plan time,
--  but the general principle holds for complex expressions)
```

---

## 2.4 Compression that works because of columns

Compression in row stores is limited because each row contains a mix of types and values: an integer next to a string next to a timestamp. Mixed data compresses poorly — it lacks patterns.

Columnar storage changes everything. A column of `status` values might look like:

```
['paid', 'paid', 'paid', 'pending', 'paid', 'paid', 'failed', 'paid', ...]
```

That's highly repetitive. Three compression schemes dominate columnar storage:

### Dictionary encoding

Replace repeated string values with small integer codes. Maintain a dictionary mapping codes to values.

```
Original column:
['Electronics', 'Books', 'Electronics', 'Electronics', 'Clothing', 'Books', ...]

Dictionary:  { 0: 'Electronics', 1: 'Books', 2: 'Clothing' }
Encoded:     [0, 1, 0, 0, 2, 1, ...]
```

A 12-byte string `'Electronics'` becomes a 1-byte integer. If `'Electronics'` appears 10 million times, you store the string once and 10 million 1-byte codes instead of 10 million 12-byte strings. **Typical savings: 5–20x.**

**Bonus:** Filtering `WHERE category = 'Electronics'` becomes integer comparison on the codes — cheaper than string comparison, and the dictionary lookup happens once.

### Run-Length Encoding (RLE)

Instead of storing repeated values, store the value once and how many times it repeats.

```
Original:  [1, 1, 1, 1, 1, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3]
RLE:       [(1, 5), (2, 3), (3, 7)]   — (value, count) pairs
```

Excellent when data has long runs of the same value. Common examples:

- A `status` column where records are sorted by status before writing
- A `date` column in a time-sorted dataset (same date repeated thousands of times)
- A boolean or enum column

**Typical savings: 10–100x for very repetitive columns.**

### Delta encoding

Instead of storing absolute values, store the difference between consecutive values.

```
Original timestamps: [1700000000, 1700000060, 1700000120, 1700000180, ...]
Deltas:              [1700000000, 60, 60, 60, ...]
```

Regular time series data often has uniform or near-uniform differences. `60` takes far fewer bits than `1700000060`. Parquet then applies bit-packing on top — if all your deltas are under 256, they each need only 8 bits instead of 64.

**Typical savings: 2–10x for monotonically increasing numeric columns.**

### Why this matters to you

1. **Parquet files are typically 5–10x smaller than the equivalent CSV**, and often 3–5x smaller than a Postgres dump. This reduces storage cost and — crucially — I/O time when reading.
    
2. **Some operations happen on compressed data.** Dictionary-encoded equality comparisons (`WHERE category = 'Electronics'`) work on the integer codes, never decompressing the strings. This is called operating on **encoded data**.
    
3. **Column sort order matters for compression.** If you sort a Parquet file by `category` before writing, all the `'Electronics'`rows cluster together, making RLE much more effective. This is one reason data layout decisions (Module 4) affect performance so much.
    

---

## 2.5 How to read an analytical EXPLAIN differently from Postgres EXPLAIN

Postgres `EXPLAIN` output is something you know. DuckDB's `EXPLAIN` output looks structurally similar but has different node names, different semantics, and different things to look for.

### Postgres EXPLAIN — what you're used to

```sql
-- Postgres
EXPLAIN (ANALYZE, BUFFERS) SELECT SUM(revenue) FROM orders WHERE category = 'Electronics';

Aggregate  (cost=180000.00..180000.01 rows=1 width=8)
           (actual time=3420.123..3420.123 rows=1 loops=1)
  ->  Seq Scan on orders  (cost=0.00..160000.00 rows=8000000 width=8)
                          (actual time=0.012..2800.456 rows=8000000 loops=1)
        Filter: ((category)::text = 'Electronics'::text)
        Rows Removed by Filter: 42000000
  Buffers: shared hit=123456 read=876544
```

Key things you read:

- Node type (`Seq Scan`, `Index Scan`, `Hash Join`)
- `cost=` estimates (startup..total)
- `actual time=`, `rows=`, `loops=`
- `Buffers:` (cache hits vs disk reads)
- `Filter:` and `Rows Removed by Filter:`

### DuckDB EXPLAIN

```sql
-- DuckDB
EXPLAIN SELECT SUM(revenue) FROM read_parquet('orders.parquet') WHERE category = 'Electronics';

┌─────────────────────────────┐
│      UNGROUPED_AGGREGATE    │
│   ~────────────────────~   │
│         sum(revenue)        │
└──────────────┬──────────────┘
               │
┌──────────────▼──────────────┐
│         PARQUET_SCAN        │
│   orders.parquet            │
│   Columns: category, revenue│
│   Filters: category='Elec..'│
└─────────────────────────────┘
```

### What the DuckDB node types mean

|DuckDB node|Postgres analog|What it does|
|---|---|---|
|`PARQUET_SCAN`|`Seq Scan`|Reads column chunks from Parquet files|
|`TABLE_SCAN`|`Seq Scan`|Reads a DuckDB in-memory or persistent table|
|`FILTER`|`Filter` (in Seq Scan)|Applies a predicate to a batch of values|
|`UNGROUPED_AGGREGATE`|`Aggregate`|Aggregates without a GROUP BY|
|`HASH_GROUP_BY`|`HashAggregate`|GROUP BY using a hash table|
|`HASH_JOIN`|`Hash Join`|Joins using a hash table|
|`PROJECTION`|(implicit in Postgres)|Selects/computes output columns|
|`STREAMING_LIMIT`|`Limit`|Limits rows in a streaming fashion|

### What to look for in DuckDB EXPLAIN

**1. Which columns are projected at the scan?**

```
PARQUET_SCAN
  Columns: category, revenue
```

If you see many columns in the scan but only need a few, your SQL may be forcing unnecessary column reads (e.g., an intermediate CTE that does `SELECT *`).

**2. Are filters pushed down to the scan?**

```
PARQUET_SCAN
  Filters: category='Electronics'
```

A filter shown at the `PARQUET_SCAN` node means it's applied during scanning — before data enters the execution engine. This is good. A `FILTER` node above the scan means the filter happens after data is already loaded.

**3. Use EXPLAIN ANALYZE for actual execution stats:**

```sql
EXPLAIN ANALYZE SELECT SUM(revenue) FROM read_parquet('orders.parquet') WHERE category = 'Electronics';
```

This shows:

- Actual rows processed per node
- Time spent per node
- How many row groups were scanned vs skipped

**4. Look for "Row Groups Scanned" vs "Row Groups Total":**

DuckDB's EXPLAIN ANALYZE (and the verbose output) will tell you how many row groups it had to read vs how many existed. If you have 100 row groups and only 3 are scanned, your predicate pushdown is working well. If all 100 are scanned, the engine can't skip any — either the data isn't sorted helpfully, or the predicate isn't pushdown-friendly.

---

---

## 2.7 Postgres → OLAP Translation Table

|Postgres concept|OLAP / DuckDB equivalent|Key difference|
|---|---|---|
|Volcano model (one tuple at a time)|Vectorized execution (batches of 1024+)|Fewer function calls, SIMD-friendly arrays|
|`Seq Scan`|`PARQUET_SCAN` / `TABLE_SCAN`|Columnar scan reads only projected columns|
|`Filter: (condition)` inside Seq Scan|Filter pushed into `PARQUET_SCAN`node|Columnar filter can use row group statistics to skip blocks|
|`Buffers: hit=N read=M`|Row groups scanned vs skipped|Both tell you about I/O, different granularity|
|`cost=startup..total`|Not shown by default in DuckDB|DuckDB uses cardinality estimates internally|
|`Index Scan` (seek)|Partition pruning + row group skip|Same goal (skip data), different mechanism|
|`EXPLAIN (ANALYZE, BUFFERS)`|`EXPLAIN ANALYZE`|DuckDB shows per-node timing and row counts|
|`loops=N`|Not present (no nested-loop style iteration)|Vectorized plans don't loop per-row|
|Tuple width (bytes per row)|Column size (bytes per column value)|Different unit: per column, not per row|

---

[[#Module 2 Summary|↑ Module 2 Summary]] · [[00 OLAP MOC|↑ TOC]] · [[04 Module 3 — Parquet & File Formats|Next: Module 3 →]]

---

