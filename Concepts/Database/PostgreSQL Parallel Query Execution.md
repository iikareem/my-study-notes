---
tags:
  - database
  - postgresql
---

### 🌟 **What is Parallel Query Execution?**

Parallel query execution allows PostgreSQL to **split a query into smaller tasks** that can run **simultaneously using multiple CPU cores**. This can **significantly improve performance**, especially for large tables, aggregations, or analytical queries.

### **Why Use Parallelism?**

- ✅ Faster query performance on large datasets
    
- ✅ Better use of multi-core CPUs
    
- ✅ Reduced response time for expensive operations (e.g., large scans, joins)

### `max_parallel_workers`

- **Scope**: System-wide (global)
    
- **Meaning**: Maximum number of parallel worker processes that PostgreSQL can run **at the same time**.
    
- **Benefit**: Controls overall concurrency to avoid CPU overload.

### `max_parallel_workers_per_gather`

- **Scope**: Per query (per Gather node)
    
- **Meaning**: Limits how many workers a **single query** can use for parallel operations.
    
- **Rule**: Should usually be ≤ `max_parallel_workers`.
    
- **Benefit**: Prevents one query from consuming all resources.

### `parallel_setup_cost`

- **Type**: Planner cost
    
- **Meaning**: Estimated one-time cost of starting parallel workers.
    
- **Default**: `1000`
    
- **Benefit**: Prevents PostgreSQL from using parallelism on cheap/simple queries.
    
- **Tuning Tip**: Lower this value to make PostgreSQL more eager to use parallelism.

### `parallel_tuple_cost`

- **Type**: Planner cost
    
- **Meaning**: Estimated cost for **each row** sent from a worker back to the main process.
    
- **Default**: `0.1`
    
- **Benefit**: Models the overhead of collecting results from workers.
    
- **Tuning Tip**: Lower this value if your hardware/network handles high-throughput efficiently.

## **How It All Works Together**

When PostgreSQL plans a query, it evaluates:

1. Is the data big enough to benefit from parallelism?
    
2. Are enough parallel workers available (`max_parallel_workers`)?
    
3. Can this query use workers (`max_parallel_workers_per_gather`)?
    
4. Is it worth the cost (`parallel_setup_cost` and `parallel_tuple_cost`)?

| Goal                         | What to Tune                                                      |
| ---------------------------- | ----------------------------------------------------------------- |
| Use more parallelism         | Lower `parallel_setup_cost` and `parallel_tuple_cost`             |
| Conserve system resources    | Lower `max_parallel_workers` or `max_parallel_workers_per_gather` |
| Balance performance vs. load | Adjust all four based on testing and system usage                 |

- `max_parallel_workers`: total parallel processes across the system
    
- `max_parallel_workers_per_gather`: max workers per query
    
- `parallel_setup_cost`: cost to start parallelism
    
- `parallel_tuple_cost`: cost per row sent from worker to main process 

### What is `parallel_tuple_cost`?

**`parallel_tuple_cost`** is a **cost estimate** that tells PostgreSQL:

> **“How expensive is it to pass a row (a tuple) from a parallel worker to the main process?”**

Every row a worker processes must be **sent back** to the main query process (the "Gather" node). This setting controls how much that "communication" is estimated to cost.

---

### 🧩 Why It Matters

When PostgreSQL is planning a query, it evaluates:

- How much time the **parallel workers** will spend
    
- How much overhead comes from **gathering results**
    

If the `parallel_tuple_cost` is **high**, PostgreSQL may decide:

> “Sending rows back to the main process is too costly. Maybe it's better to just run serially.”

## Summary for Notes (Additional Concepts)

> Additional important factors for PostgreSQL parallelism:
> 
> - `parallel_leader_participation`: enables the leader to join the work
>     
> - Only **parallel-safe** queries are eligible
>     
> - Indexes can be used in parallel if they’re large and relevant
>     
> - Table size and statistics affect planner decisions
>     
> - `work_mem` influences per-worker memory efficiency
>     
> - Use `EXPLAIN` to confirm what type of scan PostgreSQL is using

> `ANALYZE` in PostgreSQL updates internal statistics about table contents. These statistics help the query planner make smart decisions — including whether to use indexes or parallel workers. Running `ANALYZE` manually is important after bulk inserts or major updates. It improves query speed and accuracy of plans.

## What Kind of Stats Are Collected?

- Number of rows
    
- Number of dead rows (for VACUUM)
    
- Value distribution in columns
    
- Most common values
    
- Null fraction
    
- Histogram of values

PostgreSQL runs `ANALYZE` **automatically** as part of **autovacuum**, but:

- It may **not run often enough** if the table changes frequently.
    
- For **newly imported or large tables**, it’s **better to run it manually**.