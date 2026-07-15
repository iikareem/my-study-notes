---
tags:
  - database
  - postgresql
  - scaling
---

### How Neon’s LFC Works
- **Caching Mechanism**: The LFC acts as an extension of PostgreSQL’s `shared_buffers`. When a query requests data, PostgreSQL first checks `shared_buffers`. If the data isn’t there, Neon checks the LFC before fetching from the Pageserver. The LFC caches **pages** (8KB units of data) from tables and indexes based on recent access patterns.[](https://neon.com/docs/extensions/neon)[](https://neon.tech/docs/reference/glossary)
- **Automatic Management**: The LFC uses a **least recently used (LRU)** eviction policy, similar to PostgreSQL’s buffer cache, to retain frequently accessed pages. Pages that are accessed often (part of the **working set**) are more likely to stay in the LFC, while infrequently accessed pages may be evicted.[](https://neon.com/docs/extensions/neon)[](https://neon.tech/docs/manage/endpoints)
- **Metrics for Monitoring**: The `neon_stat_file_cache` view provides metrics like `file_cache_hits` and `file_cache_misses`, which show how often data is served from the LFC versus fetched from storage. The cache hit ratio (`file_cache_hits / (file_cache_hits + file_cache_misses) * 100`) indicates cache efficiency. A high ratio (e.g., 99% for OLTP workloads) suggests that frequently accessed data is cached effectively.[](https://neon.com/docs/extensions/neon)

### Can You Determine Which Indexes or Tables Are Cached?
- **No Direct Control**: Neon does not provide a mechanism to explicitly specify which tables or indexes should be cached in the LFC. The caching decisions are handled automatically by PostgreSQL’s buffer manager and Neon’s LFC, based on which pages are accessed most frequently. This is because caching is designed to adapt dynamically to query patterns, prioritizing the **working set** (frequently accessed data and indexes).[](https://neon.tech/docs/manage/endpoints)[](https://neon.tech/docs/manage/computes)
- **Why No Direct Control?**:
  - **Dynamic Workloads**: Database workloads vary, and manually specifying cached objects could lead to suboptimal performance if access patterns change. The LRU policy ensures that the most relevant data stays in memory.
  - **PostgreSQL Integration**: Neon builds on PostgreSQL’s buffer management, which doesn’t support pinning specific tables or indexes in memory. The LFC extends this model but follows similar principles.
  - **Scalability**: Neon’s serverless architecture, with separated compute and storage, relies on automated caching to support features like autoscaling and branching. Manual control would complicate this design.[](https://semaphore.io/blog/neon-database)

### How to Influence What Gets Cached in the LFC
While you can’t directly specify which tables or indexes to cache, you can influence caching behavior to ensure frequently accessed data stays in the LFC:

1. **Optimize Query Patterns**:
   - Write queries to access only the necessary data (e.g., use selective `WHERE` clauses or projections). This reduces the working set size, making it more likely to fit in the LFC.
   - Example: Instead of `SELECT * FROM large_table`, use `SELECT column1, column2 FROM large_table WHERE condition` to limit the pages accessed.

2. **Use Indexes Strategically**:
   - Create indexes on frequently queried columns to reduce the number of pages scanned. For example, an index on `customer_id` for a query like `SELECT * FROM sales WHERE customer_id = 123` can reduce heap page access, keeping index pages in the LFC.[](https://neon.tech/blog/performance-tips-for-neon-postgres)
   - Use tools like `EXPLAIN ANALYZE` to confirm that queries use indexes instead of sequential scans, which access fewer pages and are more likely to stay cached.

3. **Monitor and Size the LFC**:
   - Check the LFC hit ratio using the `neon_stat_file_cache` view or Neon’s Monitoring dashboard. If the hit ratio is low (e.g., below 90%), your working set may exceed the LFC size.[](https://neon.com/docs/extensions/neon)[](https://neon.tech/docs/introduction/monitoring-page)
   - Increase compute size to allocate more RAM to the LFC (75% of compute RAM). For example, a 4 CU compute (16 GB RAM) provides a ~12 GB LFC, which can cache more data. Compare your working set size (from the Monitoring dashboard) to the LFC size to ensure it fits.[](https://neon.tech/docs/manage/endpoints)[](https://neon.tech/blog/key-neon-metrics-to-monitor-via-datadog)

4. **Prewarm the Cache**:
   - Use the `pg_prewarm` extension to preload specific tables or indexes into memory after a compute restart or index creation. For example:
     ```sql
     SELECT pg_prewarm('index_name');
     ```
     This loads the specified index or table into `shared_buffers` (and potentially the LFC), increasing the likelihood it stays cached for subsequent queries.[](https://neon.com/docs/extensions/pg_search)
   - Note: `pg_prewarm` is a one-time operation and doesn’t pin data in the cache; the LRU policy may still evict it if it’s not frequently accessed.

5. **Manage Table Bloat**:
   - Bloated tables or indexes increase the number of pages accessed, reducing cache efficiency. Run `VACUUM` or `REINDEX` to reduce bloat, ensuring that only necessary pages are accessed and cached.[](https://neon.tech/blog/performance-tips-for-neon-postgres)[](https://neon.com/docs/introduction/usage-metrics)
   - Example: `REINDEX INDEX idx_sales_customer_id` to rebuild a bloated index, making it more compact and cache-friendly.

6. **Monitor with `pg_stat_statements`**:
   - Use `pg_stat_statements` to identify queries with high `shared_blks_read` (indicating disk access) versus `shared_blks_hit` (cache hits). Optimize these queries or their underlying tables/indexes to increase cache hits, which also benefits the LFC.[](https://neon.com/docs/extensions/neon)
   - Example query to check cache efficiency:
     ```sql
     SELECT query, calls, shared_blks_hit, shared_blks_read,
            (shared_blks_hit::float / (shared_blks_hit + shared_blks_read) * 100) AS hit_ratio
     FROM pg_stat_statements
     WHERE shared_blks_hit + shared_blks_read > 0
     ORDER BY hit_ratio ASC;
     ```

### Why `SELECT` Queries Cause `shared_blks_dirtied` and `shared_blks_written` in Neon
Your earlier question about why `SELECT` queries result in non-zero `shared_blks_dirtied` and `shared_blks_written` in `pg_stat_statements` is relevant here, as these metrics can affect caching behavior:
- **Hint Bit Updates**: A `SELECT` query may set hint bits on tuples to mark transaction status, dirtying pages (`shared_blks_dirtied`). These pages may later be written to disk (`shared_blks_written`), especially in Neon, where dirty pages are sent to the Pageserver via the Write-Ahead Log (WAL). This can evict other pages from the LFC, reducing cache efficiency.[](https://github.com/neondatabase/neon/blob/main/docs/core_changes.md)
- **HOT Pruning and Visibility Map Updates**: `SELECT` queries can trigger Heap-Only Tuple (HOT) pruning or visibility map updates, dirtying pages and potentially leading to writes. These operations make pages eligible for caching but also increase I/O if not managed.[](https://neon.com/docs/introduction/usage-metrics)
- **Neon’s Architecture**: Neon’s separation of compute and storage means that dirty pages are sent to the Pageserver, which may trigger `shared_blks_written` during `SELECT` queries if the LFC or `shared_buffers` evicts dirty pages.[](https://neon.com/blog/get-page-at-lsn)

To minimize these effects:
- Run `VACUUM` regularly to set hint bits and clean up dead tuples, reducing the need for `SELECT` queries to dirty pages.
- Increase compute size to expand the LFC, reducing evictions and writes.

### Practical Steps to Optimize Caching in Neon
1. **Check Cache Hit Ratio**:
   ```sql
   SELECT file_cache_hits, file_cache_misses,
          (file_cache_hits::float / (file_cache_hits + file_cache_misses) * 100) AS lfc_hit_ratio
   FROM neon_stat_file_cache;
   ```
   If the LFC hit ratio is low, optimize queries or increase compute size.[](https://neon.com/docs/extensions/neon)

2. **Analyze Working Set Size**:
   - Use Neon’s Monitoring dashboard to compare the working set size (unique pages accessed × 8KB) to the LFC size. If the working set exceeds the LFC, scale up the compute.[](https://neon.tech/docs/introduction/monitoring-page)

3. **Prewarm Critical Indexes/Tables**:
   - After a compute restart, use `pg_prewarm` to load key indexes or tables:
     ```sql
     SELECT pg_prewarm('my_table');
     SELECT pg_prewarm('my_index');
     ```
     This ensures they’re in `shared_buffers` or the LFC initially.[](https://neon.com/docs/extensions/pg_search)

4. **Optimize Indexes**:
   - Create compact, targeted indexes for frequent queries. For example, a BM25 index for text searches (via `pg_search`) can reduce page access compared to full-table scans.[](https://neon.com/docs/extensions/pg_search)
   - Remove unused indexes to reduce the working set size, as each index increases the pages that need caching.[](https://neon.tech/blog/performance-tips-for-neon-postgres)

### Limitations and Considerations
- **No Table/Index Pinning**: Unlike some databases that allow pinning specific objects in memory, Neon and PostgreSQL rely on automatic LRU-based caching. You can only influence caching indirectly through access patterns and prewarming.
- **Neon’s Storage Layer**: Since Neon separates compute and storage, data not in the LFC or `shared_buffers` is fetched from the Pageserver, which is slower. Ensuring your working set fits in the LFC is critical.[](https://neon.com/blog/get-page-at-lsn)
- **Compute Restarts**: Neon’s scale-to-zero feature (on the Free plan) clears the LFC and `shared_buffers` when the compute idles. Disable scale-to-zero for production workloads to maintain cache warmth.[](https://neon.com/docs/get-started-with-neon/production-checklist)

### Conclusion
Neon’s LFC cannot be explicitly configured to cache specific indexes or tables. Instead, it automatically caches frequently accessed pages based on PostgreSQL’s LRU policy, extended by Neon’s architecture. To ensure critical tables or indexes are cached, optimize queries to focus on a smaller working set, use `pg_prewarm` to preload data, maintain efficient indexes, and size the compute so the LFC can hold the working set. Monitoring tools like `neon_stat_file_cache` and `pg_stat_statements` help identify caching inefficiencies, while `VACUUM` and `REINDEX` reduce unnecessary page modifications that could disrupt caching.

If you’d like, I can provide a tailored SQL query to monitor LFC usage for specific tables/indexes or search for additional Neon-specific examples on X or the web to dive deeper into real-world use cases. Let me know!