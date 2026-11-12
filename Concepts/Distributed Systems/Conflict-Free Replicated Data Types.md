# Conflict-Free Replicated Data Types (CRDTs) — Complete Notes

## What is a CRDT?

CRDTs are data structures that can be **replicated across multiple nodes** and updated independently, then **merged automatically without conflicts** — even if updates happen simultaneously on different nodes.

> First time hearing this? Think of it as:  
> **"Multiple people editing the same list simultaneously — and it never breaks."**

---

## Why Does This Problem Exist?

### The Split-Brain Problem

```
Data Center A (New York)    Data Center B (London)
      Redis Node 1      ←→       Redis Node 2
```

Two users update the same counter at the exact same moment:

```
User A (NY) → counter += 1   (value: 100 → 101)
User B (LN) → counter += 1   (value: 100 → 101)

After sync... expected: 102
             actual:    101  ← DATA LOSS!
```

This is the **split-brain problem** — nodes diverge and last-write-wins causes data loss.

---

## CAP Theorem Connection

```
CAP Theorem — Pick 2 of 3:

        Consistency
             △
             │
             │
             │
Availability ◄──────► Partition Tolerance

CRDTs choose → Availability + Partition Tolerance (AP)
```

|Property|What CRDTs Guarantee|
|---|---|
|**Availability**|Every node always accepts reads & writes — never blocks|
|**Fault Tolerance**|If a node goes down — system keeps running|
|**Eventual Consistency**|All nodes **will** converge to same state — just not instantly|

### The Trade-off

```
Traditional Redis (CP):              Redis CRDT (AP):

User writes → must confirm           User writes → accepted immediately
across nodes before success          syncs later in background

      ✅ Strong consistency                ✅ Always available
      ❌ Blocks during partition           ❌ Reads may be slightly stale
```

---

## CRDT Types & Use Cases

|CRDT Type|Redis Data Type|Use Case|
|---|---|---|
|G-Counter|`INCR` / `INCRBY`|Page views, analytics|
|PN-Counter|`INCR` + `DECR`|Likes / unlikes|
|OR-Set|`SADD` / `SREM`|Shopping cart, tags|
|LWW-Register|`SET` with timestamp|Sensor data, user profile|

---

## Real World Use Cases

### 1. Shopping Cart

**Problem:** User opens cart on mobile and desktop simultaneously while offline. Both add different items. One overwrites the other on reconnect.

**Solution:** `SADD` (OR-Set CRDT internally)

```python
redis.sadd("cart:user:123", "shoes", "hat")      # mobile
redis.sadd("cart:user:123", "shoes", "jacket")   # desktop

redis.smembers("cart:user:123")
# → {shoes, hat, jacket} ✅
```

---

### 2. Distributed Counters

**Problem:** Multiple servers increment page views simultaneously. Nodes disagree on final count.

**Solution:** G-Counter CRDT

```
Node 1: views = 500
Node 2: views = 480
Node 3: views = 510

Merge = MAX per node + SUM → Correct global total ✅
```

---

### 3. Social Media Likes

**Problem:** Like/unlike happens rapidly across distributed servers. Inconsistent counts.

**Solution:** PN-Counter

```
P-Counter (likes):   [Node1: +5, Node2: +3]  → Total: 8
N-Counter (unlikes): [Node1: -2, Node2: -1]  → Total: 3

Net likes = 8 - 3 = 5 ✅
```

---

### 4. IoT Sensor Data

**Problem:** Multiple sensors send updates simultaneously. Which reading wins?

**Solution:** LWW-Register (Last Write Wins)

```
Sensor A @ 10:00:01 → temp = 72°F
Sensor B @ 10:00:02 → temp = 73°F

Winner = highest timestamp → 73°F ✅
```

---

## Do You Implement CRDTs Yourself?

```
Are you using Redis Enterprise / Redis Cloud?
        │
        ├── YES → Active-Active enabled?
        │              │
        │              ├── YES → CRDTs are AUTOMATIC ✅
        │              │         Just use normal commands
        │              │
        │              └── NO  → Enable Active-Active
        │
        └── NO (open-source Redis)
                       │
                       └── Need multi-master / geo-dist?
                                  │
                                  ├── YES → Build CRDT logic yourself ⚠️
                                  │
                                  └── NO  → Standard Redis is fine ✅
```

|Who You Are|What You Do With CRDTs|
|---|---|
|**App Developer (Redis Enterprise)**|Nothing — just pick the right data type|
|**DevOps / Architect**|Enable Active-Active, choose conflict resolution strategy|
|**Open-source Redis user**|Implement merge logic yourself if needed|

---

## Deep Dive — OR-Set CRDT (Shopping Cart)

### The Core Idea

> OR-Set doesn't just store the **value.**  
> It stores the value **+ a unique UUID** for every add operation.  
> That's how it knows what to keep and what to remove.

### Step by Step

#### Setup — Two Nodes in Sync

```
Node 1 (New York)          Node 2 (London)
cart = {}                  cart = {}
```

#### User Goes Offline — Adds Items on Both Devices

```
Mobile → Node 1                       Desktop → Node 2

SADD cart "shoes"                     SADD cart "shoes"
SADD cart "hat"                       SADD cart "jacket"
```

#### Internally OR-Set Stores:

```
Node 1 (NY)                           Node 2 (LN)
{                                     {
  "shoes" : [uid_101],                  "shoes" : [uid_205],
  "hat"   : [uid_102]                   "jacket": [uid_206]
}                                     }
```

> Same item "shoes" — but each node gave it a **different UUID.**  
> They are treated as **separate independent observations.**

#### Nodes Reconnect — Merge via UNION of all tags

```
Node 1 sends → { shoes: uid_101, hat: uid_102 }
Node 2 sends → { shoes: uid_205, jacket: uid_206 }

Merged Result on BOTH nodes:
{
  "shoes"  : [uid_101, uid_205],
  "hat"    : [uid_102],
  "jacket" : [uid_206]
}

Visible cart = { shoes, hat, jacket } ✅
```

---

## The 3 Core Merge Rules

### Rule 1 — ADD always wins

```
ADD → union all tags across all nodes
      never rejected, never overwritten
```

### Rule 2 — REMOVE is per node (per observation)

```
Each node can ONLY remove its own UUID
Cannot remove a UUID it never created

Node 1 added shoes → uid_101  → Node 1 can only delete uid_101
Node 2 added shoes → uid_205  → Node 2 can only delete uid_205
```

### Rule 3 — CONFLICT (add vs remove same item) → ADD wins

```
Node 1 removes shoes (deletes uid_101)
Node 2 still has shoes (uid_205 alive)

After merge → shoes survives ✅
Because Node 2's observation is independent
```

---

## UUID is The Truth, Not The Value

```
❌ Wrong way to think:
"shoes was removed, so shoes is gone"

✅ Right way to think:
"uid_101 was removed, uid_205 still exists
 shoes is alive as long as any uid points to it"
```

### Same Item Added Again = New UUID

```
Node 1 adds shoes    → uid_101   ← exists
Node 1 removes shoes → uid_101   ← gone
Node 1 adds shoes    → uid_387   ← brand new existence
```

### Item Only Truly Dies When ALL UUIDs Are Gone

```
shoes is alive as long as ONE tag is alive:

uid_101 ── Node 1 removed ❌
uid_205 ── Node 2 still has ✅
shoes SURVIVES

─────────────────────────

uid_101 ── Node 1 removed ❌
uid_205 ── Node 2 removed ❌
shoes is GONE ✅
```

---

## Full Timeline Visual

```
t=0    Node1 {shoes:101, hat:102}       Node2 {shoes:205, jacket:206}
                   │                               │
t=1    removes shoes on Node1           adds boots on Node2
                   │                               │
       Node1 {hat:102}                  Node2 {shoes:205, jacket:206, boots:310}
                   │                               │
t=2    ←—————————— nodes reconnect & sync ——————————→
                   │
t=3    BOTH nodes:
       {
         shoes  : [205],      ← uid_101 removed, uid_205 survived
         hat    : [102],
         jacket : [206],
         boots  : [310]
       }

       Final cart = { shoes, hat, jacket, boots } ✅
```

---

## When to Use CRDTs vs Not

|Scenario|Use CRDTs / AP|Avoid CRDTs|
|---|---|---|
|Shopping cart|✅||
|Like / view counters|✅||
|Collaborative editing|✅||
|IoT sensor data|✅||
|Leaderboards|✅||
|Bank transactions||❌ needs strong consistency|
|Stock inventory||❌ can't oversell|
|Payment processing||❌ must be exact|
|Authentication tokens||❌ security critical|

---

## System Design — Where CRDTs Fit

```
                    ┌─────────────────┐
                    │   Load Balancer │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐           ┌────────▼────────┐
     │   App Server 1  │           │   App Server 2  │
     └────────┬────────┘           └────────┬────────┘
              │                             │
     ┌────────▼────────┐           ┌────────▼────────┐
     │  Redis Node     │◄─────────►│  Redis Node     │
     │  (New York)     │  CRDT     │  (London)       │
     │                 │  Sync     │                 │
     └─────────────────┘           └─────────────────┘

Both nodes accept writes simultaneously
CRDTs merge automatically in background
Eventual consistency guaranteed ✅
```

---

## Key Takeaways

> 1. **CRDTs = mathematically guaranteed eventual consistency**
> 2. **You don't write CRDT code** — Redis handles it internally
> 3. **UUID per operation** is the secret — not the value itself
> 4. **ADD always beats REMOVE** when there's a conflict
> 5. **Item only dies** when ALL nodes remove ALL their UUIDs
> 6. **Best for AP systems** — availability over strict consistency
> 7. **Wrong tool** for financial, inventory, or security-critical systems