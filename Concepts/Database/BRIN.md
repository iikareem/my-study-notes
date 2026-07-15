---
tags:
  - database
  - postgresql
---

Here is your complete guide to **BRIN (Block Range Index)**.

Think of this as your architectural blueprint for when to use it, how it works under the hood, and—crucially—when it will completely break your query performance.

## 1. What is a BRIN Index?

Introduced in PostgreSQL 9.5, **BRIN** stands for **Block Range Index**.

Unlike a traditional B-Tree index that creates a direct pointer for _every single row_ in a table, BRIN takes a birds-eye view. It divides a table into physical chunks on the disk (called **block ranges**) and only records the **minimum** and **maximum**value of the indexed column for each chunk.

### The Real-World Analogy

Imagine a warehouse full of shipping boxes stacked in order by date.

- **B-Tree:** A massive spreadsheet tracking the _exact box and shelf_ for every single item in the warehouse.
    
- **BRIN:** A small piece of paper taped to the outside of each pallet that says: _"This stack contains items from Jan 1st to Jan 15th."_
    

## 2. How BRIN Works Under the Hood

To understand BRIN, you have to understand how database storage works. Postgres stores tables in **Pages** (also called **Blocks**), which are usually 8KB chunks of disk space.

1. **Pages Per Range (`pages_per_range`):** By default, BRIN groups **128 pages** together into a single "Block Range" (roughly 1MB of data).
    
2. **The Summary:** Postgres scans those 128 pages, finds the lowest (Min) and highest (Max) value in the indexed column, and writes _only_ those two values into the BRIN index.
    
3. **The Map:** The index becomes a tiny map of block boundaries:
    
    - Blocks 0–127: Min = `101`, Max = `500`
        
    - Blocks 128–255: Min = `501`, Max = `1000`
        

### How a Query Executes (Bitmap Index Scan)

When you run `SELECT * FROM table WHERE id = 750;`:

1. Postgres checks the BRIN index.
    
2. It sees `750` falls between `501` and `1000` (Blocks 128–255).
    
3. It throws away all other blocks and performs a **Bitmap Heap Scan** _only_ on blocks 128 through 255 to find the exact row.
    

## 3. B-Tree vs. BRIN: The Ultimate Comparison

| **Feature**               | **B-Tree Index**                                        | **BRIN Index**                                        |
| ------------------------- | ------------------------------------------------------- | ----------------------------------------------------- |
| **Size**                  | **Large** (often 20–30% of the table size).             | **Tiny** (often 1% or less of the table size).        |
| **Lookup Speed**          | **Ultra-Fast** (Directly hits the exact row).           | **Fast** (Must scan a small range of blocks).         |
| **Data Order Dependency** | None. Works on completely random data.                  | **Strict.** Data _must_ be physically sorted on disk. |
| **Write Performance**     | Slows down `INSERT`/`UPDATE` (Index updates instantly). | Very low impact (Index updates are deferred/lazy).    |
| **Index-Only Scans**      | **Yes.** Can return data without touching the table.    | **No.** Must always read the table blocks.            |

## 4. The Critical Requirement: Physical Correlation

BRIN is entirely dependent on **Physical Correlation** (how well the logical order of your data matches its physical order on the hard drive).

You can check this correlation in Postgres using the `pg_stats` view:

SQL

```
SELECT correlation 
FROM pg_stats 
WHERE tablename = 'your_table' AND attname = 'your_column';
```

- A value close to **1.0** means the data is perfectly sorted on disk (e.g., auto-incrementing IDs, created_at timestamps). **Perfect for BRIN.**
    
- A value close to **0.0** means the data is randomly scattered (e.g., UUIDs, text names). **BRIN will fail completely here.**
    

## 5. Maintenance and the "Lazy" Architecture

One of BRIN’s greatest strengths is that it doesn't slow down write operations.

When you insert a new row into a B-Tree index, Postgres immediately pauses to insert it into the correct spot in the tree.

With BRIN, when you append new rows to the end of a table, Postgres **does not immediately update the index**. It waits until the current block range (all 128 pages) is completely full.

- **The "Summarized" State:** Ranges that are closed and summarized.
    
- **The "Unsummarized" State:** The newest pages at the end of the table. Postgres will just do a regular sequential scan on these unsummarized pages until `VACUUM` or an explicit command closes them.
    

To manually force BRIN to summarize new data, you run:

SQL

```
SELECT brin_summarize_new_values('my_brin_index_name');
```

## 6. How to Create and Tune a BRIN Index

### Basic Syntax

SQL

```
CREATE INDEX idx_orders_date_brin 
ON orders 
USING brin (order_date);
```

### Tuning the Page Range

You can change how many pages are grouped into a single range using the `pages_per_range` parameter. It must be a power of 2 (e.g., 32, 64, 128, 256).

SQL

```
CREATE INDEX idx_orders_date_brin_custom 
ON orders 
USING brin (order_date) 
WITH (pages_per_range = 32);
```

- **Smaller `pages_per_range` (e.g., 32):** The index gets slightly larger, but queries become more precise (Postgres has to scan fewer table pages per match).
    
- **Larger `pages_per_range` (e.g., 512):** The index becomes impossibly small, but queries have to scan larger chunks of the table.
    

## Summary Checklist: When to use BRIN?

Use BRIN if you can check **all three** of these boxes:

1. [ ] The table is **massive** (tens/hundreds of millions of rows) where a B-Tree index would consume tens of gigabytes of RAM/disk.
    
2. [ ] The column data is **naturally sorted** as it gets inserted (like `created_at`, `log_date`, or a `bigserial` sequence).
    
3. [ ] Your queries are typically **analytical or range-bound** (e.g., `WHERE log_date BETWEEN '2026-01-01' AND '2026-01-07'`), rather than looking for a single needle in a haystack millions of times per second.

CRUD
When you start performing **INSERTs**, **UPDATEs**, and **DELETEs** (often called Write operations or DML), BRIN indexes behave very differently from traditional B-Tree indexes.

Because BRIN relies on the data being neatly ordered on disk, modifying data can either be highly efficient or completely ruin the index's effectiveness.

Here is exactly what happens during writes:

## 1. INSERT Operations (How BRIN Stays Fast)

When you `INSERT` data, BRIN handles it using a "lazy" approach to ensure your write performance doesn't slow down.

- **Appending Data (Best Case):** If you are inserting sequential data (like an auto-incrementing ID or a current timestamp), Postgres writes the new rows to the very end of the table.
    
- **The "Unsummarized" Zone:** BRIN does _not_ immediately update the index for every row you insert. Instead, Postgres keeps track of the new pages in an **unsummarized** state.
    
- **The Auto-Summarize Trigger:** Once a full block range is completed (e.g., another 128 pages are filled), Postgres will automatically summarize that entire chunk into a single Min/Max entry for the index.
    

> **Note:** If you want Postgres to immediately clean up and summarize the end of the table without waiting, you can run:
> 
> `SELECT brin_summarize_new_values('your_index_name');`

## 2. UPDATE Operations (The "Range Widener")

`UPDATE` is where BRIN indexes can get into serious trouble because of how PostgreSQL handles updates under the hood.

Postgres uses **MVCC (Multi-Version Concurrency Control)**. When you update a row, Postgres doesn't change the data in place; it actually marks the old row as "deleted" and **inserts a brand-new version of the row** into a completely different page (usually at the end of the table).

### The Danger of "Range Widening"

Imagine you have an `orders` table ordered by `order_date`.

- **Block Range 1 (Pages 0-127):** Contains early January data. Min: `2026-01-01`, Max: `2026-01-15`.
    
- **Block Range 50 (Pages 6272+):** Contains June data.
    

If you update an order from **January 2nd** to change its shipping status, Postgres moves that updated row to **Block Range 50** (the end of the table).

Now look at what happens to Block Range 50's BRIN entry:

- **Before Update:** Min: `2026-06-01`, Max: `2026-06-05`
    
- **After Update:** Min: **`2026-01-02`**, Max: `2026-06-05`
    

Because a single January date was moved into the June block range, **the Min/Max boundary for Range 50 has blown wide open.** Now, any query looking for January data will _also_ have to scan through all the June data pages. If you do this too many times, your boundaries overlap completely, and the BRIN index becomes useless.

## 3. DELETE Operations (The "Ghost" Data Problem)

When you `DELETE` a row, Postgres simply marks it as invisible. It does _not_ automatically shrink the BRIN index boundaries.

### The "Empty Space" Issue

- Imagine a block range has IDs from `1000` to `2000`.
    
- You delete row `1000` (the minimum) and row `2000` (the maximum).
    
- Even though those rows are gone, **the BRIN index will still say the Min/Max is `1000` to `2000`** because it doesn't instantly recalculate boundaries on delete.
    

### How it gets fixed

The boundaries will only be tightened and fixed when a **`VACUUM`** operation runs on the table. `VACUUM` scans the pages, clears out the dead rows, and prompts the BRIN index to re-summarize that range with the new, actual surviving Min and Max values.

## Summary Checklist for Writes

|**Operation**|**Impact on BRIN**|**Best Practice**|
|---|---|---|
|**INSERT (Sequential)**|High performance, practically zero overhead.|Turn on `autosummarize = on` when creating the index so Postgres handles it automatically.|
|**INSERT (Random)**|Ruined boundaries (widens Min/Max ranges instantly).|**Avoid BRIN** if you regularly insert scattered/random data.|
|**UPDATE**|Can cause "Range Widening" by copying old data to the end of the table.|Avoid updating the column that the BRIN index is built on.|
|**DELETE**|Leaves "ghost" boundaries behind until maintenance runs.|Ensure you have an aggressive `autovacuum` schedule to keep boundaries tight.|