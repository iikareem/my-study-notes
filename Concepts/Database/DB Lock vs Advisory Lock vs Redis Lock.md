# Notes: DB Lock vs Advisory Lock vs Redis Lock

## 1. DB row lock (`SELECT ... FOR UPDATE`)

- **What it is:** locks a row inside a transaction.
- **Released when:** the transaction commits or rolls back, or the connection dies. This is automatic and reliable.
- **Good for:** work that fits entirely inside one database.
- **Cost:** the transaction holds **one pool connection** for as long as the lock is held. A slow call (e.g. a bank API) inside the transaction means a busy connection doing nothing. With many workers, the pool runs out and other queries queue up.
- **Rule:** keep transactions short and never put a network call to an outside system inside one.

## 2. Advisory lock (PostgreSQL)

- **What it is:** a DB lock on an arbitrary number you choose, not tied to a row.

|Type|Belongs to|Released when|Pooling|
|---|---|---|---|
|`pg_advisory_lock(key)`|The connection|Unlock or disconnect|Needs PgBouncer **session mode** (unsafe in transaction mode)|
|`pg_advisory_xact_lock(key)`|The transaction|Commit or rollback|Works in both modes|

- **Solves:** mutual exclusion inside the DB, dead holders (auto-release), no TTL guessing, no failover problem.
- **Doesn't solve:**
    - It still holds a connection or transaction for its whole duration.
    - If the connection drops, the lock is released while the worker may still be running, so two workers can overlap.
    - It can't protect external side effects such as the bank call.
- **Good for:** short work fully inside the database.

## 3. Redis lock (`SET key token NX PX ttl`)

- **What it is:** a lock in an external service, with a TTL (a lease).
- **Good for:** coordinating across services, or holding a lock over slow external work **without holding a DB connection**.
- **Required details:**
    - Atomic `SET NX PX`, never `SETNX` then `EXPIRE`.
    - Unique token as the value.
    - Release with a Lua compare-and-delete.
    - Lease renewal (watchdog) for long work.
- **Weaknesses:**
    - The TTL can expire while the holder is still working.
    - A GC pause or clock drift means the holder can't know it lost the lock.
    - Async replication means a failover can lose the lock.
- **So:** a Redis lock gives **efficiency** (fewer conflicts), not **correctness**.

## 4. Side-by-side

||Row lock|Advisory lock|Redis lock|
|---|---|---|---|
|Lives in|DB|DB|Separate service|
|Holder crash|Auto-released|Auto-released|Released when TTL expires|
|Holds a DB connection|Yes|Yes|No|
|Can expire mid-work|No|No (but connection loss can drop it)|Yes|
|Covers external calls|No|No|No (needs idempotency)|
|Needs fencing for correctness|No, if the write is in the same transaction|Rarely|Yes|

## 5. Decision rule

1. **Work fits in one database?** Use DB locks, transactions and constraints.
2. **Work is long or touches external systems?** Claim the job with a status column, keep transactions short, and use idempotency keys with the external service.
3. **Need a real distributed lock?** Use Redis (or etcd/ZooKeeper) for efficiency, and keep correctness in the database with a state check and fencing token.

## 6. The claim pattern (what to do instead of holding a lock)

```sql
-- 1. short transaction: claim
UPDATE payouts SET status='IN_PROGRESS', worker=:me
WHERE id=:id AND status='PENDING';          -- 1 row = you own it

-- 2. call the external service (no transaction, no lock held), with an idempotency key

-- 3. short transaction: finish
UPDATE payouts SET status='PAID'
WHERE id=:id AND status='IN_PROGRESS' AND worker=:me;
```

## One-line takeaways

- A DB lock lasts as long as its transaction, and the transaction holds a connection.
- Advisory locks are DB locks you name yourself. Session-level needs session mode, and transaction-level doesn't.
- A Redis lock is a lease. It can expire while you're working, so it can't be the only protection.
- **The lock makes conflicts rare. The database checks (state, token, idempotency) make results correct.**