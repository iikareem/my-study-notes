---
aliases:
  - Pitwall ETL Lesson
  - OLAP ETL Recap
tags:
  - olap
  - etl
  - pitwall
  - lessons
---

# ETL Lesson from Pitwall — What a Backend Developer Should Keep

> Project: Formula 1 analytics lake (Jolpica → Parquet → DuckDB/NestJS)  
> Design: **Kimball star schema**, natural keys, season partitions

This note is the **valuable takeaway** — not every script line.

---

## 1. What we built (one sentence)

A **batch ETL** that turns a rate-limited REST API into a **trusted analytical lake** (Parquet), so the app reads the lake — not the source API — for analytics.

---

## 2. End-to-end ETL flow

```text
SOURCE          EXTRACT              TRANSFORM           VALIDATE         LOAD (publish)
Jolpica API  →  fetch + JSON cache → dim/fact rows    → grain checks  →  Parquet lake
                (throttle/retry)     (types, keys)       (stg_* SQL)      dims + facts
```

### Step by step

| Step | What happens | Why it matters |
| --- | --- | --- |
| **Extract** | Call Jolpica (or read cache for that URL). Throttle to respect rate limits. | Getting bytes ≠ correct data. Isolate “can we download?” |
| **Transform** | Parse times → ms. Map JSON → star-schema rows. Natural keys (`driver_id`, …). | Shape data to *your* contract, not the API’s nesting. |
| **Stage** | Load all season rows into temporary DuckDB tables (`stg_*`). | Gives you SQL to check quality before publishing. |
| **Validate** | Grain uniqueness queries (no duplicate keys). Fail = no write. | Protects dashboards from silent double-counting. |
| **Load** | Write `dims/*.parquet` (merge). Replace `facts/season=YYYY/`. | Idempotent publish: re-run safe. |

**Important:** Fetch is **per round**; Parquet write is **once per season** after all rounds are collected.

---

## 3. The data contract (star schema)

### Dimensions (who / what / where / which race)
- `dim_drivers`, `dim_constructors`, `dim_circuits`, `dim_races`
- Shared lookups — many facts point at the same dims
- Natural keys from the API (no surrogate ints in v1)

### Facts (what happened — independent grains)
- `fact_results` — one driver × one race  
- `fact_qualifying` — one driver quali  
- `fact_laps` — one driver lap  
- `fact_pit_stops` — one pit stop  

Facts do **not** depend on each other. Join them only when a question needs two grains.

### Lake layout

```text
data/parquet/f1/
  dims/*.parquet
  facts/season=2025/*.parquet
```

`season=` folders enable **partition pruning** (skip unused seasons on query).

---

## 4. How to *think* about ETL (experience mindset)

### Design backward from the lake
1. What must consumers trust? (grain, keys, types, replace semantics)  
2. What checks prove that?  
3. Only then: how do we extract from the source?

### Separate concerns
- **Extract** = availability, rate limits, retries, cache  
- **Transform** = model + types  
- **Validate** = “would I ship this?”  
- **Load** = idempotent publish  

### Idempotency is mandatory
Assume crash / 429 / retry.  
**Run twice → same lake.**  
We replace season facts; we upsert dims; we cache GETs.

### Grain is the #1 analytics skill
Before columns, write: *“One row means ___.”*  
Then enforce that unique key in validate.

### Layout is part of the design
How you **write** files decides how cheap **reads** are (partitions, Parquet columns).

---

## 5. Pitwall method map (so you can navigate code)

| Phase | Main methods | File |
| --- | --- | --- |
| Entry | `main`, `parseSeasons` | `scripts/ingest.ts` |
| Extract (+ cache) | `jolpicaGet`, `jolpicaPaginate`, `readCache`, `writeCache`, `throttle` | `scripts/lib/jolpica.ts` |
| Collect season | `fetchSeason` | `scripts/ingest.ts` |
| Transform helpers | `parseTimeToMs`, `upsertDriver`, … | `time.ts` / `ingest.ts` |
| Stage → validate → write | `writeSeasonParquet` + `GRAIN_CHECKS` | `ingest.ts` / `schema.ts` |
| Verify lake | `npm run partition` | `scripts/partition.ts` |

---

## 6. Grain check example (validate)

**Rule for results:** one row per `(season, round, driver_id)`

```sql
SELECT season, round, driver_id, COUNT(*) AS c
FROM stg_fact_results
GROUP BY 1, 2, 3
HAVING COUNT(*) > 1;
```

- 0 rows → OK → write Parquet  
- Any row → abort (don’t publish bad data)

---

## 7. Cache vs lake (don’t confuse them)

| | Role |
| --- | --- |
| `data/cache/jolpica/*.json` | Temporary API responses so re-runs don’t re-download |
| `data/parquet/f1/**/*.parquet` | **Real dataset** — what DuckDB / NestJS should use |

After ingest succeeds, **Parquet is the product**. Cache is optional speed for ETL retries.

---

## 8. Interview-ready summary

> We built a batch ETL from Jolpica into a Kimball star on Parquet. Extract is rate-limited and cached. Transform maps to natural keys and millisecond measures. We stage in DuckDB, enforce grain uniqueness, then publish dims (merge) and facts (per-season replace). Analytics consumers read the lake, not the live API.

---

## 9. Checklist you can reuse on any ETL

- [ ] Output contract clear (schema + grain)?  
- [ ] Unique keys defined and checked?  
- [ ] Types normalized at transform time?  
- [ ] Job safe to re-run (idempotent)?  
- [ ] Source limits handled (throttle, retry, cache)?  
- [ ] Publish path matches how you’ll query (partitions, columnar files)?  
- [ ] Consumers depend on the **lake**, not the source?

---

## Commands

```bash
npm run ingest -- --seasons 2025   # ETL
npm run partition                  # verify lake files exist
```

Details: [[docs/INGEST]] in the Pitwall repo (`docs/INGEST.md`).

---

**Parent:** [[00 OLAP MOC]] · [[OLAP Starter Path — Think OLAP First]] · [[Pitwall Star Schema Diagram]] · [[Pitwall Project — Everything We Learned]]
