---
tags:
  - backpressure
  - concurrency
  - interview
  - isolation
  - messaging
  - problem-2
  - problems
  - race-condition
  - reliability
  - system-design
---

# Problem 2 — "We lost orders during the flash sale"

**Domain:** Distributed Systems & Concurrency  
**Topics:** race conditions · isolation levels · inventory · concurrency

**In this folder:** scenario (this note) · [[Problem 2 Companion]]

## 🧩 The Scenario

You work at an e-commerce company. Every Friday they run a flash sale — limited stock, high demand. The `inventory` table looks like this:

```sql
CREATE TABLE inventory (
  product_id UUID PRIMARY KEY,
  stock      INTEGER NOT NULL CHECK (stock >= 0)
);
```

The order service is horizontally scaled — **8 instances** running behind a load balancer. The purchase flow looks like this:

```python
def purchase(product_id, user_id, quantity):
    stock = db.query("SELECT stock FROM inventory WHERE product_id = ?", product_id)

    if stock < quantity:
        raise OutOfStockError()

    db.execute("UPDATE inventory SET stock = stock - ? WHERE product_id = ?", quantity, product_id)
    db.execute("INSERT INTO orders (...) VALUES (...)")
    return "Order placed"
```

You had **100 units** in stock. After the sale ends you count **137 orders placed** for that product. Customers are furious. Your manager asks: _what happened and how do we fix it?_

---

## 🤔 Think About It First

There are **2 distinct bugs** and **3 valid fix strategies** with different tradeoffs. What do you see?

---

## 🔍 Root Cause Analysis

### Bug 1 — Classic Race Condition (Lost Update)

The flow is:

```
Instance A: SELECT stock → gets 10
Instance B: SELECT stock → gets 10   ← same value, read before A updated
Instance A: stock >= 1 ✅ → UPDATE stock = 9
Instance B: stock >= 1 ✅ → UPDATE stock = 9  ← should be 8, not 9
```

This is the **check-then-act** antipattern. Between the `SELECT` and the `UPDATE`, another transaction already changed the world. Your check is stale the moment you read it.

With 8 instances hammering the same row during a flash sale, this happens **constantly**.

---

### Bug 2 — The `SELECT` and `UPDATE` are not atomic

They are two separate database round trips. Even inside a transaction, at the default `READ COMMITTED` isolation level:

```python
# This is NOT protected by default isolation
BEGIN;
SELECT stock ...   ← snapshot taken here
# ... 5ms gap, other transactions commit ...
UPDATE inventory ... ← operates on potentially stale assumption
COMMIT;
```

At `READ COMMITTED`, each statement gets a **fresh snapshot** — the SELECT and UPDATE don't share the same view of the world.

---

## ✅ Fix Strategies

### Strategy 1 — Optimistic Locking (High throughput, low contention)

Add a `version` column. Read it, then assert it hasn't changed at write time:

```sql
ALTER TABLE inventory ADD COLUMN version INTEGER DEFAULT 0;
```

```python
def purchase(product_id, user_id, quantity):
    row = db.query(
        "SELECT stock, version FROM inventory WHERE product_id = ?",
        product_id
    )

    if row.stock < quantity:
        raise OutOfStockError()

    updated = db.execute("""
        UPDATE inventory
        SET stock = stock - ?, version = version + 1
        WHERE product_id = ? AND version = ?   -- ← the guard
    """, quantity, product_id, row.version)

    if updated.rowcount == 0:
        raise ConflictError("Retry")  # someone else updated first

    db.execute("INSERT INTO orders ...")
```

```
Thread A: reads version=5, stock=10
Thread B: reads version=5, stock=10
Thread A: UPDATE WHERE version=5 → succeeds, version becomes 6
Thread B: UPDATE WHERE version=5 → 0 rows affected → retries
```

**Tradeoff:** High retry rate under extreme contention (flash sale = worst case). Fine for low-to-medium concurrency.

---

### Strategy 2 — Pessimistic Locking with `SELECT FOR UPDATE`

Lock the row at read time so no other transaction can touch it until you commit:

```python
def purchase(product_id, user_id, quantity):
    with db.transaction():
        row = db.query("""
            SELECT stock FROM inventory
            WHERE product_id = ?
            FOR UPDATE          -- ← acquires row-level lock
        """, product_id)

        if row.stock < quantity:
            raise OutOfStockError()

        db.execute("""
            UPDATE inventory SET stock = stock - ?
            WHERE product_id = ?
        """, quantity, product_id)

        db.execute("INSERT INTO orders ...")
```

```
Thread A: SELECT FOR UPDATE → acquires lock
Thread B: SELECT FOR UPDATE → BLOCKS here
Thread A: UPDATE, COMMIT → releases lock
Thread B: unblocks, reads fresh stock=9, proceeds
```

**Tradeoff:** Serializes all purchases of one product. Safe and simple, but creates a **lock queue** — throughput is limited to how fast you can process one transaction at a time per product.

---

### Strategy 3 — Atomic Update + Check in One Statement (Best for this case)

Skip the SELECT entirely. Push the guard logic into the UPDATE itself:

```python
def purchase(product_id, user_id, quantity):
    updated = db.execute("""
        UPDATE inventory
        SET stock = stock - ?
        WHERE product_id = ?
          AND stock >= ?        -- ← guard is atomic with the write
    """, quantity, product_id, quantity)

    if updated.rowcount == 0:
        raise OutOfStockError()  # either not found or insufficient stock

    db.execute("INSERT INTO orders ...")
```

The database evaluates `AND stock >= ?` and applies `stock - ?` **atomically in a single operation**. There is no window between check and update.

This works because the `CHECK (stock >= 0)` constraint also acts as a safety net at the DB level.

**Tradeoff:** You lose the ability to distinguish "product not found" from "out of stock" without an extra query. Simple to reason about, highly concurrent, no retries needed.

---

## 📊 Strategy Comparison

|Strategy|Concurrency|Complexity|Retry needed|Best for|
|---|---|---|---|---|
|Optimistic locking|High|Medium|Yes|Low-medium contention|
|`SELECT FOR UPDATE`|Low (serialized)|Low|No|Safety-critical, low traffic|
|Atomic update|High|Low|No|Flash sales, high contention|

---

## 🧠 The Deeper Lesson — Database Isolation Levels

This problem exists _because_ of how isolation works:

|Level|Behavior|Prevents|
|---|---|---|
|`READ COMMITTED`|Each statement sees latest committed data|Dirty reads|
|`REPEATABLE READ`|Transaction sees snapshot from start|Dirty + non-repeatable reads|
|`SERIALIZABLE`|Transactions behave as if sequential|All anomalies including phantom reads|

Even `SERIALIZABLE` doesn't save the naive `SELECT` then `UPDATE` pattern — it would just make one transaction fail with a serialization error, forcing a retry. The atomic update is still cleaner.

---

## 🔑 Key Takeaways

|Lesson|Rule|
|---|---|
|Check-then-act|Never split a check and its mutation across two round trips|
|Optimistic vs Pessimistic|Optimistic = retry on conflict. Pessimistic = block on conflict|
|Atomic updates|Push guards into the `WHERE` clause when possible|
|DB constraints|`CHECK (stock >= 0)` is your last line of defense — always have it|
|Isolation ≠ atomicity|A transaction doesn't automatically make multi-step logic atomic|

---

## Next

- Companion deep dive: [[Problem 2 Companion]]
- Back to hub: [[Problems MOC]]
