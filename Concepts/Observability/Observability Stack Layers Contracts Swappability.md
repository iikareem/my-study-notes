---
tags:
  - observability
---

# Observability Stack: Layers, Contracts & Swappability

> A complete guide to how each layer works independently, what interface it exposes, and which tools are interchangeable at each level.

---

## Table of Contents

1. [The Mental Model: 4 Layers](#1-the-mental-model-4-layers)
2. [Layer 1 — The Target (Your App / Database)](#2-layer-1--the-target-your-app--database)
3. [Layer 2 — The Exporter (Metric Producer)](#3-layer-2--the-exporter-metric-producer)
4. [Layer 3 — The Collector / TSDB (Metric Storage)](#4-layer-3--the-collector--tsdb-metric-storage)
5. [Layer 4 — The Visualizer (Query & Dashboard)](#5-layer-4--the-visualizer-query--dashboard)
6. [The Contracts Between Layers](#6-the-contracts-between-layers)
7. [Swappability Matrix](#7-swappability-matrix)
8. [Example Stack: pg_exporter → Prometheus → Grafana](#8-your-stack-pg_exporter--prometheus--grafana)
9. [Real Swap Examples](#9-real-swap-examples)
10. [Choosing What to Swap and Why](#10-choosing-what-to-swap-and-why)

---

## 1. The Mental Model: 4 Layers

The entire observability pipeline is a chain of **producers and consumers** connected by well-defined interfaces:

```
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 1: TARGET                                                      │
│  Your database, app, OS — the thing generating real-world activity   │
└──────────────────────┬───────────────────────────────────────────────┘
                       │  native protocol (SQL, TCP, proc filesystem…)
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 2: EXPORTER                                                    │
│  Reads the target, translates to a standard metrics format           │
└──────────────────────┬───────────────────────────────────────────────┘
                       │  HTTP /metrics  (Prometheus exposition format)
                       ▼                OR  push via StatsD / OTLP
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 3: COLLECTOR / TSDB                                            │
│  Scrapes or receives metrics, stores them as time-series data        │
└──────────────────────┬───────────────────────────────────────────────┘
                       │  query API  (PromQL HTTP API / InfluxQL / SQL…)
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  LAYER 4: VISUALIZER                                                  │
│  Queries the TSDB, renders dashboards, sends alerts                  │
└──────────────────────────────────────────────────────────────────────┘
```

Each arrow is a **contract** (a defined interface). As long as both sides of an arrow speak the same contract, the layers on either side are swappable independently of each other.

---

## 2. Layer 1 — The Target (Your App / Database)

### What it is

The system whose internals you want to observe. It does not know or care about monitoring. It speaks its own native protocol.

| Target       | Native interface                        |
| ------------ | --------------------------------------- |
| PostgreSQL   | SQL queries against `pg_stat_*` views   |
| Redis        | `INFO` command over TCP                 |
| Linux kernel | `/proc` and `/sys` filesystems          |
| JVM app      | JMX (Java Management Extensions)        |
| Node.js app  | Custom HTTP endpoint or SDK             |
| Nginx        | `ngx_http_stub_status_module` HTTP page |

### How it works independently

The target just runs. It exposes internal counters, gauges, and histograms through its native interface. Nothing at this layer knows about Prometheus, Grafana, or any monitoring tool. That's intentional — the target doesn't carry monitoring overhead.

---

## 3. Layer 2 — The Exporter (Metric Producer)

### What it is

A small process that bridges Layer 1 (the target's native protocol) to Layer 3 (the collector's expected format). It is a **translator**.

### How it works independently

```
pg_exporter loop:
  every scrape_interval:
    connect to PostgreSQL
    run SQL: SELECT ... FROM pg_stat_activity
    run SQL: SELECT ... FROM pg_stat_bgwriter
    ...
    format results as Prometheus text exposition
    serve on HTTP :9187/metrics
    wait for next interval
```

The exporter is a **passive HTTP server**. It does not push anywhere. It waits to be scraped.

### The contract it exposes (Prometheus Exposition Format)

Every exporter speaks the same text format on `GET /metrics`:

```
# HELP pg_stat_bgwriter_checkpoints_timed Number of scheduled checkpoints
# TYPE pg_stat_bgwriter_checkpoints_timed counter
pg_stat_bgwriter_checkpoints_timed 42

# HELP pg_database_size_bytes Disk space used by the database
# TYPE pg_database_size_bytes gauge
pg_database_size_bytes{datname="mydb"} 1048576
```

This is plain text. Any HTTP client can read it. The format has four metric types:

|Type|What it is|Example|
|---|---|---|
|`counter`|Monotonically increasing number, resets on restart|Total queries executed|
|`gauge`|Current value, can go up or down|Active connections right now|
|`histogram`|Bucketed distribution of observations|Query latency in time buckets|
|`summary`|Pre-computed quantiles|p50, p95, p99 latency|

### Notable exporters by target

|Target|Exporter|Port|
|---|---|---|
|PostgreSQL|`postgres_exporter` (Prometheus community)|9187|
|MySQL / MariaDB|`mysqld_exporter`|9104|
|Redis|`redis_exporter`|9121|
|Linux OS|`node_exporter`|9100|
|Nginx|`nginx_prometheus_exporter`|9113|
|MongoDB|`mongodb_exporter`|9216|
|RabbitMQ|built-in plugin|15692|
|Kafka|`kafka_exporter` or JMX exporter|9308|
|JVM apps|`jmx_exporter` (Java agent)|configurable|

### Alternatives: push-based exporters

Some systems push metrics instead of waiting to be scraped:

|Protocol|What pushes|Who receives|
|---|---|---|
|StatsD (UDP)|Your app code|StatsD server → Graphite or Prometheus|
|OTLP (OpenTelemetry)|SDK in your app|OpenTelemetry Collector|
|Graphite|Your app code|Graphite / Prometheus remote write|
|InfluxDB line protocol|Telegraf agent|InfluxDB|

---

## 4. Layer 3 — The Collector / TSDB (Metric Storage)

### What it is

A **time-series database (TSDB)** that either scrapes exporters on a schedule (pull model) or receives pushed metrics. It stores every metric as a stream of `(timestamp, value, labels)` tuples and provides a query language to retrieve them.

### How Prometheus works independently

```
Prometheus loop:
  for each target in scrape_configs:
    every scrape_interval (default 15s):
      HTTP GET http://<target>/metrics
      parse exposition format
      store samples in local TSDB (chunks on disk)
      evaluate recording rules
      evaluate alerting rules → send to Alertmanager
```

The scrape interval is the fundamental resolution of your data. If `scrape_interval: 15s`, you cannot know what happened between two scrape points.

### Prometheus storage: the TSDB

Prometheus writes to a local chunked time-series store:

```
data/
  01BKGV7JC0RY8A6PMTY1QJ6PW/   ← 2-hour block (compacted)
    chunks/
      000001                     ← raw sample data
    index                        ← series index for fast lookups
    meta.json
  wal/                           ← write-ahead log (last ~2h, in memory)
```

Data is retained for 15 days by default. This is **not** designed for long-term multi-year storage.

### The contract Prometheus exposes (HTTP API)

Any visualizer that speaks the Prometheus HTTP API can query Prometheus:

```
GET /api/v1/query          — instant query (one point in time)
GET /api/v1/query_range    — range query (time series over a window)
GET /api/v1/labels         — list label names
GET /api/v1/series         — list matching series
```

Example:

```
GET /api/v1/query_range
  ?query=rate(pg_stat_bgwriter_checkpoints_timed[5m])
  &start=1700000000
  &end=1700003600
  &step=15
```

Response is JSON:

```json
{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": { "__name__": "..." },
        "values": [[1700000000, "0.02"], [1700000015, "0.02"], ...]
      }
    ]
  }
}
```

### Alternative collectors / TSDBs

| Tool                        | Model                | Query language              | Long-term storage  | Notes                                               |
| --------------------------- | -------------------- | --------------------------- | ------------------ | --------------------------------------------------- |
| **Prometheus**              | Pull (scrape)        | PromQL                      | No (15d default)   | The reference implementation                        |
| **VictoriaMetrics**         | Pull + push          | MetricsQL (PromQL superset) | Yes                | Drop-in Prometheus replacement, much more efficient |
| **Thanos**                  | Pull + object store  | PromQL                      | Yes (S3/GCS/etc)   | Runs on top of Prometheus, adds global view         |
| **Cortex / Mimir**          | Pull + push          | PromQL                      | Yes (multi-tenant) | Grafana Mimir is the current name                   |
| **InfluxDB**                | Push (line protocol) | InfluxQL / Flux             | Yes                | Different data model                                |
| **Graphite**                | Push                 | Graphite functions          | Yes                | Older, simpler                                      |
| **OpenTelemetry Collector** | Pull + push          | (passes through to backend) | No (just routing)  | Protocol translator, not a TSDB                     |
| **Datadog Agent**           | Pull + push          | Datadog Query Language      | Yes (SaaS)         | Managed, not self-hosted                            |

VictoriaMetrics, Thanos, and Mimir all expose the **same Prometheus HTTP API**, which is why Grafana doesn't care which one you run — it queries all of them identically.

---

## 5. Layer 4 — The Visualizer (Query & Dashboard)

### What it is

A frontend that sends queries to Layer 3, renders the results as charts, and optionally manages alerts. It does not store metrics itself — it is stateless with respect to metric data.

### How Grafana works independently

```
User opens dashboard
  → Grafana reads dashboard JSON (stored in its own SQLite/Postgres)
  → for each panel:
      execute PromQL query against configured data source
      receive JSON time-series response
      render chart in browser
  → repeat every refresh_interval (e.g., 30s)
```

Grafana stores only:

- Dashboard definitions (JSON)
- Data source configurations (connection URLs, credentials)
- User accounts and permissions
- Alert rules

It stores **zero metric data**.

### The contract Grafana consumes

Grafana communicates with data sources through **data source plugins**. Each plugin speaks a specific query protocol. The built-in Prometheus plugin speaks the Prometheus HTTP API. When you write PromQL in Grafana, Grafana's plugin translates it into an HTTP request to `/api/v1/query_range`.

### Alternative visualizers

| Tool                       | Protocol support                                          | Notes                                  |
| -------------------------- | --------------------------------------------------------- | -------------------------------------- |
| **Grafana**                | Prometheus, InfluxDB, Graphite, Loki, Datadog, 50+ others | Most popular, very extensible          |
| **Prometheus built-in UI** | PromQL only                                               | Minimal, good for debugging            |
| **Kibana**                 | Elasticsearch query DSL                                   | Best with ELK stack                    |
| **Chronograf**             | InfluxQL / Flux                                           | Part of TICK stack                     |
| **Netdata Cloud**          | Netdata API                                               | Real-time, 1-second resolution         |
| **Datadog**                | Datadog Query Language (SaaS)                             | Full managed stack                     |
| **Perses**                 | PromQL                                                    | New open standard, Grafana alternative |

---

## 6. The Contracts Between Layers

This is the key insight. Each interface is a **protocol contract**:

```
Layer 1 → Layer 2
  Contract: Target's native protocol
  (SQL for PostgreSQL, /proc for Linux, INFO for Redis)
  Who defines it: The target vendor
  Who must implement it: The exporter

Layer 2 → Layer 3
  Contract: Prometheus Exposition Format over HTTP
           OR: Push protocol (OTLP, StatsD, line protocol)
  Who defines it: The Prometheus project (for pull)
  Who must implement it: Both exporter (server) and collector (client)

Layer 3 → Layer 4
  Contract: Prometheus HTTP API (PromQL)
           OR: InfluxQL, Graphite API, etc.
  Who defines it: The TSDB
  Who must implement it: Both TSDB (server) and visualizer (client)
```

### Why this matters

As long as both ends of each arrow speak the same protocol, **the layers are independent**. You can swap any layer without touching the others, as long as the replacement speaks the same contract on both its incoming and outgoing interfaces.

---

## 7. Swappability Matrix

### Layer 2 → Layer 3 interface (exposition format / scrape protocol)

Can you swap the **exporter** without changing the **collector**?

|Swap|Compatible?|Condition|
|---|---|---|
|`postgres_exporter` → `node_exporter`|✅ Yes|Both serve Prometheus exposition format|
|`postgres_exporter` → custom exporter|✅ Yes|If custom exporter serves `/metrics` in exposition format|
|Any exporter → Telegraf (push mode)|⚠️ Partial|Telegraf can push to Prometheus remote write, but changes the scrape model|

Can you swap the **collector** without changing the **exporter**?

|Swap|Compatible?|Condition|
|---|---|---|
|Prometheus → VictoriaMetrics|✅ Yes|VictoriaMetrics scrapes the same `/metrics` endpoint|
|Prometheus → Thanos|✅ Yes|Thanos sidecar sits next to Prometheus|
|Prometheus → InfluxDB + Telegraf|❌ No|Telegraf uses a different agent model; exporters need reconfiguring|

### Layer 3 → Layer 4 interface (query API)

Can you swap the **TSDB** without changing the **visualizer**?

|Swap|Compatible?|Condition|
|---|---|---|
|Prometheus → VictoriaMetrics|✅ Yes|Same HTTP API, PromQL-compatible|
|Prometheus → Mimir|✅ Yes|Same Prometheus HTTP API|
|Prometheus → Thanos|✅ Yes|Thanos Querier exposes the same API|
|Prometheus → InfluxDB|❌ No|Different API and query language; need to change Grafana data source|

Can you swap the **visualizer** without changing the **TSDB**?

|Swap|Compatible?|Condition|
|---|---|---|
|Grafana → Prometheus UI|✅ Yes|Both speak PromQL to Prometheus|
|Grafana → Perses|✅ Yes|Perses supports Prometheus data source|
|Grafana → Kibana|❌ No|Kibana expects Elasticsearch, not Prometheus|

---

## 8. Example Stack: pg_exporter → Prometheus → Grafana

Let's trace exactly what happens end-to-end in your current setup:

### Step-by-step flow

```
1. PostgreSQL is running
   └─ pg_stat_activity, pg_stat_bgwriter, etc. are populated internally

2. postgres_exporter runs on the same host (or a sidecar)
   └─ connects to PostgreSQL via DSN (environment variable DATA_SOURCE_NAME)
   └─ runs SQL queries against pg_stat_* views
   └─ starts HTTP server on :9187

3. Prometheus scrape loop fires (every scrape_interval, e.g., 15s)
   └─ HTTP GET http://localhost:9187/metrics
   └─ receives plain-text exposition format
   └─ parses each line into (metric_name, labels, value)
   └─ appends to TSDB with current timestamp
   └─ evaluates any recording_rules or alerting_rules you have defined

4. You open Grafana
   └─ your dashboard panel has a PromQL query, e.g.:
        rate(pg_stat_bgwriter_checkpoints_timed[5m])
   └─ Grafana's Prometheus plugin fires:
        GET http://prometheus:9090/api/v1/query_range
          ?query=rate(pg_stat_bgwriter_checkpoints_timed[5m])
          &start=...&end=...&step=15
   └─ Prometheus evaluates the PromQL, returns JSON
   └─ Grafana renders the chart
```

### Configuration files that define each layer

**Layer 2 — postgres_exporter**

```yaml
# docker-compose.yml or systemd env
DATA_SOURCE_NAME: "postgresql://user:pass@localhost:5432/postgres?sslmode=disable"
PG_EXPORTER_WEB_LISTEN_ADDRESS: ":9187"
PG_EXPORTER_WEB_TELEMETRY_PATH: "/metrics"
```

**Layer 3 — Prometheus scrape config**

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "postgresql"
    static_configs:
      - targets: ["localhost:9187"]
        labels:
          instance: "primary-db"
```

**Layer 4 — Grafana data source**

```json
{
  "name": "Prometheus",
  "type": "prometheus",
  "url": "http://prometheus:9090",
  "access": "proxy"
}
```

---

## 9. Real Swap Examples

### Swap A: Replace Prometheus with VictoriaMetrics

**Why**: Better compression, lower memory, handles higher cardinality, has built-in long-term retention.

**What changes**:

- Stop Prometheus, start VictoriaMetrics
- VictoriaMetrics scrapes the same `/metrics` endpoints with the same config format
- VictoriaMetrics exposes the same HTTP API on its port

**What does NOT change**:

- `postgres_exporter` — untouched
- Grafana dashboards — untouched (same PromQL queries work)
- Just update Grafana's data source URL from `:9090` to VictoriaMetrics's port (default `:8428`)

```yaml
# VictoriaMetrics single-node, replaces prometheus.yml with identical format
scrape_configs:
  - job_name: "postgresql"
    static_configs:
      - targets: ["localhost:9187"]
```

### Swap B: Add Thanos for long-term storage

**Why**: Prometheus only stores 15 days. Thanos extends it to unlimited via S3/GCS.

**What changes**:

- Add Thanos Sidecar next to each Prometheus instance (reads its TSDB, uploads to object storage)
- Add Thanos Querier (exposes the same PromQL HTTP API, fans out to multiple Prometheus + Sidecar)

**What does NOT change**:

- `postgres_exporter` — untouched
- Prometheus — still runs, still scrapes
- Grafana — point data source to Thanos Querier URL instead of Prometheus URL
- All PromQL queries — untouched

### Swap C: Replace Grafana with Prometheus's built-in UI

**Why**: Debugging, no Grafana available.

**What changes**: Nothing at all in the backend.

Open `http://prometheus:9090/graph`, type your PromQL query. It's the same query engine, same data, minimal UI.

### Swap D: Replace postgres_exporter with a custom exporter

**Why**: You need metrics that postgres_exporter doesn't expose (e.g., business-level queries, custom health checks).

**What you must implement**:

```python
# custom_pg_exporter.py (minimal example)
from prometheus_client import start_http_server, Gauge
import psycopg2, time

active_sessions = Gauge('my_pg_active_sessions', 'Active sessions', ['state'])

def collect():
    conn = psycopg2.connect("...")
    cur = conn.cursor()
    cur.execute("SELECT state, count(*) FROM pg_stat_activity GROUP BY state")
    for state, count in cur.fetchall():
        active_sessions.labels(state=state).set(count)

start_http_server(9999)  # serves /metrics
while True:
    collect()
    time.sleep(15)
```

**What does NOT change**: Prometheus scrape config just points to `:9999`. Grafana is untouched.

### Swap E: Replace pull model with OpenTelemetry push model

**Why**: You want a vendor-neutral instrumentation standard, or your environment doesn't allow Prometheus to reach your exporters (firewall rules, dynamic scaling).

**Architecture**:

```
App with OTEL SDK
  → OTLP push → OpenTelemetry Collector
                  → Prometheus remote write → VictoriaMetrics
                                               → Grafana (PromQL)
```

The OTEL Collector is a **protocol router** at Layer 2/3 boundary. It receives OTLP, converts to Prometheus format, and remote-writes to any Prometheus-compatible TSDB. Grafana sees the same data in the same format.

---

## 10. Choosing What to Swap and Why

### Decision guide by problem

| Problem                                   | Layer to fix | What to swap in                                                         |
| ----------------------------------------- | ------------ | ----------------------------------------------------------------------- |
| Prometheus uses too much RAM              | Layer 3      | VictoriaMetrics (5–10x lower memory)                                    |
| Need metrics older than 15 days           | Layer 3      | Thanos or Mimir on top of Prometheus                                    |
| Need global view across multiple clusters | Layer 3      | Thanos Querier or Mimir                                                 |
| postgres_exporter missing a metric I need | Layer 2      | Fork it, or write a custom exporter alongside it                        |
| Grafana is overkill for simple checks     | Layer 4      | Prometheus built-in UI or `promtool`                                    |
| Need to monitor 10 different databases    | Layer 2      | Deploy one postgres_exporter per database; Prometheus scrapes all       |
| Moving to cloud-managed monitoring        | Layer 3+4    | Replace with Datadog, Grafana Cloud, or AWS Managed Prometheus          |
| App can't be scraped (behind NAT)         | Layer 2→3    | Use Prometheus Pushgateway (batch job) or OTEL Collector with OTLP push |

### The two axes of swappability

```
                    Same query protocol?
                    (PromQL HTTP API)
                         YES         NO
                    ┌──────────┬──────────┐
Same exposition  YES│  Full    │  Swap    │
format?             │  freedom │  TSDB    │
(/metrics HTTP) ────┼──────────┤  too     │
                 NO │  Swap    │  Full    │
                    │  exporter│  rewrite │
                    │  too     │          │
                    └──────────┴──────────┘
```

- **Top-left**: You can swap anything. The interfaces are stable and widely implemented.
- **Top-right**: Your exporters are fine, but you're changing the query language (e.g., moving to InfluxDB). You must reconfigure Grafana data sources and rewrite all dashboard queries.
- **Bottom-left**: Your TSDB and visualizer are fine, but you need a new exporter approach (e.g., push-based). You must add an adapter layer.
- **Bottom-right**: Full migration. Every layer changes.

### Summary of interfaces to remember

|Interface|Standard|Defined by|Widely adopted|
|---|---|---|---|
|Exporter → Collector|Prometheus exposition format (text/openmetrics)|Prometheus project|✅ Yes|
|Collector → Visualizer (pull)|Prometheus HTTP API|Prometheus project|✅ Yes|
|App → Collector (push)|OTLP (OpenTelemetry Protocol)|CNCF|✅ Growing|
|Collector → Collector (federation)|Prometheus remote write|Prometheus project|✅ Yes|

---

## Appendix: Quick Reference Card

```
YOUR STACK TODAY:
  PostgreSQL ──SQL──► postgres_exporter ──/metrics──► Prometheus ──PromQL──► Grafana

WHAT YOU CAN SWAP AT LAYER 2 (exporter):
  postgres_exporter → node_exporter, mysqld_exporter, custom exporter
  (all serve the same /metrics format, Prometheus and Grafana untouched)

WHAT YOU CAN SWAP AT LAYER 3 (TSDB):
  Prometheus → VictoriaMetrics, Thanos, Mimir, Grafana Mimir
  (all speak the same PromQL HTTP API, Grafana untouched)

WHAT YOU CAN SWAP AT LAYER 4 (visualizer):
  Grafana → Prometheus UI, Perses
  (all speak PromQL to the same TSDB)

KEY RULE:
  If the replacement speaks the same protocol on both its input and output,
  it is a drop-in swap. No other layer needs to change.
```