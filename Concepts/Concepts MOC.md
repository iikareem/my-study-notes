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
- [[HTTP]] · [[WebSocket vs Server-Sent Events (SSE)]]
- [[ARP]] · [[CIDR Notation]] · [[IP Address]] · [[MAC Address]]
- [[Network Routing]] · [[Protocols]]
- [[Bandwidth vs Throughput]] · [[Network Request and Response Stages]]

## Distributed Systems

- [[Consistent Hashing]] · [[Distributed Lock]] · [[Fencing Token]]
- [[Backpressure]] · [[Thundering Herd Problem]] · [[Clock Skew]]
- [[HOL]] · [[Mechanical Sympathy]] · [[Memory Ordering]]
- [[Pipeline and Branch]] · [[Microservices]]

## System Design

→ Full hub: [[System Design MOC]]

- [[Scaling Reads vs Writes]] · [[Scaling Reads]] · [[Scaling Writes]]
- [[Dealing with Contention in System Design]] · [[Long Running Tasks]] · [[Large Blobs]]
- [[Database Replication]] · [[Realtime Updates]]
- [[Scaling PostgreSQL to 800 Million Users]]

## Database

- [[Isolation Level and MVCC]] · [[VACUUM]] · [[WAL]] · [[HOT]] · [[TOAST Table]]
- [[Lock in Database]] · [[Explain Analyze]] · [[pg_stat_statements]] · [[pgstatindex]]
- [[BRIN]] · [[Btree vs LSM]] · [[LFC]] · [[Hint Bit Update]] · [[Prewarm]]
- [[Prepared Statements — The Optimization Story]] · [[Extended Statistics in PostgreSQL]]
- [[PostgreSQL Bulk Updates The Hidden Cost of Doing It All at Once]] · [[Monotonically Increasing vs Random Index Writes]]
- [[PostgreSQL Parallel Query Execution]] · [[PostgreSQL 18]]
- [[PostgreSQL Memory Management Understanding the 25% Shared Buffers Rule]]
- [[Database Architecture Notes PostgreSQL vs MySQL]] · [[PG and MySQL]]
- [[Database Replication]] · [[Search Types Mandatory Knowledge Guide]] · [[Books]]
- Join algorithms: [[Hash Join]] · [[Nested Loop Join]]
- Design / optimization: [[3VL]] · [[Where different condition types]] · [[join_collapse_limit]]
- OLAP foundations: [[00 OLAP MOC|OLAP MOC]] · [[01 Module 1 — OLTP vs OLAP Mental Model|OLTP vs OLAP]] · [[02 Module 2 — Columnar Execution Internals|Columnar Internals]] · [[04 Module 3 — Parquet & File Formats|Parquet]] · [[05 Module 4 — Partitioning & Data Layout|Partitioning]] · [[06 Module 5 — Star Schemas & Dimensional Design|Star Schemas]]

## Backend

- [[Node.js Event Loop & Internals]] *(merged)*
- [[V8 Hidden Classes]] · [[TypeScript Promises]] · [[Request Lifecycle in Nest]]
- [[Message Brokers Guide — RabbitMQ & Kafka]] · [[Messaging RabbitMQ and Kafka]]
- [[DDD]] · [[Parquet]] · [[Programming Language]]

## Security

- [[PASETO]]

## Observability

- [[Observability Stack Layers Contracts Swappability]]
- Logs: [[Structured Logs]] · [[Pino vs Winston]] · [[Logging Winston vs Pino Deep Dive]] · [[Logging Additional Notes]]
- Rate limiting: [[Rate Limiting]] · [[Adaptive]]

## Operating System

- [[Process]] · [[Process Life Cycle]] · [[Process Management]] · [[Worker]]
- [[Kernel]] · [[Virtual Memory]] · [[Context Switching]] · [[vCPU]]
- [[Memory Fragmentation]] · [[Memory Hierarchy Closest to CPU]] · [[Cache Line IO Block Read]] · [[Fetch Data from Disk to RAM]]
- [[CPU Pipeline & Branch Prediction]]
- [[Hypervisor]] · [[Virtualization vs Containerization]] · [[VMs vs Containers]]
- Concurrency: [[Concurrency & Synchronization Primitives — Code Level Guide]] · [[Concurrency & Multitasking From Zero to Production-Ready]] · [[Concurrency Complete Guide]] · [[MN Scheduling]] · [[context switch overhead]]

## Design Patterns & OOP

- [[Factory Pattern]] · [[Registry Pattern]] · [[Singleton]] · [[Strategy Pattern]] · [[Singleton vs Registry]]
- [[SOLID]] · [[Interface vs Abstract Class]]

## DevOps

- [[Docker]] · [[Docker Volumes Mounts and Networking]] · [[Jenkins Complete Guide]]

---

← [[Home]]
