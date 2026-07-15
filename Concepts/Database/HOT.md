---
tags:
  - database
  - postgresql
---

You’ve nailed the core mechanics perfectly. The breakdown you just gave is an excellent summary of **HOT (Heap-Only Tuples)**, which is easily one of Postgres’ most brilliant architectural optimizations.

Let's dive a bit deeper into _how_ this works under the hood, why that "same page" constraint exists, and what that P.S. about summarizing indexes really means.

## The Core Problem: Write Amplification

In standard MVCC without HOT, if you have a table with 5 indexes and you update a single _non-indexed_ column (like changing a user's `bio`), Postgres is forced to insert a brand new row version (tuple) somewhere in the heap.

Because this new tuple lives at a new physical address (a new CTID), **all 5 indexes must get a new entry pointing to this new address**, even though the indexed values didn't change at all! This is called **write amplification**, and it causes massive index bloat and heavy I/O.

## How HOT Solves This (The Mechanics)

When you update a row and _no_ indexed columns change, Postgres triggers the HOT optimization. It requires two strict conditions:

1. **No indexed columns are modified.**
    
2. **There is enough free space on the exact same data page (heap page) to hold the new tuple version.**
    

Here is exactly how that chain you mentioned is structured and navigated:

### 1. The Line Pointer Cheat Code

Every page in Postgres has a line pointer array (PageHeaderData) at the front, which points to the actual tuples at the bottom of the page.

- **Before HOT:** The Index points directly to `Tuple v1`.
    
- **After HOT:** `Tuple v1` is marked with a special flag (`HEAP_ONLY_TUPLE`), and a pointer is created linking `v1` to `v2` on the same page.
    

### 2. The Read Path (Traversing the Chain)

When a SELECT query uses the index to find `id=42`, the index points to `Tuple v1`. Postgres lands on the page, sees `v1`, and notices the HOT chain flag. It seamlessly follows the chain link to `Tuple v2` (and `v3`), checks the transaction visibility, and returns the correct version.

### 3. The Secret Weapon: Pruning (No VACUUM Needed)

This is the part of your note that deserves extra emphasis: **"clean up intermediate versions during normal page access."**

Because the entire chain lives on a single page, Postgres doesn't need to wait for the heavy, asynchronous `VACUUM`process to clean up dead rows. The next time a `SELECT`, `UPDATE`, or `INSERT` lands on that specific page, Postgres executes an **in-page pruning**.

It realizes `Tuple v1` and `Tuple v2` are dead to all current transactions. It physically removes them, frees up the space, and updates the root line pointer to point _directly_ to `Tuple v3`.

> **Result:** The index still points to the exact same line pointer, but the intermediate dead weight is gone instantly.

## The Catch: Why the "Same Page" Constraint Matters?

If Postgres runs out of space on that specific 8KB page and has to put `Tuple v2` on a _different_ page, **HOT breaks**.

Postgres cannot create a HOT chain across pages because line pointers cannot bridge different pages. If it spans pages, Postgres has to fallback to the old way: creating a new index entry. This is why tuning your table's `FILLFACTOR` (e.g., setting it to 80 or 90 instead of 100) is crucial for write-heavy tables—it leaves "breathing room" on the page for HOT updates to happen.

## Explaining the P.S.: Summarizing Indexes

> _"summarizing indexes (indexes that don't store a pointer to every row) don't prevent HOT."_

This is a great nuance. A "summarizing index" usually refers to a **BRIN (Block Range Index)**.

Unlike a B-Tree index (which has a dedicated entry pointing to the exact physical slot of _every single row_), a BRIN index just looks at a range of pages (usually 1M blocks) and says, _"In this entire block range, the minimum `created_at` date is X and the maximum is Y."_

Because a BRIN index doesn't map exact individual tuple coordinates (CTIDs), updating a row on a page doesn't break a specific index pointer. Therefore, **having a BRIN index on a column does not block HOT updates on that column**, as long as the new value still falls within the existing minimum/maximum range of that page block.

## Summary of the Tradeoff

|**Standard MVCC Update**|**HOT Optimization Update**|
|---|---|
|Modifies indexed or non-indexed columns.|Modifies **only** non-indexed columns.|
|Writes new tuple to _any_ page with space.|Must write new tuple to the **same** page.|
|**Creates new entries in all indexes.**|**Zero new index entries created.**|
|Requires `VACUUM` to reclaim space.|Reclaims space dynamically via **in-page pruning**.|