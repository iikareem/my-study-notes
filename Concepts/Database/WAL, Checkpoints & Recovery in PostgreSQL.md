# WAL, Checkpoints & Recovery in PostgreSQL

## What is LSN?

**Log Sequence Number** — a monotonically increasing integer assigned to every write in the WAL. Physically it is a **byte offset** into the WAL stream.

```
SELECT pg_current_wal_lsn();
-- 0/3A2F108
--  ^ ^
--  | offset within segment
--  segment number
```

WAL is split into **16MB segment files** on disk:

```
pg_wal/
  000000010000000000000001   ← 16MB
  000000010000000000000002   ← 16MB
  ...
```

So LSN `0/1001050` means: segment 1, byte offset 1050 inside it.

---

## LSN Lives in Two Places

### 1. In the WAL Segment

Every WAL record sits at its own LSN — the offset is the identity of the record.

```
WAL segment 000000010000000000000001:
  offset 1000 → INSERT record  (LSN 0/1001000)
  offset 1050 → UPDATE record  (LSN 0/1001050)
  offset 1100 → DELETE record  (LSN 0/1001100)
```

### 2. In Every Data Page on Disk

Every 8KB heap/index page has `pd_lsn` in its header — the LSN of the **last WAL record that modified that page**.

```
page 42 on disk:
┌─────────────────────────────┐
│ pd_lsn = 0/1001050          │  ← stamped when page was flushed
│ ...tuple data...            │
└─────────────────────────────┘
```

LSN is the **bridge** between the WAL stream and the data pages. The page is saying: _"I am consistent up to this point in the WAL."_

---

## What a Checkpoint Does

A checkpoint does two things:

1. **Flushes all dirty pages** from the buffer pool to disk (with their current `pd_lsn` stamped on them)
2. **Writes a checkpoint WAL record** containing the `redo_lsn` — the LSN from which recovery must start

```
Buffer pool before checkpoint:
  page 42  dirty  pd_lsn = 0/1000800
  page 99  dirty  pd_lsn = 0/1000950

After checkpoint:
  page 42  on disk  pd_lsn = 0/1000800  ✓
  page 99  on disk  pd_lsn = 0/1000950  ✓

WAL:
  ...
  LSN 0/1001000 → CHECKPOINT { redo_lsn: 0/1001000 }
```

You can inspect this:

```bash
pg_controldata $PGDATA
# Latest checkpoint's REDO location: 0/3A2F000
```

---

## Recovery After a Crash

### Starting Point

PostgreSQL reads `pg_control` to find the last checkpoint, then reads the checkpoint WAL record to get `redo_lsn`. WAL replay starts from there — not from the beginning.

### For Every WAL Record, Compare LSNs

```
WAL record LSN  vs.  pd_lsn on the disk page
```

|Case|Meaning|Action|
|---|---|---|
|`WAL LSN == pd_lsn`|Page already has this change|**Skip**|
|`WAL LSN > pd_lsn`|Page is stale, change never reached disk|**Replay**|
|`WAL LSN < pd_lsn`|Impossible|—|

### Why the Third Case Is Impossible

The WAL-before-page rule guarantees a page can only be flushed to disk **after** its WAL record is durable. So `pd_lsn` on disk can never be ahead of what the WAL has recorded.

---

## Full Recovery Flow

```
1. Read pg_control
        ↓
2. Find last checkpoint → get redo_lsn
        ↓
3. Scan WAL from redo_lsn
        ↓
4. For each WAL record:
     read target page from disk
     compare WAL record LSN vs pd_lsn
          ↓               ↓
        equal            greater
        SKIP             REPLAY
        ↓
5. Undo uncommitted transactions
        ↓
6. Database is consistent
```

---

## Two Mechanisms, Two Jobs

|Mechanism|Job|
|---|---|
|Checkpoint `redo_lsn`|**How far back** to start scanning WAL|
|Per-page `pd_lsn`|**Which records** to apply within that scan|

The checkpoint narrows the window. The `pd_lsn` gives page-level precision inside that window.