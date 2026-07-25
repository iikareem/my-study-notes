---
tags:
  - postgresql
  - indexes
  - btree
---

# Monotonically Increasing vs Random Index Writes

Here's a breakdown of the two behaviors and what's optimal for each:

---

## Sequential (monotonically increasing) inserts

When you always insert into the **rightmost leaf** — like an auto-increment ID or `created_at` — the write pattern looks like:

- Every insert goes to the **same page** until it's full, then a new page is created on the right.
- **No page splits in the middle** of the tree — splits only happen at the current right edge.
- The **hot page** is always predictable: Postgres can keep it in `shared_buffers` and flush it sequentially.
- WAL writes are also sequential → very I/O friendly.
- The index grows in one direction: **right-appended**, so old pages are cold and rarely touched.

**This is the best possible write pattern for a B-tree.** It's sometimes called _monotonic insert_ or _append-only insert_. You get:

- Minimal write amplification
- High fill factor (pages stay near 100% full)
- No wasted space from splits in arbitrary positions
- Sequential I/O on both the index and WAL

One tuning note: the default `fillfactor` for a B-tree index is **90** (leaves 10% free for in-place updates). For a truly append-only column like `created_at` where rows are never updated in-place, you can safely set `fillfactor=100` and squeeze out that last 10%.

```sql
CREATE INDEX idx_created ON orders (created_at) WITH (fillfactor = 100);
```

---

## Random inserts

When the key is random — a UUID v4, a hash, random strings — every insert lands on a **random leaf page**, which means:

- Postgres has to fetch a different page from disk almost every time (cache miss).
- Pages fill unevenly; **page splits** happen throughout the tree, not just at the edge.
- A split causes a new page to be allocated mid-tree and the parent to be updated — more I/O, more WAL.
- The **working set of hot pages** is the entire leaf level, so `shared_buffers` pressure is high.
- At high write rates, you'll see **lock contention** on random leaf pages.

**This is the worst case for B-tree write performance.** The consequences scale with index size — small tables are fine, large ones degrade noticeably.

The common mitigation is **UUIDv7** (or `ulid`): they embed a timestamp prefix so inserts are roughly sequential, giving you UUID uniqueness but sequential write behavior. Postgres 17 added `gen_random_uuid()` support for UUIDv7 natively.

```sql
-- Postgres 17+: UUIDv7 is sequential → behaves like monotonic insert
INSERT INTO orders (id) VALUES (gen_random_uuid());  -- v4 (random, bad)
-- use a UUIDv7 generator instead for sequential behavior
```

---

## Summary

||Sequential (`created_at`, serial)|Random (UUIDv4, hash)|
|---|---|---|
|**Page access pattern**|Always rightmost leaf|Random leaf each time|
|**Cache efficiency**|Excellent — hot page is predictable|Poor — entire leaf level is "hot"|
|**Page splits**|Only at right edge|Throughout the tree|
|**WAL pattern**|Sequential|Random, scattered|
|**Write throughput**|High|Degrades at scale|
|**Fillfactor tip**|Can use 100|Keep default 90 or lower|
|**Mitigation if random**|Switch to UUIDv7 / ULID|—|

	For `created_at`-style workloads you're already in the optimal case — just confirm `fillfactor=100` if you never update that column.

---

## Related

- [[Btree vs LSM]]
- [[BRIN]]
- [[HOT]]

← [[Concepts MOC]]
