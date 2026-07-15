---
tags:
  - companion
  - distributed-lock
  - fencing-token
  - idempotency
  - interview
  - problem-1
  - problems
  - system-design
---

# Companion — Upsert, Idempotency & Safe Retries

**Domain:** Concurrency & API Design  
**Topics:** upsert · idempotency · safe retries · concurrency

**Pairs with:** [[Our payment API is charging users twice]] · [[Problems MOC]]

In the context of databases, an **upsert** is a portmanteau of **"update"** and **"insert."** It is an operation that automatically determines whether to update an existing record or insert a new one based on whether a specific criterion (usually a primary key or unique index) is met.

Essentially, it follows this logic: **"If the record exists, update it; otherwise, insert it."**

### How It Works

When you perform an upsert, the database engine executes the following check:

1. **Check for existence:** The database looks for an existing record matching the unique identifier (e.g., ID, username, or email) provided in your query.
    
2. **Conditional Execution:**
    
    - **If the record is found:** The existing record is updated with the new data.
        
    - **If the record is not found:** A new record is created using the provided data.
        

### Why Use Upsert?

- **Efficiency:** It reduces the number of round-trips to the database. Instead of your application code running a `SELECT`query to see if a record exists, followed by either an `UPDATE` or an `INSERT`, the database handles the logic in a single atomic operation.
    
- **Concurrency:** It helps prevent race conditions. If two processes try to create the same record simultaneously, an upsert ensures the database handles the conflict gracefully rather than failing or creating duplicates.
    
- **Cleaner Code:** It simplifies your application logic by removing the need for "if-else" blocks to manage data synchronization.
    

### Examples in Common Databases

While the concept remains the same, the syntax varies depending on the database system:

|**Database System**|**Command / Syntax**|
|---|---|
|**PostgreSQL**|`INSERT INTO table_name (...) VALUES (...) ON CONFLICT (key) DO UPDATE SET ...`|
|**MySQL**|`INSERT INTO table_name (...) VALUES (...) ON DUPLICATE KEY UPDATE ...`|
|**MongoDB**|`db.collection.updateOne({ criteria }, { $set: data }, { upsert: true })`|
|**SQLite**|`INSERT INTO table_name (...) VALUES (...) ON CONFLICT(key) DO UPDATE SET ...`|

> **Note:** To perform an upsert, you **must** have a unique constraint (like a Primary Key or a Unique Index) on the column you are using to check for existence. Without this, the database cannot identify whether the record is a duplicate or a new entry.

## Next

- Scenario: [[Our payment API is charging users twice]]
- Back to hub: [[Problems MOC]]
