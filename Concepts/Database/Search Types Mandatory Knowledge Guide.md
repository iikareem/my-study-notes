---
tags:
  - database
  - full-text
  - gin
  - indexing
  - postgresql
  - search
---

# Search Types: Mandatory Knowledge Guide

---

## Overview

Search is not one-size-fits-all. There are three major search categories, each solving a different user problem.

|Category|User Problem|Example|
|---|---|---|
|**Fuzzy / Typo-tolerant**|User makes a spelling mistake|`"Postgress"` → PostgreSQL|
|**Full-Text Search**|User looks for keywords in content|`"error handling"` in blog posts|
|**Semantic / Meaning-based**|User uses different words with same intent|`"fix a bug"` ≈ `"resolve an issue"`|

---

## Category 1 — Fuzzy & Typo-Tolerant Search

> **Problem:** User misspells a word. Exact match fails completely.

### How it works

Measures how "close" two strings are by counting character edits (insertions, deletions, substitutions). This is called **edit distance** (Levenshtein distance).

- `"Postgress"` → `"PostgreSQL"` = 2 edits → close enough → match ✅
- `"John"` → `"Jon"` = 1 edit → match ✅

### Variants

|Type|Description|
|---|---|
|**Levenshtein / Edit Distance**|Count of character-level changes|
|**Soundex / Metaphone**|Match by how words _sound_ (phonetic)|
|**N-gram similarity**|Split words into small chunks and compare overlap|
|**Trigram**|Special case of n-gram (chunks of 3 chars); most common in DBs|

### In PostgreSQL

- Extension: **`pg_trgm`** (trigram matching)
- Operators: `%` (similarity), `<->` (distance)
- Index: **GIN or GiST** on `gin_trgm_ops`

```sql
-- Enable extension
CREATE EXTENSION pg_trgm;

-- Create index
CREATE INDEX idx_trgm ON products USING GIN (name gin_trgm_ops);

-- Query: fuzzy match on name
SELECT * FROM products
WHERE name % 'Postgress'  -- similarity threshold (default 0.3)
ORDER BY name <-> 'Postgress';
```

### External Solutions

|Tool|Notes|
|---|---|
|**Elasticsearch**|`fuzzy` query, fuzziness: `AUTO`|
|**Typesense**|Built-in typo tolerance, very simple to configure|
|**Meilisearch**|Auto typo-tolerance, zero config|
|**Algolia**|Managed, automatic fuzzy matching|

---

## Category 2 — Full-Text Search (FTS)

> **Problem:** User searches for keywords across large bodies of text (articles, descriptions, logs).

### How it works

Tokenizes text into terms, removes stopwords (`the`, `is`, `a`), applies stemming (`running` → `run`), and builds an inverted index mapping terms to documents.

- Query: `"error handling"` → matches any doc containing `error` OR `handling` (or both, ranked)
- Supports: AND/OR logic, phrase matching, ranking by relevance (TF-IDF / BM25)

### Key Concepts

|Concept|Meaning|
|---|---|
|**Tokenization**|Split text into individual words/terms|
|**Stemming**|Reduce word to root form (`errors` → `error`)|
|**Stopwords**|Common words ignored (`the`, `is`, etc.)|
|**Inverted Index**|Map of term → list of documents containing it|
|**Ranking (BM25)**|Score documents by term frequency + rarity|

### In PostgreSQL

- Native support via `tsvector` / `tsquery` types
- Index: **GIN on tsvector column**

```sql
-- Add a tsvector column (or compute on-the-fly)
ALTER TABLE articles ADD COLUMN fts tsvector
  GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || body)) STORED;

-- Create GIN index
CREATE INDEX idx_fts ON articles USING GIN (fts);

-- Query
SELECT * FROM articles
WHERE fts @@ to_tsquery('english', 'error & handling')
ORDER BY ts_rank(fts, to_tsquery('english', 'error & handling')) DESC;
```

### External Solutions

|Tool|Notes|
|---|---|
|**Elasticsearch / OpenSearch**|Industry standard for FTS; BM25 ranking built-in|
|**Typesense**|Lightweight FTS with simple API|
|**Meilisearch**|FTS with great defaults out of the box|
|**Solr**|Mature, complex, battle-tested|

---

## Category 3 — Semantic / Meaning-Based Search

> **Problem:** User uses different words that mean the same thing. Keyword search fails entirely.

### How it works

Converts text into a **vector (embedding)** — a list of numbers that captures _meaning_. Similar meanings produce vectors that are mathematically close to each other.

- `"fix a bug"` and `"resolve an issue"` → similar vectors → matched ✅
- `"car"` and `"automobile"` → similar vectors → matched ✅
- No keyword overlap needed

You use an **embedding model** (e.g., OpenAI `text-embedding-3-small`, Sentence Transformers) to generate these vectors, then find the closest ones using **vector similarity** (cosine similarity or dot product).

### Key Concepts

|Concept|Meaning|
|---|---|
|**Embedding**|Dense numerical representation of text meaning|
|**Vector similarity**|Cosine / dot product distance between two embeddings|
|**ANN (Approximate Nearest Neighbor)**|Fast "close enough" search algorithm (exact match is too slow at scale)|
|**Chunking**|Splitting large docs into segments before embedding|

### In PostgreSQL

- Extension: **`pgvector`**
- Index: **IVFFlat or HNSW** (both approximate nearest neighbor)

```sql
-- Enable extension
CREATE EXTENSION vector;

-- Store embeddings (e.g., 1536 dimensions for OpenAI)
ALTER TABLE articles ADD COLUMN embedding vector(1536);

-- Create HNSW index (best for most use cases)
CREATE INDEX idx_vec ON articles USING hnsw (embedding vector_cosine_ops);

-- Query: find 5 most semantically similar articles
SELECT id, title,
       1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM articles
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 5;
```

> `<=>` = cosine distance | `<->` = L2 distance | `<#>` = negative dot product

### External Solutions

|Tool|Notes|
|---|---|
|**Pinecone**|Managed vector DB, easiest to start|
|**Weaviate**|Open-source vector DB, supports hybrid search|
|**Qdrant**|Fast, open-source, Rust-based|
|**Chroma**|Lightweight, great for local/dev use|
|**Elasticsearch**|Added `dense_vector` field + kNN search|
|**Redis**|`RediSearch` module supports vector search|

---

## Hybrid Search — Combining All Three

In real-world systems, you often combine approaches:

```
User query
    │
    ├─ Fuzzy match     → handle typos
    ├─ Full-text (BM25) → keyword relevance
    └─ Semantic (vector)→ meaning similarity
         │
         └─ Re-rank results (RRF or custom scoring)
```

**Reciprocal Rank Fusion (RRF)** is the standard algorithm for merging ranked lists from different search methods.

Tools with built-in hybrid search: **Weaviate**, **Elasticsearch**, **Typesense** (v0.25+), **Qdrant**.

---

## Quick Decision Guide

|Situation|Use|
|---|---|
|Users typo product names|Fuzzy (`pg_trgm`, Typesense, Meilisearch)|
|Search inside articles/descriptions|Full-Text (`tsvector` + GIN, Elasticsearch)|
|Synonyms, paraphrasing, intent matching|Semantic (`pgvector` + HNSW, Pinecone)|
|All of the above|Hybrid (Elasticsearch or Weaviate)|
|Small project, stay in Postgres|`pg_trgm` + `tsvector` + `pgvector`|
|Scale / managed / no ops|Typesense, Meilisearch, or Algolia|

---

## Index Summary

| Extension/Index          | Search Type | Best For                                  |
| ------------------------ | ----------- | ----------------------------------------- |
| `GIN` + `gin_trgm_ops`   | Fuzzy       | Trigram similarity in PG                  |
| `GiST` + `gist_trgm_ops` | Fuzzy       | Fuzzy, slower writes but faster range     |
| `GIN` on `tsvector`      | Full-Text   | Keyword search in PG                      |
| `IVFFlat` (pgvector)     | Semantic    | Vector search, faster build               |
| `HNSW` (pgvector)        | Semantic    | Vector search, faster queries ✅ preferred |
