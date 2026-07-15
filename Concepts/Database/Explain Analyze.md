---
tags:
  - database
  - explain
  - postgresql
  - query-optimization
---

Reading and understanding the output of EXPLAIN ANALYZE in PostgreSQL is a powerful way to analyze how a query executes, identify performance bottlenecks, and optimize it. EXPLAIN shows the query plan (how PostgreSQL intends to execute the query), and adding ANALYZE runs the query and provides actual execution times and row counts. Here’s a step-by-step guide:

### 1. **Running EXPLAIN ANALYZE**

- **Syntax**:
    
    
    
    `EXPLAIN ANALYZE SELECT * FROM my_table WHERE column1 = 'value' ORDER BY column2;`
    
- **Key Notes**:
    - EXPLAIN alone shows the plan without running the query.
    - EXPLAIN ANALYZE executes the query, so be cautious with destructive operations (e.g., UPDATE, DELETE)—use a transaction and rollback if needed.
    - Output is a tree-like structure, with the top node summarizing the total cost and time.

### 2. **Key Components of the Output**

The output is a hierarchical plan, with nodes representing operations (e.g., scans, joins). Here’s what to look for:

- **Node Types**:
    - **Seq Scan**: Sequential scan of a table (reads all rows).
    - **Index Scan**: Uses an index to fetch specific rows.
    - **Index Only Scan**: Fetches data directly from the index, avoiding table access.
    - **Nested Loop**: A join method, often for small datasets, pairing rows from two tables.
    - **Hash Join**: Builds a hash table for one table to join with another, good for large datasets.
    - **Merge Join**: Merges sorted tables, efficient if data is pre-sorted.
    - **Sort**: Sorts rows (e.g., for ORDER BY).
    - **Aggregate**: Computes aggregates (e.g., SUM, GROUP BY).
- **Cost Estimates** (from EXPLAIN):
    - **Format**: cost=start_cost..total_cost
    - **Start Cost**: Estimated cost to begin producing rows (e.g., disk reads).
    - **Total Cost**: Estimated cost to complete the operation.
    - Units are arbitrary but relative—higher means more expensive.
    - Influenced by settings like seq_page_cost and random_page_cost.
- **Actual Metrics** (from ANALYZE):
    - **Actual Time**: actual time=first_row..total_time (in milliseconds).
        - first_row: Time to return the first row.
        - total_time: Time to complete the node.
    - **Rows**: Actual number of rows processed or returned.
    - **Loops**: Number of times the node was executed (e.g., in a nested loop, inner operations may loop multiple times).
- **Planning Time**: Time (in ms) to generate the query plan.
- **Execution Time**: Total time (in ms) to run the query.
- **Additional Clues**:
    - **Rows Removed by Filter**: If a scan filters rows (e.g., WHERE), this shows how many were discarded.
    - **Heap Fetches**: For index-only scans, indicates table lookups if index data was insufficient.
    - **Sort Method**: Shows if sorting was in memory or spilled to disk (e.g., “Sort Method: external merge” means disk was used, often slow).
    - **Hash Spills**: For hash joins, indicates if data spilled to disk due to low work_mem.

### 3. **How to Read the Output**

- **Tree Structure**: Read from bottom to top. Child nodes (indented) feed into parent nodes. The top node is the final output.
- **Cost vs. Actual**: Compare estimated costs to actual times/rows. Big mismatches suggest poor planner estimates (e.g., outdated statistics).
- **Bottlenecks**: Look for:
    - High actual time values, especially in total_time.
    - Discrepancies between planned rows and actual rows.
    - Disk spills (e.g., “Sort Method: external merge Disk: 1024kB”)—a sign work_mem may be too low.
    - High loop counts in nested loops, which can be slow for large datasets.

### 4. **Example**

Query:

sql

CollapseWrap

Copy

`EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123 ORDER BY order_date;`

Sample Output:

text

CollapseWrap

Copy

`Sort (cost=150.50..152.75 rows=900 width=120) (actual time=10.123..10.567 rows=850 loops=1) Sort Key: order_date Sort Method: quicksort Memory: 1024kB -> Index Scan using idx_orders_customer_id on orders (cost=0.42..130.25 rows=900 width=120) (actual time=0.050..9.800 rows=850 loops=1) Index Cond: (customer_id = 123) Planning Time: 0.250 ms Execution Time: 11.020 ms`

- **Interpretation**:
    - **Index Scan**: Uses idx_orders_customer_id to find rows where customer_id = 123.
        - Estimated 900 rows, actually got 850—pretty close.
        - Took 0.050 ms to start, 9.800 ms to finish.
    - **Sort**: Sorts results by order_date using quicksort in memory (good—no disk spill).
        - Took 10.123 ms to start, 10.567 ms total.
    - **Total**: Planning took 0.250 ms, execution 11.020 ms—pretty fast!
    - **No Red Flags**: No disk spills, row estimates are accurate, times are low.

### 5. **Steps to Understand and Optimize**

1. **Run the Command**:
    - Use EXPLAIN ANALYZE in a tool like psql, pgAdmin, or your app.
    - Add VERBOSE for more detail (e.g., EXPLAIN (ANALYZE, VERBOSE)).
2. **Check the Plan**:
    - Look at node types: Prefer index scans over seq scans for selective queries.
    - Check join types: Hash or merge joins are often better for large tables than nested loops.
3. **Analyze Performance**:
    - **Time**: High actual time at any node suggests a bottleneck.
    - **Rows**: If actual rows differ greatly from estimated, run ANALYZE on the table to update statistics.
    - **Spills**: Disk-based sorts or hashes? Increase work_mem (e.g., SET work_mem = '32MB';) and retest.
4. **Optimize**:
    - **Indexes**: Missing index for a WHERE or ORDER BY? Create one (e.g., CREATE INDEX idx_orders_customer_id ON orders(customer_id);).
    - **Statistics**: Run ANALYZE to improve planner estimates.
    - **Memory**: If disk spills occur, tune work_mem cautiously.
    - **Query Rewrite**: Simplify joins, avoid unnecessary columns in SELECT, or filter earlier.
5. **Recheck**: Rerun EXPLAIN ANALYZE after changes to confirm improvements.

### 6. **Tips and Tools**

- **Readability**: Use EXPLAIN (ANALYZE, FORMAT JSON) for a structured output, or tools like **pgAdmin** or **explain.depesz.com** for visual plans.
- **Stats**: Check pg_stat_all_tables for scan counts to see if indexes are used.
- **Caution**: ANALYZE runs the query, so test in a safe environment for heavy operations.

### 7. **Your Next Step**

If you have a specific query you’d like to analyze, share it with me (e.g., the SQL, table size, indexes), and I’ll walk you through the EXPLAIN ANALYZE output and suggest tweaks!