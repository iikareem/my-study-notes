---
tags:
  - companion
  - database
  - idempotency
  - interview
  - microservices
  - outbox
  - problem-7
  - problems
  - saga
  - system-design
---

# Companion — Saga, Outbox & Idempotency

**Domain:** Distributed Systems & Data Consistency  
**Topics:** saga · outbox · idempotency

**Pairs with:** [[Our microservices have data that disagrees with each other and nobody knows who's right]] · [[Problems MOC]]

> A comprehensive reference for building reliable distributed transactions across microservices.

---

## Table of Contents

1. [Why Each Pattern Alone Falls Short](#1-why-each-pattern-alone-falls-short)
2. [The Gold Standard: All Three Combined](#2-the-gold-standard-all-three-combined)
3. [Pattern Deep Dives](#3-pattern-deep-dives)
    - [Saga Pattern](#31-saga-pattern)
    - [Transactional Outbox Pattern](#32-transactional-outbox-pattern)
    - [Idempotency Pattern](#33-idempotency-pattern)
4. [Saga Strategies](#4-saga-strategies)
    - [Choreography-based Saga + Outbox](#41-choreography-based-saga--outbox)
    - [Orchestrator-based Saga + Outbox](#42-orchestrator-based-saga--outbox)
    - [Choreography vs Orchestrator Comparison](#43-choreography-vs-orchestrator-comparison)
5. [Outbox Relay Strategies](#5-outbox-relay-strategies)
    - [Polling Process](#51-polling-process)
    - [CDC via Debezium](#52-cdc-via-debezium)
    - [Hybrid Approach](#53-hybrid-approach)
    - [Polling vs CDC Comparison](#54-polling-vs-cdc-comparison)
6. [What Each Pattern Protects Against](#6-what-each-pattern-protects-against)
7. [When to Use This Stack](#7-when-to-use-this-stack)
8. [Key Design Rules](#8-key-design-rules)

---

## 1. Why Each Pattern Alone Falls Short

### Saga Alone

Orchestrates distributed transactions and handles compensations, but **message delivery is not guaranteed**. A crash between "state updated" and "event published" silently loses the event entirely.

```
Service A crashes here
        ↓
[state saved] ✅  →  [publish event] ❌  →  Service B never knows
```

### Transactional Outbox Alone

Guarantees event delivery via atomic writes, but if a downstream service processes a message twice (due to retries or broker redelivery), you get **double execution** with no protection.

```
Outbox relay publishes ✅
Broker delivers twice  →  Service B processes twice ❌ (duplicate side effects)
```

### Idempotency Alone

Protects against duplicate processing, but without guaranteed delivery, **events might never arrive** in the first place. Idempotency can't help if the message was never sent.

```
Idempotency key ready ✅
Event never published  →  Service B never receives anything ❌
```

---

## 2. The Gold Standard: All Three Combined

Together, the three patterns cover every failure scenario in distributed systems:

```
Service A                    Message Relay              Service B
─────────                    ─────────────              ─────────
┌─────────────────────────┐                          ┌──────────────────┐
│ BEGIN TRANSACTION       │                          │                  │
│  1. Update local state  │  ──── poll/CDC ────►    │ 2. Check          │
│  2. Write to outbox     │  event + idempotency_key │    idempotency   │
│ COMMIT                  │                          │    table         │
└─────────────────────────┘                          │                  │
                                                     │ 3. If new:       │
        Saga Orchestrator / Choreography             │    process +     │
        ────────────────────────────────             │    mark done     │
        Coordinates steps + compensations            │                  │
                                                     │ 4. If seen:      │
                                                     │    skip (ack)    │
                                                     └──────────────────┘
```

|Layer|Responsibility|
|---|---|
|**Saga**|Defines _what_ to do and how to compensate on failure|
|**Transactional Outbox**|Guarantees events are _delivered at least once_|
|**Idempotency**|Ensures processing is _safe to retry_ (exactly-once semantics)|

---

## 3. Pattern Deep Dives

### 3.1 Saga Pattern

The Saga pattern breaks a long-running distributed transaction into a sequence of local transactions. Each step either succeeds and triggers the next, or fails and triggers **compensating transactions** to undo prior steps.

```
Order Saga:
  Step 1: Reserve Inventory   → compensate: Release Inventory
  Step 2: Charge Payment      → compensate: Refund Payment
  Step 3: Confirm Order       → compensate: Cancel Order

If Step 2 fails:
  → Trigger compensate Step 1 (Release Inventory)
  → Mark saga as FAILED
```

### 3.2 Transactional Outbox Pattern

Instead of publishing directly to a message broker (which is a separate I/O and can fail independently), the service writes the event to an `outbox` table **in the same database transaction** as the state change.

```sql
BEGIN;
  -- State change
  UPDATE orders SET status = 'PENDING' WHERE id = 'order-123';

  -- Event written atomically in the same transaction
  INSERT INTO outbox (id, aggregate_id, event_type, payload, idempotency_key)
  VALUES (
    uuid(),
    'order-123',
    'OrderCreated',
    '{"orderId":"order-123","amount":99.99}',
    'saga-step-1-order-123'
  );
COMMIT;

-- A separate relay process reads outbox and publishes to broker
```

**The key guarantee:** if the transaction commits, the event exists in the outbox. If it rolls back, neither the state change nor the event exists. No inconsistency possible.

### 3.3 Idempotency Pattern

Each step carries an `idempotency_key` that uniquely identifies that specific operation. Before processing, the consumer checks if it has already handled this key.

```sql
BEGIN;
  -- Guard: only process if this key is new
  INSERT INTO processed_events (idempotency_key, processed_at)
  VALUES ('saga-step-1-order-123', NOW())
  ON CONFLICT DO NOTHING;

  -- Only proceed if we actually inserted (i.e., first time seeing this key)
  IF rows_affected > 0 THEN
    UPDATE inventory SET reserved = reserved + 1 WHERE product_id = ?;
  END IF;
COMMIT;
```

**Idempotency key format:** `{saga_id}-{step_id}` — globally unique per step, reused on retries.

---

## 4. Saga Strategies

Both Saga strategies — Choreography and Orchestrator — **rely on the Transactional Outbox**, but use it differently.

### 4.1 Choreography-based Saga + Outbox

Each service reacts to events and publishes the next event. There is no central coordinator. The outbox is **embedded in every service**.

```
Order Service          Inventory Service       Payment Service
─────────────          ─────────────────       ───────────────
BEGIN TX               BEGIN TX                BEGIN TX
 update orders          update inventory         charge payment
 write to outbox        write to outbox          write to outbox
COMMIT                 COMMIT                  COMMIT
    │                       │                       │
    ▼                       ▼                       ▼
[outbox relay]         [outbox relay]          [outbox relay]
    │                       │                       │
    ▼                       ▼                       ▼
OrderCreated ──────► InventoryReserved ──────► PaymentCharged
                                                    │
                                                    ▼
                                             OrderConfirmed
```

**Outbox usage in Choreography:**

```sql
-- Order Service: starts the saga, emits domain event
BEGIN;
  INSERT INTO orders (id, status) VALUES ('order-123', 'PENDING');
  INSERT INTO outbox (event_type, payload, idempotency_key)
  VALUES ('OrderCreated', '{"orderId":"order-123"}', 'order-123-created');
COMMIT;

-- Inventory Service: reacts to OrderCreated, emits next event
BEGIN;
  UPDATE inventory SET reserved = reserved + 1 WHERE product_id = ?;
  INSERT INTO outbox (event_type, payload, idempotency_key)
  VALUES ('InventoryReserved', '{"orderId":"order-123"}', 'order-123-inv-reserved');
COMMIT;

-- Compensation: Payment fails → PaymentFailed event triggers reverse chain
-- Inventory Service reacts to PaymentFailed:
BEGIN;
  UPDATE inventory SET reserved = reserved - 1 WHERE product_id = ?;
  INSERT INTO outbox (event_type, payload, idempotency_key)
  VALUES ('InventoryReleased', '{"orderId":"order-123"}', 'order-123-inv-released');
COMMIT;
```

**Choreography Characteristics:**

|Aspect|Detail|
|---|---|
|Outbox location|**Every service** has its own outbox table|
|Who triggers next step|The **event broker** (services listen to topics)|
|Compensation|Reverse events cascade **backwards** through services|
|Saga state|**Distributed** — implicit across service states|
|Failure visibility|Hard — must trace events across services|
|Coupling|Low — services only know about events|

---

### 4.2 Orchestrator-based Saga + Outbox

A **central orchestrator** commands each service and waits for replies. The outbox is used by **both the orchestrator** (to send commands) **and each service** (to send replies).

```
                    ┌─────────────────────────┐
                    │     Saga Orchestrator   │
                    │  ┌─────────────────┐   │
                    │  │   saga_state    │   │
                    │  │  step: 2        │   │
                    │  │  status: RUNNING│   │
                    │  └─────────────────┘   │
                    │  ┌─────────────────┐   │
                    │  │     outbox      │   │  ◄── Orchestrator uses
                    │  │ ReserveInvCmd   │   │      outbox for commands
                    │  │ ChargePayCmd    │   │
                    │  └─────────────────┘   │
                    └─────────────────────────┘
                          │           ▲
               commands   │           │  replies
                          ▼           │
          ┌───────────────┐     ┌───────────────┐
          │Inventory Svc  │     │ Payment Svc   │
          │ ┌───────────┐ │     │ ┌───────────┐ │
          │ │  outbox   │ │     │ │  outbox   │ │
          │ │InvReserved│ │     │ │PmtCharged │ │
          │ └───────────┘ │     │ └───────────┘ │
          └───────────────┘     └───────────────┘
```

**Outbox usage in Orchestrator:**

```sql
-- Step 1: Orchestrator starts saga, writes COMMAND to its own outbox
BEGIN;
  INSERT INTO saga_state (saga_id, current_step, status)
  VALUES ('saga-123', 'RESERVE_INVENTORY', 'RUNNING');

  INSERT INTO outbox (event_type, payload, idempotency_key)
  VALUES (
    'ReserveInventoryCommand',
    '{"sagaId":"saga-123","orderId":"order-123"}',
    'saga-123-step-reserve'
  );
COMMIT;

-- Step 2: Inventory Service receives command, replies via its own outbox
BEGIN;
  UPDATE inventory SET reserved = reserved + 1 WHERE product_id = ?;
  INSERT INTO outbox (event_type, payload, idempotency_key)
  VALUES (
    'InventoryReservedReply',
    '{"sagaId":"saga-123","success":true}',
    'saga-123-inv-reply'
  );
COMMIT;

-- Step 3: Orchestrator receives reply, advances state + sends next command
BEGIN;
  UPDATE saga_state SET current_step = 'CHARGE_PAYMENT' WHERE saga_id = 'saga-123';
  INSERT INTO outbox (event_type, payload, idempotency_key)
  VALUES (
    'ChargePaymentCommand',
    '{"sagaId":"saga-123","amount":99.99}',
    'saga-123-step-charge'
  );
COMMIT;

-- Step 4: If payment fails, orchestrator issues COMPENSATION commands
BEGIN;
  UPDATE saga_state SET status = 'COMPENSATING' WHERE saga_id = 'saga-123';
  INSERT INTO outbox (event_type, payload, idempotency_key)
  VALUES (
    'ReleaseInventoryCommand',
    '{"sagaId":"saga-123"}',
    'saga-123-compensate-inv'
  );
COMMIT;
```

**Orchestrator Characteristics:**

|Aspect|Detail|
|---|---|
|Outbox location|**Orchestrator + every service** all have outbox tables|
|Message types|Commands (orchestrator) + Reply events (services)|
|Who triggers next step|The **orchestrator** (after receiving a reply)|
|Compensation|Orchestrator **explicitly issues** rollback commands|
|Saga state|**Centralized** — stored in orchestrator's `saga_state` table|
|Failure visibility|Easy — query `saga_state` for full picture|
|Coupling|Higher — services know command contracts|

---

### 4.3 Choreography vs Orchestrator Comparison

||Choreography|Orchestrator|
|---|---|---|
|**Outbox owners**|Every service (domain events)|Orchestrator + every service (commands + replies)|
|**Message types in outbox**|Domain events|Commands (orchestrator) + Reply events (services)|
|**Saga state location**|Implicit / distributed|Explicit `saga_state` table in orchestrator|
|**Compensation trigger**|Failure event cascades|Orchestrator issues explicit compensation commands|
|**Coupling**|Low — services know events only|Higher — services know command contracts|
|**Observability**|Hard to trace flow|Easy — single source of truth|
|**Best for**|Simple, linear flows|Complex flows with branching/conditions|

---

## 5. Outbox Relay Strategies

The relay is the process that reads from the outbox table and publishes events to the message broker. There are two main approaches: **Polling** and **CDC (Change Data Capture)**.

```
┌─────────────────────────────────────────────────────────────┐
│                      Outbox Table                           │
│  id | event_type | payload | status | created_at           │
└─────────────────────────────────────────────────────────────┘
              │                        │
      ┌───────┴───────┐        ┌───────┴────────┐
      │  Polling       │        │  CDC           │
      │  Process       │        │  (Debezium)    │
      │                │        │                │
      │ SELECT * FROM  │        │ reads DB       │
      │ outbox WHERE   │        │ transaction    │
      │ status='UNSENT'│        │ log (WAL)      │
      └───────┬───────┘        └───────┬────────┘
              │                        │
              └───────────┬────────────┘
                          ▼
                   Message Broker
                  (Kafka / RabbitMQ)
```

---

### 5.1 Polling Process

A background thread or scheduled job **periodically queries** the outbox table for unpublished rows.

```
┌─────────────────────────────────────────────────────┐
│                  Polling Relay                       │
│                                                      │
│  every N seconds:                                    │
│  ┌──────────────────────────────────────────────┐   │
│  │ SELECT * FROM outbox                         │   │
│  │ WHERE status = 'PENDING'                     │   │
│  │ ORDER BY created_at                          │   │
│  │ LIMIT 100                                    │   │
│  │ FOR UPDATE SKIP LOCKED  ◄── prevents         │   │
│  └─────────────────────────── double publish    ┘   │
│           │                                          │
│           ▼                                          │
│    publish to broker                                 │
│           │                                          │
│           ▼                                          │
│  UPDATE outbox SET status = 'PUBLISHED'              │
│  WHERE id = ?                                        │
└─────────────────────────────────────────────────────┘
```

**Implementation (Java/Spring):**

```java
@Scheduled(fixedDelay = 1000)
public void relay() {
    List<OutboxEvent> events = db.query("""
        SELECT * FROM outbox
        WHERE status = 'PENDING'
        ORDER BY created_at
        LIMIT 100
        FOR UPDATE SKIP LOCKED
    """);

    for (OutboxEvent event : events) {
        try {
            broker.publish(event.topic(), event.payload());

            db.execute("""
                UPDATE outbox
                SET status = 'PUBLISHED', published_at = NOW()
                WHERE id = ?
            """, event.id());

        } catch (Exception e) {
            db.execute("""
                UPDATE outbox
                SET retry_count = retry_count + 1, last_error = ?
                WHERE id = ?
            """, e.getMessage(), event.id());
        }
    }
}
```

**Polling Characteristics:**

|Aspect|Detail|
|---|---|
|**Latency**|Depends on poll interval (100ms–5s typical)|
|**DB load**|Constant polling queries even when idle|
|**Complexity**|Simple — just a scheduled job|
|**Coupling**|Relay needs DB read credentials|
|**Ordering**|Controlled via `ORDER BY created_at`|
|**Scaling**|`SKIP LOCKED` allows multiple relay instances|
|**Status column**|Required (`PENDING` / `PUBLISHED`)|

---

### 5.2 CDC via Debezium

CDC **tails the database transaction log** (WAL in PostgreSQL, binlog in MySQL) and captures every INSERT to the outbox table in near real-time — without ever querying the table.

```
┌──────────────┐     ┌─────────────────────────────────────┐
│  PostgreSQL  │     │           Debezium                  │
│              │     │                                     │
│  WAL / binlog│────►│  Connector reads log stream         │
│              │     │       │                             │
│  outbox rows │     │       ▼                             │
│  captured as │     │  Transforms row → CloudEvent        │
│  log entries │     │       │                             │
└──────────────┘     │       ▼                             │
                     │  Publishes to Kafka topic           │
                     │  (outbox.order.OrderCreated)        │
                     └─────────────────────────────────────┘
                                    │
                                    ▼
                             Kafka Topic
                             (consumers)
```

**Debezium Connector Config:**

```json
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname": "postgres",
  "database.dbname": "orders_db",
  "table.include.list": "public.outbox",

  "transforms": "outbox",
  "transforms.outbox.type":
    "io.debezium.transforms.outbox.EventRouter",

  "transforms.outbox.table.field.event.id": "id",
  "transforms.outbox.table.field.event.key": "aggregate_id",
  "transforms.outbox.table.field.event.type": "event_type",
  "transforms.outbox.table.field.event.payload": "payload",

  "transforms.outbox.route.by.field": "aggregate_type",
  "transforms.outbox.route.topic.replacement":
    "outbox.${routedByValue}.${eventType}"
}
```

**What CDC Captures:**

```
DB Transaction Log entry:
──────────────────────────
Operation : INSERT
Table     : outbox
Row data  :
  id              = "uuid-001"
  aggregate_type  = "Order"
  aggregate_id    = "order-123"
  event_type      = "OrderCreated"
  payload         = {"orderId":"order-123","amount":99.99}
  idempotency_key = "saga-123-step-1"
  created_at      = 2026-06-27T10:00:00Z

         │
         ▼  Debezium EventRouter transforms to:

Kafka topic  : outbox.Order.OrderCreated
Kafka key    : order-123
Kafka value  : {"orderId":"order-123","amount":99.99}
Kafka header : idempotency_key = saga-123-step-1
```

**CDC Characteristics:**

|Aspect|Detail|
|---|---|
|**Latency**|Near real-time (milliseconds)|
|**DB load**|Zero — reads log, not tables|
|**Complexity**|Higher — Debezium + Kafka Connect infra|
|**Coupling**|No DB credentials needed in relay|
|**Ordering**|Guaranteed — log is sequential|
|**Scaling**|Kafka partitions handle throughput|
|**Status column**|Not needed — log is the source of truth|
|**Table growth**|Rows can be deleted after CDC captures them|

---

### 5.3 Hybrid Approach (Best of Both)

Some teams use **polling as a fallback** for CDC, getting CDC's speed with polling's reliability as a safety net.

```
Normal path  : CDC captures inserts → Kafka (milliseconds)
Fallback path: Polling job runs every 60s → catches anything CDC missed
               (e.g. Debezium downtime, connector lag, missed events)

Outbox row has:
  status       = PENDING | PUBLISHED
  published_at = NULL    | timestamp

CDC   → stateless (marks nothing, Kafka offset tracks progress)
Polling → checks: WHERE status = 'PENDING' AND created_at < NOW() - INTERVAL '30s'
```

This ensures **no event is ever permanently lost**, even during Debezium outages.

---

### 5.4 Polling vs CDC Comparison

||Polling|CDC (Debezium)|
|---|---|---|
|**Latency**|Seconds (poll interval)|Milliseconds|
|**DB overhead**|High (constant queries)|Minimal (log tailing)|
|**Infrastructure**|None extra|Kafka + Kafka Connect|
|**Ordering guarantee**|Weak (clock skew risk)|Strong (log sequence number)|
|**Outbox table growth**|Must purge manually|Delete after capture|
|**Setup complexity**|Low|High|
|**Status column needed**|Yes (`PENDING/PUBLISHED`)|No|
|**Best for**|Small systems, simple setup|High throughput, event-driven|

---

## 6. What Each Pattern Protects Against

|Failure Scenario|Saga|Outbox|Idempotency|
|---|---|---|---|
|Service crash mid-saga|✅ compensate remaining steps|||
|Crash after DB write, before publish||✅ relay retries from outbox||
|Message broker delivers twice|||✅ dedup via idempotency key|
|Consumer crashes after processing, before ack|||✅ dedup on retry|
|Network partition causes retries|||✅ dedup|
|Partial saga failure (step N fails)|✅ compensate steps 1..N-1|||
|Relay crashes mid-publish||✅ restart and republish||
|Broker redelivers due to consumer lag|||✅ dedup|

---

## 7. When to Use This Stack

This combination is **essential** whenever:

- A business operation spans **two or more services** with **no shared database**
- Each step has **side effects** (charges money, reserves stock, sends email)
- The operation must either **fully complete or fully rollback**
- You need **audit traceability** of every event

**Common use cases:**

- **E-commerce:** order → inventory → payment → fulfillment → notification
- **Banking:** multi-account transfers with ledger entries
- **Travel booking:** flight + hotel + car rental in a single reservation
- **SaaS onboarding:** account creation → provisioning → billing → welcome email

---

## 8. Key Design Rules

### Universal Outbox Contract

Regardless of Saga strategy or relay method, the contract never changes:

```
State change  ──┐
                ├── Single atomic transaction ──► outbox row
Event/Command ──┘                                    │
                                                     ▼
                                              Relay publishes
                                                     │
                                                     ▼
                                         Broker delivers (with retries)
                                                     │
                                                     ▼
                                        Consumer checks idempotency key
                                                     │
                                          ┌──────────┴──────────┐
                                          ▼                     ▼
                                     First time:           Already seen:
                                     process + mark        skip + ack
```

### Rules to Follow

1. **Idempotency key = saga ID + step ID** — globally unique per step, reused on retries.
2. **Idempotency key travels with the event** — written in outbox, passed through broker as a header, checked by consumer.
3. **Compensating transactions must also be idempotent** — compensations can also be retried.
4. **Saga state lives in DB, not in memory** — so restarts can resume or compensate correctly.
5. **Never publish directly to a broker** — always go through the outbox, even in compensations.
6. **Use `FOR UPDATE SKIP LOCKED`** for polling — prevents multiple relay instances from double-publishing.
7. **Outbox rows should be purged** — archive or delete `PUBLISHED` rows to keep the table lean.
8. **One outbox table per service** — each service owns its own outbox, never shares across service boundaries.

---

## Summary

|Pattern|Role|Guarantees|
|---|---|---|
|**Saga**|Defines the distributed workflow and compensations|Business consistency across services|
|**Transactional Outbox**|Atomically binds state change to event publication|At-least-once delivery|
|**Idempotency**|Deduplicates retried or redelivered messages|Exactly-once processing semantics|
|**Polling Relay**|Simple relay: query outbox → publish → mark done|Easy to implement, higher DB load|
|**CDC Relay**|Stream relay: tail DB log → publish to Kafka|Low latency, low DB load, complex infra|

Together, these patterns give you:

> **Atomicity across services** (Saga) + **at-least-once delivery** (Outbox) + **exactly-once semantics** (Idempotency)

...which together approximate **distributed ACID guarantees** without a distributed transaction coordinator.

## Next

- Scenario: [[Our microservices have data that disagrees with each other and nobody knows who's right]]
- Back to hub: [[Problems MOC]]
