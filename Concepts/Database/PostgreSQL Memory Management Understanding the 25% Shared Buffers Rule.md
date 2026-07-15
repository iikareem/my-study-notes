---
tags:
  - database
  - memory
  - postgresql
  - scaling
---

**Tags:** `#postgresql` `#database-performance` `#memory-management` `#os-cache`

To understand why PostgreSQL recommends allocating only 25% of total RAM to its own internal cache (`shared_buffers`), we first need to understand how the Operating System (OS) handles data storage and memory.

## Prerequisites: How Data Moves

### 1. Buffered I/O (Input/Output)

Buffered I/O is a technique used by computers to speed up the process of reading from or writing to slow storage devices (like hard drives or SSDs) by using a fast, temporary holding area in RAM called a **buffer**.

> **The Grocery Store Analogy**
> 
> - **Unbuffered I/O:** Walking to the store for a single egg, walking home, then walking back for a cup of flour. It is exhausting and slow.
>     
> - **Buffered I/O:** Going to the store once, buying a whole cart of ingredients, and keeping them in your kitchen pantry (the buffer). When you need an ingredient, you grab it instantly from the pantry.
>     

- **Buffered Reads:** When an application asks for a tiny piece of data, the OS grabs a large "block" of surrounding data from the disk and puts it in fast RAM. The next time the application needs data, it is instantly available from the RAM buffer.
    
- **Buffered Writes:** When an application saves data, it drops it into the fast RAM buffer and moves on to its next task. The OS efficiently batches these small writes together and flushes them to the physical disk in the background.
    

### 2. The OS Page Cache

The OS Page Cache is the operating system's implementation of Buffered I/O.

A common misconception is that the OS Page Cache is memory reserved for internal OS tasks (like running the kernel). In reality, **it exists to serve applications like PostgreSQL.**

- **It is "Borrowed" Memory:** The OS Page Cache is highly elastic. The OS will use almost all "free" or unused RAM on your server to cache files from the disk.
    
- **It is Expendable:** If an application suddenly needs actual RAM to execute a complex task, the OS will instantly drop old cached files and hand that RAM directly to the application. It is free memory put to good use, rather than memory locked away.
    

## Why is `shared_buffers` Only 25% of Total RAM?

If PostgreSQL is the most important application on a server, it feels intuitive to give it 80% or 90% of the RAM. However, the standard starting heuristic in the Postgres community is to set `shared_buffers` to **25% of total RAM**. Here is why:

### 1. The "Double Buffering" Problem

Unlike some databases that bypass the OS to talk directly to the disk, PostgreSQL relies heavily on the OS.

When Postgres reads data, the OS fetches it from the disk and stores a copy in the **OS Page Cache**. Postgres then copies that data into its own memory space: **`shared_buffers`**.

If you set `shared_buffers` to 80% of your RAM, a massive chunk of your database is sitting in memory _twice_—once in the OS cache and once in Postgres. Keeping `shared_buffers` at 25% avoids wasting RAM on duplicate data.

### 2. The Power of the OS Page Cache

PostgreSQL developers intentionally rely on the operating system because Linux (and other OSs) have decades of optimization for file caching:

- **Smarter "Read-Ahead":** The OS recognizes sequential scans (like reading an entire table) and pre-loads upcoming data into the Page Cache before Postgres even asks for it.
    
- **Dynamic Elasticity:** As mentioned in the prerequisites, the OS cache can instantly shrink if RAM is needed elsewhere. `shared_buffers`, however, is a rigid, locked block of memory reserved entirely for Postgres from the moment the database starts.
    
- **Write Consolidation:** The OS cache efficiently batches tiny writes and flushes them to the disk smoothly, preventing disk thrashing.
    

### 3. RAM is Needed for Operations, Not Just Caching

If `shared_buffers` hoards all the RAM, it starves other critical PostgreSQL operations:

- **`work_mem`:** Used for sorting (e.g., `ORDER BY`) and joining tables. This memory is allocated _per connection_. If 100 concurrent users run complex queries, they need plenty of free RAM, otherwise, queries spill over to the slow disk.
    
- **`maintenance_work_mem`:** Background tasks like `VACUUM` (cleaning up dead rows) and creating indexes require large chunks of memory to run quickly.
    

## Sizing Reality Check

The 25% rule is a **starting heuristic**, not an absolute law.

- **Small Servers:** On a 1GB RAM server, 25% (256MB) is a good starting point.
    
- **Massive Servers:** If a server has 512GB of RAM, 25% (128GB) is usually _too big_. Massive `shared_buffers` can cause severe CPU spikes when Postgres has to scan or flush it during a "checkpoint." In very large systems, Database Administrators often cap `shared_buffers` between 8GB and 32GB, leaving the vast majority of the RAM for the highly efficient OS Page Cache and active query execution.