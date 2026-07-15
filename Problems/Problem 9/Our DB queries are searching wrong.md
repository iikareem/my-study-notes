---
tags:
  - database
  - full-text
  - indexing
  - interview
  - problem-9
  - problems
  - query-optimization
  - search
  - system-design
---

# Problem 9 — "Our search is so slow users think it's broken"

**Domain:** Database Design & Query Optimization  
**Topics:** full-text search · indexing · LIKE pitfalls · GIN / tsvector

**In this folder:** scenario (this note) · [[Problem 9 Companion]]

## 🧩 The Scenario

You run a B2B SaaS platform. Users can search their customers by name, email, phone, company, or any combination. The search endpoint:

```python
def search_customers(user_id, query, filters=None):
    sql = """
        SELECT *
        FROM customers
        WHERE account_id = ?
          AND (
              name    LIKE ?
           OR email   LIKE ?
           OR phone   LIKE ?
           OR company LIKE ?
           OR notes   LIKE ?
          )
    """
    pattern = f"%{query}%"

    results = db.query(sql, user_id, pattern, pattern, pattern, pattern, pattern)
    return results
```

The `customers` table has 40 million rows across all accounts. Some enterprise accounts have 500,000 customers each.

Problems reported:

- **Problem A** — Search takes 8-12 seconds for enterprise accounts
- **Problem B** — Searching `"jo"` returns 180,000 results — useless to the user and kills the DB
- **Problem C** — A user searches for `"John Smith at Acme"` — gets zero results even though John Smith at Acme Corp exists
- **Problem D** — Searching `"resume"` doesn't find customers whose notes contain `"résumé"` — users think data is missing
- **Problem E** — The product team wants relevance ranking — exact matches should appear before partial matches — impossible with the current approach

Your manager says: _"We need real search."_ What does real search look like and why is `LIKE '%query%'` fundamentally the wrong tool?

---

## 🤔 Think About It First

Why can a B-Tree index not help a `LIKE '%query%'` query? What data structure is designed specifically for text search? And what's the architectural decision between doing this inside Postgres vs a dedicated search engine?

---

## 🔍 Root Cause Analysis

### Why `LIKE '%query%'` is Fundamentally Broken

```sql
WHERE name LIKE '%john%'
```

The `%` at the **start** of the pattern means:

> _"The match can start anywhere in the string"_

A B-Tree index stores values in sorted order:

```
B-Tree on name column:
Aaron Smith
Adam Jones
Beth Taylor
John Anderson   ← 'john' starts here
John Smith      ← and here
Johnathan Brown ← and here
...
Zara Williams
```

For `LIKE 'john%'` (prefix search) the index works — seek to `john`, scan forward.

For `LIKE '%john%'` (infix search) — the match could be **anywhere in the string.** There's no seek point. The database must read every single row and check it character by character:

```
40 million rows × string scan = sequential scan every time
index is completely useless
```

This is a **fundamental limitation of B-Tree indexes** — not a tuning problem. No amount of indexing tricks fixes `LIKE '%query%'`.

---

### Why Each Problem Exists

**Problem A — 8-12 seconds:** Sequential scan of 500,000 rows per account, 5 columns each, no index possible.

**Problem B — 180,000 results for "jo":** No minimum length enforcement, no relevance ranking — every row containing "jo" anywhere returns equally.

**Problem C — "John Smith at Acme" returns nothing:** `LIKE` does exact substring matching. It can't parse natural language intent — it looks for the literal string `"john smith at acme"` as a substring, not the concepts within it.

**Problem D — "resume" doesn't find "résumé":** `LIKE` is byte-level comparison. `e` ≠ `é` unless you explicitly configure collation — and even then accent-insensitive `LIKE` is slow and limited.

**Problem E — No relevance ranking:** `LIKE` returns boolean — match or no match. There's no score, no concept of "this row matches better than that row."

---

## ✅ Solution Path 1 — PostgreSQL Full-Text Search

Before reaching for Elasticsearch, Postgres has a surprisingly powerful built-in full-text search engine. For many use cases it's enough.

### How It Works — The Inverted Index

Full-text search uses an **inverted index** — the opposite of a B-Tree:

```
B-Tree index:       row → content
Inverted index:     word → [rows containing that word]

Example:
"john"   → [row 4, row 17, row 203, row 4891 ...]
"smith"  → [row 17, row 445, row 4891 ...]
"acme"   → [row 203, row 4891, row 7823 ...]

Query "john smith":
→ intersect(rows for "john", rows for "smith")
→ [row 17, row 4891]
= instant lookup, no sequential scan
```

---

### Implementation in Postgres

```sql
-- Step 1 — add a tsvector column (pre-computed search tokens)
ALTER TABLE customers
ADD COLUMN search_vector tsvector;

-- Step 2 — populate it (weighted: name most important, notes least)
UPDATE customers SET search_vector =
    setweight(to_tsvector('english', coalesce(name,    '')), 'A') ||
    setweight(to_tsvector('english', coalesce(email,   '')), 'B') ||
    setweight(to_tsvector('english', coalesce(company, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(phone,   '')), 'C') ||
    setweight(to_tsvector('english', coalesce(notes,   '')), 'D');

-- Step 3 — create GIN index on the vector (GIN = inverted index)
CREATE INDEX idx_customers_search
ON customers USING GIN(search_vector);

-- Step 4 — keep it updated automatically
CREATE OR REPLACE FUNCTION customers_search_vector_update()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.name,    '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.email,   '')), 'B') ||
        setweight(to_tsvector('english', coalesce(NEW.company, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(NEW.phone,   '')), 'C') ||
        setweight(to_tsvector('english', coalesce(NEW.notes,   '')), 'D');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER customers_search_vector_trigger
BEFORE INSERT OR UPDATE ON customers
FOR EACH ROW EXECUTE FUNCTION customers_search_vector_update();
```

---

### The Search Query — With Relevance Ranking

```python
def search_customers(account_id, query, limit=20, offset=0):
    if len(query.strip()) < 2:
        raise ValidationError("Query too short")

    # Convert query to tsquery — handles multiple words
    # 'john smith' → 'john' & 'smith' (both must appear)
    results = db.query("""
        SELECT
            id,
            name,
            email,
            company,
            phone,
            ts_rank(search_vector, query) as rank
        FROM
            customers,
            to_tsquery('english', ?) query
        WHERE
            account_id = ?
            AND search_vector @@ query
        ORDER BY rank DESC
        LIMIT ?
        OFFSET ?
    """,
        to_tsquery_string(query),
        account_id,
        limit,
        offset
    )

    return results

def to_tsquery_string(query):
    # "john smith" → "john & smith"
    # "john smith acme" → "john & smith & acme"
    words = query.strip().split()
    return " & ".join(words)
```

---

### What This Solves

**Problem A — Speed:** GIN index lookup instead of sequential scan. Sub-100ms for 500,000 rows.

**Problem B — Too many results:** Full-text search tokenizes and stems — `"jo"` alone doesn't match `"john"` unless you use prefix matching explicitly. Add minimum query length validation.

**Problem C — Natural language:** `"john smith acme"` becomes `john & smith & acme` — finds rows containing all three words anywhere across all columns.

**Problem D — Accents:** `to_tsvector` with the right dictionary handles accent normalization. `résumé` and `resume` tokenize to the same stem.

**Problem E — Relevance ranking:** `ts_rank` scores results by how well they match — name matches (weight A) outrank notes matches (weight D) automatically.

---

### The `tsvector` — What's Actually Stored

```sql
SELECT to_tsvector('english', 'John Smith works at Acme Corporation');

-- Output:
'acm':5 'corpor':6 'john':1 'smith':2 'work':3

-- Notice:
-- "at" removed (stop word)
-- "works" → "work" (stemmed)
-- "Corporation" → "corpor" (stemmed)
-- position numbers stored for proximity ranking
```

This is why `résumé` → `resumé` → `resum` (stemmed) matches `resume` → `resum`. Same stem.

---

## ✅ Solution Path 2 — Dedicated Search Engine (Elasticsearch / Typesense / Meilisearch)

When Postgres full-text search isn't enough:

```
Use Postgres FTS when:                Use dedicated search engine when:
───────────────────────               ──────────────────────────────────
Data already in Postgres              Fuzzy matching needed ("Jon" finds "John")
Search is secondary feature           Typo tolerance is critical
< 10M rows per search scope           Sub-10ms search at massive scale
Simple relevance ranking enough       Complex relevance tuning needed
Don't want extra infrastructure       Faceted search (filter by category + search)
                                      Multi-language support at scale
                                      Synonyms ("car" finds "automobile")
```

### Architecture with Elasticsearch

```python
# On customer create/update — sync to Elasticsearch
def sync_to_search(customer):
    es.index(
        index="customers",
        id=customer.id,
        body={
            "account_id": customer.account_id,
            "name":       customer.name,
            "email":      customer.email,
            "company":    customer.company,
            "phone":      customer.phone,
            "notes":      customer.notes,
            "created_at": customer.created_at
        }
    )

# Search via Elasticsearch
def search_customers(account_id, query, limit=20):
    response = es.search(
        index="customers",
        body={
            "query": {
                "bool": {
                    "must": [
                        {"term":  {"account_id": account_id}},
                        {"multi_match": {
                            "query":  query,
                            "fields": [
                                "name^4",      # name matches weighted 4x
                                "email^3",
                                "company^2",
                                "phone",
                                "notes"
                            ],
                            "fuzziness": "AUTO",   # "Jon" finds "John"
                            "type": "best_fields"
                        }}
                    ]
                }
            },
            "size": limit
        }
    )

    return [hit["_source"] for hit in response["hits"]["hits"]]
```

---

### The Sync Problem — Keeping Search Index Fresh

This is the hardest part of dedicated search engines:

```
DB is source of truth
Search index is a derived copy

→ they can diverge
```

```python
# Option A — Synchronous sync (simple, but couples write latency to ES)
def update_customer(customer_id, data):
    db.execute("UPDATE customers SET ... WHERE id = ?", customer_id)
    sync_to_search(customer)          # ← if ES is down, write fails

# Option B — Async via event/outbox (decoupled, small lag)
def update_customer(customer_id, data):
    db.execute("UPDATE customers SET ... WHERE id = ?", customer_id)
    outbox.publish("customer.updated", {"customer_id": customer_id})
    # ES sync happens asynchronously — small lag but write always succeeds

# Option C — CDC (Change Data Capture)
# Debezium reads Postgres WAL and streams changes to ES automatically
# Zero application code change, guaranteed delivery
# Most robust for large scale
```

---

## 📊 Approach Comparison

||LIKE '%query%'|Postgres FTS|Elasticsearch|
|---|---|---|---|
|Speed|❌ Sequential scan|✅ GIN index|✅ Inverted index|
|Fuzzy / typo tolerance|❌ None|❌ Limited|✅ Built-in|
|Relevance ranking|❌ None|✅ ts_rank|✅ BM25 scoring|
|Accent normalization|❌ Manual|✅ Dictionary|✅ Analyzers|
|Natural language|❌ None|✅ Stemming|✅ Stemming + synonyms|
|Infrastructure|✅ None|✅ None|❌ Extra service|
|Consistency|✅ Always fresh|✅ Always fresh|⚠️ Eventual|
|Operational cost|✅ Zero|✅ Zero|❌ High|

---

## 🧠 The Right Migration Path

```
Phase 1 — Quick win (1 week)
  → Add tsvector column + GIN index to Postgres
  → Replace LIKE with @@ operator
  → Add minimum query length validation
  → Result: 10x speed improvement, relevance ranking, accent support

Phase 2 — If needed (later)
  → Evaluate if Postgres FTS covers all requirements
  → Only add Elasticsearch if you need fuzzy matching,
    faceted search, or sub-10ms at 100M+ rows
  → Use CDC (Debezium) for sync — don't write sync code yourself
```

Don't reach for Elasticsearch first. Most teams add it prematurely and spend months managing infrastructure that Postgres FTS would have covered.

---

## 🔑 Key Takeaways

|Lesson|Rule|
|---|---|
|`LIKE '%query%'`|Never use for search — B-Tree can't help, always sequential scan|
|Inverted index|The data structure search is built on — word → rows, not row → words|
|GIN index|Postgres index type for full-text search — use `USING GIN`|
|tsvector|Pre-computed token representation — update via trigger, index via GIN|
|ts_rank|Relevance scoring built into Postgres — weight fields by importance|
|Stemming|"works" = "work" = "worked" — same stem, same search token|
|Postgres vs ES|Start with Postgres FTS — add ES only when you need fuzzy or faceting|
|Search sync|Outbox or CDC for ES sync — never synchronous in the write path|

---

## Next

- Companion deep dive: [[Problem 9 Companion]]
- Back to hub: [[Problems MOC]]
