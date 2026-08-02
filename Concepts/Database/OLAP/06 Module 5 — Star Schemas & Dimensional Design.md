# Module 5 — Analytical Data Modeling: Star Schemas & Dimensional Design

[[05 Module 4 — Partitioning & Data Layout|← Module 4]] · [[00 OLAP MOC|↑ Table of Contents]]

---

## Module 5 Summary

> **One-liner:** Analytical modeling pre-joins your data into a shape where GROUP BY is fast and filters land on small tables — the opposite of 3NF normalization.

**Most important takeaways:**

1. **3NF hurts OLAP** — Normalized schemas require many joins at query time across large tables. OLAP models denormalize deliberately so queries touch fewer, wider, pre-joined structures.
2. **Star schema** — One central fact table (measurements, events) surrounded by dimension tables (context: who, what, where, when). Joins are shallow (one hop) and fast.
3. **Fact tables** — Tall and narrow: many rows, few columns (mostly foreign keys + numeric measures).
4. **Dimension tables** — Short and wide: few rows, many columns (all the descriptive attributes you filter and group by).
5. **Grain** — The single most important design decision: what does one row in the fact table represent? Define it before writing a single column.
6. **Slowly Changing Dimensions (SCDs)** — What happens when a dimension attribute changes? The answer determines whether historical analysis is accurate or silently wrong.
7. **Modeling beats query tricks** — A well-modeled star schema answers most analytical questions with a simple `GROUP BY`. Poor modeling forces complex, slow queries on every request.

**Jump to:** [[#5.1 Why 3NF is wrong for analytics|5.1]] · [[#5.2 The star schema|5.2]] · [[#5.3 Fact tables|5.3]] · [[#5.4 Dimension tables|5.4]] · [[#5.5 Grain — the most important design decision|5.5]] · [[#5.6 Slowly Changing Dimensions|5.6]] · [[#5.7 Snowflake schema vs star schema|5.7]] · [[#5.8 Common modeling mistakes|5.8]] · [[#5.9 Postgres → Dimensional Model Translation Table|5.9 Translation]]

---

## 5.1 Why 3NF is wrong for analytics

PostgreSQL's normalization is the right default for OLTP. 3NF eliminates redundancy, prevents update anomalies, and keeps writes fast. You have `orders`, `customers`, `products`, `categories` — joined at query time.

This is exactly wrong for analytical workloads.

**Joins across large tables are expensive at scale.** A typical 3NF analytical query:

```sql
SELECT c.country, p.category, SUM(o.revenue)
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN categories cat ON p.category_id = cat.id
WHERE o.created_at >= '2024-01-01'
GROUP BY c.country, p.category;
```

Five tables, four joins. At 500 million orders this is a bottleneck regardless of how well you tune it.

OLAP modeling pre-joins this at write time so the query becomes:

```sql
SELECT country, category, SUM(revenue)
FROM fact_orders
WHERE order_date >= '2024-01-01'
GROUP BY country, category;
```

No joins. The descriptive attributes were denormalized at write time.

**The key trade-off:** 3NF is optimized for writes (one place to update). Analytical data is written once and read many times. The redundancy cost of denormalization is paid once at write time; the benefit (no runtime joins) is gained on every query.

---

## 5.2 The star schema

The **star schema** is the standard pattern: a central fact table surrounded by dimension tables.

```
                    ┌─────────────────┐
                    │   dim_date      │
                    │  date_key (PK)  │
                    │  year           │
                    │  quarter        │
                    │  month          │
                    │  is_holiday     │
                    └────────┬────────┘
                             │
┌─────────────────┐          │          ┌─────────────────┐
│  dim_customer   │          │          │  dim_product    │
│ customer_key PK ├──────────┤          │ product_key  PK │
│  customer_name  │          │          │  product_name   │
│  country        │    ┌─────▼──────┐   │  category       │
│  segment        │    │ fact_sales │   │  brand          │
└─────────────────┘    │           ├───►└─────────────────┘
                       │ date_key  │
                       │ customer_key
                       │ product_key
                       │ revenue   │
                       │ quantity  │
                       └───────────┘
```

**Fact table** (`fact_sales`): one row per event/measurement. Mostly foreign keys + numeric measures. Very tall, relatively narrow.

**Dimension tables** (`dim_customer`, `dim_product`, etc.): descriptive attributes. One row per entity. Short, very wide. These fit entirely in memory, making joins cheap — the engine builds a hash table from the small dimension and probes it while scanning the large fact table.

---

## 5.3 Fact tables

### What goes in a fact table

|Column type|Example|Notes|
|---|---|---|
|Foreign keys to dimensions|`customer_key`, `product_key`, `date_key`|Integer surrogate keys, not natural keys|
|Additive measures|`revenue`, `quantity`, `cost`|Can be summed across any dimension|
|Semi-additive measures|`inventory_level`|Can be summed across some dims (store) but not others (time)|
|Non-additive measures|`margin_pct`|Cannot be summed; must be recalculated at query time|
|Degenerate dimensions|`order_id`|Identifiers with no dimension table; stored on the fact|

### What does NOT go in a fact table

- Descriptive text (customer name, product description) → belongs in dimensions
- Attributes that change independently → belongs in dimensions

### Additive vs non-additive — the critical rule

**Additive** measures can be safely summed across any dimension:

```sql
SELECT SUM(revenue) FROM fact_sales WHERE year = 2024;  -- ✅ always correct
```

**Non-additive** measures cannot be summed:

```sql
SELECT SUM(margin_pct) FROM fact_sales;  -- ❌ meaningless number
```

**Rule:** Store the raw numerator and denominator. Compute ratios at query time.

```sql
-- Store revenue and cost on the fact; compute margin at query time
SELECT SUM(revenue - cost) / SUM(revenue) AS margin_pct FROM fact_sales;  -- ✅
```

### Surrogate keys

Fact tables reference dimensions using **integer surrogate keys**, not natural keys.

```
Natural key:  customer_email = 'alice@example.com'  (changes, slow to join)
Surrogate key: customer_key = 4821                  (stable, 8-byte join)
```

Surrogate keys are required for Slowly Changing Dimensions (section 5.6) — you need a stable key to version against.

---

## 5.4 Dimension tables

### Characteristics of a good dimension table

**Wide and denormalized.** `dim_product` should contain `category`, `subcategory`, and `brand` as flat columns — even though they're hierarchically related. Don't create a separate `dim_category` table. That's a snowflake schema and defeats the purpose (see 5.7).

**Self-describing column names.** Prefix with the entity:

```
❌  id, name, city, country
✅  customer_key, customer_name, customer_city, customer_country
```

Un-prefixed columns collide when multiple dimensions are joined in one query.

**No measures.** `dim_customer` should not contain `total_lifetime_revenue`. That's a fact — it changes with every order and must be computed by aggregating `fact_sales`.

### The date dimension

Always build a `dim_date` table. Instead of calling `EXTRACT(year FROM event_timestamp)` at query time (which blocks predicate pushdown), pre-compute all calendar attributes:

```sql
-- dim_date: one row per calendar day
date_key | date       | year | quarter | month | day_of_week | is_weekend | is_holiday | fiscal_quarter
---------|------------|------|---------|-------|-------------|------------|------------|---------------
20240101 | 2024-01-01 | 2024 | 1       | 1     | Monday      | false      | true       | Q1
20240102 | 2024-01-02 | 2024 | 1       | 1     | Tuesday     | false      | false      | Q1
```

`WHERE d.quarter = 1` is a simple integer comparison on a tiny table. No function calls, pushdown works.

---

## 5.5 Grain — the most important design decision

**Grain** is the precise definition of what one row in the fact table represents. Define it before writing any columns.

### Why it matters

The grain determines what questions you can answer, what measures are valid, and whether your aggregations are correct.

### Examples

```
❌ Vague:   "one row per order"
            Does this mean per order header or per line item?

✅ Precise: "one row per order line item"
            Order #88421, Product: Keyboard, Qty: 2, Revenue: $158

✅ Precise: "one row per customer per month (snapshot)"
            Customer 4821, Month: 2024-01, Balance: $1,420
```

### Mixed grain is the #1 modeling error

```
❌ Mixed grain (wrong):
fact_sales row 1: order_id=88421, revenue=158  ← line item grain
fact_sales row 2: order_id=88422, revenue=450  ← order header grain

SUM(revenue) is now wrong — no error message, just bad numbers.
```

If you need both granularities, use two fact tables: `fact_order_lines` and `fact_orders`.

### Prefer atomic grain

Store at the lowest level of detail available. You can always aggregate up; you can never drill below the grain you stored.

```
✅ Store at line item grain → answers both "revenue by product" and "revenue by order"
❌ Store at order grain only → can never answer "which products drove revenue"
```

---

## 5.6 Slowly Changing Dimensions

A **Slowly Changing Dimension (SCD)** is what happens when a dimension attribute changes over time.

### The problem

A customer moves from `country = 'US'` to `country = 'UK'` in March 2024.

You query: "What was total revenue by country in 2023?"

- **Option A:** Revenue attributed to where the customer is **now** (UK) — history is overwritten.
- **Option B:** Revenue attributed to where they were **at the time of purchase** (US in 2023) — history is preserved.

Neither is wrong — it depends on the business question. But the answer determines your SCD type. Get it wrong and you silently produce incorrect historical numbers.

### SCD Type 1 — Overwrite

Simply update the dimension record. No history kept.

```sql
UPDATE dim_customer SET country = 'UK' WHERE customer_key = 4821;
-- All historical fact rows now show 'UK' — 2023 revenue attributed to UK
```

**Use when:** Correcting data errors, or history on this attribute genuinely doesn't matter.

### SCD Type 2 — Add a new row (full history)

Insert a new dimension row per change. Fact rows reference the row that was current at transaction time.

```
customer_key | customer_id | name  | country | valid_from | valid_to   | is_current
-------------|-------------|-------|---------|------------|------------|----------
4821         | C-001       | Alice | US      | 2020-01-01 | 2024-02-28 | false
5103         | C-001       | Alice | UK      | 2024-03-01 | 9999-12-31 | true
```

2023 purchases reference `customer_key = 4821` (US). Post-March 2024 purchases reference `customer_key = 5103` (UK). Historical accuracy is preserved automatically.

**Use when:** Historical accuracy matters — customer segment, pricing tier, geographic attribution.

**Cost:** Dimension table grows with each change. Queries for current state need `WHERE is_current = true`.

### SCD Type 3 — Add a column (limited history)

Add a `previous_country` column. Tracks only one level of history.

**Use rarely.** Only when you need a simple before/after comparison and only one change ever occurs.

### Rule of thumb

**Default to Type 2.** It preserves the most information and is the most analytically honest. Use Type 1 only for error corrections or attributes where history is irrelevant.

---

## 5.7 Snowflake schema vs star schema

A **snowflake schema** normalizes dimension tables further — `dim_product` foreign-keys to `dim_subcategory`, which foreign-keys to `dim_category`.

```
Star:       fact_sales → dim_product (category, subcategory, brand all here)
Snowflake:  fact_sales → dim_product → dim_subcategory → dim_category
```

### Why star is almost always better

|Dimension|Star|Snowflake|
|---|---|---|
|Query joins|One per dimension|Multiple per hierarchy level|
|Performance|Fast (small dim in memory)|Slower (extra joins)|
|Analyst usability|High (flat, obvious columns)|Low (must know to join category tables)|
|Storage savings|—|Negligible (dims are tiny vs fact)|

The storage saved by normalizing a dimension is almost always negligible — dimension tables are small. The cost (extra joins on every query, harder to use) is real.

**Use star by default.** Snowflake only if a dimension table itself is genuinely large (millions of rows) and redundancy is a real storage concern.

---

## 5.8 Common modeling mistakes

**Measures in dimension tables**

```
❌ dim_customer: customer_key, name, country, total_lifetime_revenue
```

`total_lifetime_revenue` changes with every order. Compute it at query time by aggregating `fact_sales`.

**Natural keys as foreign keys in the fact table**

```
❌ fact_sales: order_id, customer_email, product_sku, revenue
```

Strings join slowly. Natural keys change. SCD Type 2 is impossible without surrogate keys.

**Mixed grain in a single fact table** — described in 5.5. Produces wrong aggregates silently.

**Snowflaking dimensions unnecessarily** — described in 5.7. Adds joins, reduces usability.

**Raw timestamps instead of a date dimension**

```
❌ WHERE EXTRACT(year FROM event_timestamp) = 2024  -- function call, no pushdown
✅ JOIN dim_date d ON f.date_key = d.date_key WHERE d.year = 2024  -- pushdown works
```

**Designing the model after data is in production** — A fact table at the wrong grain requires a complete rewrite of history plus all downstream reports. Design before the first write.

---

## 5.9 Postgres → Dimensional Model Translation Table

|Postgres concept|Dimensional model equivalent|Key difference|
|---|---|---|
|Normalized 3NF schema|Star schema (denormalized)|OLAP trades write efficiency for query simplicity|
|Application table (`orders`)|Fact table (`fact_orders`)|Fact table is append-only, keyed by surrogates|
|Lookup / reference table|Dimension table|Dims are wide and denormalized, not normalized reference data|
|Foreign key (natural key)|Surrogate key (integer)|Surrogates are stable across SCD changes|
|`JOIN` across normalized tables|Pre-joined dimension columns|Star schema moves joins to write time, not query time|
|`UPDATED_AT` + row overwrite|SCD Type 1|No history; current value only|
|Append-only audit/history table|SCD Type 2|Full history with `valid_from` / `valid_to` / `is_current`|
|Materialized view (pre-aggregated)|Aggregate fact table|Pre-aggregated facts for common summary queries|
|`PARTITION BY RANGE (created_at)`|Date-partitioned fact table|Same goal; OLAP uses Hive-style file partitions|
|`EXPLAIN` slow JOIN chain|Model redesign (denormalize)|In OLAP, the fix for a slow join is often a model change, not a query hint|

---

[[#Module 5 Summary|↑ Module 5 Summary]] · [[00 OLAP MOC|↑ TOC]] · [[05 Module 4 — Partitioning & Data Layout|← Module 4]]

---

_End of Module 5._

> **Suggested next steps:**
> 
> - **Module 6 — Aggregate Design & Pre-computation:** When to build aggregate fact tables, materialized views in DuckDB, and balancing query flexibility against pre-computation cost.
> - **Module 7 — DuckDB in Practice:** Connect layout (Module 4) and modeling (Module 5) to real DuckDB queries — `EXPLAIN ANALYZE` on a star schema, partitioned reads, and join ordering behavior.