# Note: `SELECT ... FOR UPDATE`

## What it does
- Locks the rows matched by the `WHERE` until the transaction commits or rolls back.
- Used to prevent lost updates in read-modify-write flows.

## What it blocks
- Other `SELECT ... FOR UPDATE` / `FOR SHARE` on the same rows (they wait).
- `UPDATE` and `DELETE` on the same rows (they wait).

## What it does not block
- Plain `SELECT`. It uses MVCC and reads the last committed version.
  - Exception: MySQL `SERIALIZABLE` turns plain selects into locking reads.
- Rows that don't match the `WHERE`.
- In PostgreSQL, inserts of new rows matching the `WHERE` (phantoms).

## What a plain SELECT sees
- **READ COMMITTED**: latest committed data, per statement.
- **REPEATABLE READ**: snapshot from the start of the transaction.
- It never sees uncommitted changes.

## Variants
- `NOWAIT`: fail immediately if the row is locked.
- `SKIP LOCKED`: skip locked rows (good for job queues).
- `FOR SHARE`: allows other readers with `FOR SHARE`, blocks writers.

## Gotchas
- Keep the transaction short, because the lock is held until commit.
- Always lock rows in a consistent order to avoid deadlocks.
- MySQL: an unindexed `WHERE` can lock far more rows than expected, and REPEATABLE READ can add gap locks.

## Example
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

## One-liner
> `FOR UPDATE` takes an exclusive row lock so concurrent writers and other locking reads wait, while plain reads still see a committed snapshot.