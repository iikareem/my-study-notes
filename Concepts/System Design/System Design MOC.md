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

Patterns for scaling and realtime — connected to database & messaging concepts.

## Scaling

- [[Scaling Reads vs Writes]] — **start here** (complete guide)
- [[Scaling Reads]] — read progression (replicas, cache, hot keys)
- [[Scaling Writes]] — write path, sharding, primary choke points
- [[Database Replication]] — RPO/RTO, sync modes, WAL

## Realtime

- [[Realtime Updates]] — sockets → multi-server → brokers + consistent hashing
- Related: [[WebSocket vs Server-Sent Events (SSE)]] · [[Consistent Hashing]] · [[Message Brokers Guide — RabbitMQ & Kafka]]

## Case studies

- [[Scaling PostgreSQL to 800 Million Users]] — OpenAI Postgres scale notes (public engineering blog summary)

## Related concepts

- [[Thundering Herd Problem]] · [[Consistent Hashing]] · [[Backpressure]]
- [[WAL]] · [[Isolation Level and MVCC]] · [[Scaling Reads]] · [[Scaling Writes]]

---

← [[Home]] · [[Concepts MOC]]
