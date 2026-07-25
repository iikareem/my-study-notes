---
tags:
  - postgresql
  - query-planner
  - statistics
---

# PostgreSQL Extended Statistics: The Missing Half of Query Optimization

> Indexes make fast access _possible_. Extended statistics make the planner _choose them when it should._

---

When a query is slow, the first instinct is almost always the same: **add an index**.

Then come the familiar tools — B-tree vs GiST vs GIN, covering indexes, partial indexes, BRIN for huge tables, partitioning by date or tenant, rewriting joins. All valid. All useful.

What almost nobody talks about is the thing that decides whether any of those tools get used **correctly** in the first place: **statistics**.

PostgreSQL does not execute your mental model of the query. It executes a **cost estimate**. Before a single row is touched, the planner asks:

> How many rows will this filter return? How big is this join? Is an index scan cheaper than a sequential scan?

Those answers come from statistics. Get them wrong, and the planner picks the wrong plan — even if your indexes are perfect, your schema is clean, and your queries are well-written.

That gap is where **extended statistics** live.

---

## What PostgreSQL Already Tracks (And Where It Falls Short)

When you run `ANALYZE`, PostgreSQL collects per-column stats and stores them in the `pg_stats` view. Let's look at what it actually captures — and query it directly.

```sql
SELECT
  attname,
  n_distinct,
  correlation,
  most_common_vals,
  most_common_freqs,
  histogram_bounds
FROM pg_stats
WHERE tablename = 'orders'
  AND attname IN ('city', 'country', 'order_status');
```

### `n_distinct` — Estimated Distinct Values

How many unique values the planner expects in a column.

- Positive value (e.g. `120`) → absolute count of distinct values
- Negative value (e.g. `-0.8`) → fraction of total rows (80% are distinct)

```sql
SELECT attname, n_distinct
FROM pg_stats
WHERE tablename = 'orders';

--  attname      | n_distinct
-- --------------+-----------
--  city         |        85     ← 85 unique cities
--  country      |        12     ← 12 unique countries
--  order_status |    -0.0004    ← nearly all rows share a small set of values
--  customer_id  |  -0.92        ← 92% of rows have a unique customer_id
```

The planner uses `n_distinct` to estimate `GROUP BY` cardinality and hash table sizes. A bad estimate here leads to wrong memory grants and poor aggregation strategies.

---

### `correlation` — Physical vs Logical Ordering

A value between `-1.0` and `1.0` that measures how closely the physical order of rows matches the logical sort order of a column.

- `1.0` → perfectly sorted on disk (great for index scans)
- `0.0` → random order
- `-1.0` → reverse sorted

```sql
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'orders';

--  attname      | correlation
-- --------------+-------------
--  created_at   |    0.98      ← rows inserted in time order → index scans are cheap
--  customer_id  |    0.02      ← random distribution → index scan has high I/O cost
--  city         |   -0.11      ← effectively random
```

**Why it matters:** the planner uses correlation to decide between index scan vs sequential scan. A correlation near `1.0` means the heap pages you need are clustered together — index scan is efficient. A correlation near `0.0` means random I/O across many heap pages — sequential scan often wins even if the index is selective.

This is why an index on `created_at` for time-range queries is almost always used, while an index on `customer_id` might get ignored for larger result sets despite having the same selectivity.

---

### `most_common_vals` / `most_common_freqs` — Top N Values

The most frequently occurring values in the column, paired with their frequencies. Together they let the planner estimate `WHERE status = 'pending'` precisely rather than guessing.

```sql
SELECT
  attname,
  most_common_vals,
  most_common_freqs
FROM pg_stats
WHERE tablename = 'orders'
  AND attname = 'order_status';

--  attname      | most_common_vals                        | most_common_freqs
-- --------------+-----------------------------------------+---------------------
--  order_status | {completed,pending,processing,cancelled} | {0.70,0.20,0.07,0.03}
```

Reading this: `completed` appears in 70% of rows, `pending` in 20%, and so on.

**What changes the depth of this list:** the `statistics` target (default 100) controls how many values are stored. For skewed columns where a small set of values dominates, the defaults are usually fine. For high-cardinality columns with many meaningful values, increase it:

```sql
ALTER TABLE orders ALTER COLUMN city SET STATISTICS 500;
ANALYZE orders;
```

---

### `histogram_bounds` — Distribution for Range Estimates

For columns without a dominant MCV list (or for values not in the MCV), PostgreSQL falls back to a histogram. The bounds divide the data into equal-frequency buckets.

```sql
SELECT attname, histogram_bounds
FROM pg_stats
WHERE tablename = 'orders'
  AND attname = 'amount';

--  attname | histogram_bounds
-- ---------+------------------------------------------
--  amount  | {5.00,23.40,51.20,99.80,201.00,5000.00}
```

For a range filter like `WHERE amount BETWEEN 50 AND 100`, the planner looks up which buckets the range spans and estimates the fraction of rows covered. The more buckets, the more accurate the estimate — again controlled by the statistics target.

---

## The Fatal Assumption Hiding in Per-Column Stats

Here is the problem: all of the above is collected **per column in isolation**.

The fatal assumption baked into that model is:

> Columns are independent of each other.

Real data breaks this constantly:

- `city` and `country` — Paris is almost exclusively in France
- `zip_code` and `state` — one determines the other
- `order_status` and `priority` — cancelled orders are rarely high priority

When you filter on two correlated columns, the planner multiplies their selectivities as if they were independent:

```sql
-- city = 'Paris'   → planner estimates 10% of rows
-- country = 'France' → planner estimates 15% of rows
-- Combined estimate: 10% × 15% = 1.5%
-- Reality: nearly 100% overlap — almost every Paris is in France
```

The estimate can be off by **10×, 100×, or more**. And when that happens:

- An index that would be perfect gets ignored
- A nested loop runs on a result set 200× larger than expected
- Join order is wrong, filter pushdown is skipped, memory pressure spikes

The index was never the problem. The planner just never got accurate information to use it.

---

## Extended Statistics: Telling PostgreSQL What It Can't See

Extended statistics let you explicitly say:

> These columns are related — model them together.

Introduced in **PostgreSQL 10**, stored in `pg_statistic_ext` and `pg_statistic_ext_data`. They come in three types, each solving a different estimation problem.

---

### Type 1: `dependencies` — Functional Dependencies

Models how strongly knowing column A predicts column B.

**Best for:** correlated filters like `WHERE city = 'Paris' AND country = 'France'`.

```sql
CREATE STATISTICS orders_location_dep (dependencies)
ON city, country
FROM orders;

ANALYZE orders;
```

PostgreSQL computes a dependency degree from `0.0` to `1.0` and uses it to correct the independence assumption. A degree of `0.9` means "knowing `city` tells you 90% of what you need to know about `country`."

Common real-world cases:

- `zip_code` → `city` → `state`
- `product_category` + `brand`
- `age_group` + `insurance_tier`

---

### Type 2: `ndistinct` — Distinct Value Combinations

Estimates how many unique _combinations_ exist across multiple columns.

**Best for:** `GROUP BY city, country` — how many unique pairs should the planner expect?

```sql
CREATE STATISTICS orders_city_country_nd (ndistinct)
ON city, country
FROM orders;

ANALYZE orders;
```

Without this, the planner multiplies the individual `n_distinct` values. If `city` has 85 unique values and `country` has 12, it guesses up to `85 × 12 = 1020` combinations. The reality might be only 130, because not every city appears in every country.

A bad `ndistinct` estimate causes wrong memory grants for hash aggregates and wrong sort strategies.

---

### Type 3: `mcv` — Most Common Value Combinations (PostgreSQL 12+)

The most precise type. While per-column MCV tells you "`status = 'pending'` appears in 20% of rows," extended MCV tells you the actual frequency of **combinations**:

```sql
-- Per-column stats:
--   order_status = 'pending'   → 20%
--   order_type   = 'express'   → 15%
--
-- Independence assumption:
--   pending AND express → 20% × 15% = 3%
--
-- Reality (from extended MCV):
--   pending AND express → 0.2%
```

```sql
CREATE STATISTICS orders_status_type (mcv)
ON order_status, order_type
FROM orders;

ANALYZE orders;
```

#### How `mcv` Differs from `dependencies`

|Type|What it models|What it can't tell you|
|---|---|---|
|`dependencies`|How strongly A predicts B overall|The exact frequency of `(A=x, B=y)`|
|`mcv`|Actual frequency of the top N combinations|Combinations outside the stored list|

`dependencies` answers: _"If I know `status`, how much do I know about `priority`?"_

`mcv` answers: _"How often does `(status='pending', priority='high')` actually appear?"_

When in doubt, combine them:

```sql
CREATE STATISTICS orders_status_type_full (dependencies, mcv)
ON order_status, order_type
FROM orders;

-- Or all three:
CREATE STATISTICS orders_location_full (ndistinct, dependencies, mcv)
ON city, country, order_status
FROM orders;

ANALYZE orders;
```

---

### Bonus: Expression Statistics (PostgreSQL 14+)

If your filters or indexes use computed expressions, you can create statistics on the expression itself:

```sql
CREATE STATISTICS s_order_year ON (EXTRACT(year FROM created_at)) FROM orders;
CREATE STATISTICS s_email_lower ON (lower(email)) FROM users;

ANALYZE orders;
ANALYZE users;
```

Without this, the planner falls back to a crude default selectivity guess (often 0.5%), even when the expression index exists. Same principle: the index might be there, but the planner needs accurate estimates to choose it.

---

## How Bad Estimates Cascade Through a Plan

Row estimates drive far more than index selection:

- Join order
- Join type (hash join / nested loop / merge join)
- Filter pushdown
- Memory grants

Here is the cascade in a concrete example:

```sql
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN regions r ON c.region_id = r.id
WHERE o.status = 'pending'
  AND o.priority = 'high';
```

Suppose `orders` has 10M rows, `status = 'pending'` covers 40%, and `priority = 'high'` covers 30%, but together they're only **0.5% of rows** due to correlation.

**Without extended statistics:**

```text
Estimate: 40% × 30% = 12% → 1,200,000 rows
```

The planner thinks the filter is weak, so it:

1. Joins smaller tables first, passing a large `orders` result forward
2. Ignores the composite index on `(status, priority)` — 12% isn't selective enough
3. Applies the filter late on a massive intermediate result
4. Chooses nested loop based on the wrong row count

**With extended statistics:**

```sql
CREATE STATISTICS s_order_filter (dependencies, mcv)
ON status, priority FROM orders;

ANALYZE orders;
```

Estimate becomes ~50,000 rows. Join order flips. The index becomes eligible. The expensive step shrinks by ~200×.

```text
One bad estimate
        ↓
Filter looks weak (12% vs actual 0.5%)
        ↓
┌── Index ignored
├── Filter pushed late
├── Wrong table goes first in join
└── Nested loop on huge input
        ↓
Slow query, memory pressure, disk spill
```

---

## How to Diagnose the Problem

### Step 1: Spot the Estimate Gap

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
WHERE city = 'Paris' AND country = 'France';
```

Look at the `rows=` estimate vs `actual rows=`:

```text
Seq Scan on orders  (cost=... rows=12 ...) (actual time=... rows=48291 ...)
```

A **10×–1000× gap** on multi-column predicates is your signal. The planner is working from wrong information.

### Step 2: Create Statistics and Refresh

```sql
CREATE STATISTICS s_city_country (dependencies, mcv)
ON city, country FROM orders;

ANALYZE orders;
```

### Step 3: Re-explain

If estimates now track actuals, index choice and join shape will usually fix themselves without any query rewriting.

### Inspect What's Already There

```sql
-- List all extended statistics
SELECT
  n.nspname       AS schema,
  c.relname       AS table,
  s.stxname       AS stat_name,
  s.stxkind       AS types
FROM pg_statistic_ext s
JOIN pg_class c ON c.oid = s.stxrelid
JOIN pg_namespace n ON n.oid = c.relnamespace;

-- View computed stats (PostgreSQL 14+)
SELECT * FROM pg_stats_ext;
SELECT * FROM pg_stats_ext_exprs;
```

---

## Operational Checklist

### Check for stale statistics

```sql
SELECT schemaname, tablename, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
WHERE last_analyze < NOW() - INTERVAL '7 days'
ORDER BY n_live_tup DESC;
```

### Key tuning knobs

|Parameter|Default|Effect|
|---|---|---|
|`default_statistics_target`|100|Histogram and MCV depth. Increase for skewed columns.|
|`autovacuum_analyze_scale_factor`|0.2|% of table changed before auto-analyze triggers|
|`autovacuum_analyze_threshold`|50|Minimum row changes before auto-analyze triggers|

Per-column override for high-cardinality or skewed columns:

```sql
ALTER TABLE orders ALTER COLUMN city SET STATISTICS 500;
ANALYZE orders;
```

### Housekeeping

```sql
-- Remove unused statistics
DROP STATISTICS IF EXISTS myschema.s_old_stat;

-- Force a fresh compute
ANALYZE VERBOSE orders;
```

---

## Quick Reference

|Type|What It Models|Best For|
|---|---|---|
|`dependencies`|How strongly A predicts B|Correlated filters (`WHERE a=x AND b=y`)|
|`ndistinct`|Unique value combinations|`GROUP BY`, aggregate memory estimates|
|`mcv`|Joint frequency of top N combinations|Skewed multi-column distributions|
|Expressions (14+)|Stats on computed expressions|Filters on `lower()`, `EXTRACT()`, etc.|

---

## The Relationship Between Indexes and Statistics

This is the part that usually gets missed.

An index answers: _"Can I find these rows quickly?"_

Statistics answer: _"How many rows am I looking for?"_

The planner only chooses an index when the estimated row count is selective enough — rough break-even is typically **1–5% of the table**. Above that, sequential scan often wins in the cost model.

**Overestimation kills index use:**

```text
Planner thinks: 1.2M rows (12%) → seq scan is cheaper → ignores your index
Reality:            50k rows  (0.5%) → index scan would take milliseconds
```

**Underestimation chooses the wrong join type:**

```text
Planner thinks: 50 rows → nested loop looks cheap
Reality:    500,000 rows → random I/O explosion
```

Neither situation is "the index is wrong." The planner just trusted bad cardinality. Extended statistics fix the input — the index then gets to do the job you built it for.

---

## Summary

PostgreSQL's per-column statistics are powerful but built on an independence assumption that real data constantly violates. When correlated columns appear in the same predicate, estimates can be off by orders of magnitude — and bad estimates cascade into wrong index choices, wrong join orders, and slow plans.

Extended statistics close that gap. They don't replace indexes or partitioning. They feed the planner accurate cardinality so the tools you've already built actually get used correctly.

When `EXPLAIN ANALYZE` shows a large gap between estimated and actual rows on a multi-column predicate:

1. Create extended statistics on those columns
2. Run `ANALYZE`
3. Re-check the plan

Often the index you already have finally gets to do its job.

---

_Tags: `postgresql` `database` `query-optimization` `performance` `sql` `indexes`_

---

## Related

- [[Explain Analyze]]
- [[pg_stat_statements]]
- Published version: https://kareemdev.hashnode.dev

← [[Concepts MOC]]
