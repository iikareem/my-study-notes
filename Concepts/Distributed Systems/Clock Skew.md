---
tags:
  - distributed-systems
---

## **Clock Skew and Why You Cannot Trust Time in Distributed Systems**

This one breaks people's brains the first time they really sit with it. Time feels like the one thing that's objective and universal. It isn't — not in distributed systems. And the consequences of trusting it are subtle, dangerous, and notoriously hard to debug.

---

## Start from zero

On a single machine, time is simple. You call `Date.now()` or `System.currentTimeMillis()` and get a number. Every call on that machine uses the same clock. Order is preserved — if event A happened before event B, A's timestamp is smaller.

The moment you have **two machines**, this breaks.

Each machine has its own physical clock — a hardware oscillator that ticks. These clocks are not perfectly synchronized. They drift. Machine A might think it's 12:00:00.000 while machine B thinks it's 12:00:00.847. Eight hundred milliseconds apart.

That gap is **clock skew.** And it has profound consequences.

---

## Why clocks drift

Physical clocks are not perfect. The oscillator in your server runs at slightly different speeds depending on temperature, load, and manufacturing variance. Over time, clocks on different machines diverge.

The fix is **NTP — Network Time Protocol.** Servers periodically ask a time server what the correct time is and adjust their clocks accordingly.

But NTP adjustment is not instantaneous and not perfect:

- The sync happens over a network, which itself has variable latency
- NTP can only estimate what the time was when the response was sent
- Clocks can be adjusted **forward or backward** by NTP — meaning time on a single machine can go backwards

That last point is critical. Your server's clock can jump backward. Code that assumes `Date.now()` is always increasing — like anything that uses timestamps to determine event order — can break in ways that are nearly impossible to reproduce.

---

## What breaks when you trust timestamps

### Ordering events across services

Say you have two services logging events to a central store. You want to reconstruct what happened in what order.

```
Service A: event X at 12:00:01.500
Service B: event Y at 12:00:01.200
```

Based on timestamps, Y happened before X. But what if Service B's clock is 400ms ahead? Then X actually happened first — and you just built a false timeline.

This matters enormously for debugging, auditing, and anything that reconstructs causality from logs.

### Distributed locking with lease expiry

A common pattern: a service acquires a lock with a TTL — say 10 seconds. It does work. The lock expires. Another service can now acquire it.

```
Service A acquires lock. TTL = 10 seconds.
Service A's clock: 12:00:00
Lock server's clock: 12:00:09.800  ← 9.8 seconds ahead

Service A thinks it has 10 seconds.
Lock server thinks the lock expired 9.8 seconds ago.
Service B acquires the lock.
Now both A and B hold the lock simultaneously.
```

You just violated mutual exclusion — the entire point of the lock — because two clocks disagreed on what time it was.

This is not theoretical. It's a known failure mode of systems like Redlock (Redis-based distributed locking) that was publicly debated between Martin Kleppmann and the Redis creator. The core argument: any locking algorithm that relies on timing assumptions can fail under clock skew.

### "Last write wins" conflict resolution

Many databases and caches resolve conflicts by keeping the write with the latest timestamp. Seems reasonable.

But if the machine that wrote the "earlier" data has a clock that's ahead, its timestamp will be larger — and it wins even though it wrote first in real time. You just silently discarded the more recent write.

Cassandra uses this strategy by default. At small scale it's fine. At scale, with clock drift, you get silent data loss that is extremely hard to detect.

---

## The solution space

### Logical clocks — Lamport timestamps

Leslie Lamport's insight in 1978: **you don't need to know the actual time. You just need to know order.**

A Lamport clock is a simple counter:

- Each node maintains a counter starting at 0
- Every time a node does something, it increments its counter
- Every time a node sends a message, it includes its current counter
- Every time a node receives a message, it sets its counter to `max(own counter, received counter) + 1`

```
Node A: counter=1  sends message --> Node B receives, sets counter=max(1,0)+1=2
Node A: counter=2  sends message --> Node B receives, sets counter=max(2,2)+1=3
```

Now you have a **happened-before** relationship. If A's counter is lower than B's, A happened before B — without trusting any physical clock.

The limitation: Lamport clocks can tell you A happened before B, but if two events have no causal relationship, you can't determine their order. They're partially ordered, not totally ordered.

A **causal relationship** means one event could have influenced or caused another event.

In distributed systems, event **A causally happened before B** if:

- A directly caused B, or
- Information from A reached B through messages/events.

Example:

1. Service A updates a user balance.
2. Service A sends a message to Service B.
3. Service B sends an email notification.

Here:

- Update balance → caused → send message
- Send message → caused → send email

So these events have a **causal relationship**.

But if:

- Server X logs “CPU = 80%”
- Server Y processes an order at the same time

and they never communicated or affected each other, then there is **no causal relationship**.

### Vector clocks

An extension of Lamport clocks. Instead of one counter, each node maintains a **vector** — one counter per node in the system.

```
Node A: [A=1, B=0, C=0]
Node B: [A=0, B=1, C=0]
```

When nodes communicate, they merge vectors by taking the max of each position. This lets you detect **concurrent events**— events where neither happened before the other.

DynamoDB and Riak use vector clocks for conflict detection. When two writes are concurrent (neither vector dominates the other), the system can flag the conflict and let the application resolve it — rather than silently picking one based on an untrustworthy timestamp.

### Google's TrueTime — the hardware solution

Google built Spanner, a globally distributed database, and needed real timestamps for external consistency. Their solution: **TrueTime** — GPS receivers and atomic clocks in every datacenter, combined with software that gives you not a single timestamp but an **interval**: `[earliest, latest]`.

The API returns `TT.now()` which gives you `[tbefore, tafter]` — the true time is somewhere in that interval. Before committing a transaction, Spanner **waits out the uncertainty interval** — it literally sleeps until it's certain no other transaction could have a later timestamp.

This gives Spanner true external consistency with real time — at the cost of commit latency measured in the uncertainty window (typically 1-7ms).

Almost nobody has GPS clocks in their datacenters. But the insight is profound: **treat time as an interval with uncertainty, not a precise point.**

---

## The practical takeaway

You probably aren't building Spanner. But you are almost certainly doing at least one of these:

- Using timestamps to order events across services
- Using TTLs for locks or cache expiry across machines
- Using "last write wins" somewhere in your data layer
- Logging events from multiple services and assuming timestamp order is causal order

Every one of these has a latent clock skew bug in it.

The engineering response isn't to panic — it's to be deliberate:

- Use **logical clocks or sequence numbers** when you need ordering, not wall clock time
- Use **fencing tokens** instead of TTL-based locks for mutual exclusion
- Treat timestamps as **approximate** and never the sole source of truth for ordering
- When you do need real time across machines, understand your NTP sync interval and build in tolerance for that uncertainty

---

The deepest lesson here is the same one that runs through all of distributed systems:

**Things that feel like global, objective facts on a single machine — time, order, state — become local, approximate, and uncertain the moment you have two machines.**

Time isn't a background constant your system runs against. It's just another piece of state that different nodes disagree on.

HLC TO BE STUDIDED

[[Fencing Token]]