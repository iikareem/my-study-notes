---
aliases:
  - Pitwall Lessons
  - Pitwall Project Learn
  - F1 OLAP Project Recap
tags:
  - olap
  - pitwall
  - lessons
  - project
  - duckdb
  - parquet
---

# Pitwall — Everything We Learned

> **Project:** Formula 1 OLAP engine — Jolpica → Kimball Parquet lake → DuckDB → NestJS  
> **Repo:** https://github.com/iikareem/pitwall  
> **Status:** Core complete (ingest, lake, concept SQL, API, materialization, benchmark)

This note is the **full learning dump** from building Pitwall — mental models, decisions, and interview lines. Deep theory still lives in the modules; this is *what the project taught in practice*.

**Related:** [[Pitwall ETL Lesson — Developer Takeaways]] · [[Pitwall Star Schema Diagram]] · [[OLAP Starter Path — Think OLAP First]] · [[00 OLAP MOC]]

---

## 0. One-sentence story

We built a **batch analytics stack**: rate-limited REST → trusted **star-schema Parquet lake** → DuckDB queries / NestJS API — so analytics read the **lake**, not the live API.

```text
Jolpica API
    → Extract (throttle, retry, JSON cache)
    → Transform (natural keys, ms times, star rows)
    → Stage + grain validate (DuckDB stg_*)
    → Load Parquet (dims merge, facts/season= replace)
    → Query / API / summaries / benchmark
```

---

## 1. OLTP vs OLAP (why this project exists)

| OLTP (your Postgres muscle) | OLAP (Pitwall) |
| --- | --- |
| Point lookups, transactions | Scans + aggregates over history |
| Normalized 3NF | Dimensional star (denormalized for analytics) |
| B-tree indexes | **Layout** (Parquet columns + Hive folders) as the index |
| App writes many small rows | Batch publish immutable-ish fact files |

**Lesson:** Don’t force heavy historical analytics into the OLTP DB. Offload to a columnar engine over a lake.

---

## 2. Kimball star schema

### Shape
- **Dims (shared lookups):** drivers, constructors, circuits, races `(season, round)`
- **Facts (independent grains):** results, qualifying, laps, pit_stops
- Facts join dims via keys — facts do **not** depend on each other

### Grain (most important design word)
Before columns, write: *“One row means ___.”*

| Fact | One row means |
| --- | --- |
| results | one driver in one race |
| qualifying | one driver quali session |
| laps | one driver on one lap |
| pit_stops | one pit stop event |

**Lesson:** Mixing grains (e.g. joining laps to results carelessly) multiplies rows and lies. Aggregate the fine grain first, then join.

### Natural keys (not surrogates)
- Use business/API IDs: `driver_id = hamilton`, race = `(season, round)`
- Dims PK = natural key; facts FK = same values
- **Why here:** small dims, short strings, easy to debug, matches Jolpica
- **When surrogates win later:** huge dims, ugly wide keys, warehouse join cost

---

## 3. Lake layout (layout is design)

```text
data/parquet/f1/
  dims/*.parquet
  facts/season=2025/{results,laps,pit_stops,qualifying}.parquet
  summaries/{driver,constructor}_season_summary.parquet
```

### Hive-style partitions
- Folder name `season=2025` **is** a column
- Filter `WHERE season = 2025` → **partition pruning** (skip other years)
- **Facts** partitioned by season; **dims** stay flat (small, shared)

**Lesson:** Don’t partition dims by season (duplicates + messier joins). Don’t partition facts by `round` here (many tiny files).

### Glob vs one file
```sql
read_parquet('.../facts/season=*/results.parquet', hive_partitioning := true)
```
= **one logical table**, files stay **scattered on disk**. Not a merge unless you `COPY` into one file.

---

## 4. Parquet (what / why / encodings)

### Why Parquet in Pitwall
1. **Columnar** — queries read few columns → less I/O than CSV/JSON rows  
2. **Query in place** — DuckDB scans files; no mandatory “load warehouse first”  
3. **Compresses** F1-shaped data (repeated IDs, statuses)  
4. Fits batch lake: write partitions, read many times  

### Encodings — you usually don’t pick them
On write we set roughly: `FORMAT PARQUET, COMPRESSION ZSTD`.

DuckDB then **auto-picks per-column encodings**, e.g.:
- **Dictionary** — low-cardinality (`status`, `constructor_id`, `driver_id`)
- **Plain** — high-variance numerics (`time_ms`, …)
- **(+ ZSTD)** — compresses the encoded bytes

Inspect with `parquet_metadata(...)`.  
You *can* tune compression / writer options if measured need; Pitwall doesn’t hand-pick encodings.

### Parquet vs CSV (practice)
Same 2025 results: Parquet much smaller than CSV — that’s the jewelry of columnar + dictionary + ZSTD.

---

## 5. ETL flow (developer mindset)

### Design backward from the lake contract
1. What must consumers trust? (grain, keys, types, replace rules)  
2. What checks prove that?  
3. Only then: how do we extract?

### Phases

| Phase | Pitwall | Notes |
| --- | --- | --- |
| **Extract** | Jolpica + cache | Throttle (~500/hour), 429 retry |
| **Transform** | Per round → in-memory arrays | Times → ms; map to star rows |
| **Stage** | DuckDB `stg_*` | Whole season after all rounds |
| **Validate** | Grain uniqueness SQL | Fail → no publish |
| **Load** | Write Parquet | Dims **merge**; facts **replace** `season=YYYY/` |

```text
round 1..N → transform → arrays
                 ↓
         stage → validate → Parquet (once per season)
```

### Cache vs memory vs lake

| | Role | Lifetime |
| --- | --- | --- |
| **Cache** `data/cache/jolpica/*.json` | Raw API responses; skip re-download | Survives restarts |
| **Memory arrays** | Transformed dim/fact rows for this run | Process only |
| **Parquet lake** | **Product** analytics reads | Durable |

Cache hit → still transform again. Cache ≠ lake.

### Live season (e.g. 2026)
Re-ingest season = **rebuild whole `facts/season=2026/`**, not “append one race.”  
Past rounds → cache; new round → live GET; then stage → validate → replace folder.  
Watch **stale cache** if you cached a round before results existed.

### Idempotency
Run twice → same lake. Crash / 429 / retry safe.

### Grain check example
```sql
SELECT season, round, driver_id, COUNT(*) c
FROM stg_fact_results
GROUP BY 1,2,3
HAVING COUNT(*) > 1;
```
0 rows → OK to write.

### `npm run partition`
Only **validates** files exist / non-empty. Does not fetch, transform, or re-encode.

---

## 6. DuckDB & SQL jewelry practiced

### Patterns
- Star joins: facts + dims for readable analytics  
- `GROUP BY` + `RANK() OVER (...)` for standings  
- **`QUALIFY`** — filter **after** window functions (`WHERE` is too early for `rank`)  
- Why `SUM` inside `ORDER BY` of `RANK()` works: because of **`GROUP BY`** first; `PARTITION BY` only resets rank scope  
- Windows: rolling 5-race form, pace ranks per round  
- `EXPLAIN ANALYZE` — pruning / scan cost  
- Independent facts joined only when the question needs two grains  

### Materialization
**Materialize** = save the **query result** as a physical file/table (not just the SQL text).

```text
results (large) → GROUP BY season, driver → driver_season_summary.parquet
```

| View | Materialized |
| --- | --- |
| Stores SQL | Stores **answer** |
| Recomputes every time | Cheap read until rebuild |
| Always fresh | Freshness = last rebuild |

Use like any table: `WHERE`, `JOIN`, `ORDER BY`.  
Standings API: prefer summary if present → else raw `GROUP BY` (`source: "summary" | "raw"`).

Rebuild: `npm run query -- queries/materialized_aggregates.sql`

### Benchmark (`npm run benchmark`)
Compares:
1. **Pruning** — count all seasons vs one season  
2. **Materialization** — raw standings `GROUP BY` vs read summary  

Both sides still Parquet — comparing **strategy**, not format.  
Small lake → small ms gap; **pattern** matters at scale.

---

## 7. NestJS embedding

- Shared `DatabaseService` (one DuckDB connection)
- Prefer **season file paths** for API (`facts/season=2025/results.parquet`) over globs when answering one season
- Repository pattern: `F1Repository` owns SQL; service returns `{ season, source?, data }`
- Endpoints: standings, driver form, constructor pace, pitstops, teammates
- App depends on **lake**, not Jolpica at request time

---

## 8. Iceberg (later — not needed now)

```text
Now:   you manage Parquet folders
Later: Iceberg table (catalog, versions, ACID) still stores Parquet underneath
```

Finish Parquet + Hive + grain first; Iceberg clicks faster afterward.

---

## 9. Concept SQL map (`queries/`)

| File | Skill |
| --- | --- |
| `vectorized_aggregation.sql` | Columnar aggregations across seasons |
| `parquet_compression.sql` | Cardinality, encodings, CSV size compare |
| `explain_analyze.sql` | Plans / pushdown / prune vs full |
| `window_functions.sql` | Rolling form, ranks |
| `partition_pruning.sql` | Hive skip |
| `materialized_aggregates.sql` | Build summaries |

---

## 10. Commands cheat sheet

```bash
npm run ingest -- --seasons 2025
npm run partition
npm run query -- queries/vectorized_aggregation.sql
npm run query -- queries/materialized_aggregates.sql
npm run benchmark
npm run start:dev
# Postman: postman/Pitwall.postman_collection.json
```

DuckDB UI — views over lake (all seasons):

```sql
CREATE OR REPLACE VIEW fact_results AS
SELECT * FROM read_parquet(
  '/Users/kareem/Documents/Projects/Personal/Pitwall/data/parquet/f1/facts/season=*/results.parquet',
  hive_partitioning := true
);
-- dims: .../dims/drivers.parquet (etc.)
```

---

## 11. Interview / portfolio lines

> Batch ETL from Jolpica into a Kimball star on Parquet. Rate-limited cached extract, transform to natural keys and ms measures, stage in DuckDB, enforce grain uniqueness, publish dims (merge) and facts (season replace). NestJS/DuckDB serve analytics from the lake; standings prefer a materialized driver-season summary. Benchmarks show partition pruning and raw vs summary trade-offs.

**Reuse checklist for any ETL**
- [ ] Output contract (schema + grain)?  
- [ ] Unique keys checked?  
- [ ] Types fixed at transform?  
- [ ] Safe to re-run?  
- [ ] Source limits handled?  
- [ ] Write layout matches query patterns?  
- [ ] App depends on lake, not live source?  

---

## 12. What “core complete” means

**Done:** ingest → star lake → concept SQL → API → summaries → benchmark → docs/Postman  

**Optional later:** auto-rebuild summaries on ingest, more seasons, constructor summary in pace API, Iceberg, tests  

---

**Parent:** [[00 OLAP MOC]]
