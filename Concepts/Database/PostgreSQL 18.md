---
tags:
  - database
  - postgresql
---

### 1. The Pre-PG 18 World: Relying on the OS Brain

Before PostgreSQL 18, Postgres relied almost entirely on the Operating System (Linux, Windows, etc.) to handle "read-ahead" (prefetching data before the database officially asks for it).

The OS is incredibly smart, but it has a major blind spot: **it doesn't understand SQL queries.** It only sees a program asking for blocks of a file.

- **The Good (Sequential Scans):** If you run a query that reads an entire table from start to finish (`SELECT * FROM massive_table`), Postgres asks the OS for Block 1, then Block 2, then Block 3. The OS instantly recognizes this pattern. It says, _"Ah, they are reading sequentially!"_ and silently fetches Blocks 4, 5, and 6 into the RAM cache in the background. By the time Postgres needs them, they are already there. It is blazingly fast.
    
- **The Bad (Random Access):** If you run an Index Scan or a Bitmap Heap Scan, Postgres doesn't need data in order. It needs Block 12, then Block 950, then Block 42. When Postgres asks for Block 12, the OS fetches it. When it asks for Block 950, the OS fetches it. The OS cannot see a pattern, so **it completely stops prefetching.** ### 2. The Bottleneck: Synchronous Blocking
    
    Because the OS couldn't predict random access, Postgres pre-18 was trapped using **Synchronous I/O** for these operations.
    

This meant Postgres had to wait in a literal line:

1. Postgres asks the OS for Block 12.
    
2. Postgres goes to sleep (blocks) waiting for the slow disk to find it.
    
3. The disk finds it and hands it over. Postgres wakes up.
    
4. Postgres asks for Block 950 and goes back to sleep.
    

If you are running on cloud storage (like AWS EBS or Azure Managed Disks) where network latency adds a tiny delay to every single disk request, this "stop-and-wait" loop absolutely kills performance for read-heavy random workloads.

### 3. The PostgreSQL 18 Solution: Asynchronous I/O (AIO)

In PostgreSQL 18, the developers (led by hackers like Thomas Munro) decided Postgres shouldn't rely blindly on the OS anymore. Postgres _does_ know the future because it generates the **Query Plan**. If Postgres is doing a Bitmap Heap Scan, it already knows exactly which 50 random blocks it will need before it even starts fetching them.

PostgreSQL 18 introduces a full **Asynchronous I/O** subsystem. Here is how it changes the game:

- **Batching Requests:** Instead of asking for one block and going to sleep, Postgres looks at its query plan and fires off dozens of read requests for random blocks _all at the same time_.
    
- **Overlapping Work:** Postgres hands this batch of requests to new background workers (or directly to the Linux kernel via a hyper-fast method called `io_uring`). The disk starts fetching Block 12, 950, and 42 concurrently.
    
- **No More Sleeping:** While the disk is doing the heavy lifting to find those random blocks, the main Postgres process doesn't go to sleep. It stays awake and continues doing CPU work (like sorting data or processing the blocks that have already arrived).
    

### The Result

By doing its own intelligent read-ahead and asking for multiple random blocks concurrently, PostgreSQL 18 turns a slow, serialized crawl into a massive, parallel data stream. Benchmarks on cloud environments have shown read-heavy workloads speeding up by **2x to 3x** simply because the database is no longer sitting idle waiting for single blocks of data to arrive one by one.