---
tags:
  - case-study
  - database
  - mvcc
  - postgresql
  - replication
  - scaling
  - system-design
  - vacuum
---

# 📚 Scaling PostgreSQL to 800 Million Users — Learning Guide

> **Source:** [OpenAI Engineering Blog — January 22, 2026](https://openai.com/index/scaling-postgresql/) **Author:** Bohan Zhang, Member of Technical Staff at OpenAI

---

## 🎯 What This Article Is About

OpenAI shares a detailed inside look at how they scaled **a single PostgreSQL primary instance** to handle **millions of queries per second (QPS)** for **800 million ChatGPT users**, with nearly **50 read replicas** spread across multiple global regions — while maintaining **five-nines (99.999%) availability** and **low double-digit millisecond p99 latency**.

The key insight: PostgreSQL can be pushed far beyond what most engineers assume for read-heavy workloads, through disciplined optimization rather than sharding.

---

## 🧠 Prerequisites to Understand This Article

Before diving in, you should be comfortable with:

### Database Fundamentals

- **SQL & relational databases** — tables, indexes, joins, transactions
- **ACID properties** — Atomicity, Consistency, Isolation, Durability
- **Primary vs. Replica** — the difference between a write node and read nodes
- **Write Ahead Log (WAL)** — PostgreSQL's mechanism for durability and replication

### PostgreSQL Specifics

- **MVCC (Multiversion Concurrency Control)** — how Postgres handles concurrent reads/writes by keeping multiple row versions
- **Autovacuum** — the background process that cleans up dead tuples left by MVCC
- **Table/Index Bloat** — disk space wasted by unreclaimed dead tuple versions
- **`idle_in_transaction_session_timeout`** — a config setting to kill stalled transactions

### Distributed Systems Concepts

- **Replication** — streaming data changes from primary to replicas
- **Replication Lag** — the delay between a write on primary and its appearance on replicas
- **High Availability (HA)** — designing systems to survive node failures
- **Hot Standby** — a replica that is always synchronized and ready to immediately take over
- **Failover** — the process of switching traffic from a failed primary to the standby

### Application & Infrastructure Patterns

- **Connection Pooling** — reusing database connections instead of creating new ones per request (tool: PgBouncer)
- **Caching Layer** — storing query results in fast memory (e.g., Redis) to reduce DB reads
- **ORM (Object-Relational Mapping)** — frameworks (like SQLAlchemy, Django ORM) that generate SQL from code; can produce inefficient queries
- **Rate Limiting** — throttling requests to protect a resource from being overwhelmed
- **OLTP (Online Transaction Processing)** — the class of short, frequent read/write queries that applications like ChatGPT rely on
- **SEV (Severity Incident)** — an internal term for production outages, ranked SEV0 (worst) to SEV3

---

## 🏗️ Architecture Overview

OpenAI's PostgreSQL setup is intentionally **not sharded**. The architecture is:

```
                          ┌─────────────────────┐
                          │  PostgreSQL PRIMARY  │
                          │   (single writer)    │
                          │   + HA Hot Standby   │
                          └──────────┬──────────┘
                                     │  WAL Stream
                    ┌────────────────┼────────────────┐
                    ▼                ▼                ▼
             ┌──────────┐    ┌──────────┐    ┌──────────┐
             │ Replica  │    │ Replica  │    │ Replica  │
             │ Region A │    │ Region B │    │ Region C │
             └────┬─────┘    └────┬─────┘    └────┬─────┘
                  │               │               │
             PgBouncer       PgBouncer       PgBouncer
             (connection     (connection     (connection
               pooling)        pooling)        pooling)
                  │               │               │
             App clients    App clients    App clients
```

**Why no sharding?** Sharding existing workloads would require rewriting hundreds of application endpoints, potentially taking months or years. Since workloads are mostly read-heavy, the single-primary approach with many replicas provides enough headroom.

---

## 🔥 The Core Problem: The Vicious Cycle Under Load

When a stressor hits (cache failure, expensive query spike, write storm), this cycle emerges:

```
  Upstream Issue (cache miss storm, expensive query spike, write storm)
          ↓
  DB load spikes → CPU saturation → query latency rises
          ↓
  Requests time out → clients retry
          ↓
  Retry storm amplifies load further
          ↓
  Cascading failure across ChatGPT & API
```

Every optimization in the article is designed to **break this cycle** at different points.

---

## 🛠️ The 8 Key Optimizations — Deep Dive

---

### 1. Reducing Load on the Primary

**The Problem:** The primary is the only writer; write spikes can quickly overwhelm it.

**The Solution:**

- Offload ALL reads to replicas where possible
- Keep only reads that are part of write transactions on the primary (and make those very fast)
- Migrate write-heavy, shardable workloads to **Azure CosmosDB**
- Fix application bugs causing **redundant writes**
- Introduce **lazy writes** — defer non-critical writes to smooth out spikes
- Apply **strict rate limits** when backfilling table fields

**Key Takeaway:** Treat the primary as precious. Every unnecessary read or write you remove from it increases your headroom for real writes.

---

### 2. Query Optimization

**The Problem:** A single expensive query joining 12 tables caused multiple high-severity incidents when it spiked in volume.

**The Solutions:**

- Eliminate or rewrite multi-table joins (12-table join = disaster at scale)
- Move complex join logic to the **application layer** — fetch data in multiple simple queries and join in code
- Audit all ORM-generated SQL; ORMs can silently produce terrible queries
- Configure **`idle_in_transaction_session_timeout`** to kill idle transactions that block autovacuum

**OLTP Anti-Patterns to Avoid:**

|Anti-Pattern|Why It's Dangerous|
|---|---|
|Multi-table joins (5+ tables)|Exponential cost growth under load|
|SELECT *|Fetches unnecessary columns; wider rows = more I/O|
|Missing indexes on WHERE/JOIN columns|Full table scans under load|
|Long-running transactions|Block autovacuum; cause lock contention|
|N+1 query patterns|Generates O(n) queries for one logical request|

---

### 3. Single Point of Failure Mitigation

**The Problem:** One writer = one catastrophic failure point. If the primary dies, all writes fail.

**The Solutions:**

**For Reads:**

- Route the most critical read queries to replicas so they survive a primary failure
- This downgrades a primary failure from **SEV0** (total outage) to a partial degradation

**For Writes:**

- Run the primary in **High Availability (HA) mode** with a **hot standby**
- The standby is continuously synchronized and can be promoted within seconds
- Azure PostgreSQL team specifically hardened failover to be safe under very high load

**For Replicas:**

- Deploy **multiple replicas per region** with capacity headroom
- A single replica failure never causes a regional outage

---

### 4. Workload Isolation

**The Problem:** A new feature launch with inefficient queries can degrade performance for all users ("noisy neighbor" problem).

**The Solution:**

- Split requests into **tiers** (e.g., high-priority vs. low-priority)
- Route each tier to **dedicated replica instances**
- Isolate by **product/service** as well (one product's spike can't impact another's)

```
High-Priority Requests  →  Replica Set A  (e.g., ChatGPT core)
Low-Priority Requests   →  Replica Set B  (e.g., analytics, exports)
Product A Traffic       →  Replica Set C
Product B Traffic       →  Replica Set D
```

**Key Takeaway:** Isolation is a form of blast radius reduction. You're accepting more infrastructure cost to buy reliability guarantees.

---

### 5. Connection Pooling with PgBouncer

**The Problem:** PostgreSQL has a hard connection limit (5,000 in Azure). Connection storms can exhaust this instantly.

**The Solution:** Deploy **PgBouncer** as a proxy between applications and PostgreSQL.

**How PgBouncer Works:**

- Apps connect to PgBouncer (cheap, many connections allowed)
- PgBouncer maintains a small pool of real PostgreSQL connections
- Reuses connections across requests in **statement or transaction pooling mode**

**Results at OpenAI:**

|Metric|Before PgBouncer|After PgBouncer|
|---|---|---|
|Connection setup time|~50ms|~5ms|
|Active DB connections|Thousands (risk of exhaustion)|Managed pool|

**Architecture detail:** Each replica has its own Kubernetes deployment running multiple PgBouncer pods, all behind a Kubernetes Service for load balancing.

**Critical config:** `idle_timeout` in PgBouncer must be tuned carefully to avoid both connection exhaustion and premature connection drops.

---

### 6. Caching

**The Problem:** Cache miss storms (e.g., when the caching layer fails) flood PostgreSQL with reads, triggering the vicious cycle.

**The Solution:** Two layers of defense:

**Layer 1 — Cache most reads:** A caching layer (e.g., Redis) serves the majority of reads, keeping PostgreSQL mostly for writes and cache misses.

**Layer 2 — Cache Locking (Mutex/Lease pattern):**

```
Without cache locking:
  1000 requests miss on key "user:123"
  → 1000 simultaneous DB reads
  → CPU spike → slowdown

With cache locking:
  1000 requests miss on key "user:123"
  → 1 request acquires lock, fetches from DB, repopulates cache
  → 999 requests wait for cache to be populated
  → 1 DB read instead of 1000
```

This pattern is also called **"dog-pile prevention"** or **"thundering herd mitigation"**.

---

### 7. Scaling Read Replicas

**The Problem:** Each new replica requires the primary to stream WAL to it. At ~50 replicas, this consumes significant primary CPU and network bandwidth. You can't scale indefinitely with this fan-out model.

**Current State:** ~50 replicas across multiple geographic regions, replication lag kept near zero.

**Future Solution — Cascading Replication:**

```
Current:
  Primary → Replica 1
  Primary → Replica 2
  Primary → Replica 3  (primary does ALL the WAL shipping work)
  ...
  Primary → Replica 50

Cascading:
  Primary → Intermediate Replica A → Replica 1, 2, 3 ...
  Primary → Intermediate Replica B → Replica 4, 5, 6 ...
  (Primary only ships WAL to intermediate replicas)
```

Cascading replication allows scaling to 100+ replicas without overloading the primary. OpenAI is building this with the Azure PostgreSQL team. It's still in testing because failover management becomes more complex in a multi-tier replication tree.

---

### 8. Rate Limiting

**The Problem:** Traffic spikes, expensive query bursts, or retry storms can exhaust CPU, I/O, and connections all at once.

**The Solution — Multi-Layer Rate Limiting:**

```
Request flow:
  Client
    ↓  [Application-level rate limit]
  App Server
    ↓  [Proxy-level rate limit]
  PgBouncer
    ↓  [Connection-level rate limit]
  PostgreSQL
    ↓  [Query-level rate limit / digest blocking]
  Result
```

Key capabilities:

- **Block specific query digests** — if a specific expensive query pattern appears, block it immediately
- **ORM-layer rate limiting** — rate limit at the point queries are generated
- **Avoid short retry intervals** — exponential backoff is critical to prevent retry storms

---

### 9. Schema Management

**The Problem:** A seemingly innocent `ALTER TABLE` can trigger a full table rewrite, locking the table and causing downtime.

**The Rules OpenAI Enforces:**

|Operation|Allowed?|Notes|
|---|---|---|
|Add column (no default)|✅ Yes|Lightweight in modern PG|
|Drop column|✅ Yes|Lightweight|
|Create index CONCURRENTLY|✅ Yes|Non-blocking|
|Drop index CONCURRENTLY|✅ Yes|Non-blocking|
|Change column type|❌ No|Triggers full table rewrite|
|Add column with non-null default (old PG)|❌ No|Full rewrite on old versions|
|Add new tables to main PostgreSQL|❌ No|New tables must go to CosmosDB|
|Backfill a column without rate limits|❌ No|Can cause write storm|

**Enforcement rules:**

- 5-second **hard timeout** on all schema changes — if it doesn't complete in 5 seconds, it's rolled back
- Backfills are rate-limited even if they take over a week

---

## 📊 Results

|Metric|Result|
|---|---|
|Users served|800 million|
|Scale of PostgreSQL load growth (1 year)|10x|
|Number of read replicas|~50 across multiple regions|
|Replication lag|Near zero|
|Client-side p99 latency|Low double-digit milliseconds|
|Availability|Five-nines (99.999%)|
|SEV-0 incidents (12 months)|1 (during ChatGPT ImageGen launch with 10x write surge)|

---

## 🧩 MVCC Deep Dive (The Root Cause of Write Challenges)

MVCC is why PostgreSQL is great for reads but challenging for heavy writes:

**How MVCC works:**

- When a row is `UPDATE`d, PostgreSQL does NOT modify it in place
- It writes a **new version** of the entire row (the "tuple"), even if only one column changed
- The old version remains on disk as a "dead tuple" until autovacuum cleans it up

**Consequences at scale:**

```
Write Amplification:
  UPDATE users SET last_seen = NOW() WHERE id = 123;
  → Full row copy written to disk (even though only 1 field changed)

Read Amplification:
  SELECT * FROM users WHERE id = 123;
  → Must scan past dead tuples to find the latest live version

Table/Index Bloat:
  Dead tuples accumulate → tables and indexes grow on disk
  → More I/O per query → autovacuum struggles to keep up
```

OpenAI's response: migrate write-heavy workloads away from PostgreSQL to systems better suited for writes (CosmosDB), and heavily tune autovacuum.

---

## 🗺️ Key Mental Models

### The "Treat the Primary as Sacred" Model

Every read and write you can remove from the primary extends its lifespan before you need to shard or redesign. Replicas are cheap; the primary is irreplaceable.

### The "Defense in Depth" Model

No single technique is enough. Layer them:

- Cache → reduces reads reaching DB
- Rate limits → caps damage from any single spike
- Connection pooling → prevents connection exhaustion
- Workload isolation → limits blast radius
- Query optimization → removes the worst offenders

### The "Blast Radius Reduction" Model

Design failures to be partial, not total:

- Route critical reads to replicas → primary failure isn't SEV0
- Isolate workloads → one feature's bug doesn't take down everything
- Multiple replicas per region → one replica failure isn't a regional outage

---

## 🔗 Further Reading

- [The Part of PostgreSQL We Hate the Most](https://www.cs.cmu.edu/~pavlo/blog/2023/04/the-part-of-postgresql-we-hate-the-most.html) — by Bohan Zhang & Prof. Andy Pavlo (CMU); a deep-dive on MVCC challenges cited in the PostgreSQL Wikipedia page
- [PostgreSQL Cascading Replication Docs](https://www.postgresql.org/docs/current/warm-standby.html#CASCADING-REPLICATION)
- [When Does ALTER TABLE Require a Rewrite?](https://www.crunchydata.com/blog/when-does-alter-table-require-a-rewrite) — Crunchy Data
- [Azure PostgreSQL Flexible Server Overview](https://learn.microsoft.com/en-us/azure/postgresql/overview)
- [PgBouncer Documentation](https://www.pgbouncer.org/config.html)

---

## 💡 Quick Reference: Lessons Learned

1. **A single-primary PostgreSQL can scale further than you think** — with enough optimization, replicas, and discipline
2. **Never let a 12-table join reach production** — move join logic to the application layer
3. **ORM-generated SQL must be reviewed** — it's often the source of expensive query patterns
4. **Cache locking is non-negotiable** — without it, cache failures cascade directly into DB failures
5. **Schema changes need a 5-second timeout** — if it takes longer, something is wrong
6. **No new tables in the monolith** — new features get new sharded systems
7. **Retry storms kill databases** — implement exponential backoff and circuit breakers everywhere
8. **Workload isolation is worth the infra cost** — noisy neighbor problems are silent killers
9. **Connection pooling cuts latency 10x** — PgBouncer is non-negotiable at scale
10. **Cascading replication is the path to 100+ replicas** — direct fan-out from primary doesn't scale forever

---

_Guide compiled from the OpenAI Engineering Blog post by Bohan Zhang, January 22, 2026._

## See also

- [[Database Replication]] · [[Scaling Reads vs Writes]] · [[VACUUM]] · [[Isolation Level and MVCC]]
- [[System Design MOC]]
