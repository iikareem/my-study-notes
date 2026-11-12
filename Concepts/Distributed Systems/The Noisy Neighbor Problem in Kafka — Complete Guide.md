# The Noisy Neighbor Problem in Kafka — Complete Guide

---

## 🧱 1. Core Concepts First

Before anything, let's define the building blocks clearly.

### Topic

A topic is just a **named log** where messages are stored. Think of it as a channel or a category.

```
Topic: "events"
→ all customer events go here
→ producers write to it
→ consumers read from it
```

### Partition

A topic is split into **partitions** — smaller ordered logs inside the topic. Each partition holds messages for a specific group (in our case, a tenant).

```
Topic: "events"
├── Partition 0  → Tenant A's messages
├── Partition 1  → Tenant B's messages
└── Partition 2  → Tenant C's messages
```

### Partition Key

The partition key is just a value you attach to each message (e.g. `tenant_id`). Kafka hashes this key to decide **which partition the message lands in**.

```
Message with key = "tenant_A"  → Kafka hashes it → always goes to Partition 0
Message with key = "tenant_B"  → Kafka hashes it → always goes to Partition 1
Message with key = "tenant_C"  → Kafka hashes it → always goes to Partition 2
```

The key guarantees that **all messages from the same tenant always land in the same partition.**

### Consumer Group

A consumer group is a group of workers that **subscribe to a topic together**. Kafka distributes the partitions evenly across the workers in the group.

```
Consumer Group: "my-group" subscribes to topic "events"

Kafka assigns:
Partition 0  ←──── Worker 1   (one-to-one, locked)
Partition 1  ←──── Worker 2   (one-to-one, locked)
Partition 2  ←──── Worker 3   (one-to-one, locked)
```

> **Important:** Consumer group is per **topic**, not per partition or key. Partitions are distributed evenly and consistently — not randomly. Reassignment only happens when a worker joins or leaves the group.

### Worker

A worker is just a **regular service or program** — written in any language (TypeScript, Go, Python, etc.) — that runs in a loop:

```typescript
while (true) {
  const task = await getNextTask()   // ask for work
  await doTheWork(task)              // process it
  await markAsDone(task.id)          // report done
  await sleep(100)                   // repeat
}
```

Nothing magical. Just a loop that keeps asking for work and doing it.

---

## 💥 2. The Noisy Neighbor Problem

### What happens

One customer backfills **500,000 events** into your Kafka topic all at once.

Because their `tenant_id` key always routes to the same partition:

```
Topic: "events"
├── Partition 0 (Tenant A) → 500,000 messages 🔥 HOT
├── Partition 1 (Tenant B) → 10 messages
└── Partition 2 (Tenant C) → 5 messages
```

### The one-to-one lock is the root cause

```
Partition 0 (500K) ←──── Worker 1   😵 drowning for 20+ minutes
Partition 1 (10)   ←──── Worker 2   😴 done in seconds, then idle
Partition 2 (5)    ←──── Worker 3   😴 done in seconds, then idle
```

Worker 2 and Worker 3 are completely free — but they **cannot help** Worker 1.

Kafka's rule is strict: **one partition belongs to one consumer at a time.** Worker 2 cannot jump in and steal work from Partition 0, even when it is idle.

### The result

```
20 minutes later...
→ Tenant B's data is stale
→ Tenant C's data is stale
→ Both are asking "why is nothing working?"
→ Your inbox is full of support tickets 😅
```

### Why Kafka cannot fix this alone

Kafka is designed for **speed and ordering**, not fairness. Its partition-to-consumer assignment is static and deliberate. It has no concept of "this partition is overloaded, let me rebalance the work."

```
Kafka's job:  deliver messages fast, in order, reliably
Kafka's NOT:  a fair task scheduler
```

---

## ✅ 3. The Solution — Postgres as a Smart Queue

### The key insight

The fix is not to change Kafka. The fix is to **add a smart layer between Kafka and your workers.**

This layer is not magic — it is just a smart human-designed system that sits in front of the workers and decides **who gets to work next, and how much.**

```
Before (Kafka alone):
Tenant A (500K) ──► Partition 0 ──► Worker 1  😵
Tenant B        ──► Partition 1 ──► Worker 2  😴
Tenant C        ──► Partition 2 ──► Worker 3  😴

After (Postgres queue in front):
Tenant A (500K) ──┐
Tenant B        ──┼──► Kafka ──► Ingestion Service ──► Postgres Queue
Tenant C        ──┘                                          │
                                                    ┌────────▼────────┐
                                                    │ Smart Fair Rules │
                                                    └────────┬────────┘
                                                    Worker 1 ◄┤
                                                    Worker 2 ◄┤ all workers
                                                    Worker 3 ◄┘ serve all tenants
                                                               fairly
```

### Each piece has a clear job

|Component|Job|
|---|---|
|Kafka|Absorbs massive event spikes at high speed|
|Ingestion Service|Reads from Kafka, writes tasks into Postgres|
|Postgres Queue|The smart fair scheduler — decides who works next|
|Workers|Ask Postgres for the next task, process it, report done|

### Why Postgres instead of just Kafka?

```
Kafka:    workers are PUSHED whatever is next in the log
          → no control, blind, follows log order

Postgres: workers PULL the next fair task by asking
          → full control, intelligent, follows fairness rules
```

That pull moment — the SQL query — is where **all the intelligence lives.**

---

## 🔧 4. The Fair Queue — How It Actually Works

### The naive (unfair) SQL query

Every Postgres queue starts like this:

```sql
SELECT *
FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

This is **unfair** — the `LIMIT 10` is global. It grabs the 10 oldest tasks regardless of tenant. With 500K of Tenant A's tasks at the front, A gets all 10 slots every time.

### The fair SQL query — built in 3 steps

**Step 1 — Find distinct tenants that have pending work:**

```sql
SELECT DISTINCT tenant_id
FROM tasks
WHERE status = 'pending'
```

Cost scales with **number of tenants** (e.g. 100), never with the 500K backlog. ✅

**Step 2 — Skip tenants already running a task:**

```sql
AND NOT EXISTS (
  SELECT 1 FROM tasks r
  WHERE r.tenant_id = tasks.tenant_id
  AND r.status = 'running'
)
```

> Why `NOT EXISTS` if we already filtered `pending`? Because they check **different things:**
> 
> - Outer `WHERE pending` → "does this tenant have work to do?"
> - `NOT EXISTS running` → "is this tenant already being worked on right now?"
> 
> Without this, Tenant A could pile up multiple running tasks and dominate again.

**Step 3 — Grab exactly 1 task per remaining tenant:**

```sql
SELECT DISTINCT ON (tenant_id) *
FROM tasks
WHERE status = 'pending'
ORDER BY tenant_id, created_at
FOR UPDATE SKIP LOCKED;
```

One index probe per tenant — never a full sort of the backlog. ✅

### The two limits that enforce fairness

```
Limit 1 — Per-tenant cap:   max 1 running task per tenant at a time
Limit 2 — Global worker cap: max N tasks total per worker at a time
```

**Per-tenant cap** stops the noisy neighbor. **Global cap** stops the thundering herd (1000 tenants all ready at once = too many tasks).

### Random probe order

When worker slots < active tenants (e.g. 10 slots, 50 tenants), you cannot serve everyone in one round.

If you always probe tenants in the same order (alphabetical, oldest first), the same tenants always win — which is just a subtler noisy neighbor problem.

**Fix:** probe tenants in **random order** each round:

```sql
ORDER BY RANDOM(), created_at
```

Now every tenant has an equal chance every round.

---

## ⚖️ 5. Two Valid Architectures — Choose Based on Your Goal

### Pattern 1: Fair Shared Queue (Postgres in front)

```
Goal: Maximum utilization + fairness

Kafka → Postgres Queue → Worker 1 ┐
                       → Worker 2 ├ any worker serves any tenant, fairly
                       → Worker 3 ┘
```

✅ No idle workers ✅ No noisy neighbor ✅ Cost efficient — fewer workers needed ❌ Tenants share resources — not fully isolated

**Best for:** Multi-tenant SaaS, cost-sensitive setups, most customers

---

### Pattern 2: Dedicated Consumers Per Tenant (Scale horizontally)

```
Goal: Full isolation per tenant

Tenant A → Partition 0 → [Worker 1][Worker 2][Worker 3]  ← scale as needed
Tenant B → Partition 1 → [Worker 4]
Tenant C → Partition 2 → [Worker 5]
```

✅ Full isolation — A's burst never affects B or C ✅ Scale each tenant independently ❌ Workers idle when tenant is quiet ❌ Expensive — dedicated resources per tenant

**Best for:** Enterprise clients, strict SLA guarantees, premium paid tiers

---

### Real world: most companies do both

```
Small / medium tenants  → shared fair queue (Pattern 1)
Large enterprise tenants → dedicated consumers (Pattern 2)
```

---

## 📐 6. The Complete Picture

```
                    Producers
          ┌─────────────────────────┐
          │  Tenant A (500K events) │
          │  Tenant B (10 events)   │
          │  Tenant C (5 events)    │
          └────────────┬────────────┘
                       │ partition key routes messages
                       ▼
                  Kafka Topic
          ┌─────────────────────────┐
          │  Partition 0 → Tenant A │
          │  Partition 1 → Tenant B │
          │  Partition 2 → Tenant C │
          └────────────┬────────────┘
                       │ ingestion service reads & writes
                       ▼
               Postgres Task Queue
          ┌─────────────────────────┐
          │ tenant_id │ status      │
          │ A         │ pending     │
          │ A         │ pending     │
          │ B         │ pending     │
          │ C         │ pending     │
          └────────────┬────────────┘
                       │ smart fair SQL query
                       │ - max 1 per tenant
                       │ - global cap per worker
                       │ - random probe order
                       ▼
                    Workers
          ┌─────────────────────────┐
          │ Worker 1: [A][B][C]     │
          │ Worker 2: [A][B][C]     │  ← all workers busy
          │ Worker 3: [A][B][C]     │  ← all tenants served
          └─────────────────────────┘
```

---

## 📚 7. Lessons Learned

|Problem|Root Cause|Fix|
|---|---|---|
|Noisy neighbor in Kafka|One partition → one worker, locked|Add smart queue layer in front|
|Workers idle|Heavy tenant monopolizes one worker|Workers pull from shared fair queue|
|Unfair SQL queue|Global `LIMIT` grabs from queue head|Move limit inside tenant partitions|
|Slow backlog scan|Sorting 500K rows per query|Index probe per tenant, never full scan|
|Tenant piling up slots|No check for already-running tasks|`NOT EXISTS` on running tasks|
|Thundering herd|Per-tenant fairness without global cap|Add global worker slot limit|
|Same tenants always win|Sequential tenant probing|Random probe order each round|

---

## 💡 8. The Philosophy — Not Magic, Just Smart Design

> Postgres is not magic. It is a smart human-designed system placed **in front of the workers** that decides who works next, how much, and in what order.

The workers themselves stay simple — they just ask:

```
"What should I work on next?"
```

And Postgres answers with fairness rules baked in.

The result:

```
✅ Every worker is always busy         → maximum utilization
✅ No tenant monopolizes the workers   → fairness
✅ Every tenant gets a turn            → predictability
✅ Everything happens in an exact,
   controlled, predictable way         → reliability
```

That is the whole idea. Simple concept, careful implementation.