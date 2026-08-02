# Module 1 — OLTP vs OLAP Mental Model

[[00 OLAP MOC|↑ Table of Contents]] · [[02 Module 2 — Columnar Execution Internals|Next: Module 2 →]]

---

## Module 1 Summary

> **One-liner:** OLTP finds a few rows fast (seek); OLAP aggregates many rows fast (scan) — and that difference drives storage layout.

**Most important takeaways:**

1. **Seek vs scan** — Postgres indexes shine for point lookups; they rarely help full-population aggregates (`SUM`, `GROUP BY` over most rows).
2. **Row store vs columnar** — Postgres stores whole rows together; OLAP stores columns together, so a query reading 2 of 40 columns can cut I/O by ~20× before any smarter algorithm.
3. **When to stay / offload** — Stay in Postgres for small data, freshness, or simple indexed aggregates. Offload when full scans grow, dashboards need `LIMIT`/sampling, or analytics hurts OLTP.
4. **Columnar trade-off** — Fast multi-row aggregations; expensive single-row writes and `SELECT *` (row must be reassembled from many column segments).

**Jump to:** [[#1.1 What problem OLAP solves that Postgres struggles with|1.1]] · [[#1.2 Seek-heavy vs scan-heavy workloads|1.2]] · [[#1.3 When to keep analytics in Postgres vs offload|1.3]] · [[#1.4 Row store vs columnar store|1.4]] · [[#1.6 Postgres → OLAP Translation Table|1.6 Translation]]

---

## 1.1 What problem OLAP solves that Postgres struggles with

PostgreSQL was built to answer questions like:

> _"What is the current status of order #88421?"_

That question touches **one row**, or at most a few rows. Postgres's entire architecture — B-tree indexes, MVCC, WAL, row-level locking — is optimized to find and update that one row safely under many concurrent users.

OLAP systems are built to answer a fundamentally different kind of question:

> _"What was the total revenue per product category, per month, across all 200 million orders in the last three years?"_

That question does not care about any individual row. It needs to scan a column (or a few columns) across hundreds of millions of rows, aggregate, and return a small result. No index helps — you have to touch the entire dataset. Speed comes from:

1. Reading fewer bytes per row (only the columns you need)
2. Processing many rows at once (batched, vectorized)
3. Leveraging compression (because similar values cluster together)

**The root mismatch:** Postgres stores each row's fields together on disk. To read one column across 10 million rows, you load every column. OLAP systems store each column's values together, so reading `revenue` across 10 million rows reads _only_ the revenue bytes. Everything else is never touched.

**Analogy:** Imagine a spreadsheet printed on paper. Postgres stores it row by row (each sheet is one row, all columns present). OLAP stores it column by column (one stack of papers per column). If you need to sum the "Revenue" column, Postgres hands you all 50 stacks and makes you flip to the Revenue entry on each sheet. OLAP hands you the single Revenue stack. Same data — radically different physical access cost.

---

## 1.2 Seek-heavy vs scan-heavy workloads

This distinction is the single most clarifying idea in the entire OLAP curriculum. Once you see it, tool choices become obvious.

||OLTP (Postgres)|OLAP|
|---|---|---|
|**Access pattern**|Seek — jump to specific rows|Scan — sweep across many rows|
|**Rows touched per query**|1 – ~1,000|Millions – billions|
|**Columns touched per query**|Most/all (you want the full record)|A few (just SELECT / WHERE columns)|
|**Does an index help?**|Yes, dramatically — avoids the scan|Rarely — you're reading most rows anyway|
|**Primary bottleneck**|Latency: time to find the row|Throughput: bytes read per second|
|**Write profile**|High-frequency, small, concurrent|Bulk loads, infrequent|
|**Ideal storage layout**|Row-oriented (fetch the full record fast)|Column-oriented (read only what you aggregate)|

### The seek metaphor

**Seek-heavy** is like looking up a word in a dictionary. You need one entry; the index gets you there in milliseconds. Nobody reads a dictionary cover to cover.

**Scan-heavy** is like an auditor reading every expense report in a company to compute total travel spend. There is no index that helps her. She needs to read fast and add numbers. What she _doesn't_ need is to read the employee name, department, approver, and notes on each report — only the amount column matters.

### Why Postgres indexes stop helping

A Postgres B-tree index is a seek structure. When you `SELECT * FROM orders WHERE order_id = 42`, the index finds the exact heap tuple in microseconds. But when you `SELECT SUM(revenue) FROM orders`, the planner correctly ignores the index and does a sequential scan — because you need _every row anyway_. The seek structure gives you nothing when the destination is "all rows."

OLAP workloads are almost entirely in the "auditor" category. You're rarely looking up one row. You're aggregating many.

---

## 1.3 When to keep analytics in Postgres vs offload

This is a real engineering decision. Don't reach for a columnar system as a default; reach for it when Postgres is actually hurting you.

### Stay in Postgres when:

- **Data is small** — under ~50 million rows or ~20 GB. Postgres sequential scans on small data are fast. The overhead of an OLAP pipeline isn't worth it.
- **You need freshness** — analytical queries on transactional data where even seconds of lag are unacceptable. Columnar systems involve ETL delay.
- **Queries are simple aggregates on indexed columns** — `SELECT COUNT(*) FROM orders WHERE status = 'paid'` with an index on `status` is a seek, not a scan.
- **You want one system** — fewer moving parts is always worth something. Operational simplicity has real cost.
- **You're already using Postgres features for analytics** — `pg_partman`, materialized views, partial indexes, `BRIN`indexes. These can carry you far before you need a separate system.

### Offload to OLAP when:

- **Full table scans take seconds and are growing** — query time scales with data volume, and you've already tuned indexes as far as they'll go.
- **You're adding `LIMIT` or sampling to make dashboards usable** — this is the clearest signal.
- **Analytics is degrading OLTP performance** — shared I/O, autovacuum contention, buffer pool eviction. Analytics and transactions competing for the same resources is a common and underdiagnosed problem.
- **You're querying across multiple sources** — files on disk, S3, other databases. Postgres can do this with FDWs, but awkwardly.
- **Historical data dwarfs operational data** — you have 5 years of orders history but only care about current-week operations in your app. The historical data is pure analytics.

### The practical path

Start in Postgres. Add `EXPLAIN (ANALYZE, BUFFERS)` to your slow queries. When you see Seq Scan on a large table with a high buffer hit count, and indexes can't help because the query genuinely needs most rows — that's your signal. Export to Parquet, query with DuckDB, and compare. Don't migrate prematurely.

---

## 1.4 Row store vs columnar store

This is the most important mental model in the curriculum. Everything else — compression, vectorized execution, Parquet internals, partitioning — is a consequence of this layout decision.

### Row store (how Postgres heap files work)

Every row's fields are stored contiguously on disk. A page (8 KB in Postgres) holds as many complete rows as fit.

```
Page on disk (Postgres heap):
┌────────────────────────────────────────────────────────────────────┐
│ Row 1: [order_id=1, customer='Alice', product='Keyboard', rev=79]  │
│ Row 2: [order_id=2, customer='Bob',   product='Mouse',    rev=29]  │
│ Row 3: [order_id=3, customer='Alice', product='Monitor',  rev=399] │
│ Row 4: [order_id=4, customer='Carol', product='Webcam',   rev=89]  │
└────────────────────────────────────────────────────────────────────┘
```

To compute `SUM(revenue)` across 4 rows, Postgres reads the entire page — all fields of all 4 rows — even though it only needs the `revenue` values.

**What row storage is excellent at:** You want the full row. `SELECT * FROM orders WHERE order_id = 42` hits one page, reads one row, returns everything. Perfect.

**What row storage is bad at:** You want one column across millions of rows. You pay I/O for every unused column on every row. On a table with 40 columns and you need 2, you're reading 38 columns of garbage per row.

### Columnar store (how Parquet and DuckDB work)

Each column's values are stored contiguously. The `revenue` column for all rows lives together; the `customer` column lives elsewhere.

```
On disk (columnar layout):
order_id column:  [1,       2,     3,         4      ]
customer column:  ['Alice', 'Bob', 'Alice',   'Carol' ]
product column:   ['Keyboard','Mouse','Monitor','Webcam']
revenue column:   [79,      29,    399,       89     ]
```

`SUM(revenue)` reads only the revenue bytes. Four values, contiguous on disk. The other three columns are never touched.

### The I/O math

Say you have a table with:

- 100 million rows
- 40 columns
- Average row size: 400 bytes
- You need 2 columns (8 bytes total per row) for your query

|Storage|Bytes read|I/O ratio|
|---|---|---|
|Row store|100M × 400 = 40 GB|100% of the table|
|Columnar|100M × 8 = 800 MB|2% of the table|

That's a 50x I/O reduction before any other optimization. Not from a smarter algorithm — from physical layout alone.

### What columnar storage gains on CPU too

Columnar layouts don't just reduce I/O. They also unlock:

- **Better compression** — a column of integers from 1–5 (like a `status` enum) is highly compressible; mixed row data is not. (Covered in Module 2.)
- **SIMD operations** — modern CPUs can add 8 integers at once using vector instructions. A contiguous array of 64-bit floats is exactly the shape SIMD needs. A row of mixed types is not.
- **Cache efficiency** — if you're summing a float column, every CPU cache line fetched contains only floats. Nothing is wasted on neighboring string data.

### The trade-off columnar storage makes

Columnar is excellent for reads across many rows. It is bad for writes and point lookups:

- **Writes are expensive** — inserting one row means updating 40 column files, not one row slot. Most columnar systems batch writes to amortize this cost.
- **Full-row reads are more expensive** — `SELECT *` on a columnar store must reassemble the row from 40 separate column segments. A row store just reads the heap tuple.
- **No cheap random row access** — you can't jump to "row 88421" like a Postgres heap page with a TID.

This is not a flaw — it's a deliberate trade-off. Columnar systems accept expensive writes and expensive full-row reads in exchange for extremely fast aggregations over a few columns of many rows.

---

---

## 1.6 Postgres → OLAP Translation Table

|Postgres concept|OLAP equivalent|Key difference|
|---|---|---|
|Heap file (row store)|Column file / Parquet file|Layout is column-by-column, not row-by-row|
|Sequential scan|Column scan|Columnar scan reads only needed columns|
|Index scan|Partition pruning / row group filtering|Different mechanism, same goal: skip data|
|`EXPLAIN (ANALYZE)`|`EXPLAIN ANALYZE` (DuckDB)|Different node names; columnar plan looks different|
|Buffer cache (shared_buffers)|Column buffer / memory_limit|Same idea: hot data in RAM|
|Autovacuum|Compaction jobs|Different mechanism for reclaiming space / reorganizing|
|INSERT INTO|Bulk COPY / append-only write|OLAP writes are batched, not row-by-row|
|MVCC (row versioning)|Immutable file model|Many OLAP systems avoid MVCC by treating files as immutable|
|pg_stat_user_tables|File-level statistics / SUMMARIZE|OLAP stats are in file footers, not a system catalog|

---

[[#Module 1 Summary|↑ Module 1 Summary]] · [[00 OLAP MOC|↑ TOC]] · [[02 Module 2 — Columnar Execution Internals|Next: Module 2 →]]

---

