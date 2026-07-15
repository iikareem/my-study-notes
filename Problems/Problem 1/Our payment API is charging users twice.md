---
tags:
  - distributed-lock
  - exactly-once
  - fencing-token
  - idempotency
  - interview
  - problem-1
  - problems
  - system-design
---

# Problem 1 — "Our payment API is charging users twice"

**Domain:** Concurrency & API Design  
**Topics:** idempotency · distributed locks · fencing tokens · exactly-once

**In this folder:** scenario (this note) · [[Problem 1 Companion]]

## 🧩 The Scenario

You run a subscription platform. The payment endpoint looks like this:

```python
def charge_user(user_id, amount, payment_method_id):
    # Check if user already has a pending charge
    existing = db.query("""
        SELECT id FROM payments
        WHERE user_id = ? AND status = 'pending'
    """, user_id)

    if existing:
        raise ConflictError("Payment already in progress")

    payment = db.execute("""
        INSERT INTO payments (user_id, amount, status, created_at)
        VALUES (?, ?, 'pending', NOW())
    """, user_id, amount)

    # Call external payment gateway (Stripe, etc.)
    result = stripe.charge(payment_method_id, amount)

    db.execute("""
        UPDATE payments SET status = 'completed'
        WHERE id = ?
    """, payment.id)

    return {"status": "charged", "payment_id": payment.id}
```

Users are getting **double charged.** Support tickets are flooding in. You dig into the logs and find two patterns causing it:

- **Pattern A** — User clicks "Pay" button twice fast on a slow connection
- **Pattern B** — The mobile app has a retry mechanism that retries on network timeout — but the first request already succeeded at the Stripe level

---

## 🤔 Think About It First

There are **3 layers** to this problem. The `existing` check looks like it should prevent duplicates — why doesn't it? And why is Pattern B particularly dangerous?

---

## 🔍 Root Cause Analysis

### Why the `existing` check fails — Race Condition

```
Request A: SELECT → no pending payment found ✅
Request B: SELECT → no pending payment found ✅   ← A hasn't inserted yet

Request A: INSERT pending payment
Request B: INSERT pending payment   ← both pass the check

Request A: stripe.charge() → charged 💳
Request B: stripe.charge() → charged again 💳💳
```

Same race condition as Problem #2 — **check-then-act across two round trips.** Both requests pass the check before either inserts. The `existing` check is completely ineffective under concurrency.

---

### Why Pattern B is Particularly Dangerous

```
Request A → INSERT pending → stripe.charge() → network timeout to client
                                                (but Stripe DID charge)
Client thinks it failed → retries
Request B → INSERT pending → stripe.charge() → charged AGAIN 💳💳
```

The charge happened. The client never got the response. The retry creates a **second real charge** with no way to know the first succeeded.

This is the **partial failure problem** in distributed systems — the operation succeeded on the external system but the acknowledgement never reached you.

---

## ✅ The Fix — Idempotency Keys

The industry standard solution. Every payment request carries a **client-generated idempotency key** — a unique ID representing _this specific payment intent_, not the request.

```
Same idempotency key = same logical operation = execute only once
```

---

### Step 1 — Client sends an idempotency key

```python
# Client side — generate once per payment attempt
idempotency_key = str(uuid.uuid4())  # e.g. "a1b2c3d4-..."

# Retry as many times as needed — same key every time
response = api.post("/charge", {
    "user_id": user_id,
    "amount": amount,
    "idempotency_key": idempotency_key   # ← same key on every retry
})
```

---

### Step 2 — Server stores and deduplicates on the key

```python
def charge_user(user_id, amount, payment_method_id, idempotency_key):
    
    # Atomic upsert — insert only if key doesn't exist
    result = db.execute("""
        INSERT INTO idempotency_keys
            (key, user_id, amount, status, created_at)
        VALUES
            (?, ?, ?, 'processing', NOW())
        ON CONFLICT (key) DO NOTHING
    """, idempotency_key, user_id, amount)

    if result.rowcount == 0:
        # Key already exists — return stored result
        existing = db.query("""
            SELECT status, payment_id, response
            FROM idempotency_keys
            WHERE key = ?
        """, idempotency_key)

        if existing.status == 'processing':
            raise ConflictError("Payment in progress")  # concurrent retry
        
        return existing.response   # ← return exact same response as first time
    
    try:
        payment = db.execute("""
            INSERT INTO payments (user_id, amount, status)
            VALUES (?, ?, 'pending')
        """, user_id, amount)

        result = stripe.charge(payment_method_id, amount)

        response = {"status": "charged", "payment_id": payment.id}

        # Store the result against the key
        db.execute("""
            UPDATE idempotency_keys
            SET status = 'complete', payment_id = ?, response = ?
            WHERE key = ?
        """, payment.id, json.dumps(response), idempotency_key)

        return response

    except Exception as e:
        db.execute("""
            UPDATE idempotency_keys SET status = 'failed' WHERE key = ?
        """, idempotency_key)
        raise
```

Now the flow is:

```
Request A → INSERT idempotency_key → succeeds → stripe.charge() → store result
Request B → INSERT idempotency_key → ON CONFLICT DO NOTHING → rowcount=0
         → fetch stored result → return same response ← no second charge
```

---

### Step 3 — Handle the Partial Failure (Pattern B)

```
Request A → INSERT key → stripe.charge() → timeout ← client never got response
Request B → INSERT key → ON CONFLICT → key exists, status='processing'

What now?
```

This is the hardest case. The key exists but we don't know if Stripe charged or not. The solution is **reconciliation via Stripe's own idempotency:**

```python
result = stripe.charge(
    payment_method_id,
    amount,
    idempotency_key=idempotency_key   # ← pass YOUR key to Stripe too
)
```

Stripe stores results by idempotency key on their end. If you call them again with the same key, they return the **original result** instead of charging again. This is how all major payment processors work.

```
Request A → stripe.charge(key="a1b2c3") → charged → timeout
Request B → stripe.charge(key="a1b2c3") → Stripe returns original result → no double charge
```

---

## 📊 Full Picture — Layers of Protection

```
Layer 1: Client
─────────────────
Generate idempotency_key once per intent
Send same key on every retry

Layer 2: Your API
──────────────────
ON CONFLICT DO NOTHING on idempotency_keys table
Return stored response for duplicate keys
Pass key to downstream services

Layer 3: Payment Processor
───────────────────────────
Stripe/Braintree deduplicate on their end
Same key = same result returned, no new charge
```

Each layer handles a different failure mode. You need all three.

---

## 🧠 Idempotency Key Table Schema

```sql
CREATE TABLE idempotency_keys (
    key          TEXT PRIMARY KEY,         -- client generated UUID
    user_id      UUID NOT NULL,
    amount       INTEGER NOT NULL,
    status       TEXT NOT NULL,            -- processing / complete / failed
    payment_id   UUID,                     -- set after success
    response     JSONB,                    -- stored response to replay
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    expires_at   TIMESTAMPTZ DEFAULT NOW() + INTERVAL '24 hours'
);

CREATE INDEX ON idempotency_keys (expires_at);  -- for cleanup job
```

Keys expire after 24 hours — a retry after 24 hours is a genuinely new payment intent.

---

## ⚠️ Edge Case — Mismatched Parameters on Same Key

What if a client sends the same key but with a **different amount**?

```python
if existing.amount != amount or existing.user_id != user_id:
    raise BadRequestError(
        "Idempotency key reused with different parameters"
    )
```

Always validate that the key maps to the **same operation.** Different parameters = different intent = client bug, reject it loudly.

---

## 🔑 Key Takeaways

|Lesson|Rule|
|---|---|
|Check-then-act|Still broken here — `ON CONFLICT` makes the insert atomic|
|Idempotency key|Client owns the key, server deduplicates on it|
|Partial failure|Pass your key downstream — let Stripe deduplicate too|
|Store the response|Replay the exact same response — don't recompute|
|Key expiry|24h window — after that, a retry is a new intent|
|Parameter mismatch|Same key + different params = reject loudly|

---

## Next

- Companion deep dive: [[Problem 1 Companion]]
- Back to hub: [[Problems MOC]]
