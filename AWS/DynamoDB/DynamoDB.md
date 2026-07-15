---
tags:
  - aws
  - certification
  - dynamodb
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[DynamoDB Quiz]] · [[DynamoDB Hints]]

# DynamoDB Complete Exam Guide

---

## Table of Contents

1. [What is DynamoDB?](#1-what-is-dynamodb)
2. [Core Data Model](#2-core-data-model)
3. [Primary Keys](#3-primary-keys)
4. [Read / Write Capacity Modes](#4-read--write-capacity-modes)
5. [RCU & WCU Calculations](#5-rcu--wcu-calculations)
6. [Consistency Models](#6-consistency-models)
7. [API Operations](#7-api-operations)
8. [Indexes (LSI & GSI)](#8-indexes-lsi--gsi)
9. [DynamoDB Streams](#9-dynamodb-streams)
10. [DAX (DynamoDB Accelerator)](#10-dax-dynamodb-accelerator)
11. [Global Tables](#11-global-tables)
12. [Transactions](#12-transactions)
13. [TTL (Time To Live)](#13-ttl-time-to-live)
14. [Backup & Restore](#14-backup--restore)
15. [Security](#15-security)
16. [Design Patterns](#16-design-patterns)
17. [Common Exam Traps](#17-common-exam-traps)
18. [Exam Quick-Reference Cheat Sheet](#18-exam-quick-reference-cheat-sheet)

---

## 1. What is DynamoDB?

DynamoDB is a **fully managed, serverless, NoSQL** key-value and document database provided by AWS.

### Key Characteristics

- **Serverless**: No servers to manage, provision, or patch.
- **Fully managed**: AWS handles replication, scaling, hardware failures, and maintenance.
- **Performance**: Single-digit millisecond latency at any scale.
- **Scalability**: Can handle millions of requests per second and store terabytes/petabytes of data.
- **Multi-region**: Supports active-active replication across regions via Global Tables.
- **High availability**: Data is automatically replicated across 3 Availability Zones within a region.

### When to Use DynamoDB (vs other databases)

|Scenario|Use DynamoDB?|
|---|---|
|Need flexible schema with key-value or document data|✅ Yes|
|Need high throughput at low latency|✅ Yes|
|Need complex SQL joins or aggregations|❌ No — use RDS/Redshift|
|Access patterns are well-defined upfront|✅ Yes|
|Access patterns are unpredictable / ad-hoc|❌ Harder — requires careful design|

> **Exam hint**: DynamoDB is the go-to answer for "serverless", "NoSQL", "millisecond latency", "massive scale", or "key-value" workloads.

---

## 2. Core Data Model

DynamoDB stores data in **tables**. A table is a collection of **items** (analogous to rows). Each item has **attributes** (analogous to columns).

### Key Terms

|Term|Description|
|---|---|
|**Table**|The top-level container for data|
|**Item**|A single record in the table (like a row). Max size: **400 KB**|
|**Attribute**|A data field on an item (like a column)|
|**Partition**|DynamoDB physically splits data into partitions based on the partition key|

### Schema-less Design

Unlike relational databases, DynamoDB is **schema-less** — each item can have different attributes. The only requirement is that every item must have the primary key attribute(s).

```
Item 1: { userId: "u1", name: "Alice", age: 30 }
Item 2: { userId: "u2", email: "bob@example.com", orders: 5 }
```

Both items are valid in the same table — they don't need the same attributes.

### Attribute Data Types

|Category|Types|
|---|---|
|**Scalar**|String (S), Number (N), Binary (B), Boolean (BOOL), Null (NULL)|
|**Document**|Map (M) — like JSON object, List (L) — like JSON array|
|**Set**|String Set (SS), Number Set (NS), Binary Set (BS)|

> **Exam hint**: Item max size is **400 KB**. If your item exceeds this, store large data in S3 and keep a reference (URL/key) in DynamoDB.

---

## 3. Primary Keys

Every item in DynamoDB must have a unique primary key. There are two types:

### Option 1 — Simple Primary Key (Partition Key only)

- Also called **Hash Key**.
- The partition key value must be **unique** across all items in the table.
- DynamoDB hashes the partition key to determine which physical partition stores the item.

```
Table: Users
Partition Key: userId (must be unique per item)

userId  | name    | email
--------|---------|------
"u1"    | "Alice" | ...
"u2"    | "Bob"   | ...
```

### Option 2 — Composite Primary Key (Partition Key + Sort Key)

- Also called **Hash Key + Range Key**.
- The **partition key** can repeat across items.
- The **sort key** differentiates items within the same partition.
- The **combination** of partition key + sort key must be unique.
- Within the same partition, items are **sorted** by the sort key.

```
Table: Orders
Partition Key: customerId | Sort Key: orderId

customerId | orderId     | total
-----------|-------------|------
"c1"       | "2024-001"  | 100
"c1"       | "2024-002"  | 200   ← same partition key, different sort key
"c2"       | "2024-001"  | 50
```

### Why This Matters

The partition key determines **data distribution** across partitions. A poor partition key choice causes **hot partitions** — one partition receiving most traffic and getting throttled while others are idle.

**Good partition keys**: high cardinality, evenly distributed access (e.g., userId, orderId, deviceId).

**Bad partition keys**: low cardinality or skewed access (e.g., status ["active"/"inactive"], date [everyone writes to today's date]).

> **Exam hint**: A hot partition is when one partition key value receives disproportionately high traffic. Solution: **write sharding** — add a random suffix (e.g., `userId#1`, `userId#2`) to distribute load.

---

## 4. Read / Write Capacity Modes

DynamoDB has two modes for managing capacity and cost:

### Provisioned Capacity Mode

- You **explicitly specify** how many Read Capacity Units (RCU) and Write Capacity Units (WCU) per second you need.
- You are **billed for provisioned capacity** whether you use it or not.
- Supports **Auto Scaling** — DynamoDB can automatically adjust RCU/WCU within defined min/max bounds based on traffic.
- Supports **Reserved Capacity** — commit to 1 or 3 years for up to 76% cost savings.

**Best for**: Predictable, steady-state workloads where you know your traffic patterns.

### On-Demand Mode

- DynamoDB **automatically scales** to any request rate — no capacity planning needed.
- You pay **per request**: Read Request Units (RRU) and Write Request Units (WRU).
- Instantly handles traffic spikes with no throttling.
- Roughly **2–3× more expensive** than Provisioned at sustained high load.
- You can **switch between modes once per 24 hours**.

**Best for**: Unpredictable traffic, new applications, or workloads with large spikes.

### Comparison Table

|Feature|Provisioned|On-Demand|
|---|---|---|
|Capacity planning|Required|None|
|Cost model|Pay for capacity|Pay per request|
|Auto-scale|Yes (with Auto Scaling)|Automatic|
|Throttling possible|Yes (if limit exceeded)|No (absorbs any traffic)|
|Reserved pricing|Yes|No|
|Mode switch|Once per 24 hours|Once per 24 hours|
|Best for|Predictable workloads|Spiky/unpredictable workloads|

> **Exam hint**: "Unpredictable traffic" or "sudden spikes" → **On-Demand**. "Cost optimization for steady traffic" → **Provisioned + Auto Scaling or Reserved Capacity**.

---

## 5. RCU & WCU Calculations

This is frequently tested. Memorize these formulas.

### Write Capacity Units (WCU)

**1 WCU = 1 write per second for an item up to 1 KB**

If item > 1 KB, round up to the nearest KB:

```
WCU needed = ceil(item size in KB) × writes per second

Example: 3.5 KB item, 10 writes/sec
→ ceil(3.5) = 4 KB
→ 4 × 10 = 40 WCU needed
```

### Read Capacity Units (RCU)

**1 RCU = 1 strongly consistent read per second for an item up to 4 KB**

```
RCU (strongly consistent) = ceil(item size / 4 KB) × reads per second

Example: 10 KB item, 5 reads/sec, strongly consistent
→ ceil(10/4) = 3
→ 3 × 5 = 15 RCU needed
```

**Eventually consistent reads cost half**:

```
RCU (eventually consistent) = ceil(item size / 4 KB) × reads per second / 2

Same example, eventually consistent:
→ 15 / 2 = 7.5 → round up = 8 RCU needed
```

### Transactional Costs

- **Transactional writes** cost **2× WCU**
- **Transactional reads** cost **2× RCU**

### Summary Table

|Operation|Cost|
|---|---|
|1 strongly consistent read (≤ 4 KB)|1 RCU|
|1 eventually consistent read (≤ 4 KB)|0.5 RCU|
|1 transactional read (≤ 4 KB)|2 RCU|
|1 write (≤ 1 KB)|1 WCU|
|1 transactional write (≤ 1 KB)|2 WCU|

---

## 6. Consistency Models

### Eventually Consistent Reads (Default)

- Data may be **slightly stale** right after a write (usually milliseconds).
- DynamoDB returns data from the nearest replica — which may not have received the latest write yet.
- **Costs 0.5 RCU** per 4 KB read.
- Higher throughput, lower cost.

### Strongly Consistent Reads

- Returns the **most up-to-date data** — guaranteed to reflect all prior successful writes.
- DynamoDB reads from the **leader node** of the partition.
- **Costs 1 RCU** per 4 KB read.
- Higher latency possible, not available on GSIs.
- Request by setting `ConsistentRead: true` in API calls.

### Transactional Reads

- ACID reads across multiple items.
- See Transactions section below.
- **Costs 2 RCU** per 4 KB.

> **Exam hint**: GSIs **only support eventual consistency** — you cannot do a strongly consistent read on a GSI. If the exam question requires strong consistency AND a secondary index, use an LSI.

---

## 7. API Operations

### Read Operations

#### GetItem

- Retrieves a **single item** by its exact primary key (partition key + optional sort key).
- Supports strongly consistent reads.
- Most efficient single-item read operation.

#### BatchGetItem

- Retrieves **up to 100 items** from one or more tables in a single call.
- Max response size: **16 MB**.
- Operations are independent — some may succeed while others fail (partial results returned).
- Unprocessed items are returned and must be retried.

#### Query

- Retrieves **all items** that share the same **partition key** value.
- Can filter by sort key using key condition expressions (=, <, >, between, begins_with).
- Can use `FilterExpression` to further filter results (applied after reading — still consumes RCU for all items read).
- Works on a table or an index.
- Much more efficient than Scan.

#### Scan

- Reads **every item** in the table or index.
- Can filter results with `FilterExpression`, but DynamoDB reads ALL items first and then filters — you are charged RCU for every item read, even items that are filtered out.
- Very expensive on large tables.
- Use **parallel scan** to speed it up by splitting the table into segments and scanning each in parallel.
- Use `ProjectionExpression` to return only specific attributes (reduces payload but NOT the RCU cost).

> **Exam hint**: Scan is almost always the "wrong" answer in exam questions about efficiency. Query is preferred when you know the partition key.

### Write Operations

#### PutItem

- Creates a new item, or **fully replaces** an existing item with the same primary key.
- If an item with that key exists, it is completely overwritten.

#### UpdateItem

- **Partially updates** an existing item — only the specified attributes are changed.
- If the item doesn't exist, it creates it (upsert behavior).
- Supports **atomic counters** — increment/decrement a numeric attribute atomically.

#### DeleteItem

- Removes a single item by its primary key.

#### BatchWriteItem

- Performs **up to 25 PutItem or DeleteItem** operations in a single call.
- Max size: **16 MB** total.
- Does **not** support UpdateItem in a batch.
- Not atomic — some operations may succeed while others fail.

### Conditional Writes

All write operations support a `ConditionExpression` — the write only executes if the condition is true. If the condition fails, DynamoDB returns a `ConditionalCheckFailedException`.

**Use cases**:

- Prevent overwriting an existing item: `attribute_not_exists(pk)`
- Optimistic locking: `version = :expected_version`
- Business logic enforcement: `balance >= :amount`

Conditional writes are **idempotent** by design — safe to retry.

---

## 8. Indexes (LSI & GSI)

Indexes allow you to query data on attributes other than the primary key.

### Local Secondary Index (LSI)

An LSI uses the **same partition key** as the base table but a **different sort key**. It is "local" to a partition.

**Key facts**:

- Must be created **at table creation time** — cannot add/remove LSIs after the table exists.
- Maximum **5 LSIs** per table.
- **Shares** the RCU/WCU of the base table (no separate capacity).
- Supports **strongly consistent reads** (unlike GSI).
- All items with the same partition key across the base table and LSI share a combined size limit of **10 GB** per partition key value.
- **Projected attributes**: you choose which attributes to copy into the LSI (ALL, KEYS_ONLY, or specific attributes).

**When to use**: You need to query items by a different sort key but within the same partition as the base table.

### Global Secondary Index (GSI)

A GSI has a **completely different partition key** (and optional sort key) from the base table. It is "global" because queries can span all partitions.

**Key facts**:

- Can be created or deleted **at any time** — even after the table exists.
- Maximum **20 GSIs** per table (default limit, can be raised).
- Has its **own separate RCU/WCU** capacity — must be provisioned independently.
- Supports **eventual consistency only** — no strongly consistent reads.
- If a GSI runs out of write capacity, **writes to the base table are rejected** (not just the GSI write — the entire base table write fails).
- GSI data is eventually consistent with the base table.

**When to use**: You need to query by an attribute that is not the table's partition key.

### LSI vs GSI Comparison

|Feature|LSI|GSI|
|---|---|---|
|Partition key|Same as base table|Any attribute|
|Sort key|Different attribute|Any attribute (optional)|
|Created when|Table creation only|Any time|
|Consistency|Strongly or eventually|Eventually only|
|Capacity|Shared with base table|Own capacity|
|Max per table|5|20 (default)|
|Throttle behavior|Table throttles affect LSI|GSI throttle blocks base table writes|
|Size limit|10 GB per partition|No limit|

> **Exam trap**: If a GSI is under-provisioned for writes, **the base table write fails**. This is a common trick question. Always ensure GSI WCU is adequate.

---

## 9. DynamoDB Streams

### What is it?

DynamoDB Streams is an **ordered log of item-level changes** in a DynamoDB table. Every time an item is created, updated, or deleted, a record is written to the stream.

### Key Facts

- Streams retain data for **24 hours** only.
- Records appear in the stream in the **order they happened**, within each shard (partition).
- Streams are read by shards — similar to Kinesis shards.
- Often used to **trigger Lambda functions** for event-driven architectures.

### Stream View Types

| Type                 | What is captured                                 |
| -------------------- | ------------------------------------------------ |
| `KEYS_ONLY`          | Only the key attributes of the modified item     |
| `NEW_IMAGE`          | The entire item as it appears after the change   |
| `OLD_IMAGE`          | The entire item as it appeared before the change |
| `NEW_AND_OLD_IMAGES` | Both before and after images of the item         |

### Common Use Cases

- **Trigger Lambda** on data changes (e.g., send notification when order is placed).
- **Cross-region replication** (basis of Global Tables).
- **Analytics**: stream changes to Kinesis or Elasticsearch.
- **Audit logging**: record all changes to an item over time.
- **TTL**: when items expire via TTL, a delete record appears in the stream — useful for triggering cleanup logic.

### DynamoDB Streams vs Kinesis Data Streams for DynamoDB

DynamoDB can also write changes directly to **Kinesis Data Streams** (a separate integration):

|Feature|DynamoDB Streams|Kinesis Data Streams|
|---|---|---|
|Retention|24 hours|Up to 1 year|
|Consumers|Lambda, DynamoDB-specific|Many (Lambda, Firehose, Analytics, etc.)|
|Cost|Free|Extra Kinesis cost|
|Use case|Event-driven, replication|Long-term audit, complex fan-out|

> **Exam hint**: If the exam mentions **24-hour retention** and Lambda triggers → DynamoDB Streams. If it mentions **longer retention** or **multiple consumers** → Kinesis Data Streams integration.

---

## 10. DAX (DynamoDB Accelerator)

### What is it?

DAX is an **in-memory cache** for DynamoDB that delivers **microsecond read latency** (compared to single-digit milliseconds without DAX).

### How it Works

- DAX sits **between your application and DynamoDB**.
- Reads are served from DAX cache if available (cache hit) — no DynamoDB call needed.
- Writes go to DynamoDB first, then update the DAX cache (**write-through cache**).
- **No application code change required** — DAX uses the same DynamoDB API.

### Key Facts

- Reduces read load on DynamoDB → lower RCU consumption → cost savings at scale.
- DAX runs as a **cluster** — you choose instance size and number of nodes.
- Supports **item cache** (for GetItem, BatchGetItem) and **query cache** (for Query, Scan).
- Cache TTL: default **5 minutes** for item cache, **1 minute** for query cache.

### When NOT to Use DAX

- **Strongly consistent reads** — DAX only caches eventually consistent reads. For strongly consistent reads, the request passes through DAX directly to DynamoDB.
- **Write-heavy workloads** — DAX provides little benefit if most operations are writes.
- **Infrequent access** — cache is only useful if data is accessed repeatedly.
- **Already low latency** — if millisecond is fine, DAX adds cost without benefit.

> **Exam trap**: DAX does NOT help with strongly consistent reads. Any exam question requiring strongly consistent + caching → DAX is the wrong answer. Go directly to DynamoDB with `ConsistentRead: true`.

---

## 11. Global Tables

### What is it?

Global Tables provides **multi-region, active-active** replication for DynamoDB. Tables in multiple regions are automatically synchronized — you can read and write to any region.

### Key Facts

- All replicas are **writable** (active-active, not active-passive).
- Replication is **asynchronous** — eventual consistency across regions.
- **Conflict resolution**: last writer wins (based on timestamp).
- Requires **DynamoDB Streams** to be enabled (Global Tables uses Streams internally for replication).
- Must use **On-Demand** capacity or **Provisioned with Auto Scaling**.
- Each region's replica has its own WCU/RCU — you provision independently per region.

### Use Cases

- Low-latency global reads (users in multiple continents).
- Disaster recovery — if one region goes down, other regions continue serving traffic.
- Data residency requirements — keep data close to users in specific regions.

> **Exam hint**: "Multi-region, active-active, low latency for global users" → Global Tables.

---

## 12. Transactions

### What are Transactions?

DynamoDB supports **ACID transactions** — operations that are all-or-nothing across multiple items and/or tables.

- **Atomic**: either all operations succeed, or none do.
- **Consistent**: data remains valid after the transaction.
- **Isolated**: transactions don't interfere with each other.
- **Durable**: committed writes are persisted.

### Transaction APIs

#### TransactWriteItems

- Up to **25 Put, Update, Delete, or ConditionCheck** operations in a single transaction.
- All items can span **multiple tables**.
- All-or-nothing: if one operation fails, the entire transaction is rolled back.

#### TransactGetItems

- Up to **25 GetItem** operations across multiple tables.
- All reads are strongly consistent and taken at the same point in time (snapshot isolation).

### Capacity Cost

Transactions cost **2× the normal RCU/WCU**:

- Each transactional write = 2 WCU per 1 KB
- Each transactional read = 2 RCU per 4 KB

### When to Use

- Financial transfers (debit one account, credit another — both must succeed or fail together).
- Inventory management (check stock exists, then decrement it — atomically).
- Any multi-item operation where partial success is unacceptable.

> **Exam hint**: "ACID", "all-or-nothing", "multiple tables/items atomically" → Transactions.

---

## 13. TTL (Time To Live)

### What is it?

TTL allows you to automatically **delete items** from DynamoDB after a specified time, without any manual intervention or additional cost.

### How it Works

1. Choose an attribute name on your table (e.g., `expiresAt`).
2. Enable TTL and specify that attribute as the TTL attribute.
3. Set the attribute value on each item to a **Unix epoch timestamp** (seconds since Jan 1, 1970) representing when the item should expire.
4. DynamoDB automatically scans for expired items and deletes them — typically within **48 hours** of the expiration time (no SLA guarantee on exact deletion time).

### Key Facts

- **No additional cost** — TTL deletions are free (do not consume WCU).
- Expired items **may still be readable** for up to 48 hours before deletion.
- TTL deletions **appear in DynamoDB Streams** as delete events — you can trigger Lambda to react to expirations.
- Filter expired items in your application using `FilterExpression` if needed while items await deletion.

### Use Cases

- Session data with automatic expiry.
- Temporary tokens or OTPs.
- Event logs with a retention period.
- Shopping cart items that expire if not purchased.

> **Exam hint**: "Auto-expire items", "session management", "no extra cost for deletion" → TTL.

---

## 14. Backup & Restore

### On-Demand Backup

- Create a **full backup** of the table at any point in time with a single API call or console action.
- **No performance impact** on the table during backup.
- Backups are retained **until you explicitly delete them** (no automatic expiry).
- Restore to a **new table** in the same or a different AWS region.
- Restores include all indexes (LSI, GSI), streams settings, etc.

### Point-in-Time Recovery (PITR)

- **Continuous, automatic backups** — DynamoDB maintains a full incremental backup rolling window.
- Can restore the table to **any second within the last 35 days**.
- Must be **explicitly enabled** per table — it is not on by default.
- Restores to a **new table only** (cannot restore in-place).
- No additional storage cost beyond what you use (pay for backup storage).

### Comparison

|Feature|On-Demand Backup|PITR|
|---|---|---|
|Enabled by default|No|No (must enable)|
|Granularity|Specific moment (when triggered)|Any second in last 35 days|
|Retention|Until deleted|35-day rolling window|
|Restore target|New table|New table|
|Performance impact|None|None|

> **Exam hint**: "Restore to any second in the last 35 days" → **PITR**. "Take a backup right now before a major change" → **On-Demand Backup**.

---

## 15. Security

### Encryption

- **Encryption at rest**: All DynamoDB data is encrypted at rest by default.
    - **AWS owned key** (default): Free, managed entirely by AWS.
    - **AWS managed key** (aws/dynamodb): Stored in AWS KMS, you can audit via CloudTrail.
    - **Customer managed key (CMK)**: You create and manage the key in KMS. Most control, additional KMS cost.
- **Encryption in transit**: All communication with DynamoDB uses **TLS (HTTPS)** — enforced automatically.

### IAM Access Control

- Control access via **IAM policies** attached to users, roles, or groups.
- Supports **fine-grained access control** using IAM condition keys:
    - `dynamodb:LeadingKeys` — restrict a user to only access items where the partition key matches their user ID (e.g., a user can only read/write their own data).
    - `dynamodb:Attributes` — restrict which attributes a user can read or write.

### VPC Endpoints

- Use **VPC Gateway Endpoints** for DynamoDB to keep traffic within the AWS network (no public internet).
- Useful for Lambda functions or EC2 inside a VPC that need to access DynamoDB privately.

### CloudTrail

- All DynamoDB API calls are logged in **AWS CloudTrail** for auditing.
- Tracks who made requests, from where, and when.

---

## 16. Design Patterns

### Large Object Pattern

When items exceed 400 KB:

1. Store the large data (e.g., images, documents) in **S3**.
2. Store the S3 object key or URL as an attribute in the DynamoDB item.
3. Application retrieves the DynamoDB item, then fetches from S3 using the stored reference.

### Write Sharding (Hot Partition Fix)

When a single partition key receives too much traffic:

1. Append a random **suffix** (e.g., `1` to `10`) to the partition key: `userId#3`.
2. Distribute writes across `userId#1` through `userId#10`.
3. To read all data for a user, query all shards and aggregate results.

### Adjacency List Pattern

Model many-to-many relationships (e.g., users and groups):

- Partition key: entity ID (e.g., `USER#u1` or `GROUP#g1`)
- Sort key: related entity ID (e.g., `GROUP#g1` or `USER#u1`)
- One item per relationship, and entity metadata stored with `pk = sk`.

### Sparse Indexes

GSI only includes items that have the indexed attribute. If most items don't have a particular attribute, a GSI on that attribute is a **sparse index** — only a small subset of items appear in it, making queries faster and cheaper.

**Use case**: Index only "open" orders by setting a GSI attribute only on open orders. When closed, remove the attribute → item disappears from the GSI automatically.

### One Table Design

A single DynamoDB table stores multiple entity types. Items are differentiated using a `type` attribute or by conventions in the sort key (e.g., `PROFILE#`, `ORDER#`, `ITEM#`). This maximizes efficiency and avoids cross-table joins.

---

## 17. Common Exam Traps

### Trap 1: LSI vs GSI Creation Timing

- **LSI** → created at **table creation only**. Cannot add or remove later.
- **GSI** → can be **added or removed at any time**.

### Trap 2: GSI Throttling Blocks Base Table Writes

- If a GSI runs out of WCU, writes to the **base table** are **rejected** — not just the GSI write.
- Always provision adequate WCU on GSIs.

### Trap 3: DAX and Strongly Consistent Reads

- DAX only helps with **eventually consistent reads**.
- Strongly consistent reads bypass the DAX cache and go directly to DynamoDB.

### Trap 4: Scan Charges for All Items Read

- `FilterExpression` on a Scan does **not** reduce RCU cost.
- You are charged for every item DynamoDB reads before filtering.
- Solution: use Query instead, or add GSI to query on the filtered attribute.

### Trap 5: On-Demand Mode Switch Limit

- You can switch between Provisioned and On-Demand modes **only once per 24 hours**.

### Trap 6: DynamoDB Streams 24-Hour Retention

- Streams data is only available for **24 hours**.
- If you need longer retention or more consumers, use **Kinesis Data Streams** integration.

### Trap 7: Transactions Cost 2×

- Every transactional read/write costs double the normal RCU/WCU.
- Factor this into capacity calculations.

### Trap 8: PITR Must Be Enabled

- Point-in-Time Recovery is **not enabled by default**.
- Must be explicitly turned on per table.

### Trap 9: Global Tables Require Streams

- Global Tables internally use DynamoDB Streams for replication.
- You must have Streams enabled on the table.

### Trap 10: Item 400 KB Limit

- No single item can exceed **400 KB**.
- Use the Large Object Pattern (store in S3) for larger data.

---

## 18. Exam Quick-Reference Cheat Sheet

| If the question says...                                 | Answer                                          |
| ------------------------------------------------------- | ----------------------------------------------- |
| Unpredictable / spiky traffic                           | On-Demand capacity mode                         |
| Predictable traffic, cost optimize                      | Provisioned + Auto Scaling or Reserved Capacity |
| Microsecond latency / reduce read load                  | DAX                                             |
| Cross-region active-active replication                  | Global Tables                                   |
| Trigger Lambda on data change                           | DynamoDB Streams                                |
| ACID multi-item/multi-table operations                  | TransactWriteItems / TransactGetItems           |
| Query on non-primary-key attribute (add any time)       | GSI                                             |
| Query on non-primary-key attribute (same partition key) | LSI                                             |
| LSI can't be added after creation                       | Confirm: must be at table creation              |
| Auto-expire items, no extra cost                        | TTL                                             |
| Restore to any second in the last 35 days               | PITR (must enable explicitly)                   |
| Take a full backup before a change                      | On-Demand Backup                                |
| Item size > 400 KB                                      | S3 + reference pointer in DynamoDB              |
| Throttling on one partition key                         | Write sharding + exponential backoff            |
| GSI writes throttled → table writes fail                | Increase GSI WCU                                |
| DAX + strongly consistent reads                         | DAX bypassed, reads go to DynamoDB directly     |
| Streams retention > 24 hours                            | Kinesis Data Streams for DynamoDB               |
| Fine-grained per-user access                            | IAM condition: dynamodb:LeadingKeys             |
| Keep traffic off the internet                           | VPC Gateway Endpoint for DynamoDB               |

---

_Good luck on your exam!_
