---
tags:
  - caching
  - distributed-systems
  - thundering-herd
---

## **The Thundering Herd Problem — When Recovery Kills You Faster Than the Outage Did**

Most engineers design for failure. Far fewer design for **recovery** — and that's exactly where this problem lives. Your system goes down, comes back up, and then immediately gets crushed by its own users trying to reconnect all at once.

---

## Start from zero

Imagine a popular restaurant that closes unexpectedly for 30 minutes.

When it reopens, every single person who was waiting outside rushes in at the exact same moment. The kitchen, which was just fine handling a steady stream of customers, is now overwhelmed by a simultaneous flood. Service collapses again — not because of the original problem, but because of the recovery.

That's the thundering herd.

In software: your server goes down or your cache expires, and every client that was waiting **retries at the same moment**, sending a simultaneous spike of requests to a system that is already in a fragile state.

---

## The two main scenarios where this happens

### Scenario 1: Cache expiration

You have a expensive database query — say, loading a product catalog — and you cache the result for 60 minutes.

```
[Request] --> [Cache] --> hit? serve it. miss? --> [Database query] --> cache it
```

When the cache expires, the **next request** triggers a fresh DB query. Fine. But in a high-traffic system, you don't get _one_next request. You get **thousands simultaneously** — all finding an empty cache, all deciding to go to the database, all at the same moment.

Your database just absorbed 10,000 simultaneous expensive queries that it normally only sees once per hour. This is called a **cache stampede.**

### Scenario 2: Server restart / reconnection

Your service goes down for 2 minutes. You have 50,000 connected clients — mobile apps, websocket connections, background workers — all with retry logic.

The retry logic in most systems looks like:

```javascript
// Typical naive retry
async function connect() {
  while (true) {
    try {
      await establishConnection();
      break;
    } catch {
      await sleep(5000); // wait 5 seconds, try again
    }
  }
}
```

Every client failed at roughly the same time. Every client waits exactly 5 seconds. Every client retries at **exactly the same moment.** Your server, which just recovered, gets hit by 50,000 simultaneous connection attempts and falls over again.

---

## Why naive retry logic is the villain

Fixed retry intervals are the core of the problem. When all clients share the same interval, they stay synchronized forever — even after the initial spike, they'll continue retrying in waves.

```
t=0    Server goes down. All clients fail.
t=5    All 50,000 clients retry simultaneously. Server falls again.
t=10   All 50,000 clients retry simultaneously. Server falls again.
t=15   ...
```

The system never recovers because it never gets a chance to breathe between waves.

---

## The fixes

### Fix 1: Jitter

Add randomness to your retry interval. Instead of every client waiting exactly 5 seconds, each client waits somewhere between 0 and 10 seconds — randomly.

```javascript
async function connect() {
  let attempt = 0;
  while (true) {
    try {
      await establishConnection();
      break;
    } catch {
      const jitter = Math.random() * 5000; // random 0-5 seconds
      await sleep(5000 + jitter);
    }
  }
}
```

Now 50,000 clients are spread across a 5 second window instead of hitting simultaneously. Your server sees a manageable ramp instead of a spike.

This sounds almost too simple. It is simple — and it works remarkably well. AWS explicitly documented jitter as a core reliability pattern after studying retry behavior across their infrastructure.

### Fix 2: Exponential backoff with jitter

Don't just add randomness — also increase the wait time with each failed attempt. The longer the outage, the more spread out the retries become.

```javascript
async function connect() {
  let attempt = 0;
  while (true) {
    try {
      await establishConnection();
      break;
    } catch {
      attempt++;
      const base = Math.min(30000, 1000 * 2 ** attempt); // caps at 30s
      const jitter = Math.random() * base;
      await sleep(jitter); // "full jitter" strategy
    }
  }
}
```

This is the industry standard pattern. Most serious client libraries implement this by default — but many home-rolled retry loops don't.

### Fix 3: Cache stampede — the mutex / lock solution

For the cache expiration problem specifically, the fix is to ensure only **one request** triggers the recomputation while others wait.

```javascript
async function getCatalog() {
  const cached = await cache.get('catalog');
  if (cached) return cached;

  // Only one request should recompute
  const lock = await acquireLock('catalog_recompute');
  if (!lock) {
    // Someone else is recomputing — wait and read cache
    await sleep(100);
    return cache.get('catalog');
  }

  try {
    const data = await db.query('SELECT * FROM catalog');
    await cache.set('catalog', data, { ttl: 3600 });
    return data;
  } finally {
    await releaseLock('catalog_recompute');
  }
}
```

Only one request does the expensive work. Everyone else waits briefly and reads the freshly populated cache.

### Fix 4: Probabilistic early expiration

A more elegant cache solution — instead of waiting for the cache to fully expire, **randomly decide to refresh it early** as it approaches expiration. The probability of refreshing increases as the TTL runs out.

This means the cache gets quietly refreshed in the background before it ever actually expires, so no request ever hits a cold cache. It's used in systems like Redis at scale.

---

## The deeper lesson

The thundering herd is really about **synchronization emerging accidentally.** You never designed your clients to coordinate — but they end up perfectly synchronized anyway, because they all experienced the same event at the same time and all have the same retry logic.

The fix in every case is to **deliberately introduce randomness** to break that accidental synchronization.

Any time you have many clients or processes that can all fail simultaneously and retry on a fixed schedule — you have a latent thundering herd waiting for the first outage to trigger it.
