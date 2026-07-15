---
tags:
  - observability
  - rate-limiting
  - sliding-window
  - token-bucket
---

Here's a clear breakdown of the most important rate limiting algorithms you need to know:Here's the full breakdown in markdown:

![[rate_limiting_algorithms 1.png]]
---

# Rate Limiting Algorithms — The 4 You Must Know

---

## 1. Token Bucket

**How it works:** A bucket holds tokens. Tokens refill at a fixed rate (e.g. 1 per second, up to a max capacity). Each request consumes 1 token. No tokens → request rejected.

**Pros:**

- Handles bursts naturally — tokens accumulate when traffic is low, so a user who was quiet can fire a quick burst without being penalized. This matches how real users behave.
- Very memory efficient — you only store 2 values per user: current token count and last refill timestamp.
- Easy to tune — change the refill rate to control average throughput, change bucket size to control burst tolerance independently.
- Battle-tested — AWS, Stripe, and most major APIs use this. Redis even has native support for it.

**Cons:**

- Two parameters to tune (rate + capacity) instead of one, which can be confusing when setting limits.
- At the exact same moment two requests can race to consume the last token, so in distributed systems you need atomic operations (e.g. Redis Lua scripts) to avoid double-spending tokens.
- The Token Bucket limits _how many_ requests are processed over a period of time, but it does not control _when_ they are processed. It permits 'micro-bursts' where multiple requests hit the server simultaneously, which can cause sudden spikes in CPU and memory usage

**Use case:** Public APIs where you want to give users headroom for occasional spikes without letting them hammer you indefinitely.

---

## 2. Fixed Window Counter

**How it works:** Time is divided into fixed windows (e.g. every 60 seconds). A counter increments with each request and resets to 0 when the window flips. If the counter hits the limit → reject.

**Pros:**

- Simplest algorithm to implement — literally one counter and one timestamp per user.
- Lowest memory footprint of all four algorithms.
- Easy to reason about: "you get 100 requests per minute, the minute starts at :00" — users understand this.
- Fast to check — a single Redis INCR + EXPIRY is all you need.

**Cons:**

- The boundary exploit is a real and well-known attack. A client can send 100 requests at 0:59 and 100 more at 1:01, firing 200 requests in 2 seconds while never technically breaking the limit. For anything sensitive, this is a serious gap.
- The window reset is abrupt — a user who hits the limit at 0:30 has to wait until 1:00 even though 30 seconds of quota could reasonably have refreshed by then.
- Does not protect against short sharp spikes within a window — 100 requests in the first second of a 60-second window all pass through.

**Use case:** Internal dashboards, low-risk endpoints, admin tooling — anywhere the boundary exploit isn't a real threat and simplicity wins.

---

## 3. Sliding Window Log

**How it works:** Every request's timestamp is stored in a log. On each new request, all timestamps older than the window size (e.g. 60 seconds) are pruned, then the remaining count is checked against the limit. No fixed reset point — the window moves with time.

**Pros:**

- The most accurate algorithm. There is no edge to exploit — the count at any given millisecond reflects exactly the last 60 seconds of activity.
- No boundary spike problem at all. A user can't game a reset point because there is no reset point.
- Fair to users — quota is consumed and released smoothly as time passes, not in chunks.

**Cons:**

- Memory is the big problem. You store one log entry per request, per user. At scale (millions of users, hundreds of requests each), this gets expensive fast.
- Pruning old timestamps on every request adds computation. Under high traffic this overhead adds up.
- Hard to pre-announce limits to users in a clean way ("you have 37 requests left in the next 60 seconds" requires real-time calculation rather than a simple counter).

**Use case:** Sensitive or high-value endpoints where accuracy matters most — login attempts, payment submissions, password resets. Worth the memory cost when correctness is critical.

---

## 4. Leaky Bucket

**How it works:** Requests enter a queue (the "bucket"). A worker processes them at a fixed constant rate (e.g. 1 per second). If the queue is full when a new request arrives → it is dropped immediately. Output is always steady regardless of how bursty the input is.

**Pros:**

- Guarantees smooth, predictable output. Downstream services always receive traffic at a constant rate, never spiked. This is the only algorithm that offers this guarantee.
- Protects slow downstream systems (databases, payment processors, third-party APIs) from being overwhelmed even if your own traffic spikes.
- Queue size is the only thing you need to store per client — very memory efficient.

**Cons:**

- Bursts are not absorbed, they are either queued (adding latency) or dropped. Legitimate users who send a burst of valid requests can have those requests silently dropped, which leads to a bad experience.
- The queuing adds latency even for requests that do get processed. A request that enters a full-ish queue may wait seconds before being handled.
- Queue management adds operational complexity — you need to decide queue size carefully. Too small and you drop too much. Too large and you just delay the overload problem.
- Not suitable when your use case needs to respond to bursts — e.g. a user uploading 10 images at once expects them all to process, not to have 7 of them dropped.

**Use case:** Network traffic shaping, outbound API call throttling to a third party (e.g. you can only call a payment processor 2 times/sec — you queue your calls and drain them at that rate), video streaming output buffering.

---

## Summary: Pros & Cons at a Glance

|Algorithm|Biggest Pro|Biggest Con|
|---|---|---|
|Token bucket|Absorbs bursts, memory-efficient|Needs atomic ops in distributed systems|
|Fixed window|Dead simple, near-zero cost|Boundary exploit doubles effective limit|
|Sliding window|Perfectly accurate, no exploits|Expensive memory at scale|
|Leaky bucket|Smooth guaranteed output|Drops or delays legitimate burst traffic|

---

## The Decision in One Rule

If you don't have a strong reason to pick otherwise, start with **token bucket** — it handles real-world traffic patterns well, scales easily, and is what most engineers expect when they hear "rate limiting." Switch to sliding window when accuracy is non-negotiable, and leaky bucket when you're protecting a slow downstream system.

## Sliding Window Log

**How it works:**

Every time a request comes in, you save its timestamp. On the next request, you first throw away any timestamps older than your window (e.g. 60 seconds ago), then count what's left. If the count is at the limit → reject. If not → allow and save this new timestamp too.

There is no fixed reset. The window is always "the last 60 seconds from right now," and it moves forward with every passing moment.

---

**Concrete example:**

Your limit is 3 requests per 60 seconds. Here is what the log looks like over time:

- 0:10 → request arrives. Log: `[0:10]`. Count = 1. ✅ Allow.
- 0:20 → request arrives. Log: `[0:10, 0:20]`. Count = 2. ✅ Allow.
- 0:50 → request arrives. Log: `[0:10, 0:20, 0:50]`. Count = 3. ✅ Allow.
- 0:55 → request arrives. Prune anything before 0:55 - 60s = nothing to prune yet. Log: `[0:10, 0:20, 0:50]`. Count = 3. ❌ Reject.
- 1:11 → request arrives. Prune anything before 1:11 - 60s = 0:11. The 0:10 entry is now expired and gets removed. Log: `[0:20, 0:50]`. Count = 2. ✅ Allow.

Notice what happened at 1:11 — quota freed up not because a clock hit :00 and reset everything, but because an old entry naturally aged out. That's the core mechanic.

---

**Why it has no boundary exploit:**

In fixed window, a user can abuse the reset point — fire 100 requests just before midnight, then 100 more just after, because the counter wiped clean. In sliding window there is no midnight. At any moment in time, the window is looking back exactly 60 seconds and counting. There is no seam to exploit.

---

**Why memory is the real cost:**

Every allowed request leaves a timestamp behind in storage (usually Redis sorted sets). If you have 10,000 users each making 500 requests per minute, you are storing 5,000,000 live entries at any given time, all needing to be pruned and counted on every request. Compare that to token bucket which stores just 2 numbers per user regardless of how many requests they make.

---

**The one-line mental model:**

> It's a conveyor belt — only what's on the belt right now (last 60 seconds) counts, and items fall off the back end as time moves forward.

---

**Your instinct is correct:**

Leaky bucket is almost never used to rate limit users directly. It is used to control what you send _out_ to another service. The other three are typically used to control what comes _in_ from users.

That's a clean mental split:

- Token bucket, fixed window, sliding window → **protecting your server from your users.**
- Leaky bucket → **protecting a downstream service from your server.**

---

**How to choose:**

**Use fixed window when** simplicity is the priority and the stakes are low. Internal tools, admin panels, non-critical endpoints. You know the boundary exploit exists and you're okay with it because nobody is trying to game your internal dashboard. It's the fastest to implement and the cheapest to run.

**Use token bucket when** you are building a public API and want to be fair to real users. Real users are bursty by nature — they click a button 5 times quickly, then go quiet. Token bucket rewards that natural behavior instead of punishing it. This is the right default for most API rate limiting.

**Use sliding window when** the endpoint is sensitive and accuracy is non-negotiable. Login attempts, password resets, OTP verification, payment submissions. You cannot afford someone exploiting the boundary to double their attempts on a login form. The memory cost is worth it because the traffic on these endpoints is naturally low.

**Use leaky bucket when** you are calling a third party that has strict rate limits on their side. A payment processor, an SMS provider, an external API you don't control. You absorb whatever traffic your users generate and release it downstream at a safe, controlled pace.

---

**A real world scenario to tie it together:**

Imagine you are building a fintech app.

- Your login endpoint → **sliding window.** Too sensitive to allow any exploit.
- Your general API (fetch account balance, view transactions) → **token bucket.** Users browse naturally, occasional bursts are fine.
- Your internal analytics dashboard → **fixed window.** Low risk, keep it simple.
- Your outbound calls to the payment processor → **leaky bucket.** They allow 2 calls per second, so you queue everything and drain at that rate regardless of how many users are checking out simultaneously.

---

**The one rule to remember:**

If you are thinking about what a user sends to you → token bucket is your safe default. If you are thinking about what you send to someone else → leaky bucket is your tool.
