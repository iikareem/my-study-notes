
A summary of the approach: understand the domain, organize the monolith, separate data ownership, and extract services only when there is a reason.

---

## 1. The Big Picture

```
Monolith -> Modular Monolith -> (identify boundaries & dependencies) -> Extract services
```

1. **Start with a modular monolith.** Boundaries are not known yet, and a wrong boundary is a cheap refactor inside a monolith but an expensive migration between services.
2. **Split by business capability, not by technical layer.** Ordering, Billing, Inventory, Shipping. Avoid "UI service / API service / DB service".
3. **Discover boundaries with DDD** (Ubiquitous Language, Bounded Contexts, Context Map).
4. **Map dependencies** (who calls whom, sync or async, how often).
5. **Enforce boundaries inside the monolith** before splitting.
6. **Extract a service only when there is a concrete reason.**

---

## 2. Finding Boundaries with DDD

|Concept|Meaning|Example|
|---|---|---|
|**Ubiquitous Language**|The exact words domain experts use, shared by code and business|If the same word means different things to different people, that is a boundary signal|
|**Bounded Context**|A boundary inside which one model and one language are consistent|"Customer" in Sales (lead + pipeline), Billing (payer + invoice address), Support (person + tickets) are three models|
|**Context Map**|How contexts relate: upstream/downstream, Customer-Supplier, Shared Kernel, Anti-Corruption Layer, Published Language|Shows dependencies|
|**Event Storming**|Workshop placing domain events on a wall (`OrderPlaced`, `PaymentCaptured`)|Clusters reveal contexts, arrows reveal coupling|

### Heuristics for judging a boundary

- **Cohesion:** things that change together live together.
- **Change test:** a typical feature should touch one module, not five.
- **Data ownership test:** one clear source of truth per piece of data.
- **Transaction test:** a strong transaction across two modules suggests they belong together.
- **Team test (Conway's Law):** one small team can own it end to end.

### Common mistakes

- Distributed monolith (services that must deploy together or chain synchronous calls)
- Entity services (`UserService`, `OrderService` as CRUD wrappers)
- One giant shared model (the god `Customer`)
- Splitting too early or too fine
- Code names that do not match the business language

---

## 3. Enforcing Boundaries Inside the Monolith

- One module per bounded context, with a small public API and hidden internals.
- No cross-module joins or shared tables; each module owns its schema.
- Modules communicate via interfaces or in-process domain events.
- Add architecture tests (ArchUnit, NetArchTest, dependency-cruiser, Spring Modulith) so violations fail the build.

### When to extract a service (good reasons)

- Independent scaling needs
- Different release cadence or separate team
- Different technology or compliance requirements
- Fault isolation

"Microservices are trendy" is not a reason.

---

## 4. Choosing the Decomposition Approach

|Starting point|Best approach|
|---|---|
|Clean modular monolith (clear API, owns its data)|**Extract the module directly** into its own repo/service|
|Messy monolith, good tests, clear target contexts|**Tactical forking**|
|Messy monolith, weak tests, unclear boundaries|**Organize and add tests first** (refactor into modules), then choose one of the above|

### 4.1 Direct extraction (clean module)

Moving code is the easy part. You still must:

1. Replace in-process calls with network calls (timeouts, retries, versioned contracts).
2. Separate the database (the module should already own its tables).
3. Replace in-process events with a broker (ordering, duplicates, outbox pattern).
4. Give it its own CI/CD, monitoring, logging, tracing.
5. Route traffic gradually (strangler fig pattern).

### 4.2 Tactical forking (messy monolith)

Copy the **entire repository** once per target service, then **delete** what each does not need.

```
Monolith: [Orders | Billing | Shipping | Shared utils]

Fork 1 -> delete Orders, Shipping  -> [Billing  | needed utils]
Fork 2 -> delete Billing, Shipping -> [Orders   | needed utils]
Fork 3 -> delete Orders, Billing   -> [Shipping | needed utils]
```

**Why it helps**

- You do not need to know all dependencies up front; the compiler and tests tell you as you delete.
- Every step starts from a working application.
- Fast to start.

**Requirements and risks**

- Needs decent tests and a clear idea of what each fork should become.
- The fork is **temporary**: stopping halfway leaves bloated copies full of dead code.
- Copying code does not separate the database.
- Bug fixes must be applied to the original and the forks until the monolith is retired.

**Related meaning:** tactical forking is also used for two teams colliding on one shared module. Copy it, work independently, then decide to merge, keep separate, or delete. A fork that never merges back is a sign you found two bounded contexts.

---

## 5. Database Decomposition

**Principle:** separate _ownership_ first (logical), and move data _physically_ last.

### Step 1: Map who touches what

- Inspect code (queries, ORM entities, stored procedures).
- Inspect the database (query logs, foreign keys, views, triggers).
- Build a **table-to-module matrix**.

### Step 2: Assign ownership

- The owner is the module that **writes** the table and **enforces its rules and validation**.
- Everyone else calls the owner (API or events) instead of touching the table.
- Hard, shared tables are **left until last**.

### Step 3: Separate logically (same physical database)

1. Give each module its own **schema** and its own **DB user** with permissions only on that schema.
2. Remove cross-module access.
3. Replace direct access with an **API call**, **events plus a local copy**, or (temporarily) a **read-only view**.
4. Break cross-module foreign keys and joins (store the other module's ID as a plain value).
5. Replace cross-module transactions with **sagas / outbox** and eventual consistency.
6. **Test in staging** with the restricted users and the normal test suite. A permission error reveals a hidden dependency. Also check query logs.

### Step 4: Move physically (only with a real reason)

1. Create the new database and replicate the schema.
2. Copy existing data.
3. Keep in sync (CDC such as Debezium, or replication).
4. Verify (row counts, checksums, samples).
5. Switch reads, then writes.
6. Keep old tables for rollback, then drop them.

### Why schemas before physical separation

||Separate schema|Separate physical DB|
|---|---|---|
|Rollback|Easy|Hard|
|Cross-module transactions|Still possible while transitioning|Gone|
|Cross-module joins|Still work|Break immediately|
|Infrastructure|None|Servers, backups, monitoring, networking|

Hidden dependencies show up as **permission errors in staging** instead of **production failures**. You can change your mind cheaply, work gradually, and you may find the logical separation is all you ever need.

### Common traps

- Shared reference data (countries, currencies): copy it or have one owner publish changes.
- Reporting queries that join everything: use a separate read store fed by events.
- Stored procedures, triggers, views with hidden cross-module logic.
- Auto-increment IDs shared across modules: use UUIDs or separate ranges.
- Dual-write drift: prefer CDC.

---

## 6. Worked Example: Shared `products` Table

```sql
products (
  id, sku, name, description, images, category, is_active,      -- Catalog
  price, tax_code,                                              -- Pricing
  stock_quantity, reserved_quantity, warehouse_location, weight -- Inventory
)
```

**Who writes each column?** Catalog (name, description, images, price) and Inventory (stock fields). Orders only _reads_, so it should own nothing.

**Split by owner:**

```sql
catalog.products   (product_id, sku, name, description, images, category, price, tax_code, is_active)
inventory.stock_items (product_id, sku, stock_quantity, reserved_quantity, warehouse_location, weight)
```

Same `product_id` in both, with no cross-schema foreign key.

### Ownership rules for shared data

|Question|Answer|
|---|---|
|Who may change it?|One owner only|
|Who just needs it?|Read-only copy or API call|
|Must it be frozen in time?|Snapshot (e.g. price and name on an order line)|
|Is there a cross-module transaction?|Saga or outbox|
|Is there a cross-module join?|API composition or a read model|

### Local copy vs API call (Orders needs product name and price)

||Local copy (event-updated)|API call to owner|
|---|---|---|
|How|Catalog publishes `ProductPriceChanged`; Orders updates `orders.product_snapshot`|Orders calls `GET /catalog/products/{id}`|
|Freshness|Slightly stale|Always current|
|If owner is down|Still works|Fails|
|Complexity|More (events, extra table)|Less|
|Best for|Hot, frequently read data|Rare or must-be-exact reads|

Common mix: local copy for browsing, one API check at the final confirmation step. In both cases, the **order line stores its own snapshot** so old orders never change when prices change.

---

## 7. Cross-Module Transactions: Saga Pattern

|Style|How it works|Best for|
|---|---|---|
|**Orchestration**|A central coordinator calls each step and triggers compensation on failure (Temporal, Camunda, AWS Step Functions)|Many steps, branching, need visibility|
|**Choreography**|No coordinator; services react to events (`OrderPlaced` -> `StockReserved` -> `PaymentCaptured`)|2 to 3 simple steps|

Pair either with the **outbox pattern** to publish events reliably.

### What lives where (with a workflow engine)

|Workflow engine|Your services|
|---|---|
|Order of steps, retries, timeouts|Validation and business rules|
|Saga state tracking|Each service's data and models|
|Triggering compensation|The actions and the compensation logic|

Requirements: every step needs a compensation, steps must be **idempotent**, and contracts must be versioned.

### AWS Step Functions setup

- Each team deploys its own Lambdas from its own repo, with its own IAM role limited to its own data.
- One team (the owner of the business process, e.g. Orders) writes the state machine: sequence, retries, compensation.
- The state machine role only has `lambda:InvokeFunction` on the needed Lambdas.
- Teams agree on each Lambda's **input and output**; this is the only shared contract. Version it (e.g. Lambda aliases).
- Use **Standard workflows** for sagas. Use `.waitForTaskToken` for long waits.

### Multiple workflows (one per team)

|Option|How|Best when|
|---|---|---|
|**Parent calls children**|Parent state machine starts each team's child workflow (`startExecution.sync`)|You need one owner and end-to-end visibility|
|**Events (EventBridge)**|Each workflow starts from an event and publishes one when done|Teams must be very independent, flow is simple|
|**Hybrid**|Child workflows for internal steps, one parent for the cross-team flow, events for loose reactions|Most real systems|

Whoever owns the **business outcome** owns the parent workflow. Use a shared **correlation ID** (e.g. `orderId`), naming conventions, versioned contracts, and shared monitoring.

---

## 8. Checklist

- [ ] Run Event Storming with domain experts
- [ ] Write a glossary per bounded context
- [ ] Draw the context map and dependencies
- [ ] Create one module per context with enforced boundaries
- [ ] Map table ownership; give each module its own schema and DB user
- [ ] Test with restricted users in staging; fix every permission error
- [ ] Split shared tables by column ownership
- [ ] Replace cross-module joins and transactions (events, read models, sagas)
- [ ] Choose the extraction approach (direct extraction, tactical forking, or refactor first)
- [ ] Extract a service only with a concrete reason
- [ ] Move data physically last, with CDC and verification