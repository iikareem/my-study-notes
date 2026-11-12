# Why Kafka Outperforms Databases for High-Throughput Data Ingestion

## Table of Contents

1. [What is High-Throughput Data?](https://claude.ai/chat/4ba19ee0-1228-4907-8216-dfcfbb61d7d1#1-what-is-high-throughput-data)
2. [Trying to Handle It with PostgreSQL (Relational)](https://claude.ai/chat/4ba19ee0-1228-4907-8216-dfcfbb61d7d1#2-trying-to-handle-it-with-postgresql-relational)
3. [Trying to Handle It with NoSQL (MongoDB / Cassandra)](https://claude.ai/chat/4ba19ee0-1228-4907-8216-dfcfbb61d7d1#3-trying-to-handle-it-with-nosql-mongodb--cassandra)
4. [Trying to Handle It with In-Memory Databases (Redis)](https://claude.ai/chat/4ba19ee0-1228-4907-8216-dfcfbb61d7d1#4-trying-to-handle-it-with-in-memory-databases-redis)
5. [Why Kafka Wins](https://claude.ai/chat/4ba19ee0-1228-4907-8216-dfcfbb61d7d1#5-why-kafka-wins)
6. [The Right Architecture](https://claude.ai/chat/4ba19ee0-1228-4907-8216-dfcfbb61d7d1#6-the-right-architecture)
7. [Summary Table](https://claude.ai/chat/4ba19ee0-1228-4907-8216-dfcfbb61d7d1#7-summary-table)

---

## 1. What is High-Throughput Data?

High-throughput data refers to scenarios where your system must **ingest, process, or store a massive volume of events in a very short time window**. This is not about occasional bursts — it is a sustained, continuous stream.

### Common Real-World Examples

|Domain|Example|Approximate Rate|
|---|---|---|
|IoT / GPS Tracking|Vehicle location events|100k–1M events/sec|
|E-Commerce|User click and page view events|50k–500k events/sec|
|Financial Markets|Trade ticks, price feeds|500k–5M events/sec|
|Mobile Apps|App usage, crash reports|10k–1M events/sec|
|Infrastructure|Server logs, metrics|100k–10M events/sec|

### Why It Is Hard

High-throughput data has specific characteristics that make it uniquely difficult:

- **Write-dominant** — reads are rare or delayed; writes are constant
- **Time-sensitive** — data must be accepted immediately or it is lost
- **Fan-out** — multiple consumers (analytics, ML, alerting) need the same data
- **Ordered** — event sequence often matters
- **Replayable** — you may need to reprocess past data

The challenge is not just storing data — it is **accepting it fast enough without dropping events**, and then making it available to multiple downstream systems without those systems slowing down ingestion.

---

## 2. Trying to Handle It with PostgreSQL (Relational)

PostgreSQL is one of the best relational databases ever built. It is ACID-compliant, feature-rich, and battle-tested. But it was designed for **transactional workloads with rich queries** — not for being a high-speed event bus.

### 2.1 How a Write Actually Works in PostgreSQL

When you `INSERT` a single row into PostgreSQL, the following chain of operations happens internally:

```
INSERT INTO events (user_id, lat, lng, ts) VALUES (...);
```

**Step 1 — WAL Write (Write-Ahead Log)** Before anything is written to the actual table, PostgreSQL writes the operation to the WAL — a sequential log on disk — so it can recover on crash. This is a mandatory fsync.

**Step 2 — Shared Buffer Update** The row is written into the in-memory shared buffer pool. If the buffer is full, a dirty page must be flushed to disk first (random I/O).

**Step 3 — Index Updates** Every index on the table (primary key, foreign keys, any additional indexes) must be updated. Each index is a B-tree that may require **rebalancing** on every write. With 3 indexes, you pay the cost 3 times.

**Step 4 — Lock Acquisition** Postgres must acquire a row-level or page-level lock to prevent concurrent conflicts. Under high concurrency, lock contention can create a queue of waiting writers.

**Step 5 — MVCC Versioning** PostgreSQL uses Multi-Version Concurrency Control (MVCC). Instead of overwriting rows, it creates a **new version** of the row and marks the old one as dead. This is what allows non-blocking reads — but it creates dead tuples.

```
Single INSERT cost:
  WAL write
  + buffer/page management
  + N × index B-tree update
  + lock acquisition
  + MVCC row version creation
```

This is manageable at low volume. At 100,000 inserts/second, every one of these costs multiplies catastrophically.

---

### 2.2 The Retention Problem: DELETE + VACUUM

For high-throughput scenarios, you often only need recent data (last 7 days, last 30 days). So you add a cleanup job:

```sql
DELETE FROM events WHERE created_at < NOW() - INTERVAL '7 days';
```

This feels simple. Internally, it is a disaster at scale.

**Dead Tuple Accumulation** `DELETE` in PostgreSQL does not physically remove rows. It marks them as dead tuples using MVCC. The rows still occupy disk space. If you delete 10 million rows, you have 10 million dead tuples sitting in your table.

**VACUUM Must Clean Up** PostgreSQL's background VACUUM process periodically scans the table and reclaims dead tuple space. Under high write + delete rates:

- Dead tuples accumulate faster than VACUUM can process them
- Table size **bloats** — a table with 10M live rows may occupy the space of 100M rows
- VACUUM itself consumes significant I/O and CPU, competing with your live writes
- `VACUUM FULL` (to fully reclaim space) **locks the entire table**

**Index Bloat** As rows are deleted and inserted, B-tree index pages become partially empty. The index grows larger but becomes less dense, degrading query performance. Periodic `REINDEX` is needed, which is also expensive.

**The Cycle of Pain**

```
High writes → dead tuples accumulate
           → VACUUM runs → I/O spike → writes slow down
           → Index bloats → queries slow → REINDEX needed
           → REINDEX locks index → queries blocked
           → repeat
```

---

### 2.3 Multi-Consumer Fan-Out

In high-throughput systems, multiple consumers need the same data — analytics, fraud detection, alerting, ML pipelines.

With PostgreSQL, you have no native fan-out. Your options are:

- Each consumer polls the table with `SELECT` → **adds read load, competes with writes**
- You implement a queue table manually → **you are now building Kafka inside Postgres**
- You use LISTEN/NOTIFY → **not designed for high volume, drops messages**

There is no concept of a consumer tracking its own offset. You have to build this yourself with offset columns, processed flags, and coordination logic — all of which adds write overhead and race condition risk.

---

### 2.4 PostgreSQL Throughput Reality

|Workload|Approximate Sustainable Throughput|
|---|---|
|Simple indexed inserts|10,000–50,000 rows/sec|
|With multiple indexes|5,000–20,000 rows/sec|
|With concurrent reads|2,000–10,000 rows/sec|
|With DELETE + VACUUM pressure|Degrades unpredictably|

This is not a failure of PostgreSQL — it is doing exactly what it was designed to do. It is simply the wrong tool for this job.

---

## 3. Trying to Handle It with NoSQL (MongoDB / Cassandra)

NoSQL databases remove some relational overhead, but they introduce their own costs at high throughput.

### 3.1 MongoDB

MongoDB uses a document model and WiredTiger storage engine. It improved significantly over earlier versions, but under high write throughput:

**Write Concern Overhead** MongoDB's default write concern (`w:1`) acknowledges a write after it hits the primary. For durability (`w:majority`), it must wait for replica acknowledgment — adding latency under load.

**Journaling** Like PostgreSQL's WAL, MongoDB writes to a journal before committing. This is necessary for durability but adds I/O on every write.

**Index Cost** MongoDB indexes have the same fundamental problem as relational indexes — every write must update every index. B-tree rebalancing is inherent to the data structure, not a database-specific limitation.

**Oplog Pressure** MongoDB's replication uses the oplog (operations log). Under heavy writes, the oplog can fill and wrap around, causing secondaries to fall out of sync. This requires careful sizing and monitoring.

**No Native Consumer Offset Tracking** Like PostgreSQL, MongoDB has no concept of a consumer tracking where it is in the stream. Change Streams (introduced in 3.6) provide some streaming capability, but they are not designed for millions of events/second fan-out across many independent consumers.

---

### 3.2 Apache Cassandra

Cassandra is often cited as a high-throughput write database — and it genuinely is better than PostgreSQL for raw writes. But it still has important limitations.

**What Cassandra Does Well**

- Distributed writes with no single master (leaderless)
- Very fast writes via append-only SSTables (similar concept to Kafka)
- Tunable consistency

**What Still Hurts at Very High Throughput**

_Compaction Cost_ Cassandra writes to SSTables (immutable files) and periodically merges them in a process called **compaction**. Under high write rates:

- Many SSTables accumulate
- Compaction is I/O-intensive and competes with live writes
- Read performance degrades if compaction falls behind

_Tombstones_ Like PostgreSQL's dead tuples, Cassandra marks deletes with **tombstones** rather than removing data immediately. Tombstones accumulate and slow down reads until compaction removes them. With retention-style deletes at high volume, tombstone pressure becomes severe.

_Read-Repair and Hinted Handoff Overhead_ Distributed consistency mechanisms add background I/O that competes with your write path.

_No Fan-Out_ Cassandra also has no concept of multiple consumers tracking independent offsets. You build this yourself or you don't have it.

**Cassandra Write Throughput** Cassandra can sustain higher write throughput than PostgreSQL — sometimes 100k–500k writes/sec per node — but it still pays compaction and tombstone costs, and it offers none of Kafka's consumer group semantics.

---

## 4. Trying to Handle It with In-Memory Databases (Redis)

Redis is extremely fast because it stores everything in RAM. It seems like the obvious solution for high-throughput ingestion.

### 4.1 What Redis Offers

- Sub-millisecond write latency
- Simple data structures (lists, sorted sets, streams)
- Redis Streams — a built-in log-like structure with consumer groups

### 4.2 Where It Falls Apart at Scale

**Memory Is Finite and Expensive** RAM is the scarcest and most expensive storage resource. At 1 million events/second with each event being 500 bytes, you generate **~43 GB of data per day**. Storing even 7 days of retention requires ~300 GB of RAM. This is extremely costly compared to disk-based storage.

**Persistence Has a Cost** Redis is in-memory first. Persistence options (RDB snapshots, AOF append-only file) add I/O overhead and reduce your write throughput. Under maximum write load with AOF enabled (`appendfsync always`), Redis throughput drops significantly.

**No True Distributed Partitioning** Redis Cluster shards data across nodes, but it is not designed with the same partition-level parallelism and consumer group semantics that Kafka has natively.

**Data Loss Risk** Without careful persistence configuration, a Redis crash can lose recent data. For high-throughput event ingestion where every event matters, this is a serious risk.

**Redis Streams vs Kafka** Redis Streams (`XADD`, `XREADGROUP`) implement a log structure that resembles Kafka, but:

- Not designed for millions of events/second across dozens of consumer groups
- Memory-bound retention
- Less operational tooling for monitoring consumer lag, partition assignment, rebalancing

---

## 5. Why Kafka Wins

Kafka was designed from the ground up for exactly this problem. Its architecture makes the hard things trivially cheap.

### 5.1 The Core Design: Append-Only Partitioned Log

Kafka's fundamental data structure is a **partitioned, append-only log**.

```
Partition 0: [event_1] [event_2] [event_3] ... [event_N] → append here
Partition 1: [event_1] [event_2] [event_3] ... [event_N] → append here
Partition 2: [event_1] [event_2] [event_3] ... [event_N] → append here
```

Every write is a sequential append to the end of a partition file. There is no:

- Index to update
- Lock to acquire
- MVCC version to create
- B-tree to rebalance
- Dead tuple to create

Sequential disk writes are **10–100x faster** than random disk writes. This is the foundation of Kafka's throughput advantage.

**The Broker Does No Business Logic** A Kafka broker's job is simple: accept the message, append it to the right partition, acknowledge. No transformation, no validation, no join, no computation. This simplicity is what makes it fast.

---

### 5.2 Zero-Copy and OS-Level Optimization

Kafka uses the OS `sendfile()` system call (zero-copy) to transfer data from disk to network without copying it through user space. This dramatically reduces CPU usage when consumers are reading.

```
Normal data path:  disk → kernel buffer → user space → socket buffer → network
Kafka (zero-copy): disk → kernel buffer → network  (skips user space entirely)
```

---

### 5.3 Batching at Every Level

Kafka batches aggressively:

- **Producer side** — messages are batched before being sent to the broker
- **Broker side** — messages are written to disk in batches
- **Consumer side** — messages are fetched in batches

A single disk write or network call can carry thousands of events. This amortizes the fixed overhead of I/O operations across many messages, dramatically increasing effective throughput.

---

### 5.4 Retention Is Trivially Cheap

Kafka stores data in **log segment files**. When a segment is older than your retention period, Kafka simply deletes the file.

```
segment_000.log  [7 days old] → delete the file   (one OS operation)
segment_001.log  [5 days old] → delete the file   (one OS operation)
segment_002.log  [2 days old] → keep
segment_003.log  [active]     → writing here
```

There are no dead tuples. No VACUUM. No tombstones. No compaction. No index cleanup. **Retention is free.**

---

### 5.5 Consumer Groups and Independent Offset Tracking

Every consumer in Kafka tracks its own **offset** — its position in the partition log. The broker stores nothing about consumer state beyond this simple integer.

This gives you:

**Independent consumers at different speeds**

```
Producer → Kafka Partition
                 ↓
Consumer A (real-time alerting)    at offset 1,500,000
Consumer B (ML feature pipeline)   at offset 1,200,000
Consumer C (data warehouse)        at offset  900,000
```

All three read from the same partition. None blocks the others. None affects write throughput.

**Replay from any point**

```
Consumer C: "I need to reprocess from the beginning"
→ reset offset to 0
→ replay all historical data at maximum speed
→ no impact on other consumers or on producers
```

**Parallel consumption within a group** Each partition can be assigned to exactly one consumer in a consumer group. With 12 partitions, 12 consumer instances process in parallel — no coordination required, no race conditions.

---

### 5.6 Replication Without Write Penalty

Kafka replicates partitions to multiple brokers for durability. But unlike a database, replication happens **asynchronously in the background** (for `acks=1`) or via a highly optimized ISR (In-Sync Replica) mechanism (for `acks=all`). The write path is not blocked by replica coordination the way database synchronous replication is.

---

### 5.7 Throughput Numbers

|System|Sustained Write Throughput|Notes|
|---|---|---|
|PostgreSQL|10k–50k rows/sec|Degrades under index and VACUUM pressure|
|MongoDB|20k–100k docs/sec|Compaction and journal overhead|
|Cassandra|100k–500k rows/sec/node|Compaction and tombstone pressure|
|Redis|500k–1M ops/sec|Memory-bound, persistence cost|
|**Kafka**|**1M–10M+ messages/sec**|Per cluster, scales linearly with partitions|

---

### 5.8 Schema Evolution with Avro + Schema Registry

Kafka combined with Apache Avro and the Confluent Schema Registry gives you a robust contract between producers and consumers:

- **Compact binary serialization** — much smaller than JSON, reducing broker storage and network bandwidth
- **Schema versioning** — producers and consumers can evolve independently within compatibility rules (backward, forward, full)
- **Schema enforcement** — a producer cannot accidentally send malformed data that breaks consumers

```
Producer → serialize with Avro schema v2
Kafka Broker → stores bytes (schema-agnostic)
Consumer A → deserializes with schema v1 (backward compatible)
Consumer B → deserializes with schema v2
```

---

### 5.9 Stream Processing Integration

Because your data is already in Kafka, you get streaming computation for free:

|Tool|Use Case|
|---|---|
|Kafka Streams|Embedded stream processing in your application|
|Apache Flink|Stateful, exactly-once stream processing at scale|
|ksqlDB|SQL queries over live Kafka topics|
|Spark Structured Streaming|Batch + streaming unified model|

You can detect anomalies, compute aggregations, join streams, and enrich events **without moving data out of Kafka first**.

---

## 6. The Right Architecture

Kafka does not replace your database. It **protects it** and acts as the nervous system of your data platform.

```
                         ┌─────────────────────────────┐
IoT Sensors              │                             │
Mobile Apps    ──────→   │        KAFKA CLUSTER        │
Web Events               │   (append-only, partitioned │
Server Logs              │    replicated, retained)    │
                         └──────────────┬──────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
           ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
           │   Consumer   │   │   Consumer   │   │   Consumer   │
           │  (Alerting)  │   │ (Data Ware-  │   │ (ML Feature  │
           │              │   │   house)     │   │  Pipeline)   │
           └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
                  │                  │                   │
                  ▼                  ▼                   ▼
           ┌──────────┐      ┌────────────┐      ┌────────────┐
           │  Redis   │      │  Postgres  │      │  Feature   │
           │ (alerts) │      │(aggregated │      │   Store    │
           └──────────┘      │  reports) │      └────────────┘
                             └────────────┘
```

**What this achieves:**

- Kafka absorbs the write firehose — nothing else is under write pressure
- Each downstream system receives only the data it needs, at its own pace
- PostgreSQL only receives **clean, aggregated, filtered** writes — well within its capabilities
- Each consumer scales independently
- Any consumer can replay historical data without affecting others

---

## 7. Summary Table

|Dimension|PostgreSQL|MongoDB|Cassandra|Redis|**Kafka**|
|---|---|---|---|---|---|
|Write model|Random I/O + B-tree|Journaled + B-tree|Append SSTable|In-memory|**Append-only log**|
|Index cost on write|High|High|Medium|Low|**None**|
|Locking / MVCC|Yes|Yes|No|No|**No**|
|Retention cost|DELETE + VACUUM|Delete + compaction|Tombstones + compaction|TTL (cheap, memory)|**Segment deletion (free)**|
|Multi-consumer fan-out|Manual|Change Streams (limited)|None native|Redis Streams (limited)|**Consumer groups (native)**|
|Offset / replay|Manual|Manual|None|Limited|**Native, free**|
|Throughput ceiling|~50k/sec|~100k/sec|~500k/sec|~1M/sec (memory-bound)|**1M–10M+/sec**|
|Storage cost|High (MVCC bloat)|Medium|Medium|Very high (RAM)|**Low (disk, compressed)**|
|Horizontal scale|Hard|Medium|Good|Medium|**Excellent**|
|Stream processing|None native|None native|None native|None native|**Native (Streams, Flink)**|
|Schema evolution|Migrations|Schema-less|Migrations|None|**Avro + Schema Registry**|

---

## Final Takeaway

> **Kafka is not better than databases. It is a different tool solving a different problem.**

Databases are built for **consistency, rich queries, and transactional correctness**. They pay a structural cost for those guarantees — and that cost shows up as a throughput ceiling.

Kafka is built for **accepting data as fast as possible and making it available to many consumers in order**. It pays no cost for rich queries, transactions, or MVCC — because it does none of those things.

The mistake is using a database as an event bus. The moment you do, you are fighting its design. You pay for MVCC you don't need, indexes you don't query, VACUUM that fights your writes, and you get none of the consumer group semantics you actually want.

Use Kafka where the primary requirement is **fast, durable, ordered, fan-out event ingestion**. Use databases where the primary requirement is **querying, aggregation, and transactional correctness on settled data**.

Used together, they are far more powerful than either alone.