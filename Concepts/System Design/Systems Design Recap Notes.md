# Systems Design Recap Notes

## Topic 1: Scaling a service

**Flow:** Define the goal (SLO) → measure → find the limit of one instance → optimize → scale out with headroom.

**Reliability numbers**

- **SLI** is the measurement, **SLO** is the internal target, and **SLA** is the contract with penalties.
- The SLO must be stricter than the SLA, which gives you a buffer before a penalty.
- **Error budget** = 100% − SLO. A 99.9% SLO allows about 43 minutes of downtime per 30 days. Alert on **burn rate**, not single spikes.

**Observability**

- **Four golden signals:** latency, traffic (RPS), errors, saturation. CPU, memory, event loop lag, and GC all fall under saturation.
- **Three pillars of observability:** metrics, logs, traces (a different list).
- Use **percentiles (p50/p95/p99)** instead of averages, because averages hide tail latency.
- Use **histograms**, because percentiles can't be averaged across instances but histogram buckets can be merged.
- Collect data over long periods (days, spikes, months) and break it down per endpoint and per service.

**Testing**

- **Workload characterization:** request mix, payload sizes, read/write ratio, peak vs. average.
- Test order: **smoke → load → stress → soak** (long steady run for leaks). Also **spike** tests.
- Use an **open model** (arrivals at a fixed rate) for internet traffic. A closed model hides overload.
- **Knee of the curve:** near saturation, queues build and latency grows nonlinearly. **Capacity** is the highest rate that still meets the SLO.
- Stress tests check the **failure mode**: graceful degradation, load shedding, and self-recovery.

**Laws and sizing**

- **Little's Law:** concurrency = arrival rate × latency. For example, 1,000 RPS × 0.2 s = 200 in flight. It sizes pools and workers.
- **Amdahl's Law:** speedup = 1 / ((1−p) + p/n). The serial fraction (locks, shared state) caps the gain. The **Universal Scalability Law** adds coordination cost, so throughput can drop as you add instances.
- **Headroom:** target about 70% CPU and keep 30% to absorb spikes while autoscaling spins up.
- **Sizing example:** 5,000 RPS ÷ 800 per instance = 6.25, so 7 instances, plus N+1 = **8**.

**Scaling out**

- Stateless services, no unnecessary locks, a load balancer in front.
- Run at least 2 instances even if one has the capacity, to avoid a single point of failure.
- A monolith or single instance is often enough. Scale only when the data says so.
- **API gateway** handles cross-cutting concerns (auth, rate limiting, routing). It's separate from the BFF.

**Gaps to remember**

- **The database is the next bottleneck:** instances × pool size must stay under the DB's max connections. Use PgBouncer, read replicas, caching, and indexes.
- **Resilience:** timeouts everywhere, retries with backoff and jitter, retry budgets, circuit breakers, bulkheads, rate limiting, and load shedding. Naive retries cause retry storms.
- **Node.js:** one process per core, worker threads for CPU work, and event loop lag as the real saturation signal.
- **Autoscaling:** scale on RPS per instance, queue depth, or event loop lag rather than CPU alone. Account for warm-up time, and set a minimum replica count and cooldowns.
- **Other:** caching and stampede protection, multi-AZ, graceful shutdown on SIGTERM, readiness vs. liveness probes, tracing, and flame graphs.

## Topic 2: Service decomposition

**Principles**

- Start with a monolith, and split only for a real problem, never for the buzzword.
- **Modular monolith first:** explicit imports and exports (NestJS modules). Enforce that no module touches another's tables or internals.
- Split by **business capability**, not technical layer. A generic "user service" becomes a god service.
- Boundaries are hard to see early. Wait until the system and domain mature.
- Reasons to split: independent deployment, team ownership, different tech needs, fault isolation, and scaling. Scaling is not the only one.

**DDD (strategic)**

- **Bounded context:** each context has its own model. "User" means different things in orders, payments, and auth.
- **Ubiquitous language:** consistent **within** a context. Different contexts translate between each other.
- **Context map** shows how contexts relate. An **anti-corruption layer (ACL)** translates so one model doesn't leak into another. **Shared kernel** is the risky exception.
- **Subdomains:** core, supporting, and generic. Invest in core and buy or simplify generic.
- Find boundaries with **event storming** and the "things that change together stay together" heuristic.
- **Conway's Law:** architecture mirrors team communication structure. The **inverse Conway maneuver** shapes teams first.

**Pitfalls**

- **Distributed monolith:** services that must deploy together or call each other in chains.
- **Signs you split wrong:** chatty services, one change touching many services, lockstep deploys, shared tables, and long synchronous chains.
- **Anti-patterns:** nano-services, a shared database, CRUD-only "entity services," and shared libraries that couple everything.

**Data migration**

- Order: same DB with a **schema per service** → **separate DB per service**. Never share a database between services.
- Each service owns its data, validation, and rules, and exposes data only through its API.
- **Strangler fig:** put a facade in front, route piece by piece, and keep a rollback path. Don't do a big-bang migration.
- Extraction checklist: pick a seam, add a facade, shadow traffic, migrate data, cut over, keep rollback.
- Extract first the service with clear boundaries, little coupling, and a real reason.

**Patterns (what problem each solves)**

- **BFF:** a backend tailored per frontend type.
- **API composition:** replaces the cross-service joins you lost.
- **CQRS:** separate read and write models when their needs differ.
- **Saga:** replaces distributed transactions, using compensating actions.
- **Outbox:** solves the dual-write problem by writing the event in the same DB transaction.
- Reading another service's data: API call, event-carried state transfer (local copy), or composition. Each has a consistency trade-off.

**Costs you accept**

- Network latency and failures, eventual consistency, no cross-service transactions, harder debugging, contract versioning, and operational overhead.
- Prerequisites: correlation IDs, distributed tracing, and centralized logs before you split.
- **Contracts:** versioning and consumer-driven contract tests (Pact).
- **Ownership:** one team, one on-call rotation, and its own SLOs per service. Cognitive load limits how many services a team can own.

## Topic 3: Messaging

|Type|What it is|Examples|
|---|---|---|
|**Job queue**|"Do this work" offloaded from the main thread: retries, delays, scheduled jobs, scalable workers|BullMQ (Redis), SQS|
|**Message queue**|One message is processed by **one consumer**. Competing consumers scale workers. Deleted after ack.|RabbitMQ, SQS|
|**Pub/sub**|One message fans out to **many subscribers**, each with its own queue|RabbitMQ exchange, SNS|
|**Event stream**|**Append-only log**, partitioned, retained by policy, replayable from any offset|Kafka|
|**Event bus**|**Content-based routing** across many producers and targets, including SaaS, with transformation and schemas|EventBridge, Event Grid|

**Key distinctions**

- Queues **are durable**. The difference is that a queue **deletes after ack**, while a log **retains until expiry**. The distinction is consumption semantics, not durability.
- **Backpressure:** push brokers need a **prefetch limit (QoS)**. Pull systems (SQS, Kafka) let the consumer set the pace.
- A job is a **command** ("do X"), while a message is often a **notification** ("X happened"). The line is conceptual, since the tools overlap.
- **Kafka details:**
    - Ordering holds **only within a partition**.
    - Max parallelism per consumer group equals the **partition count**.
    - Choose partition keys carefully, since a hot key creates a hot partition.
    - The broker is simple and the consumer is smart, so there is no per-message retry, delay, or priority.
    - A poison message blocks its partition, so you need your own DLQ handling.
    - Uses go beyond analytics: event sourcing, CDC, and rebuilding read models.
    - Monitor **consumer lag**, and know that a rebalance pauses processing.
- **Event bus vs. pub/sub:** pub/sub routes by topic, while a bus routes by rules on event content. Its trade-offs are hard-to-trace flows, schema governance, and being a critical dependency.

**Reliability (senior essentials)**

- **Delivery semantics:** at-most-once, at-least-once, exactly-once. In practice you get at-least-once plus **idempotent consumers**.
- **Idempotency:** idempotency keys, a processed-messages table with a unique constraint, and naturally idempotent operations (set vs. increment).
- **Retries and DLQs:** retry with backoff, cap attempts, park failures in a dead-letter queue, alert on DLQ depth, and keep a replay tool.
- **Outbox + CDC** (outbox table with a relay, or Debezium) publishes events reliably from the DB.
- **Schema evolution:** a schema registry (Avro/Protobuf), backward and forward compatibility, and versioned events.
- **Event types:** commands vs. events vs. queries. Name events in the past tense ("OrderPlaced").
- **Event sourcing** is a separate, heavier pattern. Know when not to use it.

**Orchestration vs. choreography** _(reference only; we hadn't discussed this part in detail yet)_

- **Choreography:** services react to each other's events with no central coordinator. It's loosely coupled, but the flow is hard to see.
- **Orchestration:** a central coordinator directs the steps and tracks state. The flow is explicit and monitorable, but it can become a coupling point.
- Choreography suits simple, few-step reactions. Orchestration suits long-running processes with compensation, timeouts, and human steps.
- Tools: Temporal, AWS Step Functions, Camunda, commonly used to implement sagas.

**Decision guide**

|Need|Pick|
|---|---|
|Offload slow or heavy work with retries and delays|Job queue|
|One consumer pool handles each message|Message queue|
|Notify many independent services|Pub/sub|
|Replay, per-key ordering, huge throughput|Event stream|
|Content-based routing across producers and third parties|Event bus|

## Senior habits (across all topics)

1. **Failure first:** ask "what happens when it's slow, not just down?"
2. **Trade-off statements:** "I'd choose X because of Y, accepting Z, and I'd revisit if W changes."
3. **Size with numbers**, and ask "what breaks next?" after every step.
4. **Name things precisely** (golden signals, Little's Law, Amdahl's Law, Conway's Law, ubiquitous language, ACL, soak test).
5. **Count cost** in infrastructure, operations, and cognitive load.
6. **Tie each topic to a real story** from your BFF and OCS migration: what you measured, what you decided, and what you'd change.

## My self-assessment from the session

- Scaling: about 7/10 for mid-level, about 6/10 against a senior bar.
- Decomposition: about 7.5/10 for mid-level, about 6.5/10 against a senior bar.
- Messaging: about 7/10 for mid-level, about 6/10 against a senior bar.
- **Gap to senior:** vocabulary precision, sizing numbers, failure modes, and orchestration/idempotency.

I can also turn this into a one-page cheat sheet or a Word/PDF file if you'd like to keep it outside the chat.