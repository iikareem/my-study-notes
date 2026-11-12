# CDC: The Right Tool for the Wrong Job

This is a really sharp observation, and it cuts to something deeper than just "use the right tool." Let me unpack the idea with you.

---

## What CDC Actually Is

CDC is fundamentally a **data synchronization primitive**. It answers one question:

> _"What bytes changed in this table?"_

That's it. It doesn't know if the change came from a user clicking "Pay Now", an admin running a repair script, a database migration, a developer testing in production (we've all seen it), or a rollback.

The replication log is intentionally **context-free**. That's a feature for its intended use case, and a footgun everywhere else.

---

## The Core Confusion: Events vs. State Changes

The root of the misuse is conflating two different things:

|Concept|Question it answers|Example|
|---|---|---|
|**State change**|_What is different?_|`payment_status = PAID`|
|**Business event**|_Why did this happen?_|`OrderPaymentConfirmed`|

CDC captures state changes. Business logic should react to **events**. These _look_ the same in the happy path, which is exactly why the mistake is so seductive.

---

## Why the Happy Path Lies to You

The fulfillment example you gave is perfect. Let's trace through it:

```
Normal flow:
  User pays → payment_status = PAID → CDC fires → fulfillment kicks off ✅

Admin repair flow (months later):
  Admin fixes stale record → payment_status = PAID → CDC fires → fulfillment kicks off again 💥
```

The CDC system behaved _perfectly correctly_ both times. The problem is that you asked the wrong question. You asked _"did this column change?"_ when you meant _"did a customer complete payment?"_

These are the same event exactly once: the first time, in the normal path. After that, they diverge.

---

## The Deeper Problem: Implicit vs. Explicit Intent

When you use CDC as a business trigger, you're making intent **implicit**. The system has no way to distinguish:

- Legitimate payment confirmation
- Admin correction
- Data migration
- Bug fix backfill
- Accidental update

When you use an **outbox, job queue, or direct call**, you're making intent **explicit**. The fulfillment service gets called _because the application decided fulfillment should happen_ — not because a column changed.

This is a huge difference in **who owns the decision**:

- **CDC approach**: The database owns the decision (passively, by logging a change)
- **Explicit approach**: The application owns the decision (actively, by enqueuing an event)

---

## Where CDC Genuinely Shines

The valid uses are all about **data replication without semantic meaning**:

```
Postgres → Elasticsearch     (search index sync)
Postgres → Snowflake         (analytics warehouse)
Postgres → Redis             (cache warming)
Old DB   → New DB            (migration)
```

Notice the pattern: the downstream system just wants _a copy_. It doesn't care _why_ the data changed. An Elasticsearch index doesn't need to know if an order was paid by a customer or corrected by an admin — it just needs the latest state.

That's the key litmus test:

> **Does the downstream system care _why_ the change happened, or just _what_ the current state is?**

If it only cares about _what_ — CDC is great. If it cares about _why_ — CDC is wrong.

---

## A Useful Mental Model

Think of CDC like a **security camera**. It faithfully records everything that happens. It doesn't interpret. It doesn't judge. It doesn't know if the person who walked in was a customer, an employee, or someone who shouldn't be there.

Business logic is the **security guard** — it needs context, intent, and the ability to make decisions. You wouldn't replace your security guard with a camera and call it a day.

---

## What to Use Instead

When you need to trigger business logic on a state transition, the application layer needs to be explicit about it:

- **Outbox pattern** — write the event to a transactional outbox table alongside the state change, atomically. A relay picks it up and publishes it.
- **Job queue** — enqueue a job explicitly when the business event occurs (not when the DB row changes).
- **Direct call** — if in the same transaction boundary, just call the downstream dependency.

All three share the same property: **the code that understands business context makes the decision**, not the replication log.

---

What aspect do you want to dig into further — the outbox pattern specifically, how to migrate away from CDC-as-trigger in an existing system, or something else?


