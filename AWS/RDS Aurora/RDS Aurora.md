---
tags:
  - aws
  - certification
  - database
  - indexing
  - postgresql
  - rds
  - scaling
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[RDS Quiz]] · [[RDS Aurora Exam Guide]] · [[RDS Aurora Full Guide]] · [[RDS Aurora Cheatsheet]] · [[Aurora Exam]]

# Amazon RDS & Aurora — Complete Knowledge Guide

---

## Table of Contents

**Part I — Understanding the Landscape**

1. [The Confusion Explained — RDS vs Aurora in Plain Terms](#1-the-confusion-explained--rds-vs-aurora-in-plain-terms)
2. [What is Amazon RDS?](#2-what-is-amazon-rds)
3. [What is Amazon Aurora?](#3-what-is-amazon-aurora)
4. [Side-by-Side Comparison](#4-side-by-side-comparison)
5. [When to Choose RDS vs Aurora](#5-when-to-choose-rds-vs-aurora)

**Part II — Amazon RDS Deep Dive**

6. [RDS Architecture](#6-rds-architecture)
7. [RDS Database Engines](#7-rds-database-engines)
8. [RDS Instance Classes](#8-rds-instance-classes)
9. [RDS Storage](#9-rds-storage)
10. [RDS Multi-AZ](#10-rds-multi-az)
11. [RDS Read Replicas](#11-rds-read-replicas)
12. [RDS Backups and Snapshots](#12-rds-backups-and-snapshots)
13. [RDS Security](#13-rds-security)
14. [RDS Parameter Groups and Option Groups](#14-rds-parameter-groups-and-option-groups)
15. [RDS Proxy](#15-rds-proxy)
16. [RDS Maintenance and Upgrades](#16-rds-maintenance-and-upgrades)

**Part III — Amazon Aurora Deep Dive**

17. [Aurora Architecture — The Fundamental Difference](#17-aurora-architecture--the-fundamental-difference)
18. [Aurora Storage Engine](#18-aurora-storage-engine)
19. [Aurora Cluster Anatomy](#19-aurora-cluster-anatomy)
20. [Aurora Endpoints](#20-aurora-endpoints)
21. [Aurora Replicas](#21-aurora-replicas)
22. [Aurora Global Database](#22-aurora-global-database)
23. [Aurora Serverless](#23-aurora-serverless)
24. [Aurora Multi-Master](#24-aurora-multi-master)
25. [Aurora Backups and Backtrack](#25-aurora-backups-and-backtrack)
26. [Aurora Security](#26-aurora-security)
27. [Aurora Parallel Query](#27-aurora-parallel-query)

**Part IV — Shared Concepts**

28. [Networking and VPC Placement](#28-networking-and-vpc-placement)
29. [High Availability Patterns](#29-high-availability-patterns)
30. [Monitoring and Observability](#30-monitoring-and-observability)
31. [Performance Insights](#31-performance-insights)
32. [Limits and Quotas](#32-limits-and-quotas)
33. [Pricing Model](#33-pricing-model)
34. [Best Practices](#34-best-practices)
35. [Common Failure Modes and How to Think About Them](#35-common-failure-modes-and-how-to-think-about-them)

---

# Part I — Understanding the Landscape

---

## 1. The Confusion Explained — RDS vs Aurora in Plain Terms

The confusion is completely understandable, and it comes from AWS's own naming. Here is the mental model that clears it up:

**Amazon RDS** is the **service**. It is AWS's managed relational database platform. When you use RDS, AWS handles patching, backups, failover, replication, and hardware — you just run SQL.

**Amazon Aurora** is a **database engine** that runs on top of the RDS service. It is an AWS-built, cloud-native database engine. You manage Aurora through the RDS service — the same console, same CLI commands, same APIs.

So the question is not "RDS or Aurora?" as if they are opposites. The question is really: **which database engine do I want to run on the RDS managed platform?**

Your choices are:

- MySQL (community edition, on RDS)
- PostgreSQL (community edition, on RDS)
- MariaDB (on RDS)
- Oracle (on RDS)
- SQL Server (on RDS)
- Aurora MySQL (AWS-built engine, MySQL-compatible, on RDS)
- Aurora PostgreSQL (AWS-built engine, PostgreSQL-compatible, on RDS)

When people say "RDS vs Aurora," they typically mean "community MySQL/PostgreSQL on RDS" vs "Aurora MySQL/Aurora PostgreSQL." Aurora is AWS's proprietary re-engineering of the database engine itself — it is not just MySQL or PostgreSQL running on managed infrastructure. It is a fundamentally different architecture that happens to speak the MySQL and PostgreSQL wire protocols.

The rest of this guide uses "RDS" to mean the non-Aurora engines (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server) and "Aurora" to mean Aurora MySQL and Aurora PostgreSQL specifically.


### 1. Amazon RDS = The "As-Is" Community Edition

When you spin up a MySQL database in RDS, AWS is basically taking the standard **Community Edition** of MySQL and installing it for you on an EC2 virtual instance.

- **The Code:** It is the exact same open-source code you would download onto your laptop.
    
- **The Engine:** It uses traditional community storage engines (like InnoDB for MySQL).
    
- **The AWS Part:** AWS simply manages the tedious stuff—the automated backups, the OS patching, and clicking a button to add a Multi-AZ standby.
    

### 2. Amazon Aurora = Modified Code + Custom AWS Engine

Aurora is **fully compatible** with MySQL and PostgreSQL, meaning your application code won't know the difference. But under the hood, it is not just the basic community edition.

- **The Code:** AWS took the open-source MySQL and PostgreSQL code and modified it. They stripped out the parts that handle writing data to traditional hard drives.
    
- **The Engine:** They replaced the standard community storage engine with a **custom, proprietary AWS cloud storage engine**.

---

## 2. What is Amazon RDS?

Amazon RDS (Relational Database Service) is a managed service that automates the operational work of running a relational database:

- Provisioning the underlying hardware and compute
- Installing and patching the database software
- Taking automated backups
- Enabling encryption at rest and in transit
- Managing Multi-AZ failover (high availability)
- Scaling compute and storage (with some caveats)
- Creating read replicas

What RDS does NOT do:

- Write your queries or optimize your schema
- Eliminate the need to understand your database engine
- Make every operational problem disappear

	The engines available on RDS are community editions of well-known databases. When you run MySQL on RDS, it is standard MySQL — the same behavior, the same features, the same quirks. AWS is simply removing the operational burden of running it yourself on EC2.

This is RDS's primary value proposition: you get the database you already know (MySQL, PostgreSQL, Oracle, SQL Server) without managing servers, storage, patching, or backups.

---

## 3. What is Amazon Aurora?

Aurora is AWS's ground-up re-engineering of a relational database for cloud infrastructure. AWS built Aurora because they observed that traditional database engines were designed for the constraints of physical hardware — spinning disks, single-machine I/O, synchronous replication over slow links. The cloud fundamentally changed those constraints.

Aurora's design goals:

- Decouple compute from storage
- Distribute storage across many nodes for fault tolerance
- Reduce the impact of replication on write performance
- Make failover faster and more transparent
- Scale reads independently and cheaply

Aurora is wire-protocol compatible with MySQL and PostgreSQL. This means existing applications, ORMs, drivers, and tools that work with MySQL work with Aurora MySQL. Same for PostgreSQL. You do not change SQL syntax, connection libraries, or query structure.

What changes is everything underneath the SQL layer — the storage engine, the replication mechanism, the I/O path, the cluster architecture. Aurora is deeply different from MySQL and PostgreSQL at the infrastructure level, even though it looks identical from the application level.

Aurora is the answer to: "I want the compatibility of MySQL/PostgreSQL, but I want AWS to optimize the architecture specifically for cloud-scale reliability and performance."

---

## 4. Side-by-Side Comparison

### Architecture

|Dimension|RDS (MySQL/PostgreSQL)|Aurora|
|---|---|---|
|Storage model|EBS volume per instance|Distributed cluster volume shared across all instances|
|Replication|Binary log replication (async or semi-sync)|Storage-level replication (no binlog overhead on writes)|
|Write path|Write to EBS, replicate log to standby|Write to distributed storage (6 copies across 3 AZs), acknowledge when 4/6 confirm|
|Storage scaling|Manual or auto-scaling EBS|Automatic, up to 128 TB per cluster|
|Max storage|64 TB (varies by engine)|128 TB|
|Read replicas|Up to 5|Up to 15|
|Replica lag|Minutes (async replication)|Typically < 100ms (storage-level replication)|
|Failover time|60–120 seconds|Typically < 30 seconds|

### Compatibility

|Dimension|RDS MySQL|Aurora MySQL|RDS PostgreSQL|Aurora PostgreSQL|
|---|---|---|---|---|
|Wire protocol|MySQL|MySQL|PostgreSQL|PostgreSQL|
|Supported versions|5.7, 8.0|MySQL 5.7-compatible, 8.0-compatible|13, 14, 15, 16|PostgreSQL 13, 14, 15, 16|
|Application compatibility|Full|Near-full (minor differences)|Full|Near-full (minor differences)|
|Engine features|All community features|Most community features + Aurora-specific|All community features|Most community features + Aurora-specific|

### Performance

|Dimension|RDS MySQL|Aurora MySQL|RDS PostgreSQL|Aurora PostgreSQL|
|---|---|---|---|---|
|Write throughput|Baseline|Up to 5x MySQL (AWS benchmark)|Baseline|Up to 3x PostgreSQL (AWS benchmark)|
|Read throughput|Scales with read replicas|Scales with up to 15 low-lag replicas|Scales with read replicas|Scales with up to 15 low-lag replicas|
|Max IOPS|~80,000 (io1)|Higher (distributed I/O)|~80,000 (io1)|Higher|

Take AWS's "5x MySQL" benchmark with appropriate skepticism. Real-world gains depend heavily on workload type. Write-heavy OLTP workloads benefit most. Read-heavy workloads may see modest improvement.

### Availability

|Dimension|RDS|Aurora|
|---|---|---|
|Multi-AZ|1 standby instance (hot standby, not readable)|All replicas are in different AZs by default and are readable|
|Failover|DNS record update, ~60–120 seconds|DNS record update, ~30 seconds typical, < 120 seconds guaranteed|
|Data durability|Replication to one standby|6 copies of data across 3 AZs|
|Self-healing|Limited|Storage layer automatically repairs corrupt/lost blocks by copying from other replicas|

### Features Unique to Aurora

- **Aurora Serverless**: Automatically scales compute capacity up and down based on demand, including scaling to zero (pay nothing when idle)
- **Aurora Global Database**: Single database spanning multiple AWS regions with < 1 second replication lag
- **Aurora Backtrack**: Rewind the database to a previous point in time without restoring from a backup (Aurora MySQL only)
- **Aurora Parallel Query**: Pushes analytical query processing down to the distributed storage layer for faster analytics on operational data
- **Aurora Multi-Master**: Multiple writer nodes (limited availability)
- **Cluster endpoint abstraction**: Multiple endpoint types for different traffic patterns

### Features Unique to RDS (Not in Aurora)

- **Oracle and SQL Server engines**: Aurora only supports MySQL and PostgreSQL compatibility
- **Bring your own license (BYOL)**: For Oracle and SQL Server licensing
- **Some engine-specific features**: Certain MySQL and PostgreSQL features have minor behavioral differences in Aurora

### Cost

|Dimension|RDS|Aurora|
|---|---|---|
|Instance pricing|Lower per-instance cost|Higher per-instance cost (~20% more)|
|Storage pricing|You provision storage size and IOPS|Pay for actual storage used (per GB-month) + I/O requests|
|Read replica cost|Each replica has a full instance + storage cost|Each replica has instance cost only (shared storage)|
|Serverless option|No|Yes (Aurora Serverless v2)|

Aurora can be cheaper or more expensive than RDS depending on the workload. High-write workloads paying for I/O requests can make Aurora more expensive than RDS with provisioned IOPS. Read-heavy workloads with many replicas are cheaper on Aurora because replicas share storage.

---

## 5. When to Choose RDS vs Aurora

### Choose RDS (non-Aurora) when:

**You need Oracle or SQL Server.** Aurora only supports MySQL and PostgreSQL compatibility. For Oracle or SQL Server, RDS is your only managed option on AWS.

**You have a small, predictable workload with cost as the primary concern.** RDS instances are cheaper than Aurora instances. For a low-traffic application where performance headroom is not needed, RDS is the right economic choice.

**You need specific community database features that Aurora does not support.** Aurora has minor compatibility gaps. If your application relies on specific MySQL or PostgreSQL behaviors that Aurora does not implement, RDS gives you the full community engine.

**You are migrating an existing on-premises database with minimal changes.** RDS provides the most 1:1 compatibility with the community engines, minimizing migration risk.

**You have a development/test environment.** RDS is cheaper for non-production databases where Aurora's high availability and performance are unnecessary.

### Choose Aurora when:

**You need high availability with fast failover.** Aurora's 30-second failover and six-way storage replication make it significantly more resilient than RDS Multi-AZ.

**You have high write throughput requirements.** Aurora's storage-layer replication eliminates the write overhead of binary log replication, enabling higher sustained write throughput.

**You need many read replicas with minimal lag.** Up to 15 Aurora Replicas share the same storage volume, so there is no replication lag from disk I/O — just network propagation. RDS read replicas replicate data through the binary log, introducing lag under high write load.

**You need a multi-region database.** Aurora Global Database is the best managed solution for cross-region active-passive replication with sub-second lag.

**You have variable or unpredictable traffic.** Aurora Serverless v2 scales compute automatically. Pay for what you use, scale to zero when idle.

**You are building a new application and are choosing MySQL or PostgreSQL anyway.** For new applications without legacy constraints, Aurora's benefits (availability, scale, features) outweigh the modest cost premium.

**You need storage that scales automatically.** Aurora storage grows automatically as data grows, up to 128 TB. With RDS, you must provision or manually trigger storage scaling.

---

# Part II — Amazon RDS Deep Dive

---

## 6. RDS Architecture

RDS is a managed platform that runs a database engine on a dedicated compute instance backed by EBS (Elastic Block Store) storage. The core architecture is straightforward: one primary instance reads and writes to an EBS volume.

The primary instance is a single EC2-like compute node running the database engine. AWS manages the operating system, the database software installation, patching, and monitoring. You interact with it through a standard database connection (endpoint, port, username, password or IAM).

When you enable Multi-AZ, RDS provisions a second instance in a different Availability Zone. This standby instance maintains a synchronized copy of the data through synchronous replication. The standby is not accessible for reads — it exists solely for failover purposes.

Storage is EBS — a network-attached block storage volume. The instance mounts the EBS volume and treats it like a local disk. This has important performance implications: all I/O goes over the network between the instance and the EBS volume, and EBS has its own throughput and IOPS limits that can constrain database performance.

This architecture is essentially "running a database on a server," with AWS handling the server management. It is simple, well-understood, and maps directly to how databases have been run for decades.

---

## 7. RDS Database Engines

### MySQL on RDS

The most popular RDS engine. Supports MySQL 5.7 and 8.0. Full community MySQL feature set including InnoDB storage engine, JSON support, window functions (8.0), common table expressions (8.0), and all standard MySQL replication features.

Considerations: MySQL 5.7 reaches end of life — plan migration to 8.0. MySQL 8.0 has significant changes from 5.7 (authentication defaults, character set defaults, removed features).

### PostgreSQL on RDS

A full-featured PostgreSQL with support for versions 13 through 16. PostgreSQL's rich feature set is fully available: extensions (PostGIS, pgvector, pg_stat_statements, etc.), advanced indexing (GiST, GIN, BRIN), JSON/JSONB, full-text search, table inheritance, foreign data wrappers.

RDS PostgreSQL supports many PostgreSQL extensions, but not all. The list of supported extensions is version-specific and maintained by AWS.

### MariaDB on RDS

MariaDB is a community fork of MySQL with additional features and different storage engines. The data formats are compatible with MySQL for migration purposes, but MariaDB has diverged significantly in features and behavior.

### Oracle on RDS

Oracle Database Enterprise Edition and Standard Edition 2 on managed infrastructure. Two licensing models:

**License Included (LI)**: AWS includes the Oracle license in the hourly price. Convenient, but more expensive per hour.

**Bring Your Own License (BYOL)**: You provide your existing Oracle licenses and pay only for the RDS infrastructure. Requires Oracle license compliance on your part.

Oracle on RDS has significant restrictions compared to running Oracle on-premises or on EC2. Many DBA-level features (custom tablespace management, certain Oracle features) are restricted or unavailable because AWS manages the OS and DBA-level access.

### SQL Server on RDS

Microsoft SQL Server (Express, Web, Standard, Enterprise editions) with License Included or BYOL licensing. Supports SQL Server authentication and Windows Authentication.

SQL Server on RDS uses a single-AZ architecture by default. Multi-AZ uses SQL Server Database Mirroring or Always On Availability Groups internally.

---

## 8. RDS Instance Classes

Instance classes define the compute (CPU and memory) available to the database. They follow the standard AWS instance naming conventions.

### Instance Class Families

**Standard (m classes)**: General-purpose compute/memory balance. The m6g, m7g (Graviton-based), m6i, m7i families. Suitable for most production workloads.

**Memory Optimized (r classes)**: Higher memory-to-CPU ratio. The r6g, r7g, r6i, r7i families. Better for memory-intensive workloads — databases with large working sets that benefit from fitting more data in the buffer pool.

**Burstable (t classes)**: The t3, t4g families. Low-cost instances with burstable CPU performance. Use only for development, test, or very low-traffic production workloads. CPU credit exhaustion under sustained load causes severe performance degradation.

**I/O Optimized**: For workloads requiring extreme IOPS — typically large enterprise or very high-throughput databases.

### Graviton Instances (g suffix)

ARM-based AWS Graviton processors. Typically offer better price/performance than Intel/AMD equivalents — 10–20% more throughput at 5–10% lower cost for most database workloads. Fully supported for MySQL, PostgreSQL, MariaDB, and Aurora.

### Sizing Considerations

Database sizing is nuanced. The most important factors:

**Buffer pool / shared buffer size**: The database engine caches frequently accessed data in memory. The larger the buffer pool (MySQL) or shared_buffers (PostgreSQL), the more data fits in RAM and the fewer disk reads occur. In general, the working set (the data actively queried) should fit in memory. If it does not, you will see high read IOPS and slower queries.

**CPU**: Database CPU usage is driven by concurrent connections, query complexity, sorting, hashing, and connection overhead. CPU is less often the bottleneck than memory or I/O, but complex analytical queries can be CPU-bound.

**Connections**: Each database connection consumes memory. For PostgreSQL, each connection is a separate process (~10 MB). High connection counts exhaust memory before CPU.

---

## 9. RDS Storage

### Storage Types

**General Purpose SSD (gp2 and gp3)**: The standard storage type.

- **gp2**: Performance scales with volume size (3 IOPS per GB, minimum 100 IOPS, maximum 16,000 IOPS). Burstable to 3,000 IOPS for volumes under 1 TB. Tightly couples storage size to IOPS — to get more IOPS, you must increase volume size.
    
- **gp3**: Decoupled sizing — you can independently configure storage size and IOPS (and throughput). Baseline 3,000 IOPS regardless of volume size. More cost-efficient than gp2 for most workloads. AWS recommends migrating from gp2 to gp3.
    

**Provisioned IOPS SSD (io1 and io2)**: For workloads that need guaranteed, consistent high IOPS. You specify the IOPS independently of the volume size. Maximum 256,000 IOPS with io2 Block Express. Significantly more expensive than gp3.

**Magnetic**: Legacy, not recommended for new deployments.

### Storage Auto Scaling

RDS supports automatic storage scaling. When enabled, if the free storage space falls below a threshold, RDS automatically increases the volume size (up to the maximum you configure). This prevents running out of disk space, which would make the database unavailable.

Auto-scaling is purely size-based — it does not increase IOPS. If you need more IOPS, you must modify the instance manually.

### The Storage Size Relationship

For gp2 storage, the coupling between size and IOPS is a common source of confusion. A 100 GB gp2 volume gives 300 IOPS. A 1,000 GB volume gives 3,000 IOPS. If your workload needs 3,000 IOPS but only 100 GB of data, you still need to provision 1,000 GB. gp3 resolves this by decoupling them.

### Storage Modification Behavior

Increasing RDS storage size is always online (no downtime). However, after a storage modification, RDS enters an optimization phase that can last hours to days during which you cannot modify storage again. Plan storage sizing carefully — it is much easier to start with appropriate sizing than to frequently resize.

Decreasing storage size is not supported. You cannot shrink an RDS volume.

---

## 10. RDS Multi-AZ

### What Multi-AZ Is

Multi-AZ is RDS's high availability feature. When enabled, RDS maintains a synchronous standby replica in a different Availability Zone. The standby is a complete copy of the primary instance, including all data.

The standby is invisible to applications. It has no endpoint. You cannot connect to it, query it, or use it for reads. It exists solely as a failover target.

### How Replication Works

Every write committed on the primary is synchronously replicated to the standby before the transaction is acknowledged to the application. This ensures zero data loss on failover — the standby always has every committed transaction.

Synchronous replication adds latency to writes. The primary must wait for the standby to confirm receipt before acknowledging the write. Over a fast inter-AZ network link, this overhead is typically a few milliseconds but is measurable under high write load.

### Failover Mechanism

When RDS detects that the primary is unhealthy (hardware failure, AZ failure, network issue, operating system crash), it promotes the standby to primary and updates the DNS record for the database endpoint to point to the new primary.

Failover takes 60–120 seconds:

- Detection: 60 seconds or less
- DNS TTL propagation: depends on application DNS caching
- Connection re-establishment: depends on application connection handling

Applications must be designed to handle brief connection failures and retry — the endpoint DNS name stays the same, but it resolves to a different IP after failover. Applications that cache IP addresses (bypassing DNS TTL) will fail to reconnect until they re-resolve DNS.

### What Multi-AZ Does NOT Protect Against

- Query errors or data corruption: if a bad query deletes data, the deletion is replicated to the standby
- Slow queries: the standby mirrors the primary's performance issues
- Region failure: both AZs are in the same region

### Multi-AZ for SQL Server

SQL Server on RDS uses a different internal mechanism — Database Mirroring or Always On Availability Groups. The behavior from the application perspective is the same (automatic failover, DNS update), but the underlying implementation differs.

---

## 11. RDS Read Replicas

### Purpose

Read replicas exist to scale read traffic. They are separate database instances that replicate data from the primary and serve read queries. By directing read traffic (reports, analytics, dashboards, search queries) to replicas, you reduce load on the primary and allow it to focus on writes.

### How Replication Works

RDS read replicas use **asynchronous replication** — unlike Multi-AZ's synchronous replication. The primary commits writes and acknowledges them to the application immediately, then replicates to replicas in the background.

This means replicas may lag behind the primary. Replication lag is the delay between a write being committed on the primary and becoming visible on a replica. Lag can be milliseconds under light load or minutes under heavy write load.

Applications that read from replicas must tolerate eventual consistency. If a user writes data and immediately reads it back, they may read from a replica that hasn't received the write yet. This is a fundamental design constraint when using read replicas.

### Replica Topology

You can create up to 5 read replicas for MySQL, PostgreSQL, and MariaDB (15 for Aurora). Replicas can be in the same region, a different AZ in the same region, or a different region (cross-region read replicas).

Replicas can be promoted to standalone primary instances in disaster recovery scenarios. Promotion is irreversible — a promoted replica becomes an independent database.

Replicas can themselves have replicas (replica chaining), but each replication hop adds lag.

### Cross-Region Read Replicas

Cross-region replicas serve two purposes:

- Low-latency reads for users in other regions
- Disaster recovery — if the primary region fails, promote a cross-region replica

Cross-region replication uses the public internet or AWS backbone and incurs data transfer costs.

---

## 12. RDS Backups and Snapshots

### Automated Backups

RDS automatically backs up the database daily during a configurable backup window. The backup is stored in S3 (managed by AWS, not visible in your S3 buckets).

Retention period: 0 to 35 days. Setting retention to 0 disables automated backups. The retention period determines how far back you can perform a point-in-time restore.

How it works: RDS takes a full snapshot daily and continuously archives transaction logs. Point-in-time restore works by restoring the snapshot and replaying transaction logs up to the target time. This allows restore to any second within the retention period.

The backup window is a daily time range (e.g., 03:00–04:00 UTC). During the backup, brief I/O performance degradation may occur (depending on storage type and instance class). This is minimized on Multi-AZ instances because the backup is taken from the standby.

### Manual Snapshots

You can take snapshots on demand at any time. Unlike automated backups, manual snapshots do not expire — they persist until explicitly deleted. Use manual snapshots before major changes (engine upgrades, large data migrations, schema changes) as a restore point.

Snapshots are stored in S3 and can be copied to other regions for cross-region disaster recovery or to share with other AWS accounts.

Restoring a snapshot always creates a new RDS instance — you do not restore in place. After restoring, you must update your application's connection string to point to the new instance.

### Transaction Log Backups

For continuous backup, RDS backs up transaction logs every 5 minutes. This is what enables point-in-time restore to any second, not just the daily snapshot time.

### Backup Storage Costs

Automated backup storage equal to the provisioned database storage is included at no extra charge. Storage beyond that is charged per GB-month.

---

## 13. RDS Security

### Network Isolation (VPC)

Every RDS instance lives inside a VPC (Virtual Private Cloud). You place it in a DB Subnet Group — a collection of subnets across multiple AZs. The instance is assigned a private IP in that subnet.

For most production databases, the instance should be in private subnets (no route to the internet gateway). Only your application servers (in the same VPC or connected VPCs) can reach the database.

Public accessibility is an option but is strongly discouraged for production. If enabled, the instance gets a public IP and can be reached from the internet — this is a significant security risk.

### Security Groups

Security groups act as a stateful firewall at the instance level. The RDS instance's security group should only allow inbound traffic on the database port (3306 for MySQL, 5432 for PostgreSQL) from the security group(s) of your application servers.

This means: even if someone knows your database endpoint, they cannot connect unless they are in an EC2 instance (or other resource) whose security group is permitted.

### Encryption at Rest

RDS supports encryption of the database volume, automated backups, snapshots, and read replicas using AWS KMS keys.

Encryption must be enabled at creation time — you cannot enable encryption on an existing unencrypted instance directly. The workaround is: take a snapshot of the unencrypted instance, copy the snapshot with encryption enabled, restore a new encrypted instance from the encrypted snapshot.

All replicas of an encrypted instance are automatically encrypted. Cross-region snapshot copies can use different KMS keys in the destination region.

### Encryption in Transit

RDS supports TLS/SSL for all connections. You download an AWS certificate bundle and configure your database client to verify the server certificate. Depending on the engine and configuration, TLS can be required (reject connections without TLS) or optional.

### IAM Database Authentication

Instead of username/password, applications can authenticate using AWS IAM credentials (temporary tokens). The application generates an authentication token using IAM credentials, presents it as the database password. The token expires after 15 minutes.

Benefits: no long-lived database passwords, token rotation is automatic, integrates with IAM roles (no credentials in code or config files).

Supported for MySQL and PostgreSQL.

### Secrets Manager Integration

RDS integrates with AWS Secrets Manager to store and automatically rotate database credentials. Rotation happens without application downtime — Secrets Manager creates a new credential, validates it, updates the secret, then invalidates the old credential. Applications retrieve credentials from Secrets Manager at connection time.

---

## 14. RDS Parameter Groups and Option Groups

### Parameter Groups

A parameter group is a collection of database engine configuration settings. Instead of editing configuration files directly (which you cannot do on a managed service), you create a parameter group and assign it to an instance.

Parameters control fundamental database behavior:

- Buffer pool size (innodb_buffer_pool_size for MySQL)
- Maximum connections (max_connections)
- Query timeout settings
- Character set and collation
- Logging settings (slow query log, general log)
- InnoDB settings (innodb_flush_log_at_trx_commit, innodb_io_capacity)
- Replication settings

**Static parameters** require a database restart to take effect. **Dynamic parameters** apply immediately without restart.

Each instance must have exactly one parameter group. You cannot edit AWS's default parameter groups — you create a custom group, modify it, and assign it to the instance. Parameter groups can be shared across multiple instances of the same engine version.

### Option Groups

Option groups are specific to RDS and apply to additional features that can be added to the database engine (not available in Aurora):

- Oracle options (Oracle Application Express, Statspack, etc.)
- SQL Server options (SQL Server Audit, transparent data encryption, etc.)
- MySQL options (memcached plugin, MariaDB Audit Plugin)

Option groups are less commonly needed for MySQL and PostgreSQL but are critical for Oracle and SQL Server.

---

## 15. RDS Proxy

RDS Proxy is a fully managed database proxy that sits between your application and your RDS or Aurora database. It pools and shares database connections.

### The Connection Problem

Traditional database architectures assume a small number of long-lived connections from server processes. Modern serverless and containerized architectures create a very large number of short-lived connections — Lambda functions, container tasks, and microservices all open connections frequently.

The problem: each connection to PostgreSQL is a forked process consuming ~10 MB of memory. MySQL is lighter but still has per-connection overhead. A Lambda function with concurrency of 1,000 creates up to 1,000 simultaneous database connections — far exceeding what most database instances can handle.

### How RDS Proxy Solves This

RDS Proxy maintains a pool of long-lived connections to the database. Application connections (from Lambda, containers, etc.) connect to the proxy. The proxy multiplexes thousands of application connections over a smaller number of actual database connections.

The proxy uses connection borrowing — when an application connection needs to execute a query, it borrows a database connection from the pool, executes the query, and returns the connection to the pool. Between queries, the application connection holds no database connection.

Benefits:

- Reduces the number of connections to the database by up to 80%
- Dramatically increases the number of clients that can connect simultaneously
- Faster connection establishment (application connects to proxy; proxy already has connections to DB)
- Automatic failover handling — the proxy absorbs the failover period and reconnects transparently

### Failover Handling

During an RDS or Aurora failover, the proxy automatically re-routes connections to the new primary. Applications connected to the proxy experience a much shorter interruption than applications connected directly to the database — the proxy absorbs the DNS propagation delay and re-connection time.

### Authentication

RDS Proxy supports IAM authentication and Secrets Manager integration for credentials. This means Lambda functions can use their IAM roles to authenticate to the database through the proxy, without database passwords in environment variables.

---

## 16. RDS Maintenance and Upgrades

### Maintenance Windows

RDS applies patches and minor upgrades during a configurable weekly maintenance window. AWS notifies you before applying maintenance that requires a restart. You can defer maintenance temporarily or choose an appropriate low-traffic time window.

### Minor Version Upgrades

RDS can automatically apply minor version upgrades (e.g., MySQL 8.0.31 → 8.0.32) during the maintenance window if auto minor version upgrade is enabled. Minor upgrades generally preserve full compatibility.

### Major Version Upgrades

Major version upgrades (e.g., MySQL 5.7 → 8.0, PostgreSQL 14 → 15) are significant changes that may break compatibility. AWS does not apply these automatically. You must initiate them manually after testing your application against the new version.

Major upgrades modify the instance in place. The process: RDS stops the database, runs the upgrade, and restarts. This results in downtime of several minutes to tens of minutes depending on database size and complexity.

Before any major upgrade:

- Test thoroughly in a non-production environment
- Take a manual snapshot (you cannot roll back an in-place upgrade; the snapshot is your restore point)
- Review the engine's release notes for breaking changes
- Check AWS's engine-specific upgrade documentation for any RDS-specific caveats

### End of Life

AWS provides extended support for database engine versions past their community end-of-life dates, for an additional charge. This allows more time to migrate but is not a substitute for keeping engines current.

---

# Part III — Amazon Aurora Deep Dive

---

## 17. Aurora Architecture — The Fundamental Difference

Aurora's architecture is the most important thing to understand about it. Everything else — the performance, the availability, the features — flows from the architectural decisions.

### The Traditional Database Problem

In traditional databases (including RDS MySQL/PostgreSQL), the database engine and its storage are tightly coupled on a single machine. The engine writes to local or network-attached storage. For high availability, you replicate the entire state (through binary logs or WAL) to a standby machine, which also writes to its own storage.

This means every write is done twice (once on primary, once replicated to standby), and all I/O — even reads — goes through a single storage volume with a fixed IOPS ceiling.

### Aurora's Answer: Separate Compute from Storage

Aurora fundamentally separates the database compute layer (the SQL engine, query planner, buffer pool) from the storage layer (where data is actually written and read). The storage layer is a distributed, shared system that all compute nodes access.

The Aurora storage layer:

- Spans three Availability Zones automatically
- Maintains six copies of every piece of data (two in each AZ)
- Uses a quorum-based write protocol: a write is durable when 4 of 6 storage nodes confirm
- Repairs itself automatically: if a storage node fails, the system copies data from other nodes to restore the 6-copy guarantee
- Grows automatically in 10 GB increments as data grows
- Has no IOPS or throughput configuration — Aurora manages I/O internally

### What This Changes About Replication

In RDS MySQL, a write goes: application → primary → commit to primary's EBS → replicate binary log to standby → standby applies log to its own EBS. Two full write paths, two storage systems.

In Aurora, a write goes: application → primary compute → distributed storage (6 copies). Only one write to storage, acknowledged when 4 of 6 nodes confirm. Aurora replicas do not write to storage at all — they read from the same shared storage volume the primary writes to.

This eliminates replication lag in the traditional sense. A replica sees data as soon as the storage commits it. The "lag" is reduced to the time for the replica's cache to be invalidated and updated from storage, typically under 100 milliseconds.

### What the Compute Layer Does

The compute layer runs the SQL engine — parsing queries, planning execution, managing connections, running the buffer pool. Each Aurora DB instance is a compute node. The primary compute node handles both reads and writes. Replica compute nodes handle only reads. All compute nodes share the same underlying storage.

---

## 18. Aurora Storage Engine

### The Cluster Volume

Aurora's storage is called the cluster volume. It is a single logical storage volume that appears as one database to the SQL engine, but is physically distributed across dozens of storage nodes in three AZs.

The cluster volume:

- Starts at 10 GB
- Grows automatically in 10 GB increments as data is written
- Maximum size: 128 TB
- You pay only for the storage actually used, not provisioned storage

### The Quorum Write

Every write to Aurora goes to all 6 storage replicas. The write is considered durable when 4 of 6 replicas acknowledge it (a write quorum). The primary can continue operating even if 2 storage nodes are unavailable — it still has 4/6 acknowledgment capability.

For the write to be blocked, 3 or more storage nodes must be unavailable simultaneously — an extremely rare event given that storage nodes are spread across three AZs.

### The Quorum Read

Aurora reads use a read quorum of 3 of 6. The system confirms that data is consistent by reading from at least 3 nodes and confirming they agree.

### Crash Recovery

Traditional databases (MySQL, PostgreSQL) must replay the transaction log (redo log / WAL) after a crash to bring the database to a consistent state. This recovery process can take minutes for large databases.

Aurora does not do redo log replay. Because the storage layer maintains durable, consistent data at all times (the quorum write ensures this), the compute layer can restart and immediately open the database without log replay. Aurora crash recovery takes seconds, not minutes.

### Self-Healing Storage

If a storage node becomes unavailable (hardware failure, network issue), the Aurora storage system automatically:

1. Detects the unavailability
2. Continues operating using the remaining 5 nodes (still above the read and write quorum)
3. Copies data from existing nodes to a replacement node
4. Reintegrates the replacement node transparently

This process is fully automatic and invisible to applications. The database continues operating throughout.

---

## 19. Aurora Cluster Anatomy

An Aurora cluster consists of:

**One Primary Instance (Writer)**: Handles all writes and can also serve reads. There is exactly one primary at any time (except Multi-Master, discussed separately).

**Zero to 15 Aurora Replicas (Readers)**: Handle read queries. Replicas share the cluster volume with the primary — they do not have separate storage. Up to 15 replicas.

**The Cluster Volume**: The shared distributed storage. Not a separate "instance" — it is the underlying storage infrastructure.

**Cluster Endpoints**: Logical connection points that route to the right instance(s). See Section 20.

### Replica Tiers

Aurora Replicas have a priority tier (0 highest, 15 lowest). During failover, Aurora promotes the highest-priority available replica. Replicas in the same tier are promoted based on which has the largest volume of changes (closest to the primary's state).

You use tier configuration to control which replica becomes primary during failover — ensure your largest-capacity replica has the highest priority.

---

## 20. Aurora Endpoints

Aurora's endpoint abstraction is one of its most powerful features and most commonly misunderstood.

### Cluster Endpoint (Writer Endpoint)

Points to the current primary instance. Always routes to whichever instance is currently the writer. After a failover, the cluster endpoint automatically routes to the new primary — no application DNS change required.

Use this endpoint for all write operations and for read operations that require reading your own writes (strong consistency).

### Reader Endpoint

Points to one of the Aurora Replicas using a round-robin load balancing mechanism. Every connection made to the reader endpoint is routed to one of the available replicas.

Use this endpoint for read traffic that can tolerate replica lag (eventual consistency). Directing read traffic to the reader endpoint reduces load on the primary.

The reader endpoint does not provide session affinity — consecutive connections may go to different replicas. If a query sequence requires reading data written in the same session, use the cluster endpoint.

If no replicas exist, the reader endpoint falls back to the primary.

### Instance Endpoints

Each specific instance (primary or replica) has its own endpoint. These are rarely used in application code but are useful for:

- Maintenance operations targeting a specific instance
- Monitoring tools connecting to a specific replica
- Testing specific replica behavior

### Custom Endpoints

You can create custom endpoints that route to a specific subset of instances. Use cases:

- Designate large-memory replicas for analytics queries and smaller replicas for OLTP reads, with different endpoints for each workload
- Route specific applications to specific replicas for isolation
- Gradually add new replicas to a custom endpoint for testing before full traffic

Custom endpoints override the reader endpoint logic — connections go only to the instances in the custom endpoint, not all replicas.

---

## 21. Aurora Replicas

### How Aurora Replicas Differ from RDS Read Replicas

The key difference: Aurora Replicas share the cluster volume with the primary. They do not have their own copy of the data. Instead, they have their own compute (CPU, memory, buffer pool) and read from the shared storage.

Because there is no data replication (the data is already in shared storage), the replica lag is dramatically lower — typically under 100ms, versus potentially minutes for RDS binary log replication under heavy write load.

Adding an Aurora Replica takes minutes — AWS provisions compute and connects it to the existing cluster volume. There is no data copy operation. Adding an RDS read replica requires copying the entire database.

### Aurora Replicas as Failover Targets

Unlike RDS Multi-AZ standbys (which are invisible and read-only), Aurora Replicas serve reads AND act as automatic failover targets. When the primary fails, Aurora promotes the highest-priority available replica to primary. Failover typically completes in under 30 seconds.

This is a significant architectural advantage: your read replicas serve double duty as high-availability standbys. You are not paying for an idle standby — every replica is doing useful work.

### Replica Scaling

Aurora Replicas can be added or removed dynamically. Aurora Auto Scaling can automatically add replicas when CPU or connections exceed thresholds and remove them when demand decreases. This provides elastic read scaling.

---

## 22. Aurora Global Database

Aurora Global Database extends a single Aurora cluster across multiple AWS regions. It is AWS's managed solution for globally distributed databases.

### Architecture

One **primary region** contains the full Aurora cluster (primary instance + optional replicas). Up to 5 **secondary regions** each contain read-only Aurora clusters.

Replication from the primary region to secondary regions is handled at the storage layer — Aurora replicates storage blocks directly, bypassing the SQL engine. This achieves:

- Replication lag under 1 second (typically 100–400ms)
- Minimal impact on primary region performance (storage-layer replication, not SQL-layer)

### Use Cases

**Low-latency reads for global users**: Users in Europe read from a European secondary region cluster instead of sending read queries across the Atlantic to US East.

**Disaster recovery with minimal data loss**: If the primary region (US East) becomes unavailable, you promote a secondary region (EU West) to be the new primary. The RPO (Recovery Point Objective) is typically under 1 second — you lose at most 1 second of data. The RTO (Recovery Time Objective) is typically under 60 seconds.

**Business continuity**: Maintain operations during regional AWS outages, which are rare but have occurred.

### Planned Failover vs Unplanned

**Planned switchover** (e.g., for maintenance): Secondary is promoted gracefully. The primary replicates all remaining changes to the secondary before promoting it. Zero data loss (RPO = 0).

**Unplanned failover** (region unavailable): You manually initiate promotion of a secondary. Up to ~1 second of data in transit may be lost (the storage-layer replication lag).

### Cost

Global Database incurs costs for each cluster in each region (primary + all secondaries) plus cross-region replication data transfer.

---

## 23. Aurora Serverless

Aurora Serverless is a deployment mode where the compute capacity scales automatically based on actual demand, including scaling to zero when there are no connections.

### Aurora Serverless v1 vs v2

**Aurora Serverless v1** (the original): Scales in discrete steps (Aurora Capacity Units). Scaling up takes 15–30 seconds — connections may drop during scaling. Scales to zero when idle. Suitable for intermittent, unpredictable workloads where occasional latency is acceptable. Not suitable for steady high-throughput workloads.

**Aurora Serverless v2** (current, recommended): Scales in fine-grained increments (0.5 ACU increments) almost instantly. Does NOT scale to zero (minimum 0.5 ACU is always running). Designed for production workloads — it can handle thousands of requests per second and scale within seconds. Coexists with provisioned instances in the same cluster.

### Aurora Capacity Units (ACU)

An ACU is approximately 2 GB of memory and corresponding CPU and network. You configure minimum and maximum ACUs. Serverless v2 scales between these bounds automatically.

ACU pricing is per ACU-hour — you pay only for the capacity consumed, not provisioned capacity.

### Scale to Zero (v1 only)

When configured with minimum ACU of 0, Aurora Serverless v1 scales compute to zero after a configurable inactivity period. While at zero, you pay only for storage.

The first connection after scaling to zero incurs a cold start — provisioning compute and warming the buffer pool. This can take 15–30 seconds. Applications must be tolerant of this latency (use connection timeouts appropriately).

Scale to zero is ideal for development databases, staging environments, and infrequently used applications where cost during idle periods is a concern.

### Serverless v2 in Hybrid Clusters

A single Aurora cluster can have a mix of provisioned instances and Serverless v2 instances. For example:

- Provisioned writer (predictable baseline write capacity)
- Serverless v2 readers (elastic read capacity that scales with demand)

This provides fine-grained control over cost and capacity.

---

## 24. Aurora Multi-Master

Aurora Multi-Master (currently Aurora MySQL only, with limited availability) allows multiple write nodes in a single cluster. All instances can handle writes simultaneously.

This is different from Global Database, which has one primary region. Multi-Master has multiple writers within a single region cluster.

### What Multi-Master Solves

Multi-Master provides write high availability: if one writer fails, other writers continue serving writes without failover. Standard Aurora clusters require failover when the writer fails (read replicas become unavailable for writes during failover). Multi-Master eliminates this write interruption.

### Multi-Master Conflicts

When multiple nodes write simultaneously, write conflicts can occur (two nodes updating the same row). Aurora Multi-Master uses an optimistic concurrency model — writes are allowed to proceed, but if the storage layer detects a conflict, one write fails with an error. Applications must handle write conflicts and retry.

This conflict handling requirement makes Multi-Master complex to use correctly. It is not a drop-in replacement for single-writer clusters.

### Current Status

Aurora Multi-Master is available only for Aurora MySQL 5.6-compatible clusters. Given MySQL 5.6 is end-of-life, this feature has limited current applicability. AWS's recommended pattern for write availability is Aurora Global Database with fast regional failover rather than Multi-Master.

---

## 25. Aurora Backups and Backtrack

### Automated Backups

Aurora continuously backs up data to S3. Unlike RDS, Aurora does not have a daily backup window — backup is a continuous background process at the storage layer. The backup retention period is 1–35 days (configurable).

Point-in-time restore works the same as RDS: restore to any second within the retention period. Aurora restores faster than RDS because the storage layer is already distributed and the restore can read from multiple nodes in parallel.

Restoring creates a new Aurora cluster — you do not restore in place.

### Aurora Backtrack (MySQL only)

Backtrack is a unique Aurora MySQL feature that allows you to rewind the cluster to a previous point in time **without creating a new cluster and without restoring from a backup**.

Backtrack works by storing a change log of all write operations up to the configured backtrack window (up to 72 hours). To backtrack, you specify a target time. Aurora applies the changes in reverse to reach that state.

Backtrack is dramatically faster than restore from backup. For small backtrack targets (minutes to hours), the operation takes minutes. Restore from backup can take much longer for large databases.

Use cases:

- Accidentally dropped a table? Backtrack 5 minutes.
- Bad data migration? Backtrack to before the migration.
- Testing a destructive operation? Take a note of the time, run the operation, evaluate, then backtrack if needed.

Limitations:

- Aurora MySQL only (not Aurora PostgreSQL)
- The cluster must have had backtrack enabled at creation time
- Backtracking pauses the database briefly during the operation
- The backtrack window has storage costs (you pay for the change log)

### Manual Snapshots

Same as RDS: on-demand, persist indefinitely, can be copied across regions, can be shared with other accounts.

---

## 26. Aurora Security

Aurora uses the same security model as RDS (Section 13 covers the fundamentals), with some Aurora-specific considerations.

### Network Security

Same VPC, subnet, and security group model as RDS. Aurora clusters must be placed in a DB Subnet Group spanning at least two AZs (three recommended).

Because all compute nodes share the cluster volume, each compute node needs network access to the distributed storage layer — this is handled internally by AWS and does not require additional configuration.

### Encryption

Encryption at rest applies to the entire cluster volume and all backups. Must be enabled at cluster creation. Encryption cannot be added to an existing unencrypted cluster in place (same workaround as RDS: snapshot, copy with encryption, restore).

### Cluster-Level Authentication

Aurora supports the same authentication mechanisms as RDS: native database authentication, IAM database authentication, and Secrets Manager integration.

### Audit Logging

Aurora MySQL supports the Advanced Auditing feature — database activity logging (connections, queries, schema changes, users). Audit logs can be published to CloudWatch Logs.

Aurora PostgreSQL supports pgaudit (PostgreSQL Audit Extension) for fine-grained audit logging.

---

## 27. Aurora Parallel Query

Aurora Parallel Query is a feature for Aurora MySQL that pushes query processing down to the distributed storage layer for analytical queries, rather than pulling all data up to the compute layer.

### The Problem It Solves

Analytical queries (full table scans, aggregations, complex filters on large tables) on operational databases are expensive. Traditionally, the database reads all relevant data from storage into the buffer pool (if not already cached), then processes it in the compute layer. For large tables, this saturates the I/O path and evicts frequently-used OLTP data from the buffer pool.

### How Parallel Query Works

With Parallel Query enabled, Aurora pushes the filtering and aggregation logic down to the storage nodes. Each storage node processes its portion of the data in parallel and returns only the relevant results. The compute node receives pre-filtered, partially aggregated data and combines the results.

This means:

- Large scans do not saturate the single I/O path between compute and storage
- Analytical queries can run significantly faster (Aurora claims up to 2x faster for certain query patterns)
- OLTP performance is less impacted because analytical queries do not flood the buffer pool

### Limitations

Parallel Query is not used for every query — the optimizer decides when to use it. It is most effective for large full-table scans with simple filters and aggregations. Complex joins or queries that benefit from indexes may not use Parallel Query.

Parallel Query is available for Aurora MySQL 5.6 and 5.7 compatible clusters. It is not available for Aurora PostgreSQL.

---

# Part IV — Shared Concepts

---

## 28. Networking and VPC Placement

### DB Subnet Groups

Both RDS and Aurora instances are placed in a DB Subnet Group — a named collection of VPC subnets across multiple AZs. The subnet group defines where in your VPC the database instances can be placed.

For high availability (Multi-AZ RDS or any Aurora cluster), the subnet group must span at least two AZs. Three AZs is recommended for Aurora, as Aurora distributes storage across three AZs.

### Private vs Public Access

**Private (recommended)**: Instances get private IPs only. Accessible only from within the VPC or from connected VPCs/networks (VPN, Direct Connect). No internet exposure.

**Public**: Instances get both private and public IPs. Accessible from the internet (subject to security group rules). Never recommended for production databases.

### VPC Peering and Connectivity

Databases in one VPC can be accessed from:

- EC2 instances in the same VPC (directly)
- EC2 instances in a peered VPC (via VPC peering)
- On-premises networks (via VPN or Direct Connect)
- Lambda functions in the same VPC (Lambda must be VPC-enabled)
- ECS/EKS containers in the same VPC

### Multi-AZ Network Considerations

RDS Multi-AZ and Aurora clusters span multiple AZs. Cross-AZ network traffic (replication, storage communication) is charged at inter-AZ data transfer rates. For Aurora, the storage layer handles cross-AZ communication internally — this cost is factored into Aurora's I/O pricing.

---

## 29. High Availability Patterns

### Pattern 1: RDS Multi-AZ (Active-Passive)

Single active primary, one passive standby. Standby is not readable. Failover takes 60–120 seconds.

Best for: Simple high availability requirements where read scaling is not needed and cost matters.

### Pattern 2: Aurora Cluster (Active-Readable Standbys)

Single writer, multiple readable replicas (up to 15) that also serve as failover targets. Failover takes ~30 seconds. Replicas serve reads.

Best for: Applications that need both HA and read scaling. Most production Aurora use cases.

### Pattern 3: Aurora Global Database (Active-Passive Cross-Region)

Primary region cluster + secondary region cluster(s). Secondary serves reads. Failover to secondary promotes it to primary (minutes). RPO < 1 second.

Best for: Geographically distributed applications, compliance requirements for geographic redundancy, cross-region DR.

### Pattern 4: Aurora Serverless v2 with Provisioned Writer

Provisioned primary (predictable write performance) + Serverless v2 readers (elastic read scaling). Combines cost efficiency with elastic capacity.

Best for: Applications with predictable write patterns but variable read demand.

### Pattern 5: RDS with Read Replicas + Proxy

Primary + read replicas + RDS Proxy in front. Proxy pools connections, handles failover. Read replicas absorb read traffic.

Best for: Applications with many short-lived connections (Lambda, containers) connecting to RDS.

### RPO and RTO Reference

|Setup|RPO (data loss)|RTO (downtime)|
|---|---|---|
|RDS Single-AZ|Minutes (last backup)|Minutes–hours (restore from backup)|
|RDS Multi-AZ|~0 (synchronous)|60–120 seconds|
|Aurora Cluster|~0|20–30 seconds typical|
|Aurora Global DB (planned)|0|< 60 seconds|
|Aurora Global DB (unplanned)|< 1 second|60–120 seconds|

---

## 30. Monitoring and Observability

### CloudWatch Metrics

Both RDS and Aurora publish a rich set of metrics to CloudWatch.

**Compute and memory:**

- `CPUUtilization` — percentage of CPU used
- `FreeableMemory` — RAM available; a continuously decreasing value indicates a memory leak or insufficient memory for the working set
- `DatabaseConnections` — current number of connections; approaching max_connections is a critical warning

**Storage (RDS):**

- `FreeStorageSpace` — remaining storage; alert well before it reaches 0
- `ReadIOPS` / `WriteIOPS` — actual IOPS consumed
- `ReadLatency` / `WriteLatency` — per-operation I/O latency
- `DiskQueueDepth` — I/O operations waiting; high values indicate storage saturation

**Aurora storage:**

- `AuroraVolumeBytesLeftTotal` — remaining capacity in the cluster volume
- `StorageNetworkThroughput` — data flowing between compute and storage

**Replication:**

- `ReplicaLag` — seconds behind the primary (RDS read replicas and Aurora replicas)
- `AuroraReplicaLag` — Aurora-specific; typically < 100ms
- `AuroraBinlogReplicaLag` — if Aurora is replicating to an external MySQL

**Cache performance:**

- `BufferCacheHitRatio` (Aurora) — percentage of reads served from the buffer cache; should be > 95% for most workloads. A low ratio means the working set doesn't fit in memory
- `SwapUsage` — database swapping to disk; a critical indicator of memory pressure

### Enhanced Monitoring

Enhanced Monitoring provides real-time metrics at 1-second granularity from the OS layer (not the CloudWatch layer). Shows: detailed CPU per-core, memory breakdown, disk I/O per device, network throughput, top processes.

This is useful when CloudWatch metrics (sampled at 60-second intervals) are too coarse for diagnosing transient performance issues.

### CloudWatch Logs

Database engine logs can be published to CloudWatch Logs:

- **Error log**: Engine errors, warnings, crash recovery messages
- **Slow query log** (MySQL): Queries exceeding a configurable execution time threshold
- **General log** (MySQL): All queries (extremely verbose; use sparingly)
- **PostgreSQL logs**: configurable via log_min_duration_statement, log_connections, log_disconnections

---

## 31. Performance Insights

Performance Insights is a database performance monitoring and analysis tool built into RDS and Aurora. It is one of the most powerful tools for understanding database performance and diagnosing bottlenecks.

### The Core Concept: DB Load

Performance Insights measures **DB Load** — the average number of active sessions in the database at any point in time. A session is active if it is executing a query or waiting for something.

DB Load is decomposed by **wait events** — the reasons sessions are waiting. Common wait events:

- `io/file/*` — waiting for disk I/O (storage bottleneck)
- `lock/table/*` — waiting for table locks (contention)
- `wait/synch/*` — waiting for mutexes or semaphores (concurrency bottleneck)
- `CPU` — actively executing on CPU (CPU is the limiting resource)
- `io/aurora_respond_to_fcntl` — Aurora-specific I/O waiting

### How to Read Performance Insights

The top section shows DB Load over time as a stacked area chart. Each color represents a different wait event. The horizontal line shows `Max CPU` — the number of vCPUs on the instance.

If DB Load is consistently below `Max CPU` and the dominant wait event is CPU, the database is CPU-bound. Add CPU capacity.

If DB Load is above `Max CPU` and the dominant wait is I/O, the database is I/O-bound. Investigate which queries are generating the most I/O, add memory (larger buffer pool), or increase storage IOPS.

If the dominant wait is locking, there is contention between queries — investigate and optimize the queries causing the locks.

### Top SQL

Performance Insights shows the top SQL statements by their contribution to DB Load. You can sort by:

- DB Load (total contribution to database busyness)
- Rows examined per second (I/O-heavy queries)
- Rows sent per second
- Execution count

This is the fastest path from "the database is slow" to "this specific query is causing it."

### Retention

Performance Insights data is retained for 7 days free. Extended retention (up to 2 years) is available for an additional charge.

---

## 32. Limits and Quotas

### RDS Limits

|Limit|Value|
|---|---|
|DB instances per account per region|40 (default)|
|Read replicas per source|5|
|Max storage per instance (gp2/gp3)|64 TB|
|Max storage per instance (io1/io2)|64 TB|
|Max IOPS (io2 Block Express)|256,000|
|Max connections (MySQL, db.r6g.16xlarge)|~15,000|
|Max connections (PostgreSQL, depends on memory)|~5,000 (1 GB) to ~87,000 (large instances)|
|Backup retention period|0–35 days|
|Multi-AZ standby instances|1|
|Parameter groups per account|50|

### Aurora Limits

|Limit|Value|
|---|---|
|Aurora Replicas per cluster|15|
|Max cluster volume size|128 TB|
|Max Aurora clusters per account per region|40|
|Aurora Global Database secondary regions|5|
|Backtrack window|Up to 72 hours|
|Aurora Serverless v2 min ACU|0.5|
|Aurora Serverless v2 max ACU|256|
|Aurora Serverless v1 scale-to-zero|Supported (with cold start)|
|Custom endpoints per cluster|5|

### Important Fixed Constraints

**Max connections**: PostgreSQL's connection limit is based on available memory (`max_connections` parameter). As a rule of thumb, you should not exceed ~100 connections per GB of RAM. For connection-heavy applications, use RDS Proxy.

**Storage decrease**: Neither RDS nor Aurora support decreasing allocated storage. Plan sizing carefully.

**Aurora storage billing**: Aurora charges for the maximum storage ever used in the cluster volume's lifetime, not the current usage. If you load 50 TB of data and delete 40 TB, you still pay for 50 TB until the volume is reorganized (which requires creating a new cluster and migrating data).

---

## 33. Pricing Model

### RDS Pricing Dimensions

**Instance hours**: Charged per hour the instance is running. On-Demand pricing is the default. Reserved Instances (1-year or 3-year commitment) offer up to 60% discount.

**Storage**: Charged per GB-month for the provisioned storage. For gp3, IOPS above 3,000 and throughput above 125 MB/s are charged additionally.

**Backup storage**: Automated backup storage equal to the provisioned instance storage is free. Additional backup storage is charged per GB-month.

**Data transfer**: Data transfer in is free. Data transfer out to the internet is charged at standard rates. Cross-AZ data transfer (for replication) is charged.

**Multi-AZ**: Roughly doubles the instance cost (you pay for two instances — primary and standby — though AWS prices it as a multiplier on the primary rate).

**Read replicas**: Each replica is a separate instance charge plus storage.

### Aurora Pricing Dimensions

**Instance hours**: Same as RDS — per instance, On-Demand or Reserved.

**Storage**: Charged per GB-month for actual data stored (not provisioned). The billing is based on the maximum high-water mark of storage used.

**I/O requests**: This is Aurora's key differentiator from RDS pricing. Every read and write I/O to the cluster volume is charged per million requests ($0.20 per million I/Os in US East). High-write workloads can make Aurora significantly more expensive than RDS with provisioned IOPS.

**Backup storage**: Same as RDS — free up to the cluster volume size, charged beyond that.

**Aurora Serverless v2**: Charged per ACU-hour. No instance charge when scaled to minimum ACU.

**Aurora Global Database**: Charged for all instances in all regions plus cross-region replication data transfer.

### Reserved Instances

Both RDS and Aurora support Reserved Instances:

- **1-year, no upfront**: ~30% discount vs On-Demand
- **1-year, partial upfront**: ~35% discount
- **1-year, all upfront**: ~40% discount
- **3-year, all upfront**: ~60% discount

Reserved Instances apply to instance hours only — storage, I/O, and backup are still billed at On-Demand rates.

### RDS vs Aurora Cost Comparison

Aurora is typically more expensive per instance than RDS. However, the total cost comparison depends on:

**Replica count**: Aurora replicas do not pay for additional storage (shared cluster volume). RDS replicas each pay for a full copy of the storage. If you need 5+ replicas, Aurora can be cheaper overall.

**I/O pattern**: Aurora charges per I/O request. A write-heavy workload on Aurora with high I/O rates can exceed the cost of RDS io1 with provisioned IOPS.

**Storage size**: Aurora charges for actual storage used (not provisioned). If your database is 10 GB but you provisioned 500 GB on RDS (for IOPS), Aurora is significantly cheaper.

**Serverless**: If your workload is intermittent, Aurora Serverless v2 can be far cheaper than a provisioned RDS instance sitting idle.

---

## 34. Best Practices

### Schema and Query Design

Database performance is determined more by schema and query design than by the database infrastructure. The most powerful instance cannot compensate for missing indexes, unnecessary full table scans, or N+1 query patterns.

**Always have a primary key.** Every table should have a primary key. For InnoDB (MySQL/Aurora MySQL), the primary key is the clustered index — its structure determines physical data layout. A well-chosen primary key (sequential, compact) dramatically affects insert performance and range query performance.

**Index for your queries.** Indexes are designed around how data is queried, not how it is stored. Analyze your most frequent and most expensive queries and ensure they have supporting indexes. Use EXPLAIN / EXPLAIN ANALYZE to verify query plans.

**Avoid N+1 queries.** The most common application-level database problem: fetching a list of items, then fetching related data for each item with a separate query. Use JOINs or eager loading at the ORM layer.

**Limit result sets.** Always paginate queries. Never SELECT * FROM large_table without a WHERE clause and LIMIT.

**Connection management.** Open connections once and reuse them. Connection pooling at the application level (or via RDS Proxy) is essential for high-concurrency applications.

### Instance Sizing

Prefer memory-optimized instances (r classes) for production databases. Database performance is heavily memory-dependent. The working set (frequently accessed data) should fit in the buffer pool.

Do not use burstable instances (t classes) for production. CPU credit exhaustion under sustained load causes severe, hard-to-diagnose performance degradation.

Right-size with data. Monitor CPUUtilization, FreeableMemory, and ReadIOPS over weeks, not days. Database load is bursty. Average metrics can be misleading — watch percentiles (P95, P99).

### Storage

Use gp3 for most workloads. It decouples size from IOPS, is more cost-efficient than gp2, and supports higher baseline performance.

Use io1/io2 only when you need more than 16,000 IOPS (the gp3 maximum) or require guaranteed low latency at very high IOPS.

Enable storage auto-scaling with a generous maximum. Running out of storage takes the database offline — it is a hard outage. Set the maximum to what you would need in a worst-case growth scenario.

### Backup and Recovery

Never set backup retention to 0 on production databases. This disables automated backups and eliminates point-in-time restore capability.

Test restores periodically. A backup that has never been restored is a backup of unknown quality. Know how long restore takes for your database size. Practice restore procedures before you need them in an emergency.

Take manual snapshots before any major change: major version upgrade, large data migration, schema migration.

### Security

Never put a database in a public subnet. Never enable public accessibility. Databases should only be reachable from within the VPC.

Use security groups to restrict access to the minimum necessary. The database security group should only allow connections from the application tier's security group on the specific port.

Enable encryption at rest for all production databases. Use customer-managed KMS keys for databases with compliance requirements (audit trail for key usage, key rotation control).

Use Secrets Manager for credential management. Enable automatic rotation. Never put database credentials in code, configuration files, or environment variables passed as plaintext.

Enable TLS/SSL enforcement for all connections in production.

Use IAM database authentication for applications running on AWS (Lambda, EC2, ECS). Eliminates long-lived passwords.

### High Availability

Always use Multi-AZ or Aurora clusters for production. A single-AZ RDS instance has no failover capability — a hardware failure requires manual intervention and restore from backup (potentially hours of downtime).

Understand your failover behavior. Test it periodically. Reboot with failover in RDS, or manually trigger Aurora failover, to verify your application reconnects correctly.

Ensure your application's database connection layer:

- Does not cache IP addresses (respect DNS TTL)
- Retries failed connections with backoff
- Surfaces connection errors appropriately
- Uses connection pools with proper health checking

For applications with many connections (Lambda, containers), use RDS Proxy. It dramatically improves failover behavior and connection management.

### Monitoring and Alerting

Set CloudWatch Alarms on:

- `FreeStorageSpace` < 20% of provisioned size (alert), < 10% (critical)
- `CPUUtilization` > 80% for sustained periods
- `DatabaseConnections` > 80% of `max_connections`
- `ReplicaLag` > acceptable threshold for your application
- `FreeableMemory` below a workload-specific floor
- `ReadLatency` / `WriteLatency` above SLA thresholds

Enable Performance Insights on all production instances. It provides the fastest path to diagnosing the cause of any performance problem.

Enable Enhanced Monitoring for production instances with unexplained OS-level issues.

---

## 35. Common Failure Modes and How to Think About Them

### Slow Queries — The Universal Problem

Almost every database performance problem is ultimately a query problem. The investigation path:

1. Open Performance Insights. Identify which queries are contributing most to DB Load.
2. Run EXPLAIN on the offending queries. Look for full table scans (type: ALL in MySQL EXPLAIN).
3. Add the missing index or rewrite the query to use an existing index.
4. Verify with EXPLAIN that the query now uses the index.
5. Monitor that the DB Load contribution of the query has decreased.

Never immediately size up the instance in response to high CPU or high I/O. First determine if the root cause is a missing index or inefficient query. Scaling up gives headroom but does not fix the underlying problem.

### Connection Exhaustion

Symptom: Applications start receiving "Too many connections" errors. New connections cannot be established.

Root cause: The number of simultaneous database connections has exceeded `max_connections`.

Common triggers: Sudden traffic spike causing many application instances to open connections simultaneously; connection leak in application code (connections opened but not closed); Lambda function concurrency spike; deployment of new application instances without retiring old ones.

Solutions:

- Immediate: Identify and terminate idle connections (SHOW PROCESSLIST in MySQL, pg_stat_activity in PostgreSQL)
- Short-term: Increase max_connections (requires parameter group change and possibly instance resize for PostgreSQL, since max_connections is memory-limited)
- Long-term: Implement connection pooling (application-level or RDS Proxy); use shorter connection lifetimes; size the application's connection pool to match the database's capacity

### Storage Full

When an RDS instance runs out of storage, the database sets itself to read-only mode to protect data integrity. Writes fail. Applications see write errors.

Prevention is far better than cure. Set storage auto-scaling with a generous maximum. Set CloudWatch Alarms on FreeStorageSpace well before it becomes critical.

Recovery: Increase the allocated storage via a storage modification. The modification is online (no downtime) but the instance enters an "optimizing" state. Alternatively, delete large tables or data that is no longer needed, or truncate tables that accumulate transient data (logs, sessions, caches stored in the DB).

### Replication Lag

Symptom: Reads from replicas return stale data. The ReplicaLag metric grows.

For RDS read replicas (binary log replication): lag grows when the primary is writing faster than the replica can apply changes. Root causes include: large transactions (long-running writes that generate huge binary log entries), DDL operations on large tables (ALTER TABLE, index creation), or the replica instance being slower/smaller than the primary.

Solutions: Ensure replica instance class matches or exceeds the primary; avoid large transactions (break into smaller batches); use pt-online-schema-change or similar tools for large table DDL; route time-sensitive reads to the primary.

For Aurora replicas: lag > 100ms is unusual and indicates a problem. Check for network issues between the compute layer and storage layer, or excessive load on the replica's compute.

### Failover Not Working as Expected

Symptom: After RDS Multi-AZ failover, the application does not reconnect. Or reconnection takes longer than expected.

Common causes:

**DNS caching**: The application (or an intermediate library) cached the IP address of the old primary. After failover, the DNS record points to the new primary, but the cached IP still routes to the dead instance. Fix: ensure DNS TTL is respected (typically 5 seconds for RDS/Aurora). In Java, set `networkaddress.cache.ttl=1`. In Node.js and Python, DNS is not typically cached aggressively, but check your specific libraries.

**No connection retry**: The application opens a connection, it fails during failover, and the application crashes or returns an error without retrying. Fix: implement connection retry logic with exponential backoff.

**Connection pool health checking**: The connection pool holds dead connections from before failover and does not detect they are dead until a query fails. Fix: configure connection pool health checking (`testOnBorrow`, `keepalive` settings depending on the pool library).

**Long failover**: If failover takes more than 2 minutes, something is wrong. Check: is the standby in a different AZ? Is there an ongoing maintenance operation? Is the security group allowing traffic to the new endpoint?

### Aurora Cluster Endpoint Confusion

Symptom: After configuring Aurora, all traffic goes to the primary even though the application is supposed to use the reader endpoint.

Common cause: The reader endpoint was set up in configuration, but application code still hardcodes the cluster endpoint or has a fallback to the cluster endpoint when the reader endpoint is unavailable.

Aurora reader endpoint behavior: if all replicas are unavailable, the reader endpoint routes to the primary. This means during testing with no replicas, the reader endpoint works (routes to primary). When replicas are added, reads should automatically be served by replicas. Monitor BufferCacheHitRatio and CPUUtilization per instance to verify reads are distributed.

### Aurora I/O Cost Surprise

Symptom: Aurora costs significantly more than expected.

Root cause: I/O request charges are higher than anticipated. Aurora charges per million I/O operations. A write-heavy workload with millions of writes per second generates substantial I/O charges that dwarf instance and storage costs.

Investigation: Check the `VolumeReadIOPs` and `VolumeWriteIOPs` CloudWatch metrics at the cluster level. Calculate the monthly I/O count and multiply by the per-million rate.

Mitigation:

- Optimize queries to reduce unnecessary I/O (missing indexes causing table scans)
- Increase buffer pool size (memory-optimized instance) so more reads are cache hits and do not hit storage
- Consider switching high-write workloads to RDS with provisioned IOPS if I/O costs exceed provisioned IOPS costs
- Aurora I/O-optimized pricing (a newer option) trades a higher instance price for no per-I/O charges — better for I/O-intensive workloads

---

_This guide reflects Amazon RDS and Aurora capabilities as of mid-2024. AWS regularly adds features and changes pricing — always verify with the official AWS documentation and pricing pages._
