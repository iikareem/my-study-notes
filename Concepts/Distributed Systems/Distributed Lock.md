---
tags:
  - distributed-lock
  - distributed-systems
  - locking
  - redis
---

# Distributed Synchronization: Locks, Redlock & Fencing Tokens

When multiple servers compete for the same resource, you need a way to say **"only one of you goes at a time."** This guide walks through how that works, what breaks, and how the industry actually solves it.

---

## The Core Problem

Imagine two servers both pick up the same payment job from a queue. Without coordination, both charge the customer. This is why distributed locking exists — it's a **traffic light for your servers.**

---

## 1. Basic Distributed Lock

The simplest approach: use Redis as a shared "who has permission right now" store.

```
SET lock_key my_unique_id NX PX 10000
```

- `NX` — only set if the key doesn't exist (prevents overwriting someone else's lock)
- `PX 10000` — auto-expire after 10 seconds (so a crashed server doesn't block everyone forever)

**To release**, use a Lua script (more on Lua below) to safely delete only your own lock:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### The Fatal Flaw

TTL expiry creates a dangerous gap:

```
Server A acquires lock (TTL = 10s)
Server A freezes — GC pause, network hiccup (15s)
Lock expires automatically
Server B acquires the same lock
Server A wakes up, still thinks it owns the lock
=> Two servers acting as the owner at the same time
```

Also, **one Redis node = one point of failure.** If it crashes, the whole locking mechanism is gone.

---

## 2. Redlock — Surviving Redis Node Failures

Redlock solves the single-point-of-failure problem by requiring agreement from **multiple independent Redis nodes** (typically 5), not just one.

### The Core Idea

You don't own the lock unless a **majority (3 out of 5)** of Redis nodes agree you have it. If one node crashes, the others still hold the decision.

### How It Works

```
Record start time: T1

For each of the 5 Redis nodes:
    Try: SET resource_name my_id NX PX 10000
    (use a short timeout per node — skip slow/dead nodes and move on)

If you got the lock on 3+ nodes AND the total time taken < lock TTL:
    => You own the lock. Effective remaining time = TTL - time_elapsed
Else:
    => Release on ALL nodes. Wait a random delay, then retry.
```

Random delay on retry is important — without it, two competing servers keep colliding at the exact same moment.

### Why Majority Quorum Helps

Even if 2 out of 5 nodes crash, the remaining 3 can still grant exactly one winner. No single node failure can break the system.

### Still Not Perfect — The Zombie Problem

Redlock still relies on **wall-clock time (TTL)** to decide when a lock is valid. This is its weakness:

```
Server A gets Redlock across 3 nodes, valid for 10s
Server A freezes for 15s (GC pause, slow VM...)
Lock expires on all 5 nodes
Server B acquires Redlock
Server A wakes up — has no idea time passed — still writes to the resource
=> Both A and B acted as owner
```

Having 5 nodes instead of 1 does **not** fix this. The problem is time, not node count.

> **Martin Kleppmann's argument:** Redlock is fine for best-effort tasks where occasional overlap is tolerable. It is not safe enough when correctness is critical (e.g., financial writes, file system changes).

> **Antirez's rebuttal:** In practice, pauses long enough to break Redlock are rare and the algorithm is good enough for most real-world use cases — it was never meant to replace ZooKeeper.

### When to Use Redlock

|Situation|Use|
|---|---|
|Avoid duplicate cron jobs|✅ Redlock is fine|
|Prevent double-charging a customer|❌ Use ZooKeeper/etcd + fencing tokens|

---

## 3. Fencing Tokens — The Real Safety Net

Fencing tokens solve what no lock can solve alone: **a paused process that woke up late and still thinks it owns the lock.**

The key insight: **stop trusting the lock to be the final word. Let the resource itself decide.**

### How It Works

Every time a lock is granted, the lock service issues a **monotonically increasing number** (the fencing token). The resource being protected tracks the highest token it has seen and rejects anything older.

```
Server A gets lock → token = 100
Server A freezes...
Server B gets lock → token = 101
Server B writes to DB with token 101 → accepted, DB stores "last token = 101"
Server A wakes up, writes with token 100 → REJECTED (100 < 101)
```

The stale server's write is dead on arrival — no matter how confident Server A is that it holds the lock.

### Why This Works

The database (or whatever resource you're protecting) **becomes the real authority**, not the lock service. A monotonically increasing integer cannot lie, even when clocks drift, networks lag, or processes pause.

### Real-World Examples

These are not theoretical — they are production systems used at massive scale:

|System|Token Name|Who Enforces It|What It Prevents|
|---|---|---|---|
|**HDFS** (Hadoop)|Generation stamp|DataNodes|Stale NameNode corrupting file blocks|
|**Apache Kafka**|Producer epoch|Broker|Zombie producer writing duplicate messages|
|**Kubernetes**|`resourceVersion`|API Server (etcd)|Two controllers overwriting each other's changes|

All three systems independently arrived at the same pattern — that should tell you something.

### The One Caveat

Fencing tokens only work if **the resource being protected can check the token.** If you're calling a third-party API or a storage system you don't control, you can't enforce fencing there. The lock layer remains your only defense in those cases.

---

## 4. Lua Scripts in Redis — Why They Matter Here

When you do distributed locking, you often need to run multiple Redis commands together (e.g., "check if I own the lock, then delete it"). If you send these as separate commands, another server can slip in between them.

**Lua scripts solve this** — Redis runs the entire script as a single, uninterruptible operation.

### The Three Benefits

**1. Atomicity** — No other command can run while the script is executing. It's all-or-nothing, no gaps.

**2. One network round-trip** — Instead of sending 3 commands (3 trips to Redis), you send 1 script (1 trip). Faster and safer.

**3. Logic lives on the server** — Less state management in your application code.

### Example: Safe Lock Release

```lua
-- Only delete the lock if WE are the owner
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0  -- Someone else owns it, do nothing
end
```

Without the Lua script, the `GET` and `DEL` are two separate commands — another server could acquire the lock between them, and you'd accidentally delete their lock.

> ⚠️ **Keep Lua scripts short.** Redis is single-threaded — a slow or infinite Lua script freezes your entire Redis instance for every client.

---

## 5. Advanced: Fencing Token + `SELECT FOR UPDATE`

For most cases, a fencing token is enough. But sometimes your workflow looks like this:

```
1. Read a record from the DB
2. Run complex business logic for 2 seconds
3. Write the result back
```

The gap between step 1 and step 3 is dangerous. Another process could read and modify the same row while you're thinking. Use both tools together:

- **`SELECT FOR UPDATE`** — locks the DB row so no one else can read or modify it while you're working
- **Fencing token** — ensures that even if your lock somehow expired during those 2 seconds, the final write is rejected if someone else has already written

Think of it as: `SELECT FOR UPDATE` guards your work session, fencing token guards the final write.

---

## Decision Guide

```
Do you need coordination across multiple servers?
│
├── Yes, but failures are tolerable (duplicate email, duplicate job)
│   └── Simple Redis lock or Redlock ✅
│
├── Yes, and correctness is critical (payments, data integrity)
│   └── ZooKeeper or etcd (consensus-backed) + Fencing Tokens ✅
│
└── Yes, with complex read-modify-write logic
    └── ZooKeeper/etcd + Fencing Token + SELECT FOR UPDATE ✅
```

---

## Summary Cheat Sheet

|Strategy|Speed|Safety|Best For|
|---|---|---|---|
|Simple TTL Lock (single Redis)|Fast|Low|Non-critical background tasks|
|Redlock (multi-node Redis)|Fast|Medium|Best-effort deduplication|
|Fencing Token Only|Fastest|High|Standard DB updates, API state changes|
|ZooKeeper/etcd + Fencing|Slower|Highest|Financial writes, file systems, critical data|
|Fencing + `SELECT FOR UPDATE`|Medium|Highest|Long read-modify-write cycles|

---

## Golden Rules

1. **The database is the final authority** — let it reject stale writes via token checks, not your application logic.
2. **Every lock needs a TTL** — not for correctness, but so a crashed server doesn't block everyone forever.
3. **Fencing tokens are not automatic** — the resource you're protecting must be built to check and enforce them.
4. **Match the tool to the risk** — Redlock for "nice to avoid duplicates," ZooKeeper + fencing for "must never corrupt data."
5. **Handle rejections gracefully** — a rejected write (`0 rows updated`) is not a crash, it's a signal. Catch it and retry with fresh state.

# Fencing Token & The SPOF Problem

Yes — you caught a real gap. If your fencing token is just a Redis `INCR` on a single node, you've traded one SPOF (the lock) for another SPOF (the token generator). The whole safety guarantee collapses if that node goes down.

---

## Why It Matters

The fencing token only works if it is **always increasing and never resets.** The moment your token generator crashes and comes back with a reset counter, or a replica takes over with stale state, you can issue token `45` after token `99` — and now old writes get accepted again.

```
Token generator is at 99
Node crashes, replica takes over at last synced value: 60
New token issued = 61
Server with old token 99 writes → ACCEPTED (99 > 61)
Server with new token 61 writes → ACCEPTED
=> Two stale writers both get through
```

The whole point of fencing breaks.

---

## How to Fix It: 3 Options

### Option 1 — Use a Consensus-Backed Store (The Right Way)

Replace your single Redis node with **ZooKeeper or etcd** — both use consensus protocols (ZAB and Raft respectively) to keep multiple nodes in sync before confirming any write.

When you ask ZooKeeper for the next token, it doesn't respond until a **majority of its nodes agree** on that value. So even if one node crashes, the token counter is safe on the surviving majority.

```
You ask ZooKeeper: "give me the next token"
ZooKeeper checks with 3 out of 5 nodes: "all agree counter = 100?"
Yes → returns 100, increments to 101 safely
One node crashes → other 4 still hold the correct value
```

This is exactly how **HDFS and Kubernetes** generate their fencing tokens — they're backed by ZooKeeper and etcd, not a single Redis node. That's what makes them genuinely safe.

**Tradeoff:** Slightly slower than Redis because of the consensus round-trip. But for correctness-critical systems, this is the right call.

---

### Option 2 — Use a Database Sequence (Simple & Practical)

Most relational databases (PostgreSQL, MySQL) have built-in **auto-increment sequences** that are:

- Durable (written to disk)
- Atomic (no two callers get the same number)
- Recoverable (survives restarts)

```sql
CREATE SEQUENCE fencing_token_seq;
SELECT nextval('fencing_token_seq');  -- always returns the next unique number
```

If your database is already replicated and highly available (which it usually is in production), your token generator inherits that availability for free — no extra infrastructure needed.

**Best for:** Teams already running a reliable PostgreSQL/MySQL cluster who don't want to add ZooKeeper just for tokens.

---

### Option 3 — Redis With Persistence + Sentinel (Good Enough for Medium Risk)

If you want to stay in the Redis world, you can reduce (not eliminate) the SPOF risk by:

- Enabling **AOF persistence** so Redis writes every increment to disk before confirming it — survives restarts
- Running **Redis Sentinel** — monitors the primary node and promotes a replica if it crashes, using the persisted data

```
Primary Redis crashes (counter was at 88, saved to disk)
Sentinel promotes replica
Replica loads from disk → counter resumes at 88
Next token = 89 ✅ (no reset, no gap)
```

**The honest caveat:** There's still a tiny window between the primary crashing and the replica catching up where tokens could be re-issued. This is acceptable for medium-risk systems but not for hard correctness guarantees.

---

## Decision: Which Option to Pick

```
How critical is it if two servers get the same token?
│
├── Catastrophic (payments, file systems, data integrity)
│   └── ZooKeeper or etcd ✅
│
├── Serious but survivable (duplicate job processing, API calls)
│   └── DB sequence (PostgreSQL/MySQL) ✅
│
└── Low risk (best-effort deduplication, logging)
    └── Redis + AOF + Sentinel is fine ✅
```

---

## One-Line Summary

> A fencing token generator on a single node is itself a SPOF — the fix is to back it with either a consensus system (ZooKeeper/etcd) or a replicated database sequence, so the counter survives any single node failure without resetting.

The pattern is the same everywhere: **whatever you trust as the source of truth must itself be fault-tolerant.** Fencing tokens solve the zombie process problem, but they're only as reliable as the system generating them.

why not us
