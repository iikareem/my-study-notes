---
tags:
  - database
  - mysql
  - postgresql
---

# PostgreSQL vs. MySQL: Comprehensive Architecture & Performance Notes

> **Note on scope:** This document covers PostgreSQL (default storage engine) vs. MySQL with the InnoDB storage engine. MyISAM and other MySQL engines have different characteristics and are not covered.

---

## 1. Core Architecture: Process vs. Thread Model

### The Fundamental Difference

**PostgreSQL — Multi-Process Model**

PostgreSQL uses a _process-per-connection_ model supervised by a master process called the **postmaster**. Every client connection causes the postmaster to `fork()` a brand-new OS process called a **backend process**. These processes are fully isolated from each other in terms of memory address space and communicate via POSIX shared memory (the shared buffer cache, lock tables, WAL buffers).

```
postmaster (supervisor)
├── backend process  ← connection 1
├── backend process  ← connection 2
├── backend process  ← connection 3
├── autovacuum worker
├── WAL writer
├── checkpointer
└── background writer
```

**MySQL — Multi-Thread Model**

MySQL runs as a single OS process (`mysqld`). Each client connection is served by a **thread** within that process. All threads share the same process address space, including the InnoDB buffer pool, undo log structures, and the data dictionary.

```
mysqld (single process)
├── connection thread  ← connection 1
├── connection thread  ← connection 2
├── connection thread  ← connection 3
├── InnoDB purge thread
├── InnoDB background I/O threads
└── replication threads
```

---

### Performance Trade-offs of Process vs. Thread

#### Memory Overhead

|Aspect|PostgreSQL (Process)|MySQL (Thread)|
|---|---|---|
|Per-connection overhead|~5–10 MB per OS process|~1–2 MB per thread|
|Memory isolation|Full (separate address spaces)|None (shared heap)|
|Memory leak blast radius|Single connection only|Entire server|
|RAM cost at 1000 connections|~5–10 GB|~1–2 GB|

Each PostgreSQL backend process carries its own stack, local buffers, and memory maps. `work_mem` is allocated _per operation per process_ — meaning a query doing a sort and a hash join in the same process can use `2 × work_mem`. At 200 concurrent connections each running one complex query with parallel workers, the multiplication becomes:

```
200 connections × 4 workers × work_mem = potentially gigabytes
```

**Developer action:** Keep `work_mem` low globally (4–16 MB) and raise it per session only for known heavy queries:

```sql
SET work_mem = '256MB';
SELECT ... ORDER BY ...;
RESET work_mem;
```

MySQL threads share the heap, so memory usage per connection is much lower — but a bug in a shared structure can silently corrupt data visible to all threads simultaneously.

#### Context Switching

Process-to-process context switches are more expensive than thread-to-thread switches because the OS must flush the Translation Lookaside Buffer (TLB) and switch the virtual memory address space. Thread switches within the same process skip this overhead.

However, this difference matters most at **high connection counts**. At 50–200 connections, modern Linux schedulers (CFS) handle PostgreSQL process scheduling efficiently. The cost becomes significant at 500+ simultaneous active connections.

#### Fault Isolation

This is PostgreSQL's clearest architectural advantage. If a backend process crashes (due to a buggy extension, memory error, or corrupted query plan), **only that one connection dies**. The postmaster detects the exit, logs it, and spawns a fresh backend for the next connection.

In MySQL, a thread crash inside `mysqld` can trigger a server-wide crash or data structure corruption visible to all connections. Extensions and plugins running inside the MySQL process have no isolation boundary.

---

### Connection Scalability & Pooling

**PostgreSQL** degrades significantly at high direct connection counts because:

- Each connection forks a new OS process (fork has a measurable cost)
- The shared lock table, proc array, and WAL structures have more contention points as process count grows
- Postgres's internal `PGPROC` array (process slots) is fixed at compile time based on `max_connections`

**Production rule:** `max_connections` in PostgreSQL should be kept deliberately low (100–200). Use **PgBouncer** in transaction pooling mode to multiplex thousands of application connections onto a small pool of actual Postgres backends. Without this, PostgreSQL performs poorly at scale.

```
Applications (1000s of connections)
        ↓
    PgBouncer  (transaction pooling)
        ↓
  PostgreSQL  (50–200 actual backends)
```

**MySQL** handles higher raw connection counts more gracefully because threads are cheaper. The `thread_cache_size` setting allows MySQL to reuse threads across connections, avoiding thread creation cost. That said, **ProxySQL** is the recommended pooler for MySQL at scale, adding query routing, read/write splitting, and query caching.

---

## 2. Query Parallelism & CPU Utilization

### Intra-Query Parallelism

**PostgreSQL** supports _intra-query parallelism_: a single SQL query can be split across multiple CPU cores simultaneously. The query planner estimates whether parallelism is beneficial based on table size and available `max_parallel_workers_per_gather`. Worker processes are spawned by the backend and operate on disjoint chunks of the data.

Operations that can run in parallel in PostgreSQL:

- Parallel sequential scans
- Parallel index scans (partial)
- Parallel hash joins and merge joins
- Parallel aggregates (`COUNT`, `SUM`, `AVG`)
- Parallel `CREATE INDEX`

```sql
-- PostgreSQL can use 4 workers on this:
SELECT region, SUM(revenue)
FROM sales
GROUP BY region;
-- EXPLAIN shows: Gather → Parallel Hash Aggregate → Parallel Seq Scan
```

**MySQL** historically executes each query on a single thread (one CPU core). A 30-second analytical query will fully saturate one CPU core while all remaining cores sit idle. MySQL 8.0 introduced hash joins (previously only nested loop joins were supported), which improves join performance on large tables, but _intra-query parallelism remains extremely limited_.

This is the dominant reason PostgreSQL is significantly better suited to OLAP workloads, while MySQL is optimized for OLTP patterns where individual queries are fast and parallelism per query is irrelevant.

### Join Algorithm Availability

|Algorithm|PostgreSQL|MySQL|
|---|---|---|
|Nested Loop Join|✅|✅ (primary)|
|Hash Join|✅|✅ (MySQL 8.0+)|
|Merge Join (Sort-Merge)|✅|❌|
|Parallel execution of above|✅|❌|

Merge joins require sorted input and are efficient when data is pre-sorted via indexes. PostgreSQL's planner selects between all three based on cost estimates; MySQL's planner has fewer options and falls back to nested loops more frequently.

---

## 3. MVCC: How Both Databases Handle Concurrent Reads and Writes

MVCC (Multi-Version Concurrency Control) is the mechanism both databases use to allow reads and writes to proceed concurrently without blocking each other. Their implementations differ fundamentally and produce different performance characteristics.

### PostgreSQL MVCC — Append-Only / In-Table Versioning

When a row is updated in PostgreSQL:

1. The old row version is **not deleted** — it is marked dead by setting its `xmax` system column to the current transaction ID
2. A **new row version** is appended to the table (possibly to a different page)
3. All indexes that pointed to the old row's physical location (`ctid`) must now point to the new location

This creates "dead tuples" — row versions that are no longer visible to any active transaction but still occupy physical space on disk. The **autovacuum** background process is responsible for reclaiming this space.

```
BEFORE UPDATE:                     AFTER UPDATE:
┌─────────────────────┐           ┌─────────────────────┐
│ xmin=100  xmax=0    │           │ xmin=100  xmax=200  │  ← dead tuple
│ name='Alice' age=25 │           │ name='Alice' age=25 │
└─────────────────────┘           ├─────────────────────┤
                                  │ xmin=200  xmax=0    │  ← live tuple (new)
                                  │ name='Alice' age=26 │
                                  └─────────────────────┘
```

**Consequences:**

- Table bloat: high-write tables grow on disk due to accumulated dead tuples
- Index bloat: indexes accumulate pointers to dead tuples
- Autovacuum load: constant background I/O to clean up — can become a bottleneck under sustained heavy writes
- **Heap-Only Tuple (HOT) optimization:** if the updated column is not indexed and the new row fits on the same page, PostgreSQL can skip index updates — but this is a best-case optimization, not guaranteed

**Full Page Writes:** PostgreSQL writes complete 8 KB pages to the WAL on the first modification of a page after a checkpoint. This ensures crash-safe recovery but significantly increases WAL volume on write-heavy workloads.

### MySQL/InnoDB MVCC — Update-In-Place / Undo Log Versioning

When a row is updated in MySQL/InnoDB:

1. The **old row data** is written to the **Undo Log** (a separate, sequential file)
2. The row is **updated in-place** in the buffer pool page — the physical row location does not change
3. Hidden columns on the row (`DB_TRX_ID`, `DB_ROLL_PTR`) are updated — `DB_ROLL_PTR` points to the undo log entry for the old version

```
Main Table Page (in buffer pool):
┌──────────────────────────────────────────────┐
│ DB_TRX_ID=200  DB_ROLL_PTR=→undo_entry_42   │
│ name='Alice'   age=26   (CURRENT VALUE)      │
└──────────────────────────────────────────────┘

Undo Log:
┌──────────────────────────────────────────────┐
│ undo_entry_42: name='Alice' age=25 txn=100   │
└──────────────────────────────────────────────┘
```

Because the row's physical location never changes, index entries **never need to be updated** for row updates (unless the indexed column itself changes). This is a meaningful write amplification reduction compared to PostgreSQL.

**No vacuum needed:** Once a transaction commits and no active readers need the old version, InnoDB's **purge threads** discard undo log entries sequentially. There is no table bloat because the row is always updated in-place.

**Undo log contention:** The undo log is a global shared structure. Under extreme write concurrency, threads contend on undo log mutex points. Additionally, long-running transactions prevent purge threads from cleaning old undo entries, causing the undo tablespace to grow unboundedly — which slows down every MVCC read that needs to traverse an undo chain to find the correct version.

### How MySQL Repeatable Reads Work Without Blocking Readers

Every row in InnoDB carries two hidden metadata columns:

- `DB_TRX_ID`: the transaction ID that last modified this row
- `DB_ROLL_PTR`: a pointer into the undo log chain for older versions

When a transaction starts under `REPEATABLE READ`, InnoDB creates a **Read View**: a snapshot recording which transaction IDs were active/uncommitted at that moment.

When reading a row, InnoDB checks:

- If `DB_TRX_ID` is older than (committed before) the Read View → read the row directly
- If `DB_TRX_ID` is newer or active → follow `DB_ROLL_PTR` into the undo log, traverse backward through the undo chain until finding a version visible to this Read View

This traversal is the cost of long undo chains: if a row has been updated 1000 times since your Read View was created, InnoDB must walk 1000 undo entries to find your version. This is why **long-running read transactions under heavy write workloads are expensive in MySQL**.

### MVCC Comparison Summary

|Aspect|PostgreSQL|MySQL/InnoDB|
|---|---|---|
|Old version location|In main table (dead tuples)|Undo log (separate)|
|Row physically moves on update|Yes (usually)|No|
|Index updates on row update|Yes (usually all indexes)|No (only changed columns)|
|Table bloat|Yes — needs VACUUM|No|
|Undo log bloat|No|Yes — long txns cause growth|
|Reader performance|Snapshot from heap|Undo chain traversal|
|Maintenance background process|autovacuum|purge threads|

---

## 4. Write Path: WAL vs. Redo Log & Buffer Management

### The Write-Ahead Logging Principle (Both Databases)

Neither database writes directly to the main data file during a transaction. Both first write the change to a sequential log file durably — this is the essence of Write-Ahead Logging (WAL). Sequential I/O is orders of magnitude faster than random I/O, so this makes commits fast regardless of where the data row ends up on disk.

- PostgreSQL writes to the **WAL** (`pg_wal/`)
- MySQL writes to the **InnoDB Redo Log** (`ib_logfile0`, `ib_logfile1`, or the newer `#ib_redo*` files in MySQL 8.0)

Both also buffer writes in RAM (PostgreSQL: shared buffers; MySQL: InnoDB buffer pool) and flush dirty pages to disk asynchronously in the background.

### PostgreSQL Write Path

```
Transaction COMMIT
      ↓
Write change to WAL buffer
      ↓
WAL writer flushes WAL to disk (fsync if synchronous_commit=on)
      ↓
COMMIT returns to client
      ↓ (asynchronously)
Background writer / checkpointer flushes dirty pages to main data file
```

**Full Page Writes:** On the first write to a page after a checkpoint, PostgreSQL includes the entire 8 KB page in the WAL, not just the change delta. This protects against partial page writes during a crash but significantly amplifies WAL volume. On an update-heavy workload, WAL can easily be 2–5× larger than the actual data changed.

**Freelist-based allocation:** Postgres does not simply append new row versions to the end of a file. It maintains a free space map (FSM) per table and fills existing empty pockets in pages first — which means writes to the main data file are partially random, not sequential.

### MySQL Write Path

```
Transaction COMMIT
      ↓
Write change to Redo Log buffer
      ↓
Redo Log flushed to disk (behavior controlled by innodb_flush_log_at_trx_commit)
      ↓
COMMIT returns to client
      ↓ (asynchronously)
InnoDB buffer pool dirty pages sorted and flushed in semi-sequential batches
```

**Group Commit:** MySQL batches multiple concurrent transaction commits into a single `fsync()` call on the redo log — this is called group commit. Under high concurrency, many transactions accumulate in the redo log buffer between flushes, dramatically improving I/O throughput compared to one `fsync` per transaction. PostgreSQL also has group commit but MySQL's implementation is historically more aggressive.

**`innodb_flush_log_at_trx_commit` options:**

|Value|Behavior|Durability|Performance|
|---|---|---|---|
|`1` (default)|fsync on every commit|Full ACID — no data loss|Slowest|
|`2`|Write to OS cache, fsync every second|Up to 1 second of data loss|Fast|
|`0`|Write to InnoDB buffer, flush every second|Up to 1 second of data loss|Fastest|

PostgreSQL's equivalent is `synchronous_commit`:

|Value|Behavior|
|---|---|
|`on` (default)|WAL flushed to disk before COMMIT returns|
|`remote_write`|WAL sent to replica before COMMIT returns|
|`off`|COMMIT returns before WAL is flushed (risk: last ~3 transactions lost on crash)|

---

## 5. Crash Recovery: Undo Log vs. WAL Roll-Forward

### PostgreSQL Crash Recovery

On restart after a crash, PostgreSQL:

1. Reads the last checkpoint location from the control file
2. **Replays WAL records** forward from the checkpoint (roll-forward / redo phase)
3. Any transactions that were in-flight at crash time have no COMMIT record in the WAL — their changes are simply absent from the replayed state (they were never written to the data files without a commit)
4. No separate undo phase is needed — incomplete transactions leave no trace in the data files

This is clean and simple: the WAL is the single source of truth.

### MySQL/InnoDB Crash Recovery

MySQL's recovery is two-phase because writes go to the main data file (via the buffer pool) before the transaction commits:

1. **Roll-forward (Redo Log):** MySQL replays the redo log to recover all committed changes that were in RAM at the time of crash. This brings the data files to the state they would have been in had the buffer pool been flushed before the crash.
    
2. **Roll-back (Undo Log):** After roll-forward, some transactions exist in the data files that were never committed (they were written to the buffer pool but crashed before COMMIT). MySQL uses the undo log to reverse these uncommitted changes, restoring the data to a consistent state.
    

```
Crash at time T:
  - Transaction A: committed, changes in redo log  → Roll FORWARD  ✅
  - Transaction B: in-flight, changes in buffer pool → Roll BACK   ↩️
```

The undo log serves double duty: it is both the MVCC history store (for reader snapshots) and the recovery rollback mechanism. This is why MySQL's crash recovery can take longer on restart after heavy write workloads — there may be many uncommitted transactions in the undo log to roll back.

**Correctness guarantee:** Even if the undo log write succeeds but the redo log write fails mid-transaction, the LSN (Log Sequence Number) embedded in both logs ensures MySQL can detect the incomplete state and roll back correctly on restart. The undo log entry without a matching committed redo record is treated as an uncommitted transaction and reversed.

---

## 6. Locking, Isolation, and Deadlocks

### PostgreSQL Locking

PostgreSQL uses a hierarchical lock system: table-level locks, page-level locks (for some operations), and row-level locks. Row locks are implemented by writing the locking transaction's XID into the row header (`xmax`), not via a separate lock table — this means row locks are essentially free in terms of memory.

**DDL is dangerous in PostgreSQL:** `ALTER TABLE` acquires `ACCESS EXCLUSIVE` — the strongest lock — which blocks _all_ reads and writes on the table until it completes. On large tables, even adding a column (if it requires a table rewrite) can cause prolonged outages. Use `pg_repack` or the `NOT VALID` constraint pattern for zero-downtime schema changes.

### MySQL/InnoDB Locking

InnoDB uses row-level locks, but under `REPEATABLE READ` isolation (the default), it also uses **gap locks** and **next-key locks**. These lock _ranges_ between index values to prevent phantom reads.

**Gap lock surprise:** If you run `SELECT ... WHERE id BETWEEN 10 AND 20 FOR UPDATE` and no rows exist in that range, InnoDB still places a gap lock on the range — preventing any other transaction from inserting a row with `id` between 10 and 20. This can cause deadlocks that are completely non-obvious to developers used to simpler locking models.

**Common fix:** Switch the isolation level to `READ COMMITTED` for write-heavy OLTP workloads. This disables gap locks (only row locks are used), significantly reducing deadlock frequency at the cost of allowing phantom reads (which most applications can tolerate).

```sql
-- Session level:
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Global (my.cnf):
transaction_isolation = READ-COMMITTED
```

### Isolation Level Comparison

|Isolation Level|PostgreSQL|MySQL/InnoDB|Gap Locks|
|---|---|---|---|
|READ UNCOMMITTED|Behaves as READ COMMITTED|✅|No|
|READ COMMITTED|✅|✅|No|
|REPEATABLE READ|✅|✅ (default)|Yes (InnoDB only)|
|SERIALIZABLE|✅ (SSI — no extra locks)|✅ (2PL — range locks)|Yes|

PostgreSQL's `SERIALIZABLE` uses **Serializable Snapshot Isolation (SSI)** — a non-blocking approach that detects conflicts at commit time rather than locking ranges. MySQL's `SERIALIZABLE` uses traditional range locking (2PL), which is more conservative and more likely to cause contention.

---

## 7. Replication

### PostgreSQL Replication

**Physical (Streaming) Replication:** The primary sends WAL bytes to replicas, which apply them directly. Replicas are byte-for-byte identical to the primary. This is highly reliable and low-latency but inflexible — replicas must run the same major version and same architecture.

**Logical Replication:** PostgreSQL decodes WAL into logical change records (INSERT/UPDATE/DELETE) and sends those. This allows selective replication (specific tables), replication across major versions, and replication to non-PostgreSQL targets (e.g., Kafka via logical decoding plugins).

**Limitation:** DDL changes are not automatically replicated in logical replication — schema changes must be applied to subscribers manually.

### MySQL Replication

**Binary Log (Binlog) Replication:** MySQL replication is based on the binlog, which records logical change events. Three binlog formats:

- `STATEMENT`: logs the SQL statement (compact but non-deterministic functions can cause divergence)
- `ROW`: logs the actual changed row data (safe, verbose)
- `MIXED`: statement for safe queries, row for non-deterministic ones (default in modern MySQL)

**Group Replication / InnoDB Cluster:** MySQL 8.0 supports synchronous multi-primary replication via Group Replication. This is useful for high availability without an external orchestrator (like Patroni/Pacemaker for PostgreSQL). All nodes can accept writes; the group uses a distributed consensus protocol (based on Paxos) to ensure consistency.

---

## 8. Developer Checklist: What to Watch For in Production

### PostgreSQL-Specific

**Dead tuples and VACUUM:** Monitor with:

```sql
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Tune autovacuum aggressively on write-heavy tables:

```sql
ALTER TABLE high_write_table SET (
  autovacuum_vacuum_scale_factor = 0.01,   -- vacuum when 1% dead (default 20%)
  autovacuum_vacuum_cost_delay = 2         -- less throttling
);
```

**Transaction ID Wraparound:** PostgreSQL's transaction IDs are 32-bit. After ~2 billion transactions, the counter wraps — PostgreSQL will refuse all writes and enter read-only emergency mode to prevent data corruption. Monitor `age(datfrozenxid)` in `pg_database` and ensure autovacuum is keeping it well below 2 billion.

```sql
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;
-- Alert if age > 1.5 billion
```

**Connection pool is not optional:**

```
max_connections = 100–200 (in postgresql.conf)
PgBouncer pool_size = max_connections - superuser_reserved_connections
Application → PgBouncer (port 6432) → PostgreSQL (port 5432)
```

### MySQL-Specific

**Long-running transactions bloating the undo log:**

```sql
SELECT trx_id, trx_started, trx_query, trx_mysql_thread_id
FROM information_schema.INNODB_TRX
WHERE trx_started < NOW() - INTERVAL 60 SECOND;
```

Kill offending transactions with `KILL <thread_id>`.

**Gap lock deadlocks:** Enable deadlock logging and switch to `READ COMMITTED` if gaps locks are causing problems:

```sql
-- Log last deadlock:
SHOW ENGINE INNODB STATUS\G
-- Then look for LATEST DETECTED DEADLOCK section
```

**Redo log sizing:** The InnoDB redo log should be sized to absorb at least 60 minutes of writes without a checkpoint. Too small → frequent checkpoints → bursts of I/O. In MySQL 8.0, the redo log is automatically sized by default.

---

## 9. Key Tuning Parameters

### PostgreSQL

```ini
# postgresql.conf
max_connections = 100               # Keep low; use PgBouncer
shared_buffers = 25% of RAM         # Primary data cache
work_mem = 4MB                      # Per sort/hash per process — careful!
maintenance_work_mem = 256MB        # For VACUUM, CREATE INDEX
effective_cache_size = 75% of RAM   # Planner hint (does not allocate)
wal_buffers = 64MB
checkpoint_completion_target = 0.9  # Spread checkpoint I/O
max_parallel_workers_per_gather = 4 # Parallel query workers
autovacuum_max_workers = 5
synchronous_commit = on             # Set to 'off' for async (with understood risk)
```

### MySQL/InnoDB

```ini
# my.cnf [mysqld]
innodb_buffer_pool_size = 70-80% of RAM     # Most important setting
innodb_flush_log_at_trx_commit = 1          # 1=ACID, 2=fast, 0=fastest/risky
innodb_log_file_size = 1G                   # Larger = fewer checkpoints (MySQL 5.7)
thread_cache_size = 32                      # Reuse threads across connections
innodb_thread_concurrency = 0               # Let InnoDB auto-manage
transaction_isolation = READ-COMMITTED      # Reduces gap lock contention
innodb_undo_tablespaces = 4                 # Separate undo for easier truncation
innodb_purge_threads = 4                    # Parallel undo purge
```

---

## 10. Decision Framework: PostgreSQL or MySQL?

### Choose PostgreSQL when:

- Workload involves complex queries: multi-table JOINs, CTEs, window functions, subqueries
- You need advanced data types: JSONB, arrays, range types, custom types, full-text search
- Mixed OLTP + analytical queries on the same dataset
- Geospatial data (PostGIS is the gold standard)
- Vector similarity search (pgvector)
- Strong ACID with Serializable isolation (SSI)
- You can commit to operating connection pooling (PgBouncer) and autovacuum tuning
- Parallel query execution matters

### Choose MySQL when:

- High-concurrency OLTP: millions of simple reads and high-frequency single-row writes/updates
- You want to avoid connection-pooling complexity and vacuum overhead in exchange for simpler ops
- You need flexible replication topologies (cross-version, heterogeneous targets via binlog)
- You are building on an existing MySQL/MariaDB ecosystem
- Multi-primary synchronous replication (InnoDB Cluster) is required without external orchestrators
- Raw write throughput per connection at high concurrency is the dominant concern

---

## Summary Table

|Dimension|PostgreSQL|MySQL/InnoDB|
|---|---|---|
|Concurrency model|Multi-process|Multi-thread|
|Memory per connection|Higher (~5–10 MB)|Lower (~1–2 MB)|
|Fault isolation|Excellent (per-process)|Weaker (shared process)|
|Connection pooling|**Mandatory** (PgBouncer)|Recommended (ProxySQL)|
|MVCC mechanism|Dead tuples in heap|Undo log (separate)|
|Table bloat|Yes — needs VACUUM|No|
|Undo log bloat|No|Yes — long txns|
|Intra-query parallelism|✅ Full support|❌ Very limited|
|Write throughput (high concurrency)|Good|Better (group commit)|
|OLAP / analytical queries|Excellent|Limited|
|OLTP point lookups|Excellent|Excellent|
|Gap locking surprises|No|Yes (REPEATABLE READ)|
|Serializable isolation|SSI (non-blocking)|2PL (blocking)|
|DDL impact|Heavy (ACCESS EXCLUSIVE)|Online DDL (MySQL 8.0)|
|Replication|Streaming WAL + Logical|Binlog + Group Replication|
|Crash recovery|Single-phase WAL replay|Two-phase (redo + undo)|
|Ecosystem / extensions|Rich (PostGIS, pgvector, etc.)|Moderate|

---

_Sources: Combined from personal session notes and verified against PostgreSQL 16 and MySQL 8.0 official documentation._