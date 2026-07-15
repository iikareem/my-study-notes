---
tags:
  - database
  - postgresql
---

The TOAST (The Oversized-Attribute Storage Technique) table in PostgreSQL is a mechanism designed to handle large data values that exceed the database's fixed page size, typically 8 KB, since tuples (rows) cannot span multiple pages. Here's a detailed explanation of what TOAST is, how it manages data with inline pages, and how it stores data, based on PostgreSQL’s documentation and related sources.

### What is a TOAST Table?

TOAST is a storage technique in PostgreSQL that allows the database to efficiently manage large data fields, such as `TEXT`, `VARCHAR`, `BYTEA`, `JSON`, `JSONB`, or other variable-length (`varlena`) data types, which can exceed the page size limit. Since PostgreSQL uses fixed-size pages (commonly 8 KB), large field values that don’t fit within a single page are compressed and/or moved to a separate TOAST table to keep the main table compact and efficient. This process is transparent to users and applications, with minimal impact on backend code. The term TOAST stands for "The Oversized-Attribute Storage Technique," often humorously referred to as "the best thing since sliced bread."[](https://www.postgresql.org/docs/current/storage-toast.html)[](https://medium.com/%40atarax/toast-the-oversized-attribute-storage-technique-of-postgresql-694e868de1b1)[](https://access.crunchydata.com/documentation/postgresql93/latest/storage-toast.html)

Each table with TOAST-able columns (i.e., columns with data types that support TOAST) has an associated TOAST table, identified by an Object Identifier (OID) stored in the main table’s `pg_class.reltoastrelid` entry. If no large data is stored, the TOAST table may remain empty, but it’s created automatically when a table includes TOAST-able columns like `TEXT` or `BYTEA`.[](https://www.postgresql.org/docs/9.5/storage-toast.html)[](https://www.gojek.io/blog/a-toast-from-postgresql)

### How TOAST Manages Data with Inline Pages

PostgreSQL’s storage operates on fixed-size pages (default 8 KB), and TOAST ensures that rows remain within this limit. Here’s how it interacts with inline pages (data stored directly in the main table) and manages large data:

1. **Triggering TOAST**:
   - TOAST activates when a row’s size exceeds the `TOAST_TUPLE_THRESHOLD`, which is typically 2 KB (one-quarter of the default 8 KB page size). The goal is to reduce the row size to fit within the `TOAST_TUPLE_TARGET` (also typically 2 KB, adjustable via `ALTER TABLE ... SET (toast_tuple_target = N)`).[](https://www.postgresql.org/docs/current/storage-toast.html)[](https://docs.arenadata.io/en/ADPG/current/concept/data-model/tables/toast-storage.html)[](https://postgrespro.com/docs/enterprise/15/storage-toast)
   - If a field value is small enough to fit inline (i.e., within the main table’s page without exceeding the threshold), it’s stored directly in the main table, potentially compressed if the storage strategy allows it.[](https://hakibenita.com/sql-medium-text-performance)

2. **Compression**:
   - For TOAST-able columns, PostgreSQL first attempts to compress the data using a fast LZ-family compression algorithm (see `src/backend/utils/adt/pg_lzcompress.c`). If compression reduces the data size to fit within the 2 KB target, it’s stored inline in the main table’s page.[](https://www.postgresql.org/docs/9.5/storage-toast.html)[](https://medium.com/%40atarax/toast-the-oversized-attribute-storage-technique-of-postgresql-694e868de1b1)
   - If compression isn’t sufficient, the data is moved to the TOAST table (out-of-line storage), and a pointer is stored in the main table to reference it.[](https://medium.com/%40Amir_M4A/toast-in-postgresql-storing-large-data-efficiently-0423ce21dd05)

3. **Out-of-Line Storage**:
   - When data cannot be compressed to fit within the target size, TOAST splits it into chunks of up to `TOAST_MAX_CHUNK_SIZE` (approximately 2 KB, chosen so four chunks fit on an 8 KB page). Each chunk is stored as a separate row in the TOAST table with columns:
     - `chunk_id`: An OID identifying the specific TOASTed value.
     - `chunk_seq`: A sequence number indicating the chunk’s order within the value.
     - `chunk_data`: The actual data chunk.[](https://www.postgresql.org/docs/9.5/storage-toast.html)[](https://docs.arenadata.io/en/ADPG/current/concept/data-model/tables/toast-storage.html)
   - A unique index on `chunk_id` and `chunk_seq` ensures fast retrieval of these chunks. The main table stores an 18-byte TOAST pointer, which includes the TOAST table’s OID, the `chunk_id`, the original uncompressed size, and the stored size (if compressed).[](https://access.crunchydata.com/documentation/postgresql93/latest/storage-toast.html)[](https://docs.arenadata.io/en/ADPG/current/concept/data-model/tables/toast-storage.html)

4. **Transparent Retrieval**:
   - When querying data, PostgreSQL automatically retrieves the TOASTed chunks, decompresses them (if compressed), and reconstructs the original value. This process is seamless to the user, though it may add slight performance overhead due to compression/decompression and accessing the TOAST table.[](https://www.cloudthat.com/resources/blog/toast-in-postgresql-for-large-data-management)

### TOAST Storage Strategies

PostgreSQL supports four storage strategies for TOAST-able columns, which determine how data is managed between inline storage and the TOAST table. These can be set using `ALTER TABLE ... SET STORAGE <strategy>`:

1. **PLAIN**:
   - Disables both compression and out-of-line storage. Only used for non-TOAST-able data types (e.g., integers, booleans) or explicitly set for TOAST-able types to prevent TOASTing. Attempts to store oversized data will fail if it exceeds the page size.[](https://www.percona.com/blog/unlocking-the-secrets-of-toast-how-to-optimize-large-column-storage-in-postgresql-for-top-performance-and-scalability/)[](https://wiki.postgresql.org/wiki/TOAST)

2. **EXTENDED** (Default):
   - Allows both compression and out-of-line storage. PostgreSQL first tries to compress the data to fit inline. If that fails, it moves the data to the TOAST table. This is the default for most TOAST-able types like `TEXT`, `VARCHAR`, `BYTEA`, and `JSONB`.[](https://medium.com/quadcode-life/toast-tables-in-postgresql-99e3403ed29b)[](https://www.percona.com/blog/unlocking-the-secrets-of-toast-how-to-optimize-large-column-storage-in-postgresql-for-top-performance-and-scalability/)[](https://medium.com/%40Amir_M4A/toast-in-postgresql-storing-large-data-efficiently-0423ce21dd05)

3. **EXTERNAL**:
   - Allows out-of-line storage but disables compression. Useful for data where compression is ineffective (e.g., already compressed formats like images) or when substring operations on text/bytea need to be faster, at the cost of increased disk usage.[](https://medium.com/quadcode-life/toast-tables-in-postgresql-99e3403ed29b)[](https://www.percona.com/blog/unlocking-the-secrets-of-toast-how-to-optimize-large-column-storage-in-postgresql-for-top-performance-and-scalability/)

4. **MAIN**:
   - Allows compression but prefers inline storage. Out-of-line storage is used only as a last resort when compression cannot reduce the row size enough to fit on a page. Suitable for columns where inline storage is preferred for performance.[](https://www.postgresql.org/docs/9.5/storage-toast.html)[](https://www.percona.com/blog/unlocking-the-secrets-of-toast-how-to-optimize-large-column-storage-in-postgresql-for-top-performance-and-scalability/)[](https://postgrespro.com/docs/enterprise/15/storage-toast)

You can check a column’s storage strategy using a query like:

```sql
SELECT attr.attname, t.typname, 
       CASE WHEN attstorage = 'p' THEN 'plain'
            WHEN attstorage = 'x' THEN 'extended'
            WHEN attstorage = 'e' THEN 'external'
            WHEN attstorage = 'm' THEN 'main'
       END AS attstorage
FROM pg_attribute attr
INNER JOIN pg_type t ON t.oid = attr.atttypid
WHERE attrelid = 'table_name'::regclass;
```

### Data Storage and Management Details

- **TOAST Table Structure**:
  - Each TOAST table is a regular heap table with the same properties as the main table (e.g., inherits its tablespace). It’s named `pg_toast_<OID>`, where `<OID>` is the TOAST table’s identifier.[](https://medium.com/quadcode-life/toast-tables-in-postgresql-99e3403ed29b)[](https://www.gojek.io/blog/a-toast-from-postgresql)
  - Chunks are stored as rows, with each chunk limited to ~2 KB to optimize page usage. A unique index on `chunk_id` and `chunk_seq` ensures efficient retrieval.[](https://docs.arenadata.io/en/ADPG/current/concept/data-model/tables/toast-storage.html)

- **In-Memory Handling**:
  - TOAST also optimizes in-memory handling of large data. In-memory TOAST pointers (e.g., for indirect or expanded data) are used during query execution but are not stored on disk. Before storage, these are expanded to inline `varlena` values or converted to on-disk TOAST pointers if the row is too large.[](https://postgrespro.com/docs/enterprise/15/storage-toast)[](https://postgrespro.com/docs/enterprise/16/storage-toast.html)

- **Performance Considerations**:
  - **Advantages**:
    - **Smaller Main Table**: By moving large values to the TOAST table, the main table remains compact, fitting more rows in the shared buffer cache, which improves query performance.[](https://www.postgresql.org/docs/current/storage-toast.html)[](https://medium.com/%40atarax/toast-the-oversized-attribute-storage-technique-of-postgresql-694e868de1b1)
    - **Efficient Queries**: Queries often use small key values, so the executor primarily accesses the main table, only retrieving TOASTed data when needed (e.g., when selected in the result set).[](https://www.postgresql.org/docs/9.5/storage-toast.html)[](https://medium.com/%40atarax/toast-the-oversized-attribute-storage-technique-of-postgresql-694e868de1b1)
    - **Reduced Sort Overhead**: Smaller main tables reduce the size of sort sets, often allowing sorts to occur in memory.[](https://postgrespro.com/docs/enterprise/15/storage-toast)
  - **Trade-offs**:
    - **Overhead**: Compression/decompression and accessing the TOAST table add slight performance costs, especially for frequent queries on large data.[](https://medium.com/%40atarax/toast-the-oversized-attribute-storage-technique-of-postgresql-694e868de1b1)[](https://medium.com/quadcode-life/toast-tables-in-postgresql-99e3403ed29b)
    - **Disk Usage**: The `EXTERNAL` strategy or uncompressed TOASTed data can increase disk usage.[](https://medium.com/quadcode-life/toast-tables-in-postgresql-99e3403ed29b)[](https://www.percona.com/blog/unlocking-the-secrets-of-toast-how-to-optimize-large-column-storage-in-postgresql-for-top-performance-and-scalability/)
    - **OID Limits**: Each out-of-line TOASTed value consumes an OID, with a global limit of 2^32 (4 billion) per table. Large tables with many TOASTed values may approach this limit, requiring partitioning as a workaround.[](https://wiki.postgresql.org/wiki/TOAST)

- **Updates and Vacuuming**:
  - During `UPDATE` operations, unchanged TOASTed fields are preserved, avoiding additional TOAST overhead unless the TOASTed values change.[](https://www.postgresql.org/docs/9.5/storage-toast.html)[](https://postgrespro.com/docs/enterprise/15/storage-toast)
  - Table bloat (e.g., from frequent updates) can occur if dead tuples aren’t cleaned up. Regular `AUTOVACUUM` is critical to reclaim space, especially if a table grows rapidly due to TOASTed data. A case reported on Stack Overflow showed a table growing from 80 GB to 165 GB in days, reduced to 19 GB after a `VACUUM FULL`, indicating bloat from unvacuumed dead rows.[](https://stackoverflow.com/questions/23120072/how-to-check-if-toast-is-working-on-a-particular-table-in-postgres)

- **Monitoring TOAST Usage**:
  - To check if a table has a TOAST table:
    ```sql
    SELECT relname
    FROM pg_class
    WHERE oid = (SELECT reltoastrelid FROM pg_class WHERE relname = 'table_name');
    ```
  - To analyze TOAST storage for a specific column:
    ```sql
    SELECT id, column_name, pg_column_size(column_name), length(column_name),
           CASE WHEN pg_column_size(column_name) < length(column_name) THEN 'inline-compressed'
                WHEN pg_column_size(column_name) = length(column_name) THEN 'toast'
                ELSE 'inline'
           END AS storage_strategy
    FROM table_name;
    ```
    This query compares `pg_column_size` (stored size) to `length` (uncompressed size) to determine if data is inline, inline-compressed, or TOASTed.[](https://dba.stackexchange.com/questions/301669/detecting-inline-inline-compressed-and-toast-storage)

### Practical Example

Consider a table `blog_posts` with a `TEXT` column for article content:

```sql
CREATE TABLE blog_posts (
    id INTEGER PRIMARY KEY,
    content TEXT
);
```

- If `content` is a 10 KB text string, TOAST will:
  1. Attempt to compress it. If it compresses to <2 KB, it’s stored inline.
  2. If compression doesn’t suffice, it’s split into ~2 KB chunks and stored in the TOAST table `pg_toast_<OID>`.
  3. The main table stores an 18-byte pointer to the TOAST table.[](https://medium.com/%40Amir_M4A/toast-in-postgresql-storing-large-data-efficiently-0423ce21dd05)
- Querying `SELECT content FROM blog_posts` retrieves the chunks, decompresses them (if compressed), and returns the original text transparently.

To force a specific strategy, e.g., `EXTERNAL` for uncompressed out-of-line storage:

```sql
ALTER TABLE blog_posts ALTER COLUMN content SET STORAGE EXTERNAL;
```

### Optimization Tips

- **Adjust `toast_tuple_target`**: Lowering it (minimum 128 bytes) can force more data to be TOASTed, reducing main table size but potentially increasing TOAST table access overhead.[](https://hakibenita.com/sql-medium-text-performance)
- **Choose Appropriate Strategies**: Use `EXTERNAL` for data that doesn’t compress well (e.g., images) or `MAIN` for frequently accessed columns to favor inline storage.[](https://www.percona.com/blog/unlocking-the-secrets-of-toast-how-to-optimize-large-column-storage-in-postgresql-for-top-performance-and-scalability/)
- **Monitor Bloat**: Use `VACUUM` and `ANALYZE` to manage table bloat, especially for tables with frequent updates to large data.[](https://stackoverflow.com/questions/23120072/how-to-check-if-toast-is-working-on-a-particular-table-in-postgres)
- **Consider Alternatives**: For very large objects (e.g., images), store them in a file system or object storage and keep references in the database to avoid TOAST overhead.[](https://www.percona.com/blog/unlocking-the-secrets-of-toast-how-to-optimize-large-column-storage-in-postgresql-for-top-performance-and-scalability/)
- **Partitioning**: For tables with many TOASTed values approaching the OID limit, partition the table to distribute the load.[](https://wiki.postgresql.org/wiki/TOAST)

### Limitations

- **Non-TOAST-able Types**: Fixed-width types (e.g., `INTEGER`, `BOOLEAN`) don’t support TOAST, so many such columns can still cause row size issues.[](https://wiki.postgresql.org/wiki/TOAST)
- **Performance Overhead**: Accessing TOASTed data involves retrieving and possibly decompressing chunks, which can slow queries, especially for large datasets.[](https://www.crunchydata.com/blog/postgres-toast-the-greatest-thing-since-sliced-bread)
- **Structured Data**: TOAST is less effective for structured data like `JSON` or specific workflows (e.g., `BYTEA`), where custom TOAST implementations (TOASTers) in Postgres Pro Enterprise may help.[](https://postgrespro.com/docs/enterprise/16/storage-toast.html)

### Conclusion

TOAST is a powerful, transparent mechanism in PostgreSQL for managing large data by compressing it or moving it to a separate TOAST table, keeping the main table compact and queries efficient. It uses fixed-size chunks (~2 KB) stored in a dedicated TOAST table, referenced by pointers in the main table. By offering four storage strategies (`PLAIN`, `EXTENDED`, `EXTERNAL`, `MAIN`), TOAST provides flexibility to balance performance and storage needs. However, it introduces some overhead, and careful monitoring (e.g., vacuuming, checking TOAST table size) and strategy tuning are essential for optimal performance, especially in data-heavy applications.[](https://www.postgresql.org/docs/9.5/storage-toast.html)[](https://www.postgresql.org/docs/current/storage-toast.html)[](https://medium.com/%40atarax/toast-the-oversized-attribute-storage-technique-of-postgresql-694e868de1b1)

If you have a specific use case or table schema, I can provide tailored advice or queries to analyze TOAST usage further. Let me know!

Yes, your understanding is mostly correct! Let me clarify and confirm the details about how TOAST tables work in PostgreSQL, focusing on the chunking, metadata, and page storage aspects you mentioned, while keeping it concise.

### Key Points on TOAST Table Storage and Chunking

1. **TOAST Table as a Separate Heap**:
   - The TOAST table is a separate heap table in PostgreSQL, stored in the same database (and typically the same tablespace as the main table unless specified otherwise). It’s created automatically for tables with TOAST-able columns (e.g., `TEXT`, `BYTEA`, `JSONB`) and is identified by an Object Identifier (OID) linked to the main table via `pg_class.reltoastrelid`.

2. **Chunking Large Data**:
   - When a value exceeds the `TOAST_TUPLE_THRESHOLD` (~2 KB, or one-quarter of the default 8 KB page size), and compression can’t reduce it enough to fit inline, PostgreSQL moves it to the TOAST table.
   - The value is split into chunks, each up to `TOAST_MAX_CHUNK_SIZE`, which is approximately 2 KB (specifically, 2,000 bytes in PostgreSQL’s source code). This size is chosen so that four chunks fit within a single 8 KB page (4 × 2 KB = 8 KB, accounting for page overhead).

3. **Chunk Metadata**:
   - Each chunk is stored as a row in the TOAST table with three columns:
     - `chunk_id`: An OID uniquely identifying the TOASTed value (shared across all chunks of the same value).
     - `chunk_seq`: An integer indicating the chunk’s order (e.g., 0, 1, 2, …) for reconstructing the original value.
     - `chunk_data`: The actual data chunk (up to ~2 KB).
   - A unique index on `(chunk_id, chunk_seq)` ensures efficient retrieval of chunks in the correct order.
   - The main table stores an 18-byte TOAST pointer for each TOASTed value, containing:
     - The TOAST table’s OID.
     - The `chunk_id` to locate the value’s chunks.
     - The original (uncompressed) size and stored size (if compressed).

4. **Fitting Chunks on Pages**:
   - The ~2 KB chunk size is designed to maximize page usage. An 8 KB page can typically hold four chunks (4 × 2 KB = 8 KB), accounting for minimal page overhead (e.g., page headers). This ensures efficient storage and access.
   - If the data is compressed before chunking (e.g., with the `EXTENDED` storage strategy), each chunk may be smaller than 2 KB, but the maximum chunk size is still capped to fit this model.

5. **Compression and Decompression**:
   - For columns with the `EXTENDED` or `MAIN` storage strategy, PostgreSQL attempts to compress the data using a fast LZ-family algorithm before chunking. If compressed data fits within the ~2 KB target, it’s stored inline in the main table.
   - If moved to the TOAST table, chunks may still be compressed (in `EXTENDED` mode). When retrieved, PostgreSQL decompresses the chunks (if compressed) and reassembles them in the correct order using `chunk_seq`, making the process transparent to the user.

### Example Workflow
Suppose you insert a 10 KB `TEXT` value into a table:
- PostgreSQL tries to compress it (if `EXTENDED` or `MAIN` strategy).
- If compression reduces it to <2 KB, it’s stored inline in the main table’s page.
- If not, it’s split into ~2 KB chunks (e.g., five 2 KB chunks for 10 KB).
- Each chunk is stored as a row in the TOAST table with `chunk_id` (same for all chunks of this value), `chunk_seq` (0 through 4), and `chunk_data`.
- The main table stores a TOAST pointer linking to these chunks.
- An 8 KB page in the TOAST table can hold up to four chunks, so the 10 KB value might span two pages (e.g., four chunks in one page, one in another).

### Clarifications
- **Maximum Chunk Size**: The `TOAST_MAX_CHUNK_SIZE` is ~2 KB (2,000 bytes), not exactly 2 KB, to account for tuple overhead and ensure four chunks fit on an 8 KB page.
- **Four Chunks per Page**: The design assumes four chunks per page to optimize storage, but actual page usage depends on tuple headers, alignment, and other metadata. PostgreSQL’s page structure includes a header (~24 bytes) and tuple overhead, so the effective data per chunk is slightly less than 2 KB.
- **Compression**: Compression occurs before chunking for `EXTENDED` or `MAIN` strategies. The `EXTERNAL` strategy skips compression, storing chunks uncompressed in the TOAST table.

### Verifying TOAST Usage
To check how data is stored:
```sql
SELECT id, pg_column_size(large_column) AS stored_size, length(large_column) AS uncompressed_size
FROM your_table;
```
- If `stored_size` < `uncompressed_size`, the data is compressed (inline or in TOAST).
- If `pg_column_size` indicates a small size (e.g., ~18 bytes), it’s a TOAST pointer, meaning the data is in the TOAST table.

To confirm the TOAST table’s existence and size:
```sql
SELECT relname, pg_total_relation_size(reltoastrelid) AS toast_size
FROM pg_class
WHERE relname = 'your_table';
```

### Summary
The TOAST table is a separate heap table where large values are split into ~2 KB chunks, each stored as a row with `chunk_id`, `chunk_seq`, and `chunk_data`. Metadata (via the unique index and TOAST pointer) ensures chunks can be retrieved and reassembled. The chunk size is designed so four chunks fit on an 8 KB page, optimizing storage. Compression (if enabled) occurs before chunking, and decompression happens transparently on retrieval. This keeps the main table compact while handling large data efficiently.

If you need specific examples, queries, or performance tips for your use case, let me know!

++ im summary why postgres not enable the reading the row from multiple pages ? why that is hard ??

In PostgreSQL, rows (tuples) are confined to a single page (default 8 KB) due to design choices that prioritize simplicity, performance, and reliability. Enabling rows to span multiple pages is technically challenging and avoided for the following reasons:

1. **Storage and Access Efficiency**:
   - **Single-Page Access**: Storing a row within one page ensures fast, atomic disk I/O operations. Reading a row from multiple pages would require multiple disk seeks, increasing latency and complexity, especially for random access patterns.
   - **Buffer Management**: PostgreSQL’s buffer manager handles fixed-size pages (8 KB). Spanning rows across pages would complicate caching, locking, and concurrency control, as pages could be independently modified or evicted.

2. **Concurrency and Locking**:
   - **MVCC Complexity**: PostgreSQL uses Multi-Version Concurrency Control (MVCC), where each row version (tuple) is stored atomically. Spanning rows across pages would require locking multiple pages for a single row update, increasing contention and complicating transaction isolation.
   - **Deadlock Risk**: Multi-page rows could lead to more frequent deadlocks if different transactions access parts of the same row on different pages.

3. **TOAST as a Solution**:
   - **Handling Large Data**: Instead of multi-page rows, PostgreSQL uses TOAST to store large field values in a separate TOAST table, split into ~2 KB chunks. This keeps the main table’s rows compact (fitting within one page) while allowing large data to be stored and retrieved efficiently.
   - **Transparent Access**: TOAST pointers in the main table link to chunked data, making the process transparent to users while avoiding the need to redesign the core storage engine for multi-page rows.

4. **Implementation Complexity**:
   - **Indexing Challenges**: Indexes (e.g., B-trees) reference tuples by their page location and offset (CTID). Spanning rows across pages would require a new addressing scheme, complicating index maintenance and lookups.
   - **Crash Recovery**: PostgreSQL’s Write-Ahead Logging (WAL) assumes page-level atomicity. Multi-page rows would require logging changes across multiple pages, increasing recovery complexity and potential for inconsistency.
   - **Fragmentation**: Managing row fragments across pages could lead to fragmentation, complicating space allocation and vacuuming.

5. **Performance Trade-offs**:
   - **Sequential Reads**: While sequential reads of multi-page rows might be feasible, random access (common in OLTP workloads) would suffer due to scattered page reads.
   - **Vacuum and Maintenance**: Operations like `VACUUM` and `ANALYZE` are page-based. Multi-page rows would require tracking and cleaning fragments across pages, slowing these processes.

### Why It’s Hard
Supporting multi-page rows would require a fundamental overhaul of PostgreSQL’s storage engine, including:
- Redesigning tuple storage and page layout to track row fragments.
- Modifying the buffer manager, index structures, and MVCC to handle multi-page tuples.
- Updating WAL and crash recovery to ensure atomicity across pages.
- Handling increased I/O and locking overhead, which could degrade performance for common workloads.

### Summary
PostgreSQL avoids multi-page rows to maintain simplicity, performance, and reliability in its page-based storage model. The TOAST mechanism efficiently handles large data by storing it in separate tables as chunks, avoiding the need for complex multi-page row management. This design choice balances the trade-offs, prioritizing fast access and robust concurrency for typical use cases.

If you want to dive deeper into a specific aspect (e.g., TOAST vs. multi-page rows or performance implications), let me know!