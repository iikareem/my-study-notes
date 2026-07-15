---
tags:
  - aws
  - certification
  - lambda
  - practice
---

**Hub:** [[AWS MOC]] · **Role:** Hints
**Also:** [[Lambda]]

# DVA-C02 — Lambda: Exam Hints

## Limits (MUST MEMORIZE)
| Limit | Value |
|---|---|
| Max timeout | **15 minutes (900 seconds)** |
| Max memory | **10,240 MB** (10 GB) |
| Max /tmp storage | **10 GB** |
| Deployment package (ZIP) | 50 MB compressed, 250 MB uncompressed |
| Container image (ECR) | Up to **10 GB** |
| Concurrency (default) | **1,000 per region** (soft limit, can be raised) |

## CPU & Memory
- You only configure **memory** — CPU scales proportionally
- More memory = more CPU (not just more RAM)
- If code is CPU-heavy → increase memory slider (not a separate CPU setting)

## Cold Starts vs Warm Starts
- **Cold start**: Lambda allocates container, downloads code, initializes runtime
- **Warm start**: reuses existing execution context (faster)
- **Optimization**: Initialize SDK clients, DB connections, etc. **outside** the handler function
  - Code outside handler runs ONCE during cold start
  - Code inside handler runs on EVERY invocation

## Concurrency
- **Reserved Concurrency**: hard limit for a specific function (prevents runaway scaling, guarantees capacity)
- **Provisioned Concurrency**: keeps N instances warm (eliminates cold starts, costs money)
- If a function hits reserved concurrency → additional requests get **throttled (error 429)**

## VPC Networking
- By default, Lambda can reach the internet but **cannot** reach resources inside your VPC
- When attached to a VPC: Lambda **CAN** reach VPC resources but **LOSES internet access**
- To give Lambda internet access while in VPC:
  - Place Lambda in **private subnet**
  - Route traffic through **NAT Gateway** in public subnet
- Lambda in a **public subnet** with IGW does NOT give it internet access (Lambda ENIs don't get public IPs)

## Invocation Types
- **Synchronous** (e.g., API Gateway, SDK invoke): returns response immediately, handle errors in code
- **Asynchronous** (e.g., S3, SNS, EventBridge): Lambda retries twice, can use **DLQ** (SQS/SNS) for failed events
- **Event source mappings** (e.g., DynamoDB Streams, SQS, Kinesis): Lambda polls the source, batch processing

## Deployment
- **Environment variables**: store config (DB_HOST, etc.), can encrypt with KMS
- **IAM Execution Role**: Lambda needs a role with permissions to access other AWS services (at minimum: CloudWatch Logs)
- **Versions & Aliases**: `$LATEST` = working draft, version = immutable snapshot, alias = pointer (e.g., "prod" → version 2)

## Exam Scenarios
- "Optimize DB performance" → Initialize connection **outside** handler
- "Eliminate cold starts" → **Provisioned Concurrency**
- "Lambda can't reach internet after VPC attach" → **NAT Gateway** in public subnet
- "Error 429" → hitting concurrency limit, increase limit or use reserved concurrency
- "Process DynamoDB Stream events" → Lambda event source mapping with filter criteria
