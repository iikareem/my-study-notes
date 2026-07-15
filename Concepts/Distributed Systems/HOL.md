---
tags:
  - distributed-systems
---

## **Head-of-Line Blocking — Why Being Polite in Order Destroys Your Latency**

This one is invisible until you understand it, and then you see it everywhere — in HTTP, in databases, in message queues, in your own application code.

---

## Start from zero

Imagine a single-lane road through a tunnel. Cars go through one at a time, in order.

A fast sports car pulls up behind a slow truck. The sports car could easily go 100mph — but it's stuck. It cannot pass. It waits behind the truck for the entire duration of the truck's slow passage.

The sports car's speed is irrelevant. Its latency is determined entirely by the truck in front of it.

That truck is the **head of line.** The sports car is being **blocked** by it.

This is head-of-line blocking — **a fast request forced to wait behind a slow one simply because of ordering.**

---

## Where it lives in HTTP

### HTTP/1.1

In HTTP/1.1, each TCP connection handles **one request at a time.** You send a request, wait for the full response, then send the next one.

Browsers worked around this by opening **6 parallel TCP connections per domain** — essentially 6 separate tunnels. But each tunnel still has the same problem internally.

If one request on one connection is slow — large image, slow endpoint — everything else queued on that connection waits.

### HTTP/2 was supposed to fix this

HTTP/2 introduced **multiplexing** — multiple requests share a single TCP connection simultaneously, interleaved as frames. No more waiting for one response before sending the next.

```
HTTP/1.1:  [Request 1]---->[Response 1]  then  [Request 2]---->[Response 2]
HTTP/2:    [Request 1 frame][Request 2 frame][Response 2 frame][Response 1 frame]
```

This looks like it solves head-of-line blocking. And at the HTTP layer, it does.

But there's a deeper problem.

### TCP head-of-line blocking — HTTP/2's hidden flaw

HTTP/2 runs over **TCP.** TCP guarantees ordered, reliable delivery. If a single packet is lost in transit, TCP stops delivering everything behind it and waits for that packet to be retransmitted — even if the data behind it belongs to a completely different request.

So at the HTTP layer you have multiplexing. But at the TCP layer, one lost packet freezes **all streams** simultaneously. HTTP/2 actually made this _worse_ than HTTP/1.1 in lossy network conditions — because HTTP/1.1's 6 separate connections meant a packet loss on one connection didn't affect the other five.

This is why **HTTP/3 exists.** HTTP/3 runs over **QUIC** instead of TCP — a UDP-based protocol that implements its own reliability per stream, so a lost packet only blocks the one stream it belongs to, not every stream on the connection.

---

## Where it lives in your database connection pool

You have a connection pool of say 10 connections to PostgreSQL. Your app handles many concurrent requests.

A slow query — maybe a report generation, maybe a missing index — holds one connection for 30 seconds. Meanwhile fast queries pile up waiting for a free connection.

```
Pool: [slow query 30s][fast][fast][fast][fast][fast][fast][fast][fast][fast]

Waiting: [request][request][request][request][request]...
```

Your fast queries that would take 5ms are sitting in queue for 30 seconds waiting for a connection — not because they're slow, but because a slow query is at the head of the line consuming a resource they need.

This is why connection pool sizing is not just about throughput — it's about **isolating slow operations from fast ones.** A common pattern in serious systems is maintaining **separate connection pools** for different query classes: one pool for fast transactional queries, a separate pool for slow analytical queries. They can't block each other.

---

## Where it lives in message queues

You have a queue of jobs. A worker pulls them off one at a time in order.

Job #1 is broken — it keeps failing and retrying. Jobs #2 through #1000 are perfectly fine but cannot be processed because job #1 is permanently at the head of the line.

This is called a **poison message** — and it's a direct manifestation of head-of-line blocking in queues.

The fixes:

- **Dead letter queue** — after N failed attempts, move the message sideways into a separate failure queue so processing can continue
- **Parallel consumers** — multiple workers pulling from the queue so one stuck message doesn't block everything
- **Priority queues** — separate lanes for different message types so a slow lane can't block a fast one

---

## Where it lives in your own code

```javascript
// Processing jobs sequentially
for (const job of jobs) {
  await processJob(job); // if job[0] is slow, job[1] waits regardless
}
```

This is head-of-line blocking you wrote yourself. Job at index 0 being slow has nothing to do with job at index 5 — but job 5 waits anyway because you're processing in strict order.

The fix depends on whether order matters:

```javascript
// If order doesn't matter — process concurrently with a concurrency limit
await Promise.all(jobs.map(job => processJob(job))); // all at once (careful with limits)

// Controlled concurrency — p-limit or similar
import pLimit from 'p-limit';
const limit = pLimit(5); // max 5 concurrent
await Promise.all(jobs.map(job => limit(() => processJob(job))));
```

Now a slow job doesn't hold up unrelated fast jobs.

---

## The unifying idea

Head-of-line blocking appears whenever you have:

- **A shared resource** (connection, channel, worker, queue)
- **Strict ordering** enforced on that resource
- **Variable processing time** across items

The slow item at the front determines everyone else's latency — not their own complexity, not their own priority. Just their position relative to something slow.

Once you see this pattern you start asking a different question when designing systems: _not just "how fast is this on average" — but "what happens to everything else when one thing is slow?"_

That question leads to better architecture almost every time.

**HTTP/1.1** — one request at a time per connection. Request 2 can't even be _sent_ until request 1 gets a response. The 6-connection hack is literally just "open more pipes so we can have 6 in-flight at once." It works but it's wasteful — 6 TCP handshakes, 6 TLS negotiations, memory on both sides for 6 connections.

**HTTP/2** — you send request 1 and request 2 on the same connection simultaneously, labeled with stream IDs. Response 2 can come back before response 1 if it's ready. The connection is genuinely shared concurrently, not just pipelined. That's multiplexing. But they're still sharing one TCP pipe underneath, so a lost packet freezes _all_ streams until retransmission fills the gap — you solved the application ordering problem and walked into the transport ordering problem.

**HTTP/3** — moves to QUIC which implements streams at the transport layer itself, so a lost packet only affects its own stream. Each stream is independently delivered.

