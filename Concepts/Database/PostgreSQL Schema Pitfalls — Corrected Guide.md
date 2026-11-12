# PostgreSQL Schema Pitfalls — Corrected Guide

> Schema decisions made in five minutes can define performance characteristics for years. These problems don't throw errors — they show up months later, with no commit to blame.

---

## 1. TEXT vs VARCHAR

**The setup:** Most people think `TEXT` is "unlimited VARCHAR" and treat the choice as style. Mostly true, but incomplete.

**How storage actually works:** `TEXT`, `VARCHAR(n)`, and `CHAR(n)` all use the same on-disk format (`varlena` — a length header plus payload). A `VARCHAR(50)` and a `TEXT` column holding the same string produce **identical bytes on disk**. The `(n)` in `VARCHAR(n)` is purely a write-time length check — it has zero storage or index benefit.

**TOAST:** any value over roughly 2 KB gets compressed and/or moved out of the main row into a separate TOAST table, with a pointer left behind. This means:

- reading a large value costs an extra fetch (row → pointer → TOAST table)
- TOASTed values can't be indexed directly (you'd be indexing a pointer)
- compression costs CPU on every read, not just once

**Why unconstrained `TEXT` is dangerous:** without a length limit, a column has no enforced contract. Consequences compound over time:

- PostgreSQL's B-tree index has a hard ceiling — an indexed value can't exceed about 1/3 of a page (~2712 bytes on the default 8 KB page). Rows with longer values are silently skipped from a plain index, no error.
- the query planner has no length bound to reason about, so its row-count estimates get worse as content varies more.
- application code drifts — a `notes TEXT` column meant for short comments slowly starts receiving JSON, HTML, and UUIDs from different code paths, because nothing stops it.

**Fix:**

```sql
-- Enforce meaning, don't just pick a type
notes VARCHAR(1000)                              -- built-in limit
notes TEXT CHECK (char_length(notes) <= 1000)     -- equivalent, explicit
body TEXT                                         -- fine for genuinely unbounded content,
                                                   -- but pair with a GIN/tsvector index for search
```

Avoid `CHAR(n)` entirely — it pads with spaces and causes comparison bugs, with no upside.

**Lesson learned:** the type name isn't the decision. The decision is whether you're enforcing a length constraint at the schema level. `VARCHAR(n)` and `TEXT + CHECK` are equally valid ways to do that; bare `TEXT` with no constraint is how columns become dumping grounds.

---

## 2. NULL Inside UNIQUE Constraints

**The trap:** in SQL, `NULL` is never equal to anything, including another `NULL`. A multi-column `UNIQUE` constraint treats any row with a `NULL` in a constrained column as distinct from every other row — even one with `NULL` in the exact same column.

```sql
CREATE TABLE campaign_codes (
    user_id    INT  NOT NULL,
    campaign   TEXT NOT NULL,
    promo_code TEXT,                 -- nullable
    UNIQUE (user_id, campaign, promo_code)
);

INSERT INTO campaign_codes VALUES (1, 'summer', NULL);
INSERT INTO campaign_codes VALUES (1, 'summer', NULL);  -- succeeds, no violation
INSERT INTO campaign_codes VALUES (1, 'summer', NULL);  -- succeeds again
-- 3 "duplicate" rows, constraint never complained
```

**Where it bites:**

- `COUNT(DISTINCT ...)` overcounts
- `INSERT ... ON CONFLICT DO UPDATE` silently inserts new rows instead of updating, whenever the conflict key includes a NULL
- joins on the "unique" key produce duplicated rows downstream

**Fixes (pick one, deliberately):**

```sql
-- 1. Sentinel value instead of NULL
promo_code TEXT NOT NULL DEFAULT ''

-- 2. Partial unique index — NULLs allowed, non-null values still unique
CREATE UNIQUE INDEX ON campaign_codes (user_id, campaign, promo_code)
WHERE promo_code IS NOT NULL;

-- 3. PostgreSQL 15+: make NULL count as equal to NULL for this constraint
CREATE UNIQUE INDEX ON campaign_codes (user_id, campaign, promo_code)
NULLS NOT DISTINCT;
```

**Lesson learned:** a `UNIQUE` constraint on a nullable column is not a full uniqueness guarantee unless you've explicitly decided what NULL should mean for it. Silent — no error, no log — so it has to be caught by design, not by testing.

---

## 3. Unnecessary Nullability

**The default:** PostgreSQL columns are nullable unless you say `NOT NULL`. That default is a historical SQL-standard artifact, not a recommendation — and it means schemas drift toward "nullable everywhere" unless someone actively pushes back.

**What nullability actually costs:**

- a small, mostly negligible per-row bitmap overhead
- planner conservatism: nullable columns require the planner to account for NULLs in joins, filters, `COUNT(col)` vs `COUNT(*)`, and sort order
- application complexity: every nullable column becomes an `Optional<T>` or a null-check somewhere downstream — the schema's laziness becomes the app's problem

**The test before allowing NULL:**

1. What does NULL _mean_ here, in one sentence?
2. Does anything ever query `WHERE col IS NULL`?
3. Is there a sensible non-null default (empty string, zero, sentinel, `NOW()`) that would actually be more correct?

```sql
-- Default-everything-nullable
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id INT,
    amount NUMERIC,
    status TEXT,
    created_at TIMESTAMPTZ,
    notes TEXT
);

-- Deliberate
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id INT         NOT NULL,
    amount      NUMERIC     NOT NULL CHECK (amount >= 0),
    status      TEXT        NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    notes       TEXT        -- genuinely optional
);
```

**Retrofitting `NOT NULL` without locking the table (PG 12+):**

```sql
ALTER TABLE orders ADD CONSTRAINT orders_customer_id_not_null
    CHECK (customer_id IS NOT NULL) NOT VALID;         -- instant, no scan

ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_id_not_null;  -- scans, doesn't block writes

ALTER TABLE orders ALTER COLUMN customer_id SET NOT NULL;  -- now cheap, constraint already proven
ALTER TABLE orders DROP CONSTRAINT orders_customer_id_not_null;      -- redundant now
```

**Lesson learned:** nullable should be an opt-in decision per column, not the path of least resistance. If you can't state what NULL means for a column in one sentence, it probably shouldn't be nullable.

---

## 4. JSON vs JSONB vs TEXT — and why they're not just "two flavors of JSON"

This is the part most explanations gloss over, so here it is explicitly:

||`TEXT`|`JSON`|`JSONB`|
|---|---|---|---|
|What's stored|Raw string, PostgreSQL has no idea it's JSON|The exact raw JSON string, byte-for-byte|A parsed, decomposed binary structure — not a string at all|
|Validates it's valid JSON|No|Yes, on write|Yes, on write|
|Preserves key order / whitespace / duplicate keys|N/A|Yes — reproduces input exactly|No — keys are sorted, duplicates collapsed (last wins), whitespace stripped|
|Can you index _into_ it|No|No (only expression indexes by re-parsing each time)|Yes — GIN indexes on the whole document, B-tree on extracted paths|
|Operators (`->`, `->>`, `@>`, `?`)|No|Yes, but each call re-parses the text|Yes, and efficiently — no re-parsing|
|Write cost|Cheapest|Cheap (just validates)|Slightly more expensive (parses + decomposes)|

**The one-sentence version:** `JSON` stores your text as text and just double-checks it parses; `JSONB` throws away the text and stores a structure instead — which is why `JSONB` can be indexed and queried efficiently, and `JSON` basically can't. Almost nobody has a real reason to reach for plain `JSON` — the only case is when you need to preserve the exact original formatting or key order (e.g., an audit log of a client payload), which `JSONB` cannot do.

**When JSONB is the right call:** the structure genuinely varies row to row (a product catalog where attributes differ by category), and you actually query into the document (`->`, `@>`, `?`).

```sql
CREATE TABLE products (
    id         BIGSERIAL PRIMARY KEY,
    name       TEXT  NOT NULL,
    category   TEXT  NOT NULL,
    attributes JSONB NOT NULL DEFAULT '{}'
);

CREATE INDEX ON products ((attributes->>'color')) WHERE attributes ? 'color';  -- path lookup
CREATE INDEX ON products USING GIN (attributes);                              -- containment queries
```

**When JSONB is the wrong call — using it to dodge modeling:**

```sql
-- Every row has the same keys, nobody ever queries into it, no GIN index exists
CREATE TABLE users (
    id   BIGSERIAL PRIMARY KEY,
    data JSONB NOT NULL  -- always {"email": ..., "name": ..., "role": ...}
);
```

If every row has the same shape and the app always reads the whole blob and deserializes it in code, JSONB is just `TEXT`with extra write overhead and false "flexibility." Use real columns:

```sql
CREATE TABLE users (
    id    BIGSERIAL PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    name  TEXT NOT NULL,
    role  TEXT NOT NULL DEFAULT 'user'
);
CREATE INDEX ON users (role);
```

**Index choice within JSONB:** GIN indexes the whole document (good for `@>` containment and `?` key-existence, but bigger and slower to write); a B-tree on an extracted path (`(doc->>'status')`) is small and fast but only helps equality/range queries on that one field.

**Lesson learned:** the real fork isn't `JSON` vs `JSONB` — plain `JSON` is rarely the right choice at all. The real fork is `JSONB` vs real columns, and the tell for "you should be using real columns" is: same keys every row, no queries into the document, no GIN index.

---

## 5. UUID v4 as a Primary Key at Scale

**Why UUIDs look appealing:** globally unique without a round-trip to the DB, opaque (no record count or insert order leaked), portable across systems, safe in URLs. All real. The problem is specifically **random (v4)** UUIDs, not UUIDs in general.

**Why it hurts at scale:** a B-tree index is optimized for appending to the end — sequential IDs land on the rightmost page almost every time. Random UUIDv4 values land uniformly across the entire key space, so:

- **page splits** happen constantly instead of rarely, and each split is a real write cost
- **index bloat** follows — split pages end up half-full and often never refill before splitting again
- **cache thrashing** — every insert may touch a page that isn't in memory, forcing a disk read before the write can even happen
- the effect is invisible at 10K rows, noticeable around 10M, and significant by 100M+

**Fixes, in order of preference for new projects:**

|Option|Sequential|Size|Client-generatable|Notes|
|---|---|---|---|---|
|`BIGSERIAL`|Yes|8 B|No|Simplest, fastest; exposes count/order|
|UUID v7|Yes (ms-ordered)|16 B|Yes|Time-ordered, keeps UUID's benefits without the scatter|
|ULID|Yes (ms-ordered)|16 B / 26 chars|Yes|Same idea as v7, different encoding|
|UUID v4|No|16 B|Yes|Fine for small tables; costly at scale|

**Corrected version note:** `gen_random_uuid()` (UUID v4) has been a built-in PostgreSQL function since **version 13** — no extension required. Native **UUID v7 generation** (`uuidv7()`) was added to PostgreSQL in **version 18**. On any version before 18, generating v7 requires a third-party extension (e.g. `pg_uuidv7`) or application-side generation.

```sql
-- PostgreSQL 18+
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT uuidv7(),
    payload JSONB NOT NULL
);

-- Pre-18: use an extension, or generate v7 UUIDs in application code
```

**Lesson learned:** UUID as a concept isn't the performance problem — randomness is. If you need UUID-like properties (global uniqueness, client-side generation, opacity) at scale, prefer a time-ordered variant (v7 or ULID) over v4, and know which PostgreSQL version you're targeting before assuming native support.

---

## Summary Table

|Pattern|What breaks|When it shows up|Fix|
|---|---|---|---|
|Unbounded `TEXT`|Index size limits, column drift, bad planner estimates|Gradually, as data grows|`CHECK (char_length(...) <= n)` or `VARCHAR(n)`|
|NULL in `UNIQUE`|Duplicate rows, silent upsert failures|On the first NULL insert|`NOT NULL` + sentinel, partial index, or `NULLS NOT DISTINCT` (PG 15+)|
|Unnecessary nullable columns|Planner overhead, app-level null handling everywhere|Permanent, low-grade tax|Default `NOT NULL`; nullable only when absence is meaningful|
|`JSONB` instead of real columns|No real indexing, untyped queries|As query complexity grows|Model actual columns; reserve `JSONB` for genuinely variable structure|
|UUID v4 primary key|Index bloat, cache misses, slow writes|Roughly 10M+ rows|UUID v7 (PG 18+), ULID, or `BIGSERIAL`|

None of these fail loudly. They degrade quietly until the table is big enough that the cost is obvious — and by then it's a migration, not a one-line change. **Make the decision at schema time, not incident time.**