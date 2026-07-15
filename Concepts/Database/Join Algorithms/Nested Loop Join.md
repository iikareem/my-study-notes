---
tags:
  - backend
  - database
  - design-patterns
  - joins
  - nestjs
  - oop
  - postgresql
  - query-optimization
---

![[Pasted image 20250623162350.png]]
https://bertwagner.com/posts/visualizing-nested-loops-joins-and-understanding-their-implications

A **nested loop join** is a database query execution algorithm used to join two tables, where for each row in the outer table (the first table), the database scans the inner table (the second table) to find matching rows based on a join condition. It’s one of the simplest join algorithms but can be computationally expensive for large datasets.

### How Nested Loop Join Works
1. **Outer Loop**: Iterate through each row of the outer table.
2. **Inner Loop**: For each row in the outer table, iterate through every row in the inner table to check if the join condition (e.g., `table1.column = table2.column`) is satisfied.
3. **Result**: If a match is found, the rows from both tables are combined and included in the result set.

**Pseudocode**:
```sql
FOR each row in outer_table
    FOR each row in inner_table
        IF join_condition is true
            Output combined row
```

### Characteristics
- **Performance**: Works best when at least one table is small or when indexes are available on the join columns of the inner table.
- **Cost**: High for large tables, as the complexity is O(n * m), where `n` and `m` are the number of rows in the outer and inner tables, respectively.
- **Flexibility**: Can handle any join condition, not just equality-based conditions (unlike hash joins).

### Types of Nested Loop Joins
1. **Simple Nested Loop Join**: Scans the entire inner table for each row of the outer table. Inefficient for large tables.
2. **Index Nested Loop Join**: Uses an index on the inner table’s join column to quickly locate matching rows, significantly improving performance.
3. **Block Nested Loop Join**: Processes blocks of rows from the outer table at once, reducing I/O operations by caching inner table data in memory.

### Use Cases
Nested loop joins are useful in the following scenarios:
1. **Small Tables**: When one or both tables are small, the overhead of scanning the inner table repeatedly is manageable.
2. **Indexed Columns**: When the inner table has an index on the join column, allowing for fast lookups (index nested loop join).
3. **Non-Equi Joins**: When the join condition involves non-equality operators (e.g., `<`, `>`, `!=`), as hash joins typically require equality conditions.
4. **Low Data Volume Queries**: For queries returning a small number of rows, such as when filtering with a `WHERE` clause reduces the outer table’s size.
5. **Real-Time Processing**: When immediate results are needed for small datasets, and the overhead of setting up other join methods (like hash or merge joins) is unnecessary.

### When to Use Nested Loop Join
A nested loop join is typically chosen by the database query optimizer when:
- **One Table is Small**: The outer table has few rows, or the inner table is small enough to fit in memory or has an efficient index.
- **Indexes Are Available**: An index on the inner table’s join column allows for quick lookups, making an index nested loop join viable.
- **Complex Join Conditions**: The join involves non-equality conditions or complex predicates that other algorithms (like hash joins) can’t handle efficiently.
- **Memory Constraints**: When memory is limited, and other join methods (like hash joins requiring hash tables) are too resource-intensive.
- **Highly Selective Queries**: When a `WHERE` clause or join condition significantly reduces the number of rows processed from the outer table.

### When to Avoid Nested Loop Join
- **Large Tables Without Indexes**: If both tables are large and no indexes are available, the join becomes extremely slow due to the O(n * m) complexity.
- **High Data Volume**: For large-scale joins, hash joins or merge joins are often more efficient.
- **Frequent Joins**: In high-performance systems with frequent large joins, other algorithms may be preferred to reduce processing time.

### Example
Suppose you have two tables:
- `orders` (small, 100 rows): Contains order details.
- `customers` (large, 1M rows): Contains customer details with an index on `customer_id`.

**Query**:
```sql
SELECT orders.order_id, customers.customer_name
FROM orders
JOIN customers ON orders.customer_id = customers.customer_id;
```

- The database may choose a nested loop join if:
  - `orders` is the outer table (small size).
  - `customers` has an index on `customer_id`, enabling an index nested loop join.
- For each row in `orders`, the database uses the index to quickly find matching rows in `customers`, reducing the need to scan the entire `customers` table.

### Comparison with Other Joins
- **Vs. Hash Join**: Hash joins are faster for large tables with equality conditions but require more memory and don’t handle non-equality conditions well.
- **Vs. Merge Join**: Merge joins are efficient for sorted data or large tables but require sorted input and are less flexible for complex conditions.

### Conclusion
Nested loop joins are best for scenarios involving small tables, indexed inner tables, or complex join conditions. They are less efficient for large, unindexed datasets, where other join algorithms like hash or merge joins may perform better. Database optimizers typically choose nested loop joins automatically based on table sizes, indexes, and query conditions.