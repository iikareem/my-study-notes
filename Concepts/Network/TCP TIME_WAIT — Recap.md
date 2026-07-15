---
tags:
  - network
  - tcp
---

---

## 1. Opening a Connection: 3-Way Handshake

```
Client → SYN       → Server
Client ← SYN + ACK ← Server
Client → ACK       → Server
```

Connection is alive.

---

## 2. Closing a Connection: 4-Way Handshake

```
Client → FIN   "I want to close"
Server ← ACK   "Noted"
Server → FIN   "I'm ready too"
Client ← ACK   "Got it" ← client sends this, then enters TIME_WAIT
```

- **Step 1: The Client Initiates the Breakup**
    
    - **[Client Side]:** Sends a **FIN** packet and waits.
        
    - **[Network]:** `[Client] ────────── FIN ──────────> [Server]`
        
    - **[Server Side]:** Receives the FIN packet.
        
- **Step 2: The Server Acknowledges**
    
    - **[Server Side]:** Sends an **ACK** packet to say "I hear you, let me finish processing."
        
    - **[Network]:** `[Client] <────────── ACK ────────── [Server]`
        
    - **[Client Side]:** Receives the ACK and waits for the server to finish up.
        
- **Step 3: The Server Says Goodbye Too**
    
    - **[Server Side]:** Finishes its tasks, then sends its own **FIN** packet.
        
    - **[Network]:** `[Client] <────────── FIN ────────── [Server]`
        
    - **[Client Side]:** Receives the server's FIN packet.
        
- **Step 4: The Final Handshake**
    
    - **[Client Side]:** Sends the final **ACK** packet, **then immediately enters TIME_WAIT**.
        
    - **[Network]:** `[Client] ────────── ACK ──────────> [Server]`
        
    - **[Server Side]:** Receives the final ACK and **instantly CLOSES** the connection permanently.
        
- **Step 5: The Ghost Phase (The Wait)**
    
    - **[Client Side]:** Sits silently in the **TIME_WAIT** state for a few minutes.
        
    - **[Server Side]:** Already completely dead/closed.
        
- **Step 6: The Final Wipe**
    
    - **[Client Side]:** The timer expires, and the client finally **CLOSES** and deletes the connection.

The side that sends FIN first = **Active Closer** = pays the penalty.

---

## 3. Why TIME_WAIT Exists

The final ACK might get lost. If the server never receives it, it retransmits its FIN. The client must still be alive to respond — otherwise the server hangs forever in `LAST_ACK`.

So the client waits **2 × MSL (~60s)** before releasing the port.

Second reason: guarantees all stale packets from the dead connection expire before the port is reused — preventing old packets from poisoning a new connection on the same 4-tuple.

---

## 4. The Exhaustion Problem

Every outgoing connection consumes one **ephemeral port** from a finite OS pool (~28,000–64,500 ports). Ports are locked in TIME_WAIT for ~60s before recycling.

```
500 req/sec × 60s TIME_WAIT = 30,000 ports stuck
Default pool   = ~28,000 ports
Result         = EADDRNOTAVAIL — new connections fail
```

Your app breaks. The external API is perfectly fine.

---

## 5. The Asymmetry (Your Key Insight)

**The Active Closer pays everything. The Passive Closer pays nothing.**

```
Your Backend (Active Closer)     External API (Passive Closer)
────────────────────────────     ─────────────────────────────
TIME_WAIT ~60s per connection    Hits CLOSED immediately
Port pool draining               Zero residual state
Heading toward outage            Status page: green
```

If your backend initiates thousands of short-lived calls and closes them — you choke. They don't even notice.

---

## 6. The Fixes

|Fix|How|Best For|
|---|---|---|
|**Connection Pooling**|Reuse connections, never close them|Always — this is the real fix|
|`tcp_tw_reuse = 1`|Kernel reuses TIME_WAIT ports safely for outgoing conns|Linux tuning, outgoing traffic only|
|`SO_REUSEADDR`|App can rebind a port still in TIME_WAIT|Server restarts, not client exhaustion|
|Widen port range|`ip_local_port_range = 1024 65535`|Band-aid, buys time|
|**Let server close first**|Server sends FIN first → TIME_WAIT shifts to them|When you control both sides|

---

## 7. The One-Line Mental Model

> TIME_WAIT is the client keeping the connection's ghost alive just long enough in case the server's goodbye got lost and it needs to say it again.

---

## Key Terms

|Term|Meaning|
|---|---|
|**4-tuple**|(src IP, src port, dst IP, dst port) — uniquely identifies a connection|
|**Ephemeral port**|Temporary source port the OS picks for outgoing connections|
|**MSL**|Maximum Segment Lifetime — max time a packet survives in the network (~60s)|
|**TIME_WAIT**|State held by Active Closer for 2×MSL after sending final ACK|
|**Active Closer**|The side that sends FIN first — always enters TIME_WAIT|
|**Passive Closer**|The side that receives FIN first — hits CLOSED immediately, pays nothing|
|**EADDRNOTAVAIL**|The error you see when ephemeral ports are exhausted|

**MSL (Maximum Segment Lifetime)** is the absolute maximum time a packet is legally allowed to survive in the network before it is destroyed. By universal agreement, 1 MSL is set to **60 seconds**.

Yes, absolutely! A single network connection can absolutely handle multiple requests. In fact, modern internet traffic relies heavily on this exact capability to make web browsing fast and efficient.

However, **how** a connection handles multiple requests depends entirely on the protocol being used.

## 1. The Old Way: HTTP/1.0 (One Request Per Connection)

In the early days of the web, a connection could _not_ handle multiple requests.

- Your browser would open a TCP connection, ask for `index.html`, get the file, and the connection would immediately close.
    
- If that HTML file had 10 images, your browser had to open and close **10 separate TCP connections** to get them. This was incredibly slow and wasteful.
    

## 2. The Step Forward: HTTP/1.1 (Keep-Alive / Persistent Connections)

To fix this, HTTP/1.1 introduced **Persistent Connections** (often called `Keep-Alive`).

- Instead of closing the connection after one request, the browser keeps it open.
    
- It sends a request for an image, waits for the response, and then uses that **same connection** to send the next request.
    

> ⚠️ **The Catch:** While it uses the same connection, it has to do so **in a line (serially)**. Request B cannot start until Request A is completely finished. If Request A gets stuck, everything behind it is blocked. This is known as **Head-of-Line (HoL) Blocking**.

## 3. The Modern Way: HTTP/2 and HTTP/3 (Multiplexing)

This is where connections became truly powerful. Modern web protocols use a technique called **Multiplexing**.

Instead of waiting in line, **multiple requests and responses are broken down into tiny pieces (frames) and sent down the exact same connection at the same time.**

- **How it works:** Your browser can request `image1.png`, `style.css`, and `script.js` all at once over a single connection. The server interleaves the pieces of these files together and sends them back. Your browser then reassembles them on the fly.
    
- **The Benefit:** No more waiting in line. One single TCP (or UDP in HTTP/3) connection can download dozens of files simultaneously.
    

## How the OS Tells Requests Apart

Going back to your previous question about ports: even if a single connection handles 50 requests at once via HTTP/2, your computer still only uses **one ephemeral port** for that entire connection.

The protocol inside the connection attaches a unique **Stream ID** to each request and response. Your browser looks at the Stream ID to know exactly which image or script a piece of data belongs to.

---

← Part of [[TCP & Connection Lifecycle]]
