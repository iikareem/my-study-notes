# Database Migrations for Backend Developers

### A Complete Guide — From Scratch to Zero-Downtime Shadow Migrations

---

## Table of Contents

1. [What Is a Database Migration?](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#1-what-is-a-database-migration)
2. [Types of Schema Changes](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#2-types-of-schema-changes)
3. [The Core Problem: Live Systems](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#3-the-core-problem-live-systems)
4. [Simple Migrations (Low Traffic / Non-Critical)](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#4-simple-migrations-low-traffic--non-critical)
5. [Shadow Migration: The Safe Pattern](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#5-shadow-migration-the-safe-pattern)
6. [Dual-Write Options](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#6-dual-write-options)
7. [Scenario Walkthroughs](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#7-scenario-walkthroughs)
    - [A. Adding a New Column](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#a-adding-a-new-column)
    - [B. Renaming a Column](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#b-renaming-a-column)
    - [C. Splitting One Table into Two](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#c-splitting-one-table-into-two)
    - [D. Merging Three Tables into One](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#d-merging-three-tables-into-one)
8. [Backfilling Existing Data](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#8-backfilling-existing-data)
9. [The Expand / Contract Pattern](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#9-the-expand--contract-pattern)
10. [Rollback Strategy](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#10-rollback-strategy)
11. [Tooling Overview](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#11-tooling-overview)
12. [Checklist Before Any Migration](https://claude.ai/chat/abbb7075-1352-4a7a-9bdc-ea523a4c5f63#12-checklist-before-any-migration)

---

## 1. What Is a Database Migration?

A **database migration** is any change to the structure or content of a database that must be applied in a controlled, versioned, and repeatable way.

It covers two categories:

|Category|Examples|
|---|---|
|**Schema migration**|Adding/removing columns, renaming tables, changing data types, creating indexes|
|**Data migration**|Moving data between columns/tables, transforming values, backfilling new columns|

Migrations are almost always **both** — a schema change usually requires a data change to go with it.

---

## 2. Types of Schema Changes

### Safe changes (usually non-breaking)

- Adding a **new nullable column**
- Adding a **new table**
- Adding an **index** (with `CONCURRENTLY` in Postgres)
- Widening a column type (e.g., `VARCHAR(50)` → `VARCHAR(255)`)

### Dangerous changes (potentially breaking)

- **Renaming** a column or table
- **Dropping** a column or table
- **Narrowing** a column type
- Adding a **NOT NULL** constraint to an existing column
- Changing a column's **default value**
- **Splitting** one table into two
- **Merging** multiple tables

The dangerous ones need the shadow migration approach.

---

## 3. The Core Problem: Live Systems

When your application is running in production, the database is being read and written to **continuously**. If you run `ALTER TABLE` or `DROP COLUMN` directly, you risk:

- **Locking** the table and blocking all reads/writes
- **Crashing** old code that still references the old column name
- **Data loss** if old writes are still coming in while you're moving data
- **Inconsistency** between what the database looks like and what the app expects

The naive approach:

```
1. Deploy new code
2. Run ALTER TABLE
3. Hope nothing breaks
```

This works for toy projects. **It fails in production.**

The right approach is **shadow migration**: you never change the live schema in place. Instead, you build the new structure alongside the old one, sync data between them, then swap — all while the application stays online.

---

## 4. Simple Migrations (Low Traffic / Non-Critical)

For small projects, a maintenance window is acceptable:

```
1. Put the app in maintenance mode (block traffic)
2. Take a database backup
3. Run the migration scripts
4. Deploy the new code
5. Remove maintenance mode
6. Monitor
```

This is fine for small teams with low SLAs. It does not work for systems that must stay online 24/7.

---

## 5. Shadow Migration: The Safe Pattern

Shadow migration is the principle that **you never mutate the live structure directly**. Instead:

```
Old Structure (live) ─────────────────────────────► still works
        │                                               │
        │  (sync writes to both)                       │
        ▼                                               │
New Structure (shadow) ──── verify ──── swap ──────────┘
```

The four phases:

### Phase 1 — Expand

Add the new structure (column, table, index) **without removing the old one**. Both exist at the same time. The app still uses the old one.

### Phase 2 — Sync (Dual-Write + Backfill)

- **Dual-write**: New application code writes to **both** old and new structures simultaneously.
- **Backfill**: A background job copies all existing data from old → new. This runs in small batches so it doesn't lock the table.

### Phase 3 — Verify

Read from the new structure, cross-check against the old one. Confirm counts, spot-check values, run consistency queries.

### Phase 4 — Contract

Once verified and confident, stop writing to the old structure, switch reads to the new one, then — after a safe hold period — drop the old structure.

This can be done with **zero downtime** and **zero data loss** because at no point is the old path broken before the new path is ready.

---

## 6. Dual-Write Options

Dual-write is a **concept**, not a single implementation. There are several ways to achieve it, each with different tradeoffs depending on your stack, team size, and risk tolerance.

---

### Option 1 — Application-Level Dual-Write

The most common approach. Your code explicitly writes to both the old and new structure in the same function.

```python
def update_user_name(user_id, name):
    db.execute("UPDATE users SET name = %s WHERE id = %s", (name, user_id))
    db.execute("UPDATE users SET full_name = %s WHERE id = %s", (name, user_id))
```

**Good:** Simple, explicit, no extra infrastructure. **Bad:** Every write path in your codebase must be updated — easy to miss one. If the second write fails, you have inconsistency that needs careful handling.

---

### Option 2 — Database Trigger-Based Sync

You put the sync logic in the database itself, not the app. A trigger fires on every write and copies the value to the new column/table automatically.

```sql
CREATE OR REPLACE FUNCTION sync_full_name()
RETURNS TRIGGER AS $$
BEGIN
    NEW.full_name := NEW.name;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_full_name
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION sync_full_name();
```

**Good:** The app doesn't need to know about the migration at all — old code keeps working unchanged. Guaranteed to catch every write path. **Bad:** Triggers are invisible to developers, hard to test, and add latency to every write. They don't help with cross-table splits either.

---

### Option 3 — Change Data Capture (CDC)

You read the database's own replication log (e.g., Postgres WAL, MySQL binlog) and use that stream to sync changes to the new structure — completely outside the application.

Tools: **Debezium**, **Maxwell**, **AWS DMS**.

```
App writes to old table
       │
       ▼
  Postgres WAL ──► Debezium ──► Kafka topic ──► Consumer syncs to new table
```

**Good:** Zero application changes. Works even across multiple apps writing to the same DB. Scales well for large tables. Great for cross-service migrations. **Bad:** Requires real infrastructure (Kafka or equivalent). Introduces replication lag — the new structure is eventually consistent, not immediately consistent. Operationally complex.

---
### How to Choose

|Situation|Best option|
|---|---|
|Simple column rename/add, small team|Application-level dual-write|
|Multiple apps writing to the same DB|CDC (Debezium) or trigger|
|Good layered architecture / repository pattern|ORM layer abstraction|
|Legacy app you can't touch|Trigger or shadow view|
|Cross-service or microservice migration|CDC into a Kafka pipeline|
|Pure column rename, zero app changes wanted|Trigger (short-lived)|

Regardless of **which mechanism** you use, the **phases** are always the same — Expand, Sync, Verify, Contract. The dual-write option just determines _where_ the syncing logic lives during the sync phase.

---

## 7. Scenario Walkthroughs

---

### A. Adding a New Column

**Goal**: Add a `phone_number` column to the `users` table.

This is the simplest case because it's additive.

#### Step 1 — Add the column as nullable

```sql
ALTER TABLE users ADD COLUMN phone_number VARCHAR(20);
```

Add it nullable. If you add it as `NOT NULL` without a default, Postgres will rewrite the entire table — which locks it. Add it nullable first, backfill it, then add the constraint.

#### Step 2 — Backfill existing rows (if needed)

```sql
-- Run in batches, not all at once
UPDATE users
SET phone_number = legacy_phone_data.phone
FROM legacy_phone_data
WHERE users.id = legacy_phone_data.user_id
  AND users.id BETWEEN :batch_start AND :batch_end;
```

#### Step 3 — Add NOT NULL constraint (after backfill is complete)

```sql
-- Postgres 12+: check constraint first, then convert (avoids full table scan)
ALTER TABLE users ADD CONSTRAINT users_phone_not_null
  CHECK (phone_number IS NOT NULL) NOT VALID;

ALTER TABLE users VALIDATE CONSTRAINT users_phone_not_null;

ALTER TABLE users ALTER COLUMN phone_number SET NOT NULL;

ALTER TABLE users DROP CONSTRAINT users_phone_not_null;
```

---

### B. Renaming a Column

**Goal**: Rename `users.name` → `users.full_name`.

You **cannot** just run `ALTER TABLE users RENAME COLUMN name TO full_name` in production, because old application code still references `name`.

#### Step 1 — Add the new column

```sql
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
```

#### Step 2 — Deploy dual-write code

Your application now writes to **both** `name` and `full_name` on every insert/update:

```python
def update_user(user_id, name_value):
    db.execute("""
        UPDATE users
        SET name = %s, full_name = %s
        WHERE id = %s
    """, (name_value, name_value, user_id))
```

#### Step 3 — Backfill

```sql
UPDATE users
SET full_name = name
WHERE full_name IS NULL
  AND id BETWEEN :start AND :end;
```

#### Step 4 — Verify

```sql
SELECT COUNT(*) FROM users WHERE full_name IS NULL;
SELECT COUNT(*) FROM users WHERE full_name != name;
-- Both should return 0
```

#### Step 5 — Switch reads to new column

Deploy code that reads from `full_name` instead of `name`. Still dual-writing.

#### Step 6 — Remove writes to old column

Deploy code that only writes to `full_name`. Stop writing to `name`.

#### Step 7 — Drop old column (after safe hold period)

```sql
ALTER TABLE users DROP COLUMN name;
```

---

### C. Splitting One Table into Two

**Goal**: Split `orders` into `orders` + `order_items`.

Before:

```
orders
  id, user_id, product_id, quantity, price, created_at
```

After:

```
orders              order_items
  id                  id
  user_id             order_id (FK → orders.id)
  created_at          product_id
                      quantity
                      price
```

#### Step 1 — Create the new table

```sql
CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    product_id  BIGINT NOT NULL,
    quantity    INT NOT NULL,
    price       NUMERIC(10, 2) NOT NULL
);
```

#### Step 2 — Dual-write in the application

Every new order write now creates both an `orders` row and an `order_items` row:

```python
def create_order(user_id, product_id, quantity, price):
    with db.transaction():
        order_id = db.execute("""
            INSERT INTO orders (user_id, created_at)
            VALUES (%s, NOW()) RETURNING id
        """, (user_id,)).fetchone()[0]

        db.execute("""
            INSERT INTO order_items (order_id, product_id, quantity, price)
            VALUES (%s, %s, %s, %s)
        """, (order_id, product_id, quantity, price))
```

#### Step 3 — Backfill historical data

```sql
INSERT INTO order_items (order_id, product_id, quantity, price)
SELECT id, product_id, quantity, price
FROM orders
WHERE id BETWEEN :start AND :end
  AND NOT EXISTS (
    SELECT 1 FROM order_items WHERE order_items.order_id = orders.id
  );
```

#### Step 4 — Verify

```sql
SELECT COUNT(*) FROM orders o
LEFT JOIN order_items oi ON oi.order_id = o.id
WHERE oi.id IS NULL;
-- Should be 0: every order has a corresponding order_item
```

#### Step 5 — Switch reads to new structure

Update all queries that join or reference the new `order_items` table.

#### Step 6 — Remove old columns from orders

```sql
ALTER TABLE orders
  DROP COLUMN product_id,
  DROP COLUMN quantity,
  DROP COLUMN price;
```

---

### D. Merging Three Tables into One

**Goal**: Merge `cats`, `dogs`, `birds` into a single `pets` table (polymorphic model).

#### Step 1 — Create the unified table

```sql
CREATE TABLE pets (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    type       VARCHAR(10) NOT NULL CHECK (type IN ('cat', 'dog', 'bird')),
    name       VARCHAR(255) NOT NULL,
    breed      VARCHAR(255),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Step 2 — Backfill all three sources

```sql
INSERT INTO pets (user_id, type, name, breed, created_at)
SELECT user_id, 'cat', name, breed, created_at FROM cats;

INSERT INTO pets (user_id, type, name, breed, created_at)
SELECT user_id, 'dog', name, breed, created_at FROM dogs;

INSERT INTO pets (user_id, type, name, breed, created_at)
SELECT user_id, 'bird', name, NULL, created_at FROM birds;
```

#### Step 3 — Dual-write period

New writes go to both the type-specific table and `pets`. This is the critical window.

#### Step 4 — Add indexes

```sql
CREATE INDEX CONCURRENTLY idx_pets_user_id ON pets (user_id);
CREATE INDEX CONCURRENTLY idx_pets_type ON pets (type);
```

#### Step 5 — Switch reads, verify, then drop old tables

```sql
-- Only after full verification and a hold period
DROP TABLE cats;
DROP TABLE dogs;
DROP TABLE birds;
```

---

## 8. Backfilling Existing Data

A backfill on a large table must **never** run as a single query. It will lock the table and kill performance.

### Batched backfill pattern

```python
BATCH_SIZE = 1000

def run_backfill():
    min_id, max_id = db.execute(
        "SELECT MIN(id), MAX(id) FROM users"
    ).fetchone()

    batch_start = min_id
    while batch_start <= max_id:
        batch_end = batch_start + BATCH_SIZE - 1

        db.execute("""
            UPDATE users
            SET full_name = name
            WHERE id BETWEEN %s AND %s
              AND full_name IS NULL
        """, (batch_start, batch_end))

        batch_start = batch_end + 1
        time.sleep(0.05)  # breathing room for the DB
```

### Rules for safe backfills

- Always use `WHERE new_column IS NULL` to make the backfill **idempotent** (safe to re-run).
- Use a small sleep between batches (50–100ms) to avoid saturating I/O.
- Monitor replication lag if you have read replicas — a heavy backfill can push replicas behind.
- Log progress so you can resume from where you stopped if the job is interrupted.

---

## 9. The Expand / Contract Pattern

This is the formal name for what shadow migrations do, and it applies to **any** breaking schema change.

```
EXPAND ──────────────────────────────────────────────────────────┐
  • Add new column / table (non-breaking, additive)              │
  • Deploy code that writes to both old and new                  │
  • Backfill old data into new structure                         │
                                                                  │
VERIFY ───────────────────────────────────────────────────────── │
  • Assert row counts match                                       │
  • Assert no NULLs where there shouldn't be                     │
  • Assert sample values are correct                              │
                                                                  │
CONTRACT ────────────────────────────────────────────────────────┘
  • Deploy code that reads only from new structure               │
  • Deploy code that writes only to new structure                │
  • Wait (hold period: usually 1 deploy cycle or a few days)    │
  • Drop old column / table                                      │
```

The hold period before dropping is important. You want to be able to roll back to old code if you discover a bug — and old code may still reference the old column. If you drop it immediately, rollback is impossible.

---

## 10. Rollback Strategy

Every migration must have a rollback plan **written before the migration runs**.

|Phase|Rollback action|
|---|---|
|After Expand (new column added)|Drop the new column|
|During Dual-Write|Deploy previous app version; new column has data but old column was also being written to|
|After Backfill|Revert app to old code; old column still has correct data|
|After Contract (old column dropped)|**No easy rollback** — this is why the hold period exists|

### The point of no return

Once you drop the old column or table, rollback requires restoring from a backup. This is why the contract phase must only happen after you're fully confident.

**Always take a backup before the contract phase.**

```bash
# Postgres example
pg_dump --table=users --data-only mydb > users_backup_before_drop.sql
```

---

## 11. Tooling Overview

|Tool|Language|What it handles|
|---|---|---|
|**Flyway**|JVM / any|SQL-based versioned migrations, checksum validation|
|**Liquibase**|JVM / any|XML/YAML/SQL changelogs, rollback support|
|**Alembic**|Python|SQLAlchemy-native, auto-generates diffs|
|**Django Migrations**|Python|Built-in ORM migrations with state tracking|
|**Rails Active Record Migrations**|Ruby|Convention-driven, integrated with ORM|
|**golang-migrate**|Go|SQL file-based, supports many DBs|
|**Prisma Migrate**|Node.js / TypeScript|Schema-first, diff-based|
|**gh-ost**|Any (GitHub)|Online schema change for MySQL with shadow table built-in|
|**pgroll**|Postgres|Expand/contract automation for Postgres|

For Postgres specifically, always prefer:

- `CREATE INDEX CONCURRENTLY` over `CREATE INDEX` (non-blocking)
- `NOT VALID` constraints followed by `VALIDATE CONSTRAINT` (avoids full table scan lock)
- `ALTER TABLE ... ADD COLUMN` with a nullable default (Postgres 11+ doesn't rewrite the table for this)

---

## 12. Checklist Before Any Migration

```
Pre-migration
  [ ] Full database backup taken
  [ ] Rollback SQL written and tested in staging
  [ ] Staging migration ran successfully
  [ ] Row counts and data spot-checked in staging
  [ ] Batch size tested — no lock contention in staging
  [ ] Monitoring and alerts are active
  [ ] Team is on standby during migration window

During migration
  [ ] Backfill running in batches (not a single UPDATE)
  [ ] Replication lag being monitored
  [ ] Application error rate being monitored
  [ ] Each phase committed separately (not one giant transaction)

Post-migration
  [ ] Counts verified: old count == new count
  [ ] NULL check: no unexpected NULLs in new structure
  [ ] Spot check: sampled rows look correct
  [ ] Application behavior tested end-to-end
  [ ] Old structure on hold period (not dropped immediately)
  [ ] Old structure dropped only after hold period passes
```

---

## Summary

The mental model to internalize:

> **Never break what's running. Build alongside it, sync it, verify it, then swap. Only clean up the old thing after you're sure the new thing works.**

That is shadow migration. Whether you're adding a column, renaming one, splitting a table into two, or collapsing three into one — the phases are always the same: **Expand → Sync → Verify → Contract**. Each phase is independently deployable. No data loss. No downtime. No surprises.