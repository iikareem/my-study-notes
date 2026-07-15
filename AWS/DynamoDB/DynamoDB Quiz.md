---
tags:
  - aws
  - certification
  - dynamodb
  - practice
---

**Hub:** [[AWS MOC]] · **Role:** Quiz
**Also:** [[DynamoDB]] · [[DynamoDB Hints]]

# DynamoDB — DVA-C02 Real Exam Questions

> Questions sourced from real DVA-C02 exam reports and ExamTopics community discussions. Attempt each question before reading the answer.

---

## Question 1 — Hot Partition / ProvisionedThroughputExceededException

A developer is designing a game that stores data in an Amazon DynamoDB table. The partition key of the table is the **country** of the player. After a sudden increase in the number of players in a specific country, the developer notices **ProvisionedThroughputExceededException** errors.

What should the developer do to resolve these errors?

- A. Use strongly consistent table reads.
- B. Revise the primary key to use more unique identifiers.
- C. Use pagination to reduce the size of the items that the queries return.
- D. Use the Scan operation to retrieve the data.

---

### ✅ Correct Answer: B

### Explanation

`ProvisionedThroughputExceededException` caused by a spike from a single country is a classic **hot partition** problem. When many players share the same partition key value (`country`), all their requests land on the same physical partition, overwhelming it.

**Why B is correct:** Revising the partition key to something with higher cardinality (e.g., `playerId`, or `country#<random_suffix>`) distributes traffic across more partitions and eliminates the hot spot.

**Why the others are wrong:**

- A — Strongly consistent reads _consume more RCU_, making throttling worse, not better.
- C — Pagination only reduces response size; it does nothing to distribute partition load.
- D — Scan reads the _entire table_ — far more expensive and does not fix the partition problem.

**Key concept:** A good partition key has high cardinality and evenly distributed access. Using `country` violates this — low cardinality and spike-prone.

---

## Question 2 — LSI vs GSI (Querying a Different Sort Key)

A developer is writing an application that stores data in an Amazon DynamoDB table. The developer wants to query the table using the **partition key and a different sort key value**. The developer needs the **latest data with all recent write operations**.

How should the developer write the DynamoDB query?

- A. Add a local secondary index (LSI) during table creation. Query the LSI using eventually consistent reads.
- B. Add a local secondary index (LSI) during table creation. Query the LSI using strongly consistent reads.
- C. Add a global secondary index (GSI) during table creation. Query the GSI using eventually consistent reads.
- D. Add a global secondary index (GSI) during table creation. Query the GSI using strongly consistent reads.

---

### ✅ Correct Answer: B

### Explanation

Two conditions must be satisfied:

1. Query using the **same partition key but a different sort key** → LSI (not GSI).
2. **Latest data / all recent writes** → strongly consistent reads.

**Why B is correct:** An LSI uses the same partition key as the base table but allows a different sort key — exactly what the developer needs. LSI supports **strongly consistent reads**, which guarantees the most up-to-date data.

**Why the others are wrong:**

- A — LSI is correct for the key structure, but eventually consistent reads may return stale data.
- C & D — A GSI has a _different_ partition key, not the same one. Also, GSI supports **eventual consistency only** — option D is impossible. GSIs do not support strongly consistent reads.

**Key rule to memorize:**

> LSI = same partition key + different sort key → supports strong consistency. GSI = different partition key → eventual consistency only.

---

## Question 3 — GSI for Querying Non-Key Attributes

A developer is working on an existing application that uses Amazon DynamoDB. The table has: `partNumber` (partition key), `vendor` (sort key), `description`, `productFamily`, and `productType`.

Application modules frequently look for a list of products based on **productFamily** and **productType**. The developer wants to improve query performance.

Which solution will meet these requirements?

- A. Create a global secondary index (GSI) with `productFamily` as the partition key and `productType` as the sort key.
- B. Create a local secondary index (LSI) with `productFamily` as the partition key and `productType` as the sort key.
- C. Recreate the table. Add `partNumber` as the partition key and `vendor` as the sort key. During table creation, add a local secondary index (LSI) with `productFamily` as the partition key and `productType` as the sort key.
- D. Update the queries to use Scan operations with `productFamily` as the partition key and `productType` as the sort key.

---

### ✅ Correct Answer: A

### Explanation

The application needs to query by `productFamily` and `productType`, which are **not** the table's primary key. A GSI allows you to define a completely different partition key and sort key for querying.

**Why A is correct:** A GSI with `productFamily` as the partition key and `productType` as the sort key allows efficient queries on these attributes. Critically, GSIs can be **added to an existing table at any time** — no need to recreate the table.

**Why B is wrong:** An LSI must use the **same partition key** as the base table (`partNumber`). You cannot set `productFamily` as the partition key of an LSI. This makes option B structurally invalid. Also, LSIs can only be created at **table creation time**.

**Why C is wrong:** Same LSI partition key problem. LSI cannot have `productFamily` as the partition key when the table's partition key is `partNumber`. Recreating the table is also unnecessary disruption.

**Why D is wrong:** Scan reads the **entire table** then applies a filter. This is extremely inefficient and wastes RCU. It gets worse as data volume grows.

---

## Question 4 — TTL for Automatic Data Deletion

A company built an online event platform. The company stores leaderboard data in Amazon DynamoDB and retains the data for **30 days** after an event is complete. The company uses a scheduled job to delete the old leaderboard data.

During busy months, the DynamoDB write API requests are **throttled** when the scheduled delete job runs. A developer must create a long-term solution that deletes old data and **optimizes write throughput**.

Which solution meets these requirements?

- A. Configure a TTL attribute for the leaderboard data.
- B. Use DynamoDB Streams to schedule and delete the leaderboard data.
- C. Use AWS Step Functions to schedule and delete the leaderboard data.
- D. Increase the provisioned write capacity before the scheduled job runs, then decrease it afterward.

---

### ✅ Correct Answer: A

### Explanation

The problem is that a batch delete job consumes WCU and causes throttling. The solution is to eliminate the delete job entirely by using TTL.

**Why A is correct:** TTL automatically deletes items when their expiration timestamp is reached — **at no WCU cost**. DynamoDB handles the deletion in the background without consuming write capacity. This solves both problems: the data is removed after 30 days AND write throughput is no longer impacted.

**Why B is wrong:** DynamoDB Streams captures changes — it does not schedule or trigger deletions on its own.

**Why C is wrong:** Step Functions can orchestrate a delete workflow, but the underlying DeleteItem calls still consume WCU — the throttling problem is not solved.

**Why D is wrong:** Scaling up/down WCU is a workaround, not a long-term optimized solution, and incurs extra cost.

**Key rule:** TTL deletions are free and do not consume WCU. Always prefer TTL over scheduled delete jobs for time-based expiration.

---

## Question 5 — TTL + Streams for Archive-and-Delete

A developer supports an application that accesses data in an Amazon DynamoDB table. One attribute is `expirationDate` in timestamp format. The application finds items, archives them, and removes them based on this timestamp.

The application will be decommissioned. The developer needs to implement the same functionality with the **least amount of code**.

Which solution meets these requirements?

- A. Enable TTL on the `expirationDate` attribute. Create a DynamoDB Stream. Create an AWS Lambda function to process the deleted items. Create a DynamoDB trigger for the Lambda function.
- B. Create two AWS Lambda functions: one to delete items, one to process them. Create a DynamoDB Stream. Use the DeleteItem API to delete items based on `expirationDate`.
- C. Create two AWS Lambda functions: one to delete items, one to process them. Create an EventBridge scheduled rule to invoke the Lambda functions. Use DeleteItem to delete items based on `expirationDate`.
- D. Enable TTL on the `expirationDate` attribute. Specify an Amazon SQS dead-letter queue as the target. Create a Lambda function to process the items.

---

### ✅ Correct Answer: A

### Explanation

The original application does two things: archives items and deletes them. The replacement must do the same with minimal code.

**Why A is correct:**

- **TTL** handles deletion automatically — no code needed to delete items.
- **DynamoDB Streams** captures the TTL deletion events (as DELETE records with `OLD_IMAGE`).
- **Lambda** (triggered by the Stream) handles archiving the item before it's gone. This replaces the entire application with just one Lambda function — minimum code.

**Why B is wrong:** Two Lambda functions + manual DeleteItem API calls = more code, more complexity, and WCU consumed.

**Why C is wrong:** EventBridge + two Lambda functions is the most complex approach. Not least code.

**Why D is wrong:** TTL does not write to an SQS dead-letter queue. DLQs are for failed message processing in SQS/Lambda, not for TTL deletions. This option is technically incorrect.

---

## Question 6 — Sorting Query Results in Descending Order

A developer creates an Amazon DynamoDB table with `OrderID` as the partition key and `NumberOfItemsPurchased` as the sort key. When the developer queries the table, results are sorted by `NumberOfItemsPurchased` in **ascending order**.

The developer needs the query results sorted in **descending order**. Which solution will meet this requirement?

- A. Create a local secondary index (LSI) on the `NumberOfItemsPurchased` sort key.
- B. Change the sort key from `NumberOfItemsPurchased` to `NumberOfItemsPurchasedDescending`.
- C. In the Query operation, set the `ScanIndexForward` parameter to `false`.
- D. In the Query operation, set the `KeyConditionExpression` parameter to `false`.

---

### ✅ Correct Answer: C

> Confirmed by multiple users as appearing on the real exam (July 2024).

### Explanation

DynamoDB always stores items within a partition sorted by the sort key in ascending order. However, you can control the **direction of results** returned by a Query using a single parameter.

**Why C is correct:** `ScanIndexForward = false` reverses the sort order, returning results in **descending** order. `ScanIndexForward = true` (default) returns results in ascending order. This is a simple flag — no schema changes, no new indexes required.

**Why A is wrong:** Creating an LSI does not change sort direction. LSIs also cannot be added after table creation, and the sort key is already `NumberOfItemsPurchased` anyway.

**Why B is wrong:** You cannot reverse sort order by renaming the attribute. DynamoDB sorts by value, not by attribute name.

**Why D is wrong:** `KeyConditionExpression` specifies _which items_ to query (partition key and sort key conditions). Setting it to `false` is invalid — it would cause an error, not a reversed sort.

---

## Question 7 — BatchGetItem and UnprocessedKeys

A developer's application uses `BatchGetItem` to retrieve items from an Amazon DynamoDB table. The application occasionally receives responses containing an `UnprocessedKeys` element.

The developer wants to increase the **resiliency** of the application when this occurs.

Which TWO actions should the developer take? (Select TWO)

- A. Immediately retry the batch operation as soon as `UnprocessedKeys` is returned.
- B. Implement exponential backoff when retrying requests for unprocessed keys.
- C. Add more read capacity units (RCU) to the table to handle the additional load.
- D. Use a Scan operation instead of BatchGetItem for better resilience.
- E. Use the `ProjectionExpression` to retrieve fewer attributes per item.

---

### ✅ Correct Answer: B and C

### Explanation

`UnprocessedKeys` in a `BatchGetItem` response means DynamoDB could not process some items due to insufficient provisioned throughput or internal limits.

**Why B is correct:** Exponential backoff is the AWS-recommended retry strategy. Retrying immediately (option A) can make throttling worse by hammering the table further. Exponential backoff gives the table time to recover between retries.

**Why C is correct:** If `UnprocessedKeys` occurs frequently, it may indicate the table is under-provisioned. Adding RCU gives the table more capacity to handle the batch operations.

**Why A is wrong:** Immediately retrying without delay increases pressure on an already throttled partition — the opposite of resilient behavior.

**Why D is wrong:** Scan reads the entire table — it is far less efficient than BatchGetItem and does not improve resilience. It would make things worse.

**Why E is wrong:** `ProjectionExpression` reduces the size of response payloads but does not reduce the number of items DynamoDB must read. RCU consumption is based on items read, not attributes returned.

---

## Question 8 — DynamoDB Streams + Lambda Event Filtering

A company uses an Amazon DynamoDB table to store sales data with DynamoDB Streams enabled. The `TransactionStatus` attribute can be `failed`, `pending`, or `completed`. The `Price` attribute stores the sale price.

The company wants to be **notified of failed sales where the Price is above a specific threshold**. A developer needs to set up notification with the **LEAST development effort**.

Which solution meets these requirements?

- A. Create an event source mapping between DynamoDB Streams and an AWS Lambda function. Use Lambda event filtering to trigger the function only if `TransactionStatus = failed` AND `Price > threshold`. Configure the Lambda function to publish to an Amazon SNS topic.
- B. Create an event source mapping between DynamoDB Streams and a Lambda function. In the Lambda handler code, check `TransactionStatus` and `Price`. If conditions match, publish to SNS. Otherwise, discard.
- C. Create an event source mapping between DynamoDB Streams and a Lambda function. Trigger an SNS topic for every DynamoDB change. Use SNS filter policies to filter by `TransactionStatus` and `Price`.
- D. Create a scheduled Lambda function that performs a Scan on the table, filters for failed sales above the threshold, and sends notifications via SNS.

---

### ✅ Correct Answer: A

### Explanation

Lambda event source mappings for DynamoDB Streams support **event filtering** — you can define filter criteria that prevent Lambda from being invoked at all unless the event matches.

**Why A is correct:** Lambda event filtering is applied at the service level before Lambda is even invoked. This means Lambda only runs when `TransactionStatus = failed` AND `Price > threshold`. This is the least development effort — no filtering logic inside the Lambda code, and SNS handles the notification. Least code, least invocations, lowest cost.

**Why B is wrong:** Functionally correct, but requires custom filtering code inside the Lambda handler. More development effort than option A, and Lambda is invoked for every DynamoDB change (wasting invocations on non-matching events).

**Why C is wrong:** SNS filter policies work on message attributes, not on arbitrary DynamoDB event fields. You cannot directly filter DynamoDB Streams records using SNS filter policies this way. Also more complex to set up.

**Why D is wrong:** A scheduled Scan is extremely inefficient — reads the entire table on every run. This consumes a lot of RCU, has high latency for notifications, and requires the most code.

---

## Question 9 — ACID Transactions Across Multiple Items

A developer needs a solution that supports **ACID-compliant operations** across multiple DynamoDB items. The solution must ensure that either all operations succeed or none do.

Which TWO AWS options provide ACID compliance? (Select TWO)

- A. Amazon DynamoDB with operations made with the `ConsistentRead` parameter set to `true`.
- B. Amazon ElastiCache for Memcached with operations made within a transaction block.
- C. Amazon DynamoDB with reads and writes made using `Transact*` operations.
- D. Amazon Aurora MySQL with operations made within a transaction block.
- E. Amazon Athena with operations made within a transaction block.

---

### ✅ Correct Answer: C and D

### Explanation

**Why C is correct:** `TransactWriteItems` and `TransactGetItems` provide ACID transactions in DynamoDB. All operations within a transaction are atomic — they either all succeed or all fail. This is the DynamoDB-native way to achieve ACID compliance across multiple items/tables.

**Why D is correct:** Amazon Aurora MySQL is a relational database that fully supports ACID transactions using standard SQL `BEGIN / COMMIT / ROLLBACK` blocks.

**Why A is wrong:** `ConsistentRead = true` gives you strongly consistent _reads_ — it is not a transaction. It does not make reads and writes atomic across multiple items.

**Why B is wrong:** ElastiCache for Memcached does not support transactions. It is a cache, not a transactional store.

**Why E is wrong:** Amazon Athena is a query service for S3. It does not support transactional write operations.

---

## Question 10 — Movie Database Design with GSI

A developer is building a movie catalog application. The application must:

- Retrieve information about a specific movie by **title and release year**.
- Retrieve all movies in a specific **genre**.
- Store flexible attributes (cast, crew roles vary per movie).

Which solution meets these requirements?

- A. Create a DynamoDB table with `title` as the partition key and `release_year` as the sort key. Create a GSI with `genre` as the partition key and `title` as the sort key.
- B. Create a DynamoDB table with `genre` as the partition key and `release_year` as the sort key. Create a GSI with `title` as the partition key.
- C. Create a DynamoDB table with `title` as the partition key and `genre` as the sort key. Add separate columns for each cast and crew role.
- D. Create a DynamoDB table with `genre` as the partition key and `title` as the sort key. Use an RDS table for flexible cast and crew attributes.

---

### ✅ Correct Answer: A

### Explanation

There are three requirements to satisfy:

**Requirement 1 — Retrieve by title + release year:** Set `title` as partition key and `release_year` as sort key. A GetItem or Query with both values retrieves the specific movie directly.

**Requirement 2 — Retrieve all movies in a genre:** Create a GSI with `genre` as the partition key. A Query on the GSI with a specific genre returns all movies in that genre efficiently.

**Requirement 3 — Flexible attributes (different roles per movie):** DynamoDB is schema-less — each item can have different attributes. No schema change needed.

**Why B is wrong:** If `genre` is the partition key of the base table, you cannot efficiently retrieve a movie by `title + release_year` without a Scan (expensive). The GSI only has `title`, so `release_year` filtering is not efficient.

**Why C is wrong:** Adding a separate column for every possible cast/crew role is rigid, not flexible, and violates good DynamoDB design. Also, `genre` as sort key prevents querying all movies in a genre efficiently across different titles.

**Why D is wrong:** Splitting data between DynamoDB and RDS adds unnecessary complexity and defeats the purpose of using DynamoDB's flexible schema. Joins across two database systems are not supported natively.

---

## Summary — What Each Question Tests

|#|Topic Tested|Key Concept|
|---|---|---|
|1|Hot partition|Bad partition key = hot partition; fix with higher cardinality key|
|2|LSI vs GSI|Same PK + diff SK = LSI; LSI supports strong consistency|
|3|GSI on existing table|GSI can be added anytime; LSI cannot have different PK|
|4|TTL|TTL deletes are free, no WCU consumed|
|5|TTL + Streams|TTL deletions appear in Streams; use Lambda to archive|
|6|ScanIndexForward|Set to `false` for descending sort order in Query|
|7|BatchGetItem + UnprocessedKeys|Exponential backoff + increase RCU|
|8|Streams + Lambda event filtering|Filter at the mapping level, not in handler code|
|9|ACID Transactions|Use `Transact*` APIs; `ConsistentRead` is NOT a transaction|
|10|Table design + GSI|Design for access patterns; use GSI for alternate query dimensions|

---

_Good luck on your DVA-C02 exam!_
