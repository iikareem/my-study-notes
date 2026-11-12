# Distributed Locks: A Problem-by-Problem Deep Dive

> **How to read this document:** Each section starts with a **problem**, shows the **solution** people reach for, then shows **what breaks next**. That break becomes the next section's problem. By the end you will understand why each piece of a "correct" distributed lock exists.

---

## Table of Contents

1. [The Scenario](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#1-the-scenario)
2. [Two Reasons to Lock: Efficiency vs Correctness](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#2-two-reasons-to-lock-efficiency-vs-correctness)
3. [Problem 0: No Lock (Race Condition)](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#3-problem-0-no-lock-race-condition)
4. [Attempt 1: In-Process Mutex](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#4-attempt-1-in-process-mutex)
5. [Attempt 2: Database Locks](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#5-attempt-2-database-locks)
6. [Attempt 3: Naive Redis Lock (`SETNX`)](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#6-attempt-3-naive-redis-lock-setnx)
7. [Fix: Add an Expiry (TTL)](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#7-fix-add-an-expiry-ttl)
8. [Fix: Unique Token and Safe Release](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#8-fix-unique-token-and-safe-release)
9. [Problem: The Lock Expires While You Are Still Working](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#9-problem-the-lock-expires-while-you-are-still-working)
10. [Mitigation: Lease Renewal (Watchdog)](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#10-mitigation-lease-renewal-watchdog)
11. [The Core Problem: Pauses and Clock Drift](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#11-the-core-problem-pauses-and-clock-drift)
12. [The Real Fix: Fencing Tokens](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#12-the-real-fix-fencing-tokens)
13. [Problem: Redis Failover Loses the Lock](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#13-problem-redis-failover-loses-the-lock)
14. [Redlock and the Debate Around It](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#14-redlock-and-the-debate-around-it)
15. [Consensus-Based Locks: ZooKeeper and etcd](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#15-consensus-based-locks-zookeeper-and-etcd)
16. [Avoiding Locks Entirely](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#16-avoiding-locks-entirely)
17. [Decision Guide](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#17-decision-guide)
18. [Pitfalls Checklist](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#18-pitfalls-checklist)
19. [Exercises](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#19-exercises)
20. [Summary](https://claude.ai/chat/205df6af-2b45-4fc5-b607-9476b73947fd#20-summary)

---

## 1. The Scenario

You run a payments platform. Every night, a **payout worker** pays merchants what they are owed.

- You run **5 worker servers** for availability.
- Each worker pulls payout tasks from a queue.
- Sometimes the same merchant's payout is picked up by **two workers** (retries, duplicate messages, overlapping cron runs).

The payout logic for one merchant:

```
1. Read the merchant's balance          (e.g. $1,000)
2. Send $1,000 to the merchant's bank   (external API call)
3. Set the merchant's balance to $0
4. Mark the payout as PAID
```

**What we need:** for a given merchant, only one worker runs this logic at a time. A wrong result means real money is lost or paid twice.

This is the setting for the whole document.

---

## 2. Two Reasons to Lock: Efficiency vs Correctness

This distinction decides how much engineering you need. It comes from Martin Kleppmann's well-known analysis of distributed locking.

|Purpose|What it protects|What happens if the lock fails|Example|
|---|---|---|---|
|**Efficiency**|Avoids doing the same work twice|Wasted compute or a duplicate email. Annoying, not harmful|Two workers both build the same cache entry|
|**Correctness**|Protects data from concurrent corruption|Data loss, double payment, corrupted state|Two workers both pay the same merchant|

**Rule of thumb:**

- If the lock only saves effort, a simple Redis lock is fine.
- If the lock protects correctness, **a lock alone is never enough**. You also need a fencing mechanism (Section 12) or a design that avoids locks (Section 16).

Our payout scenario is a **correctness** problem.

---

## 3. Problem 0: No Lock (Race Condition)

**Problem:** Two workers process merchant M at the same time.

```
Time   Worker A                         Worker B
─────  ───────────────────────────────  ───────────────────────────────
t1     read balance = $1,000
t2                                      read balance = $1,000
t3     send $1,000 to bank
t4                                      send $1,000 to bank   <-- DOUBLE PAYMENT
t5     set balance = 0
t6                                      set balance = 0
```

The merchant is paid **$2,000** from a $1,000 balance. This is a classic **check-then-act race condition**: both workers checked the same state before either acted on it.

**Why it happens in distributed systems:** the workers are separate processes on separate machines. Nothing coordinates them.

**What we need:** a mechanism so that only one worker can be inside steps 1 to 4 for merchant M at a time. That mechanism is a **mutual exclusion lock**.

---

## 4. Attempt 1: In-Process Mutex

**Idea:** use the language's built-in lock (`threading.Lock`, `synchronized`, `sync.Mutex`).

```python
lock = threading.Lock()

def process_payout(merchant_id):
    with lock:
        balance = read_balance(merchant_id)
        send_to_bank(merchant_id, balance)
        set_balance(merchant_id, 0)
```

**Why it fails:** an in-process mutex only coordinates threads **inside one process**. Worker A and Worker B are different processes on different machines. Each has its own `lock` object, and neither sees the other.

**Problem this reveals:** we need a lock that lives **outside** any single worker, in a place all workers can reach. That is what makes it a _distributed_ lock.

---

## 5. Attempt 2: Database Locks

**Idea:** you already have a shared database. Let it arbitrate.

### 5.1 Row lock with `SELECT ... FOR UPDATE`

```sql
BEGIN;
SELECT balance FROM merchants WHERE id = 42 FOR UPDATE;  -- blocks other workers
-- ... do the work ...
UPDATE merchants SET balance = 0 WHERE id = 42;
COMMIT;                                                   -- releases the lock
```

Any other transaction trying to lock row 42 waits until this one commits or rolls back.

### 5.2 Advisory locks (PostgreSQL)

```sql
SELECT pg_advisory_lock(42);      -- session-level, held until unlock or disconnect
-- ... do the work ...
SELECT pg_advisory_unlock(42);
```

Advisory locks lock an arbitrary number you choose, not a table row. This is useful when the thing you are protecting is not a single row.

### 5.3 Strengths

- Simple. No new infrastructure.
- If the lock and the data are in the **same database**, the lock and the write are protected by the same transaction. This is very safe.
- The lock is released automatically if the connection dies.

### 5.4 Weaknesses (and the problems they create)

|Weakness|Why it matters|
|---|---|
|**Long transactions.** Our work includes a slow external bank call. Holding a DB transaction open during a network call ties up a connection and can cause lock pile-ups|The database becomes the bottleneck|
|**The protected resource is not the DB.** The bank API is external. A DB lock cannot stop two workers from both calling the bank|The lock protects the wrong thing|
|**Load.** Every lock operation hits your primary database|Lock traffic competes with real traffic|
|**Connection death while work continues.** If the DB connection drops, the lock is released, but the worker might still be mid-call to the bank|Two workers can now overlap|

**When DB locks are the right answer:** the whole critical section is inside one database. If so, **use them and stop reading**. They are the most correct option.

**Problem this reveals:** when the work touches external systems, or you need a lighter, faster lock, people reach for a dedicated lock service. Redis is the usual first choice.

---

## 6. Attempt 3: Naive Redis Lock (`SETNX`)

**Idea:** use Redis as a shared "whiteboard". A key means "locked".

`SETNX` = "SET if Not eXists". It succeeds for exactly one caller.

```python
def acquire(merchant_id):
    return redis.setnx(f"lock:payout:{merchant_id}", "locked")   # True if we got it

def release(merchant_id):
    redis.delete(f"lock:payout:{merchant_id}")

def process_payout(merchant_id):
    if not acquire(merchant_id):
        return  # someone else is doing it
    try:
        do_payout(merchant_id)
    finally:
        release(merchant_id)
```

Redis executes commands one at a time, so `SETNX` is atomic. Only one worker wins.

**Why this works in the happy path:** Worker A sets the key. Worker B's `SETNX` returns false. B backs off.

**Why it breaks:** the worker crashes between acquire and release.

```
Worker A: acquire lock  ✔
Worker A: *process crashes / machine dies / network cut*
Worker A: release()     ✘ never runs
```

The key `lock:payout:42` stays in Redis **forever**. No worker can ever process merchant 42 again. This is a **deadlock caused by a dead lock holder**.

**Problem this reveals:** a lock must **not** depend on its holder staying alive to release it.

---

## 7. Fix: Add an Expiry (TTL)

**Solution:** give the lock an expiry. If the holder dies, Redis deletes the key automatically.

```python
# Atomic: set only if not exists, with a 30,000 ms expiry
ok = redis.set(f"lock:payout:{merchant_id}", "locked", nx=True, px=30_000)
```

> **Important:** use a **single** command `SET key value NX PX ttl`. Do **not** do `SETNX` then `EXPIRE` as two commands. If the process crashes between them, you are back to a permanent lock. The two-step version is a classic bug.

A lock with an expiry is called a **lease**: "you own this for 30 seconds, then ownership lapses unless renewed."

**What this solves:** a crashed worker no longer blocks everyone forever.

**What it breaks:** now the lock can vanish **while its holder is still alive and working**. Two new problems follow (Sections 8 and 9).

---

## 8. Fix: Unique Token and Safe Release

**Problem:** with expiry, this timeline is possible:

```
Time   Worker A                              Redis                  Worker B
─────  ────────────────────────────────────  ─────────────────────  ──────────────────────
t0     SET lock NX PX 30000  → OK            lock = "locked"
t1     (working... slow bank call)
t30    ── lock expires ──                    lock = (gone)
t31                                          lock = "locked"   ←    SET lock NX → OK
t35    work finishes
t36    DEL lock   ← deletes B's lock!        lock = (gone)
t37                                                                  Worker C acquires
                                                                     (B and C both inside)
```

Worker A **deleted a lock that belongs to Worker B**, because the key name is the same for everyone. Now B and C can overlap.

**Solution:** each acquirer stores a **unique token** as the value, and only deletes the key **if the value still matches its own token**.

```python
token = uuid.uuid4().hex
ok = redis.set(key, token, nx=True, px=30_000)

# ... later, release:
RELEASE_SCRIPT = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""
redis.eval(RELEASE_SCRIPT, 1, key, token)
```

**Why a Lua script?** "Check the value, then delete" must be **atomic**. If you do `GET` then `DEL` as two commands, the lock can expire and be re-acquired by someone else between them. Redis runs a Lua script atomically.

**What this solves:** a worker can never release someone else's lock.

**What it does not solve:** Worker A still thinks it holds the lock after expiry and is still **doing the work**. That is the next problem.

---

## 9. Problem: The Lock Expires While You Are Still Working

The lock has a 30 s TTL, but the bank call takes 45 s on a bad day.

```
t0    A acquires lock (TTL 30s)
t30   lock expires; A is still waiting on the bank
t31   B acquires lock, starts the same payout
t45   A's bank call returns, A writes "PAID"
t50   B's bank call returns, B writes "PAID"
      → both paid the merchant
```

**Why this is hard:** you cannot choose a "safe" TTL.

|TTL choice|Consequence|
|---|---|
|Too short|Lock expires mid-work, so you get overlap|
|Too long|If the holder crashes, everyone waits a long time before recovery|

You cannot predict the worst-case duration of a network call.

**Problem to solve:** keep the lock alive while the worker is genuinely making progress.

---

## 10. Mitigation: Lease Renewal (Watchdog)

**Solution:** run a background thread that **extends the TTL** while the worker is alive and working. If the worker dies, renewals stop and the lock expires on its own.

```python
def renew_loop(key, token, ttl_ms, stop_event):
    while not stop_event.wait(ttl_ms / 3000):       # renew at 1/3 of the TTL
        RENEW_SCRIPT = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("pexpire", KEYS[1], ARGV[2])
        else
            return 0
        end
        """
        if redis.eval(RENEW_SCRIPT, 1, key, token, ttl_ms) == 0:
            # We lost the lock! Abort the work.
            abort_work()
            return
```

Notes:

- The renew script also checks the token so it never extends someone else's lock.
- If renewal returns 0, you have **lost the lock** and must stop immediately.
- Libraries like **Redisson** (Java) implement this as a "watchdog".

**What this solves:** slow-but-healthy workers keep their lock. Crashed workers lose it quickly (within one TTL).

**What it does not solve:** the worker can be **alive but unable to renew or notice**. That is the core problem, explained next.

---

## 11. The Core Problem: Pauses and Clock Drift

This section is the heart of the topic. Read it slowly.

### 11.1 Process pauses

A process can freeze for reasons it **cannot detect**:

- **Stop-the-world GC pause** (seconds, in bad cases)
- VM live migration or the hypervisor pausing the VM
- OS swapping memory to disk
- `SIGSTOP` or a debugger attached
- CPU starvation on an overloaded host
- Network stalls delaying packets

While paused, the worker's code does not run. It cannot renew the lease. It also does not **know** time has passed.

### 11.2 The failing timeline

```
Time   Worker A                              Redis                 Worker B          Storage
─────  ────────────────────────────────────  ────────────────────  ────────────────  ─────────
t0     acquire lock (TTL 30s) → OK           lock = A
t1     checks "do I hold the lock?" → yes
t2     ══ long GC pause begins ══
t30                                          lock expires
t31                                                                B acquires lock
t32                                                                B writes payout   ✔ written
t40    ══ GC pause ends ══
t40    A resumes. In its memory,
       "I hold the lock" is still true.
t41    A writes payout  ─────────────────────────────────────────────────────────►    ✔ written
                                                                                       (CORRUPTION)
```

The crucial insight:

> **A's check "do I hold the lock?" happened _before_ the pause. After the pause, that answer is stale, but A has no way to know.**

Adding another "am I still the owner?" check just before writing does not fix it. The pause can happen **between the check and the write**. Any check has a gap after it.

### 11.3 Clock drift and jumps

Leases depend on **time**, and clocks are unreliable:

- Server clocks drift at different rates.
- NTP can **step the clock** forward or backward.
- Redis key expiry uses the Redis server's clock; the worker's timeout logic uses its own.
- A clock jump on the lock server can expire a lock early.

So even without pauses, "I have the lock for 30 more seconds" is a statement you cannot fully trust.

### 11.4 The conclusion

> In an asynchronous system (unbounded delays, unbounded pauses, unreliable clocks), **no lock can guarantee that its holder is the only one acting at any instant.**

So what do we do? We stop trusting the **holder** to know whether it still owns the lock, and make the **resource** enforce the rule. That is fencing.

---

## 12. The Real Fix: Fencing Tokens

### 12.1 The idea

Every time the lock is granted, the lock service hands out a **monotonically increasing number**, the **fencing token**.

```
Grant #33 → Worker A
Grant #34 → Worker B   (after A's lease expired)
```

Every worker includes its token on every write to the protected resource. The **resource** remembers the highest token it has seen and **rejects anything lower**.

```
Time   Worker A (token 33)                   Worker B (token 34)      Storage (highest seen)
─────  ────────────────────────────────────  ───────────────────────  ──────────────────────
t0     acquires lock, gets token 33
t2     ══ GC pause ══
t31                                          acquires lock, token 34
t32                                          write(token=34)  ──►     accepts. highest = 34
t40    resumes (still thinks it's owner)
t41    write(token=33)  ──────────────────────────────────────────►   REJECTED (33 < 34) ✔
```

A's stale write is rejected. **Correctness is preserved even though A was wrong about owning the lock.**

### 12.2 Implementation with a database

Add a `fence_token` column to the protected row:

```sql
UPDATE payouts
SET    status = 'PAID',
       fence_token = :token
WHERE  merchant_id = :merchant_id
  AND  fence_token < :token;       -- only accept newer tokens

-- check rows affected; if 0, the write was stale → abort
```

In application code:

```python
rows = db.execute(UPDATE_SQL, token=my_token, merchant_id=mid).rowcount
if rows == 0:
    raise StaleLockError("A newer lock holder exists; aborting")
```

This is essentially **optimistic concurrency control** where the lock service supplies the version number.

### 12.3 The external API problem

Our payout calls a **bank API**. The bank does not know about our fencing tokens. How do we fence it?

Options, from best to worst:

1. **Idempotency keys.** Send a deterministic key such as `payout-{merchant_id}-{date}` with the bank call. The bank deduplicates, so a second call returns the original result. (Most payment APIs support this.)
2. **Record intent in your own DB first, fenced.** Write `payout_attempt(merchant, token, status=PENDING)` using the fenced update, and only the winner proceeds to call the bank.
3. **Reconciliation.** Periodically compare your records with the bank's and fix or alert on duplicates. This is a safety net, not a prevention mechanism.

### 12.4 Where do tokens come from?

|Source|Token|Notes|
|---|---|---|
|**etcd**|Key's `mod_revision` (or lease revision)|Monotonic across the cluster|
|**ZooKeeper**|`zxid`, or the sequence number of the lock node|Monotonic|
|**Database sequence**|`nextval('lock_seq')`|Fine if the DB is your lock source of truth|
|**Redis**|`INCR` on a counter|Weak: if Redis fails over and loses the latest increment, tokens can repeat. Be careful|

A fencing token **must** be strictly increasing across lock grants, even across lock-service failures. That is why consensus systems (Section 15) are preferred when correctness matters.

### 12.5 Requirements recap

- The lock service issues increasing tokens.
- The **resource** checks tokens and rejects stale ones.
- If the resource cannot check tokens, fall back to idempotency or redesign (Section 16).

> **Key takeaway:** The lock tells you _who should be working_. The fencing token makes the resource _enforce it_. Without fencing, a lock is only an efficiency optimization.

---

## 13. Problem: Redis Failover Loses the Lock

So far we assumed Redis is always up. In production you run Redis with a **replica** for high availability.

### 13.1 The failing timeline

Redis replication is **asynchronous**. The primary acknowledges writes before the replica has them.

```
Time   Worker A              Redis Primary       Redis Replica      Worker B
─────  ────────────────────  ──────────────────  ─────────────────  ───────────────────
t1     SET lock=A  → OK      lock = A            (not yet copied)
t2                           ── primary crashes ──
t3                                               promoted to primary
                                                 lock = (missing!)
t4                                                                  SET lock NX → OK
       A and B both hold "the lock".
```

The lock was acknowledged to A but never replicated before the crash. After failover, B acquires it too.

**Problem to solve:** a lock service that survives a node failure without forgetting granted locks.

Two families of solutions exist:

1. **Redlock:** use several independent Redis instances and require a majority (Section 14).
2. **Consensus systems:** use software built for this, such as etcd or ZooKeeper (Section 15).

---

## 14. Redlock and the Debate Around It

### 14.1 The algorithm (by the Redis creator)

Use **N = 5 independent Redis masters** (not replicas of each other). To acquire:

1. Record the current time `t_start`.
2. Try `SET key token NX PX ttl` on all 5 instances, one after another, with a small per-instance timeout.
3. The lock is acquired if **at least 3 of 5** (a majority) said OK **and** the elapsed time is less than the TTL.
4. Effective validity = `TTL − elapsed − clock_drift_allowance`.
5. If acquisition fails, **release on all 5** (including ones that did not respond).

```
        ┌──────────┐
 A ───► │ Redis 1  │ OK
        ├──────────┤
 A ───► │ Redis 2  │ OK
        ├──────────┤
 A ───► │ Redis 3  │ OK       → 3/5 majority, within TTL → lock acquired
        ├──────────┤
 A ───► │ Redis 4  │ (down)
        ├──────────┤
 A ───► │ Redis 5  │ (slow)
        └──────────┘
```

**What it solves:** the single-Redis-failure problem from Section 13. One failed instance does not lose the lock because a majority must agree.

### 14.2 The criticism

Martin Kleppmann (author of _Designing Data-Intensive Applications_) argued that Redlock is **not safe for correctness**, because:

1. **It relies on timing assumptions.** It assumes bounded network delay, bounded pauses, and bounded clock drift. Section 11 showed real systems violate these.
2. **It has no fencing token.** Redlock gives no monotonically increasing number, so it cannot protect the resource against stale holders.
3. **Clock jumps on a Redis node** can make it expire a key early and hand the lock to someone else.

The Redis creator (Salvatore Sanfilippo) responded that the timing assumptions are reasonable in practice and that clock jumps can be managed operationally.

### 14.3 The practical reading of the debate

|Your goal|Is Redlock acceptable?|
|---|---|
|**Efficiency** (avoid duplicate work, occasional overlap tolerable)|Yes, but a single Redis with a TTL may already be enough, so Redlock's extra complexity is often unnecessary|
|**Correctness** (overlap causes harm)|Not on its own. Use a consensus system **and** fencing tokens, or design out the lock|

Redlock sits in an awkward middle: heavier than a single-Redis lock, but without the guarantees of a consensus system.

---

## 15. Consensus-Based Locks: ZooKeeper and etcd

**Problem:** we want a lock service that does not lose locks on failure and can hand out ordered tokens.

**Solution:** build the lock on a **consensus protocol** (Raft in etcd, ZAB in ZooKeeper). A write is acknowledged only after a **majority of nodes** have durably stored it, so a minority failure cannot lose it. Leader failover is safe by design.

### 15.1 etcd

Building blocks:

- **Lease:** a TTL'd object. Keys can be attached to it. The client sends **keep-alives**. If they stop, the lease expires and attached keys are deleted.
- **Revision:** a global, strictly increasing counter incremented on every change. A key's `mod_revision` works as a **fencing token**.
- **Transactions (compare-and-swap):** "create this key only if it doesn't exist."

Lock acquisition (simplified):

```
1. Grant a lease (TTL 10s) and start keep-alive.
2. Txn: if key "/locks/payout/42" does not exist → put it with the lease.
   Otherwise → wait for it to be deleted (watch).
3. Your fencing token = the key's create/mod revision.
4. On release (or lease expiry), the key is deleted and the next waiter proceeds.
```

etcd's client libraries provide this as a ready-made **mutex** (`concurrency.NewMutex`).

### 15.2 ZooKeeper

Classic recipe using **ephemeral sequential znodes**:

```
/locks/payout-42/
    lock-0000000001   ← Worker A (lowest number = holds the lock)
    lock-0000000002   ← Worker B (watches 0000000001)
    lock-0000000003   ← Worker C (watches 0000000002)
```

- **Ephemeral:** the node is deleted automatically if the client's session ends (crash, partition). This solves "dead holder."
- **Sequential:** ZooKeeper appends an increasing number. Lowest number holds the lock, and that number is a natural **fencing token**.
- **Watch the predecessor only:** each waiter watches the node just before it, not the lowest one. When a lock is released only **one** waiter wakes up, avoiding the **thundering herd** problem.

### 15.3 Is a consensus lock enough by itself?

**No.** Consensus fixes the lock _service_. It does not fix a paused _client_.

- The lease can still expire while the client is frozen.
- The client can still wake up believing it holds the lock.

So even with etcd or ZooKeeper, you still use **fencing tokens at the resource** for correctness-critical work. What consensus gives you is: no lost locks, ordered tokens you can trust, and safe failover.

### 15.4 Costs

|Cost|Detail|
|---|---|
|Operational complexity|You run and monitor a 3 or 5 node cluster|
|Latency|Each lock operation needs a majority round trip, so it is slower than a single Redis|
|Throughput|Not designed for very high lock-operation rates|
|Availability|Lost majority = no locks (by design, the safe failure mode)|

---

## 16. Avoiding Locks Entirely

Often the best distributed lock is **no lock**. Each alternative below solves the payout problem differently.

### 16.1 Idempotency keys

**Problem solved:** duplicate execution is harmless if repeating the operation yields the same result.

```
payout_id = "payout-merchant42-2026-10-31"
→ unique constraint in the DB, and the same key is passed to the bank
→ second attempt hits the constraint or returns the original result
```

```sql
INSERT INTO payouts (payout_id, merchant_id, amount, status)
VALUES ('payout-m42-2026-10-31', 42, 1000, 'PENDING')
ON CONFLICT (payout_id) DO NOTHING;   -- second inserter gets 0 rows → stops
```

**Why it is powerful:** no coordination service, and it works across crashes and retries. This is the standard approach in payment systems.

### 16.2 Optimistic concurrency (compare-and-set / versioning)

**Problem solved:** detect conflicting writes instead of preventing them.

```sql
UPDATE merchants
SET    balance = 0, version = version + 1
WHERE  id = 42 AND version = :version_i_read;   -- fails if someone changed it
```

If 0 rows are updated, someone else won. Re-read and retry (or give up).

### 16.3 Single-writer partitioning

**Problem solved:** if only one consumer is ever assigned a given merchant, no lock is needed.

- Hash `merchant_id` to a **queue partition** (Kafka partition, SQS FIFO message group).
- Each partition has **one consumer at a time**.
- All payouts for merchant 42 flow through the same partition, in order.

Caveat: during consumer rebalances, two consumers can briefly overlap. So combine this with idempotency or fencing.

### 16.4 State machines with atomic transitions

**Problem solved:** make invalid concurrent transitions impossible.

```sql
-- Only one worker can move PENDING → IN_PROGRESS
UPDATE payouts SET status = 'IN_PROGRESS', worker = :me
WHERE  id = :id AND status = 'PENDING';
-- rows affected = 1 → you own it; 0 → someone else does
```

This is a lock implemented as a **database compare-and-set**, and it is a very common production pattern. Pair it with a heartbeat timestamp so stuck jobs can be reclaimed safely.

### 16.5 Queue claiming: `SELECT ... FOR UPDATE SKIP LOCKED`

**Problem solved:** many workers pulling from a table-based queue without blocking each other.

```sql
SELECT id FROM payout_tasks
WHERE status = 'PENDING'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;     -- skip rows another worker has locked
```

---

## 17. Decision Guide

### 17.1 Choosing an approach

```
Is the whole critical section inside ONE database?
├── Yes → Use DB transactions / row locks / advisory locks. Done.
└── No
    Is overlap harmless (efficiency only)?
    ├── Yes → Single Redis: SET NX PX + unique token + safe release (+ watchdog).
    └── No (correctness matters)
        Can you make the operation idempotent or use CAS/versioning?
        ├── Yes → Do that. Add a lock only as an optimization.
        └── No
            Use a consensus system (etcd / ZooKeeper)
            + fencing tokens enforced by the resource
            + reconciliation as a safety net.
```

### 17.2 Comparison table

|Approach|Survives holder crash|Survives lock-server failure|Survives client pause|Complexity|Use for|
|---|:-:|:-:|:-:|:-:|---|
|In-process mutex|n/a|n/a|n/a|Low|Single process only|
|DB row/advisory lock|✔|✔ (DB HA)|✔ if write is in same txn|Low|Work fully inside one DB|
|Redis `SET NX PX`|✔ (TTL)|✘ (async failover)|✘|Low|Efficiency locks|
|Redis + watchdog|✔|✘|✘|Medium|Long-running efficiency locks|
|Redlock|✔|✔ (mostly)|✘|Medium-High|Rarely the best choice|
|etcd / ZooKeeper|✔|✔|✘ (needs fencing)|High|Leader election, coordination|
|**Any lock + fencing token**|✔|depends|**✔**|Medium|**Correctness**|
|Idempotency / CAS (no lock)|✔|✔|✔|Low-Medium|Most business operations|

---

## 18. Pitfalls Checklist

Use this when reviewing a design or code.

- [ ] Acquire uses **one atomic command** (`SET NX PX`), never `SETNX` then `EXPIRE`.
- [ ] The lock value is a **unique token**, not a constant.
- [ ] Release is **compare-and-delete in a Lua script** (or equivalent), not a blind `DEL`.
- [ ] The TTL is based on **measured work duration with headroom**, not a guess.
- [ ] Long work uses **lease renewal**, and the worker **aborts** if renewal fails.
- [ ] The protected resource **checks a fencing token** (if correctness matters).
- [ ] External side effects use **idempotency keys**.
- [ ] You never assume "I checked I hold the lock" stays true after the check.
- [ ] You do not rely on **wall-clock time** comparisons between machines.
- [ ] Lock-service failure has a defined behavior: **fail closed** (refuse work) vs **fail open** (proceed). Choose deliberately.
- [ ] Lock acquisition has a **timeout** and the caller handles failure.
- [ ] You have **metrics and alerts** for lock wait time, lease expirations, and rejected stale writes.
- [ ] A **reconciliation job** exists for money-like operations.

---

## 19. Exercises

1. **Break it yourself.** Write the naive Redis lock from Section 6. Simulate a crash by killing the process. Observe the stuck lock. Then add the TTL.
2. **Reproduce the stale release.** Using a short TTL and a `sleep`, show Worker A deleting Worker B's lock. Fix it with the Lua script.
3. **Simulate a GC pause.** Add `time.sleep(40)` after acquiring a 30 s lock. Show two workers writing. Then add a fencing column and show the stale write rejected.
4. **Implement the fenced update** (Section 12.2) in SQLite or Postgres and write a test with two threads using tokens 33 and 34 in reversed order.
5. **Failover thought experiment.** Draw the timeline for Section 13 with synchronous replication (`WAIT 1`). Does it remove the problem? (Hint: think about what `WAIT` actually guarantees on primary crash.)
6. **Design question.** Redesign the payout system with **no distributed lock** using idempotency keys, a state machine, and a unique constraint. What failure modes remain?
7. **Try etcd.** Run a local etcd, use its mutex, and read the `mod_revision` as a fencing token. Pause the client with `kill -STOP` and watch the lease expire.

---

## 20. Summary

|#|Problem|Solution|New problem it creates|
|---|---|---|---|
|0|Concurrent workers corrupt data|Need mutual exclusion|Where does the lock live?|
|1|In-process mutex is invisible to other machines|Shared external lock|Which store?|
|2|DB locks are heavy and cannot cover external calls|Lightweight lock in Redis|Crashed holder|
|3|Crashed holder leaves a permanent lock|**TTL** (lease) via `SET NX PX`|Lock can expire mid-work; can delete others' locks|
|4|Deleting someone else's lock|**Unique token + Lua compare-and-delete**|Lock expiring during slow work|
|5|Lock expires during slow work|**Lease renewal (watchdog)**|Holder paused; cannot renew or notice|
|6|Pauses and clock drift make "I hold the lock" unreliable|**Fencing tokens** enforced by the resource|Resource must support it; token source must be reliable|
|7|Redis failover loses locks|**Redlock** or **consensus (etcd/ZooKeeper)**|Complexity, cost, still needs fencing|
|8|Locks are fragile in general|**Avoid locks**: idempotency, CAS, single-writer, state machines|Requires redesigning the operation|

### The five sentences to remember

1. A lock with an expiry is a **lease**, and leases can expire while the holder is still running.
2. A process **cannot know** whether it was paused, so "I hold the lock" is never a safe belief on its own.
3. **The resource, not the lock holder, must enforce exclusion**, using fencing tokens.
4. If you can use **idempotency or compare-and-set**, you often do not need a distributed lock at all.
5. Ask first: **is this lock for efficiency or for correctness?** The answer decides everything else.

---

## Further Reading

- Martin Kleppmann, _How to do distributed locking_ (blog), and _Designing Data-Intensive Applications_, Chapter 8 (The Trouble with Distributed Systems) and Chapter 9 (Consistency and Consensus)
- Salvatore Sanfilippo, _Is Redlock safe?_ (blog reply) and the Redis documentation on distributed locks
- Mike Burrows, _The Chubby Lock Service for Loosely-Coupled Distributed Systems_ (Google's paper that introduced the lock-service idea and discusses sequencers, the fencing-token concept)
- ZooKeeper Recipes and Solutions: Locks
- etcd documentation: leases, the concurrency package, and revisions