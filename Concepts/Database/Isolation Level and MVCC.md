---
tags:
  - concurrency
  - database
  - isolation
  - mvcc
  - postgresql
  - transactions
---

### Isolation Levels

**Isolation levels** in databases define the degree to which **transactions** are **isolated from one another**, balancing data consistency with performance. They are part of the ACID properties (Atomicity, Consistency, Isolation, Durability) and address how concurrent transactions interact, particularly to prevent issues like dirty reads, non-repeatable reads, and phantom reads.

#### Why We Need and Care About Isolation Levels

Isolation levels are critical because they:

- **Ensure data consistency**: **Prevent anomalies when multiple transactions access or modify the same data concurrently.**
- **Balance performance and correctness**: Higher isolation levels (stricter) ensure stronger consistency but may reduce concurrency and performance. Lower levels allow more concurrency but risk inconsistencies.
- **Address concurrency issues**: Different applications have different requirements for data accuracy versus speed, so isolation levels provide flexibility to tune the database behavior.

#### Standard Isolation Levels (Defined by SQL Standard)

1. **Read Uncommitted**:
    - Allows transactions to read uncommitted changes from other transactions (dirty reads).
    - Lowest isolation, highest concurrency, but risks inconsistent data.
    - Use case: Rare, typically for applications where performance is critical and some data inconsistency is tolerable.
2. **Read Committed**:
    - Prevents dirty reads; transactions only see committed data.
    - Non-repeatable reads and phantom reads are possible.
    - Common default in many databases (e.g., PostgreSQL, SQL Server).
    - Use case: General-purpose applications where basic consistency is enough.
3. **Repeatable Read**:
    - Prevents dirty reads and non-repeatable reads by ensuring a transaction sees the same data throughout its duration.
    - Phantom reads are still possible.
    - Use case: Applications needing stable reads within a transaction, like financial reporting.
4. **Serializable**:
    - Highest isolation level; prevents all anomalies (dirty reads, non-repeatable reads, phantom reads).
    - Transactions are executed as if they were run sequentially, ensuring full consistency.
    - Lower concurrency due to locking or conflict detection.
    - Use case: Critical systems requiring strict correctness, like banking or inventory management.
#### Concurrency Anomalies Addressed

- **Dirty Read**: Reading uncommitted data that may be rolled back.
- **Non-repeatable Read**: Data changes between reads within the same transaction.
- **Phantom Read**: New rows appear or disappear in a result set during a transaction due to other transactions inserting/deleting rows.

### MVCC (Multi-Version Concurrency Control)

**MVCC** is a database mechanism that allows multiple transactions to access the same data concurrently without blocking, by maintaining multiple versions of data. Each transaction sees a snapshot of the database at a specific point in time, ensuring consistency without requiring extensive locking.

#### How MVCC Works

- **Versioning**: When a transaction modifies a row, the database creates a new version of that row rather than overwriting the original. Older versions are retained for other transactions.
- **Snapshots**: Each transaction operates on a consistent snapshot of the database, reflecting the state when the transaction began (or a specific point defined by the isolation level).
- **Garbage Collection**: Old versions are cleaned up (garbage collected) once no transactions need them.
- **Visibility Rules**: Transactions determine which version of a row to read based on timestamps or transaction IDs, ensuring isolation.
#### Benefits of MVCC

- **Improved Concurrency**: Readers don’t block writers, and writers don’t block readers, unlike traditional locking mechanisms.
- **Consistency**: Each transaction sees a consistent view of the data, even during concurrent modifications.
- **Support for Isolation Levels**: MVCC enables databases to implement various isolation levels efficiently.

#### MVCC in Practice

Databases like **PostgreSQL**, **MySQL (InnoDB)**, and **Oracle** use MVCC. For example:

- In PostgreSQL, MVCC uses transaction IDs (XIDs) and tuple visibility to manage versions. Each row has metadata indicating which transactions can see it.
- In MySQL (InnoDB), MVCC uses undo logs to store old versions of data.

### -- Relationship Between Isolation Levels and MVCC

MVCC is a mechanism that databases use to **implement** isolation levels, particularly for higher levels like Read Committed, Repeatable Read, and Serializable. The relationship lies in how MVCC enables the database to provide the required isolation guarantees:

1. **Read Uncommitted**:
    - MVCC may not be heavily utilized, as transactions can read uncommitted data directly.
    - Minimal versioning is needed, as there’s no snapshot isolation.
2. **Read Committed**:
    - MVCC ensures transactions see only committed data by using snapshots or version checks.
    - For each read, the transaction fetches the latest committed version of a row.
3. **Repeatable Read**:
    - MVCC creates a snapshot at the start of the transaction. All reads within the transaction see this snapshot, preventing non-repeatable reads.
    - The database ensures that the transaction only sees versions of data that were committed before the transaction began.
4. **Serializable**:
    - MVCC supports serializable isolation by tracking dependencies between transactions and detecting conflicts.
    - Some databases (e.g., PostgreSQL) use **Serializable Snapshot Isolation (SSI)**, an MVCC-based approach that checks for serialization anomalies and aborts transactions if conflicts occur.
#### How They Work Together

- **MVCC as the Foundation**: MVCC provides the versioning and snapshot mechanisms that allow databases to enforce isolation levels without excessive locking.
- **Isolation Levels Define Behavior**: The isolation level determines which data versions a transaction can see and how conflicts are handled.
    - For example, in Read Committed, MVCC ensures a transaction sees the latest committed version for each read, while in Repeatable Read, it uses a fixed snapshot.
- **Conflict Resolution**: In Serializable mode, MVCC tracks transaction dependencies to ensure serializable execution, potentially aborting transactions if they would violate serializability.

#### Example in PostgreSQL

- **Read Committed**: A transaction reads the latest committed version of a row at the time of each query.
- **Repeatable Read**: The transaction uses a snapshot taken at the start, seeing only versions committed before that point.
- **Serializable**: MVCC with SSI ensures no serialization anomalies by detecting conflicts (e.g., write skew) and rolling back transactions if needed.

### Why We Care About Their Interaction

- **Performance vs. Consistency**: MVCC allows databases to achieve high concurrency while supporting various isolation levels, but each level has trade-offs. For example, Serializable may lead to more transaction aborts, impacting performance.
- **Application Requirements**: Choosing the right isolation level, backed by MVCC, ensures the application meets its consistency needs without sacrificing performance unnecessarily.
- **Scalability**: MVCC reduces contention (e.g., fewer locks), making it ideal for high-concurrency environments like web applications.

- **Isolation Levels**: Define how transactions interact, controlling the trade-off between consistency and concurrency.
- **MVCC**: A mechanism that supports isolation levels by maintaining multiple data versions, allowing concurrent reads and writes with consistent snapshots.
- **Together**: MVCC enables databases to implement isolation levels efficiently, ensuring the desired level of consistency while minimizing locking and maximizing concurrency.
- **Practical Considerations**: Choose isolation levels based on application needs (e.g., strict consistency for banking, relaxed for analytics). MVCC makes this flexibility possible without sacrificing performance in most cases.

### **Why MVCC Minimizes Locking:**

- **Reads don’t block writes**: Readers don’t need to lock rows — they just read a consistent snapshot.
    
- **Writes don’t block reads**: Writers create a new version of the data, so readers can continue reading the old version.
    
- **Less need for shared/exclusive locks**, which are common in traditional locking-based concurrency control.

### **Why MVCC Maximizes Concurrency:**

- **Multiple readers and writers** can operate at the same time without waiting.
    
- Readers are **never blocked** by writers.
    
- Writers only conflict with other writers, and even then, only when they update the same row.

MVCC minimizes locking by avoiding row-level read locks, and maximizes concurrency by letting transactions work on different versions of data.
