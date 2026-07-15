---
tags:
  - backpressure
  - distributed-systems
  - flow-control
  - messaging
---

## **Backpressure — The Concept That Explains Half of All Production Meltdowns**

Most engineers know what a slow database looks like. Fewer understand the _systemic failure pattern_ that happens when you don't respond to slowness correctly — where one slow component causes everything upstream to collapse in a cascade.

That pattern has a name: **missing backpressure.**

---

## Start from zero: the pipe analogy

Imagine water flowing through pipes of different widths.

```
[Source] ==wide pipe==> [Narrow pipe] ==> [Destination]
```

If water flows into the narrow pipe faster than it can flow out — what happens? It backs up. Pressure builds. Eventually something bursts.

In software, data flows between components. Every component has a **throughput limit** — how much it can process per second. When a fast producer sends data to a slow consumer, you have the same problem.

**Backpressure is the signal the slow consumer sends upstream saying: "slow down, I can't keep up."**

A system that handles this signal correctly stays stable. A system that ignores it — or has no mechanism for it — eventually falls apart.

---

## The three ways systems respond to overload

When a consumer is overwhelmed, there are only three options:

**1. Drop** — throw away the excess. Fast, but you lose data.

**2. Buffer** — queue the excess and process it later. Preserves data, but the queue grows. If the producer never slows down, the queue grows **forever** until you run out of memory and crash.

**3. Push back** — tell the producer to slow down. This is backpressure done right.

Most systems accidentally choose option 2 — not as a deliberate decision, but because buffering is the default behavior of queues, streams, and message brokers. The buffer just quietly grows until something breaks.

---

## A real Node.js example

```javascript
const fs = require('fs');

const readable = fs.createReadStream('huge-file.csv'); // very fast
const writable = fs.createWriteStream('output.csv');   // slower

readable.on('data', (chunk) => {
  writable.write(chunk); // what if writable can't keep up?
});
```

`fs.createReadStream` can read from disk extremely fast. `writable.write()` returns `false` when its internal buffer is full — meaning _"I'm overwhelmed, stop sending."_

But this code ignores that signal entirely. It keeps pumping data in. The internal buffer grows unbounded. On a large enough file, this crashes with an out-of-memory error.

The fix is one line conceptually — pause the readable when the writable says stop, resume when it drains:

```javascript
readable.pipe(writable); // pipe handles backpressure automatically
```

`pipe` was specifically designed to wire up the backpressure signals between streams. That's most of what it does.

---

## Now scale this to distributed systems

The same problem exists between services, just harder to see.

```
[API Server] --> [Message Queue] --> [Worker Service] --> [Database]
```

Say your database gets slow — maybe a long-running migration, maybe just load. Workers slow down. The queue starts filling up. The API server keeps accepting requests and enqueuing jobs because it has no idea the downstream is struggling.

If nobody pushes back:

- The queue grows until it hits memory or disk limits
- Workers start timing out trying to write to the DB
- The queue retries, making DB load worse
- API server is still happily accepting new traffic
- Everything collapses at once

This is a **cascade failure** — and the root cause is that the slowness at the DB had no mechanism to propagate upstream as a signal to slow down intake.

---

## Where backpressure is handled for you vs. where it isn't

**Handled automatically:**

- Node.js `stream.pipe()`
- TCP — the protocol has built-in flow control at the network level
- gRPC streaming — has flow control built into HTTP/2

**You have to handle it yourself:**

- Reading from a database and writing to an API
- Consuming a message queue and writing to another service
- Any `async` loop that pulls and processes data

The dangerous pattern is any place where you have:

```javascript
const jobs = await queue.getAll(); // pulls everything
for (const job of jobs) {
  await processJob(job); // what if this gets slow?
}
```

If `processJob` slows down, you're still pulling all jobs upfront. You need to think about **how many things you process concurrently**, **what your queue depth limit is**, and **what happens when you hit it**.

---

## The mental shift

Most engineers think about each component in isolation — is this service healthy? is this query fast?

Backpressure forces you to think about **the flow between components** — what happens to the system as a whole when one part slows down? Does the slowness propagate as a controlled signal, or does it propagate as a catastrophic backup?

A well-designed system treats backpressure as a first-class concern from the start. A poorly designed one discovers it during a 3am incident when the queue has 4 million unprocessed jobs and nobody knows why.

,,

RabbitMQ handles back pressure mainly with **prefetch and ACKs**. Since it is push-based, the broker limits how many unacked messages a consumer can have (`prefetch`). If the broker itself becomes overloaded, it applies publisher flow control and slows/block producers.

Apache Kafka handles back pressure naturally through its pull model. Consumers fetch messages at their own speed, so slow consumers simply accumulate lag instead of being overwhelmed. Producers may also slow down if brokers cannot accept more writes.

# Push vs Pull Message Queues

## Core Difference

**Push-Based**: Queue actively sends messages to your consumer

- Consumer is passive (listens and reacts)
- Lower network overhead (no polling)
- Hard to handle backpressure (can get flooded)

**Pull-Based**: Consumer asks queue for messages when ready

- Consumer is active (requests messages)
- Higher network overhead (constant polling)
- Easy backpressure (you control pace)

## Network Cost

**Pull-Based**:

- Sends polling requests constantly (even when empty)
- 100s of wasted requests if nothing is happening
- Example: Poll every 100ms = 10 requests/sec

**Push-Based**:

- Only sends when data exists
- No wasted network calls
- More efficient for bandwidth

## Backpressure Handling

**Pull-Based** (✅ Easy):

```
Pull message → Process → Ack → Pull next
You control the flow naturally
```

**Push-Based** (❌ Hard):

```
Queue pushes messages → You process → Ack
Messages pile up while you're busy
Hard to tell queue to slow down
```

## Acknowledgment (Ack)

**Pull-Based**: You ack after processing succeeds

- Safe by default (failures don't lose messages)
- Acking is part of flow control

**Push-Based**: Ack can be automatic or manual

- Auto-ack: Risky (messages lost if you crash)
- Manual-ack: Still have backpressure issues

## The Tradeoff

|Aspect|Pull|Push|
|---|---|---|
|Network Cost|❌ High|✅ Low|
|Backpressure|✅ Easy|❌ Hard|
|Control|✅ You decide pace|❌ Queue decides pace|
|Safe Acking|✅ Natural|❌ Requires care|

## When to Use Each

**Use PULL when**:

- Processing is slow/expensive
- Limited resources
- Need fine-grained control

**Use PUSH when**:

- Processing is very fast
- Plenty of resources
- Network bandwidth is critical
