---
type: moc
aliases:
  - System Design
  - System Design Patterns
  - Scaling
tags:
  - moc
  - system-design
---

# System Design MOC

Patterns for scaling, storage, async work, and realtime — connected to database & messaging concepts.

## Scaling

- [[Scaling Reads vs Writes]] — **start here** (complete guide)
- [[Scaling Reads]] — read progression (replicas, cache, hot keys)
- [[Scaling Writes]] — write path, domain partition → shard, primary choke points
- [[Hot Key Problem]] — concentrating load on one key / partition
- [[Database Replication]] — RPO/RTO, sync modes, WAL

## Caching

- [[Caching Explained Simply]] · [[Caching Made Simple]]

## Contention & correctness

- [[Dealing with Contention in System Design]] — race conditions on scarce resources (locks, optimistic concurrency, queues)
- [[Multi-Step Processes in System Design]] — sagas / workflows across services

## Async & long-running work

- [[Long Running Tasks]] — job queues, workers, status tracking, orchestration

## Storage & media

- [[Large Blobs]] — uploads/downloads without routing bytes through the app (presigned URLs, CDN, resumable uploads)
- [[Object Storage Design Patterns]]
- Related AWS service detail: [[S3]]

## Infrastructure & routing

- [[Infrastructure Routing The Complete Mental Model]]
- [[Deployment Strategies, API Versioning & WebSocket Gateways]]
- Related DevOps: [[Deployment Strategies for Backend Engineers]]

## Realtime

- [[Realtime Updates]] — sockets → multi-server → brokers + consistent hashing
- [[Realtime Scaling WebSocket & SSE]]
- [[WebSocket Gateway Pattern]] · [[WebSocket Gateway Study Notes]]
- Related: [[WebSocket vs Server-Sent Events (SSE)]] · [[WebSocket API Design Guide]] · [[Consistent Hashing]] · [[Message Brokers Guide — RabbitMQ & Kafka]]

## Decomposition & NFRs

- [[Service and DB Decomposition Guide]]
- [[NFR and Estimation Roadmaps]]
- [[System Design Review Checklist]] · [[Systems Design Recap Notes]]

## Case studies

- [[Scaling PostgreSQL to 800 Million Users]] — OpenAI Postgres scale notes (public engineering blog summary)

## Related concepts

- [[Thundering Herd Problem]] · [[Consistent Hashing]] · [[Backpressure]] · [[Distributed Lock]] · [[Redlock and CS Comparison]]
- [[WAL]] · [[Isolation Level and MVCC]] · [[Scaling Reads]] · [[Scaling Writes]]

---

← [[Home]] · [[Concepts MOC]]
