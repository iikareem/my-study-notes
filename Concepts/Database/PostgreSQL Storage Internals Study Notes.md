# PostgreSQL Storage Internals: Study Notes

Topics: heap file layout, TOAST, side tables, buffer cache and clock sweep, ring buffers, and how they connect in production.

Each topic ends with a **Mix-ups corrected** block listing the points that were confused or wrong in the first version of the notes.

---

## 1. Heap File Layout

### What is correct

- A table's default storage is a **heap**: rows are stored **unordered**, wherever there is space.
- The unit of storage and I/O is the **8KB page**. Reading one row reads the whole page into memory.
- A page has this layout:

```
+--------------------------------------------+
| Page header                                |
| Line pointer array (grows downward)        |
|        ...  free space  ...                |
|          pd_lower ^   v pd_upper           |
| Tuples (grow upward from the page end)     |
+--------------------------------------------+
```

- **Page header** holds:
    - the **LSN**, the WAL position of the last change to the page
    - a checksum and flags
    - `pd_lower`, the end of the line pointer array
    - `pd_upper`, the start of tuple data
    - `pd_special`
    - Free space is the gap between `pd_lower` and `pd_upper`.
- **Line pointers** are 4 bytes each. They are an indirection layer: each stores an offset, a length, and a 2-bit state (unused, normal, redirect, dead).
- **Tuples** are the row versions. Each has a header of about 23 bytes, followed by a null bitmap if needed, alignment padding, and then the column data.
- **TID (the row's address)** is `(page number, line pointer index)`. Indexes store TIDs, so they point to the line pointer, and the line pointer points to the tuple. This indirection lets the tuple move inside the page without touching any index.
- **Tuple header fields for MVCC:**
    - `xmin`: the transaction that created this version.
    - `xmax`: the transaction that deleted or updated it.
    - `t_ctid`: points to the newer version of the row, or to itself if this is the latest.
- **An UPDATE is a delete plus an insert.** The old version gets `xmax` set and stays on the page until VACUUM removes it. The new version is a fresh tuple whose address is stored in the old one's `t_ctid`.
- **HOT updates:** if the new version fits on the same page and no indexed column changed, the indexes don't need new entries. The old line pointer redirects to the new tuple.

### Mix-ups corrected

> **Line pointer size.** It is **4 bytes**, not 8.

> **LSN in the page header.** It is **not a checkpoint**. It is the WAL position of the last change to this page.

> **"New current tuple identifier."** This is `t_ctid`, the pointer from an old version to its newer version.

> **"MVCC doesn't change anything on disk."** It does write: an update sets `xmax` on the old tuple and adds a new tuple. The old version is kept only so that other transactions can still see it.

> **Clustered index vs. Postgres.** `CLUSTER` in Postgres is a **one-time physical reorder** and is not maintained afterwards. New rows go wherever there is space. A clustered index that stays ordered, as in InnoDB or SQL Server, is a different design.

### Why it matters in production

- **Row width:** fewer rows per page means more I/O per query.
- **Dead tuples (MVCC):** updates leave old versions behind, which becomes bloat until VACUUM cleans it.
- **`fillfactor`:** leaving free space in each page (for example 80-90) makes HOT updates possible on update-heavy tables.
- **Physical order:** range scans on a low-correlation column jump between random pages.

---

## 2. TOAST (Out-of-Line Storage)

### What is correct

- **Why it exists:** a tuple cannot span two pages, so large values must be handled separately. Postgres also supports big types (text, jsonb, bytea).
- **How it works:**
    1. If a **row** exceeds about **2KB** (a quarter of the 8KB page), Postgres works on the largest toastable columns until the row fits.
    2. It first **compresses** them (pglz by default, LZ4 available from PG14 via `default_toast_compression`).
    3. If that is not enough, it moves the values **out of line** into the table's TOAST table.
- **TOAST table structure:**
    - Named `pg_toast.pg_toast_<oid>`.
    - Columns: `chunk_id`, `chunk_seq`, `chunk_data`.
    - Chunks are about 2KB each.
    - It has its own B-tree index on `(chunk_id, chunk_seq)`.
    - The main row keeps only a small pointer, about 18 bytes.
- **Storage strategies** (`ALTER TABLE ... ALTER COLUMN ... SET STORAGE`):

|Strategy|Meaning|
|---|---|
|`PLAIN`|No compression, never out of line|
|`EXTENDED`|Default: compress, then move out of line|
|`EXTERNAL`|Move out of line, no compression (faster substring on big text/bytea)|
|`MAIN`|Compress, try to keep inline|

- **TOAST tables are vacuumed** by autovacuum like any other table, so deleted large values leave dead chunks there too.
- **TOAST is paid only when the column is read.** Queries that skip the big column never touch the TOAST table. `SELECT *` and ORMs that load every column pay for it.
- **Updates that don't touch the big column** reuse the existing TOAST pointer, so the large value isn't rewritten. Updating the big value itself (even one key inside a JSONB document) rewrites the whole value.
- **Index limit:** a B-tree entry can't exceed about a third of a page (about 2.7KB), so very large values can't be B-tree indexed directly. GIN works for jsonb and tsvector because it indexes the elements inside the value. For long text, index a hash or `md5()` expression instead.

### Mix-ups corrected

> **Threshold is per row, not per column.** It is triggered when the **whole row** exceeds about 2KB. It is tunable per table with `toast_tuple_target`.

> **"Toast costs two I/Os."** It costs **more than two** in the worst case: the main page, the TOAST index page(s), and then one or more chunk pages. A 100KB value is about 50 chunks.

> **Compression.** Compressed out-of-line values are stored compressed. Compression is tried first, but if the row still doesn't fit, the value moves out of line.

### Monitoring

- `pg_total_relation_size` (includes TOAST) vs. `pg_relation_size` (main table only).
- `pg_column_size(col)` shows the stored size of a value.
- `pgstattuple` measures bloat and density.

---

## 3. Side Tables (Vertical Partitioning)

### What is correct

- If a few large, rarely read columns sit next to hot, small columns, move the large ones to a **separate regular table** that shares the **same primary key** (a 1:1 table). This is vertical partitioning.
- Reasons it helps:
    1. **Page density:** narrow rows give more rows per page, so one 8KB read serves more of your query.
    2. **Accidental reads:** `SELECT *` on the hot table cannot touch the large data. You must join on purpose.
    3. **Different lifecycles:** the hot table can have a low `fillfactor`, aggressive autovacuum, and a small footprint. The cold large-data table doesn't need any of that.
    4. **Buffer cache:** pages full of rarely read large values stop competing with hot pages for `shared_buffers`.
    5. **Copy costs:** WAL volume, backup size, and restore time grow with blob size.
- **Blobs in object storage (S3 etc.)** with just a reference in the database keep the DB small. The trade-off is losing transactional consistency, so you handle orphaned objects and dangling references yourself.
- **Not always worth it:** if large values are occasional and moderate, TOAST alone is fine. Measure first.

### Mix-ups corrected

> **"A toast table in a different table."** The side table is a **normal table you create**, not a TOAST table. TOAST still applies inside the side table if a value is large enough, so the two mechanisms stack.

> **"Fillfactor doesn't serve a lot of gap."** The idea is right: fillfactor reserves free space on each page for HOT updates. On a wide table the reserved space is dominated by the large values, so the benefit is diluted. Separating the tables lets you tune the hot table on its own.

---

## 4. Buffer Cache (`shared_buffers`) and Clock Sweep

### What is correct

- `shared_buffers` is a **fixed-size** region of RAM divided into 8KB buffers. Every page read or modified (heap, index, TOAST) must be in it first.
- Each buffer has a **descriptor** with the page identity (relation and block), flags (dirty, valid), `usage_count`, and `refcount`.
- **Eviction uses clock sweep** instead of LRU. LRU needs a shared list reordered on every access, which causes lock contention. With clock sweep, an access is just a counter bump.
- **How the clock hand works:**
    - It moves only when a backend needs a free buffer.
    - For each unpinned buffer: if `usage_count` is 0, evict it. Otherwise decrement it and move on.
    - A page therefore survives several passes if it keeps being accessed.
- **Dirty victims:** if the chosen buffer is dirty, it must be written out first, and WAL must be flushed up to that page's LSN before the page write. A backend that hits this pays a synchronous page write plus a WAL flush, which is slow.
- **Background writer:** writes dirty pages ahead of the clock hand so backends rarely have to write. It writes to the **OS page cache**, not necessarily to disk.
- **Checkpointer:** writes all dirty pages and fsyncs. This **bounds crash recovery time**. Durability at commit comes from the **WAL flush**, not the checkpoint.
- **OS page cache and double caching:** Postgres relies on the OS cache, so `shared_buffers` is kept moderate (rule of thumb about 25% of RAM) and the rest is left for the OS. The same page can sit in both, so double caching is not avoided but is a compromise.

### Mix-ups corrected

> **Two different counters.** `refcount` is the **pin count** (how many backends are using the buffer right now), and pinned buffers are never evicted. `usage_count` is the **popularity**, capped at **5**. A new page enters with 1 and each access bumps it up to the cap.

> **"Decrement happens when the cache is full."** It happens **as the clock hand passes** a buffer, and the hand moves only when a free buffer is needed.

> **"Eviction means flushing."** Eviction picks a victim. Only a **dirty** victim must be written first.

> **"Background writer makes data durable."** It only moves pages into the OS cache. **Checkpoints** bound recovery time, and **WAL flush** gives durability.

> **"20% buffer cache, 80% OS cache."** The usual starting point is about **25%**. Also, `blks_read` is not necessarily real disk I/O, because the OS cache may have served it.

### Monitoring

- `pg_buffercache`: which relations occupy the cache, with usage counts.
- `blks_hit` vs. `blks_read`: hit ratio (with the OS-cache caveat).
- `pg_stat_io` (PG16+): shows who performs the writes. Many backend writes mean bgwriter and checkpoint tuning is behind.
- `pg_prewarm` and autoprewarm: help with cold caches after restart or failover.

---

## 5. Ring Buffers (Protection Against Scan Pollution)

### What is correct

- **Problem:** a big seq scan could read a huge table through the normal pool and evict every hot page. This is cache pollution.
- **Solution:** for some operations, Postgres gives the operation a **small private ring of buffers**. It recycles only those buffers instead of taking buffers from the whole pool.

|Operation|Ring size|
|---|---|
|Seq scan on a table larger than 1/4 of `shared_buffers`|256KB (32 buffers)|
|VACUUM|256KB|
|Bulk writes (COPY, CREATE TABLE AS, etc.)|16MB|

- The ring is **part of** `shared_buffers`, not extra memory. Pages the scan reads still land in shared memory. The scan just reuses the same 32 buffers in a loop.
- Pages of the table that are already cached are read from `shared_buffers` as normal.
- **Index scans get no ring.** An index scan that fetches a large share of a table randomly goes through the normal pool and can evict a lot of hot pages. This is a reason a seq scan can be better for the cache when a query touches a large fraction of a table.

### Mix-ups corrected

> **"The ring doesn't use the buffer cache."** It does. It is a small slice of it.

> **"It triggers when the read is more than a quarter of the cache."** The trigger is that the **table being scanned** is larger than 1/4 of `shared_buffers`. With 4GB, a seq scan on a 1GB+ table uses the ring, but a 500MB table uses the normal pool.

---

## 6. Connecting It All: Wide Rows, TOAST, and Cache

### What is correct

- Rows per page drives scan cost and cache efficiency. A 100-byte row gives about 80 rows per page. A 1.5KB row gives about 5 (roughly 16x more pages for the same row count).
- **The danger zone is rows just under ~2KB.** They never trigger TOAST, so they stay inline and uncompressed, and nothing separates the wide data from the hot columns. Every query pays for the wide pages, **even queries that don't read the big column**.
- **Once rows cross the threshold,** the big values move out of line and the main table becomes dense again. Queries that skip the big column are fast and cache-friendly.
- **Trade-off of TOAST:**
    - Cheap for the common case (queries that don't need the big value).
    - Expensive for queries that do read it: TOAST index lookup, chunk reads, decompression. All those pages also enter `shared_buffers` and compete with hot pages.
- **Effect of wide inline rows:**
    - **Index scans:** each matching row costs a heap page fetch, so N matches read close to N pages.
    - **Seq scans:** more pages read, and the table is more likely to cross the 1/4 threshold and use the ring.
    - **Buffer cache:** each buffer holds fewer rows, so the same cache covers fewer rows and the hit ratio drops.
- **Index size doesn't grow with row width**, because an index stores only the indexed columns. Only the heap fetches get costlier. An index-only scan avoids them entirely (when the visibility map allows).
- **Fixes**, from least to most invasive:
    1. Lower `toast_tuple_target` so Postgres toasts earlier.
    2. Use `SET STORAGE MAIN` or `EXTERNAL` per column, depending on whether you want inline compressed or out-of-line storage.
    3. Move wide columns to a side table (works at any size).
    4. Store large blobs in object storage with a reference.

### Mix-ups corrected

> **"Wide rows increase correlation."** Wrong term. **Correlation** (`pg_stats`) measures how well physical order matches a column's value order, and depends on insert and update patterns, not row width. What wide rows change is the **number of pages** and **rows per page**.

> **"Toast not triggered is worse than toast triggered."** True **for queries that don't read the big column**. For queries that do read it, toasting costs extra I/O. So it is a trade-off, and the real problem zone is rows just under the threshold.

> **"Why a separate table if toast already separates?"** TOAST separates automatically only above ~2KB per row. A side table gives the separation at any size, and makes the expensive read an explicit join instead of an accidental `SELECT *`.

---

## 7. Write-Ahead Log (WAL)

### MUST REMEMBER: Checkpoint and Recovery

> ### Checkpoint
> 
> **redo = current LSN → flush dirty pages (buffer cache) → save redo**
> 
> ### Recovery
> 
> **read saved redo → replay WAL forward to the end → skip if pageLSN is already newer**

Why each step is there:

- **Checkpoint:** record the redo point first (the current LSN). Then write out the pages that were dirty at that point (spread over time using `checkpoint_completion_target`). Finally write the checkpoint record and save the redo pointer in `pg_control`. After this, WAL older than the redo point is no longer needed for crash recovery.
- **Recovery:** start at the saved redo point and replay every record forward to the end of the WAL. For each record, compare the page's LSN with the record's LSN. If the page LSN is already newer, skip it, because that change is already on disk. This check makes replay safe to repeat.

### What is correct

- **The problem WAL solves:** disk is the slowest component, and writing data pages directly means random I/O on every change. WAL gives durability without paying that cost.
- **The idea:** every change is appended to the WAL, which is a sequential write and therefore fast. The actual data page is written to disk later, in the background.
- **Not Postgres-only:** the same idea exists in other databases. InnoDB and Oracle call it the redo log, SQL Server calls it the transaction log.
- **The write path:**
    1. The change is recorded as a WAL record in WAL buffers.
    2. The page in `shared_buffers` is modified and marked dirty, and its **page LSN** is set to that record's LSN.
    3. At **commit**, only the WAL is flushed (fsync) up to the commit record. The data page is **not** written.
    4. Later, bgwriter or the checkpointer writes the dirty page.
- **The core rule:** a dirty page can never be written to disk before WAL up to its LSN has been flushed. This is the same rule from the buffer cache topic.
- **LSN** is a 64-bit, monotonically increasing byte position in the WAL stream. Every WAL record has one, and every page header stores the LSN of its last change.
- **WAL segments** are fixed-size files (16MB by default, set at `initdb`). They are just slices of the single WAL stream.
- **Torn pages and full-page writes (FPI):**
    - The data file page is 8KB, but the OS or disk may write it in smaller pieces. A crash in the middle leaves half-old, half-new content.
    - WAL records are small **deltas** that assume the page is intact, so they can't repair it.
    - So the **first change to each page after a checkpoint** logs a **full page image** into WAL. Recovery restores that image first, then applies the later deltas.
    - `wal_compression` compresses the full-page images.
- **Replication:**
    - **Physical:** ships the WAL stream, and the standby replays it. The result is a block-identical copy. It needs the same major version and architecture, and the standby is read-only.
    - **Logical:** needs `wal_level = logical`, which adds extra information to WAL. It is decoded into **row-level changes** (insert, update, delete), not SQL queries. Updates and deletes need a **replica identity** (usually the primary key). It allows table-level selection and cross-version replication, but DDL is not replicated.
    - Both read the same WAL. Logical is not a separate log.

### Mix-ups corrected

> **"MVCC updates the buffer cache and then WAL records it."** The order is **WAL record first, then the page change, then the page LSN is set**. MVCC is only the version-tracking scheme (`xmin`/`xmax`), and WAL logs those tuple changes like any others.

> **"WAL has no cost."** It is cheap, not free. Commit latency is mostly one WAL fsync. Group commit lets concurrent commits share an fsync. `synchronous_commit = off` trades a small window of lost transactions for lower latency, and it never corrupts the database.

> **"A segment can be removed once the LSN is past it and it is replicated."** A segment is kept until **all** of these are satisfied:
> 
> - it is older than the last checkpoint's redo point
> - it has been archived (if archiving is on)
> - no replication slot still needs it
> - it is outside `wal_keep_size`

> **"Segments group logical things together."** They don't. They are fixed-size slices of one continuous stream.

> **"The checkpoint checks the current LSN, and the dirty pages before it are flushed."** Close. The redo point is taken **first**, then the dirty pages are flushed, then the checkpoint record is written. That is why the memory aid says "save redo" last.

> **"Torn pages are a WAL write problem."** The danger is the **data file write** of an 8KB page. WAL records have their own CRC. Full-page images exist because deltas can't fix a half-written data page.

> **"Compression applies to the deltas."** `wal_compression` compresses the **full-page images**, not the small delta records.

### Why it matters in production

- **Checkpoint frequency drives WAL volume.** After every checkpoint, the first change to each page logs a full-page image again. Frequent checkpoints mean more WAL and more replication lag.
    - Tune `max_wal_size` and `checkpoint_timeout` so checkpoints are mostly time-triggered. Use `log_checkpoints` to see which kind you get.
    - Trade-off: longer intervals mean longer crash recovery.
- **Replication slots can fill the disk.** A slot holds WAL until its consumer catches up. A stalled standby or logical subscriber makes `pg_wal` grow until the disk is full. Monitor slot lag and set `max_slot_wal_keep_size`.
- **Replication lag is measured in LSN.** Compare `pg_current_wal_lsn()` with the replay position in `pg_stat_replication` (including its `*_lag` columns).
- **Synchronous replication** adds a network round trip to every commit. Choose the `synchronous_commit` level on purpose: `on`, `remote_write`, `remote_apply`, or `off`.
- **Backup and point-in-time recovery:** a base backup plus archived WAL lets you restore to any moment. This is the same mechanism as replication, used for backups.
- **Hardware:** low-latency storage for WAL matters more than throughput, because commit latency is fsync latency. Some setups put `pg_wal` on a separate volume.

---

## Quick Reference: Numbers and Facts

|Item|Value|
|---|---|
|Page size|8KB|
|Line pointer|4 bytes|
|Tuple header|about 23 bytes|
|TOAST trigger|row exceeds about 2KB (tunable via `toast_tuple_target`)|
|TOAST chunk size|about 2KB|
|TOAST pointer in main row|about 18 bytes|
|B-tree max entry size|about 2.7KB (a third of a page)|
|`usage_count` cap|5|
|Seq scan / VACUUM ring|256KB (32 buffers)|
|Bulk write ring|16MB|
|Ring trigger|table larger than 1/4 of `shared_buffers`|
|`shared_buffers` rule of thumb|about 25% of RAM|
|WAL segment size|16MB (default, set at `initdb`)|
|LSN|64-bit byte position in the WAL stream|
|Full-page image|logged on the first change to a page after a checkpoint|
|Checkpoint|redo = current LSN, flush dirty pages, save redo|
|Recovery|read saved redo, replay forward, skip if pageLSN is newer|

## Topics to Study Next

- VACUUM internals and the visibility map
- B-tree indexes and how they interact with TIDs
- Replication slots, standby lag, and point-in-time recovery in practice