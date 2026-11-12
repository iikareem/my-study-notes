# MVCC (Multi-Version Concurrency Control)

> MVCC keeps multiple versions of each row so every transaction reads from a consistent snapshot. Readers don't block writers, writers don't block readers. The **storage engine** creates and cleans up versions. The **isolation level** is the policy that decides how they're used.

---

## 1. The problem it solves

- Without MVCC, consistent reads need **locking (2PL)**: readers take shared locks and writers take exclusive locks.
- Shared and exclusive locks conflict, so **reads block writes and writes block reads**.
- Result: low throughput and more **deadlocks**.
- With MVCC, each transaction works on its own snapshot.
- **Writers still conflict with other writers** on the same row. Row locks and the isolation level handle that.

---

## 2. Where old versions live

| Engine | Version storage | Cleanup |
|---|---|---|
| **PostgreSQL** | In the **heap**, next to the live tuple | **VACUUM** |
| **MySQL/InnoDB** | In the **undo log**; the table holds the latest row, older versions are rebuilt from undo | **Purge** |
| **Oracle** | Undo segments | Undo retention |

**Trade-off:**
- Postgres has a simple read path, but suffers **bloat** and needs vacuum.
- InnoDB keeps the table cleaner, but reading an old version means walking the undo chain.
- Long transactions hurt both.

---

## 3. Tuple header and snapshot (Postgres)

- Each tuple has a **23-byte header** with:
  - **xmin**: the transaction that created it.
  - **xmax**: the transaction that deleted or updated it.
- The header lives on the **tuple**, not the page header.
- An `UPDATE` is a delete plus an insert: it sets xmax on the old tuple and creates a new tuple with a new xmin.
- A **snapshot** records:
  - **xmin**: the oldest still-active transaction.
  - **xmax**: the next transaction ID to be assigned.
  - **xip**: the list of in-progress transactions.

### Visibility rule (the same everywhere)

> A version is visible if its **creator committed before the snapshot** and its **deleter did not commit** (or is not visible to the snapshot).

---

## 4. Isolation level = the policy

The visibility rule is fixed. The isolation level decides **when the snapshot is taken** and **how conflicts are handled**.

| Level | Snapshot | Behavior | On write conflict |
|---|---|---|---|
| **Read Committed** | Per **statement** | Each statement sees the latest committed data; non-repeatable reads are possible | Re-reads the latest version and continues |
| **Repeatable Read** | Per **transaction** | One consistent view for the whole transaction | Aborts with an error; **retry** |
| **Serializable** (SSI) | Per transaction, plus dependency tracking | Detects **write skew** | Aborts on dangerous read-write cycles; **retry** |

- **Write skew:** two transactions read overlapping data, then write different rows, and each write is valid only given what was read.
  - Example: two doctors both go off-call because each saw the other still on-call.
- **Serializable in Postgres** tracks read-write dependencies (SIREAD locks) and raises a **serialization failure**. The application must retry.

### Choosing a level from business constraints

| Business need | Use | Cost |
|---|---|---|
| Fresh data per statement, minor anomalies acceptable (most CRUD/APIs) | **Read Committed** | Don't compute from two reads in one transaction |
| Consistent view across several reads (reports, balances, exports) | **Repeatable Read** | Retry update conflicts; write skew still possible |
| Invariants across multiple rows (on-call rules, seat or limit checks) | **Serializable**, or RR plus `SELECT ... FOR UPDATE` | Application must retry serialization failures |

**Rule of thumb:** use the lowest level that protects your invariant. If a rule depends on rows you only *read*, you need Serializable or explicit row locks.

---

## 5. Locks

- **Table-level locks:** modes run from `ACCESS SHARE` (plain SELECT) up to `ACCESS EXCLUSIVE` (DROP, most ALTER TABLE, VACUUM FULL).
  - `ACCESS EXCLUSIVE` blocks everything, including reads.
- **Row-level locks:** `SELECT ... FOR UPDATE` and `FOR SHARE`.
  - In Postgres they are stored in the **tuple header** (xmax plus flag bits), so locking a row **dirties the page**.
- **Plain SELECT takes no row locks** under MVCC.

---

## 6. Deadlocks

- Two transactions each hold a lock the other needs, so they wait in a cycle.
- The database **detects the cycle** (Postgres checks after `deadlock_timeout`, 1s by default) and **aborts one transaction**.
- **Prevention:**
  - Access tables and rows in the **same order**.
  - Keep transactions **short**.
  - Use a consistent `ORDER BY` in `FOR UPDATE` queries.

---

## 7. Production must-knows

- **Long transactions are the main enemy.** They hold the cleanup horizon back, so dead versions pile up (bloat in Postgres, undo growth in InnoDB).
- Watch for **`idle in transaction`** sessions. They keep snapshots alive and block vacuum.
- Tune **autovacuum** on hot, write-heavy tables.
- **Always code retry logic** for deadlocks and serialization failures.
- Heavy DDL (`ACCESS EXCLUSIVE`) can stall a whole table. Use short lock timeouts.
- Pick the isolation level **per business invariant**, not globally by habit.

---

## One-liner

> MVCC is the mechanism in the storage engine: versions plus snapshots plus a fixed visibility rule. The isolation level is the policy that decides which snapshot you read from and how conflicts are handled. The price is storing old versions, which means vacuum in Postgres and purge in InnoDB.