---
tags:
  - redis
  - pubsub
  - socket-io
  - real-time
  - websockets
  - scaling
  - backend
---

# Redis PubSub & Socket.IO Redis Adapter

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [Redis PubSub — The Simple Tool](#2-redis-pubsub--the-simple-tool)
3. [The Socket.IO Redis Adapter](#3-the-socketio-redis-adapter)
4. [How They Relate](#4-how-they-relate)
5. [What The Adapter Hides From You](#5-what-the-adapter-hides-from-you)
6. [Who Gets The Message?](#6-who-gets-the-message)
7. [The Sharded Adapter](#7-the-sharded-adapter)
8. [Using Raw PubSub Too](#8-using-raw-pubsub-too)
9. [Which One Should I Use?](#9-which-one-should-i-use)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. The Problem

You have **one** Socket.IO server. A user connects. You call `io.emit("news", data)`. Everyone gets it. Easy.

Now you have **three** servers behind a load balancer.

```
User A ──→ Server 1
User B ──→ Server 2
User C ──→ Server 3
```

Server 1 calls `io.emit("news", data)`.

Only **User A** gets the message. Server 2 and Server 3 never knew it happened.

Why? Because each server only knows about its own connections. They do not talk to each other.

**This is the problem Redis solves.** It becomes the middle man between your servers.

---

## 2. Redis PubSub — The Simple Tool

PubSub means **Publish / Subscribe**. It is built into Redis.

The idea is very simple:

- You **subscribe** to a channel (a channel is just a name, like `news`)
- Someone **publishes** a message to that channel
- Everyone subscribed gets the message right away

### The Commands

```
SUBSCRIBE   news              # start listening
PUBLISH     news "hello"      # send to everyone listening
UNSUBSCRIBE news              # stop listening
```

### How It Looks

```
Publisher → PUBLISH news "hello"
                  ↓
                Redis
             ↓         ↓
      Subscriber A   Subscriber B
```

Both subscribers get `"hello"` at the same time. Redis does the sending.

### Things To Know

| Thing | What it means |
|---|---|
| **No saving** | If you are not listening right now, you miss the message. Gone forever. |
| **No queue** | Messages are not stored anywhere. |
| **Everyone gets it** | Every subscriber on that channel gets every message. |
| **Order is kept** | Messages on one channel arrive in the order they were sent. |

> Redis PubSub is **not** a message queue. Use Kafka or RabbitMQ if you need messages to be saved.

### Code Example

```ts
import Redis from "ioredis";

const pub = new Redis();
const sub = new Redis(); // must be a separate connection

sub.subscribe("news");

sub.on("message", (channel, message) => {
  console.log(`Got from ${channel}: ${message}`);
});

pub.publish("news", "hello");
```

> **Important:** once a Redis connection starts listening, it can **only** listen. You cannot run `GET` or `SET` on it. So you always need two connections — one to publish, one to subscribe.

---

## 3. The Socket.IO Redis Adapter

The adapter is a plugin for Socket.IO. You install it, give it two Redis connections, and it connects all your servers together.

You do not write any PubSub code. The adapter does it for you.

### Setup

```ts
import { Server } from "socket.io";
import { createClient } from "redis";
import { createAdapter } from "@socket.io/redis-adapter";

const pubClient = createClient({ url: "redis://localhost:6379" });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

const io = new Server({
  adapter: createAdapter(pubClient, subClient)
});

io.listen(3000);
```

That is it. Now `io.emit()` reaches users on **all** your servers.

### What It Does Behind The Scenes

When you call `io.to("room1").emit("message", data)` on Server 1:

1. Server 1 packs the event into a small binary message.
2. Server 1 publishes it to a Redis channel named `socket.io#/#room1#`.
3. The other servers are listening, so they receive it.
4. Each server checks: *do I have users in `room1`?* If yes, it sends to them.

So the adapter **is** just PubSub. It only hides it from you.

---

## 4. How They Relate

| | Redis PubSub | Socket.IO Adapter |
|---|---|---|
| What it is | A basic Redis feature | A Socket.IO plugin |
| What it does | Sends text to a channel | Keeps your servers in sync |
| Who writes the code | You | The adapter |
| Uses PubSub? | It **is** PubSub | It **uses** PubSub inside |

Short version: **the adapter is built on top of PubSub.** They are not two competing options. One sits on the other.

---

## 5. What The Adapter Hides From You

This is the real value of the adapter: **it makes many servers feel like one server.**

With raw PubSub you think about channels, publishers, listeners, and turning objects into strings. That is distributed systems work.

With the adapter you just write normal Socket.IO code:

```ts
io.to("room1").emit("msg", data);   // 1 server or 50 servers — same line
```

The API never changes. You never see a channel name. You never call `PUBLISH`.

### But the illusion is not perfect

Five places where you still feel the other servers:

**1. Sticky sessions are still required.**
Your load balancer must keep each user on the same server. If you skip this, you get HTTP 400 errors.

**2. If Redis goes down, nothing throws an error.**
Your app keeps running and looks healthy. But messages only reach users on the current server. You quietly go back to the broken behaviour from [section 1](#1-the-problem). Monitor your Redis connection — the adapter will not warn you.

**3. Your own memory does not sync.**
The adapter syncs Socket.IO events. It does not sync your variables.

```ts
const activeUsers = new Map();   // ❌ only exists on THIS server
```

Shared state belongs in Redis as normal keys, not in a local `Map`.

**4. Some reads are local by default.**

```ts
io.sockets.sockets           // ❌ only sockets on this server
await io.fetchSockets()      // ✅ asks every server
```

**5. Connection state recovery does not work.**
The Socket.IO docs list it as unsupported with the Redis adapter.

### The way to think about it

The adapter lets you **write** code like there is one server. But you still have to **run** it like there are many.

It is similar to an ORM. It hides the SQL so you move faster, but you still need to know a database is there, or it will surprise you.

---

## 6. Who Gets The Message?

This is where people get confused. There are really **two different questions** here.

### Question 1 — Which users get the event?

You control this fully. Always have.

```ts
io.to("user:123").emit("hello");   // only user 123 sees it
socket.emit("hello");              // only this one socket sees it
```

No other user will ever see the event. This is not a problem.

### Question 2 — Which servers receive the packet?

This is a different question. It is about network traffic, not about users.

With the normal `createAdapter()`, the answer is: **all of them.**

```
Server 1 calls: io.to("room1").emit("msg")
        │
        ▼
   publish to: socket.io#/#room1#
        │
        ▼  Redis sends it to EVERY server
        │
        ├──→ Server 2: "do I have users in room1?" → YES → send
        ├──→ Server 3: "do I have users in room1?" → NO  → throw it away
        └──→ Server 1: gets its own message back   → send to local users
```

Two things to notice:

- **Redis does no filtering at all.** To Redis, `socket.io#/#room1#` is just a name. It does not know what a room is. The filtering happens on each server.
- **The sender gets its own message back.** That is normal. The adapter handles it and will not send twice.

**The cost:** if you have 20 servers and only 1 has users in that room, the other 19 still receive the packet and throw it away. That is wasted network traffic.

---

## 7. The Sharded Adapter

There is a second adapter that fixes the waste above: `createShardedAdapter()`.

It uses **sharded PubSub**, a Redis 7.0 feature. But the important part is not the sharding. The important part is that a server **only subscribes to a room while it actually has users in that room.**

So Redis only sends the packet to servers that need it.

### Setup

```ts
import { createShardedAdapter } from "@socket.io/redis-adapter";

const io = new Server({
  adapter: createShardedAdapter(pubClient, subClient, {
    subscriptionMode: "dynamic-private"
  })
});
```

### The `subscriptionMode` Option

| Value | Channels | What happens |
|---|---|---|
| `static` | 2 per namespace | Everyone gets everything (same waste as before) |
| `dynamic` *(default)* | 2 + 1 per public room | Only servers with users in that room get it |
| `dynamic-private` | 2 + 1 per room | Same, plus a channel per socket — good for 1-to-1 messages |

`dynamic-private` is basically the "one channel per user" idea, already built for you.

### Why Not Build It Myself?

You could make your own channel per user, like `user:123`. It works. But then you own all of this:

| Job | Doing it yourself | `dynamic-private` |
|---|---|---|
| Subscribe when user connects | You write it | Done for you |
| Unsubscribe when user leaves | You write it | Done for you |
| Clean up after a server crash | You write it | Done for you |
| Same user on phone + laptop | You handle it | Done for you |
| Sending to a room/group | You build it | Already there |

### What You Need

- Redis **7.0 or newer**
- If you use `ioredis`, set `shardedSubscribers: true` on the client
- **Sticky sessions are still required** (true for both adapters)

> Socket.IO now recommends the sharded adapter for new projects.

---

## 8. Using Raw PubSub Too

Even with the adapter, sometimes you still want plain PubSub. This happens when something **outside Socket.IO** needs to send a message to your users.

Examples:

- A background worker that finished a job
- A Python or Go service in your system
- A cron job sending a daily reminder
- An admin tool

These things are not Socket.IO servers. They cannot call `io.emit()`. But they can publish to Redis.

### The Flow

```
Background Worker
     │
     └─ PUBLISH app:notify '{"room":"room1","text":"order shipped"}'
                ↓
              Redis
                ↓
      Your Socket.IO server is listening
                │
                └─ io.to("room1").emit("notify", { text })
                          ↓
                   adapter takes over
                          ↓
                   all users in room1
```

### The Code

```ts
// three separate connections
const pubClient = createClient({ url: "redis://localhost:6379" });
const subClient = pubClient.duplicate();   // for the adapter
const myOwnSub  = pubClient.duplicate();   // for my own listening

await Promise.all([
  pubClient.connect(),
  subClient.connect(),
  myOwnSub.connect(),
]);

const io = new Server({
  adapter: createAdapter(pubClient, subClient)
});

// listen for messages from workers
await myOwnSub.subscribe("app:notify", (message) => {
  const { room, text } = JSON.parse(message);
  io.to(room).emit("notify", { text });  // adapter spreads it to all servers
});
```

> **Why a third connection?** The adapter owns `subClient`. Do not touch it. Always make a new one with `duplicate()` for your own listening.

### Easier Option

If the outside service is also Node.js, you do not need to write channels by hand. Use the official emitter:

```ts
import { Emitter } from "@socket.io/redis-emitter";

const emitter = new Emitter(redisClient);
emitter.to("room1").emit("notify", { text: "order shipped" });
```

It talks to the adapter directly. Use raw PubSub only when this does not fit (for example, a Python service).

---

## 9. Which One Should I Use?

```
Do you have more than one server?
    → NO  → You do not need Redis at all.
    → YES → Keep going.

Is your Redis version 7.0 or newer?
    → NO  → Use createAdapter(). Simple and fine for small clusters.
    → YES → Use createShardedAdapter(). Recommended.

Do you send a lot of 1-to-1 messages (DMs, notifications)?
    → YES → subscriptionMode: "dynamic-private"
    → NO  → subscriptionMode: "dynamic"  (the default)

Do outside services (workers, Python, cron) need to send to users?
    → Node.js service?     → use @socket.io/redis-emitter
    → Other language?      → use raw PubSub + your own listener
```

---

## 10. Key Takeaways

- **Redis PubSub** is simple: publish to a channel, everyone listening gets it. Nothing is saved.
- **The Socket.IO adapter** uses PubSub inside. You do not compete between them — one sits on the other.
- The adapter's real job is **making many servers feel like one**. You write `io.emit()` and forget the cluster exists.
- But the illusion leaks: sticky sessions, silent failure when Redis dies, and local memory that does not sync.
- Keep **two questions apart**: which *users* get the event (you always control this), and which *servers* receive the packet (depends on the adapter).
- `createAdapter()` sends every packet to every server. Each server filters by itself. Redis filters nothing.
- `createShardedAdapter()` only sends to servers that have users in that room. Needs Redis 7.0+.
- **Do not build your own per-user channels just to save traffic.** Use `dynamic-private`. It does the same thing and handles connect, disconnect, and crashes for you.
- Raw PubSub is still useful when a **non-Socket.IO service** needs to reach your users.
- Always use **separate Redis connections**. A listening connection cannot do anything else.
- **Sticky sessions are required** with any Redis adapter.
