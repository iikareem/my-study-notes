---
tags:
  - database
  - postgresql
  - query-optimization
---

The `join_collapse_limit` is a PostgreSQL configuration parameter that controls how the query planner determines the order of joins in SQL queries.

### 🔍 What does it do?

In SQL queries with **multiple joins**, PostgreSQL's planner tries to find the **most efficient join order**. However, evaluating **all possible join orders** is **computationally expensive**, especially as the number of joined tables increases.

`join_collapse_limit` limits how many joins PostgreSQL will try to **reorder freely**.

---

### 🧠 How it works:

- If a query joins `N` tables, and `N <= join_collapse_limit`, then **PostgreSQL considers all possible join orders** (which grows factorially).
    
- If `N > join_collapse_limit`, PostgreSQL **uses a heuristic** (e.g., joining in the written order or applying some greedy logic) instead of trying all combinations.
    

---

### ⚙️ Default value:

- The default is `8`.
    

---

### 🧪 Example:

sql

CopyEdit

`SELECT *  FROM A  JOIN B ON ... JOIN C ON ... JOIN D ON ... JOIN E ON ... JOIN F ON ... JOIN G ON ... JOIN H ON ... JOIN I ON ...;`

This is 9 tables.

- If `join_collapse_limit = 8`, the planner **will not** explore all join orders.
    
- To force PostgreSQL to consider all possible join orders for this query, you'd set:
    

sql

CopyEdit

`SET join_collapse_limit = 9;`

---

### ⚠️ Considerations:

- **Higher values** = better plan quality but **slower planning time**.
    
- **Lower values** = faster planning but **possibly suboptimal plans**.
    
- Not recommended to set very high values in production unless needed.
    

---

### 🔧 How to check or change:

`-- Check the current setting SHOW join_collapse_limit;  -- Change it for the current session SET join_collapse_limit = 12;`

---

Let me know if you'd like to see how it affects execution plans (`EXPLAIN`).