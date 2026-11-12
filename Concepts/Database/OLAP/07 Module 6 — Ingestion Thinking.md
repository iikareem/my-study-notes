# Module 6 — Ingestion Thinking

[[#Module 5 — Analytical Data Modeling|← Module 5]] · [[#Table of Contents|↑ Table of Contents]] · [[#Module 7 — DuckDB in Practice|Next: Module 7 →]]

---

## Module 6 Summary

> **One-liner:** Ingestion is not moving data — it is making data trustworthy enough to query. Every step exists to catch a problem before it silently corrupts your numbers.

**Most important takeaways:**

1. **Extract → stage → validate → write** — Never write raw source data directly to your curated layer. Each step is a checkpoint.
2. **Idempotency** — Every pipeline run must produce the same result if run twice on the same input. If a re-run creates duplicates or overwrites good data, the pipeline is broken.
3. **Late-arriving data** — Records arrive after the period they belong to. Design for it explicitly or your aggregates will silently be wrong.
4. **Staging vs writing Parquet directly** — Staging gives you a place to validate before committing. Skipping it is faster but removes your safety net.
5. **Types, nulls, timestamps** — The most common place pipelines silently corrupt data. Explicit casting at ingest time costs almost nothing; fixing it downstream costs a lot.
6. **Raw → cleaned → curated** — Three layers of data maturity. Not a product, not a framework — a thinking tool that forces you to decide what you guarantee at each stage.

**Jump to:** [[#6.1 Batch ingest mental model|6.1]] · [[#6.2 Idempotency and re-runs|6.2]] · [[#6.3 Late-arriving data|6.3]] · [[#6.4 Staging vs writing Parquet directly|6.4]] · [[#6.5 Types, nulls, timestamps, timezones|6.5]] · [[#6.6 Raw → cleaned → curated|6.6]] · [[#6.7 Postgres → Ingestion Translation Table|6.7 Translation]]

---

## 6.1 Batch ingest mental model

The mental model for batch ingestion is a linear pipeline with four stages. Each stage has one job. Mixing jobs across stages is where pipelines become hard to debug.

```
Source
  │
  ▼
┌─────────────┐
│   EXTRACT   │  Pull data from the source. Do nothing else.
└──────┬──────┘
       │
  ▼
┌─────────────┐
│    STAGE    │  Land raw data somewhere you control. Unchanged.
└──────┬──────┘
       │
  ▼
┌─────────────┐
│  VALIDATE   │  Check types, nulls, ranges, row counts. Fail loudly.
└──────┬──────┘
       │
  ▼
┌─────────────┐
│    WRITE    │  Write clean, typed columnar files. Commit to the dataset.
└─────────────┘
```

### Extract — pull only, transform nothing

The extract step connects to the source (Postgres, an API, a CSV dump) and pulls data. Its only job is retrieval. No type casting, no filtering, no renaming columns.

```python
# Extract: pull from Postgres, write raw CSV or JSON to staging area
rows = pg_conn.execute("SELECT * FROM orders WHERE created_at >= %s", [since]).fetchall()
write_to_staging(rows, 'staging/orders/2024-01-15.csv')
```

Why no transformation here: if the extract fails halfway through, you want to re-run it cleanly. If you transform during extract, a partial run leaves transformed-some, raw-the-rest data that is hard to recover from.

### Stage — land raw data, unchanged

The staging area is a temporary holding zone where you have a copy of exactly what the source sent — no more, no less. This is your rollback point. If validation fails, you fix the pipeline and re-run against the staged data. You do not need to re-hit the source.

```
staging/
└── orders/
    └── 2024-01-15.csv   ← raw, exactly as extracted
```

### Validate — check before committing

This is the step most pipelines skip and most pipelines regret skipping. Validation runs against the staged data and fails loudly if something is wrong.

Minimum checks:

```python
# Row count — did we get what we expected?
assert row_count > 0, "Empty extract — source issue or wrong date range"
assert row_count >= yesterday_count * 0.8, "Row count dropped >20% — investigate"

# Nulls in columns that must not be null
assert null_count('order_id') == 0, "order_id has nulls — schema issue upstream"

# Type sanity
assert all values in 'revenue' are numeric, "revenue has non-numeric values"

# Range sanity
assert min('revenue') >= 0, "Negative revenue — bad data or currency conversion bug"
```

If any check fails: stop. Do not write to the curated layer. Page someone. The goal is to make bad data loud, not silent.

### Write — commit clean columnar files

Only after validation passes do you write to your Parquet dataset. At this point you apply types, cast columns, rename fields to match your schema, and write the file into the correct partition.

```sql
-- DuckDB: cast, rename, and write in one step
COPY (
    SELECT
        CAST(order_id AS BIGINT)           AS order_id,
        CAST(revenue AS DOUBLE)            AS revenue,
        CAST(created_at AS TIMESTAMPTZ)    AS created_at,
        created_at::DATE                   AS order_date   -- partition key
    FROM read_csv('staging/orders/2024-01-15.csv')
    WHERE order_id IS NOT NULL
)
TO 'data/orders/order_date=2024-01-15/part-0.parquet'
(FORMAT PARQUET, COMPRESSION 'zstd');
```

---

## 6.2 Idempotency and re-runs

**Idempotency** means: running the pipeline twice on the same input produces the same output. Not double the rows, not a conflict error — the same result.

This is not optional. Pipelines fail. Networks drop. Servers restart. You will re-run pipelines. If re-runs are not safe, every failure becomes a data integrity incident.

### The duplicate problem

The most common idempotency failure:

```python
# ❌ Not idempotent — appends on every run
def run(date):
    rows = extract(date)
    append_to_parquet(rows, f'data/orders/date={date}/')
    # Run twice → double the rows for that date
```

### The fix: delete-then-write (partition replacement)

```python
# ✅ Idempotent — replaces the partition atomically
def run(date):
    rows = extract(date)
    validate(rows)
    delete_partition(f'data/orders/order_date={date}/')   # remove old files
    write_parquet(rows, f'data/orders/order_date={date}/part-0.parquet')
    # Run twice → same files, same rows
```

Because each run replaces the entire partition for that date, re-runs are always safe. The partition is either the old version or the new version — never both.

### In DuckDB

```sql
-- Step 1: delete existing files for this partition (shell or Python)
-- Step 2: write fresh
COPY (
    SELECT * FROM read_csv('staging/orders/2024-01-15.csv')
) TO 'data/orders/order_date=2024-01-15/part-0.parquet' (FORMAT PARQUET);
```

### Idempotency checklist

|Question|What to check|
|---|---|
|What happens if I run this twice?|Same output, no duplicates|
|What happens if it fails halfway?|Partial output doesn't corrupt the existing partition|
|What happens if I run it on old data?|Old partition is cleanly replaced|
|Does it depend on previous run state?|It shouldn't — each run should be self-contained|

### Handling the "halfway fail" case

Write to a temp location first, then move atomically:

```python
# Write to temp
write_parquet(rows, 'tmp/orders/2024-01-15/part-0.parquet')
validate_output('tmp/orders/2024-01-15/part-0.parquet')

# Atomic move — either succeeds fully or not at all
delete_partition('data/orders/order_date=2024-01-15/')
move('tmp/orders/2024-01-15/', 'data/orders/order_date=2024-01-15/')
```

A failed run leaves the temp directory behind (easy to clean up) and the live partition untouched.

---

## 6.3 Late-arriving data

**Late-arriving data** means a record arrives after the time window it belongs to has already been processed.

Example: an order placed at 23:58 on January 15 is logged to the source database at 00:03 on January 16 due to network delay, retry logic, or async processing. Your January 15 pipeline already ran. That order is now missing from January 15's partition.

This is not an edge case. It is the normal state of distributed systems.

### Why it breaks naive pipelines

```
Naive pipeline: "extract everything where created_at::date = today"

Run at 01:00 on Jan 16 for Jan 15 data:
  → extracts orders with created_at between 00:00–23:59 Jan 15
  → writes data/orders/order_date=2024-01-15/

Late order arrives at 00:03 Jan 16 with created_at = Jan 15 23:58
  → it is now in the source, but your Jan 15 pipeline already ran
  → it never makes it into order_date=2024-01-15/
  → January 15's revenue is silently understated
```

### Strategy 1: Reprocess a lookback window

Instead of processing only yesterday, always reprocess the last N days. Re-running is safe because your pipeline is idempotent.

```python
# Always reprocess the last 3 days
for date in [today - 3, today - 2, today - 1]:
    run_pipeline(date)
```

This catches most late arrivals (data that arrives within 3 days of the event date). The cost is re-reading and re-writing 3 partitions per run instead of 1. Usually worth it.

Choose N based on your source system's observed latency. If late data rarely arrives more than 24 hours late, N=2 is sufficient.

### Strategy 2: Separate event time from arrival time

Store both timestamps and partition on arrival time, not event time:

```
order_date (event time):   2024-01-15  ← when the order happened
ingested_date (arrival):   2024-01-16  ← when we received it
```

Partition by `ingested_date`. Each pipeline run processes what arrived today — no records are ever "late" because you're partitioning on when you saw them, not when they happened.

The trade-off: queries like "all orders on Jan 15" now require scanning multiple partitions (the Jan 15 partition and any later partitions that contained late Jan 15 records). You gain pipeline simplicity; you pay in query complexity.

### Strategy 3: Accept the gap and document it

For some use cases, late arrivals within a short window are acceptable. Revenue reports run 24 hours after the fact, so a 3-minute late order is immaterial. Document the latency guarantee ("numbers are final after 48 hours") and move on. Don't over-engineer for a problem that doesn't affect business decisions.

---

## 6.4 Staging vs writing Parquet directly

The staging step feels like overhead. It adds latency, uses disk space, and requires cleanup. Here is when it is worth it and when it is not.

### The case for staging

**Decouples extract from transform.** If the transform step fails, you re-run it against the staged file — you do not re-hit the source. For APIs with rate limits or Postgres instances under load, this matters.

**Gives you a raw audit trail.** Staging preserves exactly what the source sent. If someone asks "what did the source actually give us on January 15?" you have the answer without reconstructing it.

**Enables validation before commit.** You validate the staged file before writing to the curated layer. A failed validation doesn't corrupt your Parquet dataset — it just leaves the staging file behind.

**Cheaper re-runs.** Re-running a transform against a local CSV is faster and cheaper than re-extracting from a production database or a rate-limited API.

### When to skip staging

- The source is a file (S3, SFTP) — it already is staging.
- The pipeline is simple and the source is always available for re-extraction.
- Data volume is very small and re-extraction is trivially cheap.
- You are prototyping. Add staging before going to production.

### Staging as a table vs staging as a file

||Staging table (Postgres / DuckDB)|Staging file (CSV / JSON on disk)|
|---|---|---|
|Validation|SQL queries against the table|DuckDB `read_csv` or Python|
|Cleanup|`TRUNCATE` or `DELETE`|Delete the file|
|Schema enforcement|Yes — table has types|No — everything is a string in CSV|
|Best for|Complex validation, joins to reference data|Simple pipelines, file-based sources|

A Postgres staging table is useful when you need to validate against reference data:

```sql
-- Check that all product_ids in the staged data exist in the products table
SELECT s.product_id
FROM staging_orders s
LEFT JOIN products p ON s.product_id = p.id
WHERE p.id IS NULL;
-- Any rows here = unknown product_ids = fail the pipeline
```

---

## 6.5 Types, nulls, timestamps, timezones

This is where most pipeline bugs live. The source sends you something; you assume it means one thing; it means another. The data loads, no error is thrown, and your numbers are wrong.

### Types

Source systems — especially APIs and CSVs — send everything as strings. You must cast explicitly. Never trust implicit casting.

```sql
-- ❌ Implicit — DuckDB guesses; it might guess wrong
SELECT revenue FROM read_csv('orders.csv');

-- ✅ Explicit — you control the type
SELECT CAST(revenue AS DOUBLE) AS revenue
FROM read_csv('orders.csv', columns={'revenue': 'VARCHAR'});
```

Common type traps:

|Source sends|You assume|Reality|
|---|---|---|
|`"1,234.56"`|DOUBLE|String with comma — cast fails or returns NULL|
|`"true"` / `"1"` / `"yes"`|BOOLEAN|Inconsistent across sources — normalize explicitly|
|`"2024-01-15"`|DATE|Fine — but `"01/15/2024"` is not, and both appear in the wild|
|`""` (empty string)|NULL|Not NULL in CSV — it's an empty string. `WHERE id IS NULL` won't catch it|

### Nulls

Empty strings and NULLs are different. CSV has no native NULL representation — different sources use `""`, `"NULL"`, `"N/A"`, `"\N"`. Decide at ingest what counts as NULL and enforce it:

```sql
SELECT
    CASE WHEN revenue = '' OR revenue = 'N/A' THEN NULL
         ELSE CAST(revenue AS DOUBLE)
    END AS revenue
FROM read_csv('orders.csv');
```

For columns that must not be null, fail the pipeline loudly:

```sql
-- Validate after casting
SELECT COUNT(*) FROM staged_orders WHERE order_id IS NULL;
-- If > 0: raise an error, stop the pipeline
```

### Timestamps

Timestamp handling is where pipelines most commonly produce wrong numbers silently.

**Always store timestamps with timezone (`TIMESTAMPTZ`).** A timestamp without timezone is ambiguous — `2024-01-15 23:00:00` means something different depending on where the server is.

```sql
-- ❌ No timezone — ambiguous
CAST(created_at AS TIMESTAMP)

-- ✅ With timezone — unambiguous
CAST(created_at AS TIMESTAMPTZ)
```

**Normalize to UTC at ingest.** Whatever timezone the source uses, convert to UTC when writing to your analytical layer. Do timezone conversion once, at the boundary. Everything downstream works in UTC.

```sql
-- Source is in US Eastern time
CAST(created_at AS TIMESTAMPTZ) AT TIME ZONE 'America/New_York' AT TIME ZONE 'UTC'
```

**Partition keys are dates in UTC.** If you partition by `order_date`, derive it from the UTC timestamp:

```sql
-- ❌ Local date — shifts records across partitions depending on server timezone
created_at::DATE AS order_date

-- ✅ UTC date — consistent regardless of where the pipeline runs
(CAST(created_at AS TIMESTAMPTZ) AT TIME ZONE 'UTC')::DATE AS order_date
```

A pipeline that derives `order_date` from local time will silently move records between partitions when the server changes timezone or daylight saving time kicks in.

### Timezone checklist

|Question|What to enforce|
|---|---|
|Does the source include timezone info?|If not, ask. Assume UTC only if documented.|
|Is the pipeline portable across servers in different timezones?|Use UTC everywhere so it doesn't matter.|
|Are partition date keys derived from UTC?|Yes — always.|
|Are timestamps stored as TIMESTAMPTZ?|Yes — always.|

---

## 6.6 Raw → cleaned → curated

This is a thinking tool, not a product or a framework. It forces you to be explicit about what you guarantee at each stage of your data.

### The three layers

```
┌─────────────────────────────────────────────────────────┐
│  RAW (Bronze)                                           │
│  Exact copy of what the source sent.                    │
│  No transforms, no type casting, no filtering.          │
│  You guarantee: completeness, immutability.             │
│  You do NOT guarantee: correctness, types, no nulls.    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  CLEANED (Silver)                                       │
│  Types cast, nulls handled, duplicates removed,         │
│  timestamps normalized to UTC, schema enforced.         │
│  You guarantee: correct types, no structural issues.    │
│  You do NOT guarantee: business logic correctness.      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  CURATED (Gold)                                         │
│  Business logic applied: joins, aggregations,           │
│  metric definitions, dimensional models.                │
│  You guarantee: numbers match business definitions.     │
│  Analysts and dashboards query this layer.              │
└─────────────────────────────────────────────────────────┘
```

### Why the layers matter

**Debugging.** When a number is wrong, the layers tell you where to look. Is the raw data wrong (source problem)? Is the cleaned data wrong (type casting or dedup problem)? Is the curated data wrong (business logic problem)? Without distinct layers, you have one big blob of "the data" and no way to isolate the fault.

**Re-processing.** If your business logic changes, you re-build the curated layer from the cleaned layer. You do not re-extract from the source. If your cleaning logic changes, you re-build cleaned from raw. You always have a fallback.

**Auditability.** Raw is your receipt. Whatever a source system says it sent you, you have it. If there is ever a dispute about what the source sent on January 15, you open the raw file.

### What each layer looks like on disk

```
data/
├── raw/
│   └── orders/
│       └── 2024-01-15.csv          ← exactly as extracted, no changes
│
├── cleaned/
│   └── orders/
│       └── order_date=2024-01-15/
│           └── part-0.parquet      ← typed, nulls handled, UTC timestamps
│
└── curated/
    └── fact_orders/
        └── order_date=2024-01-15/
            └── part-0.parquet      ← joined to dims, business logic applied
```

### What the layers are not

- **Not mandatory folder names.** Call them whatever makes sense. `landing / staging / marts`. `ingest / transform / serve`. The names don't matter; the guarantees do.
- **Not always three separate pipeline steps.** A simple pipeline might go raw → curated in one step. The mental model still applies — you should be able to say what you guarantee at each point.
- **Not a product.** Databricks calls them bronze/silver/gold. dbt calls them staging/intermediate/marts. These are marketing names for the same idea. Understand the concept; don't buy the branding.

### The guarantee question

For every dataset you own, you should be able to answer: **"What do I guarantee about this data?"**

- Raw: I guarantee it is what the source sent. Nothing else.
- Cleaned: I guarantee the types are correct and timestamps are UTC. I do not guarantee the revenue numbers are business-correct.
- Curated: I guarantee that `SUM(revenue)` here matches the definition in the company's revenue policy document.

If you cannot answer that question, the layer boundary is not real yet.

---

## 6.7 Postgres → Ingestion Translation Table

|Postgres concept|Ingestion equivalent|Key difference|
|---|---|---|
|`COPY FROM` (bulk load)|Extract → stage → validate → `COPY TO parquet`|OLAP adds a validation gate before writing|
|Staging table (`_stg` suffix)|Staging file (CSV/JSON) or staging table in DuckDB|Same idea; file-based staging needs manual cleanup|
|`INSERT ... ON CONFLICT DO UPDATE` (upsert)|Delete partition → rewrite partition|OLAP has no row-level upsert; replace the whole partition|
|`TRANSACTION` (atomic commit)|Write to tmp → move to live|Same goal: either fully committed or fully rolled back|
|`NOT NULL` constraint|Explicit null check in validation step|OLAP has no runtime constraint enforcement — you enforce it|
|`CHECK` constraint|Validation assertions in pipeline code|Same idea; enforced at ingest time, not query time|
|`created_at TIMESTAMPTZ`|Always `TIMESTAMPTZ`, cast at ingest|The lesson is the same; OLAP pipelines just break it more visibly|
|`pg_dump` / WAL archiving|Raw layer (immutable copy of source data)|Raw files serve the same audit/recovery purpose|
|Materialized view refresh|Rebuild curated layer from cleaned layer|Same idea: re-derive downstream from upstream|
|`pg_cron` scheduled job|External scheduler (cron, Airflow, Dagster)|DuckDB/Parquet pipelines need an external trigger|

---

[[#Module 6 Summary|↑ Module 6 Summary]] · [[#Table of Contents|↑ TOC]] · [[#Module 7 — DuckDB in Practice|Next: Module 7 →]]

---

_End of Module 6._

> **Suggested next steps:**
> 
> - **Module 7 — DuckDB in Practice:** Connect ingestion (Module 6) to the query engine — reading the cleaned and curated Parquet layers with DuckDB, views over file sets, and integrating into an application.
> - **Module 8 — ETL & Incremental Loads:** Handling append-only vs full-refresh patterns, compaction after incremental writes, and managing partition evolution over time.