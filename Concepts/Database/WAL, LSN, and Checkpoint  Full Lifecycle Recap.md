# WAL, LSN, and Checkpoint — Full Lifecycle Recap

## 1. On every write/update

- The global, ever-increasing **LSN counter** is advanced → this new LSN is the record's **address** in the WAL.
- If this is the **first modification to this page since the last checkpoint** → WAL gets a **full page image** (protects against torn pages).
- Otherwise → WAL gets just the **delta** (what changed).
- The page's own header **in the buffer cache** is stamped with this LSN (`pageLSN`).
- So: every update → gets its own LSN → written to WAL → page header updated in memory.

## 2. On commit

- The commit itself is a WAL record too, and it must be **fsync'd to disk** before the transaction is considered durable.

## 3. On checkpoint

- Checkpoint **does not compute or look up any LSN** — it just:
    1. Peeks at the current global LSN counter → saves it as `redoLSN` (a simple read).
    2. Scans the buffer cache for **dirty pages**.
    3. Flushes each dirty page to disk **as-is** — whatever `pageLSN` is already in its header (from step 1 of the update flow) goes with it, untouched.
    4. Once flushing is done, writes **one WAL record**: "checkpoint complete, redoLSN = X."
- This announcement is what lets WAL segments **before** that point be archived/recycled — they're no longer needed for recovery.

## 4. On crash recovery

1. Find the **last checkpoint record** in WAL → read its `redoLSN`.
2. Start replaying WAL **from `redoLSN` forward** — everything before it is already guaranteed flushed, no need to look further back.
3. For each WAL record from that point on:
    - Compare the record's LSN against that **page's own on-disk `pageLSN`**.
    - `pageLSN >= record LSN` → already applied → **skip**.
    - `pageLSN < record LSN` → missing → **apply it**, update the page's `pageLSN` to match.
4. Recovery only ever moves **forward** through WAL from `redoLSN` — nothing before it is touched, because "everything before it is done" (already flushed).

## One-line version

> Every update gets a new LSN and stamps it into the page header in memory → WAL logs it (full page or delta) → checkpoint flushes dirty pages as-is and marks its own starting LSN → on crash, recovery replays only from that mark forward, using each page's own LSN to know exactly what's missing.