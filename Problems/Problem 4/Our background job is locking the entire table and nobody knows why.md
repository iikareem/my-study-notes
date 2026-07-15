---
tags:
  - concurrency
  - database
  - interview
  - locking
  - mvcc
  - postgresql
  - problem-4
  - problems
  - skip-locked
  - system-design
---

# Problem 4 — "Our background job is locking the entire table and nobody knows why"

**Domain:** Database Internals & Performance  
**Topics:** row locks · SKIP LOCKED · table locks · VACUUM

**In this folder:** scenario (this note) · [[Problem 4 Companion]]

## 🧩 The Scenario

You have a daily cleanup job that deletes old audit logs:

```python
def cleanup_old_logs():
    deleted = db.execute("""
        DELETE FROM audit_logs
        WHERE created_at < NOW() - INTERVAL '90 days'
    """)
    print(f"Deleted {deleted.rowcount} rows")
```

The `audit_logs` table has **200 million rows.** About 2 million rows are older than 90 days.

Every night at 2am when this runs:

- The app **slows to a crawl** for 40 minutes
- Support gets alerts about **timeouts on unrelated endpoints**
- The DBA sees **lock waits stacking up** across the entire application
- Occasionally the job **runs out of transaction log space** and crashes the DB

---

## 🔍 Root Cause Analysis

### Problem 1 — One Massive Transaction Holds Locks Too Long

```sql
DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '90 days'
```

This runs as a **single transaction** deleting 2 million rows:

```
Transaction starts
  → acquires row-level locks on 2 million rows
  → builds WAL entries for 2 million rows
  → every other query touching those rows WAITS
Transaction commits after 40 minutes
  → releases all locks at once
```

---

### Problem 2 — Lock Escalation (Your Exact Question)

You're right that row-level locks are the default — but there's an important distinction between **Postgres** and **SQL Server / Oracle** behavior here.

#### In SQL Server and Oracle — Lock Escalation is Real

These engines track locks in a **lock manager table in memory.** Each row lock is a memory entry. When you lock millions of rows:

```
Row locks: 2,000,000 entries in lock manager
→ each entry ~100 bytes
→ total: ~200MB just for lock metadata
```

When memory pressure hits a threshold (SQL Server default: 40% of buffer pool), the engine **automatically escalates:**

```
2,000,000 row locks
→ escalate to page locks (one lock per 8KB page)
→ escalate further to TABLE lock (one lock on entire table)
```

A table-level exclusive lock means **every read and write on that table waits** — not just the rows being deleted. This is why unrelated queries on the same table time out.

```
Thread A: DELETE 2M rows → escalates to TABLE lock
Thread B: SELECT * FROM audit_logs WHERE user_id = 'X'
          → hits TABLE lock → WAITS
Thread C: INSERT INTO audit_logs ...
          → hits TABLE lock → WAITS
```

#### In PostgreSQL — No Automatic Escalation, But Still Painful

Postgres does **not** escalate row locks to page or table locks automatically. Row locks in Postgres are stored differently — not in a central lock manager but **inside the row itself** (in the `xmin`/`xmax` transaction ID fields on each tuple). So memory pressure doesn't trigger escalation.

However Postgres has its own problem at scale — it acquires a **relation-level lock** at the start of any DML statement:

```
DELETE on audit_logs
→ acquires RowExclusiveLock on the TABLE (not rows)  ← always
→ this blocks: ALTER TABLE, VACUUM FULL, DROP, TRUNCATE
→ normal SELECTs and INSERTs: not blocked by RowExclusiveLock
```

So in Postgres the locking story is:

```
Table level:  RowExclusiveLock     → blocks DDL, not normal traffic
Row level:    stored in tuple xmax → blocks concurrent writes to same rows
Page level:   does NOT exist in Postgres
```

The slowdown you see in Postgres during bulk deletes comes not from lock escalation but from:

- **WAL pressure** — 2M WAL entries queued
- **Buffer pool thrashing** — dirty pages for 2M rows evicting hot data
- **VACUUM debt** — 2M dead tuples created at once

#### Summary of Your Instinct

|Engine|Lock escalation?|Trigger|
|---|---|---|
|SQL Server|✅ Yes, automatic|Memory pressure or row count threshold|
|Oracle|✅ Yes, manual or automatic|Configurable|
|PostgreSQL|❌ No escalation|Row locks live in tuples, not lock manager|
|MySQL InnoDB|❌ No escalation|Similar to Postgres|

Your instinct is **exactly right for SQL Server and Oracle.** For Postgres, the pain comes from a different mechanism — but the end result (app slowdown, lock waits) looks identical from the outside.

---

### Problem 3 — Transaction Log (WAL) Exhaustion

Every deletion is written to the WAL before touching data files:

```
WAL entry: delete row 1
WAL entry: delete row 2
...
WAL entry: delete row 2,000,000
← all held on disk until transaction commits
```

This can **fill the WAL disk entirely** — causing the DB to pause all writes or crash the job outright.

---

### Problem 4 — Table Bloat After Deletion

In Postgres, `DELETE` marks rows as **dead tuples** — invisible but still physically on disk. Until `VACUUM` runs, the table is larger than before the delete. Index scans still traverse dead tuple pointers, slowing down all subsequent reads.

---

## ✅ Fix Strategy 1 — Batched Deletes with Controlled Pacing

### Core Pattern

```python
def cleanup_old_logs():
    BATCH_SIZE = 5_000
    SLEEP_MS   = 100
    total_deleted = 0

    while True:
        deleted = db.execute("""
            DELETE FROM audit_logs
            WHERE id IN (
                SELECT id FROM audit_logs
                WHERE created_at < NOW() - INTERVAL '90 days'
                LIMIT ?
                FOR UPDATE SKIP LOCKED
            )
        """, BATCH_SIZE)

        total_deleted += deleted.rowcount

        if deleted.rowcount == 0:
            break

        time.sleep(SLEEP_MS / 1000)

    print(f"Deleted {total_deleted} rows total")
```

Each transaction now:

```
Transaction 1: locks 5,000 rows → commits → releases → sleeps 100ms
Transaction 2: locks 5,000 rows → commits → releases → sleeps 100ms
...
```

Lock hold time drops from **40 minutes to ~50ms per batch.** WAL pressure is spread across 400 small transactions instead of one massive one.

---

### Why `FOR UPDATE SKIP LOCKED`

```sql
SELECT id FROM audit_logs
WHERE created_at < NOW() - INTERVAL '90 days'
LIMIT 5000
FOR UPDATE SKIP LOCKED
```

Without `SKIP LOCKED` — if any row in your batch is locked by another transaction, the whole batch **waits.** With `SKIP LOCKED` — locked rows are skipped, picked up next iteration. The cleanup job never blocks and never gets blocked — a good citizen on a shared table.

---

### Adaptive Pacing

```python
def cleanup_old_logs():
    BATCH_SIZE = 5_000
    TARGET_BATCH_MS = 200

    while True:
        start = time.time()

        deleted = db.execute("DELETE ... LIMIT ?", BATCH_SIZE)

        if deleted.rowcount == 0:
            break

        elapsed_ms = (time.time() - start) * 1000
        sleep_ms = max(50, elapsed_ms * 0.5)
        time.sleep(sleep_ms / 1000)
```

If the DB is under load and batches slow down, the sleep grows proportionally — giving the DB more breathing room automatically.

---

## ✅ Fix Strategy 2 — Table Partitioning (The Superior Long-Term Solution)

### Setup — Partition by Month

```sql
CREATE TABLE audit_logs (
    id         UUID,
    user_id    UUID,
    action     TEXT,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE audit_logs_2026_01
    PARTITION OF audit_logs
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE audit_logs_2026_02
    PARTITION OF audit_logs
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE TABLE audit_logs_2026_03
    PARTITION OF audit_logs
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
```

Inserts automatically route to the correct partition. No application changes needed.

---

### Cleanup Becomes Instant

```python
# Before partitioning — batched loop, 60 minutes, WAL pressure
DELETE FROM audit_logs WHERE created_at < '2026-01-01' LIMIT 5000 ...

# After partitioning — metadata operation, ~1 millisecond
db.execute("DROP TABLE audit_logs_2025_10")
```

`DROP TABLE` on a partition removes it from the system catalog. No row scanning, no WAL entries per row, no dead tuples, no VACUUM needed.

---

### Your Exact Question — Do Other Partitions Feel It?

**Zero impact. Completely isolated.**

```
audit_logs (parent — logical only)
├── audit_logs_2025_10   ← DROP TABLE here
├── audit_logs_2025_11   ← zero impact ✅
├── audit_logs_2025_12   ← zero impact ✅
└── audit_logs_2026_01   ← zero impact ✅
```

Each partition is a **physically separate table on disk** with its own heap files, its own indexes, and its own lock namespace. Dropping one acquires an `AccessExclusiveLock` only on that partition's relation — other partitions, their indexes, and any queries running against them are completely unaffected.

Compare that to the batched delete approach:

```
Batched delete — 50ms lock bursts on the SAME table live traffic reads from
Partition drop  — 1ms lock on a partition no live traffic ever touches
```

---

### Partition Pruning — Query Performance Bonus

```sql
SELECT * FROM audit_logs
WHERE created_at >= '2026-03-01'
  AND created_at <  '2026-04-01'
```

The planner knows this can only touch `audit_logs_2026_03` — **all other partitions are skipped entirely:**

```
Without partitioning: scan 200M rows
With partitioning:    scan ~16M rows (one month partition only)
```

This is called **partition pruning** and it's automatic when your `WHERE` clause includes the partition key.

---

### Automate Partition Creation

You must create future partitions in advance — missing a partition causes inserts to fail. Automate it:

```python
def ensure_future_partitions(months_ahead=3):
    for i in range(months_ahead):
        month = datetime.now() + relativedelta(months=i)
        start = month.replace(day=1)
        end   = start + relativedelta(months=1)

        db.execute(f"""
            CREATE TABLE IF NOT EXISTS audit_logs_{start.strftime('%Y_%m')}
            PARTITION OF audit_logs
            FOR VALUES FROM ('{start.date()}') TO ('{end.date()}')
        """)
```

Run this monthly via cron. Simple and reliable.

---

### Partitioning Tradeoffs

|Concern|Detail|
|---|---|
|Schema complexity|More moving parts, requires operational discipline|
|Future partition creation|Must be automated — missing partition = insert failure|
|Cross-partition queries|`WHERE` without partition key scans all partitions|
|Live table migration|Retrofitting onto 200M row live table is painful|
|Foreign keys|Limited FK support to partitioned tables in older Postgres versions|

---

## ✅ Fix Strategy 3 — Soft Delete + Hard Delete (Recovery Window)

When you need audit trail recoverability:

```python
# Step 1 — soft delete immediately (batched)
# gives you 7 days recovery window
UPDATE audit_logs SET deleted_at = NOW()
WHERE created_at < NOW() - INTERVAL '90 days'
  AND deleted_at IS NULL
LIMIT 5000

# Step 2 — hard delete soft-deleted rows after 7 days (batched)
# actually frees space
DELETE FROM audit_logs
WHERE deleted_at < NOW() - INTERVAL '7 days'
LIMIT 5000
```

Soft delete alone does **not** solve the locking or space problem — `UPDATE` on 2M rows has identical locking behavior to `DELETE` on 2M rows. Both still need batching.

---

## 📊 Full Strategy Comparison

|Strategy|Cleanup speed|Lock impact on live traffic|WAL pressure|Complexity|Best for|
|---|---|---|---|---|---|
|Single bulk delete|Fast|Catastrophic — lock escalation risk|Extreme|Low|Never at scale|
|Batched delete|~60min total|Minimal — 50ms bursts|Spread out|Low|Most cases|
|Soft delete alone|~60min|Same as batched delete|Same|Medium|Not a perf solution|
|Soft + hard delete|~60min|Minimal|Spread out|Medium|Recovery window needed|
|Partitioning + DROP|**~1ms**|**Zero on live partitions**|**Zero**|High|High volume time-series|

---

## 🔑 Key Takeaways

| Lesson              | Rule                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Lock escalation     | Real in SQL Server/Oracle when memory pressure hits. Postgres stores row locks in tuples — no escalation, different pain |
| Bulk deletes        | Always batch — never delete millions of rows in one transaction                                                          |
| Lock duration       | Shorter transactions = shorter locks = less contention                                                                   |
| WAL pressure        | Large transactions can exhaust transaction log — batching spreads it                                                     |
| `SKIP LOCKED`       | Makes bulk jobs non-blocking, good citizen on shared tables                                                              |
| Partitioning        | The real solution for time-series data — DROP partition is instant and isolated                                          |
| Soft delete         | Changes `DELETE` to `UPDATE` — same locking problem, still needs batching                                                |
| Partition pruning   | Queries with partition key in `WHERE` skip irrelevant partitions automatically                                           |
| Automate partitions | Always create future partitions in advance via cron                                                                      |

---

## Next

- Companion deep dive: [[Problem 4 Companion]]
- Back to hub: [[Problems MOC]]
