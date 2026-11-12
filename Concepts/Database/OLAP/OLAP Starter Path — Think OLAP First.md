---
aliases:
  - OLAP Starter
  - Think OLAP
  - OLAP from scratch
tags:
  - concepts
  - database
  - olap
  - starter
---

# OLAP Starter Path — Think OLAP First

> This is a **readable lesson**, not a summary card. Each section explains the idea with examples, Postgres contrasts, and what changes in how you design systems.  
> At the end of each module section, **Go deeper** links into your full notes when you want more depth.

**You:** experienced with Postgres / OLTP. New to OLAP.  
**Goal:** after reading this, you can reason about scan vs seek, row vs column, Parquet, partitions, analytical modeling, ingest, DuckDB, and query design — then apply it on a real project.

**Full curriculum:** [[00 OLAP MOC]]

---

## 0. Why this feels different from Postgres

PostgreSQL was built to answer questions like:

> “What is the current status of order `#88421`?”

That question touches **one row**, maybe a few. Postgres’s whole architecture — B-tree indexes, MVCC, WAL, row-level locking, 8 KB heap pages — is optimized to find and update that row safely under many concurrent users.

OLAP systems are built to answer a different kind of question:

> “What was total revenue per product category, per month, across all 200 million orders in the last three years?”

That question does not care about any individual row. It needs to touch a huge population of rows, read only a few columns, aggregate, and return a small result. No B-tree “finds” that answer for you — you must scan. Speed comes from:

1. Reading fewer bytes per row (only the columns you need)
2. Processing many values at once (batched / vectorized)
3. Skipping whole chunks of data that cannot match your filter (partitions, file statistics)
4. Compressing well because similar values sit together in a column

**Analogy:** Imagine a spreadsheet printed on paper.

- Postgres stores it **row by row** (each sheet is one complete row). To sum Revenue, you flip every sheet and pick out the Revenue cell.
- OLAP stores it **column by column** (one stack of papers per column). To sum Revenue, you are handed the Revenue stack only.

Same data. Radically different physical access cost.

If you keep only one sentence from this whole file:

> **OLTP is seek-heavy (find a few rows fast). OLAP is scan-heavy (process many rows fast, reading few columns).**

Everything else — Parquet, DuckDB, partitioning, star schemas — exists to serve that second world.

---

## Module 1 — Seek vs scan, and why storage layout decides

### 1.1 Seek-heavy vs scan-heavy (the clarifying idea)

**Seek-heavy** is like looking up a word in a dictionary. You need one entry. An index gets you there in milliseconds. Nobody reads a dictionary cover to cover to answer “what does *ephemeral* mean?”

**Scan-heavy** is like an auditor reading every expense report in a company to compute total travel spend. There is no index that helps her. She needs to read fast and add numbers. What she does *not* need is employee name, department, approver, and notes on every report — only the amount column matters.

Make this concrete for systems you already know:

- `SELECT * FROM orders WHERE order_id = 42` → seek. Index helps. OLTP sweet spot.
- `SELECT SUM(revenue) FROM orders` → scan. Postgres correctly ignores the PK index and does a sequential scan, because you need every row anyway. The seek structure gives you nothing when the destination is “all rows.”
- `SELECT COUNT(*) FROM orders WHERE status = 'paid'` with a selective index on `status` → still mostly a seek (index-only or index + heap for matching keys). This can stay in Postgres happily.
- `SELECT date_trunc('month', created_at), category, SUM(revenue) FROM orders GROUP BY 1, 2` over years of history → scan-heavy analytical work. This is where OLAP thinking starts to matter.

**What changes in your head:** stop asking first “which index should I add?” Ask first “am I looking up rows, or aggregating a population?”

### 1.2 Row store — how Postgres heap pages work

Every row’s fields are stored **contiguously** on disk. An 8 KB Postgres page holds as many complete rows as fit:

```text
Page on disk (Postgres heap):
┌────────────────────────────────────────────────────────────────────┐
│ Row 1: [order_id=1, customer='Alice', product='Keyboard', rev=79]  │
│ Row 2: [order_id=2, customer='Bob',   product='Mouse',    rev=29]  │
│ Row 3: [order_id=3, customer='Alice', product='Monitor',  rev=399] │
│ Row 4: [order_id=4, customer='Carol', product='Webcam',   rev=89]  │
└────────────────────────────────────────────────────────────────────┘
```

To compute `SUM(revenue)` across those 4 rows, Postgres reads the entire page — all fields of all 4 rows — even though it only needs the revenue values.

**Excellent when:** you want the full row. `SELECT * FROM orders WHERE order_id = 42` hits one page, reads one row, returns everything. Perfect for APIs and transactions.

**Painful when:** you want one or two columns across millions of rows. On a table with 40 columns where you need 2, you still pay I/O for the other 38 on every row you touch.

### 1.3 Columnar store — how Parquet and DuckDB lay data out

Each column’s values are stored together. The `revenue` column for all rows lives in one strip; `customer` lives elsewhere:

```text
On disk (columnar layout):
order_id column:  [1, 2, 3, 4, ...]
customer column:  ['Alice', 'Bob', 'Alice', 'Carol', ...]
product column:   ['Keyboard', 'Mouse', 'Monitor', 'Webcam', ...]
revenue column:   [79, 29, 399, 89, ...]   ← SUM only needs this
```

`SUM(revenue)` reads only the revenue bytes. Contiguous. The other columns are never opened for that query.

### 1.4 The I/O math (why this is not a “micro-optimization”)

Suppose:

- 100 million rows
- 40 columns
- average row size 400 bytes
- your query needs 2 columns totaling 8 bytes per row

Row store roughly reads: `100M × 400 = 40 GB`  
Columnar roughly reads: `100M × 8 = 800 MB`

That is on the order of a **50× I/O reduction** before any “smarter algorithm.” Not from clever SQL. From physical layout alone.

Columnar also helps the CPU:

- **Compression** works better when a column is one type with repeating patterns (`status` with 5 values compresses absurdly well; mixed row blobs do not).
- **SIMD** (one CPU instruction over many values) wants contiguous typed arrays — exactly what a column strip is.
- **Cache lines** fill with only the values you care about, instead of mixing floats with strings from neighboring columns.

### 1.5 The trade-off you must accept

Columnar is not “Postgres but faster at everything.”

- **Writes of single rows are expensive** — inserting one row may touch many column segments. Systems batch writes.
- **`SELECT *` is more expensive** — the engine must reassemble the row from many column strips.
- **Point lookups are not the design center** — you do not get “jump to TID of row 88421” the way a heap + index does.

That is deliberate: accept worse point updates / full-row fetches in exchange for extremely fast aggregations over a few columns of many rows.

### 1.6 When to stay in Postgres vs offload

Do not reach for DuckDB/Parquet as a default fashion. Reach for them when Postgres is actually the wrong tool for the workload.

**Stay in Postgres when:**

- Data is still small (rough mental band: tens of millions of rows / tens of GB — sequential scans are still fine).
- You need second-level freshness on transactional data (ETL lag is unacceptable).
- Aggregates are selective and indexed (`COUNT(*) WHERE status = 'paid'`).
- Operational simplicity matters more than peak analytical speed.
- You are already getting mileage from materialized views, partitioning, BRIN, etc.

**Offload when:**

- Full scans take seconds and keep growing, and indexes cannot help because you genuinely need most rows.
- You are adding `LIMIT` / sampling just to make dashboards usable — clear signal.
- Analytics is evicting OLTP from the buffer pool / competing on I/O / hurting write latency.
- Historical data dwarfs operational data (5 years of history, app only needs this week).
- You need to query files / lakes / multiple sources awkwardly via FDWs.

**Practical path:** start in Postgres. Use `EXPLAIN (ANALYZE, BUFFERS)`. When you see large Seq Scans that truly need most rows, export a slice to Parquet, query with DuckDB, compare. Migrate when evidence says so — not before.

### How Module 1 should change your thinking

For every feature or report: **lookup or scan?**  
If scan + many rows + few columns → think columnar layout and lake design, not “one more B-tree.”

**Go deeper:** [[01 Module 1 — OLTP vs OLAP Mental Model]]

---

## Module 2 — How execution works (why columnar engines feel fast)

Layout alone is not the whole story. How the engine **walks the plan** matters.

### 2.1 Volcano / tuple-at-a-time (Postgres mental model)

Postgres uses the **volcano (iterator) model**. Every operator — Seq Scan, Filter, Hash Join, Aggregate — exposes something like `next()`. The top operator pulls from its child, which pulls from its child, and so on. Each call returns **one tuple**.

```text
Aggregate.next()
  └─ HashJoin.next()
       ├─ Scan.next() → row 1
       └─ Scan.next() → row 1
  └─ HashJoin.next()
       ├─ Scan.next() → row 2
       └─ Scan.next() → row 2
  ... once per row, millions of times
```

That is flexible and great for OLTP-shaped plans. On a 10 million row analytical scan with several operators, you pay enormous coordination overhead: one function call per row per operator, plus pointer-chasing through generic tuple structures. The CPU rarely gets a clean array of floats to chew with SIMD.

### 2.2 Vectorized execution (DuckDB mental model)

Columnar engines pass **batches** (vectors) — commonly on the order of ~1024 or ~2048 values — as typed contiguous arrays:

```text
Aggregate.next_batch()          ← receives ~1024 floats
  └─ Filter.next_batch()        ← produces a batch of survivors
       └─ ColumnScan.next_batch() ← reads ~1024 floats from storage
```

Same 10M row scan becomes ~10k batch handoffs instead of 10M row handoffs. More importantly:

- A batch is `float64[1024]`, not a soup of tagged tuple fields.
- `revenue > 100` becomes a tight loop the CPU loves.
- 1024 × 8 bytes ≈ 8 KB — often sits nicely in L1/L2 cache while the batch is processed.

**Factory analogy:** volcano = workers pass one part at a time. Vectorized = workers pass a crate of 1024 parts. Same assembly line shape, fewer handoffs, better hardware use.

**What this means for you (even though you don’t configure it):**

1. Explain plans look different — think in pipelines and scans of columns, not only per-row costs.
2. Your biggest levers become **which columns you read** and **which files/partitions you open**, not only “filter rows early” (though filtering still matters).
3. Tiny tables won’t show magical 50× wins. Vectorization pays off at serious volume.

### 2.3 Late materialization (worked example)

**Materialization** = assembling the columns you need into something you process.

Postgres tends to fetch a heap tuple (many columns together) early. Columnar engines can filter using **row positions** and only fetch output columns for survivors.

Query:

```sql
SELECT order_id, revenue
FROM orders
WHERE category = 'Electronics' AND revenue > 500
```

**Early style (conceptual):**

1. Load full row (all columns).
2. Test category → maybe discard.
3. Test revenue → maybe discard.
4. Repeat. You paid to load columns of discarded rows.

**Late style (conceptual):**

1. Scan only `category`. Keep positions where it equals Electronics — say 2% of rows.
2. At those positions only, scan `revenue`, keep `> 500` — say 0.1% of rows.
3. Only now fetch `order_id` and `revenue` for the tiny survivor set.

**Rule of thumb:** the more selective the WHERE, the bigger the win. A filter that keeps 99% of rows gives almost no late-materialization benefit — you materialize almost everything anyway.

### 2.4 Predicate pushdown — three layers of skipping

When you write `WHERE season = 2024 AND status = 'Finished'`, a good analytical stack tries to skip work at multiple levels:

1. **Partition / directory skip** — never open `season=2023/` folders at all.
2. **Row-group skip** — Parquet footer says this chunk’s max `revenue` is 10; your filter needs `revenue > 1000` → skip the chunk without reading data pages.
3. **Scan-time filter** — while reading the column values that remain, drop non-matches.

**SQL that helps vs hurts pushdown:**

- Helps: `WHERE season = 2024`, `WHERE created_at >= DATE '2024-01-01'`
- Often hurts: `WHERE UPPER(status) = 'FINISHED'`, wrapping columns in functions that hide the raw value from statistics, or forcing expression evaluation before stats can apply.

You still filter correctly either way — but one version lets the engine skip huge amounts of I/O, and the other forces more reading.

### 2.5 How to look at an analytical EXPLAIN (starter habit)

In Postgres you already look for Seq Scan vs Index Scan, join type, rows estimates.

In DuckDB / lake queries, train yourself to ask:

- Which columns were projected at the scan? (did we avoid `SELECT *`?)
- Did the filter get pushed into the scan, or sit as a separate filter above a fat scan?
- Were partitions / row groups skipped?

You are measuring **data avoided**, not only operator names.

### How Module 2 should change your thinking

Stop asking only “is my JOIN written cleverly?”  
Ask: **how many bytes and row groups did I force the engine to touch?**

**Go deeper:** [[02 Module 2 — Columnar Execution Internals]]  
Optional CPU/EXPLAIN deep dive later: [[03 Volcano vs Vectorized Execution Models]]

---

## Module 3 — Parquet: the file format under the lake

### 3.1 What Parquet is (and is not)

Parquet is an **open columnar file format** for structured data. It is the lingua franca of analytical lakes: Spark, DuckDB, BigQuery, Athena, Pandas, Arrow — almost everything can read it.

It was needed because:

- **CSV** is universal but untyped, poorly compressed, and forces you to parse text; reading one column still means scanning the whole line.
- **JSON** is flexible but fat and row-oriented.
- **Avro** is excellent for row-oriented streaming (Kafka), not for column scans.

Parquet aims at: columnar layout + embedded schema + compression + per-chunk statistics + portability.

**What it is not:**

- Not a database (no query engine inside the file).
- Not a transactional format (you don’t `UPDATE` one row in place; you rewrite files).
- Not always the right tool (streaming messages → Avro; human spreadsheet interchange → CSV at the edge).

### 3.2 Inside a Parquet file — enough structure to reason

```text
PARQUET FILE
├── Row group 1  (e.g. rows 1–122,880)
│     ├── column chunk: order_id   (encoded + compressed)
│     ├── column chunk: category
│     └── column chunk: revenue
├── Row group 2  (next slice of rows)
│     └── ...
└── FOOTER
      schema (names + types)
      per row group, per column:
        byte offset, min, max, null count, encoding, ...
```

**Row group** = horizontal slice of rows. Unit of parallelism and unit of skipping.  
**Column chunk** = one column’s bytes for that row group — readable independently (column pruning).  
**Footer** = map of the file. Engines read this first (small), then decide what data pages to touch.

**Query path in practice:**

1. Read footer.
2. Compare your WHERE to min/max per row group → skip impossible groups (zero data bytes).
3. For survivors, read only the column chunks your SELECT/WHERE need.

### 3.3 Why sort / cluster at write time matters

Statistics are only useful if ranges don’t all look the same.

```text
Unsorted (bad for skipping):
  Row group 1: revenue min=5,    max=9950
  Row group 2: revenue min=8,    max=9990
  WHERE revenue > 5000 → both groups might qualify → read both

Sorted / clustered by revenue (better):
  Row group 1: revenue min=5,    max=200
  Row group 2: revenue min=201,  max=800
  Row group 3: revenue min=801,  max=9990
  WHERE revenue > 5000 → maybe only group 3
```

Same query, different write layout → different I/O. Again: **layout is design**, not an afterthought.

### 3.4 Why `SELECT *` hurts more on Parquet

In a row store, the row is already glued together. In Parquet, each column is a separate chunk (and over S3, separate remote reads). Pulling 40 columns to display 3 is paying for 37 unnecessary remote/disk reads.

In pipelines, `SELECT *` also freezes schema coupling: new unused columns suddenly flow downstream and inflate storage.

**Habit:** project explicitly. `SELECT *` is fine for a quick peek on a laptop sample — not as the spine of a production analytical path.

### 3.5 CSV vs Parquet in real decisions

Use **Parquet** when you control the lake and will query repeatedly.  
Use **CSV** when a human or external tool requires it — ideally as an export at the boundary, not as your long-term analytical store.

### 3.6 Schema evolution — starter rules that prevent pain

- **Adding a column:** usually safe; old files behave as NULL for the new field.
- **Renaming / changing types casually:** breaks multi-file reads and downstream jobs.
- Prefer **add + deprecate** over in-place rename. Treat lake schema as a contract.

### How Module 3 should change your thinking

A Parquet file is not “just storage.” It is a **bundle of columns + a skip map (footer)**. How you write it (columns, sort, schema stability) decides how well readers can skip later.

**Go deeper:** [[04 Module 3 — Parquet & File Formats]]

---

## Module 4 — Partitioning: the directory tree is your coarse index

### 4.1 Hive-style partitions

Analytical lakes often use folder names as partition values:

```text
data/laps/
  season=2022/
    laps.parquet
  season=2023/
    laps.parquet
  season=2024/
    laps.parquet
```

With hive partitioning enabled, a query with `WHERE season = 2024` can **never open** the 2022 and 2023 directories. That is **partition pruning**.

Compare to Postgres:

- B-tree index → seek to specific row locations.
- Partition pruning → skip coarse physical chunks (whole days/seasons) before reading files.

Same goal (don’t touch irrelevant data). Different granularity and mechanism.

### 4.2 Choosing partition keys (this is a product decision)

Ask: **what do almost all analytical queries filter on?**

Usually time: `date`, `month`, `season`. Sometimes `region` + `date` (multi-level).

**Rule 1 — filter patterns, not org charts.**  
Partitioning by `team_name` because “that’s how the company is organized” is useless if every query filters by `race_date`.

**Rule 2 — cardinality must be manageable.**  
Tens to a few thousand partitions is often workable. Millions of partitions (`user_id=...` folders) creates metadata hell and tiny files.

**Rule 3 — match query granularity.**  
If everyone filters by day, yearly folders still force opening huge files that contain many irrelevant days (unless you also rely on row-group stats heavily). If everyone filters by year, daily partitions may be too fine.

**Rule 4 — avoid keys that churn with updates** as your only strategy; lakes prefer immutable files and partition replacement over row updates.

### 4.3 The small-files problem (you will meet this)

If ingest writes a new tiny Parquet every few seconds/minutes, you eventually get tens of thousands of 10–100 KB files. Then:

- Opening files and reading footers dominates runtime.
- Listing directories on object storage becomes expensive.
- Parallelism gets silly (more tasks than useful work).

**Intuition:** aim for larger files (commonly discussed bands are tens of MB up toward ~1 GB depending on system). Periodically **compact** many small files into fewer larger ones, often **sorting** during compaction so statistics skip better afterward.

### 4.4 Layout beats clever SQL

If the partition key is wrong, rewriting the query forever is a losing game. You rewrite the data. That is why partition design belongs **before** the lake is huge — at the first serious write path.

### How Module 4 should change your thinking

At write time: *What will queries filter on?* Put that in the path.  
At query time: *Did we skip folders?* If EXPLAIN shows we opened everything, either the predicate doesn’t match the layout or the layout is wrong.

**Go deeper:** [[05 Module 4 — Partitioning & Data Layout]]

---

## Module 5 — Analytical modeling: grain, facts, dimensions

### 5.1 Why 3NF fights analytics

Third normal form is excellent for write-correct OLTP apps: store each fact once, join when needed, avoid update anomalies.

Analytical questions look like:

```sql
SELECT d.month, p.category, SUM(f.revenue)
FROM fact_orders f
JOIN dim_date d    ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.month, p.category;
```

If “category” lives three joins away across large normalized tables, every dashboard pays that join tax. Analytical modeling **denormalizes on purpose** so filters and GROUP BYs hit wide dimension tables and a tall fact table with shallow joins.

This is not laziness. It is optimizing for a different access pattern.

### 5.2 Star schema in concrete terms

Imagine an orders analytics domain:

**Fact table `fact_order_lines`** (tall, narrow):

- `order_line_id`
- `date_key`, `customer_key`, `product_key` (foreign keys to dimensions)
- `quantity`, `revenue`, `discount` (measures)

**Dimension `dim_product`** (shorter, wider):

- `product_key`, `sku`, `name`, `category`, `brand`, `color`, …

**Dimension `dim_date`:**

- `date_key`, `full_date`, `month`, `quarter`, `year`, `is_weekend`, …

**Dimension `dim_customer`:**

- `customer_key`, `country`, `segment`, …

Shape: one fact in the center, dimensions around it — a **star**. Queries usually join fact → each dim once (one hop), not a deep snowflake of dims normalized into more dims (that can be correct, but usually slower and harder to use).

### 5.3 Grain — the most important sentence you will write

Before you invent columns, write:

> “One row in this fact table means ________.”

Examples:

- one completed **order line**
- one **lap** for one driver in one race
- one **daily account balance**
- one **page view event**

**Why it matters:** every `SUM` and `COUNT` assumes a meaning for “one row.” If half the table is raw events and half is already daily totals, `SUM(revenue)` double-counts or undercounts and nobody notices until a meeting goes wrong.

**Mixed grain** is the classic analytical modeling disaster.

**Prefer atomic grain** (the most detailed event you care about). Aggregate upward in queries or summary tables. Do not bake the wrong rollup into the fact “to save space” unless you deeply understand the trade-off.

### 5.4 Facts vs dimensions — a practical sorting test

Ask of each field:

- Is it a **measurement of an event** (amount, count, duration, flag of occurrence)? → usually **fact measure**
- Is it **context for slicing** (who, what, where, when, product attributes)? → usually **dimension attribute**
- Does this attribute **change over time**, and do historical reports need the old value? → you are in SCD territory (slowly changing dimensions)

**SCD starter intuition (enough to know it exists):**

- **Type 1:** overwrite — “current truth only,” history of the attribute is lost.
- **Type 2:** new dimension row with validity dates — preserves history (“customer was in segment A in 2023, B in 2024”).
- **Type 3:** keep previous value in an extra column — limited history.

You do not need to master all SCD types on day one. You need to notice when overwriting a dimension attribute would silently rewrite history.

### 5.5 Additive measures (quick but important)

`SUM(revenue)` across days usually works.  
`SUM(account_balance)` across days usually **does not** mean anything useful (balances are point-in-time).  
`AVG(something)` across already-averaged rows can be wrong without weights.

When you design measures, ask: *If I SUM this across time / across products, is the result meaningful?*

### How Module 5 should change your thinking

Model for the questions you ask (`GROUP BY` dimensions, aggregate measures), not for the OLTP foreign-key graph.  
Start every design with a **grain sentence**.

**Go deeper:** [[06 Module 5 — Star Schemas & Dimensional Design]]

---

## Module 6 — Ingestion: making data trustworthy

### 6.1 Ingest is not “copy bytes”

If you only move JSON from an API into a folder, you have transport — not a pipeline. Analytical ingest exists to make data **safe to trust** for aggregates that executives and APIs will treat as truth.

### 6.2 The batch mental model (extract → stage → validate → write)

```text
Source (API / Postgres / files)
        │
        ▼
   EXTRACT   — pull raw; prefer not to “business-transform” here
        │
        ▼
   STAGE     — land unchanged for inspection, replay, debugging
        │
        ▼
   VALIDATE  — counts, nulls, types, ranges, duplicates
        │
        ▼
   WRITE     — commit Parquet under the partition layout you designed
```

**Extract:** get bytes/rows. Keep it dumb when you can.  
**Stage:** a place where “what did we actually receive?” is answerable tomorrow.  
**Validate examples:**

- expected row count vs source count
- NOT NULL on keys and critical measures
- cast `time` strings to typed timestamps / integers once
- reject impossible ranges (negative lap times, future dates you don’t allow)

**Write:** only after checks pass, write columnar files into `season=2024/` (or `date=.../`).

Skipping staging is faster until one bad batch poisons a month of dashboards and you cannot replay.

### 6.3 Idempotency — run twice, same lake

Pipelines retry. Networks fail. If retry **appends** the same day again, every `SUM` doubles.

**Broken pattern:** append-only insert of the same partition forever.  
**Healthy pattern:** **replace the partition** for that key (delete or overwrite `date=2024-01-15/`, then write fresh files). Same inputs → same outputs.

Also plan for halfway failure:

1. Write to a temp path  
2. Validate  
3. Atomic move/rename into the final partition location  

Either the new partition appears complete, or the old one remains — not a half-written mix.

### 6.4 Late-arriving data (know it exists)

A row belonging to **Monday** may arrive **Wednesday**. If Monday’s partition was already closed and aggregated, your Monday dashboard is wrong unless you:

- reprocess a lookback window (last N days) every run, or
- track event time vs arrival time explicitly, or
- accept and document the lag

Ignoring late data is how “correct” pipelines still lose trust.

### 6.5 Types, nulls, timestamps — where silent corruption lives

Examples that break analytics quietly:

- `"42"` left as string → joins and groupings behave oddly; some engines cast inconsistently.
- `""` vs `NULL` for missing status.
- timestamps without timezone discipline (`2024-01-15 00:00:00` — in which zone?).
- mixing ms and seconds for durations.

**Fix at ingest with explicit casts and normalization.** “We’ll clean it in the report query” becomes ten slightly different cleanings and ten slightly different numbers.

### 6.6 Raw → cleaned → curated (as a thinking tool)

Not a mandatory product name — a way to decide guarantees:

- **Raw:** as received (possibly ugly, possibly complete).
- **Cleaned:** typed, validated, deduped — safe to compute on.
- **Curated:** shaped for analytics (facts/dims, partitions, maybe summaries).

Ask: *What do I promise is true at this layer?* If you can’t answer, you don’t have a layer — you have a folder.

### How Module 6 should change your thinking

Every step is a checkpoint against wrong totals.  
Before shipping an ingest job, ask: **If this retries tonight, do my SUMs stay correct?**

**Go deeper:** [[07 Module 6 — Ingestion Thinking]]

---

## Module 7 — DuckDB: analytical SQL inside your process

### 7.1 What DuckDB is in practice

DuckDB is an **embedded columnar analytical engine**. Deployment feeling is a bit like SQLite (library in-process). Execution feeling is OLAP (vectorized, great at scanning Parquet).

**Good at:**

- local and embedded analytics
- reading Parquet (and other files) directly with SQL
- interactive exploration and building summary tables
- powering an API that answers analytical questions without standing up a warehouse cluster

**Not a replacement for:**

- your OLTP system of record (Postgres) — no MVCC app lifestyle, different concurrency model
- a multi-node Spark cluster when data truly exceeds one machine’s practical working set and you need distributed execution/fault tolerance

### 7.2 Persistence models you will actually use

**Ephemeral:** open DuckDB in-memory, `read_parquet(...)`, aggregate, return results, discard. Great for jobs and many API requests over an immutable lake.

**Persistent `.duckdb` file:** store tables, views, small dimensional data, pre-aggregated summaries that survive process restart.

**Hybrid (very common):** Parquet lake remains source of truth for large facts; DuckDB holds views over those files + materialized summary tables for hot endpoints.

### 7.3 Querying a lake (mental SQL)

Conceptually:

```sql
SELECT driver_id, AVG(time_ms) AS avg_lap_ms
FROM read_parquet('data/laps/*/*.parquet', hive_partitioning := true)
WHERE season = 2024
GROUP BY driver_id
ORDER BY avg_lap_ms;
```

What should happen if layout is right:

- only `season=2024` files are considered (partition prune)
- only needed columns are read from those Parquets
- aggregation runs vectorized in-process

You can also `CREATE VIEW laps AS SELECT * FROM read_parquet(...)` so application SQL looks normal while bytes stay in files until queried.

### 7.4 Single-writer constraint (design implication)

A DuckDB database file is not “Postgres with unlimited writers.” Plan NestJS / workers so **one writer** owns mutations to a given `.duckdb` file. Many analytical setups avoid concurrent writers by:

- treating Parquet as the write path (immutable files), and
- using DuckDB mainly for reads + controlled rebuilds of summaries

If you ignore this, you get surprising lock / write conflicts that look like application bugs.

### 7.5 Where it sits vs Postgres vs Spark (decision sketch)

- Transactional app, point lookups, concurrent writes → **Postgres**
- Analytical scans on files / embedded in an app / laptop-to-single-server → **DuckDB**
- Multi-TB distributed jobs, cluster fault tolerance → **Spark / warehouse**, not “force DuckDB”

### How Module 7 should change your thinking

DuckDB is a **query engine over files you control**, sitting next to your service — not a second OLTP database you accidentally invent.

**Go deeper:** [[08 Module 7 — DuckDB in Practice]]

---

## Module 8 — Query patterns that matter once data is in place

### 8.1 Aggregations and GROUP BY design

Analytical SQL spends its life in `GROUP BY`. Design it deliberately.

```sql
-- Good instinct: prune partitions + reduce rows before grouping when possible
SELECT driver_id, AVG(time_ms)
FROM laps
WHERE season = 2024
GROUP BY driver_id;
```

Things that hurt:

- Grouping by extremely high-cardinality keys without need (`user_id` × `timestamp_second`) → enormous intermediate state.
- Filtering *after* building a huge aggregation when the filter could have been applied earlier.
- Selecting columns you don’t need (forces more I/O before the aggregate).

Also remember: aggregating **already aggregated** data can be wrong (averages of averages). Know whether your input grain is raw or summary.

### 8.2 Window functions — when they shine

`GROUP BY` collapses rows.  
Window functions **keep the row grain** and add columns like running averages, ranks, lags.

Examples of the right job:

- rolling 5-race average finish for a driver (“is form real?”)
- rank of lap time within a race
- difference vs previous race / vs race 1

```sql
AVG(finish_position) OVER (
  PARTITION BY driver_id
  ORDER BY season, round
  ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
) AS rolling_5_avg_finish
```

**Cost reality:** windows still scan and sort/partition data. They are not free. Don’t use a window where a simple aggregate answer is enough. Used well, they scan once and answer “form over time” questions that are painful as self-joins.

### 8.3 Materialized aggregates / summary tables

If an API asks for season standings every request, scanning every result row every time is wasteful.

Pattern:

```text
raw / curated facts  →  nightly (or post-ingest) job  →  summary table
                                                      →  API reads summary
```

Example summary grain: one row per `(season, driver_id)` with `points`, `wins`, `podiums`, `dnfs`, `avg_finish`.

**Trade-off:** speed vs freshness. The summary is only as fresh as the last rebuild. Incremental refresh is an optimization you learn when full rebuilds become too slow — not a day-one requirement.

Summaries do **not** replace good modeling. They accelerate good models.

### 8.4 Approximate vs exact

`COUNT(DISTINCT user_id)` over huge data can be expensive. Approximations (HyperLogLog-style `approx_count_distinct` in many engines) are often ~99% accurate and much faster.

Use approx for exploration, product dashboards where exact distinct is not contractual.  
Use exact when money, compliance, or billing depends on the number.

### 8.5 Benchmarking without fooling yourself

**Cold run:** nothing in OS page cache / engine cache — closer to “first query after deploy” or large uncached lake read.  
**Warm run:** data already in memory — measures CPU/execution more than disk layout.

If you compare Parquet layouts but only look at warm runs, you may miss the I/O win that was the whole point. Restart the process / drop caches when layout is what you’re testing. And always ask: *faster at what — first hit, or steady dashboard traffic?*

### How Module 8 should change your thinking

Order of attack for slow analytics:

1. Skip partitions and unused columns  
2. Confirm grain / model is right  
3. Choose GROUP BY vs window correctly  
4. Materialize hot paths  
5. Only then micro-tune SQL expressions  

**Go deeper:** [[09 Module 8 - Query Patterns That Matter in OLAP]]

---

## End-to-end story (tie it together)

Question: *average lap time by driver for the 2024 season.*

1. **Module 1:** This is scan-heavy aggregation → OLAP path, not “index harder in OLTP.”  
2. **Module 3:** Store laps as Parquet columns (`time_ms`, `driver_id`, …), not as perpetual CSV.  
3. **Module 4:** Land under `season=2024/` so other seasons never open.  
4. **Module 5:** Grain = one row means one driver lap in one race. Facts hold measures; dims (if any) hold context.  
5. **Module 6:** Extract from API → stage → validate types → idempotent replace of the 2024 partition.  
6. **Module 7:** DuckDB `read_parquet` with hive partitioning; maybe a view `laps`.  
7. **Module 8:** `WHERE season = 2024` + `GROUP BY driver_id` + `AVG(time_ms)`; if the API is hot, materialize a season summary.  
8. **Module 2:** `EXPLAIN` should show other seasons skipped and only needed columns read.

That chain is OLAP thinking in practice.

---

## How to study this without drowning

1. Read this file fully once (don’t jump to every deep module yet).  
2. For each module section, explain the core idea out loud in your own words.  
3. Open **Go deeper** only when something still feels fuzzy or you hit that problem while building.  
4. When you start a project: apply Modules 5–7 immediately (grain, ingest, DuckDB), then Module 8 for API/query shape.  
5. Save [[03 Volcano vs Vectorized Execution Models]] for when you want EXPLAIN/CPU depth.

### Can you explain these?

- Seek vs scan; why a PK doesn’t save `SUM` over a year  
- Why reading 2 of 40 columns is ~orders of magnitude cheaper columnar  
- What a Parquet footer enables  
- Why `season=YYYY/` folders act like an index  
- A grain sentence for your domain  
- Why append-on-retry corrupts totals  
- What DuckDB is good/bad at vs Postgres  
- When to materialize a summary vs scan raw facts  

---

## Checklist before you write analytical SQL

- [ ] Is this a lookup (OLTP) or a scan (OLAP)?  
- [ ] What is the grain of one fact row — in one written sentence?  
- [ ] Which fields are measures vs dimensions?  
- [ ] What will almost every query filter on → partition key?  
- [ ] Are we storing Parquet for the lake (not CSV as system of record)?  
- [ ] Is re-ingest idempotent for that partition?  
- [ ] Does the query project only needed columns?  
- [ ] Does EXPLAIN show the skips you expect?  
- [ ] Is this question asked so often it needs a summary table?

---

**Parent:** [[00 OLAP MOC]] · [[Concepts MOC]]
