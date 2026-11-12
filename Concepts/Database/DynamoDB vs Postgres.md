# DynamoDB vs Postgres

## Core mental model

> **Postgres: model the data, then ask any question.**  
> **DynamoDB: model the questions, then shape the data to answer them.**

## When to choose which

- **DynamoDB:** access patterns are known upfront and stable, data accessed together is stored together, no joins needed, and you need predictable performance at huge scale.
- **Postgres:** data has relationships, you need joins, aggregations, or ad-hoc queries, and requirements are still evolving.
- **Default:** when unsure, start with Postgres. Moving to DynamoDB later is easier than discovering your access patterns were wrong after building on it.

## Comparison

||**Postgres**|**DynamoDB**|
|---|---|---|
|Design starting point|Entities and relationships (normalize)|Access patterns (denormalize)|
|Query flexibility|Ad-hoc SQL, joins, aggregations|Only by key or index (PK + SK, GSI/LSI)|
|Schema|Strict, enforced by the DB|Only the keys are fixed|
|Changing queries later|Cheap (add an index or write new SQL)|Expensive (new GSI, or re-model and backfill)|
|Scaling|Vertical, plus replicas|Horizontal, automatic partitioning|
|Latency at scale|Depends on query and data size|Consistent single-digit ms|
|Transactions|Full ACID|Supported but limited|
|Ops burden|You tune and scale|Fully managed|

## Why latency stays flat at a billion items

- The table is split into many **partitions** across many nodes.
- DynamoDB **hashes the partition key** to find the exact partition. There is no searching across the table.
- A lookup touches one partition, so cost depends on partition size, not table size.
- This only works when you query by key. A `Scan` reads every partition and loses the advantage.

## The 8 must-know essentials

1. **PK + SK design.** The PK picks the partition, the SK enables range queries (`begins_with`, `between`) inside it.
2. **Query vs Scan.** Query uses the key and is fast. Avoid Scan in production paths.
3. **Hot partitions.** Use a high-cardinality key that spreads traffic evenly.
4. **GSI vs LSI.**
    - **GSI:** own PK/SK, can be added later, eventually consistent only.
    - **LSI:** same PK as the table, must be created with the table, supports strong consistency.
5. **Consistency.** Eventually consistent by default, strongly consistent is optional (not on GSIs).
6. **Limits.** 400 KB max item size, about 3,000 RCU and 1,000 WCU per partition per second.
7. **Capacity modes.** On-demand for unpredictable traffic, provisioned for steady traffic.
8. **No joins.** Denormalize and duplicate data, and the application keeps copies consistent.

**Extras:** conditional writes (optimistic locking, idempotency), TTL, Streams, transactions.

## Example: customer and orders without a join

|PK|SK|Attributes|
|---|---|---|
|`CUSTOMER#123`|`PROFILE`|name, email|
|`CUSTOMER#123`|`ORDER#2026-10-01#A1`|total, status|
|`CUSTOMER#123`|`ORDER#2026-10-15#B7`|total, status|

```javascript
// Customer + all orders in one call
KeyConditionExpression: "PK = :pk"

// Only orders
KeyConditionExpression: "PK = :pk AND begins_with(SK, :prefix)"
// :prefix = "ORDER#"

// Orders in a date range
"PK = :pk AND SK BETWEEN :start AND :end"

// Latest 10 orders
ScanIndexForward: false, Limit: 10
```

- **Prefixes** (`PROFILE`, `ORDER#`) let you fetch one entity type.
- **Date in the SK** keeps orders sorted.
- **Order ID at the end** keeps each SK unique.
- **Tradeoff:** "get an order by ID without knowing the customer" has no path. You need a GSI (e.g. `PK = ORDER#A1`). This is why you list access patterns first.

## Self-test

1. Why does latency stay flat at a billion items? _(PK hash routes to one partition.)_
2. What makes a bad partition key? _(Low cardinality or skewed traffic, causing a hot partition.)_
3. When do you need a GSI, and what's its tradeoff? _(A new access pattern. It's eventually consistent and costs extra writes.)_
4. Why can't you add an LSI later? _(It must be defined at table creation.)_
5. How do you model "customer plus orders" without a join? _(Same PK, different SK prefixes.)_