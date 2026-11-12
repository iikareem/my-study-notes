## Database Internals: Write Path & B-Tree Index

---

## 1. The Write Path

### Step 1 — Buffer Cache First

- Before any modification, the page **must be in buffer cache**
- If not present → engine performs a **full read from disk first**
- Even updates and deletes require the page in memory before touching it

---

### Step 2 — MVCC (Multi-Version Concurrency Control)

- Every write (INSERT, UPDATE, DELETE) creates a **new tuple version**
- Old version is **left intact** for concurrent readers and rollback
- **No in-place modification** ever happens
- Allows reads and writes to not block each other

---

### Step 3 — Modify Buffer Cache Page

- New tuple version is written into the **buffer cache page**
- Page is now **dirty** — modified in memory but not yet flushed to disk

---

### Step 4 — WAL (Write-Ahead Log)

- Instead of immediately flushing dirty pages (expensive random I/O), engine writes to **WAL first**
- WAL is **append-only sequential log** → much faster than random writes
- This is the core durability trick:

> _"Log the change first, flush the page later"_

- Once WAL is **fsynced to disk** → transaction is durable

---

### Step 5 — Commit

- After WAL flush → engine returns **"transaction committed"** to client
- Dirty buffer page is written to disk **eventually**, not immediately

---

### Write Path Summary

```
Client Write Request
        ↓
Read page into Buffer Cache (if not already there)
        ↓
Create new MVCC tuple version (no in-place modification)
        ↓
Modify Buffer Cache page (page is now "dirty")
        ↓
Write WAL record (Full Page Write if first change after checkpoint, else delta)
        ↓
Flush WAL to disk  ←── transaction is durable here
        ↓
Return "COMMIT" to client
        ↓
(Eventually) Checkpoint flushes dirty pages → updates page header LSN
```

---

## 2. WAL Internal Structure

|Concept|Detail|
|---|---|
|**WAL Segments**|Log divided into fixed-size segment files|
|**LSN**|Every WAL record has a monotonically increasing **Log Sequence Number**|
|**Full Page Write**|First modification to a page after a checkpoint → **entire page image** written to WAL|
|**Delta**|All subsequent modifications → only **changes** are logged|
|**Index changes**|B-tree splits, new entries — all fully WAL-logged|

---

## 3. Checkpoints

- Runs **periodically** to flush dirty buffer pages to disk
- Each flushed page header is updated with the **LSN** of the latest WAL record that affected it
- A checkpoint record is written to WAL marking the **checkpoint LSN**

### Why LSN in Page Header Matters — Crash Recovery

- After a crash, engine reads each page's **LSN from its header**
- Compares it against the WAL
- **Replays** any WAL records with a higher LSN than the page header
- This tells it exactly: _"replay everything after this point"_

---

## 4. B-Tree Index Structure

- Self-balancing tree with **root**, **internal nodes**, and **leaf pages**
- All actual values (heap pointers/TIDs) live **only in leaf pages**
- Internal nodes hold **separator keys** to route searches down the tree

---

## 5. Sequential vs Random Inserts

|Insert Pattern|Behavior|
|---|---|
|**Sequential / monotonic** (timestamps, serial IDs)|Go to **rightmost leaf** — very efficient, minimal splits|
|**Random** (UUIDs, random strings)|Can land anywhere — more **page splits**, scattered I/O|

> UUIDs are painful for B-tree indexes at scale — they destroy the rightmost insertion optimization

---

## 6. What Happens When a Leaf Page is Full

PostgreSQL tries these steps **in order** (lazy before aggressive):

1. **Vacuum dead tuples** — reclaim space from dead/aborted MVCC versions first
2. **Deduplication** — if many entries share the same key, compress multiple TID pointers under one key entry
3. **Page split** — if neither works, split the leaf page into two

---

## 7. B-Tree Insert: Existing Key vs New Key

### Existing Key (Separator Already in Internal Nodes)

- Key range already known to the tree
- Insert goes directly to the correct leaf page
- If leaf is full → split **leaf pages only**
- Internal nodes **untouched**
- Tree depth **unchanged**

---

### New Key (No Separator in Internal Nodes)

- Key range is unknown to the tree
- Leaf split **plus** new separator must be added to parent
- Can **cascade upward** if parent is also full
- Worst case → root splits → tree **depth increases**
- Touches many pages → **heavy locking**, deadlock risk

---

### Quick Comparison

||Existing Key|New Key|
|---|---|---|
|Leaf split|✅ (if full)|✅|
|Internal node change|❌|✅|
|Cascade possible|❌|✅|
|Depth increase|❌|✅ (worst case)|
|Lock contention|Low|High|