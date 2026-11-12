# Caching Made Simple: From Layers to Strategies, Invalidation, and Race Conditions

> **Goal of this guide:** understand caching at the application level (the layer you will use most), know exactly which **read strategy** and **write strategy** to pick, and understand the problems that come with them: **invalidation, race conditions, and stampedes**.

---

## Table of Contents

1. [Quick tour of the caching layers](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#1-quick-tour-of-the-caching-layers)
2. [Application-level caching: the mental model](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#2-application-level-caching-the-mental-model)
3. [Read strategies](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#3-read-strategies)
4. [Write strategies](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#4-write-strategies)
5. [Combining a read and a write strategy](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#5-combining-a-read-and-a-write-strategy)
6. [Cache invalidation](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#6-cache-invalidation)
7. [Race conditions and why delete solves them](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#7-race-conditions-and-why-delete-solves-them)
8. [The cache stampede problem](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#8-the-cache-stampede-problem)
9. [Solutions to stampedes](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#9-solutions-to-stampedes)
10. [Other common caching problems](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#10-other-common-caching-problems)
11. [Cheat sheet](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#11-cheat-sheet)
12. [Final checklist and summary](https://claude.ai/chat/358fac76-c3f2-4dff-8512-4c9ab4d3b8d2#12-final-checklist-and-summary)

---

## 1. Quick tour of the caching layers

A request can pass through **many caches** before it reaches your database. Here is the full path, from the user to the data:

```
 User
  |
  v
[1] Browser cache            (the user's device)
  |
  v
[2] CDN / edge cache         (servers around the world)
  |
  v
[3] Reverse proxy cache      (Nginx, Varnish in front of your app)
  |
  v
[4] Application cache        (your code + Redis / Memcached)   <-- THIS GUIDE FOCUSES HERE
  |
  v
[5] Database cache           (buffer pool, handled by the DB)
  |
  v
 Disk / source of truth
```

A very short introduction to each:

|Layer|Where it lives|What it caches|Key thing to know|
|---|---|---|---|
|**1. Browser**|The user's device|Images, CSS, JS, API responses|Controlled by HTTP headers (`Cache-Control`, `ETag`). You **can't clear it** from the server, so use **versioned filenames** (`app.3f9a1c.js`).|
|**2. CDN / edge**|Servers near the user|Static files, sometimes whole pages|Huge speed win and protects your servers. Purging can take time. Be careful with personalized content.|
|**3. Reverse proxy**|In front of your app|Full HTTP responses|No app code needed. Supports request collapsing and serving stale content (`proxy_cache_lock`, `stale-while-revalidate`).|
|**4. Application**|Your code, plus a shared cache like Redis|Query results, objects, computed values|**You control it completely**, so this is where you make the decisions in this guide.|
|**5. Database**|Inside the DB|Pages and indexes|Automatic. You tune it, you don't manage it.|

> **Rule of thumb:** cache as close to the user as is **safe**. Layers 1 to 3 are mostly about HTTP and static or public content. Layer 4 is where your **business data** is cached, and where **freshness and correctness** become your job.

The rest of this guide is about **layer 4**: the application cache.

---

## 2. Application-level caching: the mental model

### The three players

```
   +-----------+        +---------+        +------------+
   |   Your    | -----> |  Cache  | -----> |  Database  |
   |   App     |        | (Redis) |        | (Postgres) |
   +-----------+        +---------+        +------------+
                         fast, small        slow, THE SOURCE OF TRUTH
```

- **Database:** the source of truth. It holds the real, correct data.
- **Cache:** a **disposable, fast copy** of some of that data. If the cache is wiped, nothing is lost. You just reload from the database.
- **Your app:** decides when to read from the cache, when to fill it, and when to remove old copies.

### Key words

|Term|Meaning|
|---|---|
|**Cache hit**|The data was found in the cache. Fast.|
|**Cache miss**|The data was not in the cache. You go to the database.|
|**TTL (time to live)**|How long a cache entry lives before it expires automatically.|
|**Stale data**|A cached copy that no longer matches the database.|
|**Invalidation**|Making sure stale data stops being served.|

### The two questions every caching design answers

```
Q1 (READ):   when data is missing from the cache, who loads it, and how?
Q2 (WRITE):  when data changes, what do we do with the cache?
```

So you pick **one read strategy** and **one write strategy**, and combine them. The next two sections cover the options for each.

### Running example

An online shop. The data we cache is a **product price**. Right now it is **100** in both the cache and the database.

---

## 3. Read strategies

There are two. Both do the same job (**fill the cache on a miss**). They differ only in **who** does it.

### 3.1 Cache-aside (lazy loading), the most common

**Idea:** your application code is in charge. It talks to the cache **and** the database itself. The cache is just a storage box.

**Analogy:** you check the fridge first. If the milk is there, great. If not, **you** go to the store, buy it, and put a bottle in the fridge for next time.

```
 App wants the price of product 42
   |
   1. Ask the cache: "do you have price:42?"
   |
   +-- HIT  --> return it. Done. (fast)
   |
   +-- MISS
        2. App asks the database               (slow)
        3. App PUTS the answer in the cache    (with a TTL)
        4. App returns the answer
```

```python
def get_price(product_id):
    key = f"price:{product_id}"

    cached = redis.get(key)                     # step 1
    if cached is not None:
        return float(cached)                    # hit, done

    price = db.query("SELECT price FROM products WHERE id = %s", product_id)  # step 2
    redis.set(key, price, ex=300)               # step 3 (TTL = 5 minutes)
    return price                                # step 4
```

**Pros**

- Simple and easy to understand.
- Only caches data that is **actually requested** (no wasted memory).
- If the cache dies, the app still works (slower), because it falls back to the database.

**Cons**

- The **first** request for each item is slow (a miss).
- You write the "check, load, fill" code yourself (usually in one helper function).
- Stale data is possible if you don't invalidate on writes (Sections 4 and 6).

**Use it when:** you want a safe default for almost any read-heavy app. **If unsure, start here.**

### 3.2 Read-through

**Idea:** the app **only talks to the cache**. If the data is missing, the **cache layer itself** loads it from the database. You register a "loader" once, and the cache handles misses for you.

**Analogy:** you ask a **butler** for milk. You never look in the fridge or go to the store. If it's missing, the butler fetches it.

```
 App wants the price of product 42
   |
   1. App asks the CACHE LAYER (only the cache layer)
   |
   +-- HIT  --> cache returns it
   |
   +-- MISS
        2. The cache layer calls the loader --> database
        3. The cache layer stores the result
        4. The cache layer returns it to the app
```

```python
# Configure the loader ONCE...
cache = ReadThroughCache(
    loader=lambda product_id: db.query(
        "SELECT price FROM products WHERE id = %s", product_id),
    ttl=300,
)

# ...then the app just asks the cache. No if/else, no database code.
price = cache.get(42)
```

||Who handles a miss?|
|---|---|
|**Cache-aside**|**Your application code**|
|**Read-through**|**The cache library / provider**|

**Pros:** cleaner app code; one central place to add protections (like request collapsing). **Cons:** needs a library that supports it (Caffeine `LoadingCache`, Guava, some cloud caches). **Plain Redis does not do this by itself.**

> **Tip:** many teams write a small helper `get_or_load(key, loader)` around cache-aside. That is effectively read-through, built by hand.

### 3.3 What happens on a read

|Situation|Cache action|
|---|---|
|**Hit**|Nothing, just return the value|
|**Miss**|**Fill** the cache from the database (a `set`, never a delete)|
|**TTL runs out**|The cache removes the key by itself|

On a miss there is nothing to delete, because the key is already empty. Reads only ever **fill**.

---

## 4. Write strategies

When data **changes** (a create, update or delete in your database), the cache may be holding an old copy. You must decide what to do about it.

There are **four** options. The first one is not one of the classic named patterns, but it is the **most common in real systems**.

> **Rule for all of them:** the database is the source of truth. It is always written. The question is only what you do with the **cache**.

### 4.1 Delete on write (write-invalidate), the safe default

**Idea:** update the database, then **delete the cache key**. Don't put the new value into the cache. The next read will miss, load the fresh value from the database, and fill the cache.

**Analogy:** the price on the front desk is now wrong. You don't rewrite it yourself, you just **throw the old card away**. The next customer who asks makes someone fetch a fresh one from the warehouse.

```python
def update_price(product_id, new_price):
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)  # 1. DB first
    redis.delete(f"price:{product_id}")                                                # 2. then delete the key
```

|Time|What happens|Cache|DB|
|---|---|---|---|
|10:00|Read, miss, load, fill|`100`|`100`|
|10:01|Read, **hit**|`100`|`100`|
|10:02|Admin sets price to 120: DB updated, key deleted|_(empty)_|`120`|
|10:03|Read, miss, load `120`, fill|`120`|`120`|

**Pros:** simple; **safest against race conditions** (Section 7); the cache never holds a value your code made up. **Cons:** the next read after a delete is a miss (one slow request); for a super-hot key this can cause a stampede (Section 8).

**Use it when:** this is your default for most apps.

### 4.2 Write-through

**Idea:** every write goes to the **database and the cache together**, and the write counts as "done" only when **both** succeed. The cache always holds the fresh value.

**Analogy:** you buy milk. Before leaving the kitchen you put it in the fridge **and** update the shopping log.

```
 App sets the price of product 42 to 120
   1. Write to the database
   2. Set the new value in the cache
   3. Both succeeded --> tell the app "done"
```

```python
def update_price(product_id, new_price):
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)
    redis.set(f"price:{product_id}", new_price, ex=3600)   # set, NOT delete
```

**Pros:** reads right after a write are **always fresh**; reads are fast (no miss). **Cons:** writes are slower (two writes); you may cache data nobody reads; **it has a race condition when two writers overlap** unless you lock or version the writes (Section 7.5).

**Use it when:** data is **read soon after it is written** and must be fresh (settings, profile, cart).

### 4.3 Write-behind (write-back)

**Idea:** write to the **cache only** and say "done" immediately. The cache saves to the database **later**, in the background (often in batches).

**Analogy:** you scribble on a **sticky note** now, and copy it into the official log tonight. Fast, but if you lose the note first, the data is gone.

```python
def record_view(product_id):
    redis.incr(f"views:{product_id}")         # instant, no database involved

# a background worker runs every 10 seconds
def flush_views_to_db():
    for key in redis.scan_iter("views:*"):
        product_id = key.split(":")[1]
        count = redis.getdel(key)             # take the counter and reset it
        if count:
            db.execute("UPDATE products SET views = views + %s WHERE id = %s",
                       int(count), product_id)
```

Instead of 10,000 tiny database writes per second, the database gets **one batched write** every few seconds.

**Pros:** extremely fast writes; far less database load; absorbs write spikes. **Cons:** **risk of data loss** if the cache crashes before the save; the database is temporarily behind; more complex.

**Use it when:** very heavy writes and losing a few seconds is acceptable: view counters, likes, analytics, game scores. **Never use it for:** payments, orders, balances.

### 4.4 Write-around

**Idea:** writes go **straight to the database** and **skip the cache**. The cache fills later, only when someone reads (like cache-aside).

**Analogy:** new stock goes straight to the **warehouse**. The front desk only gets an item after a customer asks for it.

```python
def save_report(report):
    db.execute("INSERT INTO reports (...) VALUES (...)", report)   # database only
```

**Why:** data written once and rarely read would **fill the cache with junk** and push out hot data. **Pros:** keeps the cache clean; simple, fast writes. **Cons:** the first read after a write is a miss. If an **old copy** is already cached, you still have to delete it. **Use it when:** logs, bulk imports, audit trails, large files.

### 4.5 The write options side by side

|Write option|What it does to the cache|Write speed|Fresh after write?|Data-loss risk|Complexity|
|---|---|---|---|---|---|
|**Delete on write**|**Deletes** the key|Fast|Yes, after the next read|None|**Low**|
|**Write-through**|**Sets** the new value|Slower (2 writes)|Always|None|Medium|
|**Write-behind**|**Sets** the cache first, DB later|**Fastest**|Cache yes, DB lags|**Yes**|High|
|**Write-around**|**Skips** the cache|Fast|First read is a miss|None|Low|

---

## 5. Combining a read and a write strategy

You pick **one from each group**:

```
READ  (who loads on a miss?)          WRITE (what happens to the cache on a change?)
-------------------------------       ----------------------------------------------
 - Cache-aside                         - Delete on write
 - Read-through                        - Write-through
                                       - Write-behind
                                       - Write-around
```

Common combinations:

|Combo|Read|Write|Good for|
|---|---|---|---|
|**The default**|Cache-aside|Delete on write|Most web apps and APIs|
|**Always-fresh data**|Cache-aside or read-through|Write-through|Settings, carts, read-after-write|
|**High-volume counters**|Cache-aside|Write-behind|Likes, views, analytics|
|**Write-heavy, rarely read**|Cache-aside|Write-around|Logs, imports, archives|

### How to choose: a simple decision path

```
Q1. Is the data critical (money, orders)?
     YES --> never write-behind. Use delete on write (or write-through).
     NO  --> continue

Q2. Written VERY often (thousands/sec) and OK to lose a few seconds?
     YES --> write-behind (with batching)
     NO  --> continue

Q3. Read right after being written, and must be fresh immediately?
     YES --> write-through (with locking or versioning)
     NO  --> continue

Q4. Written often but rarely read?
     YES --> write-around
     NO  --> delete on write  (the safe default)
```

> **Remember:** read strategies decide **who fetches on a miss**. Write strategies decide **what happens to the cache when data changes**. They are two separate choices.

---

## 6. Cache invalidation

**Invalidation = making sure the cache stops serving data that is no longer true.**

Invalidation is not a separate pattern. It is the part of your strategy that handles stale data. Some write options are invalidation methods (delete on write), and some invalidation tools sit on top of any strategy (TTL, events).

|Invalidation method|Used with|
|---|---|
|**TTL**|Every strategy (the safety net)|
|**Delete on write**|Cache-aside / read-through|
|**Update on write**|Write-through / write-behind|
|**Events, versioned keys, tags**|Any strategy, extra tools|

### 6.1 TTL (time to live)

Each entry gets a lifetime. When it ends, the entry disappears.

```python
redis.set("product:42", data, ex=60)   # expires in 60 seconds
```

- **Pros:** dead simple; **self-healing** (even if you forget to invalidate, it fixes itself).
- **Cons:** data can be stale **until the TTL ends**.
- **Choose a TTL by asking:** "How long can this data be wrong before it hurts?"

|Data|Suggested TTL|
|---|---|
|Live price / score|1 to 10 seconds|
|Product details|1 to 15 minutes|
|User profile|5 to 60 minutes|
|Country list / config|Hours to days|

> **Always set a TTL**, even if you also invalidate explicitly. It is your safety net.

### 6.2 Delete on write (explicit invalidation)

Covered in Section 4.1. Update the database first, then delete the key.

- **Pros:** fresh right after a change.
- **Cons:** you must remember **every** place that changes the data and **every cache key** it affects. If you miss one, that key stays stale until its TTL ends.

### 6.3 Event-driven invalidation

Publish an event when data changes. Everyone who caches it listens and deletes their copy.

```
Service updates the DB --> publishes "product.updated:42" --> all cache holders delete "product:42"
```

- **Tools:** Redis Pub/Sub, Kafka, change data capture (CDC) from the database log.
- **Pros:** works across many services and servers; the writer doesn't need to know every cache.
- **Cons:** more infrastructure; events can be delayed or lost (keep a TTL as backup).

### 6.4 Versioned keys (cache busting)

Instead of deleting, change the key.

```python
version = redis.get("product:42:version") or 1
key = f"product:42:v{version}"
# on update: redis.incr("product:42:version")   -> old entries become unreachable
```

- **Pros:** no race conditions; one increment invalidates many related entries; perfect for static files (`app.v123.js`).
- **Cons:** old entries linger until they expire or are evicted.

### 6.5 Tag-based invalidation

Attach tags to entries (`category:shoes`), then purge everything with a tag. Many CDNs support this (surrogate keys, cache tags).

### 6.6 Which invalidation should I use?

|Situation|Use|
|---|---|
|Slight staleness is fine|**TTL only**|
|Must be fresh after a write, one service|**TTL + delete on write**|
|Many services / many caches|**TTL + event-driven**|
|Static files, CDN, browser|**Versioned URLs**|
|One change affects many entries|**Tags or version keys**|

> **Confusion-proof rule:** _TTL always. Delete on write when freshness matters. Events when many systems are involved. Versioned keys for files._

---

## 7. Race conditions and why delete solves them

A **race condition** is when two things happen at almost the same moment and the final result depends on **which one finishes first**. In caching, the result is a cache holding a **wrong value**.

We use the same example: the price is `100`, and someone changes it to `120`.

### 7.1 Race 1: two writers, and you update the cache

```python
def update_price(product_id, new_price):
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)
    redis.set(f"price:{product_id}", new_price)        # update the cache too
```

Admin A sets the price to **1**, and Admin B sets it to **2**, at almost the same time. Admin A's request is slightly slower.

|Step|Admin A (price = 1)|Admin B (price = 2)|DB|Cache|
|---|---|---|---|---|
|1|writes DB = 1||1|100|
|2||writes DB = 2|2|100|
|3||writes cache = 2|2|2|
|4|writes cache = 1 (late)||2|**1**|

**Result:** the database says **2**, but the cache says **1**, and it stays wrong until the TTL ends.

**Why:** the two steps (write the database, write the cache) are **not one atomic action**. Requests can interleave them, so the cache gets the value from whoever wrote **last to the cache**, which isn't always the one who wrote last to the database.

### 7.2 Why delete solves Race 1

```python
def update_price(product_id, new_price):
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)
    redis.delete(f"price:{product_id}")                # delete, don't set
```

|Step|Admin A (price = 1)|Admin B (price = 2)|DB|Cache|
|---|---|---|---|---|
|1|writes DB = 1||1|100|
|2||writes DB = 2|2|100|
|3||deletes the key|2|_(empty)_|
|4|deletes the key (late, nothing there)||2|_(empty)_|
|5|_(next reader)_ miss, loads DB (**2**), fills cache||2|**2**|

**Result:** the cache ends with **2**, matching the database.

**Why it works:**

- A delete carries **no value**, so there's no old value that can arrive late and overwrite a newer one.
- The order of the two deletes doesn't matter. Both end in "empty".
- The cached value always comes from a **fresh database read**.

> **A `set` can bring a stale value into the cache. A `delete` can only take a value out.**

### 7.3 The deeper reason: let the database decide

The database already has tools for concurrency: **transactions, isolation levels, and locks**. When two admins update the same row, the database decides the order and keeps one consistent result.

The cache **can't take part in that**. The database and Redis are two separate systems, and you can't update both in one atomic step. If you also write the value into Redis from your code, you have to reproduce the database's ordering yourself, and that is where the race comes from.

Deleting avoids this. Your code doesn't try to decide which value is right. It says "my copy may be old" and lets the next read ask the database, which already knows.

> **Let the database decide the truth, and let the cache only copy it on read.**

### 7.4 Race 2: delete the cache first, then update the database (wrong order)

```python
def update_price(product_id, new_price):
    redis.delete(f"price:{product_id}")                # 1. delete first
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)  # 2. DB second
```

|Step|Writer (100 to 120)|Reader|DB|Cache|
|---|---|---|---|---|
|1|deletes the key||100|_(empty)_|
|2||miss, reads DB (still **100**)|100|_(empty)_|
|3|updates DB to 120||120|_(empty)_|
|4||stores **100** in the cache|120|**100**|

**Result:** the cache holds the old price. The window between the two steps is as long as the database update takes, so this is **easy to hit**.

### 7.5 Why "DB first, then delete" is the right order

|Step|Writer (100 to 120)|Reader|DB|Cache|
|---|---|---|---|---|
|1|updates DB to 120||120|100|
|2||hit, gets **100** (briefly stale)|120|100|
|3|deletes the key||120|_(empty)_|
|4||miss, loads **120**, fills cache|120|**120**|

The gap is now only the time of **one fast delete command**. The cache may serve the old value for a moment, but it **heals itself** at step 4.

### 7.6 Race 3: the rare leftover (a slow reader)

"DB first, then delete" is not perfect. One race remains:

|Step|Reader (slow)|Writer (100 to 120)|DB|Cache|
|---|---|---|---|---|
|1|miss, reads DB (gets **100**)||100|_(empty)_|
|2|_(reader gets delayed)_|updates DB to 120|120|_(empty)_|
|3||deletes the key (nothing there)|120|_(empty)_|
|4|stores **100** in the cache||120|**100**|

**Why it's rare:** the reader must read before the update, then be slower than the writer's whole update and delete. Reads are usually much faster than writes. But at high traffic, "rare" will eventually happen.

### 7.7 Race 4: crash between the database update and the delete

The database update is atomic **inside the database**. The delete is a **separate step in a separate system**. If your app crashes (or Redis is unreachable) between them, the database has the new value and the cache still has the old one.

```
DB update succeeded  -->  [crash / Redis down]  -->  delete never happened
```

### 7.8 How to handle the leftover races

| Tool                                         | How it helps                                                                                                                                  |
| -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **TTL (always)**                             | Turns "wrong forever" into "wrong for at most N seconds". Covers races 3 and 4.                                                               |
| **Delayed double delete**                    | Delete right after the update, then **delete again** after about 500 ms. The second delete removes any stale value a slow reader just stored. |
| **Retry the delete**                         | If the delete fails, put it in a retry queue instead of dropping it.                                                                          |
| **Event-driven invalidation (CDC / outbox)** | Deletes are triggered **after** the change is committed, in order, and can be retried.                                                        |
| **Version number**                           | Store a version with the value, and only accept a cache write that is **newer** than what's there.                                            |

```python
def update_price(product_id, new_price):
    key = f"price:{product_id}"
    db.execute("UPDATE products SET price = %s WHERE id = %s", new_price, product_id)
    redis.delete(key)                        # first delete
    schedule_after(0.5, redis.delete, key)   # second delete, about 500 ms later
```

### 7.9 Does write-through have the same race?

Yes, if it's implemented the simple way (Race 1 above). Real write-through systems avoid it by making sure **writes to the same key happen one at a time, in order**:

|Technique|How it helps|
|---|---|
|**Per-key lock**|Only one writer at a time can update the DB and cache for a key|
|**Version check**|A late writer holding an older version is rejected|
|**Single writer / queue**|All updates for a key go through one path, so the order is the same in the cache and the database|

All of these add complexity. **Delete avoids the problem without any of them**, which is why it's the default.

### 7.10 Race conditions summary

|Approach|Race?|Likelihood|Damage|Verdict|
|---|---|---|---|---|
|Update DB, then **set** cache|Yes (two writers)|Medium|Wrong until TTL|Avoid unless you lock or version|
|**Delete** cache, then update DB|Yes (reader in the gap)|**High**|Wrong until TTL|**Never use this order**|
|Update DB, then **delete** cache|Yes (slow reader, crash)|**Rare**|Fixed by TTL|**Best default**|
|Update DB, delete, **delete again**|Almost none|Very rare|Tiny|Good when staleness hurts|
|Versioned keys / events|None by design|n/a|n/a|Best for strict freshness, more work|

> **No approach removes races completely.** You choose how rare and how harmless they are, and make sure the cache can **heal itself** (TTL).

---

## 8. The cache stampede problem

Also called **thundering herd** or **dogpile effect**.

### What happens

A very popular key expires, or you delete it on a write. At that exact moment, **hundreds or thousands of requests** all miss the cache at the same time, and **all** of them go to the database to rebuild the same value.

```
Time 0:00  key "homepage" expires (or is deleted)
Time 0:00  1000 requests arrive --> all see MISS
Time 0:00  1000 identical heavy queries hit the DB --> DB overloaded --> slow --> even more misses --> outage
```

> **Note the link to delete on write:** deleting a **hot key** is the most common way to trigger a stampede on purpose. For hot keys, pair delete on write with one of the protections below.

### Why it's dangerous

- The database gets a sudden spike of identical, expensive queries.
- Rebuilding is slow, so **more** requests pile up while waiting.
- It can become a **cascading failure**: the slower it gets, the worse it gets.

### When it's most likely

- Hot keys (homepage, trending items) with one fixed TTL.
- Many keys created together with the same TTL (they expire together: a **cache avalanche**).
- Cache restart or flush (cold cache).
- Expensive computations that are slow to rebuild.

---

## 9. Solutions to stampedes

You usually combine two or three of these.

### 9.1 Request collapsing (coalescing / single-flight)

**Idea:** if many requests want the same missing key, let **one** do the work. The others **wait** and share the result.

```
1000 requests miss "homepage"
   --> 1 request goes to the DB
   --> 999 requests wait for that one result
   --> all 1000 get the answer; the DB saw only 1 query
```

**In-process version (one server)**, Python `asyncio`:

```python
import asyncio

_inflight: dict[str, asyncio.Future] = {}

async def get_with_collapsing(key, loader):
    cached = await redis.get(key)
    if cached:
        return cached

    if key in _inflight:                       # someone is already loading it
        return await _inflight[key]

    fut = asyncio.get_event_loop().create_future()   # I am the leader
    _inflight[key] = fut
    try:
        value = await loader()                 # the ONLY database call
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
    # NX = only one caller wins; EX = the lock expires if the winner crashes
    if redis.set(lock_key, "1", nx=True, ex=10):
        try:
            value = loader()
            redis.set(key, value, ex=ttl)
            return value
        finally:
            redis.delete(lock_key)
    else:
        for _ in range(20):                    # lost the race: wait briefly and re-read
            time.sleep(0.05)
            value = redis.get(key)
            if value:
                return value
        return loader()                        # fallback so we never hang forever
```

**Important details:** always give the lock an **expiry**, and always have a **fallback** if waiting takes too long. Reverse proxies do this for you (Nginx `proxy_cache_lock on;`, Varnish by default).

**Pros:** removes the duplicate load completely. **Cons:** waiting requests are slow while the leader works; some complexity.

### 9.2 Stale-while-revalidate (SWR)

**Idea:** when data has expired, **serve the old (stale) value immediately** and refresh it **in the background**. Nobody waits.

```
0s ----------- fresh (60s) ----------- stale window (up to 360s) ----------
   serve fresh                     serve STALE + refresh in background     must reload
```

**HTTP header version** (browsers, CDNs, proxies):

```
Cache-Control: max-age=60, stale-while-revalidate=300
```

For 60 s the response is fresh. For the next 300 s, serve the stale copy instantly while refreshing it in the background. `stale-if-error=86400` keeps serving the stale copy if the origin is **down**.

**Application-level version:**

```python
import json, time

def get_swr(key, loader, fresh_ttl=60, stale_ttl=300):
    raw = redis.get(key)
    if raw:
        entry = json.loads(raw)
        age = time.time() - entry["created_at"]
        if age < fresh_ttl:
            return entry["value"]                          # fresh
        if redis.set(f"refresh:{key}", "1", nx=True, ex=30):   # only one refresher
            background_pool.submit(refresh, key, loader, fresh_ttl, stale_ttl)
        return entry["value"]                              # stale but usable

    return refresh(key, loader, fresh_ttl, stale_ttl)      # real miss

def refresh(key, loader, fresh_ttl, stale_ttl):
    value = loader()
    entry = {"value": value, "created_at": time.time()}
    redis.set(key, json.dumps(entry), ex=fresh_ttl + stale_ttl)   # keep for fresh + stale
    return value
```

**Pros:** fastest user experience; the database gets one refresh; hides slow loaders. **Cons:** users may briefly see slightly old data; not for data where stale is dangerous (balances, checkout inventory).

### 9.3 Probabilistic early expiration (XFetch)

**Idea:** don't wait for the key to expire. As expiry gets closer, each request has a **small random chance** to refresh early. So **one** request usually refreshes the value **before** the herd arrives.

```
refresh early if:   now - delta * beta * ln(rand()) >= expiry
```

- `delta` = how long recomputing takes (slow values refresh earlier).
- `beta` = tuning knob, default `1.0` (larger means earlier).
- `rand()` = random number in (0, 1].

```python
import math, random, time, json

def xfetch(key, loader, ttl=300, beta=1.0):
    raw = redis.get(key)
    if raw:
        entry = json.loads(raw)          # {"value":..., "delta":..., "expiry":...}
        if time.time() - entry["delta"] * beta * math.log(random.random()) < entry["expiry"]:
            return entry["value"]        # still fine, serve cached

    start = time.time()                  # missing, or chosen to refresh early
    value = loader()
    delta = time.time() - start
    entry = {"value": value, "delta": delta, "expiry": time.time() + ttl}
    redis.set(key, json.dumps(entry), ex=ttl * 2)
    return value
```

**Pros:** no locks, no coordination, no waiting. **Cons:** probabilistic (not a hard guarantee); a few duplicate refreshes can still happen.

### 9.4 TTL jitter

If you create 10,000 keys with `ttl=300`, they all expire in the same second (an avalanche). Add randomness:

```python
ttl = 300 + random.randint(0, 60)     # 300 to 360 seconds
redis.set(key, value, ex=ttl)
```

Very cheap and very effective. **Do this almost everywhere.**

### 9.5 Background refresh / cache warming

A scheduled job refreshes hot keys **before** they expire, or pre-fills the cache after a deploy or restart. **Pros:** users never see misses on hot keys. **Cons:** you must know which keys are hot.

### 9.6 Comparison

|Technique|Prevents duplicate rebuilds|User waits?|Freshness|Complexity|
|---|---|---|---|---|
|Request collapsing / lock|Yes (strong)|Waiters wait|Fresh|Medium|
|Stale-while-revalidate|Yes (one refresh)|**No**|Slightly stale|Medium|
|Probabilistic early expiry|Mostly|**No**|Fresh|Low-Medium|
|TTL jitter|Prevents mass expiry|No|Fresh|Very low|
|Background warming|Yes|**No**|Fresh|Medium|

**A solid combo for a hot key:** **TTL jitter + request collapsing + stale-while-revalidate**.

---

## 10. Other common caching problems

### 10.1 Cache penetration

Requests for keys that **don't exist** (random IDs, attacks) always miss and always hit the database. **Fixes:** cache the "not found" result with a short TTL (negative caching), use a **Bloom filter**, validate input first.

### 10.2 Cache avalanche

Many keys expire together, or the whole cache goes down. **Fixes:** TTL jitter, a replicated or clustered cache, circuit breakers, gradual warm-up.

### 10.3 Hot key

One key gets so much traffic that a single cache node is overloaded. **Fixes:** a small **in-process (L1) cache** in front of Redis, or split the key into copies (`key:1`, `key:2`, ... and read a random one).

### 10.4 Stale data and inconsistency

The cache and database disagree, or two servers' in-process caches disagree. **Fixes:** always use TTLs, delete on write, events, shorter TTLs for critical data. Accept that caches are **eventually consistent**.

### 10.5 Cold start

After a restart or deploy the cache is empty and the database is hit hard. **Fixes:** cache warming, gradual traffic shifting, Redis persistence (RDB/AOF), request collapsing.

### 10.6 Eviction surprises

When memory is full, the cache silently drops keys by its own policy (**LRU**, **LFU**, **FIFO**, random). This is **eviction**, and it is different from invalidation (removing a key on purpose because data changed). **Fix:** size memory properly, set `maxmemory` and a policy (`allkeys-lru` is a common default), and **monitor evictions**.

### 10.7 Bad keys and values

Huge values slow the network. Unclear key names cause collisions. **Fixes:** keep values small, compress if needed, namespace keys: `app:env:entity:id:version`.

### 10.8 Caching the wrong thing

- **Personal data** in a shared cache can leak to other users. Include the user or tenant ID in the key.
- **Errors** being cached. Don't cache 500 responses for long.
- Data that is **cheap** to compute. Caching adds more complexity than it saves.

---

## 11. Cheat sheet

|I want to...|Do this|
|---|---|
|Make reads faster with a simple setup|**Cache-aside + TTL**|
|Keep data fresh after updates, safely|**Update DB first, then delete the key**|
|Have fresh data right after a write|**Write-through** (with a lock or versioning)|
|Handle huge write volume, small loss is OK|**Write-behind** (counters, analytics)|
|Avoid caching write-once data|**Write-around**|
|Invalidate across many services|**Event-driven invalidation** + TTL backup|
|Update static files safely|**Versioned filenames**|
|Avoid all keys expiring together|**TTL jitter**|
|Stop 1000 duplicate rebuilds|**Request collapsing**|
|Make users never wait on a refresh|**Stale-while-revalidate**|
|Refresh hot keys without locks|**Probabilistic early expiration**|
|Guard against random missing keys|**Negative caching + Bloom filter**|
|Survive an origin outage|**`stale-if-error`**|
|Fix the rare slow-reader race|**TTL + delayed double delete**|

---

## 12. Final checklist and summary

Before shipping a cache, ask yourself:

- [ ] **Which layer** is this cache at, and who can clear it?
- [ ] Which **read strategy** (cache-aside or read-through) and which **write strategy** (delete, write-through, write-behind, write-around)?
- [ ] Is the **database** the single source of truth? Do I always write it first?
- [ ] Does every entry have a **TTL**, with **jitter**?
- [ ] On a write, do I **delete** the key (not set it) unless I have a good reason?
- [ ] What happens if the **delete fails** or the app crashes right after the database update?
- [ ] What is the **worst staleness** the business accepts?
- [ ] What happens on a **miss storm**? Do I have **request collapsing** or **stale-while-revalidate** for hot keys?
- [ ] What happens if the **cache is down**? Does the app still work (slower, not broken)?
- [ ] Are keys **namespaced**? Is any **private data** at risk of being shared?
- [ ] Am I **monitoring** hit ratio, evictions, latency, and database load?

### The one-paragraph summary

> **Layers:** browser, CDN and proxy mostly cache public HTTP content. The **application cache** (Redis) is where your business data lives, and where you make the decisions. **Strategies:** pick one **read** strategy (cache-aside, or read-through) and one **write** strategy (delete on write, write-through, write-behind, or write-around). The safe default is **cache-aside + update the database first, then delete the key + a TTL with jitter**. **Why delete:** the database already handles concurrency, and a delete can't bring a stale value into the cache, so you let the database decide the truth and the cache copy it on the next read. **Leftover problems:** a rare slow-reader race and a crash between the two steps are covered by the TTL (and optionally a delayed double delete). **Hot keys:** a delete or expiry can cause a stampede, so add request collapsing, stale-while-revalidate, or early refresh. Start simple, and add the rest only when you see the problem.