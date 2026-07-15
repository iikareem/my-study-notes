---
tags:
  - database
  - joins
  - postgresql
  - query-optimization
---

A **hash join** is a database query execution algorithm used to join two tables by creating a hash table for one table (usually the smaller one) and using it to efficiently find matching rows from the other table based on a join condition. It is particularly effective for equality-based joins (e.g., `table1.column = table2.column`) and large datasets.

### How Hash Join Works
1. **Build Phase**:
   - The smaller table (or the one with fewer rows after filtering) is selected as the **build table**.
   - A hash table is created in memory, where the join key (column used in the join condition) is hashed, and rows are stored in hash buckets based on the hash value.
2. **Probe Phase**:
   - The larger table (the **probe table**) is scanned row by row.
   - For each row, the join key is hashed using the same hash function, and the algorithm looks for matches in the corresponding hash bucket of the build table.
3. **Result**:
   - Matching rows from both tables are combined and included in the result set.

**Pseudocode**:
```sql
// Build Phase
CREATE hash_table FOR build_table USING join_key

// Probe Phase
FOR each row in probe_table
    hash_value = HASH(probe_table.join_key)
    FOR each row in hash_table[hash_value]
        IF join_condition is true
            Output combined row
```

### Characteristics
- **Performance**: Efficient for large tables with equality-based join conditions, as the hash table allows fast lookups. Complexity is approximately O(n + m), where `n` and `m` are the number of rows in the build and probe tables.
- **Memory Usage**: Requires significant memory to store the hash table, especially for large build tables.
- **Limitations**: Best suited for equality joins (`=`). Non-equality conditions (e.g., `<`, `>`) are not efficiently handled.
- **Spill to Disk**: If the hash table doesn’t fit in memory, parts of it may spill to disk, reducing performance.

### Types of Hash Joins
1. **Simple Hash Join**: Builds a hash table for the smaller table and probes with the larger table. Assumes the hash table fits in memory.
2. **Grace Hash Join**: Used when the build table is too large for memory. The data is partitioned into smaller chunks, hashed, and processed in multiple passes, with temporary disk storage.
3. **Hybrid Hash Join**: A mix of in-memory and disk-based processing, where some partitions are kept in memory, and others are written to disk.

### Use Cases
Hash joins are ideal in the following scenarios:
1. **Large Tables with Equality Joins**: When both tables are large, and the join condition is based on equality (e.g., `orders.customer_id = customers.customer_id`).
2. **No Indexes Available**: Unlike nested loop joins, hash joins don’t rely on indexes, making them suitable when indexes are absent or impractical to use.
3. **Data Warehousing**: Common in analytical queries (e.g., in data warehouses) where large datasets are joined, and equality conditions are prevalent.
4. **Parallel Processing**: Hash joins can be parallelized easily, making them suitable for distributed databases or systems with multiple processors.
5. **Ad-Hoc Queries**: When query patterns are unpredictable, and pre-built indexes may not exist, hash joins provide consistent performance.

### When to Use Hash Join
A hash join is typically chosen by the database query optimizer when:
- **Equality-Based Joins**: The join condition uses `=` (e.g., `table1.id = table2.id`).
- **Large Tables**: Both tables are large, and scanning one table to build a hash table is more efficient than nested loops.
- **Sufficient Memory**: Enough memory is available to hold the hash table for the build table, or disk-based spilling is acceptable.
- **No Suitable Indexes**: When indexes on join columns are absent, making index nested loop joins inefficient.
- **High Selectivity**: When the join condition doesn’t drastically reduce the result set, and full table scans are unavoidable.

### When to Avoid Hash Join
- **Small Tables**: For very small tables, the overhead of building a hash table may outweigh the benefits, and a nested loop join could be faster.
- **Non-Equality Joins**: Hash joins are inefficient for conditions like `<`, `>`, or `!=`, where nested loop or merge joins are better.
- **Memory Constraints**: If memory is limited and the build table is large, disk spilling can degrade performance significantly.
- **Highly Selective Queries**: If a `WHERE` clause reduces the outer table to a few rows, a nested loop join with an index may be more efficient.

### Example
Suppose you have two tables:
- `orders` (1M rows): Contains order details.
- `customers` (100K rows): Contains customer details.

**Query**:
```sql
SELECT orders.order_id, customers.customer_name
FROM orders
JOIN customers ON orders.customer_id = customers.customer_id;
```

- The database may choose a hash join if:
  - The join condition is equality-based (`orders.customer_id = customers.customer_id`).
  - `customers` is smaller and chosen as the build table to create a hash table on `customer_id`.
  - `orders` is the probe table, and each row’s `customer_id` is hashed to find matches in the hash table.
- This is efficient because lookups in the hash table are fast, avoiding the need to scan `customers` repeatedly.

### Comparison with Nested Loop Join
- **Vs. Nested Loop Join**:
  - **Performance**: Hash join is faster for large tables with equality joins (O(n + m) vs. O(n * m) for nested loops).
  - **Memory**: Hash join requires more memory for the hash table, while nested loop joins are memory-light but slower for large datasets.
  - **Join Conditions**: Hash join is limited to equality joins, while nested loop joins handle any condition.
  - **Indexes**: Nested loop joins benefit from indexes on the inner table, while hash joins don’t require indexes.
- **When to Choose**:
  - Use **hash join** for large tables, equality joins, and when memory is available.
  - Use **nested loop join** for small tables, indexed inner tables, or non-equality joins.

### Comparison with Merge Join
- **Vs. Merge Join**:
  - **Performance**: Merge join is efficient for sorted data or when both tables are large and pre-sorted, while hash join doesn’t require sorted data.
  - **Memory**: Hash join needs memory for the hash table, while merge join needs memory for sorting (if not pre-sorted).
  - **Join Conditions**: Hash join is limited to equality joins, while merge join can handle some non-equality conditions.
- **When to Choose**:
  - Use **hash join** when data is unsorted, or equality joins are needed.
  - Use **merge join** when tables are sorted or indexes provide sorted access.

### Conclusion
Hash joins are highly efficient for large-scale, equality-based joins, especially in scenarios without indexes or in data warehousing environments. They outperform nested loop joins for large datasets but require sufficient memory and are limited to equality conditions. Database optimizers typically select hash joins when the join involves large tables, equality predicates, and memory is available for the hash table.