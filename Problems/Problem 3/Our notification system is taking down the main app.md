---
tags:
  - backpressure
  - interview
  - isolation
  - messaging
  - problem-3
  - problems
  - rate-limiting
  - system-design
---

# Problem 3 — "Our notification system is taking down the main app"

**Domain:** System Design & Scalability  
**Topics:** backpressure · queues · temporal coupling · rate limiting

**In this folder:** scenario (this note) · [[Problem 3 Companion]]

## 🧩 The Scenario

You work at a SaaS company. When a user completes a purchase, the order service does this:

```python
def complete_order(order_id, user_id):
    db.execute("INSERT INTO orders (...) VALUES (...)")
    
    # Notify everything that cares
    email_service.send(user_id, "Order confirmed")        # 300ms
    sms_service.send(user_id, "Your order is on the way") # 400ms
    analytics.track(user_id, "purchase", order_id)        # 200ms
    inventory.reserve(order_id)                           # 150ms
    loyalty.add_points(user_id, order_id)                 # 250ms
    
    return {"status": "ok"}
```

Total response time: **~1.3 seconds per order.**

Then one day the SMS provider goes down. Now every order request **hangs for 30 seconds** then fails. Orders are being lost. The main DB is getting connection pool exhaustion because threads are all stuck waiting on the SMS call.

Your CTO says: _"Notifications should never be able to take down order placement."_

---

## 🤔 Think About It First

What's the core architectural problem here? There are **2 layers of problems** and the naive fix makes one of them worse.

---

## 🔍 Root Cause Analysis

### Problem 1 — Tight Coupling (Architectural)

The order service **directly calls** every downstream service synchronously. This means:

```
Order success = Email ✅ AND SMS ✅ AND Analytics ✅ AND Inventory ✅ AND Loyalty ✅
```

If **any one** of those fails or is slow — the order fails or is slow. You've made your core business flow as reliable as your **least reliable dependency.**

This is called **temporal coupling** — services are coupled not just in code but in _time_. They all must be alive and fast at the exact same moment.

---

### Problem 2 — Wrong Failure Domain

Placing an order and sending an SMS are **not the same thing.** One is core business logic. The other is a side effect.

The failure domain should be:

```
Order placement fails  → user cannot buy    ← catastrophic, prevent at all costs
SMS fails              → user gets no text  ← unfortunate, but recoverable
```

Right now they share the same failure domain. That's the real bug — not the code, the **architecture.**

---

### The Naive Fix That Makes Things Worse

```python
# "Just make them async with threads"
def complete_order(order_id, user_id):
    db.execute("INSERT INTO orders (...) VALUES (...)")
    
    threading.Thread(target=email_service.send, args=(user_id,)).start()
    threading.Thread(target=sms_service.send, args=(user_id,)).start()
    threading.Thread(target=analytics.track, args=(...)).start()
    
    return {"status": "ok"}  # fast now ✅
```

This is **faster** but creates new problems:

- If the process crashes after `INSERT` but before threads finish → **notifications silently lost forever**
- No retry mechanism — if SMS is down, that notification is gone
- No visibility — you have no idea if notifications succeeded
- Thread explosion under load — 1000 concurrent orders = 5000 threads

You traded **latency** for **silent data loss.** Not a good deal.

---

## ✅ The Right Fix — Message Queue + Outbox Pattern

### Step 1 — Introduce a Message Queue

```
Order Service → Queue → [Email Worker]
                      → [SMS Worker]
                      → [Analytics Worker]
                      → [Inventory Worker]
                      → [Loyalty Worker]
```

The order service **publishes one event** and returns. Each downstream service consumes independently, at its own pace, with its own retry logic.

```python
def complete_order(order_id, user_id):
    db.execute("INSERT INTO orders (...) VALUES (...)")
    
    queue.publish("order.completed", {
        "order_id": order_id,
        "user_id": user_id,
        "timestamp": now()
    })
    
    return {"status": "ok"}   # ~5ms, no downstream dependency
```

Now SMS going down affects **only the SMS worker** — not order placement.

---

### Step 2 — The Outbox Pattern (Fixing the Naive Async Problem)

What if the process crashes after `INSERT INTO orders` but before `queue.publish`? The order exists but no notifications fire — **silent loss.**

The fix is the **Transactional Outbox Pattern**:

```python
def complete_order(order_id, user_id):
    with db.transaction():
        db.execute("INSERT INTO orders (...) VALUES (...)")
        
        # Write the event in the SAME transaction as the order
        db.execute("""
            INSERT INTO outbox (event_type, payload, created_at, sent)
            VALUES ('order.completed', ?, NOW(), false)
        """, json.dumps({"order_id": order_id, "user_id": user_id}))
        
    # Both committed atomically, or both rolled back
    return {"status": "ok"}
```

A separate **outbox poller** process reads unsent events and publishes them:

```python
def outbox_poller():
    while True:
        events = db.query("""
            SELECT * FROM outbox
            WHERE sent = false
            ORDER BY created_at
            LIMIT 100
        """)
        
        for event in events:
            queue.publish(event.event_type, event.payload)
            db.execute("UPDATE outbox SET sent = true WHERE id = ?", event.id)
        
        sleep(1)
```

Now the guarantee is:

```
Order inserted ✅  +  Outbox event inserted ✅  →  atomic (same transaction)
Outbox poller picks it up  →  publishes to queue  →  workers consume
```

Even if the process crashes mid-flight, the outbox row is there — the poller will pick it up on restart.

---

## 📊 Architecture Before vs After

```
BEFORE
──────
Request → Order Service → DB
                       → Email Service   (300ms)
                       → SMS Service     (400ms, can hang 30s)
                       → Analytics       (200ms)
                       → Inventory       (150ms)
                       → Loyalty         (250ms)
Total: ~1.3s, failure cascades

AFTER
─────
Request → Order Service → DB + Outbox (atomic)   ~10ms
                ↓
          Outbox Poller → Queue
                            ↓
                     [Email Worker]     independent, retryable
                     [SMS Worker]       independent, retryable
                     [Analytics Worker] independent, retryable
```

---

## 🧠 Key Concepts This Unlocks

|Concept|What it means|
|---|---|
|**Temporal coupling**|Services must be alive at the same time to work|
|**Failure domain**|Group things that _should_ fail together, separate the rest|
|**At-least-once delivery**|Queue delivers the message, worker must be idempotent|
|**Outbox pattern**|Write events to DB in same transaction, poll and publish separately|
|**Idempotency**|Workers must handle duplicate messages safely (queue can redeliver)|

---

## ⚠️ One Thing The Outbox Pattern Introduces

Because the queue can redeliver messages (at-least-once), your workers **must be idempotent:**

```python
def handle_order_completed(event):
    # Bad — duplicate message sends duplicate SMS
    sms_service.send(event.user_id, "Your order is confirmed")

    # Good — check if already processed
    if db.query("SELECT 1 FROM processed_events WHERE event_id = ?", event.id):
        return  # already handled, skip
    
    sms_service.send(event.user_id, "Your order is confirmed")
    db.execute("INSERT INTO processed_events (event_id) VALUES (?)", event.id)
```

---

## 🔑 Key Takeaways

|Lesson|Rule|
|---|---|
|Sync calls to side effects|Never let non-critical side effects live in your critical path|
|Fire and forget threads|Fast but dangerous — no durability, no retry, silent loss|
|Outbox pattern|Atomically save intent with business data, publish separately|
|Queue consumers|Always design for at-least-once, make handlers idempotent|
|Failure domains|Ask — "should this failure affect the user's core action?"|

---

## Next

- Companion deep dive: [[Problem 3 Companion]]
- Back to hub: [[Problems MOC]]
