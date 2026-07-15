---
tags:
  - backend
  - competing-consumers
  - kafka
  - messaging
  - prefetch
  - rabbitmq
  - scaling
---

# Message Brokers Guide — RabbitMQ & Kafka

---

## Table of Contents

1. [RabbitMQ Competing Consumers](#1-rabbitmq-competing-consumers)
2. [RabbitMQ Exchange Patterns](#2-rabbitmq-exchange-patterns)
3. [RabbitMQ Horizontal Scaling](#3-rabbitmq-horizontal-scaling)
4. [Prefetch — Fair Dispatch](#4-prefetch--fair-dispatch)
5. [RabbitMQ vs Kafka](#5-rabbitmq-vs-kafka)
6. [Kafka Static Membership](#6-kafka-static-membership)

---

## 1. RabbitMQ Competing Consumers

### The Common Misconception

Many developers assume:

> "One queue can only be subscribed by one consumer"

This is **incorrect**. That model describes Kafka (one partition = one consumer per group). RabbitMQ works differently.

### The Truth

**Multiple consumers can subscribe to the same queue.** RabbitMQ distributes messages between them using round-robin.

```
Producer → [Queue] → Consumer A
                   → Consumer B
                   → Consumer C
```

Each message is delivered to **only one** consumer. This is the competing consumer pattern — workers compete to grab the next available message.

### How to Implement It

```javascript
// Both workers subscribe to the SAME queue — that's it!
channel.consume('tasks', handleTask); // Worker 1
channel.consume('tasks', handleTask); // Worker 2
```

No special configuration needed. RabbitMQ handles distribution automatically.

---

## 2. RabbitMQ Exchange Patterns

### Pattern 1 — Work Queue (Competing Consumers)

Multiple consumers on one queue. Each message goes to one consumer only.

```
Producer → Exchange → [Single Queue] → Worker 1
                                     → Worker 2
                                     → Worker 3
```

**Use case:** Task queues, background jobs, scaling workers.

---

### Pattern 2 — Fanout (Pub/Sub)

One message is copied to all queues. Every consumer gets every message.

```
Producer → Fanout Exchange → [Queue A] → Consumer A
                           → [Queue B] → Consumer B
                           → [Queue C] → Consumer C
```

**Use case:** Broadcasting events — notifications, cache invalidation, logging.

> ⚠️ This is NOT competing consumers. Every consumer receives a **copy** of every message.

---

### Pattern 3 — Direct / Topic Exchange

Route messages by routing key to specific queues, then scale consumers per queue.

```
Producer → Direct Exchange --[key: "email"]--> [Email Queue] → Worker 1
                                                             → Worker 2
           Direct Exchange --[key: "sms"]---> [SMS Queue]   → Worker 3
```

**Use case:** Routing different job types to specialized workers.

---

### Pattern Summary

|Pattern|Queues|Consumers|Each message delivered to|
|---|---|---|---|
|**Competing Consumers**|1|Many|One consumer only|
|**Fanout (Pub/Sub)**|Many (1 per consumer)|Many|All consumers|
|**Direct/Topic**|Many (by routing key)|Many per queue|One per queue|

---

## 3. RabbitMQ Horizontal Scaling

RabbitMQ natively supports horizontal scaling. Just run more instances of your service pointing at the same queue.

```
                        ┌─────────────┐
                        │             │──→ Consumer 1 (Pod/Instance)
Producer → [Queue] ─────│  RabbitMQ   │──→ Consumer 2 (Pod/Instance)
                        │             │──→ Consumer 3 (Pod/Instance)
                        └─────────────┘
```

- Each message is delivered to **only one** consumer
- RabbitMQ uses **round-robin** by default
- Spin consumers up/down freely — RabbitMQ auto-balances

### Scaling with Docker Compose

```bash
# Scale to 5 worker instances instantly
docker compose up --scale worker=5
```

All 5 workers connect to the same queue. RabbitMQ distributes the load automatically.

---

## 4. Prefetch — Fair Dispatch

### The Problem Without Prefetch

RabbitMQ pre-fetches messages greedily by default:

```
Consumer 1 (fast)          Consumer 2 (slow)
─────────────────          ─────────────────
← msg #1, #2, #3, #4      ← msg #5
✅ done, idle!             ... still processing msg #5 ...
```

Consumer 1 is **idle** while Consumer 2 is **overwhelmed** — poor load balancing.

### The Fix — `prefetch(1)`

```javascript
channel.prefetch(1); // Only give me one unACKed message at a time
```

> `prefetch(1)` means: "Don't send me a new message until I ACK the current one"

It is NOT "give me only 1 message total" — it resets after every ACK.

```
Consumer 1 (fast)          Consumer 2 (slow)
─────────────────          ─────────────────
← msg #1                   ← msg #2
✅ ACK #1                  ... processing ...
← msg #3                   ... processing ...
✅ ACK #3                  ... processing ...
← msg #5                   ✅ ACK #2
                            ← msg #4
```

Fast consumer picks up the slack. Slow consumer is never overwhelmed.

### Prefetch Values Compared

|`prefetch` value|Behavior|
|---|---|
|Not set|RabbitMQ floods the fastest consumer|
|`prefetch(1)`|Fair — slow workers get fewer messages|
|`prefetch(N)`|Batched — good for throughput tuning|

---

## 5. RabbitMQ vs Kafka

### Core Architectural Difference

**RabbitMQ — Push Based, Message Deleted After ACK**

```
Producer → [Queue] → Consumer (message gone after ACK ✅)
```

**Kafka — Pull Based, Message Persisted on Disk**

```
Producer → [Topic/Partition] → Consumer Group A (offset 5)
                             → Consumer Group B (offset 3)
                             → Consumer Group C (offset 7)
```

### Full Comparison

|Feature|RabbitMQ|Kafka|
|---|---|---|
|Message persistence|Deleted after ACK|Retained for configurable time|
|Replay messages|❌ Can't re-read consumed msgs|✅ Rewind offset and re-read|
|Multiple consumer groups|❌ One group shares queue|✅ Each group reads independently|
|Ordering guarantee|❌ Not guaranteed|✅ Per partition|
|Throughput|Medium|Very high|
|Competing consumers|✅ Same queue, round-robin|✅ Partitions per consumer|
|Routing/Exchanges|✅ Rich (direct, fanout, topic)|❌ Simple topic only|
|Complexity|Low — easy to set up|Higher — needs more ops|

### When to Choose RabbitMQ

- You need **complex routing** (fanout, direct, topic, headers)
- Messages are **tasks/jobs** — process once and done
- You want **simplicity** and fast setup
- **Use cases:** email sending, notifications, background jobs

### When to Choose Kafka

- You need **message replay** — re-process historical data
- **Multiple independent systems** need the same event
- You need **very high throughput** (millions of msgs/sec)
- **Use cases:** audit logs, event sourcing, analytics pipelines, microservices events

### Clarifying "Persistence"

> "I need persistence" can mean two very different things:

|Need|Winner|
|---|---|
|Survive broker restart (don't lose msgs)|Both handle it ✅|
|**Replay** old messages after consumption|Kafka only ✅|
|Multiple independent consumers reading same event|Kafka only ✅|
|Just process each message once reliably|RabbitMQ is simpler ✅|

> The biggest Kafka advantage isn't just persistence — it's **replay + multiple independent consumer groups reading the same data.**

---

## 6. Kafka Static Membership

### Background — The Rebalance Problem

In a normal Kafka consumer group, partitions are assigned dynamically:

```
Consumer Group "orders-group"
Topic: orders (6 partitions)

Consumer A → Partition 0, 1
Consumer B → Partition 2, 3
Consumer C → Partition 4, 5
```

A **rebalance** is triggered whenever:

- A consumer restarts (deploy, crash, rolling update)
- A consumer leaves the group (even temporarily)
- A consumer heartbeat times out (slow GC, long processing)

### What Happens During Rebalance

```
Consumer B restarts (rolling deploy)...

⚠️  REBALANCE TRIGGERED!
🛑  ALL consumers stop processing
🔀  Kafka reassigns ALL partitions
⏳  Downtime: seconds to minutes depending on group size
✅  Resume processing (with accumulated lag)
```

Every time **any** consumer restarts → **everyone stops** → partitions reshuffled. This is painful at scale.

---

### Static Membership — The Solution

Static membership assigns a **stable identity** to each consumer via `group.instance.id`.

```java
// Consumer config
props.put("group.instance.id", "consumer-instance-1"); // stable ID
```

Kafka now **remembers** who you are across restarts.

### How It Works — Step by Step

**Step 1 — First Join**

```
Consumer A (instance-id: "worker-1") → joins group
Consumer B (instance-id: "worker-2") → joins group
Consumer C (instance-id: "worker-3") → joins group

Kafka assigns:
  worker-1 → Partition 0, 1
  worker-2 → Partition 2, 3
  worker-3 → Partition 4, 5
```

**Step 2 — Consumer B Restarts**

```
❌ Without Static Membership:
   B leaves → REBALANCE → everyone stops → partitions reshuffled

✅ With Static Membership:
   B leaves → Kafka WAITS (session.timeout.ms)
   B rejoins as "worker-2" → gets same partitions back
   Nobody else was interrupted!
```

### Dynamic vs Static — Side by Side

```
DYNAMIC (normal)                    STATIC
────────────────                    ──────
Consumer restarts                   Consumer restarts
       ↓                                   ↓
Rebalance triggered                 Kafka waits (session timeout)
       ↓                                   ↓
ALL consumers pause                 Only restarted consumer pauses
       ↓                                   ↓
Partitions reshuffled               Consumer rejoins → same partitions
       ↓                                   ↓
Resume (lag built up)               Resume (minimal disruption)
```

### Key Configurations

```properties
# The magic config — must be unique per consumer instance
group.instance.id=worker-1

# How long Kafka waits before assuming consumer is truly dead
# Must be longer than your worst-case restart time
session.timeout.ms=30000

# How often consumer sends heartbeat to broker
heartbeat.interval.ms=3000
```

> ⚠️ If the consumer doesn't return within `session.timeout.ms` → Kafka **then** triggers a rebalance. Tune this value higher than your slowest restart.

### Real World — Kubernetes Rolling Deploy

```
Without static membership:
Deploy 10 pods → 10 rebalances → 10x processing pauses 😱

With static membership:
Deploy 10 pods → each pod restarts quietly → 0 full rebalances 🎉
```

Assign stable IDs using the pod name from Kubernetes:

```yaml
env:
  - name: KAFKA_GROUP_INSTANCE_ID
    valueFrom:
      fieldRef:
        fieldPath: metadata.name  # pod name as stable ID
```

### When to Use Static Membership

|Scenario|Use Static?|
|---|---|
|Frequent rolling deploys|✅ Yes|
|Kubernetes / containerized consumers|✅ Yes|
|Large consumer groups (10+ consumers)|✅ Yes — rebalance cost is high|
|Small group, rarely restarts|⚪ Optional|
|Consumers are truly stateless/ephemeral|❌ Not needed|

---

## Quick Reference — Everything at a Glance

|Topic|Key Takeaway|
|---|---|
|RabbitMQ competing consumers|Multiple consumers on **same queue** — RabbitMQ round-robins automatically|
|Fanout vs Competing|Fanout = copy to all; Competing = one message to one consumer|
|Horizontal scaling|Just add more consumers pointing at the same queue|
|`prefetch(1)`|One unACKed message at a time — enables fair dispatch between slow/fast workers|
|RabbitMQ vs Kafka persistence|Both survive restarts; only Kafka supports **replay**|
|Kafka static membership|Stable `group.instance.id` prevents rebalancing the whole group on rolling deploys|

## See also

- [[Messaging RabbitMQ and Kafka]] — earlier merged ops + comparison notes
- [[Backpressure]] · [[HOL]] · [[Realtime Updates]]
- [[System Design MOC]]
