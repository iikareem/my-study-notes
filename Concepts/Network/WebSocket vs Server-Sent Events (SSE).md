---
tags:
  - network
  - realtime
  - sse
  - websocket
---

# WebSocket vs Server-Sent Events (SSE)

---

## Part 1 — WebSocket

### What Is It?

WebSocket is a **full-duplex, persistent communication protocol** built on top of a single TCP connection. After an initial HTTP handshake that "upgrades" the connection, both the client and server can send messages to each other freely and simultaneously — at any time, without waiting for the other side.

---

### How It Works

```
Client                          Server
  |                                |
  |--- HTTP Upgrade Request ------>|   (1) Handshake via HTTP/1.1
  |<-- 101 Switching Protocols ----|   (2) Server agrees
  |                                |
  |<====== WS Frame (server) =====>|   (3) Full-duplex channel open
  |======= WS Frame (client) =====>|       Both sides can send freely
  |<====== WS Frame (server) =====>|
  |                                |
  |--- Close Frame --------------->|   (4) Either side closes
  |<-- Close Frame ----------------|
```

**Handshake headers (simplified):**

```http
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

---

### Key Characteristics

|Property|Detail|
|---|---|
|**Protocol**|`ws://` or `wss://` (TLS)|
|**Direction**|Bidirectional (full-duplex)|
|**Transport**|TCP (single persistent connection)|
|**Framing**|Binary frames with opcodes|
|**HTTP version**|Requires HTTP/1.1 for upgrade|
|**Persistent**|Yes — connection stays open|
|**Message types**|Text, Binary, Ping/Pong, Close|

---

### Connection Lifecycle

1. **Opening Handshake** — Client sends HTTP `Upgrade` request; server responds with `101 Switching Protocols`.
2. **Data Transfer** — Both parties send framed messages freely.
3. **Heartbeating** — Either side may send `Ping` frames; the other must reply with `Pong`.
4. **Closing Handshake** — Either side sends a `Close` frame; the other echoes it; the TCP connection is torn down.

---

### Message Framing

WebSocket messages are split into **frames** with a binary header:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |                               |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
```

**Opcodes:**

- `0x0` — Continuation frame
- `0x1` — Text frame
- `0x2` — Binary frame
- `0x8` — Close
- `0x9` — Ping
- `0xA` — Pong

---

### Browser API

```javascript
// Connect
const ws = new WebSocket('wss://example.com/socket');

// Events
ws.onopen    = () => console.log('Connected');
ws.onmessage = (event) => console.log('Received:', event.data);
ws.onerror   = (err)   => console.error('Error:', err);
ws.onclose   = (event) => console.log('Closed:', event.code, event.reason);

// Send — text or binary
ws.send('Hello server!');
ws.send(new Uint8Array([1, 2, 3]));  // binary

// Close
ws.close(1000, 'Normal closure');
```

---

### Server Example (Node.js — `ws` library)

```javascript
const { WebSocketServer } = require('ws');
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  console.log('Client connected:', req.socket.remoteAddress);

  ws.on('message', (data) => {
    console.log('Received:', data.toString());
    ws.send(`Echo: ${data}`);           // Reply to sender
    wss.clients.forEach(client => {     // Broadcast to all
      if (client.readyState === ws.OPEN) client.send(data.toString());
    });
  });

  ws.on('close', () => console.log('Client disconnected'));
  ws.on('error', (err) => console.error('WS error:', err));
});
```

---

### Reconnection Strategy

WebSocket has **no built-in auto-reconnect**. You must implement it manually:

```javascript
function createWebSocket(url) {
  const ws = new WebSocket(url);
  let retryDelay = 1000;

  ws.onclose = () => {
    console.log(`Reconnecting in ${retryDelay}ms...`);
    setTimeout(() => createWebSocket(url), retryDelay);
    retryDelay = Math.min(retryDelay * 2, 30000); // exponential backoff, cap at 30s
  };

  ws.onopen = () => { retryDelay = 1000; }; // reset on success
  return ws;
}
```

---

### Sub-Protocols

WebSocket supports negotiated sub-protocols (e.g., `graphql-ws`, `stomp`, `mqtt`):

```javascript
const ws = new WebSocket('wss://example.com', ['graphql-ws', 'graphql-transport-ws']);
console.log(ws.protocol); // server-chosen protocol after handshake
```

---

### Use Cases

- Real-time chat applications
- Multiplayer games
- Collaborative editing (Google Docs-style)
- Live trading / financial dashboards
- IoT device control and telemetry
- Live audio/video signaling (WebRTC signaling layer)

---

### Pros and Cons

|✅ Pros|❌ Cons|
|---|---|
|True bidirectional communication|More complex to implement and debug|
|Low latency (no HTTP overhead per message)|Not plain HTTP — proxies/firewalls may block `ws://`|
|Supports binary data natively|No built-in reconnection|
|Works for high-frequency messaging|Stateful — harder to scale horizontally|
|Supports sub-protocols|HTTP/2 multiplexing not used (requires HTTP/1.1 upgrade)|

---

---

## Part 2 — Server-Sent Events (SSE)

### What Is It?

SSE is a **unidirectional, server-to-client streaming mechanism** built entirely on top of standard HTTP. The client opens a single long-lived HTTP connection, and the server pushes a stream of text events down it — indefinitely. The client **cannot send data back** over the same connection.

---

### How It Works

```
Client                          Server
  |                                |
  |--- GET /events --------------->|   (1) Standard HTTP request
  |   Accept: text/event-stream    |
  |                                |
  |<-- 200 OK ---------------------|   (2) Server opens streaming response
  |   Content-Type: text/event-stream
  |   Cache-Control: no-cache      |
  |                                |
  |<-- data: {"temp": 22} 

 ----|   (3) Server pushes events
  |<-- data: {"temp": 23} 

 ----|       whenever it wants
  |<-- data: {"temp": 24} 

 ----|
  |                                |
  |   (connection dropped)         |   (4) Browser auto-reconnects
  |--- GET /events  -------------->|       with Last-Event-ID header
  |   Last-Event-ID: 42            |
```

---

### Event Format (Wire Format)

SSE messages are **plain text** following a simple line-based format:

```
field: value

  (blank line = end of event)
```

**Fields:**

|Field|Description|
|---|---|
|`data:`|The event payload (required). Multi-line: repeat `data:`|
|`event:`|Custom event type name (optional; defaults to `"message"`)|
|`id:`|Event ID — browser stores as `lastEventId`|
|`retry:`|Reconnect delay in milliseconds|

**Example stream:**

```
retry: 3000

id: 1
event: temperature
data: {"value": 22, "unit": "C"}

id: 2
data: Simple string message

id: 3
event: alert
data: {"level": "high"}
data: {"extra": "multi-line data merges"}

```

---

### Key Characteristics

|Property|Detail|
|---|---|
|**Protocol**|Plain HTTP (`http://` / `https://`)|
|**Direction**|Unidirectional (server → client only)|
|**Transport**|HTTP/1.1 or HTTP/2|
|**Format**|UTF-8 text only|
|**Persistent**|Yes — long-lived HTTP response|
|**Reconnect**|Built-in (automatic by browser)|
|**Message types**|Named events + data payloads|

---

### Browser API

```javascript
// Connect
const source = new EventSource('https://example.com/events');

// Default "message" event
source.onmessage = (event) => {
  console.log('Data:', event.data);
  console.log('Last ID:', event.lastEventId);
};

// Named custom events
source.addEventListener('temperature', (event) => {
  const reading = JSON.parse(event.data);
  console.log('Temp:', reading.value);
});

source.addEventListener('alert', (event) => {
  console.log('Alert:', event.data);
});

// Error handling
source.onerror = (err) => {
  if (source.readyState === EventSource.CLOSED) {
    console.log('Connection closed');
  }
};

// Close manually
source.close();
```

**Sending data back** — SSE is receive-only, so use a separate `fetch` or `XMLHttpRequest`:

```javascript
async function sendToServer(payload) {
  await fetch('/actions', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  });
}
```

---

### Server Example (Node.js — plain HTTP)

```javascript
const http = require('http');

http.createServer((req, res) => {
  if (req.url !== '/events') return res.end();

  res.writeHead(200, {
    'Content-Type':  'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection':    'keep-alive',
    'Access-Control-Allow-Origin': '*',
  });

  const lastId = req.headers['last-event-id'];
  console.log('Client reconnected from ID:', lastId);

  let id = parseInt(lastId) || 0;

  const interval = setInterval(() => {
    id++;
    res.write(`id: ${id}
`);
    res.write(`data: ${JSON.stringify({ time: Date.now(), id })}

`);
  }, 1000);

  // Send a named event
  res.write(`event: connected
data: {"status":"ok"}

`);

  req.on('close', () => {
    clearInterval(interval);
    console.log('Client disconnected');
  });
}).listen(3000);
```

---

### Automatic Reconnection

SSE reconnection is **built into the browser** — nothing to implement:

- On disconnect, the browser waits `retry` ms (default ~3 seconds) then reconnects.
- It automatically sends `Last-Event-ID` header so the server can resume from where it left off.
- The server controls the retry delay: `retry: 5000

` sets it to 5 seconds.

```javascript
// Server: set custom retry interval
res.write('retry: 10000

');  // client will wait 10s before reconnecting
```

---

### HTTP/2 Multiplexing Advantage

Under HTTP/2, SSE connections share a single TCP connection with other requests — eliminating the browser's old 6-connections-per-domain limit that constrained HTTP/1.1 SSE to a maximum of 6 tabs.

```
HTTP/2 single TCP connection
├── Stream 1: GET /api/data
├── Stream 3: POST /api/submit
├── Stream 5: GET /events   ← SSE stream, no extra TCP overhead
└── Stream 7: GET /images/logo.png
```

---

### Use Cases

- Live notifications and alerts
- Real-time dashboards (metrics, logs, analytics)
- Stock price / sports score feeds
- AI/LLM streaming responses (ChatGPT-style token-by-token output)
- Progress bars for long-running server jobs
- News/social feed updates

---

### Pros and Cons

|✅ Pros|❌ Cons|
|---|---|
|Dead simple — plain HTTP, no protocol upgrade|Server → client only (no bidirectional)|
|Built-in auto-reconnect with `Last-Event-ID`|Text-only (no native binary support)|
|Works through HTTP proxies and firewalls|Client must use separate requests to send data|
|Native browser `EventSource` API|HTTP/1.1: max 6 connections per domain (tabs problem)|
|HTTP/2 multiplexing for free|Not suitable for high-frequency bidirectional messaging|
|Easy to debug (plain text stream)||
|Works with standard HTTP caching, auth, and CORS||

---

---

## Part 3 — Head-to-Head Comparison

### At a Glance

|Feature|WebSocket|SSE|
|---|---|---|
|**Protocol**|`ws://` / `wss://` (custom)|`http://` / `https://` (standard)|
|**Direction**|Bidirectional (full-duplex)|Unidirectional (server → client)|
|**Data format**|Text **and** Binary|Text only (UTF-8)|
|**Auto-reconnect**|❌ Manual|✅ Built-in|
|**Event IDs / resume**|❌ Manual|✅ Built-in (`Last-Event-ID`)|
|**HTTP/2 support**|❌ Requires HTTP/1.1 upgrade|✅ Native|
|**Firewall/proxy friendly**|⚠️ Sometimes blocked|✅ Always works|
|**Browser support**|All modern browsers|All modern browsers (not IE)|
|**Connection overhead**|Low (after handshake)|Low (standard HTTP)|
|**Server complexity**|Higher|Lower|
|**Scaling**|Harder (stateful)|Easier (stateless-friendly)|
|**Named event types**|Not native (DIY in payload)|✅ Built-in (`event:` field)|

---

### Communication Pattern

```
WebSocket (bidirectional):
  Client ←——————————→ Server
         send / receive

SSE (unidirectional):
  Client ←——————————  Server    (server pushes)
  Client ——————————→  Server    (client uses regular HTTP POST/fetch)
```

---

### When to Choose Which

#### Choose WebSocket when:

- The client **needs to send data frequently** (chat, gaming, collaborative editing)
- You need **low-latency bidirectional messaging** (sub-100ms reactions)
- You're sending **binary data** (audio, video frames, files)
- You're building **real-time multiplayer** or control systems

#### Choose SSE when:

- Communication is **mostly or entirely server-to-client** (notifications, feeds, AI streaming)
- You want **zero reconnection logic** — let the browser handle it
- You're behind **corporate proxies or strict firewalls** that block WebSocket
- You want **simpler server architecture** (stateless REST + streaming)
- You're already on **HTTP/2** and want multiplexing for free
- You're streaming **LLM responses** or log tails

---

### Scalability Comparison

**WebSocket:**

- Each connection is **stateful** — the server must remember which socket belongs to which user.
- Horizontal scaling requires a **message broker** (Redis Pub/Sub, Kafka) to route messages between nodes.
- Load balancers need **sticky sessions** or WebSocket-aware routing.

```
Browser ─── Node 1 ─── Redis Pub/Sub ─── Node 2 ─── Browser
                  (message broker required)
```

**SSE:**

- Each response is just a **long HTTP response** — much friendlier to stateless architectures.
- Works naturally with **reverse proxies** (nginx, Caddy) if buffering is disabled.
- Reconnects land on any node — just replay from the event store using `Last-Event-ID`.

```
Browser ─── Any Node ─── Event Store (DB/Kafka)
          (reconnect anywhere)
```

---

### Performance Characteristics

|Metric|WebSocket|SSE|
|---|---|---|
|**Latency**|~1–5ms after handshake|~5–15ms (HTTP overhead)|
|**Throughput**|Very high (binary frames)|High (text-optimized)|
|**Connection cost**|Low (after handshake)|Low (HTTP keep-alive)|
|**Message overhead per frame**|2–14 bytes header|~20–40 bytes (text lines)|
|**CPU per message**|Very low (binary framing)|Low (text parsing)|

---

### Security Considerations

|Concern|WebSocket|SSE|
|---|---|---|
|**TLS**|Use `wss://`|Use `https://`|
|**CORS**|Not enforced by browser (manual handling needed)|Enforced by browser|
|**Auth**|Tokens in query param or first message (no custom headers in browser API)|Standard HTTP headers (cookies, `Authorization`)|
|**CSRF**|Must verify `Origin` header on server|Standard CSRF protections apply|

---

### Quick Decision Flowchart

```
Does the client need to send data frequently?
├── YES ──→ WebSocket
└── NO
    │
    Do you need binary data?
    ├── YES ──→ WebSocket
    └── NO
        │
        Are you on HTTP/2 or behind strict proxies?
        ├── YES ──→ SSE
        └── NO
            │
            Do you want auto-reconnect without code?
            ├── YES ──→ SSE
            └── Either works — pick WebSocket for future flexibility
```

---

### Summary Table

|Dimension|WebSocket|SSE|Winner|
|---|---|---|---|
|Bidirectional|✅|❌|WebSocket|
|Simplicity|Complex|Simple|**SSE**|
|Reconnect handling|Manual|Automatic|**SSE**|
|Binary support|✅|❌|WebSocket|
|Firewall compatibility|⚠️|✅|**SSE**|
|HTTP/2 friendly|❌|✅|**SSE**|
|Latency|Lower|Slightly higher|WebSocket|
|Scaling|Harder|Easier|**SSE**|
|Auth/CORS|Trickier|Standard|**SSE**|
|Best for|Chat, games, collab|Feeds, AI streaming, alerts|**Context-dependent**|

---

> **Bottom line:** Use **WebSocket** when you need a true two-way channel. Use **SSE** when the server is doing the talking — it's simpler, more robust across networks, and handles reconnection for free.