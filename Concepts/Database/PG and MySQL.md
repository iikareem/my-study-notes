---
tags:
  - database
  - mysql
  - postgresql
---

# PostgreSQL vs MySQL: Complete Internal Architecture Guide

## Table of Contents

1. [High-Level Overview](#high-level-overview)
2. [Index Architecture](#index-architecture)
3. [MVCC Implementation](#mvcc-implementation)
4. [Cache Locality and Access Patterns](#cache-locality-and-access-patterns)
5. [Storage Layout](#storage-layout)
6. [Decision Matrix](#decision-matrix)

---

## High-Level Overview

The fundamental difference between PostgreSQL and MySQL comes down to **how they store and manage row visibility**:

- **PostgreSQL**: "Heap + Index" approach — all indexes point to physical locations, dead rows left in place
- **MySQL (InnoDB)**: "Clustered Primary Key" approach — data organized by PK, secondary indexes point to PK values

Neither approach is universally better. Each optimizes for different workload patterns.

---

## Index Architecture

### PostgreSQL: Tuple ID (TID) Based Indexing

**What it stores:**

```
Secondary Index:
┌─────────────────────────────────┐
│ email_value → TID (page, slot)  │
├─────────────────────────────────┤
│ alice@ex    → (page=42, slot=3) │
│ bob@ex      → (page=88, slot=7) │
│ carol@ex    → (page=15, slot=2) │
└─────────────────────────────────┘

Primary Key Index (also a TID pointer):
┌──────────────────────────────────┐
│ pk_value → TID (page, slot)      │
├──────────────────────────────────┤
│ 1 → (page=42, slot=3)            │
│ 2 → (page=88, slot=7)            │
│ 3 → (page=15, slot=2)            │
└──────────────────────────────────┘
```

**How a query works:**

```
SELECT * FROM users WHERE email = 'alice@ex'

1. Binary search in email index → Find TID (page=42, slot=3)
2. Follow TID to heap page 42, slot 3 → Get all columns
   (name, age, email, phone, etc.)

Result: 2 I/O operations
```

**Key characteristics:**

- **What is TID?** A physical pointer: page number + offset within page. Internal, never exposed to users.
- **All indexes are equal** — Primary key is just another index with a uniqueness constraint
- **TID never changes** — Even if you UPDATE a primary key value, the row's physical location stays the same
- **Heap contains all versions** — Both current and dead rows stored in the table

### MySQL (InnoDB): Clustered Primary Key

**What it stores:**

```
Clustered Index (organized by PK):
┌───────────────────────────────────────┐
│ PK=1 → All columns (name, age, email) │
│ PK=2 → All columns                    │
│ PK=3 → All columns                    │
└───────────────────────────────────────┘

Secondary Index (on email):
┌──────────────────────────────────────┐
│ email_value → PK_value               │
├──────────────────────────────────────┤
│ alice@ex    → 1                      │
│ bob@ex      → 2                      │
│ carol@ex    → 3                      │
└──────────────────────────────────────┘

Secondary Index (on age):
┌──────────────────────────────────────┐
│ age_value → PK_value                 │
├──────────────────────────────────────┤
│ 25 → 1                               │
│ 30 → 2                               │
│ 35 → 3                               │
└──────────────────────────────────────┘
```

**Important:** Secondary indexes store ONLY the indexed column + PK value, NOT the entire row.

**How a query works:**

```
SELECT * FROM users WHERE email = 'alice@ex'

1. Binary search in email secondary index → Find PK value = 1
2. Use PK value to search clustered index → Get all columns
   (name, age, email, phone, etc.)

Result: 2 I/O operations (but within same B-tree structure)
```

**Key characteristics:**

- **PK is clustered** — Data physically sorted by primary key
- **Only one clustered index** — The PK is the physical storage order
- **Secondary indexes mandatory** — All secondary indexes must store PK value
- **PK changes are expensive** — Requires rebuilding clustered + all secondary indexes

### Direct Comparison

|Aspect|PostgreSQL|MySQL|
|---|---|---|
|**Primary Key**|Just a unique index pointing to TID|Clustered, stores all data|
|**Secondary Index**|Points to TID|Points to PK value|
|**Can change PK?**|Yes, fast (TID unchanged)|No, very expensive|
|**Primary Key Lookups**|TID → Heap (2 hops)|B-tree traversal (2 hops)|
|**Secondary Index Lookups**|TID → Heap (2 hops)|PK lookup → all data (2 hops)|
|**Index Size**|Smaller (TID only)|Larger (includes PK)|
|**All Indexes Equal?**|Yes|No, PK is special|

---

## MVCC Implementation

Both databases use MVCC (Multi-Version Concurrency Control), but the storage strategy differs fundamentally.

### PostgreSQL: In-Row Versioning

**Version metadata stored in every row:**

```
Row in Heap:
┌──────────────────────────────────────┐
│ xmin=100 (who created me)            │
│ xmax=250 (who deleted me, if any)    │
│ ───────────────────────────────────  │
│ id=1, name=Alice, age=30, salary=50k │
└──────────────────────────────────────┘

Update Timeline:
  Txn 100: INSERT row (xmin=100, xmax=NULL) → visible
  Txn 200: UPDATE age=31 (xmax=200, new row xmin=200)
           OLD row stays with xmin=100, xmax=200 → dead
  Txn 300: UPDATE salary=55k (xmax=300, new row xmin=300)
           Row with xmin=200, xmax=300 → dead
```

**How visibility works:**

```
Transaction 250 queries the row:
  Can I see xmin=100? YES (100 < 250)
  Can I see xmax=200? YES (200 < 250, row is invisible to me)
  → Traverses to previous version via ctid (create transaction ID)
  → Finds row with xmin=200, xmax=250? YES (250 == my txn, dead)
  → Finds row with xmin=100, xmax=200? YES (200 < 250, this is it!)
  → Returns: age=30 (from xmin=100 version)

Transaction 350 queries the row:
  Can I see xmin=300? YES (300 < 350)
  → Returns: age=31, salary=55k (current version)
```

**Problem: Heap Bloat**

Dead rows stay in the heap indefinitely. They waste space and slow down scans.

```
Table after multiple UPDATEs:
┌──────────────────────────────────────┐
│ Row 1: id=1, xmin=100, xmax=200 DEAD │  ← Wasted space
│ Row 2: id=1, xmin=200, xmax=300 DEAD │  ← Wasted space
│ Row 3: id=1, xmin=300, xmax=400 DEAD │  ← Wasted space
│ Row 4: id=1, xmin=400, xmax=NULL ✓   │  ← Current
└──────────────────────────────────────┘

Table Size: 4x space needed for 1 row!
```

**Solution: VACUUM**

- Removes dead rows
- Required maintenance: tune `autovacuum` parameters
- If misconfigured: table bloats, queries slow down
- Full VACUUM: blocks table (take exclusive lock)

### MySQL (InnoDB): Undo Log Based Versioning

**Current row contains one version, old versions in undo log:**

```
Clustered Index (contains CURRENT version only):
┌──────────────────────────────────────────┐
│ PK=1, name=Alice, age=31, salary=55000   │
│ (This is the current, most recent state) │
└──────────────────────────────────────────┘

Undo Log (separate structure):
┌────────────────────────────────────────────┐
│ Version 1 (trx=300):                       │
│   Old: salary=50000                        │
│   Points to: Version 2                     │
├────────────────────────────────────────────┤
│ Version 2 (trx=200):                       │
│   Old: age=30                              │
│   Points to: NULL (or Version 3)           │
└────────────────────────────────────────────┘
```

**How visibility works:**

```
Transaction 250 queries the row:
  1. Reads current version from table: age=31, salary=55000
  2. Checks xmin: created by txn=300
  3. Is 300 visible to me (250)? NO (300 > 250, I started before it)
  4. Follow undo log chain backwards
  5. Find version created by txn=200: age=31, salary=50000
  6. Is 200 visible to me? YES (200 < 250)
  7. Return: age=31, salary=50000

Transaction 350 queries the row:
  1. Reads current version: age=31, salary=55000
  2. Checks xmin: created by txn=300
  3. Is 300 visible to me (350)? YES (300 < 350)
  4. Return: age=31, salary=55000
```

**Advantage: No Bloat**

Table stays clean. Only the current version occupies space.

```
Table after multiple UPDATEs:
┌──────────────────────────────────────┐
│ Row 1: id=1, age=31, salary=55000    │ ← Only current version
└──────────────────────────────────────┘

Table Size: Minimal, no dead rows
```

**Tradeoff: Undo Log Management**

- Old versions stored separately (uses disk/memory)
- Reading old versions requires traversing undo chain (slightly slower)
- Purge thread auto-cleans when no active transactions need old versions
- **No manual VACUUM needed** ✓

### MVCC Comparison

|Aspect|PostgreSQL|MySQL|
|---|---|---|
|**Version Storage**|In heap with row|In separate undo log|
|**Current Version**|Newest row in heap|Only copy in table|
|**Dead Rows**|Stay in heap|Removed immediately|
|**Table Bloat**|Yes, requires VACUUM|No|
|**Reading Old Versions**|Follow ctid pointer|Traverse undo chain|
|**Speed of Old Reads**|Fast (in heap)|Slightly slower (chain)|
|**Maintenance**|Manual VACUUM tuning|Automatic purge|
|**Long-Running Queries**|Prevent cleanup (bloat)|Undo log grows|
|**Best For**|Short queries, low writes|High writes, stable workload|

---

## Cache Locality and Access Patterns

This is where practical performance differences emerge.

### The Cache Hierarchy

```
CPU Cache (ns):
├── L1: 64KB per core, 4-5ns (fastest)
├── L2: 256KB per core, 12ns
└── L3: 8MB shared, 40ns

Main Memory (ns):
└── RAM: 8-200ns (miss = 100+ CPU cycles lost)
```

When you access memory, the CPU loads a **64-byte cache line**. If your next access is in the same cache line, it's instant. If it's in a different memory region, you get a cache miss.

### PostgreSQL: Scattered Memory Access

**Scenario: Secondary Index Lookup**

```
Step 1: Read secondary index
  Email index page (B-tree leaf) loaded into L1 cache
  ┌────────────────────────┐
  │ alice@ex → TID(42,3)   │ ← 64-byte cache line
  └────────────────────────┘
  CPU calculates: follow TID to page 42, slot 3

Step 2: Read heap page
  Completely different memory region
  ┌────────────────────────┐
  │ Page 42 from heap disk │ ← CACHE MISS! (100+ cycles lost)
  │ name=Alice, age=30...  │
  └────────────────────────┘
  
Result: Index cache line useless, fetch new region
```

**Problem: Two separate memory regions**

- Index pages in one part of memory
- Heap pages scattered throughout memory
- No CPU prefetching help (random jumps)
- Every lookup = index cache line + heap cache line = double the work

### MySQL: Locality in B-Tree

**Scenario: Secondary Index Lookup**

```
Step 1: Read secondary index
  Email index page loaded into L1
  ┌──────────────────────────┐
  │ alice@ex → PK=1          │ ← 64-byte cache line
  └──────────────────────────┘
  
Step 2: Read clustered index (same B-tree structure!)
  Traverse the same B-tree to reach PK=1
  B-tree parent/internal nodes may still be in L1/L2
  ┌──────────────────────────┐
  │ PK=1 → all columns       │ ← Possible L2/L3 HIT!
  │ age=30, salary=50000     │
  └──────────────────────────┘
  
Result: B-tree parent nodes reused, fewer cache misses
```

**Advantage: Same logical structure**

- Both traversals use B-tree navigation
- Parent nodes traversed in secondary lookup often still cached
- More efficient memory reuse

### Range Queries vs Random Access

**Range Query: SELECT WHERE age BETWEEN 25 AND 35**

```
PostgreSQL:
  Index traversal → TID list (page=42, 88, 15, 61, ...)
  Follow TIDs → Random heap jumps
  ✗ CPU prefetcher cannot help
  ✗ Cache miss every few rows

MySQL (with age secondary index):
  Index contains age values in order: 25, 28, 30, 32, 35
  These point to PKs sequentially (or close to sequential)
  Clustered index traversal follows ordered PKs
  ✓ Data physically ordered, prefetcher works
  ✓ Sequential disk reads possible
  Performance: 10-100x better for range scans
```

**Random Access: SELECT WHERE id IN (17, 562, 89, ...)**

```
PostgreSQL:
  Index lookups → TIDs scattered
  Heap pages scattered
  ✗ No prefetching, pure random I/O

MySQL:
  Index lookups → PKs (17, 562, 89)
  Clustered index tree traversal
  B-tree parent nodes reused across lookups
  ✓ Better than PostgreSQL (B-tree reuse)
  ✓ Still slower than sequential, but faster than random heap

Result: Both suffer on random access, MySQL suffers less
```

### Cache Locality Summary

|Workload|PostgreSQL|MySQL|Why|
|---|---|---|---|
|**Range scan**|Poor|Excellent|Physical ordering|
|**Random PK lookup**|Fair|Good|B-tree reuse|
|**Random secondary index**|Poor|Fair|B-tree + TID vs B-tree only|
|**Full table scan**|Fair|Good|Heap fragmentation vs clean table|
|**Millions of small lookups**|Okay|Better|Cache line reuse|

**Key insight:** Cache locality advantage only matters when:

1. Access pattern matches the optimization (sequential or B-tree reuse)
2. Latency-sensitive queries (API endpoints, interactive apps)
3. Very high throughput (millions of ops/sec)

For most OLTP workloads, disk I/O dominates, not L1 cache misses.

---

## Storage Layout

### PostgreSQL Physical Layout

```
Database Directory:
├── base/
│   ├── [oid]/
│   │   ├── 16384 (heap file for users table)
│   │   ├── 16384.1 (overflow page for TOASTs)
│   │   └── 16385 (index file for users_email_idx)
│   └── ...
├── pg_wal/ (write-ahead log)
└── pg_subtrans/ (transaction state)

Heap File (users table):
┌────────────────────────────────────────┐
│ Page 1 (8KB)                           │
│ ├─ Tuple 1: xmin=100, xmax=NULL ✓      │
│ ├─ Tuple 2: xmin=150, xmax=200 ✗ DEAD  │
│ ├─ Tuple 3: xmin=200, xmax=NULL ✓      │
│ └─ Tuple 4: xmin=250, xmax=NULL ✓      │
├────────────────────────────────────────┤
│ Page 2 (8KB)                           │
│ ├─ Tuple 5: xmin=100, xmax=300 ✗ DEAD  │
│ ├─ Tuple 6: xmin=300, xmax=NULL ✓      │
│ └─ ...free space (from deletes)...     │
└────────────────────────────────────────┘

Index File (email_idx):
┌────────────────────────────────────┐
│ B-tree structure                   │
├────────────────────────────────────┤
│ Internal page pointing to leaves   │
├────────────────────────────────────┤
│ Leaf page:                         │
│ alice@ex → TID(1,1)                │
│ bob@ex   → TID(1,3)                │
│ carol@ex → TID(2,6)                │
└────────────────────────────────────┘
```

### MySQL (InnoDB) Physical Layout

```
Database Directory:
├── users/
│   ├── users.ibd (tablespace, contains clustered index + data)
│   ├── users.frm (table metadata - deprecated in 8.0+)
│   └── ...
├── #undo_001 (undo log file 1)
├── #undo_002 (undo log file 2)
└── ibdata1 (shared tablespace for internal data)

Clustered Index (PK, in users.ibd):
┌────────────────────────────────────────┐
│ Page 1 (16KB)                          │
│ ├─ Leaf: PK=1, name=Alice, age=31      │
│ ├─ Leaf: PK=2, name=Bob, age=28        │
│ ├─ Leaf: PK=3, name=Carol, age=35      │
│ └─ (All current versions, no dead rows)│
├────────────────────────────────────────┤
│ Page 2 (16KB)                          │
│ ├─ Leaf: PK=4, name=Dave, age=40       │
│ └─ ...                                 │
└────────────────────────────────────────┘

Secondary Index (email, in users.ibd):
┌────────────────────────────────┐
│ Leaf page:                     │
│ alice@ex → PK=1                │
│ bob@ex   → PK=2                │
│ carol@ex → PK=3                │
└────────────────────────────────┘

Undo Log (#undo_001):
┌────────────────────────────────────────┐
│ Version segment for PK=1:              │
│ ├─ Undo record trx=300:                │
│ │  Old: salary=50000                   │
│ │  Link: → next version                │
│ └─ Undo record trx=200:                │
│    Old: age=30                         │
└────────────────────────────────────────┘
```

### Storage Comparison

|Aspect|PostgreSQL|MySQL|
|---|---|---|
|**Data File**|Separate heap file|All in tablespace|
|**Index File**|Separate files|In same tablespace|
|**Dead Rows**|Occupy space|Not present|
|**TOAST Storage**|Separate overflow file|In-page or external|
|**Undo/WAL**|WAL only|Separate undo log files|
|**Table Growth**|Bloats with updates|Stable size|
|**Fragmentation**|Yes, requires VACUUM|Minimal|
|**File Count**|Multiple per table|One per tablespace|

---

## Decision Matrix

Choose based on your actual workload and operational constraints.

### PostgreSQL is Better For:

**When you value:**

1. **Flexibility over performance**
    
    - Primary keys might change (design isn't final)
    - Schema evolution is frequent
    - Rapid prototyping
2. **Complex queries and advanced features**
    
    - Window functions, CTEs, JSON operators
    - Full-text search (built-in)
    - Geometric types, arrays, custom types
    - PostGIS for geospatial data
3. **OLAP/Analytics workloads**
    
    - Full table scans (index choice doesn't matter)
    - Complex analytical queries
    - Tablespaces across different storage
    - Parallel query execution
4. **Long-running queries are rare**
    
    - MVCC cleanup is predictable
    - No undo log bloat
    - Consistent performance
5. **Operations team wants simplicity**
    
    - Less tuning (autovacuum has good defaults)
    - Standard Linux deployment
    - No binary logs to manage

**Example Scenarios:**

- E-commerce (product catalog, frequently updated categories)
- Content management systems (many table changes)
- Data warehouses (analytical queries)
- APIs with complex business logic

### MySQL is Better For:

**When you value:**

1. **Read latency and throughput**
    
    - Sub-millisecond response times
    - Millions of requests/second
    - Latency-sensitive APIs (e-commerce product pages)
2. **Operational simplicity**
    
    - No VACUUM maintenance
    - Automatic cleanup (undo log purge)
    - Proven, standardized deployment
    - Easier DBA onboarding
3. **Range query performance**
    
    - Time-series data (SELECT WHERE date BETWEEN ...)
    - Leaderboards (SELECT WHERE score > X)
    - Range scans dominate your workload
4. **Stable schema**
    
    - Primary key is final and won't change
    - Data model is mature
    - Rare structural changes
5. **High write volume with low complex reads**
    
    - E-commerce transactions (insert orders, updates)
    - Real-time analytics ingestion
    - Write-optimized workloads

**Example Scenarios:**

- High-traffic web applications (millions of users)
- Real-time transactional systems (payments, orders)
- Time-series databases (metrics, logs)
- Session stores, caching layers
- Social media feeds (high write + sequential reads)

### Side-by-Side Workload Comparison

|Workload|PostgreSQL|MySQL|Reason|
|---|---|---|---|
|**REST API (100K req/s)**|Fair|Excellent|MySQL's cache locality|
|**Data warehouse**|Excellent|Fair|Complex queries, full scans|
|**Financial transactions**|Good|Excellent|ACID + performance|
|**User profiles**|Good|Excellent|Random lookups on index|
|**Time-series metrics**|Fair|Excellent|Range query performance|
|**Content management**|Excellent|Fair|Schema flexibility|
|**Real-time bidding**|Fair|Excellent|Sub-ms latency needed|
|**Analytics queries**|Excellent|Fair|Complex joins, window functions|
|**IoT sensor data**|Fair|Excellent|High volume inserts + ranges|
|**Social network graph**|Good|Good|Depends on query patterns|

---

## Detailed Decision Tree

```
START: Choosing PostgreSQL vs MySQL

1. Is your primary key immutable?
   → NO (might change): PostgreSQL (TID-based, PK changes are free)
   → YES (fixed): Continue to 2

2. Do you need advanced SQL features?
   → YES (CTEs, window functions, JSON, geometric): PostgreSQL
   → NO (standard SQL sufficient): Continue to 3

3. Is your workload dominated by:
   → Range queries (BETWEEN, ORDER BY): MySQL
   → Complex joins, aggregations: PostgreSQL
   → Random lookups: MySQL
   → Continue to 4

4. What's your operational priority?
   → Lowest latency (sub-ms): MySQL
   → Feature richness: PostgreSQL
   → No maintenance overhead: MySQL
   → Flexibility: PostgreSQL
   → Continue to 5

5. Scale and volume?
   → Millions of req/sec, latency-sensitive: MySQL
   → Huge data warehouse (TB+): PostgreSQL
   → Standard OLTP (thousands/sec): Either
   → Complex queries priority: PostgreSQL
   → High throughput priority: MySQL
```

---

## Performance Tuning Implications

### PostgreSQL Tuning Focus

```
1. Autovacuum Configuration (MOST IMPORTANT)
   vacuum_scale_factor = 0.05
   autovacuum_naptime = 30s
   autovacuum_vacuum_cost_limit = 2000
   ↓
   Too aggressive: wasted CPU
   Too lenient: table bloat, slow queries

2. Work Memory
   work_mem = 256MB
   ↓
   For hash joins, sorts

3. Shared Buffers
   shared_buffers = 25% of RAM
   ↓
   Buffer pool size

4. Index Choice
   Don't matter much (all point to TID)
   Focus on column selection

Tuning is SQL-first: queries > parameters
```

### MySQL Tuning Focus

```
1. Innodb Buffer Pool (MOST IMPORTANT)
   innodb_buffer_pool_size = 70% of RAM
   ↓
   Larger pool = fewer disk I/O

2. Clustered Index Design
   Choose PK wisely (this defines physical order)
   ↓
   Auto-increment: good
   UUID: bad (random, kills range queries)

3. Secondary Index Strategy
   Index for your access patterns
   ↓
   (a, b) vs (b, a) matters!
   Index prefix affects performance

4. Write Ahead Log
   innodb_flush_log_at_trx_commit = 1 (safe, slower)
   innodb_flush_log_at_trx_commit = 2 (faster, less safe)

Tuning is data-first: index design > parameters
```

---

## Real-World Performance Examples

### Scenario 1: E-Commerce Product Page (10M products, 1M req/sec)

**PostgreSQL:**

```
SELECT * FROM products WHERE id = 12345
  1. Index lookup → TID → Heap: 2ms average
  2. Heavily cached after warmup: sub-1ms
  3. Works well if product_id is stable

Issue: If you change product IDs, all indexes rebuild
```

**MySQL:**

```
SELECT * FROM products WHERE id = 12345
  1. Clustered index lookup: 0.5ms average
  2. B-tree parent nodes cached: sub-0.5ms
  3. PK doesn't change, so stable

Winner: MySQL (2-3x faster on latency-sensitive reads)
```

### Scenario 2: Time-Series Metrics (1M points/day, range queries)

**PostgreSQL:**

```
SELECT * FROM metrics 
WHERE timestamp BETWEEN '2024-01-01' AND '2024-01-02'
  1. Index scan → random TID jumps → heap pages scattered
  2. ~500K random I/Os, 30+ seconds
  3. CPU prefetcher can't help
```

**MySQL:**

```
SELECT * FROM metrics 
WHERE timestamp BETWEEN '2024-01-01' AND '2024-01-02'
  1. Clustered index sequential range
  2. Rows physically ordered by timestamp
  3. CPU prefetch works, sequential I/O
  4. ~5 seconds, 100x better throughput

Winner: MySQL (physical ordering beats TID randomness)
```

### Scenario 3: Analytics Warehouse (10TB+ data, complex queries)

**PostgreSQL:**

```
SELECT product, SUM(revenue) 
FROM orders
JOIN products ON ...
GROUP BY product
HAVING ...
  1. Query planner optimizes complex joins
  2. Window functions, CTEs available
  3. Parallel query execution
  4. Columnar storage options (Citus)

Winner: PostgreSQL (features + optimization)
```

**MySQL:**

```
Same query:
  1. No window functions (until MySQL 8.0, slower)
  2. Parallel query limited
  3. No native columnar storage
  4. JOINs less optimized at scale

Winner: PostgreSQL (feature set and scalability)
```

---

## Summary Table

|Category|PostgreSQL|MySQL|
|---|---|---|
|**Best For**|Complex queries, flexibility|High throughput, latency|
|**Index Model**|TID-based (all equal)|Clustered PK + secondary|
|**MVCC**|In-heap (bloat)|Undo log (clean)|
|**Cache Locality**|TID → Heap (scattered)|B-tree reuse (better)|
|**Range Queries**|TID scatter (poor)|Physical order (excellent)|
|**Random Lookups**|Fair|Better (B-tree reuse)|
|**Maintenance**|VACUUM tuning required|Auto-purge (simpler)|
|**PK Changes**|Free (TID stable)|Expensive (rebuild all)|
|**Features**|Rich (CTEs, JSON, etc)|Basic (standard SQL)|
|**Throughput**|Decent|Better (at latency cost)|
|**Learning Curve**|Steeper (more config)|Shallower (simpler)|
|**Scaling Pattern**|Vertical (bigger box)|Horizontal (sharding)|

---

## Key Takeaways

1. **They both do 2 I/O operations** for secondary index lookups, but the path differs:
    
    - PostgreSQL: Index → TID → Heap (unpredictable)
    - MySQL: Index → PK → Clustered (B-tree reuse)
2. **Cache locality isn't about whether data is cached**, it's about whether the CPU can reuse what's already in cache. MySQL wins on B-tree parent node reuse.
    
3. **MVCC is fundamentally different:**
    
    - PostgreSQL: Versioning in-row, dead rows stay (requires VACUUM)
    - MySQL: Undo log separation, table clean (automatic cleanup)
4. **Physical ordering matters for range queries:**
    
    - MySQL's clustered index = physical sort order = prefetcher helps
    - PostgreSQL's TID scatter = random heap jumps = no prefetching
5. **Neither is universally better.** Choose based on:
    
    - Primary workload type (OLTP vs OLAP)
    - Query complexity needs
    - Operational overhead tolerance
    - Performance sensitivity (latency vs throughput)

---

## References for Further Reading

- PostgreSQL: [Architecture Documentation](https://www.postgresql.org/docs/current/storage-page-layout.html)
- MySQL: [InnoDB Storage Engine](https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html)
- MVCC: [PostgreSQL](https://www.postgresql.org/docs/current/mvcc.html) vs [MySQL](https://dev.mysql.com/doc/refman/8.0/en/innodb-mvcc.html)
- Cache Locality: CPU caches impact on database performance (academic papers)