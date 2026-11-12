# `LIMIT 1` Does Not Mean Postgres Reads One Row

## The problem

I recently worked on a function that read stream entries. The query had `LIMIT 1` and the right index, yet it still read **1,990 rows** to return one.

The pattern looks like this:

```sql
SELECT ...
FROM parent p
JOIN child c ON c.parent_id = p.id
ORDER BY p.created_at
LIMIT N;
```

The question to ask is: **what does the `LIMIT` apply to?**

It applies to the **final joined, sorted result**. So Postgres may still need to:

- produce many joined rows
- sort them
- throw almost all of them away
- return only N

## Why it can't stop early

A sort can't return its first row until it has seen its last input row. So `LIMIT` on top of a sort caps what you *get back*, not what got *read*.

## Why joins make it worse

Every one-to-many join multiplies rows. With several joins, the row count grows at each step, and then the sort and limit run on that big multiplied set.

## The fix: order and limit before joining

### 1. For the parent: use a subquery in `FROM`

Do the `ORDER BY` + `LIMIT` on the parent first, then join:

```sql
SELECT ...
FROM (
  SELECT *
  FROM parent
  ORDER BY created_at
  LIMIT N
) p
JOIN child c ON c.parent_id = p.id;
```

Postgres can walk the index in order, stop after N rows, and only then join.

### 2. For related tables: use `LATERAL`

If each parent needs its own related rows (for example, the latest child per parent), use a lateral (correlated) join:

```sql
SELECT ...
FROM (
  SELECT *
  FROM parent
  ORDER BY created_at
  LIMIT N
) p
LEFT JOIN LATERAL (
  SELECT *
  FROM child c
  WHERE c.parent_id = p.id
  ORDER BY c.created_at DESC
  LIMIT 1
) c ON true;
```

Each parent runs a small, indexed, limited lookup instead of joining everything and sorting afterward.

## The result

Same `LIMIT N`, very different amount of work. In my case it went from **27 buffers to 4**, and the query cost stopped growing with the size of the table.

## Caveat

Limiting before the join isn't always equivalent. An inner join can drop parents, and a one-to-many join can return more than N rows in total. Make sure the rewrite matches what you actually want.

## Takeaway

Always check **when** sorting, filtering, and limiting happen. Use `EXPLAIN (ANALYZE, BUFFERS)` to see how much is actually read.