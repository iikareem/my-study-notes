---
tags:
  - aws
  - certification
  - dynamodb
  - practice
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[DynamoDB]] · [[DynamoDB Quiz]]

# DVA-C02 — DynamoDB: Exam Hints

## RCU & WCU Calculations (MUST MEMORIZE)
- **1 WCU** = 1 write/sec for an item up to **1 KB** (round up)
- **1 RCU** = 1 strongly consistent read/sec for an item up to **4 KB** (round up)
- Eventually consistent reads cost **half** (0.5 RCU per 4 KB)
- Transactional reads/writes cost **2x**

**Example:** 3.5 KB item, 10 writes/sec → ceil(3.5)=4 → 4×10 = **40 WCU**

## Primary Keys
- **Simple** (partition key only) — must be unique
- **Composite** (partition key + sort key) — combination must be unique, same PK can repeat
- Bad PK = **hot partition** (e.g., using `country` as PK when most users are in one country)
- Fix hot partition → **write sharding** (add random suffix to PK)

## LSI vs GSI
| LSI | GSI |
|---|---|
| Same PK as base table, different SK | Completely different PK |
| Created **only at table creation** | Can be created **any time** |
| Supports **strongly consistent** reads | Eventually consistent only |
| Shares base table capacity | Has its own provisioned capacity |
| Max 5 per table | Max 20 per table |
| 10 GB per partition limit | No size limit |

**If GSI is under-provisioned → base table writes are REJECTED.**

## DAX (DynamoDB Accelerator)
- In-memory cache → **microsecond** read latency
- No code changes needed (same DynamoDB API)
- **Only helps eventually consistent reads** — strongly consistent bypass DAX
- Good for read-heavy, repeated access patterns
- Bad for strongly consistent requirements, write-heavy, or infrequent access

## TTL (Time To Live)
- Automatically deletes items after a timestamp
- **Free** — no WCU consumed
- Deletion appears in DynamoDB Streams (can trigger Lambda)
- Expired items may still be readable for up to **48 hours**

## Key APIs
- **GetItem** — single item by PK (most efficient)
- **Query** — all items sharing a PK (use KeyConditionExpression)
- **Scan** — reads every item (expensive, avoid)
- **BatchGetItem** — up to 100 items, max 16MB, handle UnprocessedKeys with exponential backoff
- **TransactWriteItems / TransactGetItems** — ACID across multiple items/tables (2x cost)

## Streams
- **24-hour retention** only
- View types: KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES
- If exam says "longer retention" or "multiple consumers" → use **Kinesis Data Streams** instead

## Global Tables
- Multi-region active-active replication
- Requires **Streams** enabled
- Asynchronous, eventually consistent
- Last writer wins for conflict resolution

## Backup
- **On-Demand Backup**: full backup, retained until deleted, no performance impact
- **PITR**: continuous backup, restore to any second in last **35 days**, must be **enabled explicitly**
