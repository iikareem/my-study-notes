---
tags:
  - aws
  - certification
  - postgresql
  - rds
  - replication
  - scaling
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[RDS Aurora]] · [[RDS Aurora Exam Guide]]

# AWS Developer Associate — RDS & Aurora

## Exam Revision Guide: Key Concepts & Exam Traps

---

## 1. RDS vs Aurora — The Core Distinction

||RDS (MySQL/PostgreSQL/MariaDB/Oracle/SQL Server)|Aurora (MySQL/PostgreSQL compatible)|
|---|---|---|
|Storage|EBS-backed, single AZ, mirrored to standby in Multi-AZ|Distributed storage layer, **6 copies across 3 AZs**, automatically|
|Storage scaling|Manual (or autoscaling up to a max you set)|Automatic, 10 GiB increments, up to **128 TiB**|
|Failover time|60–120 seconds (Multi-AZ)|**< 30 seconds** (shared storage, no resync needed)|
|Read Replicas|Up to 5, async, can lag|Up to 15 Aurora Replicas, low lag (shared storage)|
|Max read replica lag|Seconds to minutes|Typically < 100ms|
|Backups|Standard, from standby if Multi-AZ|Continuous, streamed to S3|
|Serverless option|No (only Aurora)|**Aurora Serverless v2**|
|Global multi-region|Cross-region Read Replica only|**Aurora Global Database** (sub-second lag)|
|Cloning|Not native (snapshot + restore only)|**Fast Clone** (copy-on-write, minutes)|
|Point-in-time rewind|PITR only (creates new instance)|PITR **+ Backtrack** (in-place rewind)|

> ⚠️ **Exam trap:** Aurora is NOT just "managed MySQL/PostgreSQL" — it's AWS's own storage engine, wire-compatible with MySQL/PostgreSQL. This is why it has different failover, replication, and scaling behavior.

---

## 2. Multi-AZ vs Read Replicas — Don't Confuse Them

||Multi-AZ Standby|Read Replica|
|---|---|---|
|Purpose|**High availability / failover**|**Read scaling**|
|Replication|Synchronous|Asynchronous|
|Can serve reads?|**No** (RDS) — standby is failover-only|**Yes**|
|Promotion|Automatic on failure|Manual (`promote-read-replica`)|
|Endpoint|Same as primary (DNS updates on failover)|Separate endpoint|

> ⚠️ **Exam trap:** A classic wrong-answer pattern is "direct read traffic to the Multi-AZ standby" — **you cannot read from a standard RDS Multi-AZ standby.** Only Read Replicas serve reads.

> ⚠️ You can combine both: Multi-AZ for the primary (HA) + Read Replicas for read scaling. Read Replicas can themselves be Multi-AZ.

---

## 3. Aurora Endpoints — Know All Four

|Endpoint|Points to|Use for|
|---|---|---|
|**Cluster (Writer) Endpoint**|Current primary writer|All writes; reads needing latest data|
|**Reader Endpoint**|Load-balances across all Aurora Replicas|Read scaling (auto-updates as replicas added/removed)|
|**Instance Endpoint**|One specific instance|Pinning a workload (e.g., analytics) to a specific replica|
|**Custom Endpoint**|A defined subset of instances|Workload-specific routing (e.g., reporting replicas only)|

> ⚠️ **Exam trap:** RDS does **not** auto-route reads to replicas — the application must explicitly connect to the Read Replica endpoint. Aurora's Reader Endpoint does load-balance automatically, but only across instances connected through it — direct connections to the Cluster Endpoint always go to the writer.

---

## 4. Aurora Failover Mechanics (High-Yield Topic)

- Failover promotes an Aurora Replica to writer in **under 30 seconds**
- No data loss — replicas share the **same underlying storage** as the writer, so there's no replication lag to catch up on
- **Failover priority tiers (0–15)**: tier 0 = promoted first; tier 15 = promoted last (or never, if avoidable)
- If multiple replicas share the same tier, Aurora picks the one with the **least replication lag**
- Use `failover-db-cluster` API to **test failover safely** without an actual outage

> ⚠️ Aurora Global Database does **NOT auto-fail-over across regions**. A regional disaster requires **manual promotion** of the secondary cluster (managed/unplanned failover) — this is a very common exam trap.

---

## 5. Aurora Global Database vs Cross-Region Read Replica

||Aurora Global Database|Cross-Region Read Replica (RDS or Aurora)|
|---|---|---|
|Replication lag|**< 1 second** (dedicated infra)|Seconds to minutes (binlog-based)|
|Max secondary regions|Up to 5|Limited|
|Failover|Manual promotion|Manual promotion|
|Use case|DR with strict RPO/RTO|Simple cross-region reads/reporting|

> ⚠️ **Exam trap:** If a question mentions "RPO < 1 second" and "multi-region" → answer is almost always **Aurora Global Database**, not cross-region Read Replica.

---

## 6. Aurora Backtrack vs Point-in-Time Recovery (PITR)

||Backtrack|PITR|
|---|---|---|
|Result|Rewinds the **same cluster** in-place|Creates a **NEW** cluster at the target time|
|Speed|Seconds to minutes|Slower (restore process)|
|Engine support|**Aurora MySQL only** (not PostgreSQL)|All RDS & Aurora engines|
|Requires|Must be enabled at cluster creation with a backtrack window (up to 72h)|Backup retention period (1–35 days)|
|Endpoint impact|Same endpoint, no migration needed|New endpoint — must migrate app connection|

> ⚠️ **Exam trap:** Backtrack is **Aurora MySQL only**. If the question is about Aurora PostgreSQL, PITR is the only point-in-time option.

---

## 7. Aurora Serverless v2 (High-Yield)

- Scales in **fine-grained ACU increments**, near-instantaneously, **without pausing connections**
- Min/max ACUs configurable (e.g., 0.5 to 128 ACUs)
- Supports: Multi-AZ, Read Replicas, Global Database, RDS Proxy
- Ideal for: **unpredictable/spiky traffic**, dev/test environments, low-traffic apps wanting to minimize idle cost

### Serverless v1 vs v2

||v1|v2|
|---|---|---|
|Scaling|Coarse steps, can **pause** (cold start)|Fine-grained, **no pause**, near-instant|
|Multi-AZ|No|**Yes**|
|Read Replicas|No|**Yes**|
|Global Database|No|**Yes**|

> ⚠️ **Exam trap:** If a question describes cold starts/pausing, that's v1 behavior — but most current exam questions assume v2, which does NOT pause.

---

## 8. RDS Data API (Easy to Confuse)

- HTTP-based endpoint to run SQL **without managing VPC networking or persistent connections**
- **Only available for Aurora Serverless (v1 and v2)** — NOT for standard provisioned RDS or Aurora provisioned clusters
- Ideal for Lambda functions that don't want to deal with VPC config or connection pooling

> ⚠️ **Exam trap:** If the question says "provisioned Aurora cluster" (not Serverless), the Data API is **not available** — the correct answer would be RDS Proxy + VPC, not Data API.

---

## 9. RDS Proxy — Why & When

**Problem it solves:** Lambda (and other bursty clients) opens a new DB connection per invocation → exhausts `max_connections` → "Too many connections" errors.

**What it does:**

- Pools and multiplexes connections — many app-level connections share fewer real DB connections
- Reduces failover time for applications (RDS Proxy handles reconnection logic)
- Works with RDS and Aurora (MySQL, PostgreSQL)
- Requires the proxy to be **in the same VPC**

> ⚠️ **Exam trap:** RDS Proxy is the answer whenever you see "Lambda + RDS/Aurora + connection exhaustion" or "Too many connections" in a question.

---

## 10. Credentials & Authentication

### Three ways to authenticate to RDS/Aurora

|Method|How it works|Best for|
|---|---|---|
|**Secrets Manager**|Stores username/password, auto-rotates via Lambda|Most general use cases — AWS recommended default|
|**IAM Database Authentication**|App generates a 15-min token via `generate_db_auth_token()`, used as the password (SSL required)|Avoiding password management entirely; short-lived token security|
|**Plaintext in code/env vars**|❌ Anti-pattern|Never — exam will flag this as wrong|

> ⚠️ **Exam trap:** IAM DB Auth requires `rds-db:connect` IAM permission AND the connection must use SSL. It is NOT the same as an IAM user's access key — it generates a temporary signed token.

---

## 11. Encryption — Rules You Must Memorize

- Encryption at rest **must be enabled at creation time** — cannot be added to an existing unencrypted instance/cluster
- To "encrypt" an existing unencrypted DB: snapshot → copy snapshot with encryption enabled → restore from encrypted snapshot
- Once encrypted, **all derived snapshots/replicas remain encrypted** — you cannot create an unencrypted copy from encrypted data
- Encryption in transit (SSL/TLS) is **separate** from encryption at rest (KMS) — enabling one does NOT enable the other
    - PostgreSQL: `rds.force_ssl = 1` parameter
    - MySQL: require SSL via parameter group / GRANT statements

> ⚠️ **Exam trap:** "Enable encryption on a running unencrypted instance" → always requires snapshot + copy + restore. There is no in-place toggle.

---

## 12. Backups — Automated vs Manual

||Automated Backups|Manual Snapshots|
|---|---|---|
|Retention|1–35 days (0 = disabled)|Until you delete them|
|Enables PITR|Yes|No (snapshot = single point)|
|Deleted when instance deleted?|**Yes** (unless final snapshot taken)|No, persists independently|
|Performance impact|None on Multi-AZ (taken from standby); brief I/O suspension on Single-AZ|Minimal|
|Cross-account sharing|Must copy to manual snapshot first|Directly shareable (up to 20 accounts)|

> ⚠️ **Exam trap:** If retention is set to **0**, automated backups AND PITR are disabled. To recover beyond the retention window, you need a manual snapshot taken before that window — automated backups won't help.

---

## 13. Cross-Account / Cross-Region Operations

- **RDS does NOT support cross-account Read Replicas natively**
- Standard workaround: Read Replica → snapshot → share snapshot with target account → restore there → use DMS for ongoing CDC sync if needed
- **Aurora snapshots** can be shared with up to 20 AWS accounts by modifying snapshot permissions
- **Snapshot Export to S3**: exports snapshot data to S3 in **Apache Parquet** format — directly queryable via Athena/Redshift Spectrum/EMR, no ETL needed

---

## 14. Networking & Connectivity

- RDS/Aurora instances in private subnets: Lambda must be **deployed into the same VPC** to reach them — Lambda outside a VPC has no route to private subnets
- Security group rules should reference the **source security group ID** (not just CIDR) for dynamic environments like Lambda/EC2 fleets
- After Multi-AZ failover, **DNS caching at the OS/JVM level** can cause reconnect delays even if RDS's TTL is short — fix with `networkaddress.cache.ttl=0` (Java) or proper connection pool retry logic

---

## 15. Performance Troubleshooting Cheat Sheet

|Symptom|Cause|Fix|
|---|---|---|
|`IO:DataFileRead` wait events (Performance Insights)|Data not in buffer pool, disk reads dominate|Bigger instance (more RAM), add Read Replicas, optimize queries/indexes|
|`FreeStorageSpace` declining rapidly|Data growth outpacing storage|Enable **Storage Autoscaling** (RDS) — Aurora scales automatically|
|Maxed out IOPS on gp2|Sustained high I/O workload|Move to **io1/io2** (Provisioned IOPS) or **gp3**|
|"Too many connections"|Connection churn (e.g., Lambda) exceeds `max_connections`|**RDS Proxy**|
|High CPU from batch/reporting job|Single primary handling OLTP + analytics|Run batch against a **Read Replica**, not primary or Multi-AZ standby|
|Slow queries|Missing indexes / inefficient SQL|Enable **slow query log**, use **Performance Insights** to find top SQL by DB load|

---

## 16. Migration Tools

|Tool|Purpose|
|---|---|
|**AWS DMS**|Data migration with **CDC** (ongoing replication) for minimal downtime cutover|
|**AWS SCT (Schema Conversion Tool)**|Converts schema/stored procedures between different engines (e.g., Oracle → Aurora PostgreSQL) — used **together** with DMS for heterogeneous migrations|
|**Snapshot export to S3**|One-time analytical export, not for live migration|

> ⚠️ **Exam trap:** DMS handles **data**. SCT handles **schema/code**. For a heterogeneous migration (different engine types), you need BOTH.

---

## 17. Aurora Storage Quorum Model (Common Trick Question)

- Aurora replicates data **6 ways across 3 AZs** (2 copies per AZ)
- **Write quorum**: 4 out of 6 copies must acknowledge
- **Read quorum**: 3 out of 6 copies must acknowledge
- This means Aurora can lose an entire AZ (2 copies) + 1 more copy and still accept writes; it can lose an entire AZ and still serve reads

---

## 18. Quick-Hit Facts Often Tested

- **RDS Read Replica limit**: up to 5 per primary (MySQL/PostgreSQL/MariaDB); Aurora supports up to 15 Aurora Replicas
- **Aurora storage max**: 128 TiB, auto-grows in 10 GiB increments
- **Backup retention range**: 1–35 days (0 disables automated backups & PITR)
- **Backtrack window**: up to 72 hours, Aurora MySQL only
- **IAM DB Auth token validity**: 15 minutes
- **Multi-AZ failover time**: typically 60–120 seconds
- **Aurora failover time**: typically under 30 seconds
- **Major version upgrades**: irreversible — always snapshot + test first
- **Aurora binlog**: needed for CDC/Kafka Connect streaming of row-level changes; `binlog_format=ROW` recommended for CDC
- **Aurora Multi-Master**: legacy, MySQL 5.6 only, conflict resolution falls on the application — generally **not recommended**, prefer Serverless v2 for write scaling needs

---

## 🧠 Top 10 RDS/Aurora Exam Traps to Remember

1. **Standard RDS Multi-AZ standby cannot serve reads** — only Read Replicas can
2. **RDS does not auto-route reads to replicas** — the app must use the replica's endpoint explicitly
3. **Encryption at rest cannot be enabled on a running unencrypted instance** — requires snapshot → encrypted copy → restore
4. **Aurora Global Database does NOT auto-failover across regions** — manual promotion required
5. **Backtrack is Aurora MySQL only** — not available for Aurora PostgreSQL
6. **RDS Data API only works with Aurora Serverless**, not provisioned Aurora or standard RDS
7. **RDS has no native cross-account Read Replicas** — must use snapshot sharing + DMS
8. **Backup retention = 0 disables PITR entirely**, not just reduces it
9. **DMS = data, SCT = schema** — heterogeneous migrations need both
10. **Aurora write quorum = 4/6, read quorum = 3/6** — commonly tested number

---

_Good luck — combine this with the RDS/Aurora practice exam to reinforce these concepts!_ 🚀
