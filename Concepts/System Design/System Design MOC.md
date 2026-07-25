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
- [[Database Replication]] — RPO/RTO, sync modes, WAL

## Contention & correctness

- [[Dealing with Contention in System Design]] — race conditions on scarce resources (locks, optimistic concurrency, queues)

## Async & long-running work

- [[Long Running Tasks]] — job queues, workers, status tracking, orchestration

## Storage & media

- [[Large Blobs]] — uploads/downloads without routing bytes through the app (presigned URLs, CDN, resumable uploads)
- Related AWS service detail: [[S3]]

## Realtime

- [[Realtime Updates]] — sockets → multi-server → brokers + consistent hashing
- Related: [[WebSocket vs Server-Sent Events (SSE)]] · [[Consistent Hashing]] · [[Message Brokers Guide — RabbitMQ & Kafka]]

## Case studies

- [[Scaling PostgreSQL to 800 Million Users]] — OpenAI Postgres scale notes (public engineering blog summary)

## Related concepts

- [[Thundering Herd Problem]] · [[Consistent Hashing]] · [[Backpressure]] · [[Distributed Lock]]
- [[WAL]] · [[Isolation Level and MVCC]] · [[Scaling Reads]] · [[Scaling Writes]]

---

← [[Home]] · [[Concepts MOC]]
