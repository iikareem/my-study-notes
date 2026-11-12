# Databases Deep Dive: Relational vs Non-Relational, Horizontal Scaling, LSM Trees vs B-Trees

---

## Table of Contents

1. [The Fundamental Difference in Data Models](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#1-the-fundamental-difference-in-data-models)
2. [Why Relational Databases Are Hard to Scale Horizontally](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#2-why-relational-databases-are-hard-to-scale-horizontally)
3. [Why Non-Relational Databases Scale Horizontally with Ease](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#3-why-non-relational-databases-scale-horizontally-with-ease)
4. [Storage Engines: The Engine Under the Hood](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#4-storage-engines-the-engine-under-the-hood)
5. [B-Trees: How They Work and Why Reads Are Fast](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#5-b-trees-how-they-work-and-why-reads-are-fast)
6. [LSM Trees: How They Work and Why Writes Are Fast](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#6-lsm-trees-how-they-work-and-why-writes-are-fast)
7. [Head-to-Head: B-Tree vs LSM-Tree](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#7-head-to-head-b-tree-vs-lsm-tree)
8. [Putting It All Together: Choosing the Right Tool](https://claude.ai/chat/c39689d4-1688-4b81-8da0-53c03455c80e#8-putting-it-all-together-choosing-the-right-tool)

---

## 1. The Fundamental Difference in Data Models

Before we talk about scaling, we need to understand **why** the data model itself creates radically different constraints.

### The Relational Model

A relational database stores data in **tables** (relations) made of rows and columns. The key principle is **normalization**: you decompose data into its smallest logical units to eliminate redundancy.

Consider a blog application:

```
users table
┌────┬──────────┬───────────────────────┐
│ id │ name     │ email                 │
├────┼──────────┼───────────────────────┤
│  1 │ Alice    │ alice@example.com     │
│  2 │ Bob      │ bob@example.com       │
└────┴──────────┴───────────────────────┘

posts table
┌────┬─────────┬─────────────────────────────┐
│ id │ user_id │ title                       │
├────┼─────────┼─────────────────────────────┤
│  1 │       1 │ Why I Love Databases        │
│  2 │       1 │ B-Trees Explained           │
│  3 │       2 │ My Journey with NoSQL       │
└────┴─────────┴─────────────────────────────┘

comments table
┌────┬─────────┬─────────┬────────────────────┐
│ id │ post_id │ user_id │ body               │
├────┼─────────┼─────────┼────────────────────┤
│  1 │       1 │       2 │ Great article!     │
│  2 │       2 │       2 │ Very clear.        │
└────┴─────────┴─────────┴────────────────────┘
```

To get Alice's posts with all their comments and the commenter's name, you **join** across 3 tables:

```sql
SELECT u.name, p.title, c.body, cu.name AS commenter
FROM users u
JOIN posts p ON p.user_id = u.id
JOIN comments c ON c.post_id = p.id
JOIN users cu ON cu.id = c.user_id
WHERE u.id = 1;
```

The data is **spread across multiple locations**. The database must gather, correlate, and merge it at query time.

### The Non-Relational Model (Document)

A document database stores data as **self-contained documents** (usually JSON/BSON). The same blog data might look like:

```json
{
  "_id": "user:1",
  "name": "Alice",
  "email": "alice@example.com",
  "posts": [
    {
      "id": "post:1",
      "title": "Why I Love Databases",
      "comments": [
        { "author": "Bob", "body": "Great article!" }
      ]
    },
    {
      "id": "post:2",
      "title": "B-Trees Explained",
      "comments": [
        { "author": "Bob", "body": "Very clear." }
      ]
    }
  ]
}
```

Everything about Alice is **in one place**. Reading her profile, posts, and comments is a single document lookup — no joins needed.

### The Core Insight

> **Relational = data spread across many rows and tables, assembled at query time.** **Non-relational = data pre-assembled into units that map to how the application accesses it.**

This difference is not cosmetic. It has profound implications for distributed systems.

---

## 2. Why Relational Databases Are Hard to Scale Horizontally

Horizontal scaling means adding more machines (nodes) and distributing the data across them — also called **sharding** or **partitioning**.

### The Join Problem Across Nodes

When you split a relational database across multiple nodes, the fundamental contract of joins breaks down.

Imagine you shard your `users` table across Node A and Node B:

```
Node A: users with id 1–500,000
Node B: users with id 500,001–1,000,000
```

Now you shard your `posts` table:

```
Node C: posts with id 1–2,000,000
Node D: posts with id 2,000,001–4,000,000
```

To execute this query:

```sql
SELECT u.name, p.title
FROM users u
JOIN posts p ON p.user_id = u.id
WHERE u.id = 42;
```

The database must:

1. Query Node A for user 42.
2. Query Node C **and** Node D for posts where `user_id = 42`.
3. Merge and correlate the results from different machines over the network.

This is called a **cross-shard join** or **distributed join**. The network round-trips, data shuffling, and coordination overhead make this extremely expensive. The more normalized your schema (more tables), the worse it gets.

### The ACID Transaction Problem

Relational databases guarantee **ACID** properties:

- **Atomicity**: A transaction either fully succeeds or fully fails.
- **Consistency**: The database is always in a valid state.
- **Isolation**: Concurrent transactions don't interfere.
- **Durability**: Committed data survives crashes.

In a single machine, these are enforced by locking, write-ahead logs, and local coordination mechanisms. When data spans multiple nodes, ACID becomes a distributed systems problem.

Consider a bank transfer: debit account A on Node 1, credit account B on Node 2. If Node 2 crashes after Node 1's debit, you've lost money. You need a **distributed transaction protocol** like **two-phase commit (2PC)**:

```
Phase 1 (Prepare):
  Coordinator → Node 1: "Can you commit this debit?"
  Coordinator → Node 2: "Can you commit this credit?"
  Node 1 → Coordinator: "Yes (VOTE-COMMIT)"
  Node 2 → Coordinator: "Yes (VOTE-COMMIT)"

Phase 2 (Commit):
  Coordinator → Node 1: "Commit!"
  Coordinator → Node 2: "Commit!"
```

This works, but:

- It introduces a **coordinator as a single point of failure**.
- All participating nodes are **locked** during the protocol — blocking other transactions.
- Network latency multiplies: a transaction that takes 1ms locally may take 20ms across nodes.
- It fails in complex ways: if the coordinator crashes after Phase 1, nodes are stuck waiting ("in-doubt transactions").

Modern solutions like **Google Spanner** use atomic clocks and Paxos to achieve distributed ACID, but the engineering complexity and hardware cost are enormous.

### The Schema Migration Problem

In a relational database, schema changes (adding a column, changing a type) require **ALTER TABLE** operations. In a distributed setting:

- The operation must be coordinated across all nodes.
- Many ALTER TABLE operations lock the table or require table rewrites.
- A schema change on a billion-row table distributed across 50 shards can take hours and block operations.

This creates immense operational friction at scale.

### Summary: Why Relational Struggles to Scale Out

|Challenge|Root Cause|
|---|---|
|Cross-shard joins|Data is normalized across many tables; assembly requires cross-node communication|
|Distributed transactions|ACID guarantees require coordination protocols that introduce latency and blocking|
|Schema rigidity|Fixed schema requires synchronized, often locking, migrations across all nodes|
|Foreign key enforcement|Referential integrity checks may span shards, requiring distributed locks|

---

## 3. Why Non-Relational Databases Scale Horizontally with Ease

### Self-Contained Documents = Shard-Local Operations

Because a document contains all the data the application needs in one read, it can be **stored on exactly one node** and retrieved from that node alone — no cross-node joins needed.

When you shard a document store, you pick a **shard key** (e.g., `user_id`). All documents with the same shard key live on the same node:

```
Node A: all documents where hash(user_id) % 3 == 0
Node B: all documents where hash(user_id) % 3 == 1
Node C: all documents where hash(user_id) % 3 == 2
```

Reading Alice's full profile (with all her posts and comments embedded) is always a **single-node operation**:

```
Client → Router: "Get document user:1"
Router → Node A: "Fetch user:1"
Node A → Client: {Alice's entire document}
```

No inter-node communication at all for the common read path.

### The CAP Trade-off: Embracing Eventual Consistency

The **CAP theorem** states that a distributed system can guarantee at most two of three properties:

- **Consistency** (C): Every read returns the most recent write.
- **Availability** (A): Every request receives a response (even if it might be stale).
- **Partition tolerance** (P): The system continues operating despite network partitions.

Network partitions are unavoidable in real distributed systems, so the real choice is **C vs A**:

- **Relational databases** generally choose **CP**: they sacrifice availability to maintain strict consistency (via distributed locking, 2PC, etc.).
- **Many NoSQL databases** choose **AP**: they sacrifice strict consistency for availability, embracing **eventual consistency**.

In an eventually consistent system, writes are applied to one node and **propagated asynchronously** to replicas. This means:

- Writes are fast: acknowledge as soon as the primary node writes.
- Reads may return slightly stale data briefly.
- No global locking needed → massive throughput improvements.

### Schema Flexibility = No Migration Overhead

Document databases are **schema-on-read**: the schema is implicit in the application code, not enforced by the database. You can add new fields to documents without running a migration. Different documents in the same collection can have different shapes.

This means:

- Adding a new feature doesn't require an ALTER TABLE across 50 shards.
- Documents evolve independently.
- You can have V1 and V2 of a document structure coexist; the application handles both.

### Wide-Column Stores: Taking It Further

Databases like **Cassandra** and **HBase** use a **wide-column model** — essentially a distributed, sorted map of maps. The data model is designed from the ground up around a **partition key** that determines which node owns the data.

In Cassandra, you design your schema around your **query patterns**, not around normalization. You may deliberately **denormalize** and store the same data in multiple tables optimized for different access patterns:

```
-- Table for "get all posts by user"
CREATE TABLE posts_by_user (
  user_id UUID,
  post_id UUID,
  title TEXT,
  PRIMARY KEY (user_id, post_id)
);

-- Table for "get all comments by post"
CREATE TABLE comments_by_post (
  post_id UUID,
  comment_id UUID,
  author TEXT,
  body TEXT,
  PRIMARY KEY (post_id, comment_id)
);
```

The trade-off: you have duplicate data, but every single query hits exactly one partition on exactly one node. This is **linear horizontal scalability**: add more nodes, and you get proportionally more throughput with no cross-node coordination.

---

## 4. Storage Engines: The Engine Under the Hood

Both relational and non-relational databases sit on top of a **storage engine** — the component responsible for actually reading and writing bytes to disk. The two dominant data structures used in storage engines are **B-Trees** and **LSM Trees (Log-Structured Merge Trees)**.

Understanding these is crucial because they determine the fundamental performance characteristics of the database, regardless of whether it's SQL or NoSQL.

### The Core Problem: Disk I/O

Disks (even SSDs) are many orders of magnitude slower than RAM:

|Operation|Latency|
|---|---|
|L1 Cache|~0.5 ns|
|RAM|~100 ns|
|SSD read|~100 µs (100,000 ns)|
|HDD seek|~10 ms (10,000,000 ns)|

Every design choice in B-Trees and LSM trees is fundamentally about **minimizing disk I/O**.

---

## 5. B-Trees: How They Work and Why Reads Are Fast

### The Structure

A **B-Tree** (Balanced Tree) is a self-balancing, sorted tree structure. It is the dominant data structure in traditional relational databases (PostgreSQL, MySQL InnoDB, Oracle, SQL Server).

The key properties:

- Nodes are fixed-size **pages** (typically 4KB or 16KB), matching disk block sizes.
- Each node can have many children (high **fanout**), typically hundreds.
- The tree is always balanced: all leaves are at the same depth.
- Data is **sorted** by key across the entire tree.

```
                    [root]
                   [50 | 75]
                  /    |    \
           [10|30]  [60|70]  [80|90]
           /  |  \  / | \    /  | \
         [5] [20][35][55][65][78][85][95]
               ↑ leaf nodes with actual data
```

Each internal node contains **separator keys** that route the search to the correct child. Leaf nodes contain the actual key-value pairs (or pointers to row data).

### Read Path: How a B-Tree Lookup Works

To find the record with key `65`:

1. Load the root page from disk: `[50 | 75]` → 65 is between 50 and 75 → go to middle child.
2. Load the middle child from disk: `[60 | 70]` → 65 is between 60 and 70 → go to middle child.
3. Load the leaf from disk: `[65]` → found! Return the record.

For a tree with **n keys**, the depth is **O(log n)**. With a fanout of 500 and 4KB pages:

- Depth 1: 500 keys
- Depth 2: 250,000 keys
- Depth 3: 125,000,000 keys
- Depth 4: 62,500,000,000 keys (62 billion keys, only 4 disk reads!)

**This is why B-Tree reads are fast**: a point lookup requires only O(log n) page reads from disk — typically 3–5 reads even for tables with billions of rows.

### Write Path: How a B-Tree Update Works

Writing to a B-Tree is an **in-place update**:

1. Find the page where the key belongs (O(log n) reads).
2. Modify the page in memory.
3. Write the modified page back to disk.

```
Before write (key 62):
Leaf: [60 | 65]
      ↓ find the right page, modify it in place
After write:
Leaf: [60 | 62 | 65]
```

If the page overflows (too many keys), it must be **split**:

```
Leaf overflows: [60 | 62 | 65 | 67 | 68] → too many keys
                ↓ split into two pages + update parent
Left:  [60 | 62]
Right: [65 | 67 | 68]
Parent: [... | 65 | ...]  ← new separator key added
```

Splits may cascade up the tree (split propagates to parent, which may also split).

### Write-Ahead Log (WAL): Crash Safety

Before modifying any B-Tree page, the database writes the change to a **Write-Ahead Log (WAL)** — a sequential, append-only log on disk:

```
WAL:
[LSN=1001] UPDATE page 4042, offset 36, old=X, new=Y
[LSN=1002] UPDATE page 4042, offset 48, old=A, new=B
[LSN=1003] SPLIT page 4042 → pages 4042, 8891
...
```

Only after the WAL entry is durably written to disk does the engine modify the actual B-Tree pages. On crash, the WAL is replayed to restore consistency.

This means **every write generates two I/Os**: one sequential WAL write + one random page write. The random I/O is the bottleneck.

### Why B-Tree Writes Are Slower

The fundamental problem: **random I/O**.

When you update a record, you must:

1. Read the correct page into memory (random disk seek).
2. Modify it.
3. Write it back to the same location (random disk write).

Writes are scattered across the disk — pages 4, 17, 892, 3001 — wherever the B-Tree has placed them. SSDs are much better at this than HDDs (no physical seek), but random I/O is still significantly slower than sequential I/O.

Additionally, **write amplification**: updating a single key may require writing several pages due to cascading splits, WAL entries, and page writes.

---

## 6. LSM Trees: How They Work and Why Writes Are Fast

### The Core Insight: Never Rewrite, Only Append

LSM Trees (Log-Structured Merge Trees) take a radically different philosophy: **never update in place, only append**. All writes go to fast sequential structures, and are reorganized into efficient read structures in the background.

LSM Trees are used by: **LevelDB**, **RocksDB**, **Cassandra**, **HBase**, **ScyllaDB**, **InfluxDB**, and increasingly in modern databases.

### The Architecture

An LSM Tree has several components:

```
┌─────────────────────────────────────────────────────────┐
│                     MEMORY (RAM)                        │
│                                                         │
│   ┌─────────────────┐    ┌─────────────────┐           │
│   │   MemTable      │    │  Immutable       │           │
│   │  (active write  │───→│  MemTable        │           │
│   │   buffer)       │    │  (being flushed) │           │
│   └─────────────────┘    └────────┬────────┘           │
└────────────────────────────────────┼────────────────────┘
                                     │ flush (sequential write)
                                     ▼
┌─────────────────────────────────────────────────────────┐
│                      DISK                               │
│                                                         │
│   Level 0:  [SSTable 1]  [SSTable 2]  [SSTable 3]      │
│                                                         │
│   Level 1:  [───── SSTable 4 ─────]  [─ SSTable 5 ─]  │
│                                                         │
│   Level 2:  [────────────── SSTable 6 ──────────────]  │
│                                                         │
│   Level 3:  [───────────────── SSTable 7 ───────────]  │
└─────────────────────────────────────────────────────────┘
```

### Component 1: MemTable

The **MemTable** is an in-memory, sorted data structure (usually a red-black tree or skip list). All writes go here:

```
Write "apple=1":   MemTable: {apple: 1}
Write "cherry=2":  MemTable: {apple: 1, cherry: 2}
Write "banana=3":  MemTable: {apple: 1, banana: 3, cherry: 2}
Write "apple=5":   MemTable: {apple: 5, banana: 3, cherry: 2}  ← updated in place in memory
```

Writes are: memory operation (nanoseconds) + WAL append (sequential disk write). **No random disk I/O for writes.**

Also write to a **Write-Ahead Log (WAL)** sequentially for crash recovery:

```
WAL (sequential):
apple=1
cherry=2
banana=3
apple=5
```

### Component 2: SSTable (Sorted String Table)

When the MemTable reaches a size threshold (e.g., 64MB), it is **flushed to disk** as an **SSTable** — an immutable, sorted file:

```
MemTable (sorted):          SSTable on disk:
{                           [apple=5 | banana=3 | cherry=2]
  apple: 5,                  ↑ written sequentially, once, never modified
  banana: 3,
  cherry: 2
}
```

This flush is a single **sequential disk write** — the fastest possible disk operation.

An SSTable also contains an **index block** and a **bloom filter**:

```
SSTable structure:
┌──────────────────┐
│   Data Block 1   │  apple=5, apricot=7, avocado=2
├──────────────────┤
│   Data Block 2   │  banana=3, blueberry=9
├──────────────────┤
│   Data Block 3   │  cherry=2, cranberry=8
├──────────────────┤
│   Index Block    │  apple→Block1, banana→Block2, cherry→Block3
├──────────────────┤
│   Bloom Filter   │  probabilistic: "does key X exist in this SSTable?"
└──────────────────┘
```

### Component 3: Compaction

Over time, many SSTables accumulate. The same key may appear in multiple SSTables with different versions:

```
SSTable 1 (old): apple=1, banana=3
SSTable 2 (new): apple=5, cherry=2
```

The **compaction process** runs in the background: it merges SSTables, keeps only the latest version of each key, and discards deleted keys (marked with **tombstones**):

```
Compaction of SSTable 1 + SSTable 2:
Result: apple=5 (from SSTable 2, newer), banana=3, cherry=2
```

After compaction, the old SSTables are deleted and replaced by the merged, compact file.

**Compaction strategies:**

- **Size-Tiered Compaction (STCS)**: Merge SSTables of similar size. Fewer write amplification, but higher space overhead and worse read amplification (used by Cassandra by default).
- **Leveled Compaction (LCS)**: SSTables are organized into levels with size limits. Level N+1 is ~10x larger than Level N. Compaction merges data down from upper levels. Better read performance, higher write amplification (used by LevelDB, RocksDB).
- **FIFO Compaction**: Discard oldest SSTables when size limit reached. Only useful for time-series data.

### Read Path in an LSM Tree

To read a key, you search in this order:

1. **MemTable** (in memory, latest data): O(log n) in the sorted structure.
2. **Immutable MemTable** (if being flushed): O(log n).
3. **Level 0 SSTables** (newest on disk, may overlap): check each one.
4. **Level 1, 2, 3... SSTables** (older, non-overlapping): binary search in index.

For each SSTable, before loading data blocks:

- Check the **Bloom filter**: if the filter says "NO", skip this SSTable entirely (0 disk reads).
- If "maybe YES", load the index block, find the right data block, read it.

**This is why LSM reads can be slower than B-Trees**: in the worst case (key not found), you check every SSTable at every level. Even with bloom filters, a read for an existing key may require multiple disk reads from different levels.

### Why LSM Writes Are Faster: Sequential I/O

The entire write path is sequential:

```
Write request → RAM write (MemTable, nanoseconds)
             → WAL append (sequential disk write, microseconds)
             → Done (acknowledged to client)

Background: MemTable flush → sequential disk write (SSTable)
Background: Compaction → sequential read old SSTables + sequential write new SSTable
```

**No random I/O in the hot write path at all.** The database only ever appends — to the WAL and to SSTables. This is why LSM trees can sustain extremely high write throughput.

On an HDD, sequential writes can be 100x faster than random writes. On SSDs, the difference is smaller but still significant (and sequential writes cause less SSD wear).

---

## 7. Head-to-Head: B-Tree vs LSM-Tree

### Write Amplification

**Write amplification** = total bytes written to disk / bytes in the original write.

**B-Tree write amplification:**

- WAL write + page write (at minimum).
- Splits cascade: may write multiple pages.
- Buffer pool eviction: dirty pages flushed periodically.
- Typical write amplification: **3–10x**.

**LSM write amplification:**

- WAL write + MemTable (RAM) + eventual SSTable flush.
- Compaction: data is rewritten across levels multiple times.
- Leveled compaction: a key might be rewritten 10–30 times before reaching the bottom level.
- Typical write amplification: **10–30x** (higher than B-Trees in theory, but all sequential).

> **Paradox**: LSM trees have _higher write amplification_ than B-Trees, but _higher write throughput_. Why? Because sequential writes are so much faster than random writes that the net effect is higher throughput even with more bytes written.

### Read Amplification

**Read amplification** = number of disk reads to satisfy a single read request.

**B-Tree:**

- Point lookup: O(log n) page reads — typically **3–5 reads** regardless of data size.
- Range scan: follow pointers from the leftmost key, sequentially traverse leaf pages. Efficient because leaf pages are often contiguous.
- **Read amplification: 3–5 (excellent)**.

**LSM Tree:**

- Point lookup (worst case, key not in MemTable): must check bloom filter + potentially one data block per level.
- With L levels and no bloom filter: **L disk reads**.
- With bloom filters (99% accurate): bloom filter eliminates most SSTables, but the key may still require several reads.
- Range scan: must merge results from multiple SSTables (merge sort), which is expensive.
- **Read amplification: 5–20+ (significantly worse than B-Tree)**.

### Space Amplification

**Space amplification** = actual disk usage / size of logical data.

**B-Tree:**

- Pages are partially full on average (~70% full after insertions/deletions).
- Space amplification: **~1.3x**.

**LSM Tree:**

- Multiple versions of the same key exist across levels until compaction.
- SSTables contain tombstones for deleted keys until compacted away.
- Space amplification: **~1.5–3x** depending on compaction strategy.

### Side-by-Side Comparison

|Property|B-Tree|LSM Tree|
|---|---|---|
|**Point Read**|★★★★★ Excellent|★★★ Good (with bloom filters)|
|**Range Scan**|★★★★★ Excellent|★★★ Moderate (multi-SSTable merge)|
|**Write Throughput**|★★★ Moderate|★★★★★ Excellent|
|**Write Latency**|★★★ Moderate|★★★★★ Excellent|
|**Write Amplification**|Low (3–10x)|High (10–30x)|
|**Read Amplification**|Low (3–5x)|Higher (5–20x)|
|**Space Amplification**|Low (~1.3x)|Moderate (~2x)|
|**Crash Recovery**|WAL replay|WAL replay (fast, since WAL is small)|
|**Background I/O**|Low|High (compaction)|
|**Compaction Pauses**|None|Yes (can cause latency spikes)|
|**Best For**|Read-heavy OLTP|Write-heavy workloads|

### Real-World Database Choices

|Database|Storage Engine|Model|
|---|---|---|
|PostgreSQL|B-Tree (heap + index)|Relational|
|MySQL InnoDB|B-Tree (clustered)|Relational|
|SQLite|B-Tree|Relational (embedded)|
|MongoDB (WiredTiger)|B-Tree + LSM hybrid|Document|
|Cassandra|LSM (custom)|Wide-column|
|RocksDB|LSM (LevelDB fork)|KV Store (embedded)|
|LevelDB|LSM|KV Store (embedded)|
|HBase|LSM (LSM on HDFS)|Wide-column|
|InfluxDB|LSM-based (TSM)|Time-series|
|ScyllaDB|LSM (like Cassandra)|Wide-column|

---

## 8. Putting It All Together: Choosing the Right Tool

### Decision Framework

```
Is your primary bottleneck writes?
  ├── YES → Consider LSM-based store (Cassandra, RocksDB, ScyllaDB)
  └── NO → Continue...

Do you have complex join requirements?
  ├── YES → Relational (PostgreSQL, MySQL)
  └── NO → Continue...

Is your data naturally hierarchical/nested per entity?
  ├── YES → Document (MongoDB, CouchDB)
  └── NO → Continue...

Do you need massive horizontal write throughput with simple access patterns?
  ├── YES → Wide-column (Cassandra, HBase)
  └── NO → Relational is often fine

Do you need time-series analytics at scale?
  └── Time-series DB (InfluxDB, TimescaleDB)
```

### The Deeper Truth

The real choice isn't just "SQL vs NoSQL" — it's about **access patterns, data locality, and consistency requirements**:

1. **If your reads require assembling data from many entities** → denormalize into documents or wide-column rows. Accept data duplication in exchange for read locality.
    
2. **If your writes are the bottleneck** → choose an LSM-based storage engine. All sequential writes, background compaction handles organization.
    
3. **If you need strong ACID across multiple entities** → relational with a B-Tree engine is still king. Accept the scaling limitations or use techniques like logical sharding with careful schema design.
    
4. **If you need both massive writes and strong consistency** → this is the hardest problem in distributed systems. Google Spanner and CockroachDB attempt it at great engineering cost.
    

### The Spectrum of Trade-offs

```
Strong Consistency ◄──────────────────────────────────► High Availability
                        CAP Trade-off Space
PostgreSQL/MySQL        CockroachDB         Cassandra/DynamoDB
(single node)           (distributed SQL)   (eventually consistent)


Fast Reads ◄────────────────────────────────────────────► Fast Writes
               Storage Engine Trade-off Space
B-Tree (PostgreSQL)                         LSM Tree (RocksDB/Cassandra)
  - 3-5 reads per lookup                     - 1 write = sequential append
  - Random I/O for writes                    - Reads require bloom filters + multi-level search


Normalized ◄────────────────────────────────────────────► Denormalized
                  Data Model Trade-off Space
Relational (3NF)              Hybrid              Document/Wide-column
  - No data duplication         (materialized       - Data duplication
  - Complex joins               views)              - No join needed
  - Hard to shard                                   - Easy to shard
```

### A Final Mental Model

Think of it this way:

**B-Tree** is like a perfectly organized library with a card catalog. Finding any book is fast (3–5 catalog lookups). But adding a new book and re-sorting the catalog is slow because you're rearranging physical cards in place.

**LSM Tree** is like a library where new books are dropped in a pile at the front. Finding a specific book may require checking several piles (slow read). But adding new books is instant — just drop it on the pile. A librarian (compaction) periodically reorganizes piles in the background.

**Relational data model** is like a highly organized set of reference books that cross-cite each other. To get the full picture of a topic, you must look up multiple books and manually assemble the information. Deep and precise, but inefficient for full-entity reads.

**Document data model** is like an encyclopedia with comprehensive self-contained articles. Everything about one topic is in one place — fast to retrieve, but maintaining consistency across overlapping facts in different articles is your problem.

---

_Understanding these trade-offs at this level means you're not just picking a database by popularity or familiarity — you're reasoning from first principles about what your system actually needs._