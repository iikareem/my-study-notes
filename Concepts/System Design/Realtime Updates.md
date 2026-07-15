---
tags:
  - consistent-hashing
  - messaging
  - pubsub
  - realtime
  - system-design
  - websocket
---

# Realtime Updates — A Complete Guide

Many systems need to push updates to users: chat messages, live notifications, collaborative editing, dashboards, live scores. The question is never "do I need realtime?" — it's "how realtime, and at what cost?"

This guide starts from zero: the client–server **socket**, why that alone is easy, why **multiple servers** create the real problem, and how brokers + consistent hashing fix it without crushing the message system.

---

## Table of Contents

1. [The Core Problem](#1-the-core-problem)
2. [The Three Client-Side Mechanisms](#2-the-three-client-side-mechanisms)
3. [From Zero — The Socket Between Client and Server](#3-from-zero--the-socket-between-client-and-server)
4. [The Hard Problem — Talking Across Servers](#4-the-hard-problem--talking-across-servers)
5. [Brokers — Redis Pub/Sub vs Kafka](#5-brokers--redis-pubsub-vs-kafka)
6. [Naive Fix — Topic Per User (Works, But Heavy)](#6-naive-fix--topic-per-user-works-but-heavy)
7. [Optimized Fix — Consistent Hashing + Server Mailboxes](#7-optimized-fix--consistent-hashing--server-mailboxes)
8. [The Full Write Path](#8-the-full-write-path)
9. [Problems to Know](#9-problems-to-know)
10. [Trade-offs Summary](#10-trade-offs-summary)
11. [Decision Framework](#11-decision-framework)
12. [Things to Always Get Right](#12-things-to-always-get-right)

---

## 1. The Core Problem

HTTP was designed as request–response. The client asks; the server answers; the connection often closes.

Realtime breaks that model. The server must **push** data to the client — without waiting for the next poll, or faster than the client would think to ask.

Three forces are always in tension:

- **Latency** — how quickly does the client see the update?
- **Connection cost** — how expensive is holding that open socket?
- **Complexity** — how hard is it to run when you have many servers?

Client mechanisms (polling, SSE, WebSockets) only solve “how do we talk on one socket.” Most production pain comes after that — when the update must reach a client whose socket lives on a **different** server.

---

## 2. The Three Client-Side Mechanisms

### Polling

The client asks on a fixed schedule.

```
Client:  GET /updates?since=1720000000   (every 5 seconds)
Server:  200 OK { "events": [] }         (usually empty)
```

**Right when:** infrequent updates, simple environments, a few seconds of lag is fine.

**Cost:** most requests return nothing. 1,000 users every 5s ≈ 200 req/s of overhead whether or not anything changed.

**Long polling:** server holds the request open until there is data (or timeout). Fewer empty responses; each waiting client still holds an open HTTP connection.

---

### Server-Sent Events (SSE)

One-way stream: **server → client** over a long-lived HTTP response.

```
Client:  GET /stream
         Accept: text/event-stream

Server:  data: {"type":"notification","count":3}
```

Browser `EventSource` reconnects automatically and can resume from the last event ID.

**Right when:** notifications, feeds, dashboards — client rarely needs to push on the same channel (use normal HTTP POST for client → server).

---

### WebSockets

Full-duplex socket after HTTP upgrade. Either side can send anytime.

```
Client → Server:  {"type":"message","text":"Hello"}
Server → Client:  {"type":"message","text":"Hi back"}
```

**Right when:** chat, collaborative editing, games, trading — frequent bidirectional low-latency traffic.

**Cost:** the connection is **stateful** and pinned to one server for its lifetime. That is what makes multi-server delivery hard (sections 4–7).

---

### Mechanism Comparison

| | Polling | Long Polling | SSE | WebSocket |
| --- | --- | --- | --- | --- |
| **Direction** | Client → Server | Client → Server | Server → Client | Both |
| **Latency** | Interval-bounded | Near-zero | Near-zero | Near-zero |
| **Connection overhead** | Low per-request | Medium (held open) | Medium (persistent) | High (persistent, stateful) |
| **Reconnection** | Just retry | Manual | Automatic (EventSource) | Manual |
| **Proxy/firewall** | Always fine | Usually | Usually | Sometimes blocked |
| **Best for** | Status, simple dashboards | Notifications | Live feeds, notifications | Chat, collab, games |

---

## 3. From Zero — The Socket Between Client and Server

Forget brokers for a moment. Start with one user and one server.

We say **socket**, not “pipe.” A Unix pipe is an OS IPC channel (usually between processes on the same machine). Here the link is a network **TCP/WebSocket socket**: one client end, one server end. Same “two ends talk” idea — accurate name for this layer.

### What a realtime connection really is

A WebSocket (or long-held SSE) is a **socket** between exactly two ends:

```
┌────────┐                    ┌──────────────────┐
│ Client │ ←──── socket ────→   │ Realtime Server  │
└────────┘                    └──────────────────┘
```

Once the socket exists:

- **Client → Server:** messages on the socket — no special problem.
- **Server → Client:** push on the same socket — no special problem.

On **one** machine, “user 42 is connected here” is local memory. When something happens for user 42, that server writes on **their** socket and they’re done.

```
Something happened for user 42
        │
        ▼
Server looks up: "user 42's socket is in my memory"
        │
        ▼
Write frame on that socket → client sees it
```

**This is the part you already have right:** client ↔ server on an open socket is not the hard part. Both directions on that socket are fine.

### SSE is a one-way socket (still easy on one server)

SSE only pushes server → client. Client → server uses a separate normal HTTP request. Still easy on one server: you know which open stream belongs to which user.

### Why “one socket” does not mean “one global server”

In production you run **many** realtime servers behind a load balancer so you can hold millions of sockets.

```
                 Load balancer
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
   Server A       Server B       Server C
   (socket: u1)     (socket: u2)     (socket: u3)
                  (socket: u42)
```

Each user’s socket lives on **exactly one** of those servers (until reconnect). Server B holds user 42’s socket. Server A has no direct access to it.

That single fact creates the whole backend problem.

---

## 4. The Hard Problem — Talking Across Servers

### Scenario that breaks the one-socket story

User 7 (socket on **Server A**) sends a chat message to user 42 (socket on **Server B**).

```
User 7 ──socket──► Server A
                     │
                     │  “deliver to user 42”
                     ▼
                  ??? How does A reach the socket on B ???
                     │
User 42 ◄──socket── Server B
```

Server A can talk to user 7 easily (local socket).  
Server A **cannot** write on user 42’s socket — that socket object only exists in Server B’s process memory.

You need a way for **any** server that knows about an event to reach **the** server that holds the destination user’s socket.

### Two layers people mix up (keep them separate)

| Layer | Question | What’s hard? |
| --- | --- | --- |
| **Client ↔ Server socket** | How do we stream to/from *this* browser? | Mechanism choice (poll / SSE / WS). On one server: easy. |
| **Server ↔ Server delivery** | How does the server that *got* the event reach the server that *holds* the client? | Fan-out, routing, broker load. **This** is the hard part. |

Application servers that handle `POST /messages` are often **not** the same machines that hold WebSockets. Same issue: writer ≠ socket owner → need a bridge.

### What you need conceptually

1. A place every server can **publish** “something for user 42 / room 9”.
2. A way for the server that owns the socket to **receive** that and write on the socket.

That bridge is usually a **message broker**: Redis Pub/Sub or Kafka (or similar).

---

## 5. Brokers — Redis Pub/Sub vs Kafka

### Redis Pub/Sub

Publishers send to a **channel**; every current subscriber gets the message now.

```
PUBLISH user:42  '{"type":"message","text":"Hello"}'
SUBSCRIBE user:42   → realtime server pushes on user 42's socket
```

**Good for:** ephemeral events (typing, presence, live cursors) where losing a message if nobody is subscribed is OK.

**Limits:** no persistence, no replay, not ideal as the only path for “must eventually deliver” chat/notifications (you still persist to DB separately).

### Kafka

Append-only log of **topics** / **partitions**. Consumers read at their own pace and can replay.

**Good for:** durable streams, high volume, multiple consumers (realtime + analytics + search), ordering per partition key (e.g. `conversation_id`).

**Cost:** more operational complexity than Redis Pub/Sub.

### Choosing

| | Redis Pub/Sub | Kafka |
| --- | --- | --- |
| **Persistence** | None | Configurable |
| **Replay** | No | Yes |
| **Throughput** | Moderate | Very high |
| **Best for** | Presence, typing, live cursors | Chat history pipeline, notifications, multi-consumer |
| **Complexity** | Low | Higher |

Both solve the same *shape* of problem: **decouple** “who produced the event” from “who holds the client socket.”

---

## 6. Naive Fix — Topic Per User (Works, But Heavy)

Natural first design — and **your instinct is correct** that it works:

> One channel/topic per user. Each realtime server **subscribes** to the channels of users whose sockets it holds.

```
Server B holds sockets for users 42, 99, 7
  → SUBSCRIBE user:42, user:99, user:7

Anyone publishes to user:42
  → only servers subscribed to user:42 get it
  → Server B writes on user 42's socket
```

**Why this seems perfect**

- Precise targeting (no need to broadcast to every server)
- Mental model matches “deliver to this user”
- Works with Redis channels or Kafka topics/partitions mapped per user

**Why it gets heavy at scale**

| Pressure | What happens |
| --- | --- |
| **Huge channel/topic count** | Millions of users → millions of subscriptions / metadata on the broker |
| **Subscribe churn** | Connect/disconnect = subscribe/unsubscribe storms |
| **Broker fan-out work** | Each publish must match subscribers; many tiny channels = lots of bookkeeping |
| **Connection ↔ subscription sync** | If the registry is wrong, messages go nowhere or to the wrong place |

So: **topic-per-user is logically right, operationally expensive** when every user is a first-class channel on a shared broker.

You want the same routing idea — “only the right server(s) hear this” — with **far fewer** broker channels.

---

## 7. Optimized Fix — Consistent Hashing + Server Mailboxes

This is the important optimization. Read it slowly; it closes the loop with load balancing.

### Idea in one sentence

> Don’t give every user a broker channel. Give every **realtime server** a mailbox. Use **the same hash** to decide (1) which server a user’s **socket** should land on, and (2) which **mailbox** to publish into when delivering to that user.

### Step A — Consistent hashing assigns users to servers

Put servers on a hash ring. `hash(user_id)` picks the owner server.

```
hash(user_42) → Server B   (B owns user 42's "home")
hash(user_7)  → Server A
hash(user_99) → Server B
```

Roughly even load; adding/removing a server only remaps ~`1/N` of users (better with virtual nodes).

### Step B — Connection load balancing uses the **same** hash

On connect / WebSocket upgrade, the edge (LB, gateway, or sticky router) computes:

```
server = consistent_hash(user_id)   // same function as delivery
→ open the socket on THAT server
```

So user 42’s socket is **guaranteed** (in the steady state) to live on Server B — the same B that the delivery path will target.

**Why this matters:** if connect hashed to B but delivery hashed to C, you’d publish to the wrong mailbox and user 42 would never see the event (or you’d need a secondary registry lookup every time).

```
CONNECT path                          DELIVER path
─────────────                         ────────────
hash(user_42) → Server B              hash(user_42) → Server B
open socket on B                        PUBLISH mailbox:B
                                      B reads mailbox → write socket
```

Same function → same server → socket and mailbox meet.

### Step C — One mailbox (broker channel) per server, not per user

```
mailbox:server-A
mailbox:server-B
mailbox:server-C
```

Each realtime server subscribes **only to its own mailbox** (one subscription per machine, not one per user).

Delivery:

```
1. Event should reach user_42
2. owner = consistent_hash(user_42)          → Server B
3. PUBLISH mailbox:server-B  { user_id: 42, ...payload }
4. Server B receives it on its mailbox
5. Server B looks up local socket for user 42
6. Write on the socket
```

**Compare costs**

| Design | Broker subscriptions | Publish target |
| --- | --- | --- |
| Topic per user | ~number of online users | `user:{id}` |
| Mailbox per server | ~number of realtime servers | `mailbox:{server}` via hash(user) |

With 50 realtime servers and 2M online users, you go from ~2M channels to ~50. That’s the win.

### Connection registry — what it is (from scratch)

Hashing answers: *“Where **should** user 42 live?”*  
A **connection registry** answers: *“Where **is** user 42’s socket right now?”*

It is just a shared map (usually Redis):

```
user_42  →  server-B
user_7   →  server-A
user_99  →  server-B
```

**Who writes it**

| Moment | What happens |
| --- | --- |
| User connects | Server that accepted the socket sets `user_id → my_server_id` |
| User disconnects | That entry is deleted (or expires via TTL) |
| User reconnects to another server | Map is overwritten with the new server |

**Who reads it**

When you need to push an event to user 42, you look up the map, then publish to **that** server’s mailbox:

```
1. registry[user_42]  →  server-B
2. PUBLISH mailbox:server-B  { to: 42, ... }
3. Server B writes on the local socket
```

#### Hash vs registry — same goal, different source of truth

| | Consistent hash | Connection registry |
| --- | --- | --- |
| **Question** | Where should this user go? | Where is this user actually connected? |
| **Needs shared store?** | No (pure function) | Yes (Redis / etcd / …) |
| **Always correct?** | Only if connect **always** obeys the hash | Yes if kept up to date on connect/disconnect |
| **Fails when** | Socket landed on a different server than the hash predicts | Entry is stale (server died, key not deleted) |

#### Why bother with a registry if you already hash?

Hash-only works in the **happy path**: every connect is routed with `hash(user_id)`, so prediction == reality.

In real systems they sometimes disagree:

- Deploy / ring change: hash now says “go to C”, but the old socket is still on B for a few seconds  
- Bug or fallback LB: connection stuck on a random server  
- Mobile reconnect: client briefly lands elsewhere before sticky routing kicks in  

Then: `hash(user_42) → C` but the socket is still on B → you publish to the wrong mailbox → message lost.

Registry fixes that: **deliver to where the socket actually is**, not where the math hoped it was.

#### How teams usually combine them

```
CONNECT:  prefer hash(user) so load is even and stable
          then WRITE registry[user] = this_server

DELIVER:  READ registry[user]  →  publish to that mailbox
          (fallback: hash(user) if no registry entry — user offline)
```

So: **hash places connections; registry tracks them for delivery.**  
If you are early-stage and connect routing is strict, hash-only is enough. Add a registry when you see “wrong mailbox” misses during deploys or reconnects.

**Cost to remember:** the registry is critical path. Stale entries route to dead servers — use TTLs + delete-on-disconnect + health checks.

---

### Collaborative editing variant — same pattern, different key

Chat asks: *which server owns **this user**?* → hash `user_id`.

Collaborative editing asks: *which server owns **this document**?* → hash `doc_id`.

#### Why `user_id` is the wrong key here

Say Alice and Bob edit `doc_9` together.

```
hash(alice) → Server A     (Alice's socket on A)
hash(bob)   → Server B     (Bob's socket on B)
```

Every keystroke must cross the broker: A → mailbox B, B → mailbox A. You also need a shared place for document state (OT/CRDT). That state would live in Redis/DB with high chatter — or get out of sync across A and B.

#### Why hashing `doc_id` fixes it

```
hash(doc_9) → Server C     // one "home" for this document

CONNECT (Alice):  route to Server C  (because she's opening doc_9)
CONNECT (Bob):    route to Server C  (same doc → same server)
```

Now Alice and Bob share **one server**:

```
┌─────────────────────────────────┐
│           Server C              │
│  socket(Alice)   socket(Bob)        │
│         ▲         ▲             │
│         └── doc_9 ┘             │
│     (OT/CRDT state in memory)   │
└─────────────────────────────────┘
```

- Alice types → Server C applies edit locally → pushes on **Bob’s local socket** (no broker hop for the other editor)  
- Bob types → same, local fan-out  
- Broker mailboxes still matter when an API worker or another service must notify `doc_9` — publish to `mailbox:hash(doc_9)` / Server C

#### Side-by-side

| Use case | Hash key | Goal |
| --- | --- | --- |
| Chat / notifications / presence | `user_id` | Find the server that holds **that user’s** socket |
| Collaborative doc / multiplayer room | `doc_id` or `room_id` | Put **everyone in the same session** on one server |

Same machinery (hash + mailbox + optional registry). Only the **thing you hash** changes — user vs shared workspace.

#### Watch-outs

- One huge viral doc can overload one server (hot key) — same sharding hotspot problem; mitigate with doc-level limits or splitting sessions  
- A user in two docs may need two connections, or one connection with the server speaking for multiple docs via the broker  
- Registry (if used) might store `doc_id → server` or `user+doc → server`, not only `user → server`

### Virtual nodes

Each physical server appears many times on the ring so load spreads evenly and on failure clients don’t all dump onto a single neighbor.

---

## 8. The Full Write Path

Put the socket + broker + hash together:

```
[Event Source]
   e.g. user 7 sends a message to user 42
      │
      ▼
[Application Server]
   validate + persist to DB
      │
      ▼
[Routing]
   owner = consistent_hash(user_42)  → Server B
      │
      ▼
[Broker]
   PUBLISH mailbox:server-B  { to: 42, event: ... }
   (Redis Pub/Sub or Kafka topic/partition for that mailbox)
      │
      ▼
[Realtime Server B]
   subscribed only to mailbox:server-B
   finds local socket for user 42
      │
      ▼
[Client socket]  WebSocket / SSE / long-poll resume
      │
      ▼
[User 42's client]
```

**Why the broker still exists even with hashing**

Hashing answers *which* server. The broker is still how *another* process (API server, Server A, a worker) **gets the bytes** to Server B without holding a direct RPC mesh to every realtime node. Hashing reduces *how many* broker channels you need; it doesn’t remove the need for a cross-process bus.

**Why persist before (or when) publishing**

For chat/notifications: DB/commit is source of truth; pub/sub is the *fast push*. If the push is lost, reconnect replay from DB/Kafka offset still works.

---

## 9. Problems to Know

### Connection storms

Server dies → all its clients reconnect at once → next server can die too.

**Mitigate:** exponential backoff + jitter; LB rate limits on upgrades; graceful drain (signal reconnect before kill).

### Ordering and delivery

Broker delivery is async; frames can reorder.

- Chat: per-conversation sequence numbers; clients reorder  
- Collab: OT or CRDTs for concurrent edits  

Be explicit: at-most-once (pure pub/sub) vs at-least-once (persist + ack + retry) vs exactly-once (idempotency end-to-end).

### Fan-out at scale (celebrity / huge rooms)

One event × millions of destinations.

- Push only to **currently connected** users; offline catch-up from storage  
- Hybrid push (normal accounts) / pull (mega accounts)  
- Parallel workers chunk follower lists  

### Presence

Heartbeats every 15–30s; Redis TTL ~`2 × interval`; don’t broadcast “online” to every contact on every app open without debouncing.

### Backpressure

Slow client fills send buffer → drop or disconnect that client; don’t block the whole realtime process.

### Hash ring / mailbox mismatch (the “I don’t get messages” bug)

If connect routing and deliver hashing disagree (different hash functions, stale ring, LB ignoring hash), publishes hit an empty mailbox. **Always use one shared hash + ring config** for both paths — this is the bug version of the insight you already have.

---

## 10. Trade-offs Summary

| Mechanism | Latency | Complexity | Connection cost | Best for |
| --- | --- | --- | --- | --- |
| Polling | Interval | Minimal | Low | Infrequent updates |
| Long polling | Near-zero | Low | Medium | Simple push without SSE |
| SSE | Near-zero | Low | Medium | Server → client feeds |
| WebSocket | Near-zero | High | High (stateful) | Bidirectional realtime |

| Broker | Durability | Throughput | Best for |
| --- | --- | --- | --- |
| Redis Pub/Sub | None | Moderate | Ephemeral presence/typing |
| Kafka | Configurable | Very high | Durable multi-consumer streams |

| Cross-server design | Benefit | Cost |
| --- | --- | --- |
| Topic/channel per user | Precise, obvious | Broker subscription explosion |
| Mailbox per server + consistent hash | Few channels; connect & deliver agree | Shared hash/ring discipline; remap on topology change |
| Connection registry | Flexible after reconnects | Extra critical dependency |

---

## 11. Decision Framework

### Communication shape

```
Server → Client only?
  ├── Rare updates?     → Polling
  ├── Frequent?         → SSE
  └── High volume + durable backend? → SSE + Kafka (or similar)

Client ↔ Server both ways?
  ├── Client sends rarely?  → SSE + HTTP POST
  └── Both ways often?      → WebSocket
```

### Cross-server delivery

```
Ephemeral OK if dropped?
  → Redis Pub/Sub into server mailboxes (or small set of channels)

Must deliver / replay?
  → Persist first; Kafka (or DB catch-up) for durability;
     still push via mailbox/channel for online users

Scale pain = too many per-user channels?
  → Consistent hash(user) for connect + mailbox:server publish
```

### Offline

```
Seconds  → buffer / replay on reconnect
Minutes  → DB undelivered + fetch since cursor
Hours+   → push notification (APNs/FCM) to wake client
```

---

## 12. Things to Always Get Right

1. **Socket vs bus** — Client↔server on one socket is easy. Multi-server delivery needs a bus (Redis/Kafka) plus routing (hash mailbox or registry).
2. **Same hash for connect and deliver** — Otherwise mailboxes and sockets disagree and messages vanish.
3. **Prefer server mailboxes over topic-per-user at scale** — Same idea, far less broker load.
4. **Backoff + jitter** on reconnect — prevent storms.
5. **Heartbeats** — don’t trust silent sockets for presence.
6. **Resume cursor** — “last event id / seq” so reconnects don’t permanently miss history.
7. **Persist what must not be lost** — pub/sub is acceleration, not always the source of truth.
8. **Graceful drain** before killing a realtime node.
9. **Monitor:** connections per server, push latency, reconnect rate, broker lag, heartbeat misses, registry/hash mismatches.

---

## Quick mental model (keep this)

```
ONE SERVER
  socket(client ↔ server) both ways → fine

MANY SERVERS
  socket still fine locally
  problem = reach a socket on another server

BRIDGE
  Redis Pub/Sub or Kafka

NAIVE
  subscribe to every user channel → correct, heavy

OPTIMIZED
  consistent_hash(user) → home server
  same hash on connect (socket lands on home)
  publish to mailbox:home_server
  home server writes the local socket
```

Your core idea was right: the socket isn’t the problem; **cross-server delivery** is; **per-user topics work but load the broker**; **consistent hashing + per-server mailboxes**, with the **same hash on connection init**, is the scalable form of that idea.

## See also

- [[WebSocket vs Server-Sent Events (SSE)]] · [[Consistent Hashing]]
- [[Message Brokers Guide — RabbitMQ & Kafka]] · [[System Design MOC]]
