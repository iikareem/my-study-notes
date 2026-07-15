---
tags:
  - consistency
  - distributed-systems
  - idempotency
  - interview
  - messaging
  - microservices
  - outbox
  - problem-7
  - problems
  - saga
  - system-design
---

# Problem 7 — "Our microservices have data that disagrees with each other and nobody knows who's right"

**Domain:** Distributed Systems & Data Consistency  
**Topics:** saga · transactional outbox · idempotency · consistency

**In this folder:** scenario (this note) · [[Problem 7 Companion]]

## 🧩 The Scenario

You work at a fintech company. The architecture looks like this:

```
Order Service      → owns orders table
Inventory Service  → owns inventory table
Payment Service    → owns payments table
Notification Service → sends emails/SMS
```

Each service has its **own database.** They communicate via REST calls. The checkout flow looks like this:

```python
# Order Service — checkout endpoint
def checkout(user_id, cart):

    # Step 1 — create order
    order = order_db.execute("""
        INSERT INTO orders (user_id, status, total)
        VALUES (?, 'pending', ?)
    """, user_id, cart.total)

    # Step 2 — reserve inventory
    response = requests.post("http://inventory-service/reserve", {
        "order_id": order.id,
        "items":    cart.items
    })

    if response.status_code != 200:
        order_db.execute("UPDATE orders SET status='failed' WHERE id=?", order.id)
        raise CheckoutError("Inventory unavailable")

    # Step 3 — charge payment
    response = requests.post("http://payment-service/charge", {
        "order_id": order.id,
        "amount":   cart.total,
        "user_id":  user_id
    })

    if response.status_code != 200:
        requests.post("http://inventory-service/release", {"order_id": order.id})
        order_db.execute("UPDATE orders SET status='failed' WHERE id=?", order.id)
        raise CheckoutError("Payment failed")

    # Step 4 — confirm order
    order_db.execute("UPDATE orders SET status='confirmed' WHERE id=?", order.id)

    # Step 5 — notify user
    requests.post("http://notification-service/send", {
        "user_id": user_id,
        "message": "Order confirmed!"
    })

    return {"order_id": order.id, "status": "confirmed"}
```

Three weeks after launch you discover:

- **Scenario A** — Payment was charged but inventory was never reserved — items oversold
- **Scenario B** — Inventory reserved, payment charged, but order status still says `pending`
- **Scenario C** — User received "Order confirmed" email but payment actually failed
- **Scenario D** — Payment service timed out, you released inventory and marked order failed — but the payment actually went through on Stripe's side. Customer was charged for a failed order.

Your data across services **fundamentally disagrees** and you have no way to know which service is the source of truth.

---

## 🤔 Think About It First

Why can't you just wrap this in a database transaction? What is the fundamental guarantee you've lost by having separate databases per service? And what are the two main architectural patterns that solve this — and what does each trade off?

---

## 🔍 Root Cause Analysis

### The Fundamental Problem — No Distributed Transaction

In a monolith with one database:

```sql
BEGIN;
  INSERT INTO orders ...
  UPDATE inventory ...
  INSERT INTO payments ...
COMMIT;  -- all succeed or all rollback
```

**ACID** gives you atomicity across all operations. Either everything commits or nothing does.

With microservices and separate databases:

```
Order DB    ← ACID within this DB only
Inventory DB ← ACID within this DB only
Payment DB  ← ACID within this DB only

Across all three? ← NO GUARANTEE WHATSOEVER
```

You've traded **consistency for autonomy.** Each service owns its data — but now a failure anywhere in the chain leaves data in an inconsistent state across services with no automatic rollback.

---

### Why Each Scenario Happens

**Scenario A — Payment charged, inventory not reserved:**

```
Step 2: inventory reserve → succeeds ✅
Step 3: payment charge   → network timeout after Stripe processes it
→ your code sees timeout → thinks it failed
→ releases inventory, marks order failed
→ but Stripe DID charge the card 💳
```

**Scenario B — Order stuck as pending:**

```
Step 3: payment → succeeds ✅
Step 4: UPDATE orders SET status='confirmed'
→ order_db crashes mid-write
→ payment exists, inventory reserved, but order = 'pending' forever
```

**Scenario C — Confirmation email sent, payment failed:**

```
Step 3: payment → returns 500
→ your rollback code runs
→ but notification service was called BEFORE you checked payment response
→ email already sent ✅ payment failed ❌
```

Actually look at the code — notification is Step 5, after payment. But what if the rollback itself fails? The inventory release HTTP call fails — so you never reach the order status update — but the payment has already failed. Inconsistency from rollback failure.

**Scenario D — The partial failure / phantom payment:**

This is the hardest case. The payment service HTTP call times out — but timeout means **unknown**, not failed:

```
Your code:         timeout → assume failure → rollback
Stripe's reality:  charge succeeded → money taken from card
```

You have no way to distinguish "payment failed" from "payment succeeded but you never got the response."

---

## ✅ Solution 1 — The Saga Pattern

A **Saga** is a sequence of local transactions where each step publishes an event, and each step has a **compensating transaction** that undoes it if something fails later.

```
FORWARD TRANSACTIONS          COMPENSATING TRANSACTIONS
────────────────────          ─────────────────────────
1. create_order          ←→   cancel_order
2. reserve_inventory     ←→   release_inventory
3. charge_payment        ←→   refund_payment
4. confirm_order         ←→   (terminal — no compensation needed)
5. send_notification     ←→   send_cancellation_notification
```

There are two ways to coordinate a saga:

---

### Saga Style A — Choreography (Event-Driven)

Each service listens for events and reacts independently:

```python
# Order Service
def checkout(user_id, cart):
    order = create_order(user_id, cart)
    event_bus.publish("order.created", {
        "order_id": order.id,
        "items":    cart.items,
        "total":    cart.total,
        "user_id":  user_id
    })
    return {"order_id": order.id, "status": "pending"}

# Inventory Service — listens for order.created
def on_order_created(event):
    try:
        reserve_items(event.order_id, event.items)
        event_bus.publish("inventory.reserved", {"order_id": event.order_id})
    except OutOfStockError:
        event_bus.publish("inventory.failed", {"order_id": event.order_id})

# Payment Service — listens for inventory.reserved
def on_inventory_reserved(event):
    try:
        charge(event.order_id, event.total)
        event_bus.publish("payment.charged", {"order_id": event.order_id})
    except PaymentError:
        event_bus.publish("payment.failed", {"order_id": event.order_id})

# Order Service — listens for payment.failed → compensate
def on_payment_failed(event):
    event_bus.publish("inventory.release_requested", {"order_id": event.order_id})
    update_order_status(event.order_id, "failed")
```

```
Flow:
order.created
    → inventory.reserved
        → payment.charged
            → order.confirmed
                → notification.sent

Failure flow:
order.created
    → inventory.reserved
        → payment.failed
            → inventory.release_requested  ← compensation
            → order.cancelled
                → notification.cancellation_sent
```

**Tradeoff:** No central coordinator — services are decoupled. But the flow is hard to visualize, debug, and reason about. When something goes wrong you have to trace events across multiple services.

---

### Saga Style B — Orchestration (Central Coordinator)

One **Saga Orchestrator** service drives the entire flow and tells each service what to do:

```python
class CheckoutSaga:

    def execute(self, user_id, cart):
        saga_id = uuid.uuid4()
        state   = SagaState(saga_id, steps_completed=[])

        try:
            # Step 1
            order = self.order_client.create_order(user_id, cart)
            state.record("order_created", order_id=order.id)

            # Step 2
            self.inventory_client.reserve(order.id, cart.items)
            state.record("inventory_reserved")

            # Step 3
            self.payment_client.charge(order.id, cart.total, user_id)
            state.record("payment_charged")

            # Step 4
            self.order_client.confirm(order.id)
            state.record("order_confirmed")

            # Step 5
            self.notification_client.send(user_id, "Order confirmed!")
            state.record("notification_sent")

        except Exception as e:
            self.compensate(state)   # ← run compensations in reverse
            raise

    def compensate(self, state):
        # Reverse order — undo what was done
        if "payment_charged" in state.steps:
            self.payment_client.refund(state.order_id)

        if "inventory_reserved" in state.steps:
            self.inventory_client.release(state.order_id)

        if "order_created" in state.steps:
            self.order_client.cancel(state.order_id)

        self.notification_client.send(state.user_id, "Order cancelled, refund initiated")
```

**Tradeoff:** Easy to visualize and debug — one place shows the full flow. But the orchestrator becomes a central dependency. If it goes down, no checkouts process.

---

### The Compensation Problem — Not All Steps Are Reversible

Compensating transactions are **not rollbacks.** A rollback never happened. A compensation is a new forward action that corrects the effect:

```
Payment charged → compensation is "refund"
                  not "un-charge" (that's impossible)

Email sent      → compensation is "send cancellation email"
                  not "un-send" (that's impossible)

Inventory reserved → compensation is "release reservation"
                     this one actually is reversible ✅
```

This means your system will have **temporary inconsistency** — the window between a step failing and the compensation completing. You must design your UI and business logic to handle this gracefully.

---

## ✅ Solution 2 — The Outbox Pattern + Eventual Consistency

From Problem 3 you already know the outbox pattern. Here it solves the cross-service consistency problem differently — instead of compensating, you **guarantee the event is delivered and processed:**

```python
# Order Service — checkout
def checkout(user_id, cart):
    with order_db.transaction():
        order = order_db.execute("INSERT INTO orders ...")

        # Write intent atomically with the order
        order_db.execute("""
            INSERT INTO outbox (event_type, payload)
            VALUES ('checkout.initiated', ?)
        """, json.dumps({
            "order_id": order.id,
            "items":    cart.items,
            "total":    cart.total,
            "user_id":  user_id
        }))

    # Outbox poller publishes → other services consume
    # Each service processes and writes their own outbox entry
    # Chain continues until all services have processed
```

The difference from the saga:

```
Saga:               tries to do everything now, compensates on failure
Eventual consistency: guarantees everything will happen eventually
                      accepts temporary inconsistency
                      no compensation needed if delivery is guaranteed
```

---

## 📊 Saga vs Eventual Consistency

||Saga|Eventual Consistency|
|---|---|---|
|Consistency|Compensates to stay consistent|Temporarily inconsistent, converges|
|Complexity|High — must design compensations|Medium — design for async|
|Failure handling|Explicit compensation logic|Retry until success|
|Reversible steps|Must be reversible or compensated|Must be idempotent|
|User experience|Can give synchronous response|Usually async, need status polling|
|Best for|Payments, critical flows|Notifications, analytics, search index|

---

## 🧠 The Hardest Part — Phantom Payments (Scenario D)

No pattern fully solves the phantom payment problem automatically. The answer is **idempotency + reconciliation:**

```python
# Payment service — idempotent charge
def charge(order_id, amount, user_id):
    # From Problem 1 — idempotency key
    result = stripe.charge(
        amount,
        idempotency_key=f"order-{order_id}"  # ← same key = same result
    )
    return result

# Reconciliation job — runs every hour
def reconcile_payments():
    # Find orders marked failed but with a payment record
    suspicious = db.query("""
        SELECT o.id, p.stripe_charge_id
        FROM orders o
        JOIN payments p ON p.order_id = o.id
        WHERE o.status = 'failed'
          AND p.status = 'charged'
    """)

    for order in suspicious:
        # Verify against Stripe
        charge = stripe.retrieve(order.stripe_charge_id)

        if charge.status == 'succeeded':
            # Money was taken — either confirm or refund
            alert_team(order)   # human decision needed
```

Phantom payments require **human review or automated reconciliation** — there's no purely technical solution that's safe enough for money movement.

---

## 🔑 Key Takeaways

|Lesson|Rule|
|---|---|
|Distributed transactions|Don't exist across microservices — ACID stops at the DB boundary|
|Saga pattern|Sequence of local transactions with explicit compensations for failures|
|Choreography vs orchestration|Choreography = decoupled but hard to debug. Orchestration = clear but central dependency|
|Compensating transactions|Not rollbacks — new forward actions that correct prior effects|
|Eventual consistency|Accept temporary inconsistency, guarantee convergence via retries|
|Phantom payments|Idempotency keys + reconciliation jobs — no purely automatic solution|
|Timeout = unknown|A timeout from an external service never means failed — means unknown|

---

## Next

- Companion deep dive: [[Problem 7 Companion]]
- Back to hub: [[Problems MOC]]
