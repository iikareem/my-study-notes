---
tags:
  - database
  - exists
  - joins
  - postgresql
  - query-optimization
  - sql
---

### `WHERE IN`

- Used to filter rows where a column's value is **in a given list or subquery**.
- It executed first in the execution time and If it's a subquery, PostgreSQL materializes the result into memory
- ORs between the results that search it in the list
    
- If the subquery contains **`NULL`**, it may return **no results at all**, even if valid matches exist.
    
- Does **not short-circuit** — it checks the entire list even if a match is found.
    
- Best for **small, static lists** (e.g., `WHERE id IN (1, 2, 3)`).
    
- **Avoid** with large subqueries or possible `NULL` values.

### `WHERE EXISTS`

- Used to filter rows where a **matching row exists** in a subquery.
    
- Evaluates the subquery **for each row** in the outer query if there (correlated subquery). can be evaluated in the first of execution if there not correlated subquery
    
- **Short-circuits** — stops at the first match.
    
- **Safe with `NULL`s** — it doesn't break like `IN`.
    
- Optimized by PostgreSQL as an **anti join** when using `NOT EXISTS`.
    
- Best for **relationship-based filtering** (e.g., check if related rows exist).

### `NOT IN`

- Used to exclude rows with values found in a subquery or list.
    
- **Dangerous with `NULL` values** — if the subquery returns a single `NULL`, **all rows will be excluded** (unexpected behavior).
    
- Avoid using this when subquery might contain `NULL`s.
    
- PostgreSQL does **not optimize** `NOT IN` well if `NULL`s are present.
- AND's operator to get the row so that can make a problem in the null values ( unknown )
  
SELECT 1 WHERE 1 NOT IN (2,3,4, null)  
2 != 1 TRUE  
AND  
3 != 1 TRUE  
AND  
4 != 1 TRUE  
AND  
null ( unkown) != 1 FALSE   
= FALSE

### `NOT EXISTS`

- Used to exclude rows where **a matching row exists** in the subquery.
    
- **NULL-safe** and more predictable than `NOT IN`.
    
- PostgreSQL often turns this into an **anti join** for optimization.
    
- Recommended over `NOT IN` in most cases, especially with `JOIN`s or conditions.

### `EXCEPT`

- A **set operation** that returns rows from the first query that are **not in the second**.
    
- Removes **duplicates** unless `EXCEPT ALL` is used.
    
- Safe with `NULL` — behaves as expected.
    
- Best used for comparing **flat sets of values** (e.g., `SELECT id FROM A EXCEPT SELECT id FROM B`).
    
- Good for **pre-filtered lists or simple ID comparisons**.
EXCEPT is not suitable for large datasets due to the following reasons, summarized for clarity:

1. **Full Result Set Computation**: EXCEPT requires executing and materializing the entire result sets of both queries (including all joins and filters) before comparing them. With large datasets, this generates massive intermediate results, consuming significant memory and CPU.
2. **Sorting/Hashing Overhead**: EXCEPT removes duplicates and performs a set difference, typically via sorting or hashing both result sets. For large datasets, this is computationally expensive, especially with multiple columns or joins.
3. **No Short-Circuiting**: Unlike NOT EXISTS or LEFT JOIN, which can stop processing rows early (e.g., when no match is found), EXCEPT processes both queries fully, leading to unnecessary work for large tables.
4. **Poor Optimization for Joins**: With multiple joins, EXCEPT doesn’t leverage indexes as effectively as NOT EXISTS or LEFT JOIN, which can use anti-joins or index seeks to filter rows incrementally.
5. **Scalability Issues**: The cost of sorting/hashing and comparing large result sets grows with data size, making EXCEPT slower than alternatives like NOT EXISTS or LEFT JOIN ... WHERE ... IS NULL, which optimize row-by-row filtering.

**Conclusion**: For large datasets, NOT EXISTS or LEFT JOIN ... WHERE ... IS NULL are better because they process rows incrementally, leverage indexes, and short-circuit, reducing I/O and CPU usage compared to EXCEPT’s full-set processing.

### General Best Practices

- ✅ Use `NOT EXISTS` over `NOT IN` to avoid `NULL` issues.
    
- ✅ Use `EXCEPT` for clean set comparisons.
    
- ✅ Index join keys (e.g., `property_id`, `id`) for better performance.
    
- ✅ Avoid correlated subqueries unless needed — they are row-wise.
    
- ❌ Avoid `NOT IN` with subqueries unless you're sure it **won't return NULLs**.
    
- ✅ PostgreSQL optimizer is **smart** — it will rewrite many `NOT EXISTS` queries as **anti joins** automatically.

### **Which is Better?**

- **Best Overall**: NOT EXISTS is generally the safest and most efficient choice, especially for large datasets or when NULL values are possible. It’s optimized by most databases into an anti-join and handles complex conditions well.
- **Use EXCEPT** when you need to compare entire rows and want distinct results automatically. It’s less flexible for complex conditions but intuitive for set operations.
- **Avoid NOT IN** unless the subquery is small, guaranteed to be NULL-free, and involves a single column. Its performance and NULL handling issues make it less reliable.