# Module 7 — DuckDB in Practice

[[#Module 5 — Analytical Data Modeling|← Module 5]] · [[#Table of Contents|↑ Table of Contents]]

---

## Module 7 Summary

> **One-liner:** DuckDB is an embedded columnar engine — it runs inside your process, queries Parquet files directly, and replaces a surprising amount of what you'd otherwise send to a separate analytics server.

**Most important takeaways:**

1. **Embedded, not a server** — DuckDB runs in-process like SQLite. No server to deploy, no connection pool, no network hop. One process owns the database at a time.
2. **Reads Parquet natively** — You can query `.parquet` files (local or S3) directly with SQL. No loading step, no ETL.
3. **Persistence is optional** — Use a `.duckdb` file to persist tables, or query files ephemerally and store nothing. Both are valid workflows.
4. **Single-writer model** — One process can write at a time. Multiple readers are fine. Know this before designing your application around it.
5. **Not Spark** — DuckDB is single-node. It wins on simplicity and low latency. Spark wins when data exceeds one machine's RAM or you need distributed fault tolerance.
6. **Not Postgres** — DuckDB has no MVCC, no row-level locking, no connection pool, no `pg_` system catalog. Don't treat it as a drop-in replacement.

**Jump to:** [[#7.1 What DuckDB is and where it fits|7.1]] · [[#7.2 DuckDB vs Postgres vs Spark|7.2]] · [[#7.3 Reading Parquet directly|7.3]] · [[#7.4 Views over files|7.4]] · [[#7.5 Persistence vs ephemeral queries|7.5]] · [[#7.6 Useful settings and pragmas|7.6]] · [[#7.7 Integrating DuckDB into an application|7.7]] · [[#7.8 Postgres → DuckDB Translation Table|7.8 Translation]]

---

## 7.1 What DuckDB is and where it fits

DuckDB is an **embedded, in-process OLAP database**. The closest mental model is SQLite — but built for analytical queries instead of transactional ones.

- **Embedded** means it runs inside your application process, not as a separate server. There is no `duckdb start`, no port to connect to, no daemon.
- **In-process** means zero network overhead. The query runs in the same memory space as your application.
- **OLAP** means it is columnar, vectorized, and optimized for aggregations over many rows — not for concurrent row-level writes.

### What it is good at

- Aggregating millions to billions of rows on a single machine
- Querying Parquet files directly without loading them into a database first
- Local analytics workflows — data exploration, ETL, report generation
- Replacing pandas for large tabular data (DuckDB is faster and uses less memory)
- Being embedded in a Python, Go, Node, or Java application with no infrastructure

### What it is bad at

- High-concurrency transactional writes (use Postgres for that)
- Data that exceeds a single machine's disk/RAM at query time (use Spark or a cloud warehouse)
- Multiple processes writing simultaneously (single-writer constraint)
- Row-level operations: `UPDATE WHERE id = 42` exists but is not its strength

---

## 7.2 DuckDB vs Postgres vs Spark

|Dimension|DuckDB|Postgres|Spark|
|---|---|---|---|
|**Architecture**|Embedded, in-process|Client-server|Distributed cluster|
|**Deployment**|Zero — ships as a library|Server + connection pool|Cluster manager (Kubernetes, EMR, Databricks)|
|**Storage model**|Columnar|Row (heap)|Columnar (Parquet, Delta, Iceberg)|
|**Execution**|Vectorized, single-node|Volcano, single-node|Vectorized, distributed|
|**Concurrency**|Single writer, many readers|Many writers, many readers (MVCC)|Many writers via Spark coordinator|
|**Query latency**|Milliseconds to seconds|Microseconds (indexed) to minutes (full scan)|Seconds to minutes (JVM startup + shuffle overhead)|
|**Data scale**|Up to ~1–5 TB on one machine|Up to ~hundreds of GB comfortably|Petabytes|
|**Operational cost**|Zero|Low–medium|High|
|**Best for**|Local analytics, embedded reporting, file querying|Transactional apps, operational data|Large-scale batch processing, multi-node ETL|

### The practical decision

- **< 100 GB, analytical queries** → DuckDB. No contest.
- **Transactional + small analytical** → Postgres. Maybe add DuckDB for the analytical slice.
- **> 1 TB or multi-node required** → Spark or a cloud warehouse (BigQuery, Redshift, Snowflake).
- **Both transactional and large analytical** → Postgres for writes, DuckDB or cloud warehouse for reads.

---

## 7.3 Reading Parquet directly

This is DuckDB's most practically useful feature. You do not need to load data into DuckDB to query it.

### Basic file query

```sql
-- Query a local Parquet file directly
SELECT COUNT(*), SUM(revenue)
FROM read_parquet('data/orders/2024.parquet');

-- Glob multiple files
SELECT COUNT(*)
FROM read_parquet('data/orders/*.parquet');

-- Glob across directories
SELECT COUNT(*)
FROM read_parquet('data/orders/**/*.parquet');
```

### Hive-partitioned files

```sql
-- With hive_partitioning=true, folder names become columns
SELECT date, SUM(revenue)
FROM read_parquet('data/orders/**/*.parquet', hive_partitioning=true)
WHERE date >= '2024-01-01'
GROUP BY date
ORDER BY date;
```

DuckDB reads only the folders matching the `WHERE date` predicate. Everything else is skipped at the filesystem level before any file is opened.

### Reading from S3

```sql
-- Install and load the httpfs extension (once per session)
INSTALL httpfs;
LOAD httpfs;

SET s3_region = 'us-east-1';
SET s3_access_key_id = '...';
SET s3_secret_access_key = '...';

SELECT COUNT(*)
FROM read_parquet('s3://your-bucket/data/orders/**/*.parquet', hive_partitioning=true)
WHERE date = '2024-01-15';
```

DuckDB fetches only the footer first (to read row group statistics), then fetches only the column chunks it needs. It does not download the full file.

### Writing Parquet

```sql
-- Write query results to a Parquet file
COPY (
    SELECT customer_id, SUM(revenue) AS total_revenue
    FROM read_parquet('data/orders/*.parquet')
    GROUP BY customer_id
) TO 'output/customer_revenue.parquet' (FORMAT PARQUET);

-- With explicit row group size and compression
COPY (SELECT * FROM orders ORDER BY order_date)
TO 'output/orders_sorted.parquet'
(FORMAT PARQUET, ROW_GROUP_SIZE 200000, COMPRESSION 'zstd');
```

---

## 7.4 Views over files

Instead of writing `read_parquet('data/orders/**/*.parquet', hive_partitioning=true)` in every query, create a view over the files. The view holds no data — it's just a named query that runs against the files each time.

```sql
-- Create a view over the file set
CREATE VIEW orders AS
SELECT *
FROM read_parquet('data/orders/**/*.parquet', hive_partitioning=true);

-- Now query it like a table
SELECT date, SUM(revenue)
FROM orders
WHERE date >= '2024-01-01'
GROUP BY date;
```

### Why views over files are useful

- **No loading step.** Files on disk are the source of truth. The view is just a pointer.
- **Schema is always fresh.** If you add a new partition directory, the view sees it automatically.
- **Pushdown still works.** DuckDB pushes predicates through views to the underlying `read_parquet` call. Partition pruning and row group skipping still happen.
- **Composable.** You can join two views backed by different file sets as if they were tables.

```sql
-- Join two file-backed views
CREATE VIEW customers AS
SELECT * FROM read_parquet('data/customers/*.parquet');

SELECT c.country, SUM(o.revenue)
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.date >= '2024-01-01'
GROUP BY c.country;
```

### View vs table — when to use each

||View over files|Table (loaded into DuckDB)|
|---|---|---|
|Data lives in|Parquet files on disk / S3|DuckDB's own storage|
|Load step needed|No|Yes (`INSERT` or `CREATE TABLE AS`)|
|Always reflects latest files|Yes|No — must reload on change|
|Query speed|Slightly slower (file I/O each time)|Faster (DuckDB's own columnar format)|
|Use when|Files are the source of truth, updated externally|You want DuckDB to own and manage the data|

---

## 7.5 Persistence vs ephemeral queries

DuckDB has two modes of operation. Choose based on whether you need to store state.

### Ephemeral (in-memory, no file)

```python
import duckdb

# No file path = in-memory, nothing persisted
conn = duckdb.connect()
conn.execute("CREATE TABLE t AS SELECT * FROM read_parquet('data/*.parquet')")
result = conn.execute("SELECT SUM(revenue) FROM t").fetchall()
# conn closed → everything gone
```

**Use when:** You're running a one-off query or pipeline. The Parquet files on disk are the source of truth. You don't need DuckDB to remember anything between runs.

This is the most common workflow for analytics scripts and data pipelines — query files, produce output, done.

### Persistent (.duckdb file)

```python
import duckdb

# File path = persistent database on disk
conn = duckdb.connect('analytics.duckdb')
conn.execute("CREATE TABLE IF NOT EXISTS orders AS SELECT * FROM read_parquet('data/*.parquet')")
# Tables, views, indexes survive after conn is closed
```

**Use when:**

- You want DuckDB to own and manage tables (not just point at files)
- You have derived/aggregated tables you want to persist and query repeatedly
- You're building an application where DuckDB is the database, not just a query engine

### Hybrid — views in a persistent database

The most practical pattern for many applications:

```python
conn = duckdb.connect('analytics.duckdb')

# Raw data stays in Parquet files — just register views
conn.execute("""
    CREATE VIEW IF NOT EXISTS orders AS
    SELECT * FROM read_parquet('data/orders/**/*.parquet', hive_partitioning=true)
""")

# Aggregated/derived tables live in DuckDB
conn.execute("""
    CREATE TABLE IF NOT EXISTS monthly_revenue AS
    SELECT date_trunc('month', order_date) AS month, SUM(revenue) AS revenue
    FROM orders
    GROUP BY 1
""")
```

Raw files remain the source of truth. DuckDB owns only the derived layer.

---

## 7.6 Useful settings and pragmas

These are the settings you'll actually reach for in day-to-day use.

### Memory limit

```sql
-- Default is 80% of system RAM — set explicitly if running alongside other processes
SET memory_limit = '4GB';
```

DuckDB will spill to disk if a query exceeds this limit, but performance degrades significantly. Set it below your available RAM to leave headroom.

### Thread count

```sql
-- Default is number of CPU cores — reduce if sharing the machine
SET threads = 4;

-- Check current value
SELECT current_setting('threads');
```

DuckDB parallelizes across threads automatically. More threads = faster queries up to the point of I/O saturation.

### Temp directory (for spill)

```sql
-- Where DuckDB writes if it spills to disk during a large query
SET temp_directory = '/fast-disk/duckdb-tmp';
```

Point this at your fastest disk (NVMe if available). Spill performance is entirely determined by this disk's write speed.

### Progress bar (CLI)

```sql
-- Show query progress for long-running queries in the CLI
SET enable_progress_bar = true;
```

### Summarize a file quickly

```sql
-- Column-level stats: min, max, avg, null count, distinct count
SUMMARIZE SELECT * FROM read_parquet('data/orders.parquet');
```

Equivalent to `pg_stats` but runs on a file. Useful for understanding data before modeling.

### Describe a file's schema

```sql
DESCRIBE SELECT * FROM read_parquet('data/orders.parquet');
-- or
SELECT * FROM parquet_schema('data/orders.parquet');
```

### Inspect row group statistics

```sql
-- See the actual min/max stats per row group per column
SELECT * FROM parquet_metadata('data/orders.parquet');
```

Use this to verify that your data is sorted in a way that enables row group skipping.

---

## 7.7 Integrating DuckDB into an application

### Connection lifecycle

DuckDB connections are cheap to create. For a persistent database, the typical pattern is one connection per process (not per request).

```python
import duckdb

# Module-level singleton — one connection for the whole process
_conn = None

def get_conn():
    global _conn
    if _conn is None:
        _conn = duckdb.connect('analytics.duckdb')
    return _conn
```

Unlike Postgres, you do not need a connection pool. DuckDB connections are not network sockets — they're just object handles to the same in-process database.

### Single-writer constraint

This is the most important operational fact about DuckDB: **only one process can open a persistent `.duckdb` file for writing at a time.**

```
Process A: duckdb.connect('analytics.duckdb')  ← holds the write lock
Process B: duckdb.connect('analytics.duckdb')  ← blocks or errors
```

Multiple readers are fine with read-only connections:

```python
# Read-only — many processes can do this simultaneously
conn = duckdb.connect('analytics.duckdb', read_only=True)
```

**Design consequence:** if your application has multiple workers (web server with multiple processes, multiple containers), only one can write to the DuckDB file. Common patterns:

- **One writer process** handles all writes; workers query files directly (ephemeral) or use read-only connections to the `.duckdb` file.
- **No persistent file** — workers query Parquet files directly. DuckDB holds no state. This sidesteps the single-writer problem entirely and is the most scalable pattern.

### Running queries from application code

```python
import duckdb

conn = duckdb.connect('analytics.duckdb')

# Returns a list of tuples
rows = conn.execute(
    "SELECT country, SUM(revenue) FROM orders WHERE year = ? GROUP BY country",
    [2024]
).fetchall()

# Returns a pandas DataFrame
df = conn.execute("SELECT * FROM orders LIMIT 1000").df()

# Returns an Arrow table (zero-copy, very fast)
arrow_table = conn.execute("SELECT * FROM orders").arrow()
```

### Parameterized queries

Always use parameters for user-supplied values — DuckDB supports SQL injection just as Postgres does.

```python
# ✅ parameterized
conn.execute("SELECT * FROM orders WHERE customer_id = ?", [customer_id])

# ❌ string interpolation
conn.execute(f"SELECT * FROM orders WHERE customer_id = '{customer_id}'")
```

### Thread safety within a process

A single DuckDB connection is **not thread-safe** for concurrent queries. If your application uses multiple threads:

```python
import duckdb
import threading

# Option 1: one connection per thread
local = threading.local()

def get_conn():
    if not hasattr(local, 'conn'):
        local.conn = duckdb.connect('analytics.duckdb', read_only=True)
    return local.conn

# Option 2: serialize access with a lock
lock = threading.Lock()
conn = duckdb.connect('analytics.duckdb')

def run_query(sql):
    with lock:
        return conn.execute(sql).fetchall()
```

---

## 7.8 Postgres → DuckDB Translation Table

|Postgres concept|DuckDB equivalent|Key difference|
|---|---|---|
|`psql` (CLI client)|`duckdb` (CLI)|Same idea; DuckDB CLI can query files directly|
|`pg_hba.conf`, roles, auth|None|DuckDB has no auth — access is controlled by filesystem permissions|
|Connection pool (pgBouncer)|Not needed|DuckDB is in-process; connections are free|
|`\d tablename`|`DESCRIBE tablename`|Same purpose|
|`EXPLAIN (ANALYZE, BUFFERS)`|`EXPLAIN ANALYZE`|DuckDB shows vectorized plan nodes, row group stats|
|`pg_stats`|`SUMMARIZE SELECT * FROM file.parquet`|Per-column statistics on demand|
|`COPY FROM` (bulk load)|`CREATE TABLE t AS SELECT * FROM read_parquet(...)`|DuckDB reads Parquet natively; no separate load step|
|`COPY TO` (export)|`COPY (...) TO 'file.parquet' (FORMAT PARQUET)`|DuckDB writes Parquet natively|
|Materialized view|`CREATE TABLE AS SELECT` + manual refresh|No auto-refresh; you recreate or append manually|
|`pg_cron` / scheduled jobs|External scheduler (cron, Airflow)|DuckDB has no built-in job scheduler|
|`shared_buffers`|`SET memory_limit`|Same idea: how much RAM the engine can use|
|`max_connections`|Not applicable|No connection limit; single-writer constraint instead|
|`VACUUM`|`VACUUM` (exists but rare)|DuckDB's storage doesn't accumulate dead tuples the same way|
|Extensions (`CREATE EXTENSION`)|`INSTALL ext; LOAD ext;`|Same idea; `httpfs` for S3, `spatial` for geo|
|`search_path` / schemas|Schemas exist but rarely needed|Most DuckDB workflows use the default schema|

---

[[#Module 7 Summary|↑ Module 7 Summary]] · [[#Table of Contents|↑ TOC]] · [[#Module 5 — Analytical Data Modeling|← Module 5]]

---

_End of Module 7._

> **Suggested next steps:**
> 
> - **Module 8 — ETL & Incremental Loads:** How to append new data to partitioned Parquet datasets, handle late-arriving data, and compact without rewriting everything.
> - **Module 9 — Query Optimization in Practice:** Reading `EXPLAIN ANALYZE` output end-to-end, identifying the bottleneck node, and knowing whether the fix is SQL, layout, or modeling.