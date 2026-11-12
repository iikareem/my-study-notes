---
type: moc
aliases:
  - Concepts
  - Concepts Index
tags:
  - concepts
  - moc
---

# Concepts MOC

Lasting knowledge — grouped for learning and review.

## Network

- [[TCP & Connection Lifecycle]] *(merged hub)*
- [[TCP Connections, Ephemeral Ports, File Descriptors & Connection Pooling]] · [[C10K Problem]]
- [[HTTP Connections]] · [[TCP TIME_WAIT — Recap]] · [[Congestion Window]]
- [[HTTP]] · [[WebSocket vs Server-Sent Events (SSE)]] · [[WebSocket API Design Guide]]
- [[ARP]] · [[CIDR Notation]] · [[IP Address]] · [[MAC Address]]
- [[Network Routing]] · [[Protocols]]
- [[Bandwidth vs Throughput]] · [[Latency and Throughput]] · [[Network Request and Response Stages]]
- [[What Happens When You Type]]

## Distributed Systems

- [[Consistent Hashing]] · [[Distributed Lock]] · [[Distributed Locks A Problem-by-Problem Deep Dive]] · [[Fencing Token]] · [[Redlock and CS Comparison]]
- [[Backpressure]] · [[Thundering Herd Problem]] · [[Clock Skew]] · [[Hot Key Problem]]
- [[HOL]] · [[Mechanical Sympathy]] · [[Memory Ordering]]
- [[Pipeline and Branch]] · [[Microservices]] · [[Why Microservices Use Separate Databases (Database-per-Service Pattern)]]
- [[DB Transactions Sagas and Event Sourcing]] · [[Conflict-Free Replicated Data Types]] · [[Distributed Coordination Services]]
- [[CDC The Right Tool for the Wrong Job]] · [[Debezium]]
- [[The Noisy Neighbor Problem in Kafka — Complete Guide]] · [[Why Kafka Outperforms Databases for High-Throughput Data Ingestion]]

## System Design

→ Full hub: [[System Design MOC]]

- [[Scaling Reads vs Writes]] · [[Scaling Reads]] · [[Scaling Writes]] · [[Hot Key Problem]]
- [[Dealing with Contention in System Design]] · [[Long Running Tasks]] · [[Large Blobs]] · [[Object Storage Design Patterns]]
- [[Caching Explained Simply]] · [[Caching Made Simple]]
- [[Infrastructure Routing The Complete Mental Model]] · [[Multi-Step Processes in System Design]]
- [[Realtime Updates]] · [[Realtime Scaling WebSocket & SSE]] · [[WebSocket Gateway Pattern]] · [[WebSocket Gateway Study Notes]]
- [[Service and DB Decomposition Guide]] · [[Deployment Strategies, API Versioning & WebSocket Gateways]]
- [[NFR and Estimation Roadmaps]] · [[System Design Review Checklist]] · [[Systems Design Recap Notes]]
- [[Database Replication]] · [[Scaling PostgreSQL to 800 Million Users]]

## Database

- [[Isolation Level and MVCC]] · [[MVCC recap]] · [[MVCC, Indexes, and Vacuum vs Reindex]] · [[VACUUM]] · [[auto vaccum recap]] · [[WAL]] · [[WAL, Checkpoints & Recovery in PostgreSQL]] · [[WAL, LSN, and Checkpoint  Full Lifecycle Recap]] · [[HOT]] · [[TOAST Table]]
- [[Lock in Database]] · [[DB Lock vs Advisory Lock vs Redis Lock]] · [[Database Deadlocks The Comprehensive Engineering Guide]] · [[Select for update - note]]
- [[Explain Analyze]] · [[pg_stat_statements]] · [[pgstatindex]]
- [[BRIN]] · [[Btree vs LSM]] · [[B-Tree & B+ Tree]] · [[Database Internals Write Path & B-Tree Index]] · [[LFC]] · [[Hint Bit Update]] · [[Prewarm]]
- [[Prepared Statements — The Optimization Story]] · [[Extended Statistics in PostgreSQL]]
- [[PostgreSQL Bulk Updates The Hidden Cost of Doing It All at Once]] · [[Monotonically Increasing vs Random Index Writes]]
- [[PostgreSQL Parallel Query Execution]] · [[PostgreSQL 18]] · [[PostgreSQL Storage Internals Study Notes]] · [[PostgreSQL Schema Pitfalls — Corrected Guide]]
- [[PostgreSQL Memory Management Understanding the 25% Shared Buffers Rule]]
- [[Postgres `SELECT DISTINCT` Doesn't Scale — And How to Fix It]] · [[`LIMIT 1` Does Not Mean Postgres Reads One Row]]
- [[Database Architecture Notes PostgreSQL vs MySQL]] · [[PG and MySQL]] · [[Databases Deep Dive Relational vs Non-Relational]] · [[DynamoDB vs Postgres]]
- [[Database Replication]] · [[Read Replicas Shared Storage vs. Physical Storage (Classic)]] · [[Database Migrations for Backend Developers]]
- [[Search Types Mandatory Knowledge Guide]] · Geo search: [[Geohash — Complete Guide]] · [[R-Tree — Spatial Indexing]] · [[Books]]
- Join algorithms: [[Hash Join]] · [[Nested Loop Join]]
- Design / optimization: [[3VL]] · [[Where different condition types]] · [[join_collapse_limit]]
- OLAP foundations: [[00 OLAP MOC|OLAP MOC]] · [[OLAP Starter Path — Think OLAP First]] · [[01 Module 1 — OLTP vs OLAP Mental Model|OLTP vs OLAP]] · [[02 Module 2 — Columnar Execution Internals|Columnar Internals]] · [[04 Module 3 — Parquet & File Formats|Parquet]] · [[05 Module 4 — Partitioning & Data Layout|Partitioning]] · [[06 Module 5 — Star Schemas & Dimensional Design|Star Schemas]] · [[07 Module 6 — Ingestion Thinking|Ingestion]] · [[08 Module 7 — DuckDB in Practice|DuckDB]] · [[09 Module 8 - Query Patterns That Matter in OLAP|Query Patterns]]
- Pitwall case study: [[Pitwall Project — Everything We Learned]] · [[Pitwall ETL Lesson — Developer Takeaways]] · [[Pitwall Star Schema Diagram]]

## Backend

- [[Node.js Event Loop & Internals]] *(merged)* · [[Beyond the Event Loop — V8 libuv Node.js]] · [[Go Execution Model vs Node.js Event Loop]]
- [[Threads, Processes, Cluster & Worker Threads (Node.js)]] · [[V8 Hidden Classes]] · [[TypeScript Promises]] · [[JavaScript Symbol — from scratch]] · [[Request Lifecycle in Nest]]
- [[Message Brokers Guide — RabbitMQ & Kafka]] · [[Messaging RabbitMQ and Kafka]] · [[Redis PubSub & Socket.IO Redis Adapter]]
- [[Serialization vs Message Encoding vs Compression]] · [[utc and timezones-guide]]
- [[DDD]] · [[Parquet]] · [[Programming Language]]

## Security

- [[PASETO]] · [[02 - Tokens and Tokenization|Tokens and Tokenization]] · [[Cryptography Notes Essentials]]

## Observability

- [[Observability Stack Layers Contracts Swappability]] · [[Ontology-Driven Observability]]
- Logs: [[Structured Logs]] · [[Pino vs Winston]] · [[Logging Winston vs Pino Deep Dive]] · [[Logging Additional Notes]]
- Rate limiting: [[Rate Limiting]] · [[Adaptive]]

## Operating System

- [[Process]] · [[Process Life Cycle]] · [[Process Management]] · [[Worker]]
- [[Kernel]] · [[Virtual Memory]] · [[Context Switching]] · [[vCPU]]
- [[Memory Fragmentation]] · [[Memory Hierarchy Closest to CPU]] · [[Cache Line IO Block Read]] · [[Fetch Data from Disk to RAM]]
- [[Memory — What Will Actually Change How You Think]] · [[Memory-Mapped Files — From Scratch]] · [[Hashing in Memory — How It Actually Works]]
- [[CPU Pipeline & Branch Prediction]] · [[Lock-Free Programming]]
- [[Hypervisor]] · [[Virtualization vs Containerization]] · [[VMs vs Containers]]
- Concurrency: [[Concurrency & Synchronization Primitives — Code Level Guide]] · [[Concurrency & Multitasking From Zero to Production-Ready]] · [[Concurrency Complete Guide]] · [[MN Scheduling]] · [[context switch overhead]]

## Design Patterns & OOP

- [[Factory Pattern]] · [[Registry Pattern]] · [[Singleton]] · [[Strategy Pattern]] · [[Singleton vs Registry]]
- [[Design Patterns Checklist]] · [[Design Patterns in Production]]
- [[SOLID]] · [[Interface vs Abstract Class]]

## DevOps

- [[Docker]] · [[Docker Volumes Mounts and Networking]] · [[Jenkins Complete Guide]]
- [[Cloud & DevOps — From Zero to Backend-Ready]] · [[Deployment Strategies for Backend Engineers]] · [[Git Flow vs GitHub Flow]]

---

← [[Home]]
