---
tags:
  - database
  - postgresql
  - scaling
---

The `pg_stat_statements` view in PostgreSQL provides detailed statistics about SQL query performance and resource usage. Below is an explanation of the most important columns in `pg_stat_statements`, focusing on those that are critical for performance analysis and tuning:

1. **queryid**:
   - **Description**: A unique identifier for the query, generated based on the query's normalized text. It allows you to track the same query across multiple executions, even if parameters change.
   - **Importance**: Essential for grouping and analyzing the performance of specific queries over time. It helps identify which queries are executed frequently or are resource-intensive.

2. **query**:
   - **Description**: The text of the query, with parameter placeholders (e.g., `$1`, `$2`) for parameterized queries. The exact text may be normalized to remove specific values.
   - **Importance**: Provides the actual SQL statement being analyzed, allowing you to understand what the query does and correlate it with application behavior.

3. **calls**:
   - **Description**: The total number of times the query has been executed since the statistics were last reset.
   - **Importance**: Indicates query frequency. High call counts can highlight queries that are executed often and may benefit from optimization, even if individual execution time is low.

4. **total_exec_time** (or `total_time` in older versions):
   - **Description**: The total time (in milliseconds) spent executing the query across all calls, excluding planning time.
   - **Importance**: A key metric for identifying slow queries. Queries with high `total_exec_time` are often candidates for optimization, especially if they have high call counts.

5. **rows**:
   - **Description**: The total number of rows retrieved or affected by the query across all executions.
   - **Importance**: Helps assess the query's data volume. A query returning or processing many rows may indicate inefficiencies, such as missing indexes or overly broad queries.

6. **shared_blks_hit**:
   - **Description**: The number of shared buffer blocks found in memory (cache hits) during query execution.
   - **Importance**: Indicates how much the query benefits from cached data. High values suggest good cache usage, while low values may indicate frequent disk I/O, potentially requiring more memory or index tuning.

7. **shared_blks_read**:
   - **Description**: The number of shared buffer blocks read from disk during query execution. 
   - when **`shared_blks_read`** increases in PostgreSQL, it **means the block was read from disk into the shared buffer cache** (PostgreSQL’s buffer pool). so that further accesses can reuse it without hitting the disk again.
   - **Importance**: High values indicate disk I/O, which can slow down queries. Comparing `shared_blks_hit` to `shared_blks_read` helps evaluate cache efficiency.

8. **shared_blks_dirtied**:
   - **Description**: The number of shared buffer blocks modified (dirtied) by the query.
   - **Importance**: Even `SELECT` queries can dirty blocks (e.g., due to hint bit updates or HOT pruning). High values may indicate unexpected modifications or maintenance activity impacting performance.

9. **shared_blks_written**:
   - **Description**: The number of shared buffer blocks written to disk during query execution.
   - **Importance**: High values can indicate buffer pool contention or frequent evictions of dirty blocks, which may slow down queries and suggest a need for larger shared buffers or checkpoint tuning.

	1. **blk_read_time** and **blk_write_time**:
    - **Description**: The total time (in milliseconds) spent reading and writing blocks from/to disk, respectively. These are only populated if `track_io_timing` is enabled.
    - **Importance**: Critical for identifying I/O bottlenecks. High values suggest that disk performance or insufficient memory is impacting query performance.

11. **temp_blks_read** and **temp_blks_written**:
    - **Description**: The number of temporary blocks read and written, typically for operations like sorts, hashes, or materializations that spill to disk.
    - **Importance**: High values indicate that the query is generating temporary files, often due to insufficient `work_mem`. Increasing `work_mem` or optimizing the query may help.

12. **wal_records**, **wal_fpi**, **wal_bytes** (if `track_wal_io_timing` is enabled):
    - **Description**: Metrics related to Write-Ahead Logging (WAL) activity, including the number of WAL records generated, full-page images (`wal_fpi`), and bytes written (`wal_bytes`).
    - **Importance**: Useful for understanding the query’s impact on WAL activity, which affects replication and crash recovery. High values may indicate heavy write activity, even for `SELECT` queries that trigger maintenance tasks.

13. **min_exec_time**, **max_exec_time**, **mean_exec_time**, **stddev_exec_time**:
    - **Description**: The minimum, maximum, mean, and standard deviation of execution times (in milliseconds) for the query.
    - **Importance**: These provide insight into query performance variability. A high `stddev_exec_time` suggests inconsistent performance, possibly due to varying data sizes, contention, or caching effects.

### Why These Columns Matter
These columns collectively help you:
- **Identify costly queries**: High `total_exec_time`, `calls`, or I/O metrics (`shared_blks_read`, `blk_read_time`) highlight queries that consume significant resources.
- **Diagnose performance issues**: Metrics like `shared_blks_hit` vs. `shared_blks_read` reveal cache efficiency, while `temp_blks_written` points to memory-related issues.
- **Optimize database configuration**: High `shared_blks_written` or `wal_bytes` may suggest tuning `shared_buffers`, `checkpoint_timeout`, or other parameters.
- **Understand query behavior**: Columns like `rows`, `shared_blks_dirtied`, and `wal_records` help explain why even read-only `SELECT` queries can have side effects like dirtying or writing blocks.

### Practical Usage
To analyze `pg_stat_statements` effectively:
- Sort by `total_exec_time` to find the most time-consuming queries.
- Check `shared_blks_hit` vs. `shared_blks_read` to assess cache hit ratios.
- Look at `temp_blks_written` to identify queries that may benefit from increasing `work_mem`.
- Monitor `shared_blks_dirtied` and `shared_blks_written` for unexpected write activity in `SELECT` queries.
- Use `queryid` and `query` to correlate statistics with specific application code.

For deeper analysis, you can enable additional tracking (e.g., `track_io_timing`) or use extensions like `pg_stat_statements` with `EXPLAIN ANALYZE` to investigate specific queries. Would you like me to provide an example query to extract key metrics from `pg_stat_statements` or search for additional resources on this topic?

The columns in pg_stat_statements like shared_blks_hit, shared_blks_read, shared_blks_dirtied, and shared_blks_written represent the **total number of blocks** affected across **all executions** of a query, not a per-call metric multiplied by the number of calls. These metrics are cumulative totals for the query identified by queryid since the last pg_stat_statements reset. Let me clarify how these relate to calls and whether they can be interpreted as calls * some_per_call_value.

### Key Points

1. **What shared_blks_hit and Similar Metrics Represent**:
    - shared_blks_hit: Total number of shared buffer blocks found in memory (cache hits) across all executions of the query.
    - shared_blks_read: Total number of shared buffer blocks read from disk.
    - shared_blks_dirtied: Total number of shared buffer blocks modified (dirtied).
    - shared_blks_written: Total number of shared buffer blocks written to disk.
    - These are **aggregate counts** of blocks processed during all calls (executions) of the query, not a simple function of calls * some_per_call_value.
2. **Relationship with calls**:
    - The calls column shows the total number of times the query was executed.
    - The block-related metrics (shared_blks_hit, etc.) depend on the query’s behavior during each execution. For example, a query might access 10 blocks in one execution and 20 in another, so shared_blks_hit is the sum of all blocks accessed across all calls, not calls * a_fixed_number.
    - To get the **average** number of blocks per call, you can divide these metrics by calls. For example:
        - Average shared_blks_hit per call = shared_blks_hit / calls
        - Average shared_blks_read per call = shared_blks_read / calls
3. **Why It’s Not Simply calls * shared_blks_hit**:
    - The number of blocks accessed (hit, read, dirtied, or written) varies per execution based on factors like:
        - Data size and distribution.
        - Cache state (whether data is already in memory or needs to be read from disk).
        - Query plan (e.g., sequential scan vs. index scan).
        - System activity (e.g., buffer evictions or background writer activity).
    - For example, a SELECT query might hit 100 blocks in one execution (all in cache) and 50 blocks in another (some read from disk). The total shared_blks_hit is the sum of these, not a fixed number multiplied by calls.
4. **Interpreting with calls**:
    - To understand the query’s behavior per execution, calculate averages:
        
        
    
        `SELECT query, calls, shared_blks_hit, shared_blks_read, shared_blks_dirtied, shared_blks_written, (shared_blks_hit::float / calls) AS avg_blks_hit_per_call, (shared_blks_read::float / calls) AS avg_blks_read_per_call, (shared_blks_dirtied::float / calls) AS avg_blks_dirtied_per_call, (shared_blks_written::float / calls) AS avg_blks_written_per_call FROM pg_stat_statements WHERE calls > 0;`
        
    - This query computes the average number of blocks per call, which can help you understand typical resource usage for a query.
5. **Why These Metrics Vary**:
    - For shared_blks_hit: Depends on whether the required data is in the shared buffer cache. More cache hits mean lower shared_blks_read.
    - For shared_blks_read: Increases when data isn’t in the cache and must be read from disk.
    - For shared_blks_dirtied: Even SELECT queries can dirty blocks (e.g., setting hint bits or HOT pruning, as explained previously).
    - For shared_blks_written: Dirty blocks may be written to disk due to buffer eviction or checkpoints, influenced by system-wide activity, not just the query itself.

### Example

Suppose a query has:

- calls = 100
- shared_blks_hit = 10,000
- shared_blks_read = 2,000
- shared_blks_dirtied = 50
- shared_blks_written = 20

This means:

- Across 100 executions, the query accessed 10,000 blocks from the cache (shared_blks_hit) and 2,000 from disk (shared_blks_read).
- It dirtied 50 blocks and wrote 20 to disk.
- On average:
    - shared_blks_hit per call = 10,000 / 100 = 100 blocks
    - shared_blks_read per call = 2,000 / 100 = 20 blocks
    - shared_blks_dirtied per call = 50 / 100 = 0.5 blocks
    - shared_blks_written per call = 20 / 100 = 0.2 blocks
- These averages give you a sense of the query’s typical resource usage, but the actual number of blocks per call varies based on execution context.

### Practical Implications

- **High shared_blks_hit**: Indicates good cache usage, but if paired with high calls, the query might still be a performance bottleneck.
- **High shared_blks_read**: Suggests frequent disk I/O, which could be mitigated by increasing shared_buffers or optimizing the query/indexes.
- **Non-zero shared_blks_dirtied or shared_blks_written for SELECT queries**: Indicates internal maintenance (e.g., hint bits or pruning). Investigate with EXPLAIN ANALYZE to check for unexpected behavior.
- Use averages (metric / calls) to normalize metrics and compare queries with different call counts.

when shared_blks_dirtied and shared_blks_written can have a numbers and why select can have also these values

In PostgreSQL's pg_stat_statements, the shared_blks_dirtied and shared_blks_written columns track the number of shared buffer blocks that are modified (dirtied) and written to disk, respectively, during query execution. While these metrics are often associated with write operations (e.g., INSERT, UPDATE, DELETE), SELECT queries can also contribute to these values due to internal database mechanisms. Below, I’ll explain when and why these columns can have non-zero values, with a focus on why SELECT queries can result in these metrics.

### Understanding shared_blks_dirtied and shared_blks_written

1. **shared_blks_dirtied**:
    - **Definition**: Counts the total number of shared buffer blocks modified (marked as "dirty") by a query across all its executions.
    - **What it means**: A block is dirtied when its contents in the shared buffer cache are changed, requiring it to be written to disk later to persist the change.
    - **When it happens**: Any operation that modifies a block in the shared buffer cache increments this counter. This includes changes to data pages, index pages, or metadata.
2. **shared_blks_written**:
    - **Definition**: Counts the total number of shared buffer blocks written to disk during query execution.
    - **What it means**: A block is written to disk when a dirty block in the shared buffer cache is flushed, typically due to buffer pool eviction, checkpoints, or background writer activity.
    - **When it happens**: Writes occur when PostgreSQL needs to free up buffer space or ensure data consistency (e.g., during a checkpoint).

### Why SELECT Queries Can Have Non-Zero shared_blks_dirtied and shared_blks_written

Although SELECT queries are read-only from a user perspective, they can trigger internal operations in PostgreSQL that modify or write blocks, leading to non-zero values in these columns. Here’s why:

#### Causes of shared_blks_dirtied in SELECT Queries

A SELECT query can dirty shared buffer blocks due to the following mechanisms:

1. **Hint Bit Updates**:
    - **What happens**: When a tuple (row) is read for the first time, PostgreSQL may set "hint bits" in the tuple header to mark the transaction status (e.g., committed or aborted). This is done to optimize future visibility checks.
    - **Impact**: Setting hint bits modifies the data page in the shared buffer, marking it as dirty and incrementing shared_blks_dirtied.
    - **Example scenario**: A SELECT query scans a table that was recently modified (e.g., by an INSERT or UPDATE), and the tuples accessed need their hint bits updated.
2. **Heap-Only Tuple (HOT) Pruning**:
    - **What happens**: During a SELECT, PostgreSQL may opportunistically perform HOT pruning, which removes dead tuples (from previous UPDATE or DELETE operations) from a heap page to reclaim space.
    - **Impact**: Pruning modifies the heap page, marking it as dirty and incrementing shared_blks_dirtied.
    - **Example scenario**: A SELECT query scans a frequently updated table, triggering cleanup of dead tuples on the page.
3. **Visibility Map Updates**:
    - **What happens**: PostgreSQL maintains a visibility map to track which pages are "all-visible" (i.e., all tuples are visible to all transactions). A SELECT query may determine that a page is all-visible and update the visibility map accordingly.
    - **Impact**: Updating the visibility map modifies a block, marking it as dirty and incrementing shared_blks_dirtied.
    - **Example scenario**: A SELECT query scans a page after a transaction commits, allowing PostgreSQL to mark the page as all-visible.
4. **Index Maintenance**:
    - **What happens**: If a SELECT query uses an index, it may trigger minor metadata updates (e.g., updating index page pointers or statistics).
    - **Impact**: These updates can dirty index pages, contributing to shared_blks_dirtied.
    - **Example scenario**: A SELECT with a complex index scan updates index metadata during execution.

#### Causes of shared_blks_written in SELECT Queries

A SELECT query can contribute to shared_blks_written when dirty blocks (whether dirtied by the SELECT or other operations) are written to disk. This happens in the following cases:

1. **Buffer Pool Eviction**:
    - **What happens**: If the shared buffer pool is full, PostgreSQL may evict dirty blocks to make space for new data needed by the SELECT query. Evicting a dirty block requires writing it to disk.
    - **Impact**: The write operation increments shared_blks_written, even though the SELECT query didn’t directly modify the data.
    - **Example scenario**: A SELECT query scans a large table, causing other dirty blocks (from unrelated operations) to be evicted from the buffer pool.
2. **Checkpoint or Background Writer Activity**:
    - **What happens**: PostgreSQL’s background writer or checkpoint processes periodically write dirty blocks to disk to ensure data consistency. If these processes run during a SELECT query’s execution, any dirty blocks (including those dirtied by the SELECT) may be written.
    - **Impact**: This increments shared_blks_written, even for a read-only query.
    - **Example scenario**: A long-running SELECT query coincides with a checkpoint, causing dirty blocks to be written.
3. **Dirty Blocks from SELECT-Induced Changes**:
    - **What happens**: If a SELECT query dirties blocks (e.g., via hint bits, HOT pruning, or visibility map updates), those blocks may be written to disk during the query’s execution due to buffer eviction or checkpoints.
    - **Impact**: This directly ties shared_blks_dirtied to shared_blks_written.
    - **Example scenario**: A SELECT query sets hint bits, dirtying a block, and that block is later evicted during the same query.

### When These Metrics Are Non-Zero

- **Forizard: **For SELECT queries**, non-zero shared_blks_dirtied and shared_blks_written are most likely when:
    - The query accesses recently modified data (triggering hint bit updates).
    - The query triggers maintenance tasks like HOT pruning or visibility map updates.
    - The query causes buffer pool contention, leading to eviction of dirty blocks.
    - The database is under heavy load, or checkpoints/background writer processes are active.
- **For other queries** (INSERT, UPDATE, DELETE):
    - These queries directly modify data or index pages, naturally dirtying blocks (shared_blks_dirtied).
    - Dirty blocks are written to disk during checkpoints, buffer evictions, or explicit writes, incrementing shared_blks_written.

### Why SELECT Queries Specifically Have These Values

- SELECT queries are read-only but can trigger **implicit modifications** (hint bits, HOT pruning, visibility map updates) that dirty blocks.
- They can also indirectly cause writes by forcing dirty blocks out of the buffer pool or coinciding with background processes like checkpoints.
- The likelihood of non-zero values increases with:
    - Large or frequently updated tables.
    - High database activity (causing buffer contention or frequent checkpoints).
    - Queries that scan many pages or use indexes heavily.

### Practical Implications

- **High shared_blks_dirtied in SELECT queries**:
    - Investigate whether hint bit updates or HOT pruning are frequent (e.g., using EXPLAIN ANALYZE to check page access patterns).
    - Consider running VACUUM or ANALYZE to pre-set hint bits or clean up dead tuples, reducing the need for SELECT to perform these tasks.
- **High shared_blks_written in SELECT queries**:
    - Check if the buffer pool (shared_buffers) is too small, causing frequent evictions.
    - Tune checkpoint settings (e.g., checkpoint_timeout, max_wal_size) to reduce write activity during queries.
    - Optimize queries to reduce the number of blocks accessed.

### Example

Suppose a SELECT query has:

- calls = 100
- shared_blks_dirtied = 500
- shared_blks_written = 200 This means across 100 executions, the query dirtied 500 blocks (e.g., due to hint bit updates or pruning) and caused 200 blocks to be written to disk (e.g., due to buffer evictions or checkpoints). On average, that’s 5 blocks dirtied and 2 blocks written per call, but the actual number varies per execution based on data state and system activity.

If you’d like, I can provide a SQL query to analyze these metrics in pg_stat_statements or search for specific examples on the web or X to illustrate real-world cases. Let me know!