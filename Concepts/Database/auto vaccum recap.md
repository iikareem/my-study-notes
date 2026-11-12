# PostgreSQL VACUUM: Recap

## 1. Why VACUUM exists

- Postgres uses **MVCC**: readers don't block writers, and writers don't block readers.
- The cost is that `UPDATE` and `DELETE` leave old row versions behind as **dead tuples**.
- Dead tuples cause **bloat**: wasted space, slower scans, and larger indexes.
- VACUUM removes dead tuples that no transaction can see anymore and marks their space **reusable inside the table**.
- It returns space to the OS only when **empty pages at the end of the file** can be truncated.

## 2. Cleanup horizon

- It is the **oldest xmin** across all backends, replication slots, and prepared transactions.
- A tuple is removable only if its `xmax` is committed **and** older than the horizon.
- Long transactions, idle-in-transaction sessions, and stale replication slots **hold the horizon back**, which causes bloat.
- Repeatable Read and Serializable hold the snapshot for the whole transaction. Read Committed takes a snapshot per statement.

## 3. Lazy vacuum: 3 phases

1. **Heap scan:** collect dead TIDs into memory (`maintenance_work_mem`).
2. **Index vacuum:** scan **every index** and remove entries pointing to those TIDs. This is the costliest phase.
3. **Heap vacuum:** remove the dead tuples, then update the FSM and VM.

- Index entries must be removed **before** the heap slots, otherwise an index could point at a reused slot.
- If the TID memory fills up, the cycle **repeats**, so the indexes are scanned again. More memory means fewer passes.

## 4. Visibility map (VM)

- 2 bits per page: **all-visible** and **all-frozen**.
- Vacuum **skips** these pages (normal vacuum skips all-visible, aggressive skips all-frozen).
- It also enables **index-only scans**.
- A bit is cleared when the page is modified.

## 5. Free space map (FSM)

- Tracks free space **per page**, so inserts and updates know where a row fits.
- Updated mainly by **vacuum**.

## 6. Freezing and wraparound

- Transaction IDs are **32-bit**, compared circularly (about 2B in the past and 2B in the future).
- Without action, old rows would eventually look like they are in the **future** and become invisible. That is **wraparound**.
- **Freezing** marks old tuples as "visible to everyone" using a flag in the tuple header, so their xmin no longer matters.
- `relfrozenxid` (table) and `datfrozenxid` (database) track the oldest unfrozen XID. **Age** = current XID − `relfrozenxid`.
- Key settings: `vacuum_freeze_min_age`, `vacuum_freeze_table_age`, `autovacuum_freeze_max_age` (default 200M).
- If this is not handled, Postgres eventually **refuses writes**.

## 7. Normal vs aggressive vacuum

||Normal (lazy)|Aggressive|
|---|---|---|
|Skips|all-visible pages|only all-**frozen** pages|
|Freezing|opportunistic|must advance `relfrozenxid`|
|Triggered by|dead-tuple or insert thresholds|age passes `vacuum_freeze_table_age`, or anti-wraparound autovacuum|

- Anti-wraparound autovacuum **cannot be cancelled** the way a normal one can.

## 8. Autovacuum architecture

- A **launcher** starts **workers** (`autovacuum_max_workers`).
- Each worker handles **one table at a time**, and no two workers process the same table.
- **Cost-based throttling** (`autovacuum_vacuum_cost_limit` and `autovacuum_vacuum_cost_delay`) is **shared across workers** so vacuum doesn't hurt production.
- Override per table:
    
    ```sql
    ALTER TABLE t SET (autovacuum_vacuum_scale_factor = 0.01, autovacuum_vacuum_cost_limit = 2000);
    ```
    

## 9. ANALYZE (statistics)

- ANALYZE is **separate** from vacuum.
- It samples the table and updates:
    - per column, in `pg_statistic` (via `pg_stats`): `n_distinct`, null fraction, MCV, histogram, correlation
    - per table, in `pg_class`: `reltuples` and `relpages`
- The planner uses these for **cardinality estimation** and plan choice.
- It can be run manually, with `VACUUM ANALYZE`, or by **auto-analyze**, which has its own threshold based on changed rows.
- Run `ANALYZE` manually after mass loads or big deletes.

## 10. VACUUM FULL

||Plain `VACUUM`|`VACUUM FULL`|
|---|---|---|
|Method|marks space reusable in place|**rewrites the table into a new file**|
|Space returned to OS|only trailing empty pages|**all of it**|
|Indexes|cleaned|**rebuilt**|
|Lock|light, online|**`ACCESS EXCLUSIVE`**, blocks reads and writes|
|Extra disk|no|**yes**, about the size of the live data|

- Use it **rarely**, for example after a one-time massive bloat.
- For steady-state bloat, **tune autovacuum** instead.
- For an **online** rewrite, use **`pg_repack`** (or `pg_squeeze`). It builds a new copy, captures changes with a trigger and log table, then swaps with brief locks. It needs a primary key or unique index.
- `CLUSTER` rewrites in index order but takes the same exclusive lock.

## 11. Practical production notes

- **Monitor** with `pg_stat_user_tables`: `n_dead_tup`, `last_autovacuum`, `last_autoanalyze`.
- **Big tables:** the default `autovacuum_vacuum_scale_factor` is **0.2**, so a 100M-row table waits for about 20M dead tuples. Lower it per table.
- **HOT updates and `fillfactor`:** an update that changes no indexed column can stay on the same page without touching indexes. Leave free space (`fillfactor` below 100) to help.
- **Index bloat:** vacuum doesn't shrink indexes. Use `REINDEX CONCURRENTLY`.

## Interview answer: "Your table is bloated, what do you do?"

1. Check for **long transactions** and **stale replication slots** holding the horizon.
2. **Tune autovacuum** for that table (lower the scale factor, raise the cost limit).
3. Use **`pg_repack`** only if you must reclaim disk space online.
4. Use `VACUUM FULL` only with a maintenance window.

## One-line summary

> MVCC leaves dead tuples → vacuum removes those older than the cleanup horizon (heap → indexes → heap) → VM lets it skip pages, FSM reuses space → freezing prevents XID wraparound → autovacuum runs it, throttled, per table → ANALYZE refreshes stats → VACUUM FULL / pg_repack reclaim disk space.