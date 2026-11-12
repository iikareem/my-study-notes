# What Happens When You Type `https://google.com`

> One of the most common senior engineering interview questions — and one of the best windows into how the modern web actually works.

---

## Introduction

You open your browser, type `https://google.com`, and hit Enter. A fraction of a second later, Google's homepage appears. It feels instant — but underneath that single keystroke is a precisely choreographed sequence involving your operating system, global DNS infrastructure, TCP/IP networking, cryptographic handshakes, and a web server on the other side of the planet.

Understanding this flow end-to-end isn't just interview prep. It's the foundation for reasoning about **latency**, **security**, and **performance** in any web system you build. Every optimization — CDNs, HTTP/2, TLS 1.3, DNS prefetching — only makes sense once you understand the bottleneck it's solving.

Let's walk through it, step by step.

---

## Step 1 — Browser Cache Check

Before anything hits the network, your browser tries to avoid the network entirely.

It checks its **own DNS cache** first in browser, then falls back to the **OS-level DNS cache** (on Linux/Mac, this includes `/etc/hosts`). If the IP address for `google.com` is already known and still valid, DNS resolution is skipped completely — saving potentially 50–200ms before a single packet is sent.

**Lesson:** Caching is the first line of defense against latency. The best network request is the one you never make.

---

	Here's the enhanced **Step 2** with everything mandatory to know:

---

## Step 2 — DNS Resolution

If the address isn't cached, the browser needs to translate the human-readable hostname `google.com` into a machine-routable IP address. This is the job of **DNS (Domain Name System)** — essentially the phone book of the internet.

### The Resolution Chain

```
Browser
  → Recursive Resolver       (your ISP or a public resolver like 8.8.8.8)
    → Root DNS Server        (knows where .com lives)
      → TLD Server for .com  (knows where google.com lives)
        → Google's Authoritative DNS Server
          → returns 142.250.80.46
```

### Who Does What

|Component|Role|
|---|---|
|**Recursive Resolver**|Does the work on your behalf — queries the chain, caches the result. Your ISP provides one by default, or you can use `8.8.8.8` (Google) or `1.1.1.1` (Cloudflare)|
|**Root DNS Server**|Doesn't know the IP — just knows _who to ask next_ for `.com`, `.org`, `.io`, etc. There are only 13 root server clusters in the world|
|**TLD Server**|Knows which authoritative server is responsible for `google.com` specifically|
|**Authoritative DNS Server**|The final authority — actually returns the IP address. Google owns and controls this|

### Key Concepts to Know

**TTL (Time To Live)** Every DNS record has a TTL — a number in seconds that tells resolvers how long to cache the result. When TTL expires, the resolver must query again.

- Google sets _short_ TTLs (~300s) so they can reroute traffic or failover quickly during incidents
- A small blog might set TTL to 86400 (24 hours) — changes take a full day to propagate

**Anycast Routing** Google's authoritative DNS uses a single IP address globally, but that IP is advertised from dozens of datacenters simultaneously. Your query is automatically routed to the _nearest_ one by the internet's routing protocol (BGP). Same IP, closest server — that's Anycast.

**DNS Record Types** (you will be asked this)

|Record|Purpose|Example|
|---|---|---|
|`A`|Maps hostname → IPv4 address|`google.com → 142.250.80.46`|
|`AAAA`|Maps hostname → IPv6 address|`google.com → 2607:f8b0::...`|
|`CNAME`|Alias — points to another hostname|`www.google.com → google.com`|
|`MX`|Mail server for the domain|Used by Gmail routing|
|`NS`|Which servers are authoritative for this domain|Points to Google's DNS servers|
|`TXT`|Arbitrary text — used for domain verification, SPF, etc.|Proving you own a domain|

### Latency Numbers

```
Cold lookup:    50–200ms   (full chain fires)
Cached:         ~0ms       (recursive resolver has it)
```

---

The three things interviewers actually want to hear: the **resolution chain** (browser → recursive → root → TLD → authoritative), what **TTL** controls, and what **Anycast** means for why DNS is fast globally.
---

## Step 3 — TCP Handshake

Now the browser has an IP address. To send data reliably, it needs to establish a **TCP connection** — a virtual circuit that guarantees ordered, error-checked delivery.

The TCP three-way handshake:

```
Client  →  SYN          →  Server    "I want to connect"
Client  ←  SYN-ACK      ←  Server    "Acknowledged, I'm ready"
Client  →  ACK          →  Server    "Great, we're connected"
```

This costs **1 full round trip (1 RTT)** before a single byte of application data can be sent. At a typical intercontinental latency of 100ms, that's 100ms of pure overhead just to shake hands.

**Lesson:** TCP is reliable but not free. This is why connection reuse (HTTP keep-alive) and multiplexing (HTTP/2) matter so much.

---

## Step 4 — TLS Handshake

Because the URL uses `https://`, the connection must be **encrypted before any HTTP data flows**. This is handled by TLS (Transport Layer Security).

### TLS 1.2 (older)

```
Client  →  ClientHello                        (supported ciphers, TLS version)
Server  ←  ServerHello + Certificate + Done
Client  →  Verifies cert against CA, sends encrypted pre-master secret
Server  →  Both sides derive session keys → Encrypted channel open
```

**Cost: 2 RTTs** before the first HTTP byte.

### TLS 1.3 (modern — what Google uses)

```
Client  →  ClientHello + key share            (proactively sends key exchange data)
Server  ←  ServerHello + key share + encrypted certificate + Finished
           → Encrypted channel open
```

**Cost: 1 RTT** — half the overhead of TLS 1.2.

TLS 1.3 also supports **0-RTT resumption**: returning clients can send application data in their very first message, with zero round-trip overhead for the handshake. This is a massive win for repeat visitors.

> **Summary of latency saved:** TLS 1.3 cuts handshake cost by 50% vs TLS 1.2, and 0-RTT eliminates it entirely for known clients.

---

## Step 5 — HTTP Request

With an encrypted channel established, the browser sends the actual request:

```http
GET / HTTP/2
Host: google.com
Accept: text/html
Accept-Encoding: gzip, br
```

### HTTP Version Comparison

| Version      | Key Behavior                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **HTTP/1.1** | One request per connection at a time. Browsers open ~6 parallel connections to compensate.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **HTTP/2**   | **Multiplexing** — many requests share one connection simultaneously, fixes HOL at the _application layer_ only — it multiplexes streams over one TCP connection, so one slow _HTTP request_ doesn't block others. But underneath, it still runs over **TCP**, which guarantees ordered delivery. So if a **single TCP packet** is dropped, _all streams_ on that connection are frozen waiting for retransmission. That's TCP-level HOL blocking, and HTTP/2 actually makes it _worse_ in some ways — you went from 6 connections (6 independent TCP streams) down to 1, so one packet drop now freezes everything instead of just one connection. |
| **HTTP/3**   | Built on **QUIC** (UDP-based). Eliminates TCP-level head-of-line blocking entirely. Faster on lossy networks.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

Google serves over HTTP/2 and increasingly HTTP/3, so your request benefits from multiplexing out of the box.

---

## Step 6 — Server Processing

Google's infrastructure receives your request at a **load balancer**, which routes it to an available backend server. That server orchestrates calls to various internal services — search index, personalization, ads, etc. — assembles a response, and sends back HTML.

For most apps this step is simpler, but the principle holds: the server does work, then responds.

---

## Step 7 — Response & Rendering

The browser receives the HTML and immediately starts parsing it. This triggers additional requests for CSS, JavaScript, fonts, and images — all of which can now be made in parallel over the same HTTP/2 connection.

The **critical rendering path**:

```
HTML → DOM → CSSOM → Render Tree → Layout → Paint → Display
```

Every render-blocking resource (especially `<script>` tags without `async`/`defer`) delays this path. Modern performance engineering is largely about shortening and parallelizing this chain.

---

## Performance Bottlenecks & How the Web Solves Them

|Bottleneck|Problem|Solution|
|---|---|---|
|DNS lookup|50–200ms per cold lookup|DNS caching, `<link rel="dns-prefetch">`|
|TCP handshake|1 RTT overhead|Connection reuse (keep-alive), HTTP/2 multiplexing|
|TLS handshake|1–2 RTT overhead|TLS 1.3 (1 RTT), 0-RTT session resumption|
|Physical distance|Speed of light is a hard limit|CDNs — serve from the nearest edge node|
|Many resources|Each asset = latency|HTTP/2 multiplexing, inlining critical CSS|
|Large payloads|Slow transfers|Compression (gzip, Brotli), minification|
|Head-of-line blocking|One slow packet stalls everything|HTTP/3 / QUIC (UDP, no HOL blocking)|

---

## Key Latency Numbers

```
DNS lookup (cold):       20–120ms
DNS lookup (cached):     ~0ms
TCP handshake:           1 RTT   (~50–150ms)
TLS 1.2 handshake:       2 RTT   (~100–300ms)
TLS 1.3 handshake:       1 RTT   (~50–150ms)
TLS 1.3 0-RTT:           0 RTT   (returning clients only)

Total cold connection overhead:  ~200–500ms before first HTTP byte
```

These numbers are why CDNs, connection reuse, and TLS 1.3 adoption have such measurable real-world impact.

---

## The Full Picture

```
[ You press Enter ]
        ↓
[ Browser cache check ]  ← Skip DNS if cached
        ↓
[ DNS Resolution ]        ← ~20–200ms
        ↓
[ TCP Handshake ]         ← 1 RTT
        ↓
[ TLS Handshake ]         ← 1–2 RTT (1 RTT with TLS 1.3)
        ↓
[ HTTP Request sent ]
        ↓
[ Server processes ]
        ↓
[ HTML received → Render ]
```

---

## What to Remember for Interviews

- The four core layers: **DNS → TCP (1 RTT) → TLS (1–2 RTT) → HTTP**
- **TLS 1.3** cuts the handshake to 1 RTT; **0-RTT** eliminates it for returning clients
- **HTTP/2 multiplexing** solves HTTP-level head-of-line blocking
- **HTTP/3 / QUIC** solves TCP-level head-of-line blocking (runs over UDP)
- **CDNs** solve the physical distance problem — speed of light is the floor
- **DNS caching** is the first and cheapest optimization
- Know the bottleneck table — interviewers love "what would you fix and why"



Good answer overall — you clearly understand the flow. Let me give you precise feedback on each part.

---

### ✅ What You Got Right

- **DNS cache layers** — browser first, then OS level. Correct.
- **TTL expiry** triggers a new DNS lookup. Correct.
- **DNS resolution chain** — you got the concept right (recursive resolver → root → TLD → authoritative).
- **Results get cached** at browser and OS level with TTL. Correct.
- **Anycast** — nearest geographic server. Correct.
- **TCP handshake = 1 RTT** — SYN, SYN-ACK, ACK. Correct.
- **TLS has two versions** — 1.2 (older, more round trips) and 1.3 (fewer). Correct.
- **HTTP request → server processing → response**. Correct.

---

### ⚠️ Small Things to Fix

**1. DNS chain naming was slightly off**

You said _"record supervisor"_ and _"iterative DNS server"_ — the correct terms are:

```
Recursive Resolver → Root DNS Server → TLD Server → Authoritative DNS Server
```

Just clean up those names. The concept was right, just the vocabulary slipped.

**2. TCP handshake purpose**

You described it as _"both sides can reach each other"_ — that's partially right, but the precise purpose is: establishing a **reliable, ordered connection** before data flows. The handshake confirms both sides are reachable _and_ ready to communicate reliably.

**3. TLS — you skipped what it actually does**

You said TLS _"takes responsibility for encrypting the data"_ which is correct, but you missed one important thing at your level of depth: TLS also **verifies the server's identity** via a certificate. Encryption + identity verification — both matter. Without the certificate check, you could be encrypted but talking to the wrong server.

---

### Verdict

You have a solid mental model of the full flow. For the level you're targeting — knowing _which layer handles what_ without deep diving — you're about **85% there**. Fix the DNS naming and add the certificate point to TLS and your answer is clean.