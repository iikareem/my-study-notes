---
tags:
  - concurrency
  - database
  - locking
  - postgresql
---

Locks in databases are mechanisms used to control concurrent access to data, ensuring consistency and preventing conflicts when multiple transactions access or modify the same data. Locks are a fundamental part of how databases enforce isolation levels and manage concurrency, especially in systems that don’t rely solely on MVCC (Multi-Version Concurrency Control).

#### Types of Locks

1. **Shared Locks (Read Locks)**:
    - Allow multiple transactions to read a resource simultaneously but prevent writing.
    - Used when a transaction needs to read data without modifying it.
    - Example: Multiple SELECT queries can acquire shared locks on the same table.
2. **Exclusive Locks (Write Locks)**:
    - Allow only one transaction to modify a resource, blocking both reads and writes from other transactions.
    - Used for operations like UPDATE, INSERT, or DELETE.
    - Example: An UPDATE query acquires an exclusive lock on the affected rows.
3. **Row-Level Locks**:
    - Lock specific rows in a table, allowing other rows to remain accessible.
    - Common in databases like PostgreSQL and MySQL (InnoDB).
4. **Table-Level Locks**:
    - Lock an entire table, restricting access to all rows.
    - Used for operations like schema changes (e.g., ALTER TABLE).
5. **Range Locks**:
    - Lock a range of values, often used in SERIALIZABLE isolation to prevent phantom reads.
    - Example: Locking a range of keys in an index to prevent new rows from being inserted.
6. **Deadlocks**:
    - Occur when two or more transactions hold locks and wait for each other’s locks, causing a stalemate.
    - Databases detect deadlocks and abort one or more transactions to resolve them.
#### Why Locks Are Needed

- **Prevent Data Corruption**: Ensure that concurrent transactions don’t overwrite or interfere with each other’s changes.
- **Enforce Isolation Levels**: Locks help implement isolation levels like Repeatable Read or Serializable by controlling access to data.
- **Maintain Consistency**: Prevent anomalies like dirty reads, non-repeatable reads, or phantom reads.

### Optimistic vs. Pessimistic Locking

Databases use two main concurrency control strategies to manage locks: **optimistic locking** and **pessimistic locking**. These approaches differ in how they handle conflicts and concurrency.

#### 1. Pessimistic Locking

- **Definition**: Assumes conflicts between transactions are likely and prevents them by acquiring locks early in a transaction.
- **How It Works**:
    - A transaction acquires locks (shared or exclusive) on data before accessing or modifying it.
    - Other transactions are blocked from accessing locked data until the lock is released (typically at transaction commit or rollback).
    - Often used with explicit locking mechanisms like SELECT ... FOR UPDATE in SQL.
- **Characteristics**:
    - **Blocking**: Transactions may wait for locks, reducing concurrency but ensuring safety.
    - **Best for**: High-conflict scenarios, short transactions, or critical operations where conflicts must be avoided (e.g., financial transactions).
    - **Drawbacks**: Can lead to contention, reduced throughput, and deadlocks in high-concurrency systems.
- **Example**:
    - In PostgreSQL: SELECT * FROM accounts WHERE id = 1 FOR UPDATE; locks the row, preventing other transactions from modifying it until the transaction commits.
    - Use case: Updating a bank account balance to ensure no other transaction modifies it concurrently.

#### 2. Optimistic Locking

- **Definition**: Assumes conflicts are rare and allows transactions to proceed without locking, checking for conflicts only at commit time.
- **How It Works**:
    - Transactions read and modify data without acquiring locks.
    - At commit, the database or application checks if the data has changed since it was read (e.g., using a version number or timestamp).
    - If a conflict is detected (e.g., another transaction modified the data), the transaction is rolled back and may be retried.
- **Characteristics**:
    - **Non-blocking**: Higher concurrency since transactions don’t wait for locks.
    - **Best for**: Low-conflict scenarios, read-heavy workloads, or long-running transactions where locking would cause bottlenecks.
    - **Drawbacks**: Requires conflict detection and retry logic, which can complicate application code and lead to aborts in high-conflict scenarios.
- **Example**:
    - A table with a version column tracks changes. A transaction reads a row (e.g., version = 1), modifies it, and attempts to update with WHERE version = 1. If another transaction has incremented the version, the update fails, and the transaction retries.
    - In SQL: UPDATE accounts SET balance = balance - 100, version = version + 1 WHERE id = 1 AND version = 1;
- **Implementation**:
    - Often implemented at the application level or using MVCC snapshots in databases like PostgreSQL or MySQL.

![[Pasted image 20250606015045.png]]

### Other Concurrency Control Approaches

While optimistic and pessimistic locking are the most common strategies, other approaches exist, often used in specific databases or contexts:

1. **Multi-Version Concurrency Control (MVCC)**:
    - As discussed previously, MVCC maintains multiple versions of data to allow concurrent reads and writes without locking.
    - Often combined with optimistic locking (e.g., in PostgreSQL’s Serializable Snapshot Isolation).
    - Unlike traditional locking, MVCC avoids reader-writer conflicts by using snapshots, but it may still use locks for writes in certain isolation levels.
    - Example: PostgreSQL uses MVCC for Read Committed and Repeatable Read, with optional locking for specific operations.
2. **Snapshot Isolation**:
    - A specific MVCC-based approach where transactions operate on a consistent snapshot of the database.
    - Prevents dirty reads and non-repeatable reads but may allow write skew (a type of anomaly).
    - Often used as a foundation for optimistic concurrency control, as transactions check for conflicts at commit.
    - Example: Oracle and PostgreSQL use snapshot isolation for Repeatable Read.
3. **Two-Phase Locking (2PL)**:
    - A strict form of pessimistic locking where transactions have two phases:
        - **Growing Phase**: Acquire locks without releasing any.
        - **Shrinking Phase**: Release locks without acquiring new ones.
    - Ensures serializability but can lead to deadlocks.
    - Used in databases like SQL Server for certain isolation levels.
4. **Timestamp Ordering**:
    - Transactions are assigned timestamps, and the database ensures operations are processed in timestamp order to avoid conflicts.
    - Conflicts are resolved by aborting transactions that violate the order.
    - Used in some distributed databases (e.g., Google Spanner).
5. **Conflict-Free Replicated Data Types (CRDTs)**:
    - Used in distributed databases (e.g., DynamoDB, Cassandra) to handle concurrency without locks.
    - Data structures are designed to merge concurrent updates without conflicts.
    - Not a locking mechanism but an alternative for highly distributed systems.

### Relationship with Isolation Levels and MVCC

- **Locks and Isolation Levels**:
    - Pessimistic locking is often used to enforce stricter isolation levels (e.g., Serializable) by explicitly locking data.
    - Optimistic locking aligns with MVCC-based isolation levels like Read Committed or Snapshot Isolation, as it relies on version checks rather than locks.
    - For example, in PostgreSQL, Read Committed uses MVCC with minimal locking, while Serializable may use pessimistic locks or conflict detection to prevent anomalies.
- **MVCC and Locking**:
    - MVCC reduces the need for locks by allowing readers to access old versions of data, making it a natural fit for optimistic locking.
    - Pessimistic locking may still be used with MVCC for specific operations (e.g., FOR UPDATE in PostgreSQL) to ensure exclusive access.
    - In high-conflict scenarios, MVCC may fall back to pessimistic locking to enforce stricter isolation (e.g., in Serializable mode).
**Database Support**:

- PostgreSQL: Primarily MVCC with optimistic locking for most operations, supports pessimistic locking via **FOR UPDATE**.
- MySQL (InnoDB): Uses MVCC and supports both optimistic (via version columns) and pessimistic locking (via SELECT ... FOR UPDATE).
- SQL Server: Uses locking by default but supports MVCC-like behavior with Snapshot Isolation.

P**erformance**:
    - Pessimistic locking can cause contention and deadlocks in high-concurrency systems.
    - Optimistic locking requires careful conflict handling and may lead to transaction retries, impacting performance in high-conflict scenarios.
- **Deadlock Prevention**:
    - Use consistent lock acquisition order to minimize deadlocks in pessimistic locking.
    - Monitor and tune retry logic for optimistic locking to handle conflicts efficiently.

### Summary

- **Locks**: Control concurrent access to data, with types like shared, exclusive, row-level, and table-level.
- **Pessimistic Locking**: Locks data early to prevent conflicts, suitable for high-conflict scenarios but reduces concurrency.
- **Optimistic Locking**: Assumes conflicts are rare, checks for conflicts at commit, and is ideal for low-conflict, read-heavy workloads.
- **Other Approaches**: MVCC, Snapshot Isolation, Two-Phase Locking, Timestamp Ordering, and CRDTs provide alternative concurrency control mechanisms.
- **Relation to Isolation Levels and MVCC**: Locks (especially pessimistic) enforce stricter isolation levels, while MVCC supports optimistic locking and higher concurrency for levels like Read Committed or Snapshot Isolation.
