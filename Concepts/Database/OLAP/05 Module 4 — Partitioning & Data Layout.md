# Module 4 — Partitioning & Data Layout

[[04 Module 3 — Parquet & File Formats|← Module 3]] · [[00 OLAP MOC|↑ Table of Contents]] · [[06 Module 5 — Star Schemas & Dimensional Design|Next: Module 5 →]]

---

## Module 4 Summary

> **One-liner:** In OLAP, **physical layout beats clever SQL** — partition keys, file sizes, and sort order are write-time decisions that are expensive to undo.

**Most important takeaways:**

1. **Hive partitions** — Directories like `date=2024-01-15/` *are* the index. With `hive_partitioning=true`, DuckDB skips whole folders before opening files.
2. **Pruning ≠ B-tree** — Indexes find individual rows; partitions skip coarse chunks (days/months). Same goal, different granularity.
3. **Choose partition keys from query WHERE patterns** — Usually time. Cardinality sweet spot: ~50–5,000 partitions. Too many (`user_id`) → metadata hell; too few → little pruning.
4. **Small-files problem** — Streaming writes → thousands of tiny files → footer/open overhead dominates. Target ~50 MB–1 GB files; **compact** regularly and sort while compacting.
5. **Layout > SQL** — Wrong partition key cannot be fixed by rewriting the query; you must rewrite the data. Decide layout before the first write.

**Jump to:** [[#4.1 Hive-style partitions|4.1]] · [[#4.2 Partition pruning vs Postgres indexes — same goal, different mechanism|4.2]] · [[#4.3 Choosing partition keys|4.3]] · [[#4.4 The small-files problem|4.4]] · [[#4.5 Compaction and file sizing intuition|4.5]] · [[#4.6 How layout decisions affect cost more than clever SQL|4.6]] · [[#4.8 Postgres → OLAP Translation Table|4.8 Translation]]

---

## 4.1 Hive-style partitions

When data grows beyond what fits in a single Parquet file that you can query in seconds, you need to physically organize it on disk so queries can skip irrelevant data entirely — without reading any file's content.

**Hive-style partitioning** is the most common convention: data is split into directories where the directory names encode the partition key and value.

### Directory layout

```
data/
└── events/
    ├── date=2024-01-01/
    │   └── part-0.parquet
    ├── date=2024-01-02/
    │   ├── part-0.parquet
    │   └── part-1.parquet
    ├── date=2024-01-03/
    │   └── part-0.parquet
    └── date=2024-01-04/
        └── part-0.parquet
```

The folder name `date=2024-01-01` encodes:

- Partition column: `date`
- Partition value: `2024-01-01`

**Every file inside a partition directory contains only rows where `date = 2024-01-01`.** This is a physical guarantee, not an index.

### Multi-level partitioning

```
data/events/
├── year=2024/
│   ├── month=01/
│   │   ├── day=01/
│   │   │   └── part-0.parquet
│   │   └── day=02/
│   │       └── part-0.parquet
│   └── month=02/
│       └── ...
```

Multi-level partitioning allows filtering at multiple granularities, but it multiplies the number of directories quickly. Use it when queries commonly filter at two levels (e.g., year + region).

### How DuckDB reads hive partitions

```sql
-- Without hive_partitioning: date column not available, all files read
SELECT COUNT(*) FROM read_parquet('data/events/**/*.parquet')
WHERE date = '2024-01-01';  -- ERROR: column 'date' not found

-- With hive_partitioning=true: folder names become columns
SELECT COUNT(*) FROM read_parquet('data/events/**/*.parquet', hive_partitioning=true)
WHERE date = '2024-01-01';  -- ✅ reads only date=2024-01-01/ directory
```

With `hive_partitioning=true`, DuckDB:

1. Reads the directory tree to find all `key=value` folder names
2. Makes those values available as virtual columns
3. Applies WHERE clause filters against folder names before opening any files

---

## 4.2 Partition pruning vs Postgres indexes — same goal, different mechanism

They solve the same problem — skip irrelevant data — but through completely different mechanisms. Understanding the difference helps you make layout decisions correctly.

### Postgres B-tree index (seek mechanism)

A B-tree index is a sorted tree structure that maps column values to physical row locations (TIDs — tuple IDs). When you query `WHERE order_date = '2024-01-15'`, Postgres:

1. Traverses the B-tree in microseconds to find the leaf pages matching `order_date = '2024-01-15'`
2. Follows the TID pointers to the actual heap pages
3. Returns only those rows

The index is a maintained data structure — every INSERT/UPDATE/DELETE updates the index. It's fast for seeks but adds write overhead and storage cost.

### Partition pruning (physical skip mechanism)

A hive partition has no maintained structure. The data's physical location _is_ the index. When you query `WHERE date = '2024-01-15'`, the engine:

1. Lists the directory tree
2. Sees `date=2024-01-15/` as a folder name
3. Opens only that folder's files
4. Never touches files in other date directories

There is no data structure to maintain. No write overhead. The "index" is the directory layout itself.

### The trade-off table

|Dimension|Postgres B-tree Index|Hive Partition|
|---|---|---|
|Mechanism|Data structure (balanced tree)|Physical file layout (directory tree)|
|Maintained by|Database engine (on write)|You (at write time)|
|Write overhead|Yes — every write updates the index|None — partitioning happens during write|
|Storage overhead|Yes — separate index pages|None — it's just folder organization|
|Granularity|Can find individual rows|Eliminates files/directories (coarser)|
|Best for|High-cardinality, random-access queries|Time-series, date-range, categorical filters|
|Effective selectivity|0.001% of rows|5–50% of data (entire partitions)|
|Schema dependency|On a specific column|On the directory naming convention|

### The key insight

A Postgres index can find one row out of a billion. A partition can skip one month of data out of three years. They operate at completely different granularities. For analytics, skipping a month's worth of files is often more valuable than finding one row — because analytical queries touch many rows within the surviving partitions anyway.

**Use both when appropriate:** A partitioned Parquet dataset loaded into DuckDB + row group statistics in the Parquet footer is roughly equivalent to Postgres partition pruning + a BRIN index. The Parquet approach achieves it with no server, no index maintenance, and much better compression.

---

## 4.3 Choosing partition keys

Choosing the wrong partition key is one of the most costly architectural mistakes in analytical systems because it's expensive to fix — you have to rewrite all your data.

### The fundamental constraint

A partition key must appear in (almost) every query's WHERE clause for pruning to work. If queries never filter on the partition key, the engine reads all partitions on every query. You get the organizational structure of partitioning with none of the performance benefit.

### Rule 1: Choose by query filter pattern, not by what seems "logical"

```
❌ Partition by customer_id
   Queries like "what is total revenue last month?" don't filter by customer.
   Every partition is read. Zero benefit.

✅ Partition by date (or year/month)
   Almost every analytical query has a time range. Massive pruning benefit.
```

Before choosing a partition key, audit your top 10 most common queries. What's in the WHERE clause of all of them? That's your partition key candidate.

### Rule 2: Cardinality must be manageable

Cardinality = the number of distinct values = the number of partitions.

**Too low cardinality:**

```
Partition by: active (true/false) — 2 partitions
Every query still reads half the data at minimum. Essentially useless for pruning.
Also: only 2 partitions means no parallelism benefit across many files.
```

**Too high cardinality:**

```
Partition by: user_id — 10 million partitions
10 million directories on disk. Listing the directory tree alone takes seconds.
Most partitions have one or two files. Massive metadata overhead.
```

**Sweet spot:** Hundreds to low thousands of partitions. `date` gives you 365 per year — good. `year/month` gives you 12 per year — manageable. `user_id` on a large dataset — terrible.

**Rough guideline:** 50–5,000 partitions is the practical range. Below 50, pruning saves little. Above 5,000, directory listing overhead becomes significant.

### Rule 3: Partition at the granularity queries filter at

If queries always ask for a full month (`WHERE month = '2024-01'`), partitioning by day creates unnecessary granularity — 30 files to open instead of 1, with more metadata overhead and no additional pruning benefit.

If queries ask for specific days, partition by day. If queries ask for months, partition by month or year/month.

### Rule 4: Don't use partition keys with high update frequency

Partition values are embedded in directory names. If you partition by `status` ('active', 'inactive') and rows frequently change status, you have to delete them from one partition and write them to another. In append-only systems, this becomes a mess of stale data requiring compaction.

Prefer partition keys that are **immutable** or **naturally append-only**: `event_date`, `created_year`, `region`, `data_source`.

---

## 4.4 The small-files problem

The small-files problem is one of the most common production issues in OLAP data systems. It's worth understanding deeply because it silently degrades performance.

### What causes it

Every time data is written to a partitioned dataset, if the write is small, it creates a small file:

```
Streaming writes every minute to date=2024-01-15/:
  part-0.parquet    (5 KB, 120 rows — one minute of events)
  part-1.parquet    (5 KB, 118 rows)
  part-2.parquet    (6 KB, 125 rows)
  ... (1,440 files by end of day)
```

### Why small files hurt

**1. Per-file overhead dominates:** Reading a Parquet file has fixed overhead — open the file, read the footer, process metadata. For a 5 KB file, this overhead is larger than the actual read time. Multiply by 1,440 files and the overhead is enormous relative to the data volume.

**2. Row group statistics are useless:** A 5 KB file with 120 rows has one row group. Its min/max range spans 120 rows. You can never skip it — either all 120 rows match or none do. The statistical pruning that makes Parquet fast simply doesn't work at this scale.

**3. Directory listing is expensive:** Listing 1,440 files in a directory, especially on S3 (which lists in batches of 1,000 with API calls), is slow. The "find all files to query" step becomes a bottleneck.

**4. Memory pressure:** DuckDB (and other engines) open file handles and load footers for each file. 1,440 open file handles per partition × many partitions = significant overhead.

### What "right-sized" means

|Scale|Target file size|Approximate row count|
|---|---|---|
|Small dataset|50–200 MB|500K–5M rows|
|Medium dataset|200–500 MB|5M–50M rows|
|Large dataset|500 MB–1 GB|50M–200M rows|

**The 128 MB rule of thumb:** Spark defaults to 128 MB target file sizes. DuckDB row groups default to ~120K rows. These are reasonable starting points. A good Parquet file should have enough rows that row group statistics are meaningful (thousands to millions of rows per row group, multiple row groups per file).

---

## 4.5 Compaction and file sizing intuition

Compaction is the process of merging many small files into fewer larger files. It's the OLAP equivalent of Postgres VACUUM — not exactly cleaning dead tuples, but cleaning up fragmented file layouts.

### When to compact

- After high-frequency streaming writes that produce many small files
- After many incremental appends to a partition
- When query performance degrades over time even though data volume is similar (sign that file count is growing)
- Periodically as a maintenance job (daily or weekly, depending on write frequency)

### How to compact with DuckDB

```sql
-- Read all small files in a partition, write as one larger file
COPY (
    SELECT * FROM read_parquet('data/events/date=2024-01-15/*.parquet')
    ORDER BY event_time  -- sort while compacting for better row group stats
) TO '/tmp/compacted/part-0.parquet' (FORMAT PARQUET, ROW_GROUP_SIZE 200000);

-- Then replace the old files with the compacted file
-- (shell: mv /tmp/compacted/part-0.parquet data/events/date=2024-01-15/part-0.parquet)
-- (shell: rm data/events/date=2024-01-15/part-1.parquet part-2.parquet ...)
```

### Sorting during compaction

Compaction is your opportunity to sort data in a way that improves row group statistics. Sort by the column most commonly filtered in queries:

```sql
COPY (
    SELECT * FROM read_parquet('data/events/date=2024-01-15/*.parquet')
    ORDER BY user_id, event_time  -- sort by most common filter column
) TO 'compacted.parquet' (FORMAT PARQUET);
```

After sorting, `WHERE user_id = 42` will likely be satisfied by a small number of consecutive row groups, and the rest will be skipped. Without sorting, row groups have overlapping user_id ranges and nothing can be skipped.

---

## 4.6 How layout decisions affect cost more than clever SQL

This is the most important lesson in Module 4, and one that experienced OLTP engineers often miss: **in OLAP systems, the physical layout of your data determines performance more than query optimization.**

### The comparison

In Postgres:

- Bad query → `EXPLAIN` → add index → rewrite query → fast
- You tune queries and indexes _after_ the fact

In OLAP:

- Bad layout → **no SQL rewrite fixes it**. You have to rewrite the data.
- Layout decisions (partition key, file sizes, column sort order, compression) are made at write time and are expensive to change.

### Concrete example

```
Scenario: 1 TB of event data, partitioned by user_id (10M users)

Query: SELECT event_type, COUNT(*) FROM events
       WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31'
       GROUP BY event_type

With user_id partitioning:
- WHERE clause filters on event_date, not user_id
- Every partition must be read (no pruning)
- 10 million directory files scanned
- Query time: minutes

With event_date partitioning:
- WHERE clause matches the partition key exactly
- Only 31 day-partitions read out of 365
- ~91% of data skipped
- Query time: seconds
```

No SQL rewrite of the second scenario makes the first scenario as fast. The data must be reorganized.

### The layout checklist — decisions made at write time

|Decision|Impact|Change cost|
|---|---|---|
|Partition key choice|Which queries get pruning|Full data rewrite|
|Partition granularity (day vs month vs hour)|Pruning precision vs file count|Full data rewrite|
|File size (small vs large)|Metadata overhead vs parallelism|Compaction job|
|Column sort order within files|Row group statistics quality|Full file rewrite|
|Compression codec (zstd, snappy, gzip)|File size vs CPU cost|File rewrite|
|Column inclusion (wide vs narrow schema)|Storage and read cost|Schema decision at design time|

### The practical consequence

**Design your data layout before you start writing data.** Ask:

1. What are the 3 most common WHERE clause columns in my analytical queries?
2. Which of those has the right cardinality for partitioning (hundreds to thousands of distinct values)?
3. Within each partition, what column should I sort by for the best row group statistics?

Answer these questions before writing the first file. Changing the layout later means reprocessing all historical data.

---

---

## 4.8 Postgres → OLAP Translation Table

|Postgres concept|OLAP / Parquet equivalent|Key difference|
|---|---|---|
|Table partitioning (`PARTITION BY RANGE`)|Hive-style directory partitioning|Postgres: in-database. OLAP: literal filesystem folders.|
|Partition pruning (planner skips child tables)|Directory-level file skipping|Same goal; OLAP skips before any file I/O|
|B-tree index|Row group statistics + data sort order|No maintained structure; statistics are metadata in the footer|
|BRIN index (block range index)|Row group min/max statistics|Conceptually identical — both work on block-range statistics|
|`CLUSTER` (sort heap by index)|Sorting data before writing to Parquet|Same idea: co-locate related rows for range skip efficiency|
|VACUUM|Compaction job|Postgres reclaims dead tuples; OLAP merges small files|
|ANALYZE (update statistics)|Parquet statistics are written at creation time|No separate analyze step; stats are in the footer|
|`pg_partman` (partition management)|Custom scripts or orchestration (dbt, Airflow)|No built-in partition manager in OLAP file systems|
|Tablespace|Storage path / S3 bucket prefix|Same idea: physical data location|
|Partial index|Sub-partitioning (e.g., year/month/day)|Approximate equivalent; different mechanism|
|`ALTER TABLE ATTACH PARTITION`|Copying files into the partition directory|Moving a file into the right folder = attaching the partition|

---

[[#Module 4 Summary|↑ Module 4 Summary]] · [[00 OLAP MOC|↑ TOC]] · [[04 Module 3 — Parquet & File Formats|← Module 3]] · [[06 Module 5 — Star Schemas & Dimensional Design|Next: Module 5 →]]

---

_End of Modules 1–4._

> **Suggested next steps:**
> 
> - The natural progression after Module 4 is **analytical data modeling** (star schemas, fact and dimension tables) — how you shape your data for OLAP querying, not just how you store it.
> - Continue into Module 7 (DuckDB) with these layout ideas in mind — the connection between partitions, Parquet statistics, and query behavior becomes concrete once you start querying files.

