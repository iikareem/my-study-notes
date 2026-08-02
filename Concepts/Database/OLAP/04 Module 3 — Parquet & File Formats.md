# Module 3 — Parquet & File Formats

[[02 Module 2 — Columnar Execution Internals|← Module 2]] · [[00 OLAP MOC|↑ Table of Contents]] · [[05 Module 4 — Partitioning & Data Layout|Next: Module 4 →]]

---

## Module 3 Summary

> **One-liner:** Parquet is an immutable columnar *file format* (not a database) — row groups + footer statistics are what make skipping and column pruning possible.

**Most important takeaways:**

1. **Layout** — File → row groups (horizontal slices) → column chunks (one per column) → footer (schema + min/max/null stats + byte offsets).
2. **Skip without reading data** — Engine reads footer first, skips row groups whose min/max can't match the predicate, then reads only needed column chunks. **Sort at write time** so ranges don't overlap — unsorted data kills skipping.
3. **`SELECT *` costs more** — Each column is a separate I/O; project only what you need, especially over S3.
4. **CSV vs Parquet** — Write Parquet when you control the format; use CSV only when the recipient requires it.
5. **Schema evolution** — Adding columns is safe (old files → NULL). Renaming/narrowing types breaks multi-file reads — prefer add + deprecate, never casual renames.

**Jump to:** [[#3.1 What Parquet is and why it exists|3.1]] · [[#3.2 Row groups, column chunks, footers, statistics|3.2]] · [[#3.3 Why SELECT * hurts more in columnar formats|3.3]] · [[#3.4 CSV vs Parquet trade-offs|3.4]] · [[#3.5 Schema evolution basics and common pitfalls|3.5]] · [[#3.7 Postgres → Parquet Translation Table|3.7 Translation]]

---

## 3.1 What Parquet is and why it exists

Parquet is an **open-source, columnar file format** for storing structured data. It was created by Twitter and Cloudera in 2013, donated to the Apache Software Foundation, and is now the de facto standard for analytical data.

### Why it was needed

Before Parquet, the world had:

- **CSV** — simple, universal, but no types, no compression, no schema, no way to read a column without reading all columns. Terrible for large analytical datasets.
- **JSON** — flexible, but even larger than CSV, schema-free (which becomes a liability at scale), row-oriented.
- **Avro** — excellent for row-oriented serialization (Kafka messages), but not columnar.
- **SequenceFile / RCFile** — Hadoop-era formats with partial column support, but clunky and not widely portable.

None of these gave you: **columnar layout + types + compression + statistics + portability + schema embedded in the file.**

Parquet solved all of these at once, which is why every analytical tool (Spark, DuckDB, BigQuery, Redshift, Snowflake, Athena, Pandas, Arrow) can read and write it.

### What Parquet is not

- It is **not a database**. It has no query engine. It's just a file format.
- It is **not a transactional format**. You can't `UPDATE` a row in a Parquet file. Files are typically immutable — you write them once, read them many times.
- It is **not row-oriented**. Doing `SELECT *` on Parquet is fine, but it defeats the purpose.
- It is **not always the right format**. For streaming, use Avro. For row-by-row updates, stay with a database. For interchange with non-technical users, CSV is fine.

---

## 3.2 Row groups, column chunks, footers, statistics

Parquet's internal structure is the most important thing to understand for query performance. Here's the layout:

```
┌──────────────────────────────────────────────────────────────┐
│                      PARQUET FILE                            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    ROW GROUP 1                         │  │
│  │  (e.g., rows 1 – 122,880)                              │  │
│  │                                                        │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌───────────────┐  │  │
│  │  │ Column Chunk │ │ Column Chunk │ │ Column Chunk  │  │  │
│  │  │  order_id    │ │  category    │ │  revenue      │  │  │
│  │  │  [encoded,   │ │  [dict enc,  │ │  [delta enc,  │  │  │
│  │  │  compressed] │ │  compressed] │ │  compressed]  │  │  │
│  │  └──────────────┘ └──────────────┘ └───────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    ROW GROUP 2                         │  │
│  │  (e.g., rows 122,881 – 245,760)                        │  │
│  │   ... same structure ...                               │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    FILE FOOTER                         │  │
│  │                                                        │  │
│  │  Schema definition (column names and types)            │  │
│  │  Per row group, per column:                            │  │
│  │    - byte offset in file                               │  │
│  │    - min value, max value                              │  │
│  │    - null count                                        │  │
│  │    - distinct count (optional)                         │  │
│  │    - encoding used                                     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Row group

A **row group** is a horizontal slice of the data — a range of rows. The default row group size is 122,880 rows (DuckDB) or 128 MB (Spark). All column chunks within one row group contain data for the same set of rows.

**Why it exists:** It's the unit of parallelism and the unit of skipping. The engine can read different row groups on different CPU threads. The engine can skip entire row groups based on footer statistics without reading any column data.

### Column chunk

Within a row group, each column has its own **column chunk** — a contiguous byte range in the file containing that column's values for this row group, encoded and compressed. Column chunks for different columns can be read independently — that's column pruning in action.

### File footer

The **footer** is written at the end of the file after all row groups are written. It contains:

- The schema (column names and types)
- For each row group, for each column: the byte offset in the file, min/max values, null count, encoding

**How a query uses the footer:**

1. The engine reads the footer first (a few KB at most). This tells it: how many row groups exist, what columns exist, byte offsets.
2. It evaluates your WHERE clause against the min/max statistics per row group. Row groups that can't contain matching rows are skipped entirely — **zero bytes read from those row groups**.
3. For surviving row groups, it reads only the column chunks needed by your SELECT and WHERE.

This is why the footer is so important: it enables skipping without reading the data.

### Statistics and why they depend on data order

The min/max statistics per column per row group are only useful for skipping if data within the file is sorted in a way that creates non-overlapping value ranges.

```
Unsorted data (bad for skipping):
Row group 1: revenue min=5, max=9950  → almost any query must read this
Row group 2: revenue min=8, max=9990  → almost any query must read this
WHERE revenue > 5000 → engine must read BOTH row groups (ranges overlap)

Sorted data (good for skipping):
Row group 1: revenue min=5, max=100
Row group 2: revenue min=101, max=500
Row group 3: revenue min=501, max=1200
WHERE revenue > 5000 → skip all three row groups (all maxes < 5000)
```

**Key insight:** If you sort your data by a column before writing to Parquet, predicates on that column can skip most row groups. This is the columnar equivalent of a clustered index — but it's achieved at write time, not with a maintained data structure.

---

## 3.3 Why SELECT * hurts more in columnar formats

In a row store, `SELECT *` is essentially free in terms of format overhead — you're reading the rows anyway, and all columns are right there together.

In a columnar format, `SELECT *` means reading every column chunk in every row group that isn't skipped. Each column chunk is a separate byte range in the file — often a separate I/O operation. The number of I/O operations scales with the number of columns.

### Example

A Parquet file with 50 columns, 10 row groups:

- `SELECT col_a, col_b` → reads 2 column chunks per row group = 20 I/O operations
- `SELECT *` → reads 50 column chunks per row group = 500 I/O operations

On local NVMe this may be fast enough. Against S3, where each I/O operation has a latency of 5–50ms, the difference is enormous.

### The deeper problem with SELECT * in pipelines

When you use `SELECT *` in an intermediate CTE or subquery:

```sql
-- Bad: SELECT * forces all columns into memory for the CTE
WITH base AS (
    SELECT * FROM read_parquet('orders.parquet')
    WHERE category = 'Electronics'
)
SELECT SUM(revenue) FROM base;

-- Good: project only what you need at the source
WITH base AS (
    SELECT revenue FROM read_parquet('orders.parquet')
    WHERE category = 'Electronics'
)
SELECT SUM(revenue) FROM base;
```

The first query loads all columns for every row that passes the filter, then discards them. The second loads only `revenue`. DuckDB's optimizer is often smart enough to push the column pruning through CTEs, but you shouldn't rely on it — make the intent explicit.

### When SELECT * is acceptable

- Schema exploration: `SELECT * FROM 'file.parquet' LIMIT 5` — fine, reading 5 rows.
- CTAS (Create Table As Select) to replicate a file: fine, you want all columns.
- When the query genuinely needs all columns and the file has few columns (under ~10).

---

## 3.4 CSV vs Parquet trade-offs

CSV is universal and familiar. Parquet is purpose-built for analytics. Here's the full trade-off map.

|Dimension|CSV|Parquet|
|---|---|---|
|**Human readable**|Yes — open in any text editor|No — binary format|
|**Schema embedded**|No — inferred or external|Yes — schema in footer|
|**Types**|None — everything is a string|Full type system: int, float, decimal, date, timestamp, list, struct, map|
|**Nulls**|Convention-dependent (`""`, `\N`, empty)|Explicit null encoding per column|
|**Compression**|Whole-file (gzip, zstd) — blocks column reads|Per-column chunk compression — column reads stay fast|
|**Column skipping**|Impossible — must scan to find column N|Direct byte offset from footer|
|**Row skipping**|Impossible without scanning|Row group statistics enable skipping|
|**Splittability**|Only with special splitters (gzip=no, plain=yes)|Yes — row groups are independently readable|
|**File size**|Baseline|5–10x smaller (types + column compression)|
|**Write speed**|Fast (just serialize text)|Slower (encode, compress, build footer)|
|**Read speed (few columns)**|Slow (must scan all columns)|Fast (read only needed column chunks)|
|*_Read speed (SELECT _)__|Comparable|Slightly slower (reassemble from chunks)|
|**Schema evolution**|Ad hoc — you hope columns didn't move|Explicit — add columns, rename, change type carefully|
|**Tool support**|Universal|Near-universal in data tools; not in old Excel without plugins|
|**Streaming**|Easy (line-by-line)|Requires reading row group at minimum|

### When to use CSV

- Exchanging data with humans or tools that can't read Parquet (finance teams, regulators, external vendors)
- Very small files where the overhead of Parquet doesn't pay off
- Quick one-off exports you'll open in a spreadsheet
- Row-by-row streaming inputs (logs, events before batching)

### When to use Parquet

- Any data you're going to query analytically — even once
- Long-term storage of historical data
- Passing data between services or systems in a data platform
- Anything over ~1 GB where file size matters
- Anything where column types need to be preserved (dates, decimals, nested types)

### The practical rule

> **Write Parquet whenever you control the format.** Use CSV when the recipient requires it. Never use CSV as your internal storage format for analytical data.

---

## 3.5 Schema evolution basics and common pitfalls

Parquet files embed their schema in the footer. When your schema changes over time — new columns added, columns renamed, types changed — reading a mix of old and new files becomes a schema evolution problem.

### Safe changes (backward compatible)

**Adding a column:** Old files simply have `NULL` for the new column when read together with new files. Most engines handle this gracefully.

```sql
-- Old file schema: order_id, revenue, category
-- New file schema: order_id, revenue, category, discount

-- DuckDB reads both together correctly:
SELECT * FROM read_parquet(['old.parquet', 'new.parquet'])
-- discount column shows NULL for rows from old.parquet
```

**Widening a numeric type:** `INT32` → `INT64` is usually safe. Most engines coerce transparently.

### Unsafe changes (will break)

**Renaming a column:** A renamed column in new files is treated as a new column. Old files have the old name; new files have the new name. Reading them together gives you both columns, one full of NULLs for each file. There is no automatic rename awareness.

```
Old file: order_id | total_price | category
New file: order_id | revenue     | category
                     ↑ renamed

Reading both: order_id | total_price | revenue | category
              (NULLs in revenue for old rows, NULLs in total_price for new rows)
```

**Narrowing a type:** `FLOAT64` → `INT32` truncates values. Almost always a mistake.

**Changing a type incompatibly:** `VARCHAR` → `INTEGER` will fail on read if old string values can't be cast.

**Reordering columns:** Parquet matches columns by name, not position. Reordering is safe as long as names are unchanged. But tools that rely on positional column access (some CSV-based tools) will break.

### Practical schema evolution strategy

1. **Add columns; never remove or rename them.** Mark old columns as deprecated in documentation. This is the safest approach.
2. **Use a schema registry or metadata store** if you have many producers. Knowing the schema history is crucial for reading old files correctly.
3. **Partition by time** so old files with old schemas are in old date directories. You can then apply different read logic per partition when necessary.
4. **Test schema evolution before deploying.** Write one file with the old schema, one with the new, and verify DuckDB reads both correctly with `read_parquet(['old.parquet', 'new.parquet'])`.

---

---

## 3.7 Postgres → Parquet Translation Table

|Postgres concept|Parquet equivalent|Key difference|
|---|---|---|
|Table|Parquet file (or set of files)|Parquet is immutable; no UPDATE in-place|
|Heap page (8KB)|Row group (default ~120K rows or ~128 MB)|Much larger unit; optimized for scan, not seek|
|Column in a row|Column chunk within a row group|Stored separately; independently readable|
|`pg_type` / data type|Parquet logical type (INT32, UTF8, TIMESTAMP_MICROS, etc.)|Richer nested types (LIST, MAP, STRUCT); no domain types|
|`pg_statistic` (column stats)|Row group footer statistics (min, max, null_count)|Per row group, not per table; no histogram|
|Index (B-tree)|Row group statistics + sort order|Statistics in footer enable skipping; requires data to be sorted|
|Autovacuum|File compaction (external job)|Not automatic; you schedule it|
|`TOAST` (large values)|Dictionary encoding + variable-length encoding|Parquet handles large strings differently; no overflow pages|
|Schema migration (`ALTER TABLE`)|Write new files with new schema|No in-place DDL; schema lives in the footer of each file|
|`pg_dump` format|Parquet is its own portable format|Parquet is the dump format AND the query format|
|`COPY TO` (text/binary)|`COPY (...) TO 'file.parquet' (FORMAT PARQUET)`|DuckDB can write Parquet natively|

---

[[#Module 3 Summary|↑ Module 3 Summary]] · [[00 OLAP MOC|↑ TOC]] · [[05 Module 4 — Partitioning & Data Layout|Next: Module 4 →]]

---

