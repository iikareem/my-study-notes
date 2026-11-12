# Database Deadlocks — The Comprehensive Engineering Guide

> **Who this is for:** Backend engineers who already understand OS-level deadlocks (mutual exclusion, circular wait, hold-and-wait, no preemption) and want a deep, practical reference on how deadlocks manifest, are detected, observed, prevented, and handled in relational databases — with real SQL, real error codes, and real application-layer patterns.

---

## Table of Contents

1. [The Conceptual Bridge — OS vs. Database](#1-the-conceptual-bridge--os-vs-database)
2. [Database Locking Internals](#2-database-locking-internals)
   - 2.1 Lock Modes
   - 2.2 Lock Granularity
   - 2.3 Lock Compatibility Matrix
   - 2.4 Lock Escalation
   - 2.5 When Locks Are Acquired and Released
3. [Transaction Isolation Levels and Lock Behavior](#3-transaction-isolation-levels-and-lock-behavior)
4. [How Deadlocks Form — Anatomy With Examples](#4-how-deadlocks-form--anatomy-with-examples)
   - 4.1 Classic Two-Transaction Deadlock
   - 4.2 Three-Transaction Deadlock Cycle
   - 4.3 Upgrade Deadlock
   - 4.4 Index Scan Deadlock
   - 4.5 Foreign Key Deadlock
   - 4.6 Gap Lock Deadlock (MySQL InnoDB)
5. [Deadlock Detection Mechanisms](#5-deadlock-detection-mechanisms)
   - 5.1 Wait-For Graph (WFG)
   - 5.2 Timeout-Based Detection
   - 5.3 How Victim Selection Works
6. [Observing Deadlocks — Engine by Engine](#6-observing-deadlocks--engine-by-engine)
   - 6.1 PostgreSQL
   - 6.2 MySQL / InnoDB
   - 6.3 SQL Server
   - 6.4 Oracle
7. [Reading a Deadlock Log](#7-reading-a-deadlock-log)
8. [Prevention Strategies](#8-prevention-strategies)
   - 8.1 Consistent Lock Ordering
   - 8.2 Short Transactions
   - 8.3 Explicit Locking with SELECT FOR UPDATE
   - 8.4 Lowering the Isolation Level
   - 8.5 Optimistic Locking (Version Columns)
   - 8.6 Index Everything You Lock
   - 8.7 Avoid I/O Inside Transactions
   - 8.8 Bulk Operation Ordering
   - 8.9 Use SKIP LOCKED for Queue-Style Processing
   - 8.10 Advisory Locks for Application-Level Coordination
9. [Handling Deadlocks in Application Code](#9-handling-deadlocks-in-application-code)
   - 9.1 Retry Pattern
   - 9.2 Exponential Backoff with Jitter
   - 9.3 Idempotency and Safe Retries
   - 9.4 Node.js Full Example
10. [Deadlock vs. Related Phenomena](#10-deadlock-vs-related-phenomena)
11. [Deadlocks in ORMs](#11-deadlocks-in-orms)
12. [Monitoring and Alerting](#12-monitoring-and-alerting)
13. [Advanced Topics](#13-advanced-topics)
    - 13.1 Deadlocks in Distributed Databases
    - 13.2 MVCC and How It Reduces (But Doesn't Eliminate) Deadlocks
    - 13.3 Serializable Snapshot Isolation (SSI)
14. [Quick Reference Cheatsheet](#14-quick-reference-cheatsheet)

---

## 1. The Conceptual Bridge — OS vs. Database

You already know the OS model:

```
Process A holds Resource 1, wants Resource 2
Process B holds Resource 2, wants Resource 1
→ Circular wait → deadlock
```

In a database, the mapping is:

| OS Concept        | Database Equivalent                              |
|-------------------|--------------------------------------------------|
| Process / Thread  | Transaction (BEGIN … COMMIT/ROLLBACK)            |
| Resource          | Row, Page, Table, Index entry, or Gap            |
| Resource lock     | Shared (S), Exclusive (X), Update (U) lock, etc. |
| OS scheduler      | Database Lock Manager + Deadlock Detector        |
| Kill process      | Roll back victim transaction                     |

**The critical difference:** the OS kernel may or may not have a deadlock resolver — many systems just rely on timeouts or programmer discipline. A relational database engine is an *active participant*. It:

- Tracks every lock in memory in a **lock table**
- Continuously builds a **Wait-For Graph**
- Periodically scans for cycles
- Automatically chooses a **victim**, rolls it back completely, and returns a specific **SQLSTATE error code** to your application
- Expects your application to **retry**

This changes your responsibility: the database handles detection and resolution; you handle **prevention** and **retry**.

---

## 2. Database Locking Internals

### 2.1 Lock Modes

Most relational databases use a hierarchy of lock modes:

| Mode               | Symbol | Description                                              | Who can hold concurrently |
|--------------------|--------|----------------------------------------------------------|--------------------------|
| Shared             | S      | Acquired for reads                                       | Multiple transactions     |
| Exclusive          | X      | Acquired for writes (INSERT/UPDATE/DELETE)               | Only one                  |
| Update             | U      | Intent to upgrade from S to X; prevents upgrade deadlocks| Only one                  |
| Intent Shared      | IS     | "I intend to place S locks on rows below"                | Multiple                  |
| Intent Exclusive   | IX     | "I intend to place X locks on rows below"                | Multiple                  |
| Shared Intent Excl.| SIX    | S on the whole object + IX on individual rows below      | Only one                  |

**Why does the Update lock exist?**

Without it, two transactions can both acquire S locks on the same row, then both try to upgrade to X. Neither can, because the other holds S. That's a deadlock introduced purely by the engine — the U lock prevents it by making the "read with intent to write" operation serialize at acquisition time.

### 2.2 Lock Granularity

Databases can lock at multiple levels:

```
Database
  └── Table
        └── Page (typically 8KB block of rows)
              └── Row
                    └── Key / Gap / Next-Key (index-level, InnoDB)
```

**Finer granularity = more concurrency, more memory overhead.**  
**Coarser granularity = less memory, more contention.**

The engine typically starts with row locks and may escalate.

### 2.3 Lock Compatibility Matrix

This matrix tells you whether two lock modes can coexist on the same resource:

|       | IS  | IX  | S   | SIX | U   | X   |
|-------|-----|-----|-----|-----|-----|-----|
| **IS**| ✅  | ✅  | ✅  | ✅  | ✅  | ❌  |
| **IX**| ✅  | ✅  | ❌  | ❌  | ❌  | ❌  |
| **S** | ✅  | ❌  | ✅  | ❌  | ✅  | ❌  |
| **SIX**| ✅ | ❌  | ❌  | ❌  | ❌  | ❌  |
| **U** | ✅  | ❌  | ✅  | ❌  | ❌  | ❌  |
| **X** | ❌  | ❌  | ❌  | ❌  | ❌  | ❌  |

✅ = compatible (both can hold simultaneously)  
❌ = conflict (second requester must wait)

Deadlocks happen when two transactions are waiting for each other's incompatible locks.

### 2.4 Lock Escalation

When a transaction holds too many fine-grained locks (e.g., thousands of row locks), the engine may **escalate** to a coarser lock to save memory:

```
Row locks (5,000 rows locked) → escalate → Table lock
```

This is dangerous for deadlocks because a table lock suddenly blocks all other transactions, even those touching unrelated rows. SQL Server escalates at ~5,000 locks by default. PostgreSQL and MySQL do not escalate the same way but have analogous behaviors.

**To prevent unwanted escalation:** keep transactions small, or explicitly control the escalation threshold.

```sql
-- SQL Server: disable escalation on a specific table
ALTER TABLE orders SET (LOCK_ESCALATION = DISABLE);
```

### 2.5 When Locks Are Acquired and Released

This is the most important detail for understanding deadlock risk:

| Isolation Level    | Shared (S) lock released  | Exclusive (X) lock released |
|--------------------|---------------------------|------------------------------|
| Read Uncommitted   | Immediately (dirty read)  | At commit/rollback           |
| Read Committed     | Immediately after read    | At commit/rollback           |
| Repeatable Read    | At commit/rollback        | At commit/rollback           |
| Serializable       | At commit/rollback        | At commit/rollback           |

**Key insight:** Read Committed (the default in PostgreSQL, Oracle, SQL Server) releases S locks immediately, which dramatically reduces deadlock risk compared to Repeatable Read or Serializable.

---

## 3. Transaction Isolation Levels and Lock Behavior

The SQL standard defines four isolation levels, each protecting against different anomalies:

| Level              | Dirty Read | Non-Repeatable Read | Phantom Read | Deadlock Risk |
|--------------------|-----------|---------------------|--------------|---------------|
| Read Uncommitted   | Possible  | Possible            | Possible     | Very Low      |
| Read Committed     | ❌        | Possible            | Possible     | Low           |
| Repeatable Read    | ❌        | ❌                  | Possible     | Medium        |
| Serializable       | ❌        | ❌                  | ❌           | High          |

**Read Committed (default):** S locks released after each row read, X locks held until commit. Lowest practical deadlock risk. Most applications should use this.

**Repeatable Read:** S locks held until commit. Guarantees that if you read row X twice in one transaction, you get the same value. MySQL InnoDB's default. Much higher deadlock risk.

**Serializable:** Full lock-based serialization. Maximum correctness, maximum deadlock risk. Or, in PostgreSQL, implemented as SSI (see section 13.3) which reduces this risk.

**Choosing wisely:**
```sql
-- PostgreSQL: set per-transaction
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- MySQL: set per-session
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- SQL Server
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 4. How Deadlocks Form — Anatomy With Examples

### 4.1 Classic Two-Transaction Deadlock

The textbook case: two transactions acquiring the same two locks in opposite orders.

**Schema:**
```sql
CREATE TABLE accounts (
  id      INT PRIMARY KEY,
  balance DECIMAL(10,2)
);
INSERT INTO accounts VALUES (1, 1000.00), (2, 2000.00);
```

**Timeline:**
```
Time   Transaction A (transfer 100 from acc 1 → acc 2)
       Transaction B (transfer 50 from acc 2 → acc 1)

T1     A: BEGIN
T2                  B: BEGIN
T3     A: UPDATE accounts SET balance = balance - 100 WHERE id = 1;
          → Acquires X lock on row id=1 ✅
T4                  B: UPDATE accounts SET balance = balance - 50 WHERE id = 2;
                       → Acquires X lock on row id=2 ✅
T5     A: UPDATE accounts SET balance = balance + 100 WHERE id = 2;
          → Wants X lock on row id=2 → BLOCKED ⏳ (B holds it)
T6                  B: UPDATE accounts SET balance = balance + 50 WHERE id = 1;
                       → Wants X lock on row id=1 → BLOCKED ⏳ (A holds it)

T7  💀 DEADLOCK DETECTED
       Engine rolls back one victim (say B), returns error 40P01 to B's client
       A's second UPDATE now proceeds and A commits successfully
```

**Wait-For Graph at T6:**
```
A ──waits for──▶ B
B ──waits for──▶ A
        (cycle = deadlock)
```

### 4.2 Three-Transaction Deadlock Cycle

Deadlocks don't require just two parties:

```
T1: A holds lock on row 1, wants lock on row 2
T2: B holds lock on row 2, wants lock on row 3
T3: C holds lock on row 3, wants lock on row 1

Wait-For Graph:
A → B → C → A   (cycle of length 3)
```

These are harder to spot in logs because the chain is longer.

### 4.3 Upgrade Deadlock

Two transactions both read the same row and then both try to upgrade to a write lock:

```sql
-- Transaction A
BEGIN;
SELECT balance FROM accounts WHERE id = 1;   -- S lock acquired
-- ... some processing ...
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- wants X, blocked by B's S lock
```

```sql
-- Transaction B (concurrent)
BEGIN;
SELECT balance FROM accounts WHERE id = 1;   -- S lock acquired (compatible with A's S)
-- ... some processing ...
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- wants X, blocked by A's S lock
-- 💀 both hold S, both want X, neither can proceed
```

**Solution:** Use `SELECT ... FOR UPDATE` from the start if you know you'll write. This acquires U then X immediately, preventing both transactions from both holding S simultaneously.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- U/X lock, not S
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

### 4.4 Index Scan Deadlock

A subtle deadlock that happens when two operations scan an index in opposite directions, locking rows as they go.

```sql
-- Transaction A: scan forward, lock rows 1 → 2 → 3
UPDATE orders SET status = 'A' WHERE amount BETWEEN 100 AND 300 ORDER BY id ASC;

-- Transaction B: scan backward, lock rows 3 → 2 → 1
UPDATE orders SET status = 'B' WHERE amount BETWEEN 200 AND 400 ORDER BY id DESC;
```

A locks row 1, B locks row 3. A wants row 3, B wants row 1. Deadlock.

**Solution:** Ensure all bulk operations scan in a consistent direction, or use a consistent `ORDER BY` that all transactions share.

### 4.5 Foreign Key Deadlock

Foreign key checks acquire shared locks on the parent table to verify referential integrity:

```sql
-- Transaction A: insert into child table
BEGIN;
INSERT INTO order_items (order_id, product_id, qty) VALUES (1, 100, 2);
-- Engine checks: does order 1 exist? → S lock on orders row 1
-- Engine checks: does product 100 exist? → S lock on products row 100

-- Transaction B: delete from parent table (concurrent)
BEGIN;
DELETE FROM orders WHERE id = 1;
-- wants X lock on orders row 1 → blocked by A's S lock
DELETE FROM products WHERE id = 100;
-- if A is also trying to delete something B is checking... deadlock
```

**Solution:** Always perform parent-table operations before child-table operations within a transaction, or design cascades to avoid cross-transaction interference.

### 4.6 Gap Lock Deadlock (MySQL InnoDB — Repeatable Read)

InnoDB under Repeatable Read uses **gap locks** and **next-key locks** to prevent phantom reads. These can deadlock with inserts.

```sql
-- Both transactions run under REPEATABLE READ

-- Transaction A
BEGIN;
SELECT * FROM orders WHERE customer_id = 5 FOR UPDATE;
-- Next-key lock on the gap (5, ∞) to prevent phantoms

-- Transaction B (concurrent)
BEGIN;
SELECT * FROM orders WHERE customer_id = 7 FOR UPDATE;
-- Next-key lock on the gap (5, ∞) — compatible! Both can hold gap locks.

-- Transaction A
INSERT INTO orders (customer_id, amount) VALUES (6, 100);
-- Wants insert intention lock — conflicts with B's gap lock → BLOCKED

-- Transaction B
INSERT INTO orders (customer_id, amount) VALUES (6, 200);
-- Wants insert intention lock — conflicts with A's gap lock → BLOCKED
-- 💀 Deadlock
```

**Solution:** Switch to `READ COMMITTED` (disables gap locks) or restructure queries to avoid overlapping range scans with inserts.

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 5. Deadlock Detection Mechanisms

### 5.1 Wait-For Graph (WFG)

All major databases use this approach:

1. Maintain a directed graph in memory
   - Nodes = active transactions
   - Edges = "transaction X is waiting for a lock held by transaction Y"
2. Periodically (or continuously) scan for **cycles** using DFS or similar
3. If a cycle is found → deadlock → pick victim → roll back

```
Graph state at deadlock moment:

[Tx 101] ──waiting for lock on row 5──▶ [Tx 102]
[Tx 102] ──waiting for lock on row 3──▶ [Tx 101]

DFS from Tx 101:
  visit 101 → follow edge → visit 102 → follow edge → visit 101 (already visited!) → CYCLE
```

**Detection frequency:**

| Engine       | WFG Check Interval          |
|--------------|-----------------------------|
| PostgreSQL   | After `deadlock_timeout` (default 1s) from when wait begins |
| MySQL InnoDB | ~100ms (very aggressive)    |
| SQL Server   | Every 5s (adaptive, gets more frequent if deadlocks detected) |
| Oracle       | ~3s                         |

### 5.2 Timeout-Based Detection

A simpler (less precise) mechanism: if a transaction has been waiting for a lock longer than a configured threshold, kill it.

| Engine     | Parameter                | Default  |
|------------|--------------------------|----------|
| PostgreSQL | `lock_timeout`           | 0 (off)  |
| MySQL      | `innodb_lock_wait_timeout`| 50s     |
| SQL Server | `LOCK_TIMEOUT`           | -1 (off) |
| Oracle     | `WAIT n` in `SELECT FOR UPDATE` | off |

**Timeout ≠ deadlock detection.** A timeout kills one waiting transaction to break a *potential* cycle. WFG detection kills the minimum necessary victim to break an *actual* cycle.

Use `lock_timeout` as a safety net, not a replacement for WFG:

```sql
-- PostgreSQL: fail fast if lock not acquired in 2 seconds
SET lock_timeout = '2s';
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- fails after 2s if locked
```

### 5.3 How Victim Selection Works

When a deadlock cycle is detected, the engine must choose which transaction to roll back (the **victim**). It wants to minimize the cost of rollback.

**PostgreSQL:** Rolls back the transaction that *detected* the deadlock (the one that triggered the cycle check), not necessarily the one that's done the least work. This is by design — it's simpler and deterministic.

**MySQL InnoDB:** Chooses the transaction with the fewest row locks modified (lowest "weight"). Can be influenced with:
```sql
-- Tell InnoDB this transaction is high priority (harder to make victim)
-- Not a real command — InnoDB decides automatically, but you can reduce
-- your footprint by keeping transactions short
```

**SQL Server:** Most sophisticated. Uses a scoring system:
1. Transaction that has consumed the fewest resources (CPU, log space)
2. Can be manually influenced:
```sql
SET DEADLOCK_PRIORITY HIGH;   -- this session is less likely to be victim
SET DEADLOCK_PRIORITY LOW;    -- this session is preferred victim
SET DEADLOCK_PRIORITY NORMAL; -- default
-- Values range from -10 to 10 or LOW/NORMAL/HIGH
```

**Oracle:** Rolls back only the statement that caused the deadlock (not the whole transaction), then returns ORA-00060. You decide whether to retry or roll back.

---

## 6. Observing Deadlocks — Engine by Engine

### 6.1 PostgreSQL

**Configuration (`postgresql.conf`):**
```ini
deadlock_timeout = 1s          # how long to wait before checking for deadlock
log_lock_waits = on            # log any lock wait that exceeds deadlock_timeout
log_min_duration_statement = 0 # log all statements (for debugging)
```

**See current lock state:**
```sql
-- All current locks
SELECT
  pid,
  relation::regclass AS table_name,
  mode,
  granted,
  locktype,
  transactionid,
  virtualtransaction
FROM pg_locks
ORDER BY relation, pid;

-- Who is blocking whom (most useful query)
SELECT
  blocked.pid                    AS blocked_pid,
  blocked.usename                AS blocked_user,
  blocked.query                  AS blocked_query,
  blocked.query_start            AS blocked_since,
  blocking.pid                   AS blocking_pid,
  blocking.usename               AS blocking_user,
  blocking.query                 AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Show the full lock dependency chain
SELECT
  a.pid,
  a.usename,
  a.query,
  a.state,
  a.wait_event_type,
  a.wait_event,
  pg_blocking_pids(a.pid) AS blocked_by
FROM pg_stat_activity a
WHERE a.state != 'idle'
ORDER BY a.pid;
```

**Deadlock in the log (PostgreSQL):**
```
2024-01-15 10:23:41 UTC [1234] ERROR:  deadlock detected
2024-01-15 10:23:41 UTC [1234] DETAIL:  Process 1234 waits for ShareLock on transaction 789;
                                         blocked by process 5678.
                                         Process 5678 waits for ShareLock on transaction 456;
                                         blocked by process 1234.
2024-01-15 10:23:41 UTC [1234] HINT:    See server log for query details.
2024-01-15 10:23:41 UTC [1234] CONTEXT: while updating tuple (0,42) in relation "accounts"
```

**Error code returned to client:**
```
SQLSTATE: 40P01
Message:  ERROR: deadlock detected
```

### 6.2 MySQL / InnoDB

**See the most recent deadlock (complete cycle info):**
```sql
SHOW ENGINE INNODB STATUS\G
```

Look for the `LATEST DETECTED DEADLOCK` section. It shows:
- Timestamp
- Transaction 1: what it held, what it waited for, the actual SQL
- Transaction 2: same
- Which transaction was rolled back

**Enable persistent deadlock logging:**
```sql
-- Log all deadlocks to the MySQL error log
SET GLOBAL innodb_print_all_deadlocks = ON;

-- Persist it
-- In my.cnf:
-- innodb_print_all_deadlocks = 1
```

**Live lock monitoring:**
```sql
-- Active transactions
SELECT
  trx_id,
  trx_state,
  trx_started,
  trx_wait_started,
  trx_mysql_thread_id,
  trx_query,
  trx_rows_locked,
  trx_rows_modified
FROM information_schema.INNODB_TRX
ORDER BY trx_started;

-- MySQL 8+ Performance Schema (preferred)
SELECT
  r.trx_id                    AS waiting_trx_id,
  r.trx_mysql_thread_id       AS waiting_thread,
  r.trx_query                 AS waiting_query,
  b.trx_id                    AS blocking_trx_id,
  b.trx_mysql_thread_id       AS blocking_thread,
  b.trx_query                 AS blocking_query
FROM performance_schema.data_lock_waits AS w
JOIN information_schema.INNODB_TRX AS r
  ON r.trx_id = CAST(w.REQUESTING_ENGINE_TRANSACTION_ID AS CHAR)
JOIN information_schema.INNODB_TRX AS b
  ON b.trx_id = CAST(w.BLOCKING_ENGINE_TRANSACTION_ID AS CHAR);

-- All current data locks
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

**Error code returned to client:**
```
Error Code: 1213
SQLSTATE:   40001
Message:    Deadlock found when trying to get lock; try restarting transaction
```

**Lock wait timeout (different from deadlock):**
```
Error Code: 1205
SQLSTATE:   HY000
Message:    Lock wait timeout exceeded; try restarting transaction
```

### 6.3 SQL Server

**Default Extended Events capture (system_health session):**
```sql
-- Read deadlock graphs from the default system_health XE session
SELECT
  xdr.value('@timestamp', 'datetime2')    AS deadlock_time,
  xdr.query('.')                          AS deadlock_graph_xml
FROM (
  SELECT CAST(target_data AS XML) AS target_data
  FROM sys.dm_xe_session_targets AS t
  JOIN sys.dm_xe_sessions AS s
    ON s.address = t.event_session_address
  WHERE s.name = 'system_health'
    AND t.target_name = 'ring_buffer'
) AS d
CROSS APPLY target_data.nodes('//RingBufferTarget/event[@name="xml_deadlock_report"]') AS xdr(xdr)
ORDER BY deadlock_time DESC;
```

**Enable trace flags for older SQL Server:**
```sql
DBCC TRACEON(1222, -1);  -- detailed deadlock info in error log
DBCC TRACEON(1204, -1);  -- older format (pre-2005)
```

**Current blocking:**
```sql
-- Who's blocking whom right now
SELECT
  r.session_id                AS blocked_session,
  r.blocking_session_id       AS blocking_session,
  r.wait_type,
  r.wait_time / 1000.0        AS wait_seconds,
  r.last_wait_type,
  SUBSTRING(t.text, (r.statement_start_offset/2)+1,
    ((CASE r.statement_end_offset
        WHEN -1 THEN DATALENGTH(t.text)
        ELSE r.statement_end_offset
      END - r.statement_start_offset)/2)+1) AS current_statement,
  r.logical_reads,
  r.cpu_time
FROM sys.dm_exec_requests AS r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) AS t
WHERE r.blocking_session_id != 0;

-- Lock details
SELECT
  resource_type,
  resource_database_id,
  resource_associated_entity_id,
  request_mode,
  request_status,
  request_session_id
FROM sys.dm_tran_locks
WHERE resource_type != 'DATABASE'
ORDER BY request_session_id;
```

**Error returned to client:**
```
Error Number: 1205
SQLSTATE:     40001
Message:      Transaction (Process ID N) was deadlocked on {resource} resources
              with another process and has been chosen as the deadlock victim.
              Rerun the transaction.
```

### 6.4 Oracle

**Active locks and waits:**
```sql
-- Sessions waiting on locks
SELECT
  s.sid,
  s.serial#,
  s.username,
  s.status,
  s.event,
  s.wait_time,
  s.seconds_in_wait,
  sw.p1,
  sw.p2,
  sw.p3
FROM v$session s
JOIN v$session_wait sw ON s.sid = sw.sid
WHERE sw.event LIKE '%lock%'
   OR sw.event LIKE '%enq%';

-- Who is blocking whom
SELECT
  l1.sid AS waiting_session,
  l2.sid AS blocking_session,
  l1.type,
  l1.lmode,
  l1.request,
  l1.id1,
  l1.id2
FROM v$lock l1
JOIN v$lock l2
  ON l1.id1 = l2.id1
  AND l1.id2 = l2.id2
  AND l1.request > 0
  AND l2.lmode > 0
  AND l1.sid != l2.sid;
```

**Error returned to client:**
```
ORA-00060: deadlock detected while waiting for resource
```

Oracle behavior is unique: it rolls back only the **statement**, not the whole transaction. You must then decide what to do.

---

## 7. Reading a Deadlock Log

Let's decode a real MySQL InnoDB deadlock log:

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-01-15 10:23:41 0x7f8a1c002700
*** (1) TRANSACTION:
TRANSACTION 421938, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 42, OS thread handle 140234567890432, query id 1023 localhost user
UPDATE accounts SET balance = balance + 100 WHERE id = 2
                                                    ↑ This is what it's WAITING to lock

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 58 page no 4 n bits 72 index PRIMARY of table `bank`.`accounts`
trx id 421938 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 00000001; asc     ;;   ← This is row id=1 (what it HOLDS)

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 58 page no 4 n bits 72 index PRIMARY of table `bank`.`accounts`
trx id 421938 lock_mode X locks rec but not gap waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 00000002; asc     ;;   ← This is row id=2 (what it WANTS)

*** (2) TRANSACTION:
TRANSACTION 421939, ACTIVE 0 sec starting index read
...
UPDATE accounts SET balance = balance + 50 WHERE id = 1

*** (2) HOLDS THE LOCK(S):
... lock on row id=2 ...

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
... waiting for row id=1 ...

*** WE ROLL BACK TRANSACTION (2)
                        ↑ The victim chosen (fewer modified rows)
```

**How to read this:**
1. `(1) TRANSACTION` → Tx A's details
2. `(1) HOLDS THE LOCK(S)` → what Tx A currently owns (row id=1)
3. `(1) WAITING FOR THIS LOCK TO BE GRANTED` → what Tx A is blocked on (row id=2)
4. Repeat for Tx B
5. `WE ROLL BACK TRANSACTION (2)` → B is the victim

This tells you exactly what code to fix: two transactions locking the same two rows in opposite orders.

---

## 8. Prevention Strategies

### 8.1 Consistent Lock Ordering ⭐ Most Important

**The rule:** If any code path locks multiple rows/tables, all code paths must lock them in the same canonical order.

**The most reliable canonical order: sort by primary key ascending.**

```js
// ❌ BAD — order depends on caller, creates opposite-order risk
async function transfer(fromId, toId, amount, db) {
  await db.query(
    'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
    [amount, fromId]
  );
  await db.query(
    'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
    [amount, toId]
  );
}

// ✅ GOOD — always lock smaller id first, regardless of transfer direction
async function transfer(fromId, toId, amount, db) {
  const [firstId, secondId] = fromId < toId
    ? [fromId, toId]
    : [toId, fromId];

  // Lock in canonical order
  await db.query(
    'SELECT id FROM accounts WHERE id IN ($1, $2) ORDER BY id FOR UPDATE',
    [firstId, secondId]
  );

  // Then update (locks already held, no additional blocking)
  await db.query(
    'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
    [amount, fromId]
  );
  await db.query(
    'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
    [amount, toId]
  );
}
```

**For multi-table operations:** define a global lock order across tables. Example:
```
Always lock in this order: users → accounts → orders → order_items
Never lock accounts first if you also need to lock users in the same transaction.
```

### 8.2 Short Transactions

The longer a transaction runs, the more time its locks exist, the more chances for conflicts.

```sql
-- ❌ BAD: slow query inside a transaction
BEGIN;
  SELECT * FROM large_report_table WHERE ...;  -- takes 30 seconds, holds locks
  UPDATE summary SET ...;
COMMIT;

-- ✅ GOOD: read outside transaction, write fast
-- Step 1: read (no transaction, no locks)
SELECT * FROM large_report_table WHERE ...;

-- Step 2: compute in application

-- Step 3: write-only transaction (milliseconds)
BEGIN;
  UPDATE summary SET ...;
COMMIT;
```

**Anti-patterns that make transactions long:**
- Sending an email inside a transaction
- Calling a third-party API inside a transaction
- Waiting for user input inside a transaction
- Running complex reports inside a write transaction
- `SLEEP()` or `pg_sleep()` in a transaction (never in production)

### 8.3 Explicit Locking with SELECT FOR UPDATE

If you know you will write after reading, lock immediately. Don't let the engine deal with a lock upgrade later.

```sql
-- PostgreSQL / MySQL
BEGIN;
SELECT id, balance FROM accounts WHERE id = $1 FOR UPDATE;
-- Now do your application logic
UPDATE accounts SET balance = balance - $1 WHERE id = $2;
COMMIT;

-- For reads that don't need exclusive lock but should be consistent
BEGIN;
SELECT id, stock FROM products WHERE id = $1 FOR SHARE;
-- Other readers can proceed; writers must wait
COMMIT;

-- SQL Server equivalent
BEGIN TRANSACTION;
SELECT id, balance FROM accounts WITH (UPDLOCK, ROWLOCK) WHERE id = @id;
UPDATE accounts SET balance = balance - @amount WHERE id = @id;
COMMIT;
```

**Locking multiple rows atomically:**
```sql
-- Lock all needed rows at once with ORDER BY to enforce consistent ordering
BEGIN;
SELECT id FROM accounts
WHERE id IN (1, 2, 5, 7)
ORDER BY id          -- critical: consistent order
FOR UPDATE;
-- Now update all of them
UPDATE accounts SET balance = ... WHERE id IN (1, 2, 5, 7);
COMMIT;
```

### 8.4 Lowering the Isolation Level

Only use the isolation level you actually need. Each step up significantly increases deadlock risk.

```sql
-- PostgreSQL: if you only need reads to be non-dirty
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- MySQL: override the session default (Repeatable Read)
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
-- Gap locks are DISABLED at READ COMMITTED in InnoDB
-- This alone eliminates entire classes of phantom-read deadlocks
```

**Decision guide:**
- **Read Committed** — default choice for most applications
- **Repeatable Read** — only when you need the same read result twice in one transaction
- **Serializable** — only for financial/audit scenarios where correctness trumps throughput

### 8.5 Optimistic Locking (Version Columns)

Don't hold database locks at all during application processing. Use a **version** or **updated_at** column to detect conflicts at write time.

```sql
-- Schema
ALTER TABLE accounts ADD COLUMN version BIGINT NOT NULL DEFAULT 0;

-- Or use a timestamp (less reliable on fast machines)
ALTER TABLE accounts ADD COLUMN updated_at TIMESTAMP DEFAULT now();
```

```js
// Read — no lock acquired
const { rows } = await db.query(
  'SELECT id, balance, version FROM accounts WHERE id = $1',
  [accountId]
);
const { balance, version } = rows[0];

// Application processing (can take as long as needed — no lock held)
const newBalance = computeNewBalance(balance);

// Write — check version hasn't changed
const result = await db.query(
  `UPDATE accounts
   SET balance = $1, version = version + 1
   WHERE id = $2 AND version = $3`,
  [newBalance, accountId, version]
);

if (result.rowCount === 0) {
  // Someone else updated this row — retry or return conflict error
  throw new OptimisticLockError('Concurrent modification detected');
}
```

**Tradeoff:**
| Scenario | Pessimistic (FOR UPDATE) | Optimistic (version) |
|---|---|---|
| Low contention | OK | ✅ Better throughput |
| High contention | OK | ❌ Many retries/wasted work |
| Long processing time | ❌ Holds locks too long | ✅ No locks held |
| Strict ordering needed | ✅ | Difficult |

### 8.6 Index Everything You Lock

Without an index on the WHERE clause of an UPDATE or DELETE, the engine performs a full table scan and may lock the entire table (or every row it scans).

```sql
-- ❌ No index on customer_email: full scan, locks every row examined
UPDATE orders SET status = 'shipped' WHERE customer_email = 'alice@example.com';

-- ✅ Index exists: only the matching rows are locked
CREATE INDEX idx_orders_customer_email ON orders(customer_email);
UPDATE orders SET status = 'shipped' WHERE customer_email = 'alice@example.com';
```

**Check your query plans:**
```sql
-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS)
UPDATE orders SET status = 'shipped' WHERE customer_email = 'alice@example.com';
-- Look for: Seq Scan (bad, locks more) vs Index Scan (good, locks fewer)

-- MySQL
EXPLAIN UPDATE orders SET status = 'shipped' WHERE customer_email = 'alice@example.com';
-- type=ALL means full table scan (high lock surface)
-- type=ref or range means index used (low lock surface)
```

### 8.7 Avoid I/O Inside Transactions

This is the most common BFF-layer mistake. Never hold a database lock while waiting for:
- An HTTP call to a microservice or external API
- A message queue publish
- A file write
- A cache update (Redis, etc.)
- User input

```js
// ❌ DANGEROUS: lock held during external HTTP call
async function processOrder(orderId) {
  await db.query('BEGIN');
  const order = await db.query(
    'SELECT * FROM orders WHERE id = $1 FOR UPDATE', [orderId]
  );
  
  const payment = await paymentService.charge(order.rows[0]);  // HTTP call while locked!
  // If this takes 3 seconds, every other transaction touching this order is blocked
  
  await db.query('UPDATE orders SET status = $1 WHERE id = $2',
    [payment.status, orderId]);
  await db.query('COMMIT');
}

// ✅ SAFE: compute first, write fast
async function processOrder(orderId) {
  // Read without locking
  const order = await db.query(
    'SELECT * FROM orders WHERE id = $1', [orderId]
  );
  
  // External call with no lock held
  const payment = await paymentService.charge(order.rows[0]);
  
  // Short write-only transaction
  await db.query('BEGIN');
  await db.query(
    'UPDATE orders SET status = $1, payment_id = $2 WHERE id = $3',
    [payment.status, payment.id, orderId]
  );
  await db.query('COMMIT');
}
```

### 8.8 Bulk Operation Ordering

When processing multiple rows in a batch (e.g., a cron job that updates many records), process them in a consistent order so concurrent batch jobs don't lock in opposite directions.

```js
// ❌ BAD: order is undefined, two batch jobs can process in reverse order
const rows = await db.query('SELECT id FROM orders WHERE status = $1', ['pending']);
for (const row of rows.rows) {
  await db.query('UPDATE orders SET status = $1 WHERE id = $2', ['processing', row.id]);
}

// ✅ GOOD: always process in ascending id order
const rows = await db.query(
  'SELECT id FROM orders WHERE status = $1 ORDER BY id ASC', ['pending']
);
for (const row of rows.rows) {
  await db.query('UPDATE orders SET status = $1 WHERE id = $2', ['processing', row.id]);
}
```

Or better, use a single bulk update:
```sql
-- Atomic, consistent, and faster
UPDATE orders
SET status = 'processing'
WHERE status = 'pending'
  AND id IN (SELECT id FROM orders WHERE status = 'pending' ORDER BY id LIMIT 100);
```

### 8.9 Use SKIP LOCKED for Queue-Style Processing

When multiple workers process jobs from a database queue, `SKIP LOCKED` lets each worker grab a row without blocking others:

```sql
-- Worker: grab the next available job without competing for locked rows
BEGIN;
SELECT id, payload
FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- Skip rows locked by other workers
```

```js
// Node.js worker pool example
async function claimNextJob(db) {
  const result = await db.query(`
    UPDATE job_queue
    SET status = 'processing', started_at = NOW(), worker_id = $1
    WHERE id = (
      SELECT id FROM job_queue
      WHERE status = 'pending'
      ORDER BY priority DESC, created_at ASC
      LIMIT 1
      FOR UPDATE SKIP LOCKED
    )
    RETURNING *
  `, [workerId]);
  
  return result.rows[0] || null;  // null if no jobs available
}
```

`SKIP LOCKED` is supported in PostgreSQL 9.5+, MySQL 8.0+, Oracle 12c+. Not in SQL Server (use `READPAST` hint instead).

### 8.10 Advisory Locks for Application-Level Coordination

For application-specific locks that don't correspond to specific rows, use advisory locks to coordinate without touching data:

```sql
-- PostgreSQL advisory locks

-- Session-level (held until released or session ends)
SELECT pg_advisory_lock(12345);        -- acquire (blocks if taken)
SELECT pg_advisory_unlock(12345);      -- release

-- Transaction-level (released automatically at commit/rollback)
SELECT pg_advisory_xact_lock(12345);   -- no manual release needed

-- Try-lock (non-blocking)
SELECT pg_try_advisory_lock(12345);    -- returns true/false immediately
```

```js
// Example: only one worker should run a specific job type at a time
async function runExclusiveJob(db, jobTypeId, fn) {
  const lockId = jobTypeId;  // use a stable integer representing the job type
  
  const acquired = await db.query(
    'SELECT pg_try_advisory_xact_lock($1)', [lockId]
  );
  
  if (!acquired.rows[0].pg_try_advisory_xact_lock) {
    console.log('Another worker is running this job, skipping');
    return;
  }
  
  // We have the lock for this transaction's duration
  await fn();
  // Lock released automatically when transaction ends
}
```

---

## 9. Handling Deadlocks in Application Code

Prevention reduces deadlocks but cannot eliminate them entirely. You must always be prepared to retry.

### 9.1 Retry Pattern

**The contract:** when you receive a deadlock error, your transaction has already been rolled back. You must re-execute it from the beginning.

```js
// Deadlock error codes by engine
const DEADLOCK_CODES = {
  postgres:   '40P01',  // SQLSTATE
  mysql:      1213,     // errno / also SQLSTATE 40001
  sqlserver:  1205,     // error number / also SQLSTATE 40001
  oracle:     '60',     // ORA-00060
};

function isDeadlock(err) {
  // PostgreSQL (pg driver)
  if (err.code === '40P01') return true;
  
  // MySQL (mysql2 driver)
  if (err.errno === 1213) return true;
  if (err.code === 'ER_LOCK_DEADLOCK') return true;
  
  // SQL Server (mssql driver)
  if (err.number === 1205) return true;
  
  // Generic SQLSTATE check
  if (err.sqlState === '40001' || err.sqlState === '40P01') return true;
  
  return false;
}
```

### 9.2 Exponential Backoff with Jitter

Immediate retry can cause retry storms when multiple transactions deadlock simultaneously. Add jitter:

```js
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function computeBackoff(attempt, baseMs = 100, maxMs = 5000) {
  // Exponential: 100ms, 200ms, 400ms, 800ms...
  const exponential = Math.min(baseMs * Math.pow(2, attempt - 1), maxMs);
  
  // Full jitter: random between 0 and the exponential value
  // This prevents synchronized retries from multiple clients
  const jitter = Math.random() * exponential;
  
  return Math.floor(jitter);
}

async function withDeadlockRetry(fn, { maxRetries = 5, baseMs = 100 } = {}) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (isDeadlock(err) && attempt < maxRetries) {
        const backoff = computeBackoff(attempt, baseMs);
        console.warn(
          `Deadlock on attempt ${attempt}/${maxRetries}, retrying in ${backoff}ms`,
          { error: err.message }
        );
        await sleep(backoff);
        continue;
      }
      throw err;  // non-deadlock or max retries exceeded
    }
  }
}
```

### 9.3 Idempotency and Safe Retries

Before retrying, ensure the operation is safe to re-execute:

```js
// ❌ DANGEROUS: double-counting if first attempt partially succeeded
// (though with deadlock, the whole transaction was rolled back, so this is actually OK)
await db.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [100, accountId]);

// ✅ SAFER: idempotent writes using an idempotency key
await db.query(`
  INSERT INTO transactions (idempotency_key, account_id, amount, type)
  VALUES ($1, $2, $3, 'debit')
  ON CONFLICT (idempotency_key) DO NOTHING
`, [idempotencyKey, accountId, 100]);
```

**Key insight:** since a deadlock victim is *fully rolled back*, the retry is safe by definition — the first attempt left no trace. But if your code has side effects outside the database (like firing an event or sending an SMS), you need idempotency protection at those layers too.

### 9.4 Node.js Full Example (PostgreSQL with `pg`)

```js
import pg from 'pg';
const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });

// --- Utility ---

function isDeadlock(err) {
  return err.code === '40P01' || err.sqlState === '40P01';
}

function isLockTimeout(err) {
  return err.code === '55P03';
}

async function sleep(ms) {
  return new Promise(r => setTimeout(r, ms));
}

async function withRetry(fn, { maxRetries = 5, baseMs = 50 } = {}) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      const retryable = isDeadlock(err) || isLockTimeout(err);
      if (retryable && attempt < maxRetries) {
        const backoff = Math.floor(Math.random() * baseMs * Math.pow(2, attempt));
        await sleep(backoff);
        continue;
      }
      throw err;
    }
  }
}

// --- Transaction helper ---

async function withTransaction(fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// --- Business logic ---

async function transferFunds(fromId, toId, amount) {
  return withRetry(() =>
    withTransaction(async (db) => {
      // CRITICAL: lock in consistent order (smaller id first)
      const [firstId, secondId] = fromId < toId
        ? [fromId, toId]
        : [toId, fromId];

      // Acquire both locks at once, in order
      const { rows } = await db.query(
        `SELECT id, balance
         FROM accounts
         WHERE id = ANY($1::int[])
         ORDER BY id
         FOR UPDATE`,
        [[firstId, secondId]]
      );

      if (rows.length !== 2) throw new Error('One or both accounts not found');

      const from = rows.find(r => r.id === fromId);
      const to   = rows.find(r => r.id === toId);

      if (from.balance < amount) throw new Error('Insufficient funds');

      await db.query(
        'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
        [amount, fromId]
      );
      await db.query(
        'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
        [amount, toId]
      );

      // Audit log inside same transaction
      await db.query(
        `INSERT INTO transfer_log (from_id, to_id, amount, created_at)
         VALUES ($1, $2, $3, NOW())`,
        [fromId, toId, amount]
      );

      return { success: true, newBalance: from.balance - amount };
    })
  );
}
```

---

## 10. Deadlock vs. Related Phenomena

These are often confused:

| Phenomenon | Definition | Engine Response | App Error Code |
|---|---|---|---|
| **Deadlock** | Circular wait between transactions | Roll back victim | 40P01 (PG), 1213 (MySQL) |
| **Lock Wait Timeout** | One transaction waits too long | Roll back waiter | 55P03 (PG), 1205 (MySQL) |
| **Lock Contention** | High wait time, no cycle, eventual success | Wait and proceed | No error |
| **Livelock** | Transactions repeatedly retry and block each other (no forward progress, but no cycle) | Not detected | Manifests as timeouts |
| **Starvation** | One transaction perpetually losing to others (never deadlocked, just never scheduled) | Not detected | Manifests as timeouts |

**Livelock example:**
```
Tx A and Tx B each retry immediately on deadlock, at exactly the same time,
causing each other's retry to also deadlock, forever.
→ Fix: add randomized jitter to retry backoff (see section 9.2)
```

**How to distinguish deadlock from lock contention in logs:**
- Deadlock: error returned to client, transaction rolled back, immediate
- Lock contention: query is slow (blocked), but eventually succeeds or times out

---

## 11. Deadlocks in ORMs

ORMs often obscure what SQL is being generated, making deadlocks harder to diagnose and prevent.

### Sequelize (Node.js)

```js
// Sequelize transaction with retry
const { Sequelize } = require('sequelize');

async function withRetry(fn, retries = 5) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (err) {
      // Sequelize wraps database errors
      if (err instanceof Sequelize.DatabaseError &&
          err.parent?.code === '40P01') {
        if (i < retries - 1) {
          await sleep(Math.random() * 100 * Math.pow(2, i));
          continue;
        }
      }
      throw err;
    }
  }
}

// Lock rows explicitly in Sequelize
await withRetry(() =>
  sequelize.transaction(async (t) => {
    const accounts = await Account.findAll({
      where: { id: [fromId, toId] },
      order: [['id', 'ASC']],    // consistent order!
      lock: t.LOCK.UPDATE,       // FOR UPDATE
      transaction: t,
    });
    // ... updates ...
  })
);
```

**Enable SQL logging in Sequelize to see what's generated:**
```js
const sequelize = new Sequelize(connectionString, {
  logging: (sql) => console.log('[SQL]', sql),
});
```

### TypeORM (Node.js)

```typescript
import { DataSource, QueryRunner } from 'typeorm';

async function transfer(
  dataSource: DataSource,
  fromId: number,
  toId: number,
  amount: number
) {
  const qr: QueryRunner = dataSource.createQueryRunner();
  await qr.connect();
  await qr.startTransaction();

  try {
    // Lock in consistent order
    const [first, second] = fromId < toId
      ? [fromId, toId] : [toId, fromId];

    await qr.query(
      `SELECT id FROM accounts WHERE id IN ($1, $2) ORDER BY id FOR UPDATE`,
      [first, second]
    );

    await qr.manager
      .createQueryBuilder()
      .update(Account)
      .set({ balance: () => `balance - ${amount}` })
      .where('id = :id', { id: fromId })
      .execute();

    await qr.manager
      .createQueryBuilder()
      .update(Account)
      .set({ balance: () => `balance + ${amount}` })
      .where('id = :id', { id: toId })
      .execute();

    await qr.commitTransaction();
  } catch (err) {
    await qr.rollbackTransaction();
    throw err;
  } finally {
    await qr.release();
  }
}
```

### Prisma (Node.js)

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: ['query'],  // see all SQL
});

// Prisma wraps everything in interactive transactions
async function transfer(fromId: number, toId: number, amount: number) {
  return prisma.$transaction(async (tx) => {
    // Prisma doesn't support SELECT FOR UPDATE natively in findMany
    // Use $queryRaw for explicit locking
    const [first, second] = fromId < toId
      ? [fromId, toId] : [toId, fromId];

    await tx.$queryRaw`
      SELECT id FROM accounts
      WHERE id IN (${first}, ${second})
      ORDER BY id
      FOR UPDATE
    `;

    await tx.account.update({
      where: { id: fromId },
      data: { balance: { decrement: amount } },
    });

    await tx.account.update({
      where: { id: toId },
      data: { balance: { increment: amount } },
    });
  });
}
```

**Common ORM deadlock traps:**

1. **Eager loading inside transactions** — ORM auto-joins trigger extra locks on related tables
2. **Implicit transaction boundaries** — some ORMs wrap every operation in a transaction; multiple sequential "operations" may each be their own transaction, preventing you from grouping them correctly
3. **Invisible lock ordering** — the ORM decides what order to acquire locks; it may not match your expectation
4. **Always enable SQL logging during deadlock investigation**

---

## 12. Monitoring and Alerting

### What to Track

| Metric | Why | Alarm if |
|---|---|---|
| Deadlock count per minute | Trend shows worsening lock contention | > baseline + 3σ |
| Deadlock retry success rate | Retries are expected; failures mean bad code | < 95% success |
| p99 transaction latency | Contention slows everyone | > SLA threshold |
| Lock wait time (avg, p99) | High wait = potential for deadlocks | > 500ms |
| Long-running transactions | Root cause of many deadlocks | > 30s |

### PostgreSQL

```sql
-- Count deadlocks per database (cumulative since last reset)
SELECT datname, deadlocks, blks_hit, blks_read
FROM pg_stat_database;

-- Reset stats (do this at a known baseline)
SELECT pg_stat_reset();

-- Long-running transactions (potential deadlock contributors)
SELECT
  pid,
  usename,
  now() - xact_start AS duration,
  state,
  wait_event_type,
  wait_event,
  query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '30 seconds'
ORDER BY duration DESC;
```

### MySQL

```sql
-- Deadlock count (since server start)
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_waits';
SHOW GLOBAL STATUS LIKE 'Innodb_deadlocks';

-- Lock wait stats
SHOW GLOBAL STATUS LIKE 'Innodb_row_lock%';
```

### Application-Level Metrics (Node.js)

```js
// Instrument your retry wrapper
import { metrics } from './telemetry';  // your metrics client

async function withDeadlockRetry(fn, context = '') {
  const start = Date.now();
  let attempts = 0;

  for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
    attempts = attempt;
    try {
      const result = await fn();
      if (attempt > 1) {
        // Succeeded after retry
        metrics.increment('db.deadlock.retry_success', { context });
        metrics.histogram('db.deadlock.retry_attempts', attempt, { context });
      }
      return result;
    } catch (err) {
      if (isDeadlock(err)) {
        metrics.increment('db.deadlock.detected', { context });
        if (attempt < MAX_RETRIES) {
          await sleep(computeBackoff(attempt));
          continue;
        }
        // Exhausted retries
        metrics.increment('db.deadlock.retry_exhausted', { context });
      }
      throw err;
    }
  }
}
```

### Alert Rules

```yaml
# Example Prometheus/Alertmanager rule
groups:
  - name: database_deadlocks
    rules:
      - alert: HighDeadlockRate
        expr: rate(db_deadlock_detected_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High deadlock rate detected ({{ $value }}/min)"

      - alert: DeadlockRetryExhausted
        expr: rate(db_deadlock_retry_exhausted_total[5m]) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Deadlocks that could not be resolved by retry"

      - alert: LongRunningTransaction
        expr: db_transaction_duration_seconds{quantile="0.99"} > 30
        labels:
          severity: warning
```

---

## 13. Advanced Topics

### 13.1 Deadlocks in Distributed Databases

Traditional deadlock detection works within one database node. In distributed systems (PostgreSQL with Citus, CockroachDB, Vitess, distributed SQL), deadlocks can span multiple nodes — making detection harder.

**Approaches:**

**1. Centralized lock manager** — one node manages all locks. Simple but single point of failure and network overhead.

**2. Distributed Wait-For Graph** — each node maintains its local WFG and periodically ships it to a coordinator that looks for cross-node cycles. Used by CockroachDB.

**3. Wound-Wait / Wait-Die algorithms:**
- **Wound-Wait:** Older transaction (lower timestamp) *wounds* (preempts) younger. Younger transactions wait.
- **Wait-Die:** Older transactions wait for younger. If a younger holds a lock, the older *dies* (aborts) and restarts.

Both guarantee freedom from deadlock by imposing a global timestamp ordering. But they may cause unnecessary aborts.

**4. Timeout-based (pragmatic):** Just set a short `lock_timeout`. If a distributed deadlock occurs, whichever transaction times out first is effectively the victim. Simple, but wastes time up to the timeout.

**CockroachDB's approach:**
CockroachDB uses MVCC for most reads (no read locks), push transactions instead of waiting (a transaction can "push" a conflicting transaction's timestamp into the future), and detects deadlocks via distributed WFG with a coordinator.

### 13.2 MVCC and How It Reduces (But Doesn't Eliminate) Deadlocks

**Multi-Version Concurrency Control (MVCC)** means readers and writers don't block each other:

- **Writers** create a new version of the row
- **Readers** read the version that was current at their transaction's start time
- **No shared locks needed for reads** in typical MVCC implementations (PostgreSQL, MySQL InnoDB, Oracle)

```
Without MVCC:
  Reader → acquires S lock → Writer waits → potential deadlock if another Writer exists

With MVCC:
  Reader → reads snapshot → no lock acquired → Writer can proceed → no conflict
```

**MVCC eliminates:**
- Reader-Writer deadlocks
- Upgrade deadlocks triggered by reads that later become writes (in systems without FOR UPDATE)

**MVCC does NOT eliminate:**
- Writer-Writer deadlocks (two concurrent updates to the same rows still need X locks)
- `SELECT FOR UPDATE` deadlocks (you explicitly requested a lock)
- Gap lock deadlocks (MySQL InnoDB under Repeatable Read)

### 13.3 Serializable Snapshot Isolation (SSI)

PostgreSQL 9.1+ implements **Serializable Snapshot Isolation** — a technique that provides full serializability without traditional lock-based blocking, and with significantly fewer deadlocks than lock-based Serializable.

**How it works:**
Instead of acquiring range locks to prevent phantoms, SSI tracks **read-write dependencies** between transactions. It detects when a cycle of such dependencies would create a serialization anomaly and aborts one of the participants.

```sql
-- SSI in PostgreSQL
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  SELECT balance FROM accounts WHERE id = 1;  -- no lock, just a snapshot
  -- ... some processing ...
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
-- If a serialization anomaly is detected, PostgreSQL returns:
-- ERROR: could not serialize access due to concurrent update (SQLSTATE 40001)
```

The error code is `40001` (serialization failure) — **same SQLSTATE as a deadlock**. Your retry logic handles both.

**SSI vs. lock-based Serializable:**
| | Lock-Based Serializable | SSI (PostgreSQL) |
|---|---|---|
| Read blocking | Yes (S locks held) | No (MVCC reads) |
| Write blocking | Yes | Sometimes |
| False positives | Rare | Possible (may abort unnecessarily) |
| Throughput | Low | Much higher |
| Deadlock risk | High | Low (but serialization failures instead) |

---

## 14. Quick Reference Cheatsheet

### Error Codes

| Engine     | Deadlock Error              | Lock Timeout Error |
|------------|-----------------------------|--------------------|
| PostgreSQL | SQLSTATE `40P01`            | SQLSTATE `55P03`   |
| MySQL      | errno `1213`, SQLSTATE `40001` | errno `1205`   |
| SQL Server | error number `1205`, SQLSTATE `40001` | `1222` |
| Oracle     | `ORA-00060`                 | `ORA-00054`        |

### Diagnosis Queries

```sql
-- PostgreSQL: who's blocking whom
SELECT blocked.pid, blocked.query, blocking.pid AS blocking_pid, blocking.query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- MySQL: last deadlock
SHOW ENGINE INNODB STATUS\G

-- SQL Server: current blocking
SELECT r.session_id, r.blocking_session_id, r.wait_type, r.wait_time, t.text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id != 0;
```

### Prevention Checklist

```
Architecture
  ☐ Lock multiple resources in a canonical, consistent order (sort by PK)
  ☐ All code paths agree on the same locking order
  ☐ Defined global table lock order (e.g. users → accounts → orders)

Transaction Design
  ☐ Transactions are as short as possible
  ☐ No external HTTP calls, I/O, or sleeps inside transactions
  ☐ Use SELECT FOR UPDATE when you know you'll write
  ☐ Use SELECT FOR UPDATE SKIP LOCKED for queue processing
  ☐ Use advisory locks for app-level coordination

Database Configuration
  ☐ Using Read Committed unless Repeatable Read/Serializable is needed
  ☐ Indexes on all WHERE columns in UPDATE/DELETE statements
  ☐ lock_timeout set as a safety net (don't rely on infinite wait)
  ☐ Deadlock logging enabled (innodb_print_all_deadlocks, log_lock_waits)

Application Layer
  ☐ Detecting deadlock error codes (not just generic DB errors)
  ☐ Full transaction retry (not just the failing statement)
  ☐ Exponential backoff with jitter on retry
  ☐ Retry count limit + final error propagation
  ☐ Operations are idempotent (safe to re-run on retry)
  ☐ Deadlock count tracked as an operational metric
  ☐ Alert on deadlock rate spikes
```

### Mental Model

```
OS Level                      Database Level
────────────────────────────────────────────────────────────────
Process          ←→           Transaction (BEGIN…COMMIT)
Resource         ←→           Row / Page / Table / Index Gap
OS Scheduler     ←→           Lock Manager + Deadlock Detector
Kill process     ←→           Roll back victim transaction
Manual resolve   ←→           Automatic (engine handles it)
You prevent      ←→           You prevent + you retry
```

---

*The database always rolls back the victim completely and tells you exactly what happened via the error code. Your job: prevent where possible, retry when you're the victim, and monitor the rate.*
