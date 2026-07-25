---
tags:
  - system-design
  - contention
  - locking
  - concurrency
---

# Dealing with Contention in System Design


> **Problem:** Multiple users competing for the same limited resource simultaneously — tickets, auction bids, seat reservations, flash sale inventory — creates race conditions, double-bookings, and lost updates.

---

## Table of Contents

1. [Understanding the Problem](#1-understanding-the-problem)
2. [Race Condition Deep Dive](#2-race-condition-deep-dive)
3. [Solution Landscape](#3-solution-landscape)
4. [Single-Database Solutions](#4-single-database-solutions)
5. [Distributed Solutions](#5-distributed-solutions)
6. [Simplification Patterns](#6-simplification-patterns)
7. [Solution Comparison & Tradeoffs](#7-solution-comparison--tradeoffs)
8. [Choosing the Right Solution](#8-choosing-the-right-solution)
9. [Interview Playbook](#9-interview-playbook)

---

## 1. Understanding the Problem

Contention happens when **read-modify-write** cycles overlap across concurrent requests. The system reads a value, decides to act on it, and writes back — but between the read and the write, another process has already changed it.

### Classic Contention Scenarios

|Scenario|Resource|Consequence of Race|
|---|---|---|
|Ticket booking|Seats|Overselling|
|Flash sale|Inventory count|Negative stock|
|Auction|Highest bid|Two users "win"|
|Ride sharing|Driver assignment|Double dispatch|
|Bank transfer|Account balance|Overdraft|
|Hotel reservation|Room availability|Double booking|

### Why It's Hard

- **Speed mismatch:** Network latency means hundreds of requests arrive before any single one completes.
- **Optimism is wrong:** Assuming "it's probably fine" breaks under real load.
- **Distributed systems amplify it:** No global clock, no single source of truth.

---

## 2. Race Condition Deep Dive

### The Classic Lost Update

```
Time →        T1                    T2
──────────────────────────────────────────────────────
T=0     Read tickets = 1      Read tickets = 1
T=1     tickets > 0? YES      tickets > 0? YES
T=2     Book ticket           Book ticket
T=3     Write tickets = 0     Write tickets = 0
──────────────────────────────────────────────────────
Result: 2 bookings for 1 ticket ← RACE CONDITION
```

Both threads read `1`, both see the condition is satisfied, both write `0`. The second write silently overwrites the first. **One ticket sold twice.**

### Why Simple Checks Fail

```sql
-- This is NOT safe under concurrent load:
SELECT available FROM tickets WHERE id = 1;
-- (another request runs here)
IF available > 0 THEN
  UPDATE tickets SET available = available - 1 WHERE id = 1;
  INSERT INTO bookings ...
END IF;
```

The gap between `SELECT` and `UPDATE` is the vulnerability window.

---

## 3. Solution Landscape

```
                    CONTENTION SOLUTIONS
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
   Single DB          Distributed      Simplification
   Solutions           Solutions         Patterns
         │                 │                 │
   ┌─────┤          ┌──────┤           ┌─────┤
   │     │          │      │           │     │
Atomic  DB       Distrib  2-Phase   Batching  Time
 Ops  Trans-    Locks     Commit             Windows
      actions   (Redis)  (2PC)
```

---

## 4. Single-Database Solutions

### 4.1 Atomic SQL Operations

The simplest fix: **never separate the read from the write.**

```sql
-- Safe: atomic decrement with guard
UPDATE tickets
SET available = available - 1
WHERE id = 1 AND available > 0;

-- Check affected rows — if 0, ticket was already gone
```

**How it works:** The database engine serializes this at the row level. The condition and update execute as one indivisible operation.

**When to use:** Single table, simple decrement. Great for inventory counts.

**Tradeoffs:**

- ✅ Simple, no extra infrastructure
- ✅ Zero extra latency
- ❌ Only works within one database
- ❌ Doesn't help if you need multi-step operations (book + charge + send email)

---

### 4.2 Pessimistic Locking (SELECT FOR UPDATE)

Lock the row before reading it. Other transactions block until you release.

```sql
BEGIN;

SELECT available FROM tickets
WHERE id = 1
FOR UPDATE;  -- row is now locked

-- No other transaction can touch this row until COMMIT/ROLLBACK

IF available > 0 THEN
  UPDATE tickets SET available = available - 1 WHERE id = 1;
  INSERT INTO bookings (user_id, ticket_id) VALUES (?, ?);
  COMMIT;
ELSE
  ROLLBACK;
END IF;
```

**How it works:** The database holds an exclusive lock on the row. All other requests for that row queue up and wait.

**When to use:** High-conflict scenarios where most reads will lead to writes. Auction final seconds. Seat selection.

**Tradeoffs:**

- ✅ Guarantees correctness
- ✅ Straightforward mental model
- ❌ Serializes requests → throughput bottleneck
- ❌ Deadlock risk if locking multiple rows in different orders
- ❌ Long-running transactions block everything

---

### 4.3 Optimistic Locking (Version Numbers)

Read freely, but detect conflicts at write time using a version counter.

```sql
-- Schema: add a version column
ALTER TABLE tickets ADD COLUMN version INT DEFAULT 0;

-- Application flow:
-- 1. Read with version
SELECT available, version FROM tickets WHERE id = 1;
-- Returns: available=1, version=42

-- 2. Try to write, but only if version hasn't changed
UPDATE tickets
SET available = available - 1,
    version = version + 1
WHERE id = 1
  AND version = 42     -- ← conflict detection
  AND available > 0;

-- If 0 rows updated → conflict → retry or return error
```

**How it works:** If another transaction committed between your read and write, the version will have changed and your update affects 0 rows — a safe failure you can detect and retry.

**When to use:** Low-to-medium contention. Most of the time users aren't competing for the same exact resource. Read-heavy workloads.

**Tradeoffs:**

- ✅ No locking → high read throughput
- ✅ Scales well under low contention
- ❌ Under high contention, many retries → worse than pessimistic locking
- ❌ Application must handle retry logic
- ❌ Still within one database

---

### 4.4 Database Transactions (ACID)

Wrap multi-step operations in a transaction to ensure all-or-nothing execution.

```sql
BEGIN TRANSACTION;

  UPDATE accounts SET balance = balance - 100 WHERE id = 'sender';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'receiver';
  INSERT INTO transaction_log ...;

COMMIT;
-- If anything fails: ROLLBACK — both updates are undone
```

**Isolation levels** let you tune the tradeoff between correctness and performance:

|Level|Phantom Reads|Non-repeatable Reads|Dirty Reads|Performance|
|---|---|---|---|---|
|READ UNCOMMITTED|✅ possible|✅ possible|✅ possible|Highest|
|READ COMMITTED|✅ possible|✅ possible|❌ prevented|High|
|REPEATABLE READ|✅ possible|❌ prevented|❌ prevented|Medium|
|SERIALIZABLE|❌ prevented|❌ prevented|❌ prevented|Lowest|

**When to use:** Any multi-step operation that must be atomic. Financial transfers, booking + payment.

**Tradeoffs:**

- ✅ Strong consistency guarantees
- ✅ Well-understood by databases
- ❌ Locks held during transaction → contention under load
- ❌ Long transactions kill throughput
- ❌ Doesn't cross database boundaries

---

## 5. Distributed Solutions

When your data is sharded, replicated, or lives in multiple services, single-DB mechanisms no longer apply.

### 5.1 Distributed Locks (Redis)

Use Redis as a coordination layer. One holder wins the lock; everyone else waits or backs off.

```
User A ──► Try LOCK(ticket:42) ──► Redis: SET ticket:42 1 NX PX 5000
                                         (NX = only if not exists, PX = TTL ms)
           Got lock! ─────────────────────────────────────────┐
                                                               │
User B ──► Try LOCK(ticket:42) ──► Redis: already set         │
           Retry / fail ◄──────────────────────────────────────┘
                                                               │
User A finishes, calls DEL(ticket:42) ◄────────────────────────┘
```

**Implementation with Redlock (multi-node):**

```
Lock must be acquired on majority (N/2 + 1) of N Redis nodes
to prevent split-brain scenarios.

Node 1: SET ticket:42 <uuid> NX PX 5000  → OK
Node 2: SET ticket:42 <uuid> NX PX 5000  → OK
Node 3: SET ticket:42 <uuid> NX PX 5000  → FAIL
Node 4: SET ticket:42 <uuid> NX PX 5000  → OK
Node 5: SET ticket:42 <uuid> NX PX 5000  → OK

4/5 nodes acquired → lock granted ✅
```

**Critical details:**

- Always use a **UUID value** so only the lock owner can release it
- Always set a **TTL** to auto-release if the holder crashes
- Use **Redlock** for multi-node safety — single Redis node is not safe for critical locks

**When to use:** Cross-service coordination. Preventing duplicate job processing. Rate limiting per user.

**Tradeoffs:**

- ✅ Works across services and databases
- ✅ Redis is fast (sub-millisecond)
- ❌ Still a single point of coordination (Redis cluster helps)
- ❌ Lock expiry is tricky: too short → holder loses lock mid-operation; too long → failure blocks everyone
- ❌ Network partition can cause split-brain (process holds lock but can't release)
- ❌ Redlock is [controversial](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) for truly critical operations

---


## 6. Simplification Patterns

Interviewers specifically probe whether you can **redesign the problem** to reduce contention rather than fight it directly. These patterns are often the right answer.

### 6.1 Queue + Serial Processing

Funnel all requests through a queue. A single worker processes them one at a time — no concurrency, no contention.

```
Users                Queue              Worker         Database
──────────────────────────────────────────────────────────────
User A ──► Enqueue ──►│ [A, B, C, D] │──► Process A ──► Write
User B ──► Enqueue ──►│               │──► Process B ──► Write
User C ──► Enqueue ──►│               │──► Process C ──► Write
User D ──► Enqueue ──►│               │   (D: sold out, reject)
```

**Key properties:**

- Queue is durable (Kafka, SQS, RabbitMQ)
- Users get a "request received" response immediately
- Outcome is delivered via webhook, polling, or notification
- Worker is the only writer — no locking needed

**When to use:** Flash sales, ticket drops, auction close. Any scenario where demand far exceeds supply and fairness (FIFO) is acceptable.

**Tradeoffs:**

- ✅ Eliminates contention entirely at the database level
- ✅ Natural backpressure — queue absorbs traffic spikes
- ✅ Easy to scale workers independently
- ❌ Users don't get instant confirmation — UX friction
- ❌ Queue can grow unboundedly under extreme load
- ❌ Need to handle duplicate messages (idempotency keys)

---

### 6.2 Pre-allocation / Claim Tokens

Pre-generate a fixed pool of claimable tokens before the sale begins. Distributing tokens is cheaper than competing to decrement a counter.

```
Before sale:
  Generate 1000 unique tokens: [TKT-001, TKT-002, ..., TKT-1000]
  Insert into available_tokens table

During sale:
  UPDATE available_tokens
  SET claimed_by = ?, claimed_at = NOW()
  WHERE id = (
    SELECT id FROM available_tokens
    WHERE claimed_by IS NULL
    LIMIT 1
    FOR UPDATE SKIP LOCKED   ← each worker grabs a different row
  )
```

**`SKIP LOCKED`** is key: workers skip rows already locked by other workers, avoiding contention entirely. Each worker claims a different token.

**When to use:** Fixed inventory (seats, hotel rooms, numbered items). Works well with the queue pattern above.

**Tradeoffs:**

- ✅ Near-zero contention at row level
- ✅ Scales to many parallel workers
- ❌ Pre-allocation work upfront
- ❌ Tokens held by crashed workers need expiry/reclaim logic

---

### 6.3 Batching Requests

Instead of processing each request individually, collect requests over a short window and process them together.

```
Window = 100ms

T=0ms:   User A requests ticket
T=20ms:  User B requests ticket
T=60ms:  User C requests ticket
T=80ms:  User D requests ticket
T=100ms: WINDOW CLOSES

Batch processor:
  Available = 2
  Winners (random or FIFO): [A, C]
  Losers: [B, D]
  → Send results to all four users
```

**When to use:** Auction bids (collect bids in 5-second windows, determine winner), leaderboard updates, rate-limited APIs.

**Tradeoffs:**

- ✅ Drastically reduces write frequency
- ✅ Naturally handles bursts
- ❌ Adds latency (users wait for window to close)
- ❌ Requires fair winner selection logic
- ❌ Users may be confused by delayed responses

---

### 6.4 Time-Window Bidding

A specific pattern for auctions: rather than processing every bid in real-time, accept bids during a window and determine the winner at close.

```
Auction Window (e.g., 5 minutes)
──────────────────────────────────────────────────────────────
Bids accepted: [A:$50, B:$75, C:$60, D:$80, E:$70]
All stored as-is — no real-time winner computation

Window closes:
  SELECT MAX(amount), user_id FROM bids WHERE auction_id = ? 
  ORDER BY amount DESC, submitted_at ASC  -- tiebreak by time
  LIMIT 1
```

**Anti-sniping extension:** If a bid arrives in the last 60 seconds, extend the window by 60 seconds. Prevents last-second unfair wins and reduces contention at the "deadline spike."

**When to use:** Auctions. Any situation where the "final state" matters, not the real-time state.

**Tradeoffs:**

- ✅ Zero contention — writes are independent inserts, not competing updates
- ✅ Full bid history preserved
- ❌ Users don't know if they're winning until window closes
- ❌ Requires window management logic

---

### 6.5 Reservation + Expiry (Two-Phase Booking)

Reserve the resource for a short window to let the user complete checkout, then release if not confirmed.

```
Phase 1: Hold (30 seconds)
  UPDATE tickets SET status='held', held_by=?, held_until=NOW()+30s
  WHERE status='available' LIMIT 1

  → User sees "checkout" screen

Phase 2: Confirm or Release
  -- User completes payment:
  UPDATE tickets SET status='sold', sold_to=? WHERE held_by=? AND held_until > NOW()

  -- User abandons / times out:
  UPDATE tickets SET status='available', held_by=NULL WHERE held_until < NOW()
```

**When to use:** E-commerce checkout, airline seat selection, hotel booking.

**Tradeoffs:**

- ✅ Good UX — user gets guaranteed time to complete purchase
- ✅ Inventory eventually returns to pool if uncompleted
- ❌ Temporarily reduces apparent availability
- ❌ Need background job to reclaim expired holds
- ❌ Speculative holds can be abused (holding without intent to buy)

---

## 7. Solution Comparison & Tradeoffs

### At a Glance

|Solution|Consistency|Throughput|Latency|Complexity|Best For|
|---|---|---|---|---|---|
|Atomic SQL ops|Strong|High|Low|Low|Simple counter decrement|
|Pessimistic locking|Strong|Low|Medium|Low|High-conflict, short operations|
|Optimistic locking|Strong*|High|Low|Medium|Low-conflict scenarios|
|DB Transactions|Strong|Medium|Medium|Low|Multi-step, single DB|
|Distributed Locks|Strong|Medium|Medium|High|Cross-service coordination|
|Two-Phase Commit|Strong|Very Low|High|Very High|Cross-DB atomic ops|
|Saga Pattern|Eventual|High|Low|High|Microservices workflows|
|Queue + Serial|Strong|High**|High|Medium|Flash sales, ticket drops|
|Pre-allocation|Strong|Very High|Low|Medium|Fixed inventory|
|Batching|Eventual|Very High|Medium|Medium|Auctions, leaderboards|
|Reservation+Expiry|Strong|Medium|Low|Medium|E-commerce checkout|

_*Optimistic locking degrades to eventual under high contention due to retries_ _**High throughput at DB level; queue itself is the bottleneck_

---

### Contention vs. Throughput Curve

```
          High │ Optimistic    Pre-allocation
   Throughput  │  locking      Queue+Serial
               │     ●               ●
               │         Batching
               │             ●
               │  DB Transactions
               │        ●
               │ Pessimistic     2PC
          Low  │  locking         ●
               │    ●
               └──────────────────────────────
                Low               High
                     Contention Level
```

---

## 8. Choosing the Right Solution

### Decision Tree

```
Is data in a single database?
├── YES
│   ├── Is the operation a simple counter decrement?
│   │   └── YES → Atomic SQL (UPDATE ... WHERE count > 0)
│   ├── Is contention HIGH (many users, same resource)?
│   │   └── YES → Pessimistic locking (SELECT FOR UPDATE)
│   ├── Is contention LOW (occasional conflicts)?
│   │   └── YES → Optimistic locking (version column)
│   └── Is it multi-step (book + charge + notify)?
│       └── YES → DB Transaction with appropriate isolation level
│
└── NO (distributed / microservices)
    ├── Can you tolerate latency & simplify the model?
    │   ├── FIFO is acceptable → Queue + serial processing
    │   ├── Inventory is pre-known → Pre-allocation + SKIP LOCKED
    │   ├── Real-time not required → Batching / time windows
    │   └── Users need checkout time → Reservation + expiry
    │
    └── Must be truly atomic across services?
        ├── Short operation, can use Redis → Distributed lock
        ├── Can tolerate eventual consistency → Saga pattern
        └── Must be synchronously atomic → 2PC (use sparingly)
```

---

### Interview Signal: When to Propose Simplification

The interviewer will often push: _"What if 100,000 users hit the ticket drop simultaneously?"_

The correct instinct is to **reframe the problem**, not just add more locks:

1. **"Can we use a queue?"** — Decouple the request from the fulfillment. Users get a confirmation immediately; winners are determined serially.
    
2. **"Can we pre-generate tokens?"** — Turn a shared counter into independent rows. Workers use `SKIP LOCKED` and never contend.
    
3. **"Does it need to be real-time?"** — Batch bids in 5-second windows, determine winners at close. Reduces 100,000 writes/sec to 1 write per window.
    
4. **"Can we use reservations?"** — Hold the resource for 2 minutes while the user checks out. Only 1 lock per session, not per click.
    

---

## 9. Interview Playbook

### Opening Framework

When the interviewer introduces a contention problem, say:

> _"The core issue is a read-modify-write race condition. Before reaching for distributed locks, let me check if we can simplify the model — queue-based processing or batching often eliminates the problem rather than managing it."_

### Key Questions to Ask

- How many concurrent users do we expect at peak?
- Does the user need instant confirmation, or can we confirm asynchronously?
- Is the inventory fixed and pre-known?
- Is ordering/fairness (FIFO) required, or is random selection okay?
- Are we in a single database or distributed across services?

### Red Flags to Avoid

|Don't say|Say instead|
|---|---|
|"Just use a transaction"|"A transaction works within one DB; at scale we should consider queuing"|
|"We'll add more locks"|"More locking serializes more — let's reduce what needs locking"|
|"Use Redis for everything"|"Redis distributed locks add complexity and failure modes; let's first see if a simpler model works"|
|"2PC is the standard solution"|"2PC blocks on coordinator failure — Saga or queuing are usually better for this"|

### Concise Tradeoff Statement (memorize this)

> _"Pessimistic locking is correct but serializes throughput. Optimistic locking is fast but degrades under high contention. Distributed locks add coordination overhead and failure modes. The cleanest solutions rethink the request model: queues for fairness, batching for throughput, pre-allocation for known inventory. The right choice depends on whether users need synchronous confirmation and how much contention we actually expect."_

---

## Summary

|Situation|Recommended Solution|
|---|---|
|Simple inventory counter, single DB|Atomic SQL (`UPDATE ... WHERE count > 0`)|
|High-conflict, short multi-step, single DB|Pessimistic locking + Transaction|
|Low-conflict, read-heavy, single DB|Optimistic locking (version column)|
|Fixed inventory, high throughput needed|Pre-allocation + `SKIP LOCKED`|
|Flash sale / ticket drop|Queue + serial processing|
|Auction or non-real-time bidding|Batching / time windows|
|E-commerce checkout flow|Reservation + expiry|
|Cross-service, brief coordination|Distributed lock (Redis)|
|Cross-service, can tolerate eventual|Saga pattern|
|Cross-service, must be synchronous|2PC (last resort)|

> **The interviewer's meta-question is always:** _Can you identify when a problem is really a coordination problem in disguise, and can you simplify the model so coordination becomes unnecessary?_

---

## Related

- [[System Design MOC]]
- [[Distributed Lock]] · [[Fencing Token]] · [[Lock in Database]]
- [[Scaling Writes]] — write-path pressure related to hot rows
- [[Isolation Level and MVCC]]

← [[System Design MOC]] · [[Concepts MOC]]
