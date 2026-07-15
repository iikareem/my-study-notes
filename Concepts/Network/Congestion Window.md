---
tags:
  - network
  - tcp
---

# Prerequisites of the Congestion Window: Flow Control vs. Congestion Control

Before you can truly master the **Congestion Window (**$cwnd$**)**, you have to understand the fundamental network problem it was designed to solve.

To do that, we must take a step back and look at how computers talk to each other over a network. We will cover the three core prerequisites:

1. **The TCP Buffer & Sliding Window**
    
2. **Flow Control (The Receiver's Window:** $rwnd$**)**
    
3. **The Bufferbloat / Congestion Problem (The Network's Limit)**
    

## 1. The TCP Buffer & Sliding Window

TCP (Transmission Control Protocol) is a **reliable, byte-stream protocol**. "Reliable" means that if computer A sends bytes to computer B, TCP guarantees that every single byte arrives in the exact same order, with zero losses.

To guarantee this, computers cannot just throw data into the network wire and forget about it. They must use memory.

### The Buffer

Both the Sender and the Receiver have a dedicated chunk of RAM called a **Buffer**:

- **Sender Buffer:** Holds data that has been sent but not yet acknowledged (ACKed) by the receiver, in case it needs to be retransmitted.
    
- **Receiver Buffer:** Holds data that has arrived from the network but hasn't been read by the application (e.g., Node.js or a database) yet.
    

### The Sliding Window

Instead of sending a single packet, waiting for an ACK, and then sending the next one (which would be incredibly slow), TCP uses a **Sliding Window**. This allows the sender to transmit multiple packets at once without waiting for individual acknowledgments.

The "Window" is simply the range of sequence numbers of packets that the sender is allowed to transmit before it must stop and wait for an ACK.

## 2. Flow Control: The Receiver's Window ($rwnd$)

The first prerequisite to the Congestion Window is **Flow Control**.

Flow control is a 1-to-1 relationship. It prevents a fast sender from overwhelming a slow receiver.

Imagine a massive server sending a 10GB file to an old, slow mobile phone. The server can push data at 1 Gbps, but the mobile phone's CPU can only process and write data to disk at 10 Mbps. If the server sends data at full speed, the phone's physical receiver buffer will overflow, and packets will be dropped.

To prevent this, TCP uses the **Receiver's Window (**$rwnd$**)**:

1. The receiver continuously tells the sender how much free space is left in its receiver buffer. This value is sent inside the TCP header of every ACK packet.
    
2. The sender is strictly forbidden from sending more unacknowledged bytes than the size of this $rwnd$.
    

$$\text{Bytes in Flight} \le rwnd$$

If the phone's buffer fills up, it advertises an $rwnd = 0$. The sender instantly stops transmitting and waits.

## 3. The Gap: Why Flow Control Wasn't Enough

In the mid-1980s, the internet suffered from "Congestion Collapse." The throughput of the entire network would suddenly drop to near-zero.

The problem was that **Flow Control (**$rwnd$**) only looks at the two endpoints**. It knows absolutely nothing about the routers, switches, and cables in between them.

```
+------------+      1 Gbps      +----------+      10 Mbps      +------------+
|   Sender   | ===============> |  Router  | -------------->   |  Receiver  |
+------------+                  +----------+                   +------------+
  (Fast CPU)                     (Bottleneck)                   (Empty Buffer)
                                                                 Advertises 
                                                                 rwnd = Large!

```

In the diagram above:

- The Receiver is fast and has an empty buffer. It tells the Sender: _"My_ $rwnd$ _is huge! Send me everything you've got!"_
    
- The Sender starts blasting data at 1 Gbps.
    
- However, the physical router in the middle can only handle 10 Mbps.
    

The router's internal queue overflows, packets get dropped, the sender retransmits, and the router gets even more congested. The network collapses.

## 4. The Solution: Enter the Congestion Window ($cwnd$)

To solve this, Van Jacobson invented **Congestion Control** in 1988, introducing the **Congestion Window (**$cwnd$**)**.

While $rwnd$ represents the limits of the **receiver's memory**, $cwnd$ represents the limits of the **network path's bandwidth**.

The sender must now respect **both** limits. To do this, TCP dictates that the actual maximum amount of data the sender can transmit at any given time (the Transmission Window) is the **minimum** of these two windows:

$$\text{Transmission Window} = \min(cwnd, rwnd)$$

```
                    +------------------------------------+
                    |        Which Limit Wins?           |
                    +------------------------------------+
                                      |
                 Is the network or the receiver slower?
                                      |
                     +----------------+----------------+
                     |                                 |
              [ cwnd < rwnd ]                   [ rwnd < cwnd ]
                     |                                 |
         The Network is the bottleneck.    The Receiver is the bottleneck.
         Rate limited by Congestion.       Rate limited by Flow Control.

```

## 5. Summary: From Network to Server Concurrency

When we relate this back to our discussion about application rate limiting:

|Concept|Network Layer (TCP)|Application Layer (Web Server)|
|---|---|---|
|**Flow Control (**$rwnd$**)**|Prevents the sender from overwhelming the receiver's memory buffer.|Prevents upstream services from overwhelming a downstream database's memory/queue limit (e.g., backpressure).|
|**Congestion Control (**$cwnd$**)**|Prevents endpoints from overwhelming shared network routers.|Prevents global incoming traffic from overwhelming the server's CPU / Event Loop.|

To calculate how much traffic we can safely process, we must always look at the lower of the two limits!

CLAUDE

Good thinking. Understanding them at the network level first will make your Node implementation feel obvious rather than arbitrary. Let me build this up properly.

---

## The Problem Both Are Solving

Before anything else you need to understand **network congestion** because that is the exact problem AIMD and Vegas were invented to solve. Everything else builds on this.

Imagine a highway. Cars are packets. The road is the network. When too many cars enter the highway at once, traffic jams form. Packets start arriving late or not at all. This is congestion.

The question is: **how does a sender know the network is congested, and how should it react?**

This is called **congestion control** and it lives inside TCP.

---

## One Prerequisite — TCP and The Congestion Window

TCP does not just fire all packets at once. It maintains a number called the **congestion window (cwnd)**. This is how many packets the sender is allowed to have in flight simultaneously — sent but not yet acknowledged.

Think of it as a budget. If cwnd is 10, you can send 10 packets and then you must wait for acknowledgements before sending more.

The entire game of congestion control is: **how do you grow and shrink this window intelligently?**

Grow it too fast → you congest the network. Shrink it too aggressively → you underuse the available bandwidth.

Both AIMD and Vegas are strategies for managing this window. That is their entire job.

---

## AIMD — Additive Increase Multiplicative Decrease

### The core idea

AIMD is **reactive**. It does not try to predict congestion. It waits for a signal that congestion has already happened, then reacts to it.

The signal it listens for is **packet loss**. In the original TCP design, if a packet is lost it means the network is congested — a router somewhere got overwhelmed, its buffer filled up, and it started dropping packets. Loss equals congestion. That is the assumption AIMD is built on.

### How it behaves

**When things are going well — additive increase:**

Every round trip time where no loss occurs, cwnd grows by 1. Slow, steady, linear growth. You are probing the network, carefully feeling for how much bandwidth is available.

```
cwnd: 10 → 11 → 12 → 13 → 14...
```

**When loss is detected — multiplicative decrease:**

The moment a packet loss is detected, cwnd is cut in half immediately. Aggressive, fast, no hesitation.

```
cwnd: 20 → loss detected → 10
```

Then additive increase starts again from that new lower point.

### Why the asymmetry

Increase is slow because you do not want to overshoot. You are sharing the network with other connections and you want to find the ceiling gently without slamming into it.

Decrease is fast because when congestion happens you need to back off immediately. Every millisecond you stay at the high rate makes the congestion worse for everyone on the network.

### The sawtooth pattern

If you draw cwnd over time with AIMD you get a sawtooth wave. Slowly climbing, then suddenly dropping, then slowly climbing again, over and over.

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

AIMD only reacts after congestion has already happened. Packets had to be dropped before it knew there was a problem. By the time it reacts, the damage is done — latency has spiked, buffers have filled, other connections on the network have suffered. It is like driving with your eyes closed and only knowing you hit a wall after the crash.

---

## Vegas — Seeing Congestion Before It Happens

### The prerequisite — RTT and bandwidth

To understand Vegas you need two concepts:

**RTT (Round Trip Time)** — how long it takes to send a packet and get an acknowledgement back. This is your latency measurement. When the network is uncongested, RTT is at its natural minimum. When congestion builds, RTT starts climbing because packets are waiting in router buffers.

**Throughput** — how much data you are actually moving per second. Calculated as:

```
throughput = cwnd / RTT
```

More packets in flight and lower RTT means more throughput.

### The core insight Vegas is built on

Vegas noticed something AIMD ignored completely: **RTT starts climbing before packets are lost.**

When a router's buffer starts filling up, packets have to wait in that buffer before being forwarded. That waiting time shows up as increased RTT. The buffer is not full yet — no packets are being dropped yet — but the delay is already growing.

This is an early warning signal. Vegas listens to it. AIMD ignores it entirely.

### How Vegas measures congestion

Vegas continuously tracks two things:

**BaseRTT** — the minimum RTT it has ever observed for this connection. This represents the network at its cleanest, with no buffering delay. Your healthy baseline.

**CurrentRTT** — the actual RTT being measured right now.

When CurrentRTT starts climbing above BaseRTT, Vegas interprets that gap as evidence that packets are sitting in a router buffer waiting. The bigger the gap, the more congestion is building.

It then calculates an **expected throughput** versus **actual throughput**:

```
expected = cwnd / BaseRTT   ← what you should be getting if network is clean
actual   = cwnd / CurrentRTT ← what you are actually getting right now
diff     = expected - actual
```

If diff is small → network is clean, gently increase cwnd. If diff is large → congestion is building, gently decrease cwnd.

### How it behaves

Both increase and decrease are gentle and continuous. No sawtooth. No sudden halvings. Vegas is constantly making small adjustments, always trying to sit just below the congestion point without ever crossing it.

```
cwnd
 |          ___________
 |         /           \___________
 |        /
 |       /
 |______/
 |________________________ time
```

Smooth. Stable. Proactive.

### The fundamental weakness

Vegas works beautifully when it is the only connection on the network. But if it shares a network with AIMD connections, it loses badly.

AIMD is aggressive — it keeps pushing its window up until packets drop. Vegas backs off early when it detects rising RTT. So AIMD connections grab more bandwidth, Vegas connections politely step aside, and Vegas ends up starved. It is too well behaved for a competitive environment.

This is why Vegas never replaced AIMD at the TCP level despite being technically smarter. The internet is a shared environment full of aggressive senders.

---

## Side by Side

||AIMD|Vegas|
|---|---|---|
|Signal used|Packet loss|RTT increase|
|Reacts|After congestion|Before congestion|
|Increase style|Slow linear|Slow continuous|
|Decrease style|Sudden halving|Gentle reduction|
|Pattern|Sawtooth|Smooth|
|Weakness|Reactive, damage already done|Loses to aggressive senders|

---

## Why This Matters For Your Node Package

Now the translation becomes clear.

In your package, there is no network and no packets. But the analogy maps directly:

- **cwnd** → your concurrency limit (how many requests in flight)
- **Packet loss** → CPU spike, event loop delay, memory pressure (AIMD signal)
- **RTT increase** → response latency climbing above baseline (Vegas signal)
- **BaseRTT** → your minimum observed response latency
- **CurrentRTT** → your current rolling average response latency

AIMD in your package means: wait until something breaks (event loop delays, CPU spikes), then cut concurrency hard and rebuild slowly.

Vegas in your package means: watch response latency continuously, and when it starts climbing above your healthy baseline, gently reduce concurrency before anything actually breaks.

The reason Vegas fits Node better is exactly the same reason it was smarter at the network level — it catches the problem earlier, through a signal that is already available for free inside every request, without waiting for something to actually fail.

---

← Part of [[TCP & Connection Lifecycle]]
