---
type: moc
aliases:
  - Problems
  - Systems Problems
  - Interview Problems
tags:
  - moc
  - problems
  - interview
  - system-design
---

# Problems MOC

Scenario-based systems problems for learning and interview practice.

Every folder has the **same layout**:

1. **Scenario** — story, diagnosis, solution discussion  
2. **Companion** — deeper notes on the core technique  

> There is no Problem 5 (numbering gap kept on purpose).

## How to practice

1. Read the scenario first (avoid the companion until you have an answer).
2. Write your own approach: root cause → options → tradeoffs.
3. Compare with the solution section, then read the companion.

## Catalog

| # | Scenario | Companion | Topics |
|---|----------|-----------|--------|
| 1 | [[Our payment API is charging users twice]] | [[Problem 1 Companion]] | idempotency · distributed locks · fencing tokens · exactly-once |
| 2 | [[We lost orders during the flash sale]] | [[Problem 2 Companion]] | race conditions · isolation levels · inventory · concurrency |
| 3 | [[Our notification system is taking down the main app]] | [[Problem 3 Companion]] | backpressure · queues · temporal coupling · rate limiting |
| 4 | [[Our background job is locking the entire table and nobody knows why]] | [[Problem 4 Companion]] | row locks · `SKIP LOCKED` · table locks · VACUUM |
| 6 | [[Our API is slow but only for some users and we can't reproduce it]] | [[Problem 6 Companion]] | observability · latency · tracing · metrics · debugging |
| 7 | [[Our microservices have data that disagrees with each other and nobody knows who's right]] | [[Problem 7 Companion]] | saga · transactional outbox · idempotency · consistency |
| 8 | [[We added a cache and our system got slower and more broken than before]] | [[Problem 8 Companion]] | caching · thundering herd · invalidation · hot keys |
| 9 | [[Our DB queries are searching wrong]] | [[Problem 9 Companion]] | full-text search · indexing · `LIKE` pitfalls · GIN / tsvector |

## Suggested order

**Foundations:** 1 → 2 → 4  
**Scale & coupling:** 3 → 8 → 6  
**Distributed data:** 7 → 9  

## Consistent note template

Each **scenario** starts with:

- `# Problem N — "…"`  
- **Domain** · **Topics** · **In this folder** (link to companion)

Each **companion** starts with:

- `# Companion — …`  
- **Domain** · **Topics** · **Pairs with** (link to scenario)  
- ends with **Next** → scenario + [[Problems MOC]]

## Search by tag

`#idempotency` · `#locking` · `#caching` · `#observability` · `#saga` · `#search` · `#backpressure` · `#microservices` · `#companion`

---

*This folder is self-contained — safe to publish on its own.*
