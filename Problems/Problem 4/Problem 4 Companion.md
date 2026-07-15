---
tags:
  - companion
  - database
  - interview
  - locking
  - postgresql
  - problem-4
  - problems
  - skip-locked
  - system-design
---

# Companion — FOR UPDATE SKIP LOCKED

**Domain:** Database Internals & Performance  
**Topics:** FOR UPDATE · SKIP LOCKED · worker queues

**Pairs with:** [[Our background job is locking the entire table and nobody knows why]] · [[Problems MOC]]

`FOR UPDATE SKIP LOCKED`

The `FOR UPDATE SKIP LOCKED` clause is not a "new" feature (it was introduced in PostgreSQL 9.5 and is supported in MySQL 8+, Oracle, and other engines), but it is a **powerful tool** for concurrency that works very differently from a standard `SELECT`.

When you run your query—`SELECT id FROM audit_logs ... FOR UPDATE SKIP LOCKED`—you are essentially building a **high-concurrency job queue** inside your database.

### What it actually does

- **`FOR UPDATE`**: Tells the database to acquire an **Exclusive Lock** on the rows it finds. This prevents other transactions from modifying or deleting those rows until you are done.
    
- **`SKIP LOCKED`**: This is the "magic" part. Instead of waiting for other transactions to release their locks (which is the default, blocking behavior), **the database simply ignores any row that is already locked by someone else.**
    

If your query searches for 5,000 rows, and 100 of them are currently being processed (locked) by another worker, the database will skip those 100 and return the next 4,900 available rows instead.

### Why use this for `audit_logs`?

This pattern is the industry standard for **parallel background processing**. If you are building a system where multiple "worker" processes need to clean up or process old `audit_logs` simultaneously, `SKIP LOCKED` is how you keep them from stepping on each other:

1. **Zero Contention:** Worker A grabs 5,000 logs and locks them.
    
2. **Instant Parallelism:** Worker B runs the exact same query. Because it uses `SKIP LOCKED`, it doesn't wait for Worker A; it immediately skips the logs Worker A is holding and grabs the _next_ 5,000 available logs.
    
3. **Efficiency:** No processes sit in "Wait" states, and you don't need to write complex application code to track which worker is doing what.
    

### Important Warnings

While it is great for performance, keep these three things in mind:

- **Incomplete Results:** By design, you won't always get the _exact_ 5,000 rows you might expect if other workers are active. You get "whatever is currently available."
    
- **Not a Data Integrity Tool:** Do not use this to guarantee you have processed every single row in a specific order. If you need absolute 100% guarantee of order and completion, you need a formal queueing system or a status-flag based state machine.
    
- **Use Indexes:** If your `WHERE` clause (in your case, `created_at < ...`) is not indexed, the database will be forced to perform a **Table Scan**. When combined with `FOR UPDATE`, this can accidentally lock far more rows than you intended, potentially causing major performance issues across your entire application.
    

**Summary:** You aren't just "locking a page." You are implementing a **non-blocking worker pattern** that allows multiple threads to pull from the same table in parallel without ever getting stuck behind each other.

If you are setting this up for a cleanup task, would you like me to help you verify your indexing strategy to ensure this doesn't cause a table-wide performance bottleneck?

## Next

- Scenario: [[Our background job is locking the entire table and nobody knows why]]
- Back to hub: [[Problems MOC]]
