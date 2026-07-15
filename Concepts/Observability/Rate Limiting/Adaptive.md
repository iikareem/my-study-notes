---
tags:
  - adaptive
  - observability
  - rate-limiting
---

# A Full Guide to Adaptive Rate Limiting — From First Principles

---

## Why Fixed Rate Limiting Is Not Enough

Before diving into algorithms, you need to understand why the classic algorithms you already learned — token bucket, fixed window, sliding window — all share the same fundamental flaw.

They are all built on a hardcoded number. "Allow 100 requests per minute." That number never changes regardless of what is actually happening inside your server.

This creates two opposite failure modes:

**The too strict problem:** Your server is healthy, CPU at 10%, memory mostly free. The 101st request arrives and gets rejected with a 429 error. You just turned away a valid user even though your hardware had plenty of room to serve them. You are wasting capacity you paid for.

**The too lenient problem:** Your limit is 100 requests per minute and 100 requests arrive — but these 100 happen to be massive database export jobs instead of simple reads. CPU spikes to 100% and the server crashes. The rate limiter did nothing because it only counts requests, not the cost of those requests.

The core issue is that a fixed number is always a guess. It goes stale the moment your traffic patterns change, your data grows, or your queries get more complex.

---

## What Adaptive Rate Limiting Actually Is

Adaptive rate limiting replaces the hardcoded number with a feedback loop. Instead of a bouncer with a fixed clicker, think of it as a thermostat. It watches the actual health of your system in real time and continuously adjusts the limit up or down based on what it sees.

The signals it can watch include:

- CPU utilization
- Memory usage
- Response latency
- Event loop delay (Node specific)
- Database query times
- Active connection count
- Request queue depth

Instead of saying "allow 100 requests per second," an adaptive system says "allow as much traffic as possible provided CPU stays under 80%."

When the system is healthy it raises the ceiling, letting more traffic through. When stress appears it lowers the ceiling, shedding load until the system stabilizes. No human intervention needed.

**What this solves that fixed limiting cannot:**

- It never rejects users when the server has capacity to serve them.
- It detects when requests are expensive and reacts to actual resource cost, not just request count.
- It prevents cascading failures — if a database slows down, it detects the latency rise and throttles upstream traffic before the entire system collapses.

---

## Part 1 — Prerequisites You Must Understand First

The two main adaptive algorithms — AIMD and Vegas — were not invented for web servers. They were invented to solve network congestion inside TCP. To truly understand their behavior you need to understand the problem they were originally solving.

---

### Prerequisite 1 — How TCP Sends Data (Buffers and Sliding Window)

TCP is a reliable protocol. Every byte sent is guaranteed to arrive in order with no losses. To achieve this, both sides maintain a buffer — a dedicated chunk of RAM.

- The **sender buffer** holds data that has been sent but not yet acknowledged, in case it needs to be retransmitted.
- The **receiver buffer** holds data that has arrived from the network but has not yet been read by the application.

Instead of sending one packet, waiting for an acknowledgement, then sending the next — which would be extremely slow — TCP uses a **sliding window**. This allows the sender to have multiple packets in flight simultaneously without waiting for individual acknowledgements.

The window is simply the range of packets the sender is allowed to transmit before it must pause and wait.

[![Sliding Window protocols Summary With Questions - GeeksforGeeks](https://media.geeksforgeeks.org/wp-content/uploads/Sliding-Window-Protocol.jpg)

---

### Prerequisite 2 — Flow Control and the Receiver Window (rwnd)

Flow control solves a specific problem: what if the sender is much faster than the receiver?

Imagine a powerful server sending a large file to a slow mobile phone. The server can push at 1 Gbps but the phone can only process 10 Mbps. If the server sends at full speed, the phone's buffer overflows and packets are lost.

TCP solves this with the **receiver window (rwnd)**. The receiver continuously tells the sender how much free space remains in its buffer. This value travels inside every acknowledgement packet. The sender is forbidden from having more unacknowledged bytes in flight than the size of rwnd.

```
Bytes in flight ≤ rwnd
```

If the phone's buffer fills up it advertises rwnd = 0 and the sender immediately stops.

**The key point:** flow control only looks at the two endpoints — sender and receiver. It knows nothing about what is in between.

---

### Prerequisite 3 — The Problem Flow Control Could Not Solve (Congestion)

In the mid 1980s the internet suffered from what became known as congestion collapse. Network throughput would suddenly drop to near zero and the cause was that flow control was blind to the network path itself.

Here is the scenario:

```
Sender (1 Gbps) ──────────────► Router (10 Mbps bottleneck) ──► Receiver (fast, empty buffer)
```

The receiver has an empty buffer and advertises a large rwnd. It tells the sender "send me everything." The sender blasts at 1 Gbps. But the router in the middle can only handle 10 Mbps. Its internal queue overflows, packets get dropped, the sender retransmits those packets, which makes the congestion worse, and the network collapses.

Flow control had no answer for this because it never looked at the routers in between.

---

### Prerequisite 4 — The Congestion Window (cwnd)

To solve this, Van Jacobson introduced congestion control in 1988. He added a second window called the **congestion window (cwnd)**.

While rwnd represents the limits of the receiver's memory, cwnd represents the limits of the network path itself.

The sender must now respect both. The actual amount of data allowed in flight at any moment is the minimum of the two:

```
Transmission window = min(cwnd, rwnd)
```

Whichever is smaller wins. If the network is the bottleneck, cwnd is the constraint. If the receiver is the bottleneck, rwnd is the constraint.

The entire job of congestion control algorithms like AIMD and Vegas is to answer one question: **how should cwnd grow and shrink over time to use the network efficiently without overwhelming it?**

---

### How This Maps to Your Node Package

Before moving to the algorithms, here is the translation table from network concepts to your application:

| Network Layer (TCP)             | Application Layer (Your Package)             |
| ------------------------------- | -------------------------------------------- |
| Congestion window (cwnd)        | Concurrency limit (requests in flight)       |
| Packet loss                     | CPU spike, event loop delay, memory pressure |
| RTT increase                    | Response latency climbing above baseline     |
| BaseRTT (minimum ever observed) | Minimum observed response latency            |
| CurrentRTT (right now)          | Current rolling average response latency     |
| Router buffer filling up        | Server event loop backing up                 |
| Network congestion collapse     | Server crash or cascading failure            |

Everything that follows maps directly onto this table.

---

## Part 2 — AIMD (Additive Increase Multiplicative Decrease)

### The core idea

AIMD is reactive. It does not try to predict congestion. It waits for a clear signal that congestion has already happened, then responds aggressively.

The signal it listens for at the network level is **packet loss**. The assumption is simple: if a packet was dropped, some router's buffer filled up and overflowed. Loss equals congestion.

### How it behaves

**When things are healthy — additive increase:**

Every round trip where no loss occurs, cwnd grows by 1. Slow, steady, linear growth. The sender is carefully probing for how much bandwidth is available without slamming into the ceiling.

```
cwnd: 10 → 11 → 12 → 13 → 14 → 15...
```

**When loss is detected — multiplicative decrease:**

The moment packet loss is detected, cwnd is cut in half immediately. No gradual reduction. Instant halving.

```
cwnd: 20 → loss detected → 10
```

Additive increase then starts again from this new lower point.

### Why the asymmetry exists

Increase is slow and linear because you are sharing the network with other connections. You want to find the ceiling gently without slamming into it and ruining it for everyone else.

Decrease is fast and aggressive because when congestion happens every millisecond you stay at the high rate makes it worse. You need to back off immediately.

### The sawtooth pattern

If you draw cwnd over time you get a sawtooth wave — slowly climbing, suddenly dropping, slowly climbing again, repeating indefinitely.

```
cwnd
 |        /\        /\
 |       /  \      /  \
 |      /    \    /    \
 |     /      \  /      \
 |    /        \/        \
 |___/
 |________________________ time
```

This pattern is normal and expected. AIMD is always probing upward and backing off when it hits the ceiling.

### The fundamental weakness

AIMD only reacts after congestion has already happened. Packets had to be dropped — users had to experience failure — before the algorithm knew there was a problem. By the time it reacts, the damage is done.

It is like driving with your eyes closed and only knowing you hit a wall after the crash.

### What AIMD looks like in your Node package

The translation is direct:

- **Healthy signal** → no event loop delays, CPU and memory within thresholds → slowly increase the concurrency limit by 1.
- **Stress signal** → event loop delay spikes, CPU crosses threshold → immediately cut the concurrency limit in half.

```
Concurrency limit: 20 → 21 → 22 → 23 (healthy)
Event loop delay spikes → cut to 11
11 → 12 → 13 → 14 (recovering)
```

The weakness carries over too. You only detect the problem after the event loop is already delayed. A user already got a slow response before you reacted.

---

## Part 3 — Vegas (Proactive Congestion Detection)

### The core insight

Vegas was built on one observation that AIMD completely ignored:

**RTT starts climbing before packets are ever lost.**

When a router's buffer starts filling up, packets have to wait in that buffer before being forwarded. That waiting time shows up as increased RTT. The buffer is not full yet — no packets are being dropped yet — but the delay is already visible. This is an early warning signal available for free, and Vegas listens to it.

### The two numbers Vegas tracks

**BaseRTT** — the minimum RTT ever observed for this connection. This represents the network at its cleanest with no buffering delay. Your healthy baseline.

**CurrentRTT** — the actual RTT being observed right now as a rolling measurement.

When CurrentRTT climbs above BaseRTT, Vegas interprets that gap as evidence that packets are waiting in a router buffer. The bigger the gap, the more congestion is building.

### How Vegas makes its decision

Vegas calculates expected throughput versus actual throughput:

```
expected = cwnd / BaseRTT    ← what you should get if the network is clean
actual   = cwnd / CurrentRTT ← what you are actually getting right now
diff     = expected - actual
```

- diff is small → network is clean → gently increase cwnd.
- diff is large → congestion is building → gently decrease cwnd.

Both changes are small and continuous. No sudden halving. No sawtooth.

### The Netflix formula

Netflix's implementation of Vegas for server concurrency uses a cleaner version of the same idea:

```
new limit = (BaseRTT / CurrentRTT) × current limit
```

If your server usually responds in 10ms (BaseRTT) but is currently responding in 40ms (CurrentRTT):

```
new limit = (10 / 40) × current limit = 0.25 × current limit
```

The limit drops to 25% of its current value. The algorithm detected queue buildup and responded before any request actually failed.

### How it looks over time

Smooth and stable. No sawtooth. Vegas hovers just below the congestion point, making small continuous adjustments.

```
cwnd
 |          ___________
 |         /           \___________
 |        /
 |       /
 |______/
 |________________________ time
```

### The fundamental weakness

Vegas works beautifully when it is the only connection on a network. But if it shares the network with AIMD connections it loses badly.

AIMD is aggressive — it keeps pushing its window up until packets drop. Vegas backs off early when it detects rising RTT. So AIMD connections grab more and more bandwidth, Vegas connections politely step aside, and Vegas ends up starved. It is too well-behaved for a competitive shared environment.

This is why Vegas never replaced AIMD at the TCP level despite being technically smarter. The internet is full of aggressive senders.

**But this weakness disappears in your Node package.** Your server is not competing with other connections for bandwidth. You are the only one making decisions about your own concurrency limit. There is no aggressive AIMD opponent to lose to. Vegas's weakness simply does not apply in this context, which is exactly why it is the better fit.

### What Vegas looks like in your Node package

- **BaseRTT** → minimum response latency you have ever observed (your healthy baseline).
- **CurrentRTT** → rolling average of recent response latencies.
- **cwnd** → your concurrency limit.

When response times start climbing above your baseline, Vegas gently reduces how many requests you allow in flight simultaneously — before anything actually fails, before any user gets a timeout, before your event loop backs up.

```javascript
const gradient = baseLatency / currentLatency
newLimit = Math.ceil(gradient * currentLimit)
```

If baseline is 10ms and current is 15ms, gradient is 0.66 and the limit drops to 66% of its current value. Small, early, smooth correction.

---

## Part 4 — AIMD vs Vegas Side by Side

||AIMD|Vegas|
|---|---|---|
|Signal used|Packet loss / stress event|RTT / latency increase|
|When it reacts|After failure occurs|Before failure occurs|
|Increase style|Slow linear (+1)|Slow continuous|
|Decrease style|Sudden halving (×0.5)|Gentle proportional reduction|
|Behavior pattern|Sawtooth|Smooth and stable|
|Fundamental weakness|Reactive — damage already done|Loses to aggressive senders|
|Weakness in Node?|Still applies|Does not apply|
|Best signal in Node|Event loop delay|Response latency|

---

## Part 5 — Which One Should You Build

Vegas is the better fit for Node. The reasons stack up cleanly:

**The signal is free.** Latency is already available inside every request/response cycle. You do not need background timers, process sampling, or native addons. Just measure `Date.now()` at the start and end of each request.

**The signal is honest.** When Node gets overwhelmed, the first thing that happens is latency climbs — requests queue up waiting on I/O, the event loop backs up, responses slow down. Latency is the earliest and most truthful indicator of Node stress.

**The weakness does not exist here.** Vegas loses to aggressive AIMD senders on a shared network. Your server is not competing with anyone. You are the only one controlling your own concurrency limit.

**The behavior is better for users.** AIMD lets failures happen before reacting. Vegas prevents the failure from happening in the first place. Every request that gets through sees fast response times because Vegas keeps the system operating in its healthy zone.

If you want to add AIMD-style signals on top — event loop delay as a hard safety trigger — that is a reasonable addition. But Vegas as the primary control loop with latency as the signal is the right foundation for a Node adaptive rate limiting package.