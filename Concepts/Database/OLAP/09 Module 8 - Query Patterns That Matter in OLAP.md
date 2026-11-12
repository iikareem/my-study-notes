# Module 8 — Query Patterns That Matter in OLAP

[[#Module 7 — DuckDB in Practice|← Module 7]] · [[#Table of Contents|↑ Table of Contents]]

---

## Module 8 Summary

> **One-liner:** OLAP query design is about reducing what the engine has to touch — the right GROUP BY, the right window function, and knowing when "close enough" is faster and good enough.

**Most important takeaways:**

1. **GROUP BY design** — Column order in GROUP BY affects cardinality explosion. Filter before grouping, not after. Push filters as deep as possible.
2. **Window functions** — The right tool for "running totals", "rank within group", and "compare to previous period". Misused, they scan the full dataset repeatedly. Used correctly, they scan once.
3. **Approximate vs exact** — `COUNT(DISTINCT ...)` is expensive at scale. Approximations (`HLL`, `approx_count_distinct`) are 99% accurate and orders of magnitude faster. Know when exact is actually required.
4. **Materialized aggregates** — Pre-computing common GROUP BY results trades query freshness for speed. The right trade-off depends on how often the question is asked and how stale the answer can be.
5. **Benchmarking fairly** — Cold vs warm runs measure different things. Most benchmarks you read are measuring the wrong thing for your use case.

**Jump to:** [[#8.1 Aggregations and GROUP BY design|8.1]] · [[#8.2 Window functions|8.2]] · [[#8.3 Approximate vs exact|8.3]] · [[#8.4 Materialized aggregates and summary tables|8.4]] · [[#8.5 Benchmarking fairly|8.5]] · [[#8.6 Postgres → OLAP Query Patterns Translation Table|8.6 Translation]]

---

## 8.1 Aggregations and GROUP BY design

Aggregations are the point of OLAP. Every design decision — storage layout, partitioning, modeling — ultimately exists to make `GROUP BY` fast. Understanding what the engine actually does helps you write queries that cooperate with it.

### What the engine does during GROUP BY

```sql
SELECT country, product_category, SUM(revenue)
FROM fact_orders
WHERE order_date >= '2024-01-01'
GROUP BY country, product_category;
```

1. Scan the relevant column chunks (`country`, `product_category`, `revenue`, `order_date`)
2. Apply the WHERE filter to eliminate rows
3. Build a hash table keyed by `(country, product_category)` — one entry per unique combination
4. For each surviving row, accumulate `revenue` into the matching hash table entry
5. Output the hash table

The cost is dominated by two things: **how many rows survive the filter** (determines scan cost) and **how many distinct GROUP BY combinations exist** (determines hash table size).

### Cardinality of the GROUP BY key

The number of distinct combinations in your GROUP BY is called the **cardinality of the key**. It directly affects memory usage and hash table efficiency.

```sql
-- Low cardinality key (few distinct values) — small hash table, fast
GROUP BY country                         -- ~200 distinct values

-- High cardinality key — large hash table, may spill to disk
GROUP BY customer_id                     -- millions of distinct values

-- Cardinality explodes when combining columns
GROUP BY country, product_category       -- 200 × 500 = up to 100,000 combinations
GROUP BY country, customer_id            -- 200 × 1M = up to 200M combinations (dangerous)
```

**Rule:** group by the columns you need, not more. Every additional GROUP BY column multiplies the potential cardinality. Grouping by `customer_id` alongside aggregate dimensions almost always means you're solving the wrong problem — that belongs in a `WHERE` filter or a subquery.

### Filter before grouping, not after

```sql
-- ❌ Filter after grouping (HAVING on an aggregated result)
SELECT country, SUM(revenue) AS total
FROM fact_orders
GROUP BY country
HAVING total > 1000000;
-- Groups ALL countries first, then discards the small ones

-- ✅ Filter before grouping when possible (WHERE on raw rows)
SELECT country, SUM(revenue) AS total
FROM fact_orders
WHERE order_date >= '2024-01-01'   -- eliminates rows before the GROUP BY
GROUP BY country;
```

`HAVING` is for filtering on the result of an aggregation (e.g., `HAVING SUM(revenue) > 1M`) — it cannot be replaced by `WHERE`. But filtering on non-aggregated columns should always be in `WHERE`, not `HAVING`. The optimizer usually catches this, but be explicit.

### Push filters into subqueries and CTEs

Columnar engines push WHERE predicates down to the scan (Module 2). But if your filter is buried in an outer query, the engine may scan the full table before applying it.

```sql
-- ❌ Full scan of fact_orders, then filter in the outer query
WITH monthly AS (
    SELECT order_date, country, SUM(revenue) AS revenue
    FROM fact_orders
    GROUP BY order_date, country
)
SELECT * FROM monthly WHERE order_date >= '2024-01-01';

-- ✅ Filter at the source — engine pushes it to the Parquet scan
WITH monthly AS (
    SELECT order_date, country, SUM(revenue) AS revenue
    FROM fact_orders
    WHERE order_date >= '2024-01-01'   -- ← pushed here
    GROUP BY order_date, country
)
SELECT * FROM monthly;
```

DuckDB is often smart enough to push filters through CTEs, but do not rely on it. Write the filter where it logically belongs — at the source.

### Aggregation on pre-aggregated data

When querying a pre-aggregated table (Module 8.4), be careful about re-aggregating counts and averages.

```sql
-- summary table stores: date, country, order_count, total_revenue

-- ✅ SUM of a sum is correct
SELECT country, SUM(total_revenue) FROM summary GROUP BY country;

-- ✅ SUM of a count is correct
SELECT country, SUM(order_count) FROM summary GROUP BY country;

-- ❌ AVG of a pre-aggregated average is wrong
-- If you stored avg_order_value per day, averaging those gives unweighted averages
SELECT country, AVG(avg_order_value) FROM summary GROUP BY country;
-- Correct: SUM(total_revenue) / SUM(order_count)
```

Store numerator and denominator separately. Compute averages at query time. The same rule as non-additive measures in Module 5.

---

## 8.2 Window functions

Window functions compute a value for each row based on a set of related rows — without collapsing them into a single output row the way `GROUP BY` does. They are the right tool for a class of questions that `GROUP BY` cannot answer cleanly.

### The mental model

```sql
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date) AS running_total
FROM fact_orders;
```

`GROUP BY` collapses many rows into one. A window function keeps all rows and adds a column. The `OVER (...)` clause defines the "window" — the set of rows each calculation looks at.

### The OVER clause

```sql
OVER (
    PARTITION BY country        -- reset the window for each country (optional)
    ORDER BY order_date         -- defines row order within the window (required for running calcs)
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- frame: from start to current row
)
```

- **No PARTITION BY** — one window across the entire dataset
- **PARTITION BY country** — a separate window per country; calculations reset per group
- **No ORDER BY** — all rows in the partition are peers (good for `RANK`, bad for running totals)
- **ORDER BY + frame** — defines running/rolling calculations

### Common window function patterns

**Running total**

```sql
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (ORDER BY order_date
                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_revenue
FROM daily_revenue
ORDER BY order_date;
```

**Rank within group** (e.g., top customers per country)

```sql
SELECT
    country,
    customer_id,
    total_revenue,
    RANK() OVER (PARTITION BY country ORDER BY total_revenue DESC) AS rank_in_country
FROM customer_revenue;
-- Filter to top 10 per country: WHERE rank_in_country <= 10
```

**Period-over-period comparison** (compare to previous row)

```sql
SELECT
    order_date,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY order_date) AS prev_day_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_revenue;
```

**Rolling 7-day average**

```sql
SELECT
    order_date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7d_avg
FROM daily_revenue;
```

### Performance: what window functions cost

Window functions require sorting the data by the ORDER BY column. If the data is not already sorted, the engine sorts it. For large datasets this is expensive.

```sql
-- ❌ Two window functions with different ORDER BY — sorts twice
SELECT
    SUM(revenue) OVER (PARTITION BY country ORDER BY order_date) AS country_running,
    SUM(revenue) OVER (ORDER BY order_date) AS global_running
FROM fact_orders;

-- ✅ Same OVER clause — sorts once, reuses
SELECT
    SUM(revenue) OVER (ORDER BY order_date) AS global_running,
    COUNT(*) OVER (ORDER BY order_date) AS global_count
FROM fact_orders;
```

**Rule:** window functions that share the same `PARTITION BY` and `ORDER BY` reuse the sort. Different `OVER` clauses require separate sorts. Minimize distinct `OVER` clause shapes in a single query.

### When to use GROUP BY vs window functions

|Question|Use|
|---|---|
|"Total revenue per country"|`GROUP BY`|
|"Revenue per country as % of total"|Window function (`SUM OVER ()` for the denominator)|
|"Running total of revenue over time"|Window function|
|"Rank customers by revenue within each country"|Window function|
|"Compare this month to last month"|Window function (`LAG`) or self-join|
|"Top 10 products overall"|`GROUP BY` + `ORDER BY` + `LIMIT`|
|"Top 10 products per category"|Window function (`RANK OVER PARTITION BY`)|

---

## 8.3 Approximate vs exact

Some aggregate functions are cheap. Some are expensive. `COUNT(DISTINCT ...)` is the most commonly expensive one, and most uses of it do not actually require an exact answer.

### Why COUNT(DISTINCT) is expensive

```sql
SELECT COUNT(DISTINCT customer_id) FROM fact_orders WHERE order_date >= '2024-01-01';
```

To count distinct values exactly, the engine must track every unique value it has seen. For 100 million rows with 10 million distinct customers, this requires a hash set of 10 million entries — hundreds of MB of memory, potential spill to disk, and a global reduction step that cannot be parallelized cleanly.

Compare to `COUNT(*)` or `SUM(revenue)` — both can be computed independently per partition and merged with simple addition. `COUNT(DISTINCT)` cannot.

### Approximation: HyperLogLog

DuckDB (and most analytical systems) implement `approx_count_distinct()` using the HyperLogLog algorithm. It uses a fixed, small amount of memory regardless of cardinality and is typically accurate to within 1–2%.

```sql
-- Exact — expensive
SELECT COUNT(DISTINCT customer_id) FROM fact_orders;
-- Result: 10,482,341   (took 12 seconds)

-- Approximate — fast
SELECT approx_count_distinct(customer_id) FROM fact_orders;
-- Result: 10,491,000   (took 0.4 seconds, ~0.08% error)
```

### When approximate is acceptable

Most analytical questions do not require exactness:

- "How many unique visitors did we have last month?" — ±1% is fine for any business decision.
- "How many distinct products were sold?" — ±1% is fine.
- "Is our DAU growing or declining?" — The trend is what matters, not the exact count.

### When exact is required

- Financial reporting — exact revenue figures, exact transaction counts for reconciliation.
- Deduplication logic — if you're deciding whether to process a record, approximate doesn't help.
- Compliance or audit — exact counts that appear in regulatory filings.

**Rule:** default to `approx_count_distinct`. Switch to exact only when you can name a specific reason exactness is required. Most of the time you cannot.

### Other approximations worth knowing

```sql
-- Approximate quantiles (median, percentiles) — exact quantiles require sorting everything
SELECT approx_quantile(response_time_ms, 0.95) FROM request_logs;
-- Much faster than: SELECT quantile_cont(0.95) WITHIN GROUP (ORDER BY response_time_ms)

-- Reservoir sampling — approximate random sample without reading all rows
SELECT * FROM fact_orders USING SAMPLE 1%;
-- Returns ~1% of rows; useful for exploration and prototyping
```

---

## 8.4 Materialized aggregates and summary tables

A **materialized aggregate** (or summary table) is a pre-computed GROUP BY result stored as a table or Parquet file. Instead of re-scanning the fact table on every query, you query the pre-aggregated result.

### The basic pattern

```sql
-- Build the summary table once (or on a schedule)
CREATE TABLE summary_daily_revenue AS
SELECT
    order_date,
    country,
    product_category,
    COUNT(*) AS order_count,
    SUM(revenue) AS total_revenue,
    SUM(cost) AS total_cost
FROM fact_orders
GROUP BY order_date, country, product_category;

-- Query the summary — scans thousands of rows instead of hundreds of millions
SELECT country, SUM(total_revenue)
FROM summary_daily_revenue
WHERE order_date >= '2024-01-01'
GROUP BY country;
```

The summary table has one row per `(date, country, category)` combination. If you have 3 years of data, 50 countries, and 200 categories, that's about 10 million rows — compared to potentially billions in the fact table.

### The freshness trade-off

|Approach|Query speed|Data freshness|Complexity|
|---|---|---|---|
|Query fact table directly|Slower (scans billions of rows)|Real-time|Low|
|Query summary table|Faster (scans millions of rows)|Stale by the refresh interval|Medium|
|Query summary + fact for recent data|Fast for history, correct for recent|Fresh|Higher|

**The freshness question:** How stale can the answer be before it affects a business decision?

- A dashboard showing last month's revenue → refreshing once a day is fine.
- A live operations screen showing orders in the last hour → pre-aggregation is wrong; query the fact table directly.
- An annual report → refresh once before the report is generated.

### Incremental refresh vs full rebuild

**Full rebuild** — truncate and recompute the entire summary table. Simple, always correct, expensive for large date ranges.

```sql
-- Full rebuild
DELETE FROM summary_daily_revenue;
INSERT INTO summary_daily_revenue
SELECT order_date, country, product_category, COUNT(*), SUM(revenue), SUM(cost)
FROM fact_orders
GROUP BY order_date, country, product_category;
```

**Incremental refresh** — recompute only recent partitions. Fast, but requires idempotent partition replacement (Module 6.2) and care around late-arriving data (Module 6.3).

```sql
-- Incremental: replace only the last 3 days (handles late arrivals)
DELETE FROM summary_daily_revenue WHERE order_date >= CURRENT_DATE - 3;
INSERT INTO summary_daily_revenue
SELECT order_date, country, product_category, COUNT(*), SUM(revenue), SUM(cost)
FROM fact_orders
WHERE order_date >= CURRENT_DATE - 3
GROUP BY order_date, country, product_category;
```

### What to pre-aggregate and what not to

Pre-aggregate when:

- The same GROUP BY is run frequently (multiple times per hour by multiple users or dashboards)
- The fact table is large enough that the scan takes more than a few seconds
- The grain of the summary matches what users actually ask (daily, not hourly)

Do not pre-aggregate when:

- Queries are ad hoc and unpredictable — you cannot pre-compute every possible GROUP BY
- The fact table is small enough to scan in under a second
- Data freshness requirements are strict

### Summary tables are not a substitute for good modeling

A summary table is a performance optimization. If you need a summary table because your queries are slow, first check: is the slowness from a modeling problem (wrong grain, missing dimension attributes forcing joins) or a layout problem (no partitioning, small files)? Fix those first. Summary tables add operational complexity — refresh jobs, staleness management, storage duplication. They should be the last tool, not the first.

---

## 8.5 Benchmarking fairly

Benchmarking query performance is surprisingly easy to do wrong. Most benchmark numbers you read — in blog posts, vendor comparisons, and your own tests — are measuring something different from what you think.

### Cold vs warm runs

**Cold run:** the data is not in any cache. The engine reads from disk (or S3) for the first time. This measures I/O performance + query execution.

**Warm run:** the data (or parts of it) is in the OS page cache, DuckDB's buffer, or a CDN cache. This measures query execution with fast data access.

```
Cold run on a 50 GB dataset: 45 seconds
Warm run on the same query:   3 seconds
```

Same query, same data, 15x different result. Neither is wrong — they measure different things.

**What to measure for your use case:**

- **Dashboards and repeated queries** — warm run matters. Users run the same query repeatedly. Cache effects are real and intentional.
- **One-off analytical queries, ad hoc exploration** — cold run matters. The data is unlikely to be cached.
- **Scheduled batch jobs** — cold run matters. Jobs run infrequently and should not assume cache state.

Always state which you measured. Always run multiple times and take the median, not the minimum.

### What "faster" actually means

```
System A: first query 60s, subsequent queries 2s
System B: first query 10s, subsequent queries 8s
```

Which is faster? It depends entirely on your query pattern. If users run each query once → System B. If they run the same query repeatedly → System A after the first run.

### What to control when benchmarking

**Clear the cache between cold runs:**

```bash
# Linux: drop OS page cache
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches

# DuckDB: restart the process (in-memory cache is gone on restart)
```

**Use realistic data, not toy data.** A 1 GB benchmark does not predict 1 TB behavior. Columnar engines have non-linear behavior — compression ratios change, row group skipping behavior changes, memory pressure changes.

**Use realistic queries.** A single `SELECT COUNT(*) FROM table` is not representative. Benchmark the queries your application actually runs — with the actual filters, joins, and GROUP BY clauses.

**Measure end-to-end time, not just query time.** For applications, what matters is wall-clock time from "query submitted" to "result received" — including network, serialization, and any connection overhead.

### Common benchmarking mistakes

|Mistake|Why it matters|
|---|---|
|Reporting only the fastest run|Minimum time is an outlier. Report median of 5+ runs.|
|Benchmarking on warm cache without disclosing it|Inflates performance vs real-world cold queries.|
|Comparing different hardware|An NVMe benchmark vs an HDD benchmark says nothing about the query engine.|
|Using `SELECT *` in the benchmark|`SELECT *` defeats columnar pruning — it tests I/O, not analytical performance.|
|Benchmarking with toy row counts|1M rows behaves differently from 1B rows. Use realistic scale.|
|Ignoring data layout differences|System A with sorted Parquet vs System B with unsorted CSV is not a fair engine comparison.|

### The benchmark question to ask first

Before running any benchmark: **"What decision will this number inform?"**

If the answer is "which system to use for production", you need:

- Realistic data volume and distribution
- Realistic query mix (not one query)
- Cold run behavior (first run after deployment)
- Resource constraints that match production (CPU, RAM, disk type)

If the answer is "is this optimization faster than what I had before", you need:

- The same data, same hardware, same cache state
- The same query
- Multiple runs, median result

---

## 8.6 Postgres → OLAP Query Patterns Translation Table

|Postgres concept|OLAP equivalent|Key difference|
|---|---|---|
|`GROUP BY` + `SUM` / `COUNT`|Same syntax — different execution|OLAP uses vectorized hash aggregation; much faster on large scans|
|`HAVING`|Same — use for post-aggregation filters|Use `WHERE` for pre-aggregation filters; don't overuse `HAVING`|
|`COUNT(DISTINCT col)`|`approx_count_distinct(col)`|Exact is expensive; approximate is almost always sufficient|
|Window functions (`OVER`)|Same syntax — DuckDB supports all standard window functions|Performance characteristics differ; sort cost is the main factor|
|`LAG` / `LEAD`|Same|Identical syntax and semantics|
|Materialized view (`CREATE MATERIALIZED VIEW`)|Summary table (`CREATE TABLE AS SELECT`) + manual refresh|DuckDB has no auto-refresh; you schedule it externally|
|`REFRESH MATERIALIZED VIEW`|Delete partition + re-insert|OLAP refresh is manual and partition-scoped|
|`EXPLAIN (ANALYZE, BUFFERS)`|`EXPLAIN ANALYZE`|Look for row groups scanned vs skipped, not buffer hits|
|`pg_stat_statements` (query history)|No built-in equivalent|Log queries manually or use an external tool|
|Index for fast aggregation|Partition pruning + row group stats + summary table|Three different layers, each doing part of what an index does|
|`TABLESAMPLE`|`USING SAMPLE 1%`|DuckDB sampling is simpler to write|

---

[[#Module 8 Summary|↑ Module 8 Summary]] · [[#Table of Contents|↑ TOC]] · [[#Module 7 — DuckDB in Practice|← Module 7]]

---

_End of Module 8._

> **Suggested next steps:**
> 
> - **Module 9 — End-to-End Pipeline Design:** Putting Modules 1–8 together — designing a full pipeline from source to dashboard, with layout, ingestion, modeling, and query layer decisions made coherently.
> - **Module 10 — When to Reach for a Cloud Warehouse:** When DuckDB + Parquet hits its limits, what Snowflake / BigQuery / Redshift actually give you, and what they cost.