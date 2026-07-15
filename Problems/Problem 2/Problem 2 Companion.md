---
tags:
  - companion
  - concurrency
  - database
  - interview
  - isolation
  - problem-2
  - problems
  - race-condition
  - system-design
---

# Companion — Isolation Levels & Race Conditions

**Domain:** Distributed Systems & Concurrency  
**Topics:** isolation levels · anomalies · race conditions

**Pairs with:** [[We lost orders during the flash sale]] · [[Problems MOC]]

---

## 1. The Data Anomalies (What We're Protecting Against)

| Anomaly                 | What Happens                                                                                                            |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Dirty Read**          | You read uncommitted data from another transaction — if it rolls back, you read data that never existed                 |
| **Non-Repeatable Read** | You read the same row twice in one transaction and get different values — another tx updated and committed in between   |
| **Phantom Read**        | You run the same range query twice and get different rows — another tx inserted/deleted matching rows                   |
| **Lost Update**         | Two txs read the same value, both modify it, second write silently overwrites the first                                 |
| **Write Skew**          | Two txs read overlapping data, make decisions independently, and together violate an invariant neither saw individually |

---

## 2. Isolation Levels

### Read Committed

**Guarantees:** No dirty reads. Re-reading the same row may return different values.

**How it handles concurrency:**

- Short lock per row while reading, released immediately after
- In MVCC databases (Postgres): each SQL statement gets a **fresh snapshot** at the moment it runs

**Anomaly Protection:**

|Anomaly|Protected?|
|---|---|
|Dirty Read|✅|
|Non-Repeatable Read|❌|
|Phantom Read|❌|
|Lost Update|❌ (need app-level control)|

**Use cases:** Most OLTP web apps, dashboards, live data reads — the default for a reason.

---

### Repeatable Read

**Guarantees:** No dirty reads, no non-repeatable reads. Re-reading the same row always returns the same value.

**How it handles concurrency:**

- Transaction gets a **snapshot frozen at start time**
- All `SELECT`s see this frozen snapshot — even if other transactions commit changes
- `UPDATE`/`DELETE` `WHERE` clauses always hit **real current committed data** — writes bypass the snapshot

**Anomaly Protection:**

| Anomaly             | Protected?                                             |
| ------------------- | ------------------------------------------------------ |
| Dirty Read          | ✅                                                      |
| Non-Repeatable Read | ✅                                                      |
| Phantom Read        | ✅ in Postgres (MVCC), ❌ in MySQL                       |
| Lost Update         | ✅ — DB detects and aborts the loser (see gotcha below) |

**Use cases:** Financial summaries, batch jobs, long-running reads that need a stable view.

---

### Serializable

**Guarantees:** Full protection against all anomalies. Transactions behave as if they ran one at a time serially.

**How it handles concurrency:**

- **Lock-based (2PL):** predicate locks on ranges — very aggressive
- **SSI (Postgres):** tracks read/write dependencies, detects dangerous cycles at commit time, aborts one tx in the cycle. Readers and writers rarely block each other but occasional **spurious aborts require retry**

**Anomaly Protection:**

|Anomaly|Protected?|
|---|---|
|Dirty Read|✅|
|Non-Repeatable Read|✅|
|Phantom Read|✅|
|Lost Update|✅|
|Write Skew|✅|

**Use cases:** Money transfers, booking/inventory systems, anywhere correctness is non-negotiable.

---

## 3. Concurrency Control Strategies

### Pessimistic Locking — `SELECT FOR UPDATE`

```sql
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;  -- acquires exclusive row lock
UPDATE orders SET status = 'done' WHERE id = 1;
COMMIT;
```

**What happens with two concurrent transactions:**

- T1 acquires lock → T2 blocks and waits
- T1 commits → T2 unblocks, reads fresh data, proceeds
- No conflict error — just queued execution

**Best isolation level: Read Committed** The lock serializes access. When T2 unblocks, you want it to see T1's fresh committed data. The lock does the work — isolation level is secondary.

---

### Optimistic Locking — Version Column

```sql
-- Both T1 and T2 read version = 5
UPDATE orders SET status = 'done', version = 6
WHERE id = 1 AND version = 5;
-- Check rows_affected == 0 → conflict detected
```

**What happens:**

- T1 commits first → row is now version 6
- T2 runs `WHERE version = 5` → 0 rows affected → app detects conflict → retry or error

**Best isolation level: Read Committed**

- Under RC: retry `SELECT` immediately sees T1's committed data ✅
- Under RR: retry `SELECT` inside the same transaction still sees the stale snapshot — you'd need a **new transaction** per retry ⚠️

---

## 4. Key Gotchas

### Gotcha 1 — Repeatable Read: SELECT vs UPDATE behavior

> Under Repeatable Read (MVCC), `SELECT` reads the frozen snapshot but `UPDATE`/`DELETE` always hit real committed data.

```sql
-- T2's snapshot frozen at version = 5
SELECT * FROM orders WHERE id = 1;       -- sees version = 5 (snapshot)

UPDATE orders SET status = 'done', version = 6
WHERE id = 1 AND version = 5;            -- hits REAL row → version is 6 → 0 rows affected ✅
```

**Why:** Writes must go to the live table. The snapshot is a read-time lens only. So optimistic locking still catches the conflict correctly even under Repeatable Read.

---

### Gotcha 2 — Repeatable Read detects Lost Updates automatically (no optimistic lock needed)

Under Repeatable Read, if two transactions do a plain `UPDATE` on the same row:

```
T1 and T2 both read same row → both try to UPDATE
T1 commits → T2 unblocks
```

**Read Committed:** T2 re-evaluates WHERE against fresh data → proceeds → **Lost Update** ❌

**Repeatable Read (Postgres):** DB detects T2's snapshot predates T1's commit on this exact row → **aborts T2**:

```
ERROR: could not serialize access due to concurrent update
```

Your app must catch this error and retry the transaction.

|Scenario|Who catches conflict|Result|
|---|---|---|
|RC + no lock|Nobody|❌ Lost update|
|RC + optimistic lock|App (`rows_affected = 0`)|✅ App retries|
|RC + `SELECT FOR UPDATE`|DB (blocking lock)|✅ Queued execution|
|RR + no lock|DB (aborts loser)|✅ DB error → app retries tx|
|RR + optimistic lock|DB aborts before version check matters|✅ Redundant but safe|

---

### Gotcha 3 — Isolation level recommendation per strategy

| Strategy                          | Recommended Level   | Why                                                   |
| --------------------------------- | ------------------- | ----------------------------------------------------- |
| Pessimistic (`SELECT FOR UPDATE`) | **Read Committed**  | Lock serializes; T2 should see fresh data after wait  |
| Optimistic (version column)       | **Read Committed**  | Clean retries; RR makes retry SELECT stale in same tx |
| No app-level control              | **Repeatable Read** | DB catches lost updates for free                      |
| Multi-row invariants / write skew | **Serializable**    | Only level that catches write skew                    |

---

## 5. Lost Update — Deep Dive

**The problem:** Two transactions do read-modify-write separately.

```
balance = 1000

T1: read 1000 → calculate 1000 + 200 = 1200 → write 1200
T2: read 1000 → calculate 1000 - 300 = 700  → write 700

Final = 700 ❌  (correct answer = 900, T1's deposit is lost)
```

The race lives in the **gap between SELECT and UPDATE**.

---

## 6. Atomic Update — The Simplest Fix

**Move the calculation into the DB — no gap, no race.**

```sql
-- ❌ App-side (gap exists)
SELECT balance FROM accounts;       -- read
-- app calculates
UPDATE accounts SET balance = 1200; -- write

-- ✅ Atomic (no gap)
UPDATE accounts SET balance = balance + 200 WHERE id = 1;
```

The DB locks the row, reads current value, computes, writes — all in one operation. No other transaction can sneak in.

```
T1 locks → reads 1000 → writes 1200 → releases
T2 locks → reads 1200 → writes 900  → releases
Final = 900 ✅
```

**Works perfectly for:**

```sql
UPDATE posts    SET views   = views   + 1     WHERE id = 1;  -- counters
UPDATE accounts SET balance = balance - 300   WHERE id = 1;  -- balances
UPDATE products SET stock   = stock   - 1     WHERE id = 1;  -- inventory
```

**Doesn't work when logic is complex** (conditions, multi-table reads, business rules) → fall back to pessimistic / optimistic / RR.

---

## 7. Decision Tree

```
Can you express the change as col = col + X?
├── Yes → Atomic UPDATE — simplest, no locks, no version column needed
└── No  → You need read-modify-write, pick a strategy:
          │
          ├── Low contention, conflict rare?
          │   └── Optimistic Lock (version column) + Read Committed
          │
          ├── High contention, conflict likely?
          │   └── Pessimistic Lock (SELECT FOR UPDATE) + Read Committed
          │
          ├── Want DB to handle it, no app changes?
          │   └── Repeatable Read — DB aborts loser, app retries tx
          │
          └── Multi-row invariants / write skew possible?
              └── Serializable
```

---

## 8. Quick Reference — Anomaly × Level Matrix

| Level           | Dirty Read | Non-Repeatable Read | Phantom Read | Lost Update      | Write Skew | Throughput |
| --------------- | ---------- | ------------------- | ------------ | ---------------- | ---------- | ---------- |
| Read Committed  | ✅          | ❌                   | ❌            | ❌                | ❌          | ⚡⚡⚡        |
| Repeatable Read | ✅          | ✅                   | ✅ (Postgres) | ✅ (aborts loser) | ❌          | ⚡⚡         |
| Serializable    | ✅          | ✅                   | ✅            | ✅                | ✅          | ⚡          |

## Next

- Scenario: [[We lost orders during the flash sale]]
- Back to hub: [[Problems MOC]]
