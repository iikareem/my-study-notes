---
tags:
  - aws
  - certification
  - messaging
  - sqs
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[SQS Quiz]] · [[SNS]]

# Amazon SQS — AWS Developer Associate Complete Guide

> **Exam weight:** SQS appears heavily in the _Developer Tools & Messaging_ domain and weaves into almost every architecture scenario. Expect 5–10 direct questions plus SQS as the "correct" answer in dozens of decoupling/scaling scenarios.

---

## Table of Contents

1. [What is SQS?](#1-what-is-sqs)
2. [Queue Types](#2-queue-types)
3. [Core Concepts & Terminology](#3-core-concepts--terminology)
4. [Message Lifecycle](#4-message-lifecycle)
5. [Visibility Timeout](#5-visibility-timeout)
6. [Dead Letter Queues (DLQ)](#6-dead-letter-queues-dlq)
7. [Long Polling vs Short Polling](#7-long-polling-vs-short-polling)
8. [Delay Queues & Message Timers](#8-delay-queues--message-timers)
9. [Message Attributes & Body](#9-message-attributes--body)
10. [SQS Extended Client & S3](#10-sqs-extended-client--s3)
11. [Security & Encryption](#11-security--encryption)
12. [Access Control (IAM + Resource Policies)](#12-access-control-iam--resource-policies)
13. [SQS with Lambda](#13-sqs-with-lambda)
14. [SQS with Auto Scaling](#14-sqs-with-auto-scaling)
15. [FIFO Queue Deep Dive](#15-fifo-queue-deep-dive)
16. [Limits & Quotas Cheat Sheet](#16-limits--quotas-cheat-sheet)
17. [Common Exam Scenarios & Traps](#17-common-exam-scenarios--traps)
18. [Quick-Reference Summary Table](#18-quick-reference-summary-table)

---

## 1. What is SQS?

Amazon Simple Queue Service (SQS) is a **fully managed message queuing service** that enables you to decouple and scale microservices, distributed systems, and serverless applications.

**Key characteristics:**

- **Pull-based** — consumers poll the queue (contrast with SNS which is push-based)
- **At-least-once delivery** (Standard queues) — a message may be delivered more than once
- **Exactly-once delivery** (FIFO queues) — each message processed exactly once
- **Unlimited throughput** on Standard queues
- **Serverless / fully managed** — no infrastructure to manage

**When SQS is the answer on the exam:**

- Decoupling two components (producer does not need to wait for consumer)
- Handling traffic bursts (queue absorbs spikes, consumers process at their own pace)
- Ensuring no message loss when a downstream service is temporarily unavailable
- Any phrase like "decouple", "asynchronous processing", "buffer requests"

---

## 2. Queue Types

### Standard Queue

|Property|Value|
|---|---|
|Throughput|Nearly unlimited (up to ~3,000 msg/s per API action for batches)|
|Ordering|**Best-effort** — messages may arrive out of order|
|Delivery|**At-least-once** — occasional duplicates possible|
|Use case|Maximum throughput, loose ordering acceptable|

### FIFO Queue

|Property|Value|
|---|---|
|Throughput|300 msg/s (3,000 msg/s with batching)|
|Ordering|**Strict FIFO** within a Message Group|
|Delivery|**Exactly-once** — deduplication built in|
|Naming|Must end with `.fifo` suffix|
|Use case|Financial transactions, order processing, anything requiring strict order|

> **Exam tip:** If the question mentions _ordering_ or _exactly-once_, the answer is almost always FIFO. If it mentions _maximum throughput_ or says "occasional duplicates are acceptable," it's Standard.

---

## 3. Core Concepts & Terminology

### Producer

Any application that **sends** messages to the queue using the `SendMessage` API.

### Consumer

Any application that **polls** the queue using the `ReceiveMessage` API, processes the message, then deletes it using `DeleteMessage`.

### Message

- Up to **256 KB** in size
- Contains a body (string), optional attributes, and system metadata
- Has a unique **Message ID** and **Receipt Handle** (used to delete or change visibility)

### Receipt Handle

A **temporary token** returned when a message is received. Required to:

- Delete the message (`DeleteMessage`)
- Change visibility timeout (`ChangeMessageVisibility`)

> **Important:** The receipt handle changes each time a message is received. Always use the most recent one.

### Queue URL

Each queue has a unique URL in the format:

```
https://sqs.<region>.amazonaws.com/<account-id>/<queue-name>
```

This URL is used in all API calls.

---

## 4. Message Lifecycle

```
Producer                   Queue                     Consumer
   |                         |                           |
   |--- SendMessage -------->|                           |
   |                         |<-- ReceiveMessage --------|
   |                         |--- Message + Receipt ---->|
   |                         |   (becomes invisible)     |
   |                         |                    [process]
   |                         |<-- DeleteMessage ---------|
   |                         |   (permanently removed)   |
```

**What happens if the consumer does NOT delete?**

The message becomes visible again after the **Visibility Timeout** expires, and another consumer can pick it up. This is how SQS handles consumer failure automatically — no manual dead-letter routing needed at this level.

---

 

---

## 6. Dead Letter Queues (DLQ)

A DLQ is a **separate queue** where SQS moves messages that cannot be processed successfully after a certain number of attempts.

### How it works

1. Set a `maxReceiveCount` on the **source queue's Redrive Policy**
2. When a message has been received more than `maxReceiveCount` times without being deleted, it is moved to the DLQ
3. You can then inspect, debug, and optionally reprocess messages from the DLQ

### Configuration

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789:MyDLQ",
  "maxReceiveCount": 5
}
```

### Key rules

- DLQ must be the **same type** as the source queue (Standard DLQ for Standard queue, FIFO DLQ for FIFO queue)
- DLQ must be in the **same AWS account and region**
- Messages in DLQ retain their original `MessageId`
- Set the DLQ's **retention period longer** than the source queue so you have time to debug

### DLQ Redrive (re-processing)

You can move messages back from the DLQ to the source queue using the **DLQ Redrive** feature (console or API). This allows retrying after fixing the bug.

> **Exam tip:** Any question about "what happens to messages that keep failing" or "how to handle poison pill messages" → DLQ.

---

## 7. Long Polling vs Short Polling

### Short Polling (default)

- SQS queries **only a subset** of servers immediately
- Returns right away, even if the queue is empty
- Can return empty responses, wasting API calls and money
- `WaitTimeSeconds = 0`

### Long Polling (recommended)

- SQS queries **all servers** and waits up to 20 seconds for a message to arrive
- Returns as soon as a message is available, or after the wait time expires
- **Reduces empty responses**, saves cost, reduces latency
- `WaitTimeSeconds = 1–20`

### How to enable long polling

**Queue level:** Set `ReceiveMessageWaitTimeSeconds` attribute on the queue.

**Per-request level:** Pass `WaitTimeSeconds` in the `ReceiveMessage` API call.

> **Exam tip:** "Reduce the number of empty responses" or "reduce cost of polling" → Long Polling. This is a very common exam question. Always prefer long polling over short polling.

---

## 8. Delay Queues & Message Timers

### Delay Queue

- Postpones delivery of **all new messages** to the queue for a specified duration
- Consumers cannot see the messages during the delay period
- Default delay: 0 seconds | Max: 15 minutes
- Set with `DelaySeconds` attribute on the queue

### Message Timer (per-message delay)

- Postpones delivery of a **specific message**
- Set `DelaySeconds` parameter on the `SendMessage` call
- Overrides the queue-level delay for that message
- **Only works on Standard queues** — FIFO queues do not support per-message timers

> **Exam scenario:** "A message should not be processed for 5 minutes after it is sent" → Use `DelaySeconds` on the message or configure the queue's `DelaySeconds`.

---

## 9. Message Attributes & Body

### Message Body

- Plain text (string), up to **256 KB**
- For larger payloads, use S3 (see Section 10)

### Message Attributes

Up to **10 metadata attributes** per message (name-type-value triplets). Types: `String`, `Number`, `Binary`.

```json
{
  "OrderType": {
    "DataType": "String",
    "StringValue": "express"
  },
  "Priority": {
    "DataType": "Number",
    "StringValue": "1"
  }
}
```

Useful for routing logic without parsing the message body.

### Batch Operations

Use batch API calls to reduce cost and increase throughput:

|API|Batch size|
|---|---|
|`SendMessageBatch`|Up to 10 messages|
|`DeleteMessageBatch`|Up to 10 messages|
|`ChangeMessageVisibilityBatch`|Up to 10 messages|

Each batch counts as **one API request** (significant cost saving).

---

## 10. SQS Extended Client & S3

When your message payload exceeds **256 KB**, use the **Amazon SQS Extended Client Library** (available for Java and other SDKs).

**How it works:**

1. Large payload is stored in **S3**
2. SQS message contains a **pointer (reference)** to the S3 object
3. Consumer receives the pointer, fetches from S3, processes the full payload

This pattern is also called the **Claim Check Pattern**.

> **Exam tip:** "Message size exceeds 256 KB" → SQS Extended Client + S3. This is the only supported way to send large payloads through SQS.

---

## 11. Security & Encryption

### Encryption In Transit

- All SQS API calls use **HTTPS** by default
- Data is encrypted in transit using TLS

### Encryption At Rest (SSE)

SQS supports **Server-Side Encryption** using AWS KMS.

- Encrypts message body (not message attributes)
- Uses a **Customer Master Key (CMK)** in AWS KMS
- Set `KmsMasterKeyId` attribute on the queue
- Each message is encrypted individually

**KMS key options:**

|Key Type|Description|
|---|---|
|`alias/aws/sqs`|AWS-managed key (free, less control)|
|Custom CMK|Your own KMS key (cost per API call, full control)|

### Client-Side Encryption

You can also encrypt messages **before** sending to SQS using your own encryption logic. SQS is unaware of the encryption.

> **Exam tip:** If the question asks about **encrypting messages at rest** in SQS → SSE with KMS. If the question requires the **application** to control encryption → Client-side encryption.

---

## 12. Access Control (IAM + Resource Policies)

### IAM Policies

Control which **IAM identities** (users, roles, services) can perform SQS actions.

Common IAM actions:

- `sqs:SendMessage`
- `sqs:ReceiveMessage`
- `sqs:DeleteMessage`
- `sqs:GetQueueAttributes`
- `sqs:CreateQueue`
- `sqs:PurgeQueue`

### SQS Resource Policy (Queue Policy)

A **resource-based policy** attached directly to the queue. Required for **cross-account access** or allowing AWS services (like SNS, S3 Events) to send messages.

**Example — Allow SNS to send messages:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "sns.amazonaws.com" },
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:123456789:MyQueue",
    "Condition": {
      "ArnEquals": { "aws:SourceArn": "arn:aws:sns:us-east-1:123456789:MyTopic" }
    }
  }]
}
```

**Example — Allow another account:**

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::999999999:root" },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:us-east-1:123456789:MyQueue"
}
```

> **Exam rule:** Same account → use IAM policy. Cross-account or AWS service → use Queue Resource Policy.

---

## 13. SQS with Lambda

Lambda can be configured as an **event source** for SQS — it will automatically poll the queue and invoke your function.

### How it works

1. Lambda polls the queue using **long polling**
2. Lambda reads up to **10 messages** per batch (configurable, up to 10,000 for Standard; 10 for FIFO)
3. Lambda invokes your function with the batch
4. If the function **succeeds** → Lambda deletes all messages in the batch
5. If the function **fails** → messages become visible again (after visibility timeout) and will be retried

### Key configuration settings

|Setting|Description|
|---|---|
|`BatchSize`|Number of messages per Lambda invocation (1–10,000 for Standard, 1–10 for FIFO)|
|`MaximumBatchingWindowInSeconds`|Wait up to N seconds to accumulate a larger batch|
|`FunctionResponseTypes: ReportBatchItemFailures`|Partial batch failure — only failed messages are retried|

### Partial Batch Failure (important exam topic!)

By default, if **any message** in a batch fails, **all messages** in the batch are retried. This can cause duplicate processing.

To fix this, enable **`ReportBatchItemFailures`**:

```json
{
  "batchItemFailures": [
    { "itemIdentifier": "messageId1" },
    { "itemIdentifier": "messageId2" }
  ]
}
```

Return this JSON from your Lambda. Only the listed message IDs will be retried.

### Scaling behavior

- Lambda automatically scales the number of concurrent executions based on queue depth
- For Standard queues: Lambda scales up to **1,000 concurrent executions**
- For FIFO queues: Lambda scales up to the number of **active message groups**

> **Exam scenario:** "Lambda function failing causes all messages to reprocess" → Enable `ReportBatchItemFailures`. "Need to process SQS messages in a Lambda function" → SQS as event source mapping.

---

## 14. SQS with Auto Scaling

A common architecture pattern: use **SQS queue depth** as the metric to auto-scale EC2 instances (consumers).

### Pattern

```
Producers --> SQS Queue --> EC2 Auto Scaling Group (consumers)
                  |
            CloudWatch Metric
            (ApproximateNumberOfMessagesVisible)
                  |
            Auto Scaling Policy
```

### How to implement

1. Create a **CloudWatch Alarm** on `ApproximateNumberOfMessagesVisible`
2. Attach a **Step Scaling** or **Target Tracking** policy to the Auto Scaling Group
3. Scale out when queue depth is high, scale in when queue is empty

### Important CloudWatch metrics for SQS

|Metric|Description|
|---|---|
|`ApproximateNumberOfMessagesVisible`|Messages available to be received|
|`ApproximateNumberOfMessagesNotVisible`|Messages currently in flight (being processed)|
|`ApproximateAgeOfOldestMessage`|Age of the oldest message — useful for SLA monitoring|
|`NumberOfMessagesSent`|Messages added per time period|
|`NumberOfMessagesDeleted`|Messages deleted per time period|

> **Exam tip:** `ApproximateAgeOfOldestMessage` is the best metric when you have an **SLA** on processing time (e.g., "alert if any message is older than 5 minutes").

---

## 15. FIFO Queue Deep Dive

### Message Group ID

- Groups messages into **ordered sequences**
- Messages within the same group are processed **strictly in order**
- Messages in **different groups** can be processed in parallel
- Required on every `SendMessage` call to a FIFO queue

**Example:** Use `customerId` as the Message Group ID → all orders from the same customer are processed in order, but different customers are processed in parallel.

### Message Deduplication ID

Prevents duplicate messages within a **5-minute deduplication window**.

Two methods:

1. **Content-based deduplication**: SQS generates a SHA-256 hash of the message body as the deduplication ID. Enable with `ContentBasedDeduplication = true`
2. **Explicit deduplication ID**: You provide a `MessageDeduplicationId` on each `SendMessage` call

If the same deduplication ID is sent twice within 5 minutes, the second message is **silently dropped**.

### FIFO throughput limits

|Mode|Throughput|
|---|---|
|Without batching|300 transactions/second|
|With batching (up to 10 messages/batch)|3,000 messages/second|

### FIFO naming

The queue name **must end with `.fifo`**:

```
MyOrderQueue.fifo
```

### FIFO with Lambda

- Lambda processes messages **per message group**
- Only one Lambda invocation per message group at a time (preserves ordering)
- Scales up to the number of distinct message groups

---

## 16. Limits & Quotas Cheat Sheet

|Limit|Value|
|---|---|
|**Message size**|256 KB (max)|
|**Message retention**|1 minute – 14 days (default: 4 days)|
|**Visibility timeout**|0 seconds – 12 hours (default: 30 seconds)|
|**Delay seconds**|0 – 15 minutes|
|**Long poll wait time**|1 – 20 seconds|
|**In-flight messages (Standard)**|120,000|
|**In-flight messages (FIFO)**|20,000|
|**Batch size**|Up to 10 messages|
|**Max message attributes**|10 per message|
|**FIFO deduplication window**|5 minutes|
|**Standard throughput**|Nearly unlimited|
|**FIFO throughput (no batching)**|300 msg/s|
|**FIFO throughput (with batching)**|3,000 msg/s|
|**Queue name length**|1–80 characters|

---

## 17. Common Exam Scenarios & Traps

### Scenario 1: Duplicate message processing

**Symptom:** Messages are processed more than once.

**Causes & fixes:**

- Standard queue → duplicates are expected (design for idempotency)
- Visibility timeout too short → consumer doesn't finish before timeout → message reappears → extend visibility timeout
- Consumer crashes after processing but before `DeleteMessage` → implement idempotency on consumer side

### Scenario 2: Messages piling up (backlog)

**Symptom:** `ApproximateNumberOfMessagesVisible` keeps growing.

**Fixes:**

- Scale out consumers (manually or with Auto Scaling)
- Increase batch size
- Optimize consumer processing speed

### Scenario 3: "Poison pill" messages

**Symptom:** One specific message keeps failing and blocking others.

**Fix:** Configure a **DLQ** with an appropriate `maxReceiveCount`. The bad message gets moved to DLQ after N failures; other messages continue processing.

### Scenario 4: Cross-account publishing

**Requirement:** Account A's SNS topic sends to Account B's SQS queue.

**Fix:** Add a **Queue Resource Policy** on Account B's queue granting `sqs:SendMessage` to Account A's SNS topic ARN.

### Scenario 5: Ordered processing

**Requirement:** Messages from the same customer must be processed in order.

**Fix:** Use a **FIFO queue** with `customerId` as the **Message Group ID**.

### Scenario 6: Large messages (> 256 KB)

**Fix:** Use **SQS Extended Client + S3** (Claim Check Pattern).

### Scenario 7: Lambda retrying entire failed batch

**Fix:** Enable **`ReportBatchItemFailures`** and return only failed message IDs from Lambda.

### Scenario 8: Reduce empty API calls / reduce cost

**Fix:** Enable **Long Polling** (`WaitTimeSeconds = 20`).

### Scenario 9: Temporary delay before processing

**Fix:** Use `DelaySeconds` on the message or set queue-level `DelaySeconds`.

### Scenario 10: SNS Fan-out to multiple SQS queues

**Pattern:** SNS topic → multiple SQS queues (each subscribed to the topic). Each queue gets its own copy of every message. Use this when multiple independent consumers need the same event.

---

## 18. Quick-Reference Summary Table

| Feature             | Standard Queue | FIFO Queue              |
| ------------------- | -------------- | ----------------------- |
| Ordering            | Best-effort    | Strict (per group)      |
| Delivery            | At-least-once  | Exactly-once            |
| Throughput          | Unlimited      | 300/s (3,000/s batched) |
| Deduplication       | No             | Yes (5-min window)      |
| Message Group ID    | Not supported  | Required                |
| Per-message delay   | Yes            | No                      |
| DLQ support         | Yes            | Yes (FIFO DLQ only)     |
| Lambda event source | Yes            | Yes                     |
| Name suffix         | None           | Must end in `.fifo`     |

---

## Key APIs to Know

|API|Description|
|---|---|
|`CreateQueue`|Create a new queue|
|`SendMessage`|Send one message|
|`SendMessageBatch`|Send up to 10 messages|
|`ReceiveMessage`|Poll for messages (returns up to 10)|
|`DeleteMessage`|Permanently delete a processed message|
|`DeleteMessageBatch`|Delete up to 10 messages|
|`ChangeMessageVisibility`|Extend/reset visibility timeout|
|`GetQueueAttributes`|Get queue configuration/stats|
|`SetQueueAttributes`|Update queue settings|
|`PurgeQueue`|Delete **all** messages in the queue|
|`GetQueueUrl`|Get queue URL from queue name|
|`ListQueues`|List queues (optionally filter by prefix)|

---

## Architecture Pattern Cheat Sheet

```
[Tight coupling — BAD]
App A --> HTTP call --> App B (App B down = App A fails)

[Loose coupling with SQS — GOOD]
App A --> SQS --> App B (App B down = messages queue up, no data loss)

[Fan-out pattern]
Event --> SNS Topic --> SQS Queue 1 --> Consumer A
                    --> SQS Queue 2 --> Consumer B
                    --> SQS Queue 3 --> Consumer C

[Auto scaling pattern]
Producers --> SQS --> EC2 ASG (scales based on queue depth via CloudWatch)

[Lambda async processing]
API Gateway --> Lambda --> SQS --> Lambda (processor) --> DynamoDB
```

---

> **Final exam advice:** SQS is almost always the right answer when you see the word "decouple". When you see "order matters", think FIFO. When you see "messages failing repeatedly", think DLQ. When you see "empty responses from polling", think Long Polling. These four rules will cover the majority of SQS questions on the exam.

---

_Guide covers the AWS Developer Associate (DVA-C02) exam objectives for Amazon SQS._
