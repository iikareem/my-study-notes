---
tags:
  - postgresql
  - database
  - vacuum
  - batching
---

# PostgreSQL Bulk Updates: The Hidden Cost of Doing It All at Once

## Table of Contents

1. [The Problem Statement](#1-the-problem-statement)
2. [What PostgreSQL Does During a Massive UPDATE](#2-what-postgresql-does-during-a-massive-update)
3. [The Accumulation Effect](#3-the-accumulation-effect)
4. [Lock Memory and Escalation](#4-lock-memory-and-escalation)
5. [The Batching Solution](#5-the-batching-solution)
6. [The Gap: Why It Matters](#6-the-gap-why-it-matters)
7. [Implementation — Node.js + PostgreSQL](#7-implementation--nodejs--postgresql)
8. [Choosing Batch Size](#8-choosing-batch-size)
9. [Monitoring While Running](#9-monitoring-while-running)
10. [Lock Escalation Across Databases](#10-lock-escalation-across-databases)
11. [Mental Model Summary](#11-mental-model-summary)

---

## 1. The Problem Statement

When you need to update millions of rows in PostgreSQL, the instinct is to write one clean query:

```sql
UPDATE orders SET status = 'archived' WHERE created_at < '2024-01-01';
-- Affects 10,000,000 rows
```

It's simple, readable, and correct. But at scale it creates a cascade of systemic pressure on the database that goes far beyond the cost of the individual row updates themselves.

The common assumption is:

> _"Batching doesn't help because each row still pays the same MVCC, WAL, and index cost regardless."_

This is true at the **row level**. But it misses the **systemic level** — what the database accumulates and is unable to clean while that transaction is open. That is where the real problem lives.

---

## 2. What PostgreSQL Does During a Massive UPDATE

### 2.1 How Row Locking Actually Works

PostgreSQL does **not** pre-lock all matching rows before starting. It locks them **incrementally, row by row**, as the scan progresses:

```
Sequential scan / Index scan
        ↓
  Read row 1  →  matches WHERE?  →  yes  →  acquire row lock  →  write new version
  Read row 2  →  matches WHERE?  →  yes  →  acquire row lock  →  write new version
  Read row 3  →  matches WHERE?  →  no   →  skip
  ...
```

The lock type is `FOR NO KEY UPDATE` — it blocks other `UPDATE`/`DELETE` on the same row but allows plain `SELECT`(because MVCC lets readers see the old row version without needing a lock).

The critical point: **all acquired locks are held until the transaction commits**. Even though locking is incremental, by the time the scan reaches row 5,000,000 — rows 1 through 4,999,999 are already locked and will stay locked until the entire UPDATE finishes.

From the outside, it looks like all rows were locked at once.

### 2.2 MVCC Dead Tuple Explosion

PostgreSQL's MVCC model never overwrites a row in place. An update writes a **new row version** and marks the old one as dead:

```
Before UPDATE:
  Page 1: [row v1: status='active'] ← live tuple

After UPDATE:
  Page 1: [row v1: status='active'] ← dead tuple (old version, kept for concurrent readers)
           [row v2: status='archived'] ← live tuple (new version)
```

10,000,000 rows updated = **10,000,000 dead tuples** written inside one transaction. These dead tuples stay in the heap, bloating the table, until `VACUUM` reclaims them. But VACUUM cannot safely reclaim them while the transaction that created them is still open.

### 2.3 WAL Amplification

Every page change is recorded in the **Write-Ahead Log (WAL)** before it's written to the heap. A 10M row update in one transaction produces one enormous WAL segment. This creates pressure on:

- **Replication**: standbys must replay the entire WAL segment before they can catch up. Lag grows for the full duration of the transaction.
- **WAL archiving**: if you archive WAL to S3 or similar, one massive segment must be shipped before archiving is considered current.
- **Checkpointer**: dirty pages accumulate in shared buffer without a chance to flush gradually. The checkpointer may be forced to flush at the worst possible time.

### 2.4 Shared Buffer Pressure

PostgreSQL loads pages into `shared_buffers` as it scans and modifies them. A 10M row update touches a huge number of pages and marks them all dirty. This:

- **Evicts hot pages** that your application's regular queries depend on
- Forces those queries to reload their data from disk (expensive I/O)
- Degrades overall query performance across the database while the UPDATE runs

### 2.5 Transaction ID (XID) Pressure

The long-running transaction holds a **transaction ID open for its entire duration**. This prevents autovacuum from cleaning dead tuples across the **whole database** — not just the table being updated — because VACUUM needs to know which XIDs are still active before it can safely reclaim anything.

---

## 3. The Accumulation Effect

This is the core insight. A single massive UPDATE is an **uninterrupted accumulation cycle** with no opportunity for the database to act on what it has accumulated:

```
Single UPDATE timeline (10M rows):

t=0s ──────────────────────────────────────────────────── t=60s
      scan + lock accumulating                             COMMIT
      dead tuples accumulating                                │
      WAL accumulating                                        │
      shared buffer filling with dirty pages                  │
      replication lag growing                                 ▼
                                              autovacuum, checkpoint,
                                              replication catch-up
                                              can only start NOW
```

Every problem compounds on top of itself because there is no gap in which the database can settle its debt. The later rows in the scan are processed in a progressively more stressed environment than the earlier ones.

---

## 4. Lock Memory and Escalation

### 4.1 Row Locks Cost Memory

Each row lock is an entry in PostgreSQL's **lock table** — a fixed-size shared memory segment allocated at startup:

```sql
SHOW max_locks_per_transaction;  -- default: 64
SHOW max_connections;            -- default: 100

-- Approximate total lock table capacity:
-- max_locks_per_transaction × (max_connections + max_prepared_transactions)
-- default: 64 × 100 = 6,400 lock slots
```

Each lock entry holds: relation OID, block number, offset, transaction ID, lock mode, and reference count. The lock table **does not grow dynamically** — it is finite and pre-allocated.

### 4.2 Lock Escalation

When a transaction accumulates so many row locks that the lock table approaches capacity, PostgreSQL has no choice but to **escalate to a table-level lock**:

```
Row locks accumulating...
  slot 1     → row #1
  slot 2     → row #2
  ...
  slots full → cannot add more row lock entries

PostgreSQL:
  → drops all individual row lock entries (memory reclaimed)
  → acquires single ShareRowExclusiveLock on the whole table
  → lock table pressure relieved
```

One lock entry replaces millions. The memory problem is solved — but at a severe cost to concurrency:

|Lock Type|Blocks|
|---|---|
|Row lock (`FOR NO KEY UPDATE`)|Other `UPDATE`/`DELETE` on the **same rows only**|
|Table lock (`ShareRowExclusiveLock`)|**All** `UPDATE`, `DELETE`, `INSERT` on the entire table|

A query that was only blocking writes to specific rows is now blocking every write your application attempts on that table.

### 4.3 This Is a Side Effect in PostgreSQL, Not a Policy

PostgreSQL has no configurable escalation threshold. Escalation is a **pressure relief mechanism** triggered when the lock table fills up — not a deliberate optimization. You cannot set "escalate after N row locks."

The params that govern this indirectly are the lock table size params (`max_locks_per_transaction`, `max_connections`).

---

## 5. The Batching Solution

Batching breaks one long exclusive accumulation cycle into many short ones, inserting **gaps** between them where the database can act.

```
Single UPDATE:
|══════════════════════════════════════════════════════════════════| commit
 ← autovacuum blocked, replication lagging, shared buffer stressed →

Batched UPDATEs:
|══════| gap |══════| gap |══════| gap |══════| gap |══════| commit per batch
        ↑            ↑            ↑
   vacuum runs   replica    shared buffer
   dead tuples   catches up  flushes dirty
   reclaimed                 pages
```

Each batch commits its own transaction, releases all its locks, and allows the gap to occur before the next batch begins.

The total row-level cost — MVCC, WAL, index updates — is **identical** across batches and the single query. What batching controls is the **systemic cost** of holding all of that accumulated state without relief.

---

## 6. The Gap: Why It Matters

The gap between batches is not just a pause. It is a **window for the database to settle its debt**:

|Process|What it does in the gap|Why it matters|
|---|---|---|
|**Autovacuum**|Reclaims dead tuples from the last batch|Table doesn't bloat; future scans stay fast|
|**Checkpointer**|Flushes dirty pages from shared buffer to disk|Buffer has room for the next batch's pages|
|**WAL writer / replication**|Standbys replay WAL and catch up|Replica lag stays bounded|
|**Shared buffer**|App's hot pages flow back in|Regular queries don't slow down|
|**Lock manager**|All row locks from last batch released|Competing transactions unblock immediately|
|**XID**|Transaction closes; VACUUM can scan freely|Dead tuples across the whole DB can be reclaimed|

Each batch arrives into a **cleaner environment** than the previous one. The later batches are no more expensive than the first — unlike in a single query where later rows are processed into an increasingly stressed system.

> The gap doesn't just pause the work. It lets the database settle its debt from the last batch before taking on new debt, so the cost never compounds into a crisis.

---

## 7. Implementation — Node.js + PostgreSQL

### 7.1 Why Keyset Pagination, Not OFFSET

```sql
-- OFFSET: degrades as offset grows — scans all preceding rows each time
SELECT id FROM orders LIMIT 10000 OFFSET 500000;  -- gets slower every batch

-- Keyset: uses the index — O(log n) regardless of position
SELECT id FROM orders WHERE id > 500000 LIMIT 10000;  -- stays fast
```

Always use keyset pagination for batch loops.

### 7.2 Basic Batch Loop

```js
async function batchUpdate(pool, batchSize = 10_000) {
  let lastId = 0;
  let totalUpdated = 0;

  while (true) {
    const result = await pool.query(
      `UPDATE orders
       SET status = 'archived'
       WHERE id IN (
         SELECT id FROM orders
         WHERE created_at < $1
           AND id > $2
           AND status != 'archived'
         ORDER BY id
         LIMIT $3
       )
       RETURNING id`,
      ['2024-01-01', lastId, batchSize]
    );

    const count = result.rowCount;
    totalUpdated += count;

    if (count === 0) break;

    lastId = Math.max(...result.rows.map(r => r.id));

    console.log(`Updated ${totalUpdated} rows so far...`);

    // Breathing room for autovacuum, checkpoint, replication
    await sleep(50);
  }

  console.log(`Done. Total updated: ${totalUpdated}`);
}

const sleep = ms => new Promise(res => setTimeout(res, ms));
```

### 7.3 Production-Safe Version with Error Handling

```js
async function batchUpdateSafe(pool, batchSize = 10_000) {
  let lastId = 0;
  let totalUpdated = 0;

  while (true) {
    const client = await pool.connect();

    try {
      await client.query('BEGIN');

      const result = await client.query(
        `UPDATE orders
         SET status = 'archived'
         WHERE id IN (
           SELECT id FROM orders
           WHERE created_at < $1
             AND id > $2
             AND status != 'archived'
           ORDER BY id
           LIMIT $3
         )
         RETURNING id`,
        ['2024-01-01', lastId, batchSize]
      );

      await client.query('COMMIT');

      if (result.rowCount === 0) break;

      totalUpdated += result.rowCount;
      lastId = Math.max(...result.rows.map(r => r.id));

      console.log(`Updated ${totalUpdated} rows so far (lastId=${lastId})`);

      await sleep(50);

    } catch (err) {
      await client.query('ROLLBACK');
      console.error(`Batch failed at lastId=${lastId}:`, err.message);
      // Re-run from same lastId — nothing was committed
      throw err;
    } finally {
      client.release();
    }
  }

  console.log(`Done. Total updated: ${totalUpdated}`);
}
```

On failure: the batch rolls back cleanly, `lastId` hasn't moved, and you can resume from the exact same position. Nothing is partially applied.

### 7.4 Adaptive Sleep Based on Replication Lag

```js
async function adaptiveSleep(pool) {
  const result = await pool.query(`
    SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
    AS lag_seconds
  `);

  const lag = parseFloat(result.rows[0].lag_seconds) || 0;

  if (lag > 10) {
    console.warn(`Replication lag ${lag}s — sleeping 1s`);
    await sleep(1000);
  } else if (lag > 5) {
    await sleep(300);
  } else {
    await sleep(50);
  }
}
```

Use this instead of a fixed sleep to automatically back off when the replica is struggling.

---

## 8. Choosing Batch Size

There is no universal answer. The right size balances **throughput** (bigger batches = fewer round trips) against **systemic pressure** (bigger batches = longer lock hold, more dead tuples per gap).

```
Too small (e.g., 100 rows):
  → thousands of round trips
  → transaction overhead dominates
  → total wall time increases significantly

Too large (e.g., 1,000,000 rows):
  → approaches the single-query problem
  → autovacuum can't keep up in the gaps
  → replication lag builds between batches

Sweet spot: 1,000 – 50,000 rows
  → 10,000 is a reliable default to start from
```

|Table size|Starting batch size|Watch for|
|---|---|---|
|< 1M rows|5,000 – 10,000|Lock waits → go smaller|
|1M – 50M rows|10,000 – 25,000|Replica lag → add more sleep|
|> 50M rows|10,000 – 50,000 + sleep|Autovacuum falling behind → go smaller|

Tune by observing the monitoring queries in the next section while a test run is in progress.

---

## 9. Monitoring While Running

Run these in a second connection concurrently with the batch loop:

```sql
-- Are any sessions blocked waiting on locks?
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- How are dead tuples accumulating on the target table?
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- Is autovacuum actively working on the table?
SELECT pid, query, state
FROM pg_stat_activity
WHERE query ILIKE '%autovacuum%';

-- Replication lag on standbys
SELECT
  client_addr,
  write_lag,
  flush_lag,
  replay_lag
FROM pg_stat_replication;

-- Lock table utilization
SELECT count(*) AS total_locks
FROM pg_locks;
```

If `dead_pct` keeps climbing without autovacuum running in the gaps — reduce batch size or increase sleep. If `replay_lag` is growing — increase sleep or reduce batch size.

---

## 10. Lock Escalation Across Databases

Different databases handle the row-lock memory problem with different philosophies:

|Database|Escalation Behavior|Configurable?|
|---|---|---|
|**PostgreSQL**|Escalates to table lock when the fixed lock table fills — a side effect, not a policy|❌ No explicit threshold — governed indirectly by `max_locks_per_transaction`|
|**SQL Server**|Actively escalates row → page → table at 5,000 locks per transaction by default|✅ Yes — `ALTER TABLE ... SET (LOCK_ESCALATION)`, trace flags 1211/1224|
|**Oracle**|Almost never escalates — lock metadata is stored in the row itself, not a central table|❌ N/A — architecture avoids the problem|
|**MySQL InnoDB**|Lock metadata lives in the buffer pool — scales more freely, escalation is rare|❌ No explicit threshold|

The root tradeoff all of them are managing:

```
Many row locks:
  ✅ Fine-grained → other transactions touch unrelated rows freely
  ❌ Expensive   → each lock consumes a fixed-size shared memory slot

One table lock:
  ✅ Cheap       → single entry in the lock table
  ❌ Aggressive  → blocks ALL writes to the table from every session
```

Escalation is the database choosing **memory efficiency over concurrency**. Each database decides how deliberately and how early to make that trade.

In PostgreSQL: the message is — keep your transactions short enough that the lock table never fills. Batching is the mechanism that makes that possible at scale.

---

## 11. Mental Model Summary

### The Highway Analogy

> A single massive UPDATE closes an entire highway for an hour. Batches close one lane at a time for ten seconds each. The total road work is identical. But traffic keeps moving.

### The Debt Analogy

> A single UPDATE accumulates debt — dead tuples, dirty pages, WAL, replication lag, lock memory — and pays none of it until the very end. Batches pay their debt after every batch, so it never compounds into a crisis.

### What Batching Actually Controls

|Cost|Per-row level|Systemic level|
|---|---|---|
|MVCC dead tuple write|Same|Autovacuum can reclaim per batch|
|WAL write|Same|Replication can replay per batch|
|Index update|Same|Index pages unlocked per batch|
|Shared buffer dirty|Same|Checkpointer flushes per gap|
|Row lock memory|Same|Lock table resets per batch|
|XID held open|Same|VACUUM can scan DB between batches|

**Batching does not reduce row-level cost. It prevents systemic cost from compounding.**

That is the complete picture. Every row pays its dues either way. Batching controls what the database is allowed to do _between_ those payments.

---

## Related

- [[VACUUM]]
- [[WAL]]
- [[HOT]]
- [[Lock in Database]]

← [[Concepts MOC]]
