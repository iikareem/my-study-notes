
A reference note on how line pointers, B+Tree indexes, HOT updates, and cleanup (vacuum vs reindex) fit together — using PostgreSQL as the reference implementation.

---

## 1. Heap Page Layout & Line Pointers

A heap page looks like this:

```
+----------------+
| Page Header    |
+----------------+
| Line Pointers  |  <- array of (offset, length) pairs
| [1][2][3]...   |
+----------------+
|     ...free... |
+----------------+
| Tuple 3        |
| Tuple 2        |
| Tuple 1        |
+----------------+
```

A **line pointer** (`ItemId` in Postgres) is a small slot that says "the actual tuple data lives at this offset on this page." A **TID** (tuple identifier) = `(page number, line pointer number)`.

This indirection is what allows tuple data to move or be superseded without every reference needing to change — you repoint the line pointer, or add a new one.

---

## 2. MVCC: Insert vs Update

- **INSERT** → write a brand-new tuple, add a new line pointer for it.
- **UPDATE** → never overwrite in place. Instead:
    1. The old tuple is marked "expired" (`xmax` set to the updating transaction's ID).
    2. A new tuple version is written (same page or elsewhere).
    3. The old tuple's `t_ctid` field is set to point to the TID of the new version.

This creates a **version chain**:

```
TID (0,1) --ctid--> TID (0,2) --ctid--> TID (0,3)  [current]
 xmin=100             xmin=105              xmin=110
 xmax=105             xmax=110              xmax=∞
```

Each old version physically exists until vacuum removes it. Insert and update are structurally similar — both create a new physical tuple; update just also marks a predecessor as dead and links to it.

---

## 3. Indexes and Row Versions

**Key fact:** a B+Tree index entry doesn't point to "the row" — it points to one specific physical TID (one specific version).

### Case A — Non-HOT Update

If the new version can't stay on the same page (page full, or an indexed column changed):

- A **new entry is added to every index** on the table, pointing to the new TID.
- Old index entries are left dangling (pointing to now-dead versions) until vacuum removes them.

Result: the B+Tree leaf for a given key can hold **multiple TIDs**, one per version:

```
B+Tree leaf node (key = 42)
 ├── TID (0,1)   <- dead version
 ├── TID (0,2)   <- dead version
 └── TID (0,3)   <- live version
```

**How a lookup resolves this:**

1. Fetch _all_ TIDs matching the key from the leaf.
2. Go to the heap for each TID.
3. Compare that tuple's `xmin`/`xmax` against the transaction's **snapshot** (which transactions were in-progress/committed/aborted at snapshot time — not just "current transaction ID").
4. Keep the tuple where `xmin` is visible and `xmax` is either unset or belongs to a transaction not visible to the snapshot.

This per-TID visibility check is a real cost, sometimes called **index scan visibility checking**.

### Case B — HOT Update (Heap-Only Tuple)

Postgres's optimization to avoid index bloat. Applies when:

- the update doesn't touch any indexed column, **and**
- there's room for the new version on the same page.

In this case, **no new index entries are created**. The new tuple is linked via `t_ctid` from the old one, and the index still points only to the original TID (head of the chain):

```
Index entry (key=42) ---> TID (0,1) [dead]
                              |
                          t_ctid
                              v
                          TID (0,2) [dead]
                              |
                          t_ctid
                              v
                          TID (0,3) [live]
```

A lookup lands on TID (0,1), sees it's dead but part of a **HOT chain**, and walks `t_ctid` forward on the same page until it finds the version visible to its snapshot.

### Comparison

||HOT update|Non-HOT update|
|---|---|---|
|Index entries per row version|1 (points to head of chain)|1 per version|
|How old versions are found|Walk `t_ctid` chain on the heap page|Multiple leaf entries fetched directly from index|
|Visibility check|Done while walking the chain|Done per-TID after fetching from heap|

Both ultimately do the same `xmin`/`xmax`-vs-snapshot comparison — they just arrive at the candidate tuples differently.

---

## 4. Vacuum vs Reindex

### VACUUM — routine, incremental cleanup

Usually runs automatically via **autovacuum**. Does three relevant things:

1. **Heap cleanup** — finds dead tuples (committed `xmax`, older than the oldest still-running transaction, so no one could still need them) and marks their space reusable.
2. **Index cleanup** — for every dead tuple removed from the heap, removes the **corresponding index entries** pointing to that dead TID. This is the direct answer to "when do stale B+Tree leaf entries get removed?" — vacuum does it, entry by entry.
3. **HOT-chain pruning** — compacts dead links in a HOT chain on a page; the index-referenced line pointer may be updated/marked "redirect" so future lookups don't walk through dead links.

Vacuum is **incremental and surgical**: it removes exactly the dead entries without rebuilding the index structure.

### REINDEX — heavy, all-or-nothing rebuild

A completely different operation:

- **Discards the entire index** and **builds a new one from scratch** by scanning the whole table.
- Doesn't track "stale vs live" incrementally — it simply builds fresh from what currently exists in the heap, so dead tuples are naturally excluded.
- Used when:
    - The index is **bloated** (e.g., years of random deletes have left B+Tree pages sparse/unbalanced, and vacuum alone can't reclaim the structure efficiently).
    - The index is **corrupted** (rare — after a crash, bug, or manual data surgery).
    - A **collation change** requires re-sorting entries.
- Expensive: typically locks the table (unless using `REINDEX CONCURRENTLY`); it's all-or-nothing, not a targeted cleanup.

### Summary

> **Vacuum removes dead index/heap entries continuously as routine maintenance. Reindex is a rare, heavy "start over" operation for when the index structure itself has degraded — it is not part of normal MVCC garbage collection.**

If autovacuum is healthy and keeping up, reindexing for the reasons discussed above (stale TIDs, dead versions) is rarely necessary — vacuum handles it before bloat becomes a structural problem.