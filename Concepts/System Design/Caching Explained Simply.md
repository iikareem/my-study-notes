# Caching, Explained Simply: Layers, Invalidation, Stampedes, and Fixes

> Caching is amazing for **reads**, but it hides a trap: the moment your data **changes**, the cache can lie to you. This guide explains the layers, the patterns, invalidation, and how to survive cache stampedes.

---

## Table of Contents

1. [What is caching?](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#1-what-is-caching)
2. [Why caching is great for reads (and the catch)](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#2-why-caching-is-great-for-reads-and-the-catch)
3. [The caching layers](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#3-the-caching-layers)
4. [Caching patterns: which one do I use?](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#4-caching-patterns-which-one-do-i-use)
5. [Cache invalidation](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#5-cache-invalidation)
6. [The cache stampede problem](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#6-the-cache-stampede-problem)
7. [Solutions to stampedes](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#7-solutions-to-stampedes)
8. [Other common caching challenges](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#8-other-common-caching-challenges)
9. [Decision cheat sheet](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#9-decision-cheat-sheet)
10. [Final checklist](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#10-final-checklist)

---

## 1. What is caching?

A **cache** is a fast, temporary copy of data that is slow or expensive to get from its original source.

- **Source of truth**: the database, an external API, a heavy computation.
- **Cache**: memory (or something close to the user) that stores the result so the next request is quick.

```
Without cache:  User -> App -> Database (50 ms)   -> App -> User
With cache:     User -> App -> Cache    (1 ms)    -> App -> User
```

Two words you must know:

|Term|Meaning|
|---|---|
|**Cache hit**|The data was found in the cache. Fast.|
|**Cache miss**|The data was not in the cache. You go to the source, then (usually) store it in the cache.|

**Hit ratio** = hits / (hits + misses). A higher ratio means the cache is helping more.

---

## 2. Why caching is great for reads (and the catch)

Caching is amazing for data that is **read often and changes rarely**: product pages, user profiles, config, rendered HTML, API responses.

But it comes with trade-offs that people often forget:

|Benefit|Hidden cost|
|---|---|
|Much faster responses|Data can become **stale** (out of date)|
|Less load on the database|Extra system to run and monitor|
|Handles traffic spikes|**Stampedes** when popular entries expire|
|Cheaper infrastructure|**Invalidation** is hard to get right|

> A famous joke: _"There are only two hard things in computer science: cache invalidation and naming things."_ This guide is mostly about the first one.

---

## 3. The caching layers

A request can pass through **many** caches. Each layer is closer to the user or closer to the data.

```
 User
  |
  v
[1] Browser cache         (user's device)
  |
  v
[2] CDN / Edge cache      (servers around the world)
  |
  v
[3] Reverse proxy cache   (Nginx, Varnish)
  |
  v
[4] Application cache     (in-process memory)
  |
  v
[5] Distributed cache     (Redis, Memcached)
  |
  v
[6] Database cache        (buffer pool, query cache)
  |
  v
 Disk / Source of truth
```

### Layer 1: Browser cache

- **Where**: the user's browser.
- **Stores**: images, CSS, JS, API responses.
- **Controlled by**: HTTP headers like `Cache-Control`, `ETag`, `Last-Modified`.
- **Pros**: zero network cost on a hit.
- **Cons**: you **cannot clear it** from the server. Once a user has it, you wait for it to expire.
- **Tip**: use **versioned file names** (`app.3f9a1c.js`) so a new deploy gets a new URL.

### Layer 2: CDN / edge cache

- **Where**: servers near users (Cloudflare, CloudFront, Fastly).
- **Stores**: static files, and sometimes whole pages or API responses.
- **Pros**: huge latency win, protects your origin servers.
- **Cons**: purging can take time and can be costly; careful with personalized content.

### Layer 3: Reverse proxy cache

- **Where**: in front of your app (Nginx, Varnish).
- **Stores**: full HTTP responses.
- **Pros**: no app code needed; supports request collapsing and stale serving.

### Layer 4: Application (in-process) cache

- **Where**: inside your app's memory (a dictionary, Caffeine, `lru_cache`).
- **Pros**: fastest possible (no network).
- **Cons**: each server instance has **its own copy**. They can disagree. Lost on restart.
- **Good for**: small, rarely-changing data (config, feature flags, lookup tables).

### Layer 5: Distributed cache

- **Where**: a separate service (Redis, Memcached) shared by all app servers.
- **Pros**: one shared copy, so consistent across servers; survives app restarts.
- **Cons**: network hop (still fast), another service to operate.
- **Good for**: sessions, hot queries, computed results, rate limits.

### Layer 6: Database cache

- **Where**: inside the database (buffer pool, page cache).
- **Pros**: automatic.
- **Note**: you mostly tune it rather than control it directly.

### Which layer should I use?

|Question|Answer|
|---|---|
|Same for every user and rarely changes?|CDN / browser|
|Needs to be shared across servers?|Redis / Memcached|
|Tiny, super hot, OK if slightly different per server?|In-process|
|Personalized per user?|App or distributed cache (usually **not** CDN)|

> **Rule of thumb:** cache as close to the user as is **safe**. Each layer you add is one more place where stale data can hide.

---

## 4. Caching patterns: which one do I use?

A **caching pattern** answers two simple questions:

1. **READ:** when someone asks for data, who checks the cache, and who goes to the database on a miss?
2. **WRITE:** when data changes, what gets updated first: the cache, the database, or both?

That is all a pattern is. Five patterns are just five different answers to those two questions.

### The running example used in this section

Imagine an online shop. We have a **database** (the shelves in the warehouse, slow to walk to) and a **cache** (a small desk at the front with the popular items, instant to grab). The data we care about is a **product price**.

```
        +-----------+        +---------+        +------------+
        |   Your    | -----> |  Cache  | -----> |  Database  |
        |   App     |        | (Redis) |        | (Postgres) |
        +-----------+        +---------+        +------------+
                              fast, small        slow, the truth
```

### The five patterns at a glance

| #   | Pattern           | One-sentence idea                                                | Real-life analogy                                                                                    |
| --- | ----------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| 1   | **Cache-aside**   | _You_ check the cache. On a miss, _you_ fetch and fill it.       | You check the fridge. If the milk is gone, **you** go to the store and put some in the fridge.       |
| 2   | **Read-through**  | You only ask the cache. The _cache_ fetches on a miss.           | You ask a **butler**. He checks the fridge and goes to the store himself if needed.                  |
| 3   | **Write-through** | Every write goes to the cache **and** DB, and you wait for both. | You buy milk, put it in the fridge, **and** write it in the shopping log before leaving the kitchen. |
| 4   | **Write-behind**  | Write to the cache now, save to the DB **later**.                | You scribble on a sticky note now and update the official log tonight.                               |
| 5   | **Write-around**  | Write straight to the DB and skip the cache.                     | You put new stock straight in the warehouse, not on the front desk.                                  |

Patterns 1 and 2 are about **reading**. Patterns 3, 4 and 5 are about **writing**. In a real system you pick **one read pattern + one write pattern** and combine them.

---

### 4.1 Cache-aside (a.k.a. lazy loading)

**The idea:** your application code is in charge. It talks to the cache **and** the database itself. The cache is just a dumb storage box.

**Analogy:** you check the fridge first. If the milk is there, great. If not, **you** drive to the store, buy it, and put a bottle in the fridge for next time.

**Reading, step by step:**

```
 App wants price of product 42
   |
   1. Ask the cache: "do you have product:42?"
   |
   +-- YES (hit)  --> return it. Done. (fast)
   |
   +-- NO (miss)
        2. App asks the DB: "price of product 42?"   (slow)
        3. App PUTS the answer into the cache (with a TTL)
        4. App returns the answer
```

```python
def get_price(product_id):
    key = f"price:{product_id}"

    cached = redis.get(key)                     # step 1
    if cached is not None:
        return float(cached)                    # hit, done

    price = db.query("SELECT price FROM products WHERE id = %s", product_id)  # step 2
    redis.set(key, price, ex=300)               # step 3 (TTL 5 min)
    return price                                # step 4
```

**Writing, step by step:**

```python
def update_price(product_id, new_price):
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)  # 1. DB first
    redis.delete(f"price:{product_id}")                                                # 2. then delete cache
```

We **delete** the cache key instead of updating it. The next read will miss and reload the fresh value from the DB.

**Walkthrough with real values:**

|Time|What happens|Cache|DB|
|---|---|---|---|
|10:00|Read price -> miss -> load from DB -> store|`price:42 = 100`|`100`|
|10:01|Read price -> **hit** (fast)|`100`|`100`|
|10:02|Admin sets price to 120: DB updated, cache key deleted|_(empty)_|`120`|
|10:03|Read price -> miss -> load `120` -> store|`120`|`120`|

**Pros**

- Simplest to understand and implement.
- Only caches data people **actually request** (no wasted memory).
- If the cache dies, the app still works (just slower), because it falls back to the DB.

**Cons**

- The **first** request for each item is always slow (a miss).
- Short stale window is possible between a DB change and a cache delete.
- You must write the "check cache, then DB, then fill" code yourself (usually in a helper function).

**Use it when:** you want a safe default for almost any read-heavy app. **If you are unsure, start here.**

**Avoid it when:** you cannot tolerate even one slow first request, and you cannot warm the cache beforehand.

---

### 4.2 Read-through

**The idea:** the application **only talks to the cache**. If the data is missing, the **cache itself** knows how to load it from the database. You give the cache a "loader" function once; after that it handles misses for you.

**Analogy:** you ask a **butler** for milk. You never look in the fridge or visit the store. If it is not in the fridge, the butler goes and gets it. You only ever talk to the butler.

**Reading, step by step:**

```
 App wants price of product 42
   |
   1. App asks the CACHE LAYER (and only the cache layer)
   |
   +-- HIT  --> cache returns it
   |
   +-- MISS
        2. The cache layer itself calls the loader -> DB
        3. The cache layer stores the result
        4. The cache layer returns it to the app
```

```python
# You configure the loader ONCE...
cache = ReadThroughCache(
    loader=lambda product_id: db.query(
        "SELECT price FROM products WHERE id = %s", product_id),
    ttl=300,
)

# ...and from then on the app just asks the cache. No if/else, no DB code.
price = cache.get(42)
```

**The only real difference from cache-aside:** _where the "on a miss, go to the DB and fill the cache" code lives._

||Who handles a miss?|
|---|---|
|Cache-aside|**Your application code**|
|Read-through|**The cache library / provider**|

**Pros**

- Cleaner app code: no repeated "check, load, store" blocks.
- One central place to add protection (like request collapsing for stampedes, see Section 7).

**Cons**

- Needs a cache library or provider that supports it (e.g., Caffeine `LoadingCache` in Java, Guava, some cloud caches). Plain Redis does **not** do this by itself.
- The cache layer must know how to talk to your database.

**Use it when:** many parts of your code read the same kind of data and you want one consistent loading path.

**Avoid it when:** your cache is plain Redis and you do not want to build a wrapper (use cache-aside instead).

> **Tip:** many teams just write a small helper function `get_or_load(key, loader)` around cache-aside. That is effectively read-through, built by hand.

---

### 4.3 Write-through

**The idea:** every time you **write**, the data goes into the **cache and the database together**, and the write only counts as "done" when **both** are saved. The cache is therefore always up to date.

**Analogy:** you buy milk. Before leaving the kitchen you put it in the fridge **and** update the shopping log. You do not walk away until both are done.

**Writing, step by step:**

```
 App wants to set price of product 42 to 120
   |
   1. Write to the cache    (price:42 = 120)
   2. Write to the database (price = 120)
   3. Both succeeded? --> tell the app "done"
```

```python
def update_price(product_id, new_price):
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)
    redis.set(f"price:{product_id}", new_price, ex=3600)   # cache updated at the same time
```

(In a strict write-through system, the cache layer does both writes for you and ensures they succeed together.)

**Walkthrough**

|Time|What happens|Cache|DB|
|---|---|---|---|
|10:00|Admin sets price to 120 -> written to **both**|`120`|`120`|
|10:01|Read price -> **hit**, value is already fresh|`120`|`120`|

Notice there is **no moment** where the cache holds the old price.

**Pros**

- Reads right after a write are **always fresh** (no stale window).
- Reads are fast (data is already in the cache).

**Cons**

- **Writes are slower** (two writes instead of one).
- You may cache data that **nobody ever reads** (wasted memory).
- If one of the two writes fails, you must handle it, or the cache and DB disagree.

**Use it when:** data is **read soon after it is written** and must be fresh (user settings, profile changes, shopping cart).

**Avoid it when:** writes are very frequent and reads are rare.

---

### 4.4 Write-behind (a.k.a. write-back)

**The idea:** write to the **cache only** and answer "done" immediately. The cache then saves the data to the database **later**, in the background (often in batches).

**Analogy:** you scribble the new milk count on a **sticky note** (instant). Tonight you copy all the notes into the official log. Fast for you now, but if you lose the sticky note before tonight, that information is gone.

**Writing, step by step:**

```
 App wants to increase page-view count for product 42
   |
   1. Write to the cache         (views:42 = 1001)   <-- instant
   2. Tell the app "done"                            <-- app continues right away
   ...
   3. LATER, in the background: a worker saves the new value (or a batch of values) to the DB
```

```python
def record_view(product_id):
    redis.incr(f"views:{product_id}")        # instant, no DB involved

# runs every 10 seconds in a background worker
def flush_views_to_db():
    for key in redis.scan_iter("views:*"):
        product_id = key.split(":")[1]
        count = redis.getdel(key)            # take and reset the counter
        if count:
            db.execute(
                "UPDATE products SET views = views + %s WHERE id = %s",
                int(count), product_id)
```

Instead of 10,000 tiny database writes per second, the DB gets **one batched write** every few seconds.

**Pros**

- **Extremely fast writes** (memory speed).
- **Reduces database load** by batching many writes into one.
- Absorbs sudden write spikes.

**Cons**

- **Risk of data loss:** if the cache crashes **before** the background save, those writes are gone.
- The database is **temporarily out of date** (anyone reading directly from the DB sees old data).
- More complex (background workers, retries, ordering).

**Use it when:** you have **very heavy writes** and can tolerate losing a few seconds of data: view counters, "likes", analytics events, game scores, activity logs.

**Avoid it when:** the data is **critical** (payments, orders, account balances). Never use it where losing a write is unacceptable.

---

### 4.5 Write-around

**The idea:** writes go **straight to the database** and **skip the cache entirely**. The cache is only filled later when someone **reads** the data (like cache-aside).

**Analogy:** new stock is delivered straight to the **warehouse**. The front desk only gets an item after a customer asks for it.

**Writing, step by step:**

```
 App wants to save a new order / big file / log entry
   |
   1. Write directly to the database   (cache is NOT touched)
   2. Done
   ...
   later: if someone READS it, it is loaded into the cache (cache-aside style)
```

```python
def save_report(report):
    db.execute("INSERT INTO reports (...) VALUES (...)", report)   # DB only
    # no redis.set(...) here, on purpose
```

**Why do this?** Some data is written **once** and **rarely read soon after** (logs, bulk imports, archives). If you cached everything you write, you would **fill the cache with junk** and push out the useful hot data.

**Pros**

- Keeps the cache clean: only data that is actually **read** gets cached.
- Fast, simple writes.

**Cons**

- The **first read** after a write is always a miss (slow).
- If an old version of this data **is** already cached, you must still invalidate it (see Section 5).

**Use it when:** data is written a lot but read rarely, such as logs, bulk imports, audit trails, large files.

**Avoid it when:** data is read immediately after being written (use write-through).

---

### 4.6 Putting it together: common real combinations

You rarely use a pattern alone. Here are the combos you will actually see:

|Combo|Read pattern|Write pattern|Good for|
|---|---|---|---|
|**The default**|Cache-aside|Update DB, **delete** cache key|Most web apps and APIs (product pages, profiles)|
|**Always-fresh data**|Cache-aside / read-through|Write-through|Settings, carts, data read right after write|
|**High-volume counters**|Cache-aside|Write-behind|Likes, views, analytics|
|**Write-heavy, rarely read**|Cache-aside|Write-around|Logs, imports, archives|

### 4.7 How to choose: a simple decision path

Answer these in order:

```
Q1. Is the data critical (money, orders)?
     YES --> never write-behind.  Use cache-aside + delete-on-write (or write-through).
     NO  --> continue

Q2. Is it written VERY often (thousands/sec) and OK to lose a few seconds?
     YES --> write-behind (with batching)
     NO  --> continue

Q3. Is it read right after it is written and must be fresh?
     YES --> write-through
     NO  --> continue

Q4. Is it written often but rarely read?
     YES --> write-around
     NO  --> cache-aside  (the safe default)
```

### 4.8 Pattern cheat sheet

|Pattern|Who loads on a miss?|Write speed|Freshness after write|Data-loss risk|Complexity|Best for|
|---|---|---|---|---|---|---|
|**Cache-aside**|Your app|Fast|Fresh after delete; short stale window possible|None|Low|General default|
|**Read-through**|The cache layer|n/a (read pattern)|Depends on write pattern|None|Low-Medium|Cleaner code, many readers|
|**Write-through**|n/a (write pattern)|Slower (2 writes)|Always fresh|None|Medium|Read-after-write data|
|**Write-behind**|n/a (write pattern)|**Fastest**|Cache fresh, DB lags|**Yes**|High|Counters, analytics|
|**Write-around**|n/a (write pattern)|Fast|First read is a miss|None|Low|Logs, bulk data|

> **Remember:** _Read patterns_ (cache-aside, read-through) decide **who fetches on a miss**. _Write patterns_ (write-through, write-behind, write-around) decide **what gets written first**. Pick one of each.

---

## 5. Cache invalidation

**Invalidation = making sure the cache stops serving data that is no longer true.**

There are four main strategies. They are not mutually exclusive: real systems **combine** them.

### 5.1 TTL (Time To Live) expiration

Each entry gets a lifetime. When time is up, it disappears.

```python
redis.set("product:42", data, ex=60)   # expires in 60 seconds
```

- **Pros**: dead simple; self-healing (even if you forget to invalidate, it eventually fixes itself).
- **Cons**: data can be stale **until the TTL ends**.
- **Pick the TTL by asking:** "How long can this data be wrong before it hurts?"

|Data|Suggested TTL|
|---|---|
|Stock price / live score|1 to 10 seconds|
|Product details|1 to 15 minutes|
|User profile|5 to 60 minutes|
|Country list / config|Hours to days|

> **Always set a TTL**, even if you also invalidate explicitly. It is your safety net.

### 5.2 Explicit invalidation (delete on write)

When data changes, you delete (or update) the cache key.

```python
db.update(...)
redis.delete("product:42")
```

- **Pros**: fresh right after a change.
- **Cons**: you must remember **every** place that changes the data and every cache key affected. Easy to miss one.

#### Delete vs. update the cache on write?

**Prefer delete.** Updating the cache can produce wrong values due to race conditions:

```
Thread A: writes value=1 to DB
Thread B: writes value=2 to DB
Thread B: writes value=2 to cache
Thread A: writes value=1 to cache   <-- cache now has OLD value 1, DB has 2
```

Deleting is safer: the next read reloads whatever the DB currently holds.

#### Order of operations: update DB first, then delete cache

```
1. Update database
2. Delete cache key
```

Even this has a tiny race (a reader can load old data right before your update, then store it after your delete). It is rare, and the TTL fixes it. If it matters, use **delayed double delete** (delete again after ~500 ms) or event-based invalidation.

### 5.3 Event-driven invalidation

Publish an event when data changes; subscribers clear their caches.

```
Service updates DB -> publishes "product.updated:42" -> all cache holders delete "product:42"
```

- Tools: Redis Pub/Sub, Kafka, change data capture (CDC) from the database log.
- **Pros**: works across many services and servers; no need for the writer to know all caches.
- **Cons**: more infrastructure; events can be delayed or lost (keep a TTL as backup).

### 5.4 Versioned keys (cache busting)

Instead of deleting, change the key.

```python
version = redis.get("product:42:version") or 1
key = f"product:42:v{version}"
# on update: redis.incr("product:42:version")  -> old entries simply become unreachable
```

- **Pros**: no race conditions; invalidates **many** related entries with one increment; perfect for static assets (`app.v123.js`).
- **Cons**: old entries linger until they expire or are evicted.

### 5.5 Tag-based invalidation

Attach tags to entries (`category:shoes`), then purge everything with a tag. Many CDNs support this (surrogate keys / cache tags).

### Invalidation strategy cheat sheet

|Situation|Use|
|---|---|
|Slight staleness is fine|**TTL only**|
|Must be fresh after a write, single service|**TTL + delete on write**|
|Many services / many caches|**TTL + event-driven**|
|Static files, CDN, browser|**Versioned URLs**|
|One change affects many entries|**Tags or version keys**|

> **My confusion-proof rule:** _"TTL always. Delete on write when freshness matters. Events when many systems are involved. Versioned keys for files."_

---

## 6. The cache stampede problem

Also called **thundering herd** or **dogpile effect**.

### What happens

A very popular key expires. At that exact moment, **hundreds or thousands of requests** all miss the cache **at the same time**, and **all** of them go to the database to rebuild the same value.

```
Time 0:00  key "homepage" expires
Time 0:00  1000 requests arrive -> all see MISS
Time 0:00  1000 identical heavy queries hit the DB  -> DB overloaded -> slow -> even more misses -> outage
```

### Why it is dangerous

- The database receives a sudden spike of identical, expensive queries.
- Rebuilding is slow, so **more** requests pile up while waiting.
- It can become a **cascading failure**: the more it slows down, the worse it gets.

### When it is most likely

- Hot keys (homepage, trending items) with one fixed TTL.
- Many keys created at the same time with the same TTL (they all expire together; this is the related **cache avalanche**).
- Cache restart or flush (cold cache).
- Expensive computations (slow to rebuild).

---

## 7. Solutions to stampedes

You usually combine two or three of these.

### 7.1 Request collapsing (coalescing / single-flight)

**Idea:** if many requests want the same missing key, let **one** do the work. The others **wait** and share the result.

```
1000 requests miss "homepage"
   -> 1 request goes to the DB
   -> 999 requests wait for that one result
   -> all 1000 get the answer; DB saw only 1 query
```

**In-process version (one server)**, Python `asyncio` example:

```python
import asyncio

_inflight: dict[str, asyncio.Future] = {}

async def get_with_collapsing(key, loader):
    cached = await redis.get(key)
    if cached:
        return cached

    # Someone is already loading this key? Wait for them.
    if key in _inflight:
        return await _inflight[key]

    # I am the leader: create a future others can await
    fut = asyncio.get_event_loop().create_future()
    _inflight[key] = fut
    try:
        value = await loader()                 # the ONLY DB call
        await redis.set(key, value, ex=300)
        fut.set_result(value)
        return value
    except Exception as e:
        fut.set_exception(e)
        raise
    finally:
        del _inflight[key]
```

Go has this built in: `golang.org/x/sync/singleflight`.

**Across many servers (distributed lock):**

```python
def get_with_lock(key, loader, ttl=300):
    value = redis.get(key)
    if value:
        return value

    lock_key = f"lock:{key}"
    # SET NX = only one caller wins; EX = lock auto-expires if the winner crashes
    if redis.set(lock_key, "1", nx=True, ex=10):
        try:
            value = loader()
            redis.set(key, value, ex=ttl)
            return value
        finally:
            redis.delete(lock_key)
    else:
        # Lost the race: wait briefly and retry reading the cache
        for _ in range(20):
            time.sleep(0.05)
            value = redis.get(key)
            if value:
                return value
        return loader()   # fallback so we never hang forever
```

**Important details**

- Always give the lock an **expiry** (so a crashed worker does not block everyone forever).
- Always have a **fallback** if waiting takes too long.
- Reverse proxies do this for you: Nginx `proxy_cache_lock on;`, Varnish does request coalescing by default.

**Pros:** removes the duplicate load completely. **Cons:** waiting requests are slow while the leader works; adds some complexity.

### 7.2 Stale-while-revalidate (SWR)

**Idea:** when the data is expired, **serve the old (stale) value immediately** and refresh it **in the background**. Nobody waits.

```
0s ----------- TTL (fresh) ----------- 60s ---- stale window ---- 360s
   serve fresh                       serve STALE + refresh in background   must reload
```

**HTTP header version** (browsers, CDNs, proxies):

```
Cache-Control: max-age=60, stale-while-revalidate=300
```

Meaning: for 60 s the response is fresh. For the next 300 s, serve the stale copy instantly while refreshing it in the background.

There is also `stale-if-error=86400`: if the origin is **down**, keep serving the stale copy. Great for resilience.

**Application-level version:**

```python
import json, time

def get_swr(key, loader, fresh_ttl=60, stale_ttl=300):
    raw = redis.get(key)
    if raw:
        entry = json.loads(raw)
        age = time.time() - entry["created_at"]
        if age < fresh_ttl:
            return entry["value"]                     # fresh
        # stale but usable: refresh in background, return old value now
        if redis.set(f"refresh:{key}", "1", nx=True, ex=30):
            background_pool.submit(refresh, key, loader, fresh_ttl, stale_ttl)
        return entry["value"]

    return refresh(key, loader, fresh_ttl, stale_ttl)  # real miss

def refresh(key, loader, fresh_ttl, stale_ttl):
    value = loader()
    entry = {"value": value, "created_at": time.time()}
    redis.set(key, json.dumps(entry), ex=fresh_ttl + stale_ttl)  # physical TTL = fresh + stale
    return value
```

Key trick: store the entry for **fresh + stale** time, and keep your own "created_at" to decide if it is fresh.

**Pros:** fastest user experience; DB gets only one refresh; hides slow loaders. **Cons:** users may briefly see **slightly old** data; not suitable where stale data is dangerous (balances, inventory at checkout).

### 7.3 Probabilistic early expiration (XFetch)

**Idea:** do not wait for the key to expire. As the expiry gets closer, each request has a **small random chance** to refresh the value **early**. The closer to expiry, the higher the chance. So **one** request (usually) refreshes it **before** the herd arrives.

The famous formula (from the paper _"Optimal Probabilistic Cache Stampede Prevention"_):

```
refresh early if:   now - delta * beta * ln(rand()) >= expiry
```

- `delta` = how long the value takes to recompute (slow values are refreshed earlier).
- `beta` = tuning knob, default `1.0` (larger means refresh earlier).
- `rand()` = random number in (0, 1]. `ln(rand())` is negative, so the term pushes "now" forward randomly.

```python
import math, random, time, json

def xfetch(key, loader, ttl=300, beta=1.0):
    raw = redis.get(key)
    if raw:
        entry = json.loads(raw)             # {"value":..., "delta":..., "expiry":...}
        # randomly decide to recompute early
        if time.time() - entry["delta"] * beta * math.log(random.random()) < entry["expiry"]:
            return entry["value"]           # still OK, serve cached

    # Missing OR chosen to refresh early
    start = time.time()
    value = loader()
    delta = time.time() - start             # measure how long recompute took
    entry = {"value": value, "delta": delta, "expiry": time.time() + ttl}
    redis.set(key, json.dumps(entry), ex=ttl * 2)   # keep a bit longer physically
    return value
```

**Pros:** no locks, no coordination, no waiting; elegant and cheap; naturally spreads out refreshes. **Cons:** probabilistic (not a hard guarantee); a few duplicate refreshes can still happen; slightly more math to explain to your team.

### 7.4 TTL jitter (add randomness to expiry)

If you create 10,000 keys with `ttl=300`, they **all** expire at the same second (a cache avalanche). Add random jitter:

```python
ttl = 300 + random.randint(0, 60)     # 300 to 360 seconds
redis.set(key, value, ex=ttl)
```

Super cheap, very effective. **Do this almost everywhere.**

### 7.5 Background refresh / cache warming

A scheduled job (or worker) refreshes hot keys **before** they expire, or pre-fills the cache after a deploy or restart.

- **Pros:** users never see misses on hot keys.
- **Cons:** you need to know which keys are hot; wasted work on keys nobody reads.

### 7.6 Other protective measures

- **Rate limit / circuit breaker** the database path so overload cannot melt it.
- **Serve a default or degraded response** if the loader fails or is too slow.
- **Never expire the hottest keys**; refresh them in the background instead (a "logical expiry" approach).

### Comparison of stampede solutions

|Technique|Prevents duplicate rebuilds|User waits?|Data freshness|Complexity|
|---|---|---|---|---|
|Request collapsing / lock|Yes (strong)|Waiters wait|Fresh|Medium|
|Stale-while-revalidate|Yes (one refresh)|**No**|Slightly stale|Medium|
|Probabilistic early expiry|Mostly|**No**|Fresh|Low-Medium|
|TTL jitter|Prevents mass expiry|No|Fresh|Very low|
|Background warming|Yes|**No**|Fresh|Medium|

**A very solid combo for a hot key:**

> **TTL jitter + request collapsing + stale-while-revalidate**

---

## 8. Other common caching challenges

### 8.1 Cache penetration

Requests for keys that **do not exist** (e.g., random IDs, attacks) always miss the cache and always hit the DB.

**Fixes:**

- **Cache the "not found" result** with a short TTL (negative caching).
- Use a **Bloom filter** to quickly reject keys that definitely do not exist.
- Validate input before querying.

### 8.2 Cache avalanche

Many keys expire together, or the whole cache goes down, so the DB takes everything.

**Fixes:** TTL jitter, replicated/clustered cache (high availability), circuit breakers, gradual warm-up.

### 8.3 Hot key problem

One key gets so much traffic that a single cache node is overloaded.

**Fixes:** add a small **in-process (L1) cache** in front of Redis, or split the key into copies (`key:1`, `key:2`, ... and read a random copy).

### 8.4 Stale data and inconsistency

The cache and DB disagree, or two servers' in-process caches disagree.

**Fixes:** always use TTLs, delete on write, event-driven invalidation, shorter TTLs for critical data. Accept that caches are **eventually consistent** and design the product around that.

### 8.5 Cold start

After a restart or deploy, the cache is empty and the DB gets hit hard.

**Fixes:** cache warming, gradual traffic shifting, persistence (Redis RDB/AOF), request collapsing.

### 8.6 Cache eviction surprises

Memory is full, so the cache silently drops entries.

- Common policies: **LRU** (least recently used), **LFU** (least frequently used), **FIFO**, **random**.
- **Fix:** size memory properly, set `maxmemory` and a sensible policy (`allkeys-lru` is a common default), and **monitor the eviction count**.

### 8.7 Large or badly designed keys/values

Huge values slow the network and block the cache; unclear key names cause collisions.

**Fixes:** keep values small, compress if needed, use clear namespaced keys: `app:env:entity:id:version`.

### 8.8 Caching the wrong thing

- **Personal data** in a shared cache can leak to other users (always include user/tenant ID in the key, and be careful with `Cache-Control: public`).
- **Errors** being cached (do not cache 500 responses for long).
- Data that is **cheap** to compute (caching adds more complexity than it saves).

---

## 9. Decision cheat sheet

|I want to...|Do this|
|---|---|
|Make reads faster with a simple setup|**Cache-aside + TTL**|
|Keep data fresh after updates|**Delete the key on write** (update DB first)|
|Invalidate across many services|**Event-driven invalidation** + TTL backup|
|Update static assets safely|**Versioned filenames**|
|Avoid all keys expiring together|**TTL jitter**|
|Stop 1000 duplicate rebuilds|**Request collapsing** (single-flight / lock)|
|Make users never wait on refresh|**Stale-while-revalidate**|
|Refresh hot keys without locks|**Probabilistic early expiration (XFetch)**|
|Protect against random missing keys|**Negative caching + Bloom filter**|
|Survive origin outages|**`stale-if-error`**|
|Reduce load on one super-hot key|**In-process L1 cache** in front of Redis|

---

## 10. Final checklist

Before shipping a cache, ask yourself:

- [ ] **What layer** is this cache at, and who can clear it?
- [ ] Does every entry have a **TTL**? (always yes)
- [ ] Does the TTL include **jitter**?
- [ ] How does the data get **invalidated** when it changes?
- [ ] What is the **worst staleness** the business accepts?
- [ ] What happens on **miss storms** (stampede)? Do I have **request collapsing** and/or **SWR**?
- [ ] What happens if the cache is **down**? Does the app still work (slower, not broken)?
- [ ] Are keys **namespaced and versioned**? Is any **private data** at risk of being shared?
- [ ] Am I **monitoring** hit ratio, miss rate, evictions, latency, and DB load?

### The one-paragraph summary

> **Caching** = keep fast copies of slow data. **Layers** = browser, CDN, proxy, app memory, Redis, database; cache as close to the user as is safe. **Invalidation** = TTL always, delete on write when needed, events for many systems, versioned keys for files. **Stampede** = a hot key expires and everyone rebuilds it at once. **Fix it** with TTL jitter, request collapsing, stale-while-revalidate, or probabilistic early expiration. Start simple with **cache-aside + TTL + jitter**, then add the rest only when you see the problem.