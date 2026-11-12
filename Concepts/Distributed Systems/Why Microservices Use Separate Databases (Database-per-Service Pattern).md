# Why Microservices Use Separate Databases (Database-per-Service Pattern)

You're asking the right question. The cost is real — distributed transactions, complex reads, eventual consistency. So why pay it?

---

## The Core Problem with a Shared Database

Even with separate tables, a shared database creates **invisible coupling** between services. Here's why that's dangerous:

### 1. Schema Coupling

Even if every service strictly touches only its own tables, a migration can still break other services through shared database-level mechanics:

**Migration locks.** In most databases, adding a column or an index on a large table acquires a lock on that table for the duration. During that window, the database's connection pool and I/O are saturated — every other service querying _their own_ unrelated tables experiences slowdowns or timeouts.

**Sequence and enum types.** Postgres enums, shared sequences, or custom types live at the database level, not the table level. One service changing a shared type breaks any other service that references it, even indirectly.

**The migration coordination problem.** Migrations need to run against the shared database in a controlled order. Now every team must queue their deploys behind a shared migration pipeline. One team's bad migration (a missing index, a full-table rewrite) blocks everyone else's release.

**Connection pool exhaustion.** All services share the same database connection pool. If the Orders service runs a heavy migration that spikes connections, the Users service starts getting "too many connections" errors — on its own tables, doing nothing wrong.

> The break doesn't come from touching each other's tables. It comes from the fact that **the database itself is a shared, contended resource** — locks, connections, I/O, and types are all global to it.

> Services become **independently deployable** in theory but **tightly coupled** in practice.

### 2. The "Shortcut" Problem

When developers see all the data sitting in one database, they take shortcuts:

```sql
-- Order service developer writes this "quick join"
SELECT o.*, u.credit_limit, i.stock_level
FROM orders o
JOIN users u ON o.user_id = u.id        -- Users table (not yours!)
JOIN inventory i ON o.item_id = i.id    -- Inventory table (not yours!)
WHERE o.status = 'pending';
```

Now the Orders service **secretly depends** on the Users and Inventory schema. Nobody knows until someone changes a column name and production breaks at 3 AM.

### 3. No Independent Scaling

If `Inventory` is under heavy read load (Black Friday), you can't scale just its database. You have to scale the entire shared database, paying for resources that `Users` and `Orders` don't need.

### 4. No Technology Freedom

With a shared database, every service is locked to the same engine. But different problems suit different storage engines:

|Service|Ideal Storage|Why|
|---|---|---|
|User Sessions|Redis|Fast key-value, TTL support|
|Product Catalog|Elasticsearch|Full-text search|
|Financial Transactions|PostgreSQL|ACID, strong consistency|
|Social Graph|Neo4j|Graph traversal|
|Analytics|Columnar (Redshift)|Aggregation performance|

A shared relational database forces all of them into the same box.

### 5. Blast Radius of Failure

With a shared database, a bad query from **one service** can:

- Lock tables used by other services
- Exhaust connection pools shared across all services
- Cause cascading failures across the entire system

Even your hot standby replica doesn't solve this — the bad query hits the replica too, and a connection pool exhaustion on the primary takes everything down together.

---

## What Separate Databases Actually Buy You

### Independent Deployability (for real)

Each team owns their schema entirely. They can:

- Run migrations without coordinating with other teams
- Rename columns, change types, drop tables — freely
- Deploy at their own pace

This is the **whole point** of microservices: organizational autonomy at scale.

### Independent Scaling

```
High traffic on Orders?
  → Scale Orders DB only
  → Users DB untouched
  → Inventory DB untouched
  → Cost stays proportional to actual load
```

### Fault Isolation

If the `Recommendations` database goes down, `Orders` and `Payments` keep running. The blast radius of any failure is contained to one service.

### Schema Evolution Without Fear

No migration coordination. No "who owns this table?" debates. No accidental cross-service joins silently embedded in production code.

---

## "But Distributed Transactions Are a Nightmare"

Yes. That's the honest trade-off. Here's how the industry handles it:

### Saga Pattern (for writes)

Instead of a distributed ACID transaction, you break it into a sequence of local transactions with compensating actions on failure:

```
1. Order Service  → Creates order (status: PENDING)
2. Payment Service → Charges card
   ❌ If fails → Order Service compensates → marks order FAILED
3. Inventory Service → Reserves stock
   ❌ If fails → Payment Service compensates → issues refund
4. Order Service  → Marks order CONFIRMED
```

Each step is a local transaction. Failure triggers a rollback chain. No distributed lock held across services.

### CQRS + Event Sourcing (for reads)

Instead of joining across databases at query time, you **pre-build read models**:

```
Orders DB  ──┐
Users DB   ──┼──► Event Stream ──► Read Model DB (pre-joined view)
Inventory  ──┘                         └── Optimized for UI queries
```

The read model is eventually consistent but fast and simple to query.

### API Composition (for simple reads)

The API Gateway or a BFF (Backend for Frontend) calls multiple services and joins the results in memory:

```
Client Request: "Show me order #123 with user info and item details"

API Gateway:
  → GET /orders/123       (Orders Service)
  → GET /users/456        (Users Service)
  → GET /items/789        (Inventory Service)
  → Merge & return
```

Slower than a SQL join, but the boundary is explicit and owned.

---

## The Real Reason: Conway's Law

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of those organizations." — Melvin Conway

A shared database with shared tables means **teams must coordinate constantly**. At 5 engineers, that's fine. At 500 engineers across 20 teams, it becomes a bottleneck that slows everyone down.

Separate databases enforce a **team boundary in the architecture**. The complexity of distributed transactions is the price you pay for organizational independence.

---

## When a Shared Database Is Actually Fine

If you have:

- A small team (< 10 engineers)
- Low traffic
- A monolith or a majestic monolith

...then separate databases add complexity with no real benefit. Start with one database, separate tables, and **clean service boundaries in code**. Extract when the pain of coordination actually shows up.

---

## Summary

|Problem|Shared DB|Separate DBs|
|---|---|---|
|Schema coupling|❌ Always present|✅ Eliminated|
|Independent scaling|❌ Scale all or nothing|✅ Scale per service|
|Technology choice|❌ One engine for all|✅ Right tool per service|
|Blast radius|❌ One bad query hits all|✅ Isolated per service|
|Read complexity|✅ Simple SQL joins|❌ API composition / CQRS|
|Write complexity|✅ ACID transactions|❌ Sagas / eventual consistency|
|Team autonomy|❌ Constant coordination|✅ Independent teams|

The design is taken not because distributed transactions are easy — they're not — but because **at scale, the coordination cost of a shared database exceeds the complexity cost of distribution**.