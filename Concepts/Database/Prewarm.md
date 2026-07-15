---
tags:
  - database
  - messaging
  - postgresql
  - scaling
---

The `pg_prewarm` extension in PostgreSQL is used to preload relation data into either the operating system's buffer cache or PostgreSQL's shared buffer cache to improve query performance after a database restart or when specific tables need to be cached. It supports three modes: `buffer`, `prefetch`, and `read`. Below is a comparison of the `buffer` and `prefetch` modes, as these are the most commonly discussed in the context of warming caches.

### Overview of `pg_prewarm`
The `pg_prewarm` function allows you to load data from a specified relation (table or index) into memory. Its syntax is:

```sql
pg_prewarm(regclass, mode text default 'buffer', fork text default 'main', first_block int8 default null, last_block int8 default null) RETURNS int8
```

- **regclass**: The relation (table or index) to prewarm.
- **mode**: The prewarming method (`buffer`, `prefetch`, or `read`).
- **fork**: The relation fork to prewarm (usually `main` for the main data).
- **first_block** and **last_block**: Optional block range to prewarm (NULL means from start to end).
- **Returns**: The number of blocks prewarmed.

### Buffer Mode
- **Description**: The `buffer` mode loads the specified range of blocks directly into PostgreSQL's shared buffer cache (`shared_buffers`). This is the default mode.
- **Operation**: It reads the requested blocks from disk (or the OS cache) and places them into PostgreSQL's shared buffer cache, making them immediately available for database queries.
- **Performance Characteristics**:
  - **Synchronous**: The operation completes when all blocks are loaded into the shared buffer cache.
  - **Direct Impact**: Since the data is loaded into PostgreSQL’s shared buffers, subsequent queries accessing this data avoid disk I/O (assuming the data remains in the cache).
  - **Use Case**: Ideal when you want to ensure critical tables or indexes are in PostgreSQL’s memory for fast access, especially in environments with large `shared_buffers` settings.
  - **Limitations**:
    - Limited by the size of `shared_buffers`. If the relation is larger than the available shared buffer space, earlier blocks may be evicted as later ones are loaded.
    - Prewarmed data is not protected from cache eviction, so other database activity may displace it.
    - Higher memory usage within PostgreSQL, which could compete with other processes if `shared_buffers` is not appropriately sized.
  - **Example**:
    ```sql
    SELECT pg_prewarm('my_table', 'buffer');
    ```
    This loads the entire `my_table` into PostgreSQL’s shared buffer cache.

### Prefetch Mode
- **Description**: The `prefetch` mode issues asynchronous prefetch requests to the operating system, asking it to load the specified blocks into the OS buffer cache.
- **Operation**: It sends requests to the OS to preload data into its page cache without immediately loading it into PostgreSQL’s shared buffers. The actual loading into the OS cache is handled by the OS asynchronously.
- **Performance Characteristics**:
  - **Asynchronous**: The operation is non-blocking, so the `pg_prewarm` call returns quickly, but the data may not be in the OS cache immediately.
  - **OS-Dependent**: Requires OS support for asynchronous I/O (e.g., `posix_fadvise` on Linux). If not supported, it throws an error.
  - **Indirect Impact**: Data in the OS cache reduces disk I/O for PostgreSQL when it needs to read those blocks, but it still requires loading into `shared_buffers` for query execution, which may introduce a slight delay compared to `buffer` mode.
  - **Use Case**: Useful when you want to warm the OS cache without immediately consuming PostgreSQL’s shared buffer space, such as in systems with limited `shared_buffers` but ample OS cache.
  - **Limitations**:
    - Not supported on all platforms (e.g., may fail on systems without asynchronous I/O support).
    - Less direct control over PostgreSQL’s cache, as the data resides in the OS cache until accessed by the database.
    - Like `buffer` mode, prewarmed data in the OS cache is subject to eviction by other system activities.
  - **Example**:
    ```sql
    SELECT pg_prewarm('my_table', 'prefetch');
    ```
    This requests the OS to asynchronously load `my_table` into its page cache.

### Key Differences
| **Aspect**               | **Buffer Mode**                              | **Prefetch Mode**                           |
|--------------------------|----------------------------------------------|---------------------------------------------|
| **Target Cache**         | PostgreSQL shared buffer cache (`shared_buffers`) | Operating system page cache                |
| **Operation Type**       | Synchronous                                  | Asynchronous                                |
| **Platform Support**     | All platforms                                | Requires OS support for async I/O (e.g., Linux) |
| **Performance Impact**   | Immediate reduction in disk I/O for queries   | Reduces disk I/O but requires loading into `shared_buffers` |
| **Resource Usage**       | Consumes `shared_buffers` memory              | Minimal impact on PostgreSQL memory         |
| **Use Case**             | Critical tables/indexes for immediate access  | Preloading data to OS cache for later use   |
| **Eviction Risk**        | Subject to PostgreSQL cache eviction          | Subject to OS cache eviction                |
| **Error Handling**       | No platform-specific errors                  | Errors if async I/O is not supported       |

### When to Use Each Mode
- **Use `buffer` mode**:
  - When you have sufficient `shared_buffers` to hold critical tables or indexes.
  - For workloads where immediate query performance is critical (e.g., after a restart or for specific reporting tasks).
  - When you want to ensure data is directly available in PostgreSQL’s memory without relying on the OS cache.
  - Example: Prewarming frequently accessed tables or indexes at database startup to avoid initial query slowdowns.

- **Use `prefetch` mode**:
  - When you want to reduce the load on PostgreSQL’s shared buffers but still minimize disk I/O by leveraging the OS cache.
  - In environments where asynchronous I/O is supported, and you want non-blocking prewarming.
  - For large datasets where loading everything into `shared_buffers` is impractical due to memory constraints.
  - Example: Prewarming large tables on a system with ample OS cache but limited `shared_buffers`.

### Additional Notes
- **Cache Eviction**: Both modes load data that is not protected from eviction. Other database or system activity can displace prewarmed data, so prewarming is most effective at startup when caches are empty or during low-usage periods.[](https://www.postgresql.org/docs/current/pgprewarm.html)[](https://www.postgresql.org/docs/9.4/pgprewarm.html)
- **Autoprewarm**: Since PostgreSQL 11, the `pg_prewarm` extension supports an `autoprewarm` feature, which automatically saves and restores the shared buffer cache state across restarts. This is configured via `shared_preload_libraries = 'pg_prewarm'` and `pg_prewarm.autoprewarm = true` in `postgresql.conf`. It primarily uses `buffer` mode to restore the cache.[](https://www.enterprisedb.com/blog/autoprewarm-new-functionality-pgprewarm)[](https://www.enterprisedb.com/blog/autoprewarm-new-functionality-pgprewarm?lang=en)
- **Performance Testing**: Use `EXPLAIN (ANALYZE, BUFFERS)` to verify whether queries are hitting the cache (`shared hit`) or disk (`shared read`). This can help assess the effectiveness of prewarming.[](https://neon.tech/docs/extensions/pg_prewarm)
- **pg_hibernator**: Another extension, `pg_hibernator`, complements `pg_prewarm` by automatically saving and restoring buffer cache contents across restarts. It can work with multiple databases and is an alternative to `autoprewarm`.[](http://raghavt.blogspot.com/2014/06/utilising-caching-contribs-pgprewarm.html)[](https://github.com/DrPostgres/pg_hibernator)
- **Resource Considerations**: Be cautious with large tables, as prewarming can consume significant I/O and memory. Ensure sufficient `shared_buffers` for `buffer` mode or OS memory for `prefetch` mode. Monitor memory usage to avoid contention with other processes.[](https://philipmcclarence.com/guide-to-using-pg_prewarm/)

### Example Scenario
Suppose you have a table `orders` that is frequently queried after a database restart:
- To load it into PostgreSQL’s shared buffers:
  ```sql
  SELECT pg_prewarm('orders', 'buffer');
  ```
- To load it into the OS cache (if supported):
  ```sql
  SELECT pg_prewarm('orders', 'prefetch');
  ```
You can check the number of blocks loaded (each block is typically 8KB) and use `pg_buffercache` to inspect the shared buffer cache contents.[](https://neon.tech/docs/extensions/pg_prewarm)

### Conclusion
- Choose `buffer` mode for direct, immediate performance benefits in PostgreSQL’s memory, especially for critical or frequently accessed data.
- Choose `prefetch` mode for non-blocking prewarming into the OS cache, suitable for larger datasets or systems with constrained `shared_buffers`.
- Combine with `autoprewarm` or `pg_hibernator` for automated cache restoration after restarts to minimize manual intervention.

For more details, refer to the PostgreSQL documentation on `pg_prewarm`.[](https://www.postgresql.org/docs/current/pgprewarm.html)[](https://www.postgresql.org/docs/9.4/pgprewarm.html)

In the context of the `pg_prewarm` extension's `prefetch` mode in PostgreSQL, when data is prewarmed into the **operating system (OS) cache**, it is stored in the **OS page cache** (also called the buffer cache in some contexts). Here's a detailed explanation of where the data resides and whether it is still on disk:

### Where Is the Data Cached at the OS Level?
- **Location**: The data is cached in the **OS page cache**, which is a portion of **RAM** (main memory) managed by the operating system. The page cache acts as a buffer between the disk and applications (like PostgreSQL) to reduce disk I/O by keeping frequently accessed data in memory.
- **Not on Disk**: While the original data remains on disk (in the database files), the OS page cache holds a copy of the data in RAM. When PostgreSQL requests data that is in the OS page cache, the OS serves it from RAM instead of reading from the disk, significantly speeding up access.
- **Mechanism**: In `prefetch` mode, `pg_prewarm` uses system calls like `posix_fadvise` (on Linux) to advise the OS to preload specific blocks of the relation (table or index) into the page cache. This operation is asynchronous, meaning PostgreSQL doesn't wait for the data to be fully loaded into the cache before the `pg_prewarm` call completes.

### Key Points About the OS Page Cache
- **Memory-Based**: The OS page cache resides in RAM, not on disk. It uses available system memory to store file data that has been recently read from or written to disk.
- **Shared Across Processes**: The OS page cache is shared among all processes accessing the same files. For example, if multiple PostgreSQL instances or other applications read the same database files, they can benefit from the same cached data.
- **Eviction**: Data in the OS page cache is subject to eviction based on the OS's memory management policies (e.g., LRU - Least Recently Used). If the system needs memory for other tasks, cached data may be removed, requiring a disk read when accessed again.
- **Persistence**: The page cache is non-persistent; it is cleared on system reboot or when memory pressure forces the OS to reclaim space.

### Is the Data Still on Disk?
- **Yes, but Cached in RAM**: The original data remains on disk in the PostgreSQL data directory (e.g., table or index files). The OS page cache holds a copy of the requested blocks in RAM to speed up access. When PostgreSQL needs to read these blocks, the OS serves them from the cache (RAM) instead of the disk, reducing latency.
- **Behavior in `prefetch` Mode**:
  - `pg_prewarm` in `prefetch` mode tells the OS to load specific blocks into the page cache.
  - If PostgreSQL later requests these blocks, the OS provides them from RAM (if still cached) rather than reading from disk.
  - If the data is no longer in the OS cache (e.g., due to eviction), the OS will read it from disk again.

### Comparison to `buffer` Mode
- In contrast, `buffer` mode loads data directly into **PostgreSQL's shared buffer cache** (also in RAM but managed by PostgreSQL, configured via `shared_buffers`). This is distinct from the OS page cache, as it is exclusive to PostgreSQL and not shared with other processes.
- In `prefetch` mode, the data stays in the OS page cache until PostgreSQL requests it, at which point it may be copied into PostgreSQL’s shared buffers for query processing.

### Practical Implications
- **Performance**: Data in the OS page cache reduces disk I/O but still requires a copy into PostgreSQL’s shared buffers when accessed, which is slightly slower than having data already in `shared_buffers` (as in `buffer` mode).
- **Memory Usage**: `prefetch` mode is less taxing on PostgreSQL’s memory (`shared_buffers`) but relies on available system RAM. Ensure the system has enough free memory to hold the OS page cache without causing memory pressure.
- **Verification**: You can check OS cache usage on Linux using tools like `vmtouch` or by inspecting `/proc/meminfo` (look for `Cached` memory). To confirm PostgreSQL is hitting the OS cache, use `EXPLAIN (ANALYZE, BUFFERS)` and look for reduced `shared read` (disk reads) vs. `shared hit` (cache hits, though these reflect PostgreSQL’s buffer cache).

### Example
Running:
```sql
SELECT pg_prewarm('my_table', 'prefetch');
```
This advises the OS to load blocks of `my_table` into the OS page cache (RAM). The data remains on disk, but a copy is cached in RAM for faster access. If you query `my_table`, PostgreSQL may fetch the data from the OS cache (RAM) instead of disk, depending on whether the data is still cached.

### Limitations
- **Platform Dependency**: `prefetch` mode requires OS support for asynchronous I/O (e.g., `posix_fadvise` on Linux). It may fail on systems without this support (e.g., some older Unix systems or Windows).
- **Cache Eviction**: The OS may evict data from the page cache if memory is needed elsewhere, unlike PostgreSQL’s shared buffers, which are dedicated to the database.
- **Monitoring**: It’s harder to directly inspect the OS page cache contents compared to PostgreSQL’s shared buffers (which can be checked with the `pg_buffercache` extension).

### Conclusion
In `prefetch` mode, the data is cached in the **OS page cache**, which resides in **RAM**, not on disk. The original data remains on disk, but the OS keeps a copy in memory to reduce disk I/O. This makes `prefetch` mode useful for warming large datasets without consuming PostgreSQL’s `shared_buffers`, though it relies on OS memory management and support for asynchronous I/O. For critical data requiring immediate access, consider `buffer` mode to load directly into PostgreSQL’s shared buffers instead.