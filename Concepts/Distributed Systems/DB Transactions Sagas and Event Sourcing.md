---
aliases:
  - explaining
tags:
  - concepts
  - distributed-systems
---

# DB Transactions, Sagas, and "Why Isn't a Saga DB with History Just Event Sourcing?"

This re-explains points 4 and 5 from your original doc more slowly, then answers your actual question head-on: **if each service in a Saga already has its own database, and that database can keep an audit/history table, what does Event Sourcing actually add?**

---

## 1. Database Transactions — the baseline you already understand intuitively

A transaction is just: _group several writes together so they either all land, or none do._

```sql
BEGIN;
  UPDATE wallets   SET balance = balance - 100 WHERE user_id = 1;
  INSERT INTO orders (user_id, amount) VALUES (1, 100);
  UPDATE inventory SET stock = stock - 1 WHERE product_id = 42;
COMMIT;
```

If the process crashes after the second statement, Postgres throws the whole thing away on restart — you never see a wallet debited with no matching order. That's it. That's the entire idea.

**The one thing people miss:** a transaction gives you _atomicity_, not _idempotency_. If the user double-clicks "Buy Now" and your server runs this whole block twice, you get two orders and two debits — perfectly atomic, twice. That's why you still want a unique `order_request_id` column even in a single-database system, so the second attempt hits a constraint violation instead of silently duplicating.

**When this stops being enough:** the moment `wallets`, `orders`, and `inventory` live in _different databases owned by different services_. `COMMIT` only has authority over the database it's talking to — it has zero power over a database three network hops away. That's the entire reason Section 2 exists.

---

## 2. The Saga Pattern — rebuilding "all or nothing" by hand

Since you can't get one `COMMIT` across four services, a Saga fakes it: it runs the four steps one at a time, and for every step it also defines the "undo" for that step.

```
Step 1: Charge Payment       →  Undo: Refund Payment
Step 2: Deduct Inventory     →  Undo: Re-add Inventory
Step 3: Create Shipment      →  Undo: Cancel Shipment
```

If Step 3 fails, you don't roll back a transaction (there isn't one) — you manually run the undos for everything that already succeeded, in reverse order. That's the whole trick: **a real rollback is replaced by a chain of hand-written compensating actions.**

### Two ways to wire this together

**Orchestration** — one coordinator calls each service in order and decides what to undo on failure. Easy to look at and reason about; the coordinator is now a thing that can fail or become a bottleneck.

**Choreography** — no coordinator. Each service listens for the previous service's event and reacts on its own. Nothing to become a bottleneck, but there's no single place you can look at to see "where is order #482 right now" — you have to reconstruct that by tracing events across services.

### The dual-write problem (the subtle part)

When PaymentService finishes charging a card, it needs to do two things: save `status = charged` to its own table, _and_ tell Kafka so InventoryService hears about it. Those are two separate writes to two separate systems — the process can crash between them, and now your DB and Kafka disagree about what happened.

**Fix — the Outbox pattern:** instead of writing to Kafka directly, write a row into an `outbox` table _inside the same DB transaction_ as the business update.

```sql
BEGIN;
  UPDATE payments SET status = 'charged' WHERE id = 1;
  INSERT INTO outbox (event, payload) VALUES ('PaymentTaken', '{...}');
COMMIT;
```

A separate relay process reads unpublished rows from `outbox` and pushes them to Kafka, marking them as sent. Because both writes are in one local transaction, they can never split — if the business row was written, the event will eventually go out; if it wasn't, the event never gets created in the first place.

---

## 3. Your actual question: Event Store vs. "a Saga DB that also keeps history"

You're right about the setup: in Event Sourcing, each service still has its own event store — nobody shares one giant event log, same as Saga where each service owns its own DB. So on the surface it looks like you could just add a `payments_history`audit table next to your normal `payments` table in a Saga system and get the same thing "for free." Here's specifically where that intuition breaks, and where it doesn't.

### Where you're right

Yes — you absolutely _can_ bolt an audit-log table onto a Saga-style service, and plenty of production systems do exactly that:

```sql
-- normal current-state table
payments (id, status, amount)

-- bolted-on history, updated alongside it
payments_history (id, payment_id, old_status, new_status, changed_at)
```

Nothing about the Saga pattern forbids this. If that's all you need, this is a perfectly fine, much simpler solution than adopting full Event Sourcing.

### Where the two designs actually diverge

The real difference isn't "does history exist" — it's **where the current state lives, and what guarantees you get as a side effect of that.**

||Saga service + bolted-on audit table|Event Sourcing|
|---|---|---|
|Where is "the truth" stored?|The `payments` table (current state). The history table is a _side record_ of what happened to it.|The event log itself. There is no separate current-state table at all.|
|Can state and history disagree?|**Yes — this is the real risk.** Someone writes `UPDATE payments SET status=...` and forgets (or a bug skips) the matching insert into `payments_history`. Now your audit trail is silently wrong, and nothing detects it.|**Structurally impossible.** `status` isn't stored anywhere except as the result of replaying events. There's no second place it could be written to and forget — the "current state" is just a query over the log.|
|Can you rebuild state from scratch?|No. If you deleted the `payments` table, the history table doesn't give you a way to regenerate it automatically — you'd have to hand-write replay logic that was never part of the design.|Yes, by definition. Deleting every projection/read-model/snapshot and replaying the log from event 1 _is_how you get current state — that code path is exercised constantly, not a theoretical backup plan.|
|Is the history a first-class citizen or an add-on?|Add-on. It was a deliberate extra table someone had to design and remember to keep in sync.|The event log **is** the schema. There's nothing to "remember" — you can't build the system without it.|

### The one-sentence version

A Saga DB with an audit table gives you a _record_ of what happened, sitting next to the current state, that a bug can cause to drift out of sync. Event Sourcing gives you _only_ the record — current state is a derived, disposable, always-consistent-with-history view computed from it. The history isn't a feature you added; it's the only thing that exists.

### When that difference actually matters

- If you only need "let's be able to look up what happened to order #482 for support/debugging" → a bolted-on audit table is genuinely fine, and much cheaper to build and operate.
- If you need a legally airtight audit trail (finance/healthcare), or multiple independent services reacting to the same facts, or the ability to add a brand-new feature that needs _past_ data you never explicitly stored for it (e.g., "what would revenue have looked like under the new pricing rule, retroactively?") → that's when the "state can never drift from history because there is no separate state" guarantee of Event Sourcing starts pulling its weight over an audit table.

---

## Quick recap

- **Transaction:** free atomicity, but only inside one DB, and you still need idempotency keys for retries.
- **Saga:** manual "undo chain" across services, fixing the dual-write gap between your DB and your event bus via the Outbox pattern.
- **Event Sourcing vs. Saga+audit-table:** not "one has history and one doesn't" — it's whether current state and history are two things that _could_ drift apart (Saga+audit) or structurally _can't_ (Event Sourcing, because there's only one thing: the log).