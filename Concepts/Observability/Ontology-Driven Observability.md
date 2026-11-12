# Ontology-Driven Observability

## 1. Observability basics

**Observability** is how well you can understand what is happening inside a running system from the data it emits. That data is called **telemetry**.

### The pillars

|Signal|What it is|Answers|Example|
|---|---|---|---|
|**Logs**|Timestamped text or JSON events|What happened?|`ERROR payment failed for user 42`|
|**Metrics**|Numbers aggregated over time|How much, how often, how fast?|`http_requests_total`, p99 latency = 800ms|
|**Traces**|The journey of one request across services|Where did it go, where was it slow?|BFF → Auth → Billing → DB|
|**Profiling**|Resource usage inside a process|Why is this code slow or heavy?|Flame graph: 60% CPU in `JSON.parse`|

The classic "three pillars" are logs, metrics and traces. **Profiling** is the newer fourth one.

### Traces and spans

- **Trace** = the whole story of one request. **Span** = one step in it (an operation with a start time, a duration and a parent).
- All spans of one request share a **trace ID**, which is how tools stitch them together.

```
Trace abc123 (900ms)
└─ Span: GET /balance (BFF)           900ms
   ├─ Span: verify JWT                 20ms
   └─ Span: call Billing              850ms
      └─ Span: SELECT accounts        800ms   ← culprit
```

## 2. The problem: raw observability has no shared meaning

Each signal lives in its own tool and format. During an incident, a human has to connect the dots manually.

|Problem|Example|
|---|---|
|**Same thing, many names**|`billing_svc`, `BillingService`, `bill-api`, `billing-api`, and an IP are all one service|
|**No relationships stored**|Nothing says BFF calls billing, or billing uses accounts-db|
|**Context lives elsewhere**|Deploys are in the CI/CD tool, owners in a wiki|
|**Unstable identifiers**|Pods and IPs change, so history points at things that no longer exist|
|**Implicit meaning and units**|p99 in seconds, logs in ms, traces in microseconds|

Linking signals by `trace_id` helps (a trace to its logs), but it is **shallow**. It doesn't tell you what the things in your system are, how they relate, or what changed.

**Result:** 5 tools and 15 to 30 minutes per incident. Impossible at hundreds of services, and an AI agent faces the same mess and may join the wrong things.

## 3. The concepts

### Ontology

A **formal, shared agreement** about:

1. **What kinds of things exist** (Service, Team, Deployment, Database)
2. **What properties they have** (name, owner, tier)
3. **How they relate** (`Service -CALLS-> Service`, `Team -OWNS-> Service`, `Deployment -CHANGED-> Service`)

Think of it as a schema or dictionary for the whole system.

### Knowledge graph

The ontology filled with real data: nodes (actual things) and edges (actual relationships). The ontology is the rules, the graph is the data.

## 4. What is Ontology-Driven Observability?

**Design the shared model first, then map every signal onto it**, instead of collecting raw data and hoping tools make sense of it later.

```
4 pillars + extra context (deploys, owners, dependencies)
        │  mapped using
        ▼
Ontology = the rules (entities, relationships, naming, units)
        │  stored as
        ▼
Graph database = connected data
        │  queried by
        ▼
Humans and AI agents → faster root-cause analysis
```

- It goes beyond the 4 pillars: the graph also holds deployments, ownership and dependencies. Netflix even includes experiments (A/B tests).
- **The graph database alone isn't the point. The ontology is.** A graph without a shared model is just a messy graph.
- Alibaba describes it as a shift from **data-driven to model-driven** observability.
- The AI angle: an LLM agent does root-cause analysis far better on a connected, well-defined graph than on raw siloed telemetry.

## 5. How it solves the "separate pillars" problem

|Raw observability|With ontology and graph|
|---|---|
|Logs, metrics, traces and profiles each in their own tool|All attached to the same entity node|
|6 names for one service|1 canonical entity|
|Relationships live in people's heads|Explicit edges (`CALLS`, `USES`, `OWNS`)|
|Deploys and owners in separate tools|Linked to the same node|
|Units and meanings implicit|Defined in the model|
|Humans or AI guess the joins|Joins already done|

## 6. Example (BFF → billing → accounts-db)

**Ontology (rules):**

```
Entities:      Service, Database, Team, Deployment
Relationships: Service -CALLS-> Service, Service -USES-> Database,
               Team -OWNS-> Service, Deployment -CHANGED-> Service
Rules:         every Service has name, owner, tier; durations in ms
```

**Same identity in every signal (Node.js):**

```js
logger.warn({ service: 'billing', trace_id, msg: 'slow query', duration_ms: 1800 });
metrics.histogram('db_query_duration_ms', 1800, { service: 'billing' });
span.setAttributes({ 'service.name': 'billing' });
```

**Graph:**

```
(Team: Payments) -OWNS->        (Service: billing)
(Service: bff)   -CALLS->       (Service: billing)
(Service: billing) -USES->      (Database: accounts-db)
(Deployment: v2.3.1 @ 13:55) -CHANGED-> (Service: billing)
```

**Incident:** alert says BFF `/balance` is slow. One graph query walks `CALLS` edges and returns: billing is slow, deployed v2.3.1 seven minutes before the spike, owned by Payments. Matching logs ("pool exhausted") and the slow span (`SELECT accounts`, 1800ms) are reached through the same `billing` key.

## 7. What goes where (important)

You do **not** store all raw telemetry in the graph DB. It is too big, too expensive, and the wrong tool for aggregation and log search.

|Graph database holds|Raw stores hold (Prometheus, Loki, Tempo, ClickHouse...)|
|---|---|
|Entities (services, DBs, teams, deploys)|Millions of log lines|
|Relationships between them|Metric time series|
|Small snapshots (current health, p99 now)|Individual spans|
|Pointers (service name, trace ID, time window)|Profiles|

**Flow:**

1. Ask the **graph**: which service, what changed, who owns it? It returns keys.
2. Use the keys (`service.name`, `trace_id`, time window) to query the **raw stores**.
3. Combine the context with the evidence.

The join works only because the same `service.name` and `trace_id` are used everywhere. **Graph = map and index. Raw stores = warehouse.**

## 8. Reality check: who uses it?

- **Netflix** presented it at QCon London (March 2026). Their E2EGraph models users, clients, services and infrastructure, and AI agents query it for root-cause analysis.
- **Alibaba Cloud UModel** is a product built on the idea.
- **Deeptrace and Observe** are startups using knowledge graphs or context graphs for AI incident investigation.
- **Sentry** is _not_ a full ontology. It has trace-connected data (signals linked via trace IDs) focused on app-level debugging.
- It is **new and mostly custom-built**, not a mainstream checkbox feature.

**What's custom to build:** the ontology design, the ingestion pipelines (traces, CI/CD, catalog → graph), keeping it up to date, and the query or agent layer. The graph database itself is off-the-shelf (Neo4j, Neptune, Memgraph, or even Postgres with nodes and edges tables).

## 9. Practical takeaway

Start with the cheap parts, which give most of the value:

1. **Consistent naming and metadata** across logs, metrics and traces, using OpenTelemetry semantic conventions (`service.name`, version, team, `trace_id`).
2. A simple **service catalog** (owner, dependencies).
3. Add a graph DB and an AI agent only if you have many services and real incident pain.

## 10. One-line summary

> **Ontology-driven observability = the same 4 pillars, plus deploy/ownership/dependency context, mapped onto a shared model (ontology) and stored as a graph, so humans and AI can follow relationships across signals instead of hunting through separate tools.**