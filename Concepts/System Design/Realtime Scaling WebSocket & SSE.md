# Realtime Scaling: WebSocket & SSE — A Complete Guide

> **The hard part is not the connection between client and server.** The hard part is delivering a message to a client whose connection lives on a _different_ server. Both WebSocket and SSE have this exact same problem — and understanding why leads you to the full solution.

---

## Table of Contents

1. [What Is a Socket?](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#1-what-is-a-socket)
2. [The Three Connection Shapes](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#2-the-three-connection-shapes)
3. [SSE and WebSocket — Similarities and Differences](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#3-sse-and-websocket--similarities-and-differences)
4. [Why Both Have the Same Scaling Problem](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#4-why-both-have-the-same-scaling-problem)
5. [The Core Problem — One Server Is Fine, Many Servers Break](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#5-the-core-problem)
6. [The Fix — Message Brokers](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#6-the-fix--message-brokers)
7. [Routing — How to Find the Right Server](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#7-routing--how-to-find-the-right-server)
8. [The Grouping Principle — Hash by What Is Shared](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#8-the-grouping-principle--hash-by-what-is-shared)
9. [The Full Picture — End to End](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#9-the-full-picture--end-to-end)
10. [Quick Reference](https://claude.ai/chat/05b87e7a-4541-4319-8699-4b003e749a2c#10-quick-reference)

---

## 1. What Is a Socket?

Before anything else, let's be concrete about what a "socket" actually is, because this word gets used loosely and it matters.

### The Physical Reality

When a client (browser, app) connects to a server, the operating system on each side creates a **socket** — an object in memory that represents one end of the connection. Think of it like a pipe:

```
  Browser                           Server
┌──────────┐                     ┌──────────┐
│  socket  │ ════ TCP stream ═══ │  socket  │
│  object  │                     │  object  │
└──────────┘                     └──────────┘
  lives in                          lives in
  browser's                         server's
  memory                            memory
```

The socket object holds:

- The IP address and port of the other side
- A send buffer (bytes waiting to go out)
- A receive buffer (bytes that arrived but haven't been read yet)
- The state of the connection (open, closing, closed)

### What "Writing to a Socket" Means

When the server wants to push data to a client, it calls something like `socket.write(data)`. This copies bytes into the socket's send buffer, and the operating system handles flushing them over the network to the client.

This is a **local, in-process operation**. The socket object is just a variable in the server's memory. Writing to it is as simple as writing to any other in-memory object.

### Why This Matters for Scaling

The socket object lives in **exactly one server's memory**. No other server has a reference to it. No other server can write to it. This single fact is the root of every scaling challenge in realtime systems.

```
Server A memory:        Server B memory:
  socket_map = {          socket_map = {
    user_7:  <obj>,         user_42: <obj>,   ← Server A cannot
    user_15: <obj>          user_99: <obj>       touch this
  }                       }
```

If you need to push a message to User 42, and User 42's socket lives on Server B, but your code is running on Server A — you have a problem. There is no `serverB.socket_map[42]` you can call. Different process, different machine, different heap.

---

## 2. The Three Connection Shapes

Polling, SSE, and WebSocket are not a feature menu. They are three different **shapes** of communication between client and server.

|Shape|Direction|Persistent connection?|Server holds state?|
|---|---|---|---|
|Polling|Client pulls|No — new request each time|No|
|SSE|Server pushes|Yes — one long-lived response|**Yes**|
|WebSocket|Both directions|Yes — upgraded TCP connection|**Yes**|

### Polling

The client asks the server on a timer: "anything new?" The server answers immediately, even if nothing changed.

```
Client                     Server
  │─── GET /updates ───────►│
  │◄─── { events: [] } ──────│   (nothing)

  │   (wait 3 seconds)       │

  │─── GET /updates ───────►│
  │◄─── { events: [x] } ─────│   (something!)
```

Every request is independent. Any server can handle it. **No persistent connection, no scaling problem.** The downside is latency (you only get data as fast as you poll) and wasted requests when nothing has changed.

**Use when:** Admin dashboards, status checks, anything where a few seconds of lag is fine.

### SSE (Server-Sent Events)

The client makes one HTTP request and keeps it open. The server streams data back over time as lines of text, pushing whenever something happens.

```
Client                     Server
  │─── GET /stream ────────►│
  │◄─── HTTP 200 (open) ──────│
  │◄─── data: {event 1} ──────│   (pushed immediately)
  │◄─── data: {event 2} ──────│   (pushed 5 seconds later)
  │◄─── data: {event 3} ──────│   (pushed 10 seconds later)
  │                           │   (connection never closes)
```

The client never sends anything on this channel — it just listens. To send data to the server, it uses a **separate, normal HTTP POST**.

The browser has built-in SSE support (`EventSource` API) including automatic reconnection. If the connection drops, the browser reconnects and sends a `Last-Event-ID` header so the server knows where to resume.

**Use when:** Notifications, live feeds, real-time logs, scoreboards — one-way push from server to client.

### WebSocket

The client starts with HTTP, then asks to "upgrade" the connection. Once both sides agree, the connection stops being HTTP and becomes a raw two-way channel where both sides can send data at any time.

```
Client                           Server
  │─── GET /ws (Upgrade) ────────►│
  │◄─── 101 Switching Protocols ───│   (upgrade accepted)
  │                                │
  │◄══ frame: "new message" ═══════│   (server pushes)
  │══► frame: "my reply" ══════════│   (client pushes)
  │◄══ frame: "typing..." ═════════│   (server pushes)
```

Both directions flow on the same connection, with minimal overhead per message.

**Use when:** Chat, collaborative editing, multiplayer games, live trading — anything needing frequent, low-latency, bidirectional traffic.

---

## 3. SSE and WebSocket — Similarities and Differences

People often frame SSE vs WebSocket as "which is better." That's the wrong question. They cover different communication shapes. Here's what's actually similar and what's actually different.

### What They Share (The Important Part)

Both SSE and WebSocket:

- Keep a **persistent, long-lived connection** open between client and server
- Allow the **server to push data to the client** at any time without the client asking
- Pin that connection to **one specific server** for its entire lifetime
- Have the **same cross-server delivery problem** when you scale horizontally

That last point is what this whole guide is about. When people say "SSE is simpler to scale than WebSocket," they mean simpler on the frontend. On the backend, the problem is identical.

### What's Different

||SSE|WebSocket|
|---|---|---|
|**Protocol**|Stays HTTP the whole time|Starts as HTTP, upgrades to a custom protocol|
|**Direction**|Server → Client only|Both directions on the same connection|
|**Client → Server**|Separate HTTP request|Frame on the same connection|
|**Auto-reconnect**|Built into the browser|You write it yourself|
|**Binary data**|No (text only)|Yes|
|**Works through strict proxies/firewalls**|Usually yes|Sometimes blocked|
|**HTTP/2 multiplexing**|Yes (many streams per TCP connection)|No|

### The Right Mental Model

Think of SSE and WebSocket as two different pipe shapes:

```
SSE:
  Client ──────────────────────────────► Server  (sends via separate HTTP POST)
  Client ◄══════════════════════════════ Server  (receives via long-lived response)

WebSocket:
  Client ◄══════════════════════════════ Server
  Client ══════════════════════════════► Server
  (same pipe, both directions)
```

SSE is a one-way listening channel plus a separate send mechanism. WebSocket is a single two-way channel. Pick SSE when the client mostly listens. Pick WebSocket when the client sends frequently on the same channel.

---

## 4. Why Both Have the Same Scaling Problem

### What SSE Actually Is Under the Hood

SSE is not a special streaming protocol. It is a **normal HTTP response that never finishes**. The server writes the response headers, then keeps the body open and writes lines to it over time:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type": "message", "text": "Hello"}

data: {"type": "message", "text": "World"}

(server keeps writing... response never ends)
```

That open response object lives **inside one server process** — in memory, as a variable, tied to a file descriptor that represents the TCP connection to that specific client.

### The Moment You Add a Second Server

```
User 42 opens SSE stream
  → Load balancer picks Server B
  → Server B creates an open response object for User 42
  → Server B holds it in memory

User 7 sends a message to User 42
  → Load balancer sends User 7's POST to Server A
  → Server A processes the message
  → Server A wants to push a notification to User 42
  → Server A has no reference to User 42's response object
  → That object is in Server B's memory
  → Server A cannot write to it
```

This is byte-for-byte the same problem as WebSocket. The push channel — whether it's an open HTTP response or an upgraded TCP socket — lives in exactly one server's memory.

||WebSocket|SSE|
|---|---|---|
|Client sends|Frame on the socket|Separate HTTP POST (lands anywhere)|
|Server pushes|Write to the socket|Write to the open response|
|Cross-server delivery problem|**Yes**|**Yes — identical**|

> The only mechanism that avoids this entirely is **polling**, because each request is stateless and lands on any server. The tradeoff is latency and wasted bandwidth.

---

## 5. The Core Problem

### One Server: Trivial

```
┌─────────┐                  ┌────────────────────────┐
│  User 7 │ ←── socket ────► │        Server A        │
└─────────┘                  │                        │
                             │  socket_map = {        │
┌──────────┐                 │    7:  <socket_obj>,   │
│  User 42 │ ←── socket ───► │    42: <socket_obj>    │
└──────────┘                 │  }                     │
                             └────────────────────────┘
```

"Push to User 42" = one line of code: `socket_map[42].write(data)`. The socket object is right there in memory.

### Many Servers: Broken

```
                    Load Balancer
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
    Server A          Server B          Server C
  socket: user_7    socket: user_42   socket: user_99
```

User 7 (Server A) sends a message to User 42 (Server B):

```
User 7 ──► Server A

  Server A: "I need to push to User 42"
  Server A: socket_map[42] = ???   ← doesn't exist here

  Server B: socket_map[42] = <obj> ← exists, but unreachable from A
```

No shared memory. No global socket table. Server A physically cannot write to Server B's socket object.

This is the only real problem in realtime systems. Everything else — brokers, routing, registries — is the answer to this one problem.

---

## 6. The Fix — Message Brokers

You need a shared bus that every server can write to and read from. When Server A needs to reach User 42 on Server B, it publishes a message to the bus, and Server B picks it up and writes to the local socket.

```
Server A ──PUBLISH──► [ Broker ] ──► Server B ──► socket(User 42)
```

Each realtime server subscribes to its own channel (its "mailbox") on the broker. When a message arrives in its mailbox, it looks up the socket locally and writes the frame.

---

### Redis Pub/Sub

An ephemeral fire-and-forget bus. If nobody is subscribed when a message is published, the message is **gone** — no persistence, no replay.

**Architecture:**

```
                     Redis Pub/Sub
                    ┌─────────────┐
Server A ─PUBLISH──►│ mailbox:B   │──► Server B ──► socket(User 42)
                    └─────────────┘
```

Each WebSocket server subscribes to exactly one channel — its own mailbox. When a message arrives in that mailbox, the server looks up the socket locally and delivers it.

**Good for:** Typing indicators, presence updates, live cursors — things that are refreshed constantly so a lost message doesn't matter.

**Not safe for:** Chat messages, notifications — if the push is lost, the user never sees it.

**Safe pattern:** Always write to the database first, then publish to Redis. If the Redis push is lost, the client fetches missed messages from the DB on reconnect.

```
1. Write to DB          ← safe, permanent
2. Publish to Redis     ← fast push to the online client
3. Push dropped?        ← client fetches from DB on reconnect
```

---

### Kafka

Kafka is a **durable, ordered log** — not a live message bus. This distinction matters. It is not a replacement for Redis Pub/Sub; it plays a different role at a different layer of the system.

**The mental model:**

Think of Kafka as a permanent record of everything that happened. Multiple independent services can read that record at their own pace, from any point in time, without affecting each other.

```
                            Kafka Topic: "chat-messages"
                           ┌────────────────────────────┐
                           │  partition 0: room_9       │
User 7 sends "hello"  ────►│  partition 1: room_14      │
to room_9                  │  partition 2: room_3       │
                           └────────────────────────────┘
                                        │
                    ┌───────────────────┼──────────────────┐
                    ▼                   ▼                  ▼
           WebSocket Push        Email/Push           Moderation
              Service            Notifier              Service
          (delivers to         (alerts offline        (scans for
          live clients)           users)               violations)
```

Every service consumes the same event independently. If you add a new service — say, an analytics pipeline — it just reads the same Kafka topic from the beginning. No changes to anything else.

**How Kafka connects to WebSockets — the key question:**

Kafka does not know about your WebSocket servers or individual sockets. It is not the thing that delivers the final push to the client. The WebSocket Push Service reads from Kafka, then uses Redis Pub/Sub (or a connection registry) to find which WebSocket server holds that user's socket, and delivers from there.

```
FULL FLOW:

User 7 sends "hello" to room_9
  → API Server writes event to Kafka topic "chat-messages", partition for room_9

  [Kafka] stores the event durably

  → WebSocket Push Service reads from Kafka
      for each user in room_9:
        lookup registry["user:42"] → "server-b"
        PUBLISH mailbox:server-b { to: 42, payload: "hello" }

  → Server B receives from Redis, looks up socket(42), writes frame

  → Email Service also reads from Kafka
      user_42 is offline? → send push notification

  → Moderation Service also reads from Kafka
      scan message content → flag if needed
```

Kafka and Redis Pub/Sub are not competing choices. They work together:

- **Kafka** = durable fan-out to multiple services (the upstream layer)
- **Redis Pub/Sub** = fast last-mile delivery to the specific WebSocket server (the downstream layer)

**Partitioning:**

A Kafka topic is split into partitions. You control which partition an event goes to via a **partition key**. Use the same key you use for routing:

```
chat room     → partition key: room_id
collaborative doc → partition key: doc_id
user notification → partition key: user_id
```

Events with the same key always go to the same partition. Events within one partition are strictly ordered. This means all messages in a chat room arrive at the consumer in the order they were sent.

One important constraint: you cannot have more consumers in a consumer group than partitions. Plan your partition count to allow future scaling.

**Good for:** Chat history, notifications, audit logs, or any event that multiple independent services need to consume — push delivery, email alerts, analytics, moderation, all reading the same event.

**Tradeoff:** More operational complexity than Redis alone. Kafka makes sense when you have multiple services consuming the same events, or when you need durable replay.

---

### How They Work Together

```
Event flow:

  [User action]
       │
       ▼
  [API Server]
       │
       ├──► DB write (always first)
       │
       └──► Kafka (if multiple services need this event)
                │
                ├──► WebSocket Push Service
                │         │
                │         └──► Redis Pub/Sub ──► WebSocket Server ──► socket
                │
                ├──► Email/Push Notification Service
                │
                └──► Analytics / Moderation / etc.

  OR, if only one service needs the push:

  [API Server]
       │
       ├──► DB write
       │
       └──► Redis Pub/Sub directly ──► WebSocket Server ──► socket
```

---

### Choosing

|Situation|Use|
|---|---|
|Lost message is fine (presence, cursors)|Redis Pub/Sub|
|Message must reach the user eventually|DB + Redis Pub/Sub|
|Multiple services consume the same event|Kafka → each service → Redis Pub/Sub for last mile|
|Need replay on reconnect or audit log|Kafka|
|Simple system, one push service|Skip Kafka, use DB + Redis directly|

---

## 7. Routing — How to Find the Right Server

The broker moves messages between servers. But you still need to know **which server's mailbox to publish to**. That's the routing problem.

### Why Naive Per-User Channels Don't Scale

The obvious approach: give each user a channel on the broker, and have their server subscribe to it.

```
Server B subscribes to: channel:user:42
Publisher sends to:      channel:user:42
```

This works logically. The problem is scale: with 2 million online users, you have 2 million active broker subscriptions, constantly churning as users connect and disconnect. The broker collapses under the subscription management overhead.

**The fix:** flip the model. Instead of one channel per user, use **one mailbox per server**. Each server subscribes to exactly one channel. Publishers route to a server, not a user.

```
Server B subscribes to: mailbox:server-b   ← always exactly one subscription

Publisher:
  1. Find which server holds User 42's socket
  2. PUBLISH mailbox:server-b  { user_id: 42, payload: ... }

Server B:
  receives message → socket_map[42] → write frame
```

50 servers → 50 broker channels. Always.

Now the question is: how does the publisher know which server holds User 42's socket?

---

### Approach 1: Consistent Hashing (Predict the Server)

Use a hash function to deterministically map a user to a server:

```
hash(user_id) → server index → server name
```

Because the function is deterministic, everyone who runs `hash(user_42)` gets the same answer. No coordination needed at runtime — as long as everyone uses the same server list.

**The two places that must agree:**

```
Load Balancer                    Publisher Service
  user connects                    event for user_42
  hash(user_42) → Server B         hash(user_42) → Server B
  routes connection to Server B    publishes to mailbox:server-b
```

The socket ends up on Server B. The message is published to Server B's mailbox. They agree because they use the same hash function with the same server list.

**Where does the server list come from?**

This is the critical detail. The hash function itself is just an algorithm — it's the same everywhere. But it needs a list of servers as input. Both the load balancer and the publisher must have the **same list at the same time**.

Two ways to manage this:

**Option A — Static config file, deployed to every component:**

```yaml
# servers.yaml — same file deployed everywhere
servers:
  - id: server-a   host: 10.0.0.1
  - id: server-b   host: 10.0.0.2
  - id: server-c   host: 10.0.0.3
```

Every component reads this file at startup, builds the same hash ring locally, and produces identical results. Simple, but painful to update — adding a server means redeploying the config everywhere simultaneously. During the rollout window, different components may have different lists → disagreement → dropped messages.

**Option B — Shared coordination service (etcd / Consul):**

A dedicated service stores the server list. Every component watches it for changes and rebuilds its local ring when the list updates.

```
┌────────────────────────────────────┐
│         etcd / Consul              │
│  /servers/server-a → 10.0.0.1     │
│  /servers/server-b → 10.0.0.2     │
│  /servers/server-c → 10.0.0.3     │
└────────────────────────────────────┘
         ▲           ▲           ▲
    (watching)  (watching)  (watching)
         │           │           │
   Load Balancer  Publisher  Realtime Servers
```

When a new server starts, it registers itself. When it shuts down, it deregisters (or its TTL expires). All watchers get notified and rebuild their rings. This is the production-standard approach — no manual config deployments, no rollout windows.

**The remaining risk with hashing:**

Even with etcd, there's a brief window where the load balancer has updated its ring but the publisher hasn't yet. The LB sends User 42's connection to Server D (new server), but the publisher still thinks User 42 is on Server B. Message goes to the wrong mailbox → silent drop.

This is why the registry exists.

---

### Approach 2: Connection Registry (Record the Server)

Instead of predicting where a socket is, **record it** the moment it's created.

```
User 42 connects to Server B
  → Server B writes to Redis:  user:42 → "server-b"  (with TTL)

Publisher routes a message to User 42
  → reads Redis:  user:42 → "server-b"
  → publishes to mailbox:server-b

User 42 disconnects
  → Server B deletes:  user:42
```

The registry is the source of truth. No prediction, no ring, no disagreement window. The publisher always reads the actual current location.

**TTL as a safety net:** Always set a short TTL (e.g. 90 seconds) on registry entries, and have the realtime server send heartbeats to refresh them. If a server crashes without cleanup, the entry expires on its own and stops misdirecting messages.

### Hashing vs Registry — Which to Use

||Consistent Hashing|Connection Registry|
|---|---|---|
|How it works|Predicts the server|Records the server|
|Extra infrastructure|None beyond the server list|Redis (usually already present)|
|Accurate during deploys/restarts?|No — brief mismatch window|Yes — always current|
|Extra read per publish|No|Yes (one Redis read)|
|Good starting point?|Yes|Add when you see drops|

**Recommended production approach:** Use hashing at the load balancer for even distribution (so connections spread across servers naturally). Write a registry entry on every connect. Publishers read from the registry as the source of truth. You get even distribution _and_ always-accurate routing.

---

## 8. The Grouping Principle — Hash by What Is Shared

There's a deeper principle behind all routing decisions:

> **When multiple connections need to share state or communicate directly, they must land on the same server.**

The hash key determines which connections end up on the same server. Choose it based on what the connections actually share.

### Example 1: Chat and Notifications — Hash by User

In chat, the server needs to push messages **to a specific user**. Each user has their own socket, and messages are addressed to users. There's no benefit to putting two different users on the same server.

```
hash(user_id) → server

User 7  → Server A
User 42 → Server B   ← different servers, fine
User 99 → Server C
```

Cross-user messages travel through the broker. That's expected and cheap.

### Example 2: Collaborative Editing — Hash by Document

In collaborative editing, **everyone editing the same document** needs to share state: the document content, the list of active cursors, the operational transform (OT) or CRDT log. If Alice and Bob edit the same document but land on different servers, every keystroke needs to cross the broker, and the shared document state needs to be synchronized across servers — complex and slow.

The fix: **hash by document ID**, not user ID. Everyone editing the same document lands on the same server.

```
hash(doc_id) → server

Alice opens doc_9 → hash("doc_9") → Server C
Bob opens doc_9   → hash("doc_9") → Server C   ← same server!
Carol opens doc_9 → hash("doc_9") → Server C   ← same server!
```

Now Alice, Bob, and Carol are all on Server C. Their keystrokes fan out locally — no broker hops. The OT/CRDT state lives in Server C's memory, shared across all three connections.

```
Alice types "H"
  → Server C (local)
  → apply OT transform (local, in memory)
  → write to Bob's socket (local)
  → write to Carol's socket (local)
  → done — zero network hops
```

Compare this to user-based hashing:

```
Alice types "H"
  → Server A
  → PUBLISH broker → Server C (Bob) → write
  → PUBLISH broker → Server B (Carol) → write
  → synchronize OT state across A, B, C
  → much more complex and slower
```

### The General Rule

|What's shared|Hash key|Why|
|---|---|---|
|A message addressed to a user|`user_id`|Route to the server holding that user's socket|
|A live document being co-edited|`doc_id`|Put all editors on the same server|
|A multiplayer game room|`room_id`|Put all players on the same server|
|A live video stream chat|`stream_id`|Put all viewers on the same server|

The pattern is always the same: **identify what the group of connections shares, and use that as the hash key**. The load balancer routes based on that key, so all related connections land on the same server, and all the shared state stays local.

**Watch-outs:**

- A very popular document or room puts all its connections on one server — that server can become a hotspot. For extreme cases, you need to split the session or use a fallback broker path.
- A user in multiple documents needs a separate connection per document (since each document hashes to a different server). Most collaboration tools handle this naturally.

---

## 9. The Full Picture — End to End

Let's trace a message through a complete production system.

```
User 7 sends "hello" to User 42
════════════════════════════════════════════════════════

[1] User 7's client sends the message
    WebSocket frame (or HTTP POST) → arrives at Server A

[2] Server A validates and persists
    - Authenticate User 7
    - Validate the message
    - Write to database  ← ALWAYS before publishing
    - ACK to User 7: "your message was saved"

[3] Server A routes to the right server
    target = registry["user:42"]  →  "server-b"
    (or: hash("user:42")  →  "server-b")

[4] Server A publishes to the broker
    PUBLISH mailbox:server-b {
      to: 42,
      payload: { type: "message", from: 7, text: "hello" }
    }

[5] Server B receives from broker
    socket = socket_map[42]
    socket.write(frame)

[6] User 42's client receives and renders it

════════════════════════════════════════════════════════
```

**Why persist before publishing?** If you publish first and then the DB write fails, you've shown the user a message that doesn't exist — inconsistent state. If the DB write succeeds and the publish fails, the client fetches the message from the DB on reconnect. Always safe to miss the push. Never safe to push something that wasn't saved.

**Why a broker instead of Server A calling Server B directly?** If every API server maintained a direct connection to every realtime server, you'd have a full mesh — 20 API servers × 30 realtime servers = 600 persistent connections to manage. The broker reduces this to N connections to one shared bus.

---

## 10. Quick Reference

### Which Connection Shape?

```
Does the server need to push to clients?
├── No → regular HTTP request/response
└── Yes →
    ├── Does the client also need to send frequently on the same channel?
    │   ├── Yes → WebSocket (chat, games, collaborative editing)
    │   └── No  → SSE (notifications, feeds, logs)
    └── Is a few seconds of delay acceptable?
        └── Yes → Polling (simplest to operate)
```

### Which Broker?

```
Is it okay if a message is lost?
├── Yes (typing indicators, presence) → Redis Pub/Sub
└── No →
    ├── One service consumes it → DB + Redis Pub/Sub (persist first)
    └── Multiple services consume it → Kafka → Redis Pub/Sub for last mile
```

### Which Hash Key?

```
What do the connections share?
├── A user's identity (chat, notifications) → hash(user_id)
├── A document being edited together → hash(doc_id)
├── A game or room → hash(room_id)
└── A live broadcast → hash(stream_id)
```

### How to Know Which Server a Socket Is On?

```
Simple setup, stable server list → consistent hashing (predict)
Seeing dropped messages on deploy → add connection registry (record)
Production recommendation → hashing for placement + registry as source of truth
```

### Offline Users?

```
Offline for seconds → reconnect with Last-Event-ID, replay from broker
Offline for minutes → fetch from DB since last-seen message ID
Offline for hours   → mobile push notification (APNs / FCM) to wake the app
```

---

### The Things to Always Do

```
✓ Persist to DB before publishing to broker
✓ Hash key matches what the connections actually share
✓ Load balancer and publisher use the same server list
✓ Registry entries have TTLs + heartbeat refreshes
✓ Clients use exponential backoff + jitter on reconnect
✓ Drain servers gracefully before shutdown
```

---

_The socket is stuck on one server. The broker moves messages across servers. Routing finds the right server. Hash by what the connections share. These four ideas cover the whole problem._