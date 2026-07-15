---
tags:
  - aws
  - certification
  - messaging
  - sns
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[SNS Quiz]] · [[SQS]]

# Amazon SNS — AWS Developer Associate Complete Guide

> **Exam weight:** SNS appears heavily in messaging, event-driven architecture, and fan-out scenarios. It pairs almost always with SQS and Lambda. Expect 4–8 direct questions plus SNS as the right answer in notification/broadcast/fan-out scenarios.

---

## Table of Contents

1. [What is SNS?](#1-what-is-sns)
2. [Core Concepts & Terminology](#2-core-concepts--terminology)
3. [Topic Types](#3-topic-types)
4. [Supported Subscription Protocols](#4-supported-subscription-protocols)
5. [Message Publishing](#5-message-publishing)
6. [Message Filtering](#6-message-filtering)
7. [Fan-out Pattern (SNS + SQS)](#7-fan-out-pattern-sns--sqs)
8. [SNS + Lambda](#8-sns--lambda)
9. [SNS + SQS + Lambda Full Architecture](#9-sns--sqs--lambda-full-architecture)
10. [Message Attributes](#10-message-attributes)
11. [Dead Letter Queues (DLQ) for SNS](#11-dead-letter-queues-dlq-for-sns)
12. [SNS FIFO Topics](#12-sns-fifo-topics)
13. [Security & Encryption](#13-security--encryption)
14. [Access Control (IAM + Resource Policies)](#14-access-control-iam--resource-policies)
15. [Message Delivery Retries](#15-message-delivery-retries)
16. [SNS vs SQS — Key Differences](#16-sns-vs-sqs--key-differences)
17. [Limits & Quotas Cheat Sheet](#17-limits--quotas-cheat-sheet)
18. [Common Exam Scenarios & Traps](#18-common-exam-scenarios--traps)
19. [Quick-Reference Summary Table](#19-quick-reference-summary-table)

---

## 1. What is SNS?

Amazon Simple Notification Service (SNS) is a **fully managed pub/sub messaging service** that enables you to decouple microservices, distributed systems, and serverless applications through **push-based** notifications.

**Key characteristics:**

- **Push-based** — SNS pushes messages to all subscribers immediately (contrast with SQS which is pull-based)
- **One-to-many** — one message → many subscribers (broadcast / fan-out)
- **No message persistence** — if a subscriber is unavailable and has no DLQ, the message is lost
- **Serverless / fully managed** — no infrastructure to manage

**When SNS is the answer on the exam:**

- Sending notifications to multiple systems simultaneously
- Triggering multiple Lambda functions from one event
- Broadcasting to email/SMS/HTTP endpoints
- Any phrase like "fan-out", "notify multiple subscribers", "broadcast", "pub/sub"

---

## 2. Core Concepts & Terminology

### Topic

A **logical channel** that publishers send messages to and subscribers receive messages from. It is the central entity in SNS.

- Has a unique **ARN**: `arn:aws:sns:<region>:<account-id>:<topic-name>`
- You can have up to **100,000 topics** per account (soft limit)

### Publisher

Any entity that sends (publishes) a message to a topic using the `Publish` API. Can be an application, another AWS service (S3, CloudWatch, CodePipeline), or a user.

### Subscriber

Any endpoint that is registered to a topic. SNS pushes every published message to all confirmed subscribers. Supported protocols include SQS, Lambda, HTTP/S, email, SMS, mobile push.

### Subscription

The link between a topic and an endpoint. Each subscription has a unique **ARN** and a **status** (pending confirmation → confirmed).

### Subscription Confirmation

For HTTP/S and email endpoints, SNS sends a **confirmation request** with a token. The endpoint must confirm by visiting a URL or calling `ConfirmSubscription` with the token. SQS and Lambda subscriptions are auto-confirmed.

---

## 3. Topic Types

### Standard Topic

|Property|Value|
|---|---|
|Throughput|Nearly unlimited|
|Ordering|Best-effort (no guaranteed order)|
|Delivery|At-least-once|
|Subscribers|SQS, Lambda, HTTP/S, Email, SMS, Mobile Push, Kinesis Data Firehose|

### FIFO Topic

| Property    | Value                                 |
| ----------- | ------------------------------------- |
| Throughput  | 300 msg/s (3,000 msg/s with batching) |
| Ordering    | Strict FIFO                           |
| Delivery    | Exactly-once                          |
| Subscribers | **SQS FIFO queues only**              |
| Naming      | Must end with `.fifo` suffix          |

> **Exam tip:** SNS FIFO → can only fan out to **SQS FIFO queues**. If you need strict ordering AND fan-out, combine SNS FIFO + SQS FIFO.

---

## 4. Supported Subscription Protocols

|Protocol|Description|Exam Notes|
|---|---|---|
|**SQS**|Deliver to an SQS queue|Most common, enables fan-out + persistence|
|**Lambda**|Invoke a Lambda function|Serverless processing|
|**HTTP / HTTPS**|POST to a web endpoint|Requires subscription confirmation|
|**Email**|Send an email|Human-readable; requires confirmation|
|**Email-JSON**|Send raw JSON email|Machine-readable|
|**SMS**|Send a text message|Phone number format required|
|**Mobile Push**|APNS, GCM/FCM, ADM, Baidu|Requires a Platform Application|
|**Kinesis Data Firehose**|Deliver to Firehose for S3/Redshift/ES|Data archiving / analytics|

> **Exam tip:** SNS cannot directly write to S3, DynamoDB, or RDS. Use Kinesis Firehose as a subscriber if you need to land data in S3.

---

## 5. Message Publishing

### Publish to a Topic

```json
{
  "TopicArn": "arn:aws:sns:us-east-1:123456789:MyTopic",
  "Message": "Hello subscribers!",
  "Subject": "Optional subject (used for email subscriptions)",
  "MessageAttributes": { ... }
}
```

### Message Structure — Per-Protocol Messages

You can send **different message bodies to different subscriber types** using `MessageStructure = "json"`:

```json
{
  "default": "Default message for all protocols",
  "email": "Hey! You have a new order.",
  "sqs": "{\"orderId\": \"123\", \"status\": \"placed\"}",
  "lambda": "{\"orderId\": \"123\", \"priority\": \"high\"}"
}
```

Set `MessageStructure = "json"` on the `Publish` call. SNS picks the matching key for each subscriber's protocol, falling back to `"default"` if no match.

> **Exam tip:** This is tested as "how do you send different payloads to email subscribers vs SQS subscribers from the same topic."

### Message Size

- Maximum message size: **256 KB**
- For larger payloads, store in S3 and publish the S3 reference (similar to SQS Extended Client)

---

## 6. Message Filtering

By default, every subscriber receives **every message** published to a topic. Message filtering lets each subscription receive only the messages it cares about.

### How it works

1. Publisher adds **Message Attributes** to the published message
2. Each subscription has a **Filter Policy** (JSON object)
3. SNS only delivers a message to a subscription if the message attributes **match** the filter policy
4. If a subscription has no filter policy → it receives **all** messages

### Filter Policy example

**Published message attributes:**

```json
{
  "orderType": { "DataType": "String", "StringValue": "express" },
  "region": { "DataType": "String", "StringValue": "us-east" }
}
```

**Subscription A filter (receives express orders):**

```json
{
  "orderType": ["express"]
}
```

**Subscription B filter (receives all standard orders from any US region):**

```json
{
  "orderType": ["standard"],
  "region": [{"prefix": "us-"}]
}
```

### Filter Policy operators

|Operator|Example|
|---|---|
|Exact match|`["express", "standard"]`|
|Numeric match|`[{"numeric": [">=", 100]}]`|
|Prefix match|`[{"prefix": "us-"}]`|
|Exists|`[{"exists": true}]`|
|Anything-but|`[{"anything-but": ["free"]}]`|

### Filter Policy Scope

- **MessageAttributes** (default) — filter on message attributes
- **MessageBody** — filter on fields inside the JSON message body (newer feature)

> **Exam scenario:** "Different teams subscribe to the same SNS topic but each team should only receive relevant events" → **Message Filtering with Filter Policies**.

---

## 7. Fan-out Pattern (SNS + SQS)

The most important SNS architecture pattern on the exam.

### The problem without fan-out

If you publish directly to multiple SQS queues, you need to make N separate `SendMessage` calls — tightly coupling the publisher to each queue.

### The solution

```
Publisher
    |
    v
SNS Topic
    |
    |--- SQS Queue A (consumed by Service A)
    |--- SQS Queue B (consumed by Service B)
    |--- SQS Queue C (consumed by Service C)
    |--- Lambda Function (real-time processing)
    |--- Email (ops notification)
```

**Benefits:**

- Publisher only makes **one `Publish` call**
- Each subscriber is decoupled from the publisher and from each other
- SQS adds **persistence and buffering** — if Service B is slow, its queue holds messages
- Each queue can have its own **DLQ, visibility timeout, and scaling policy**
- New subscribers can be added **without changing the publisher**

### Cross-account fan-out

To fan-out to an SQS queue in another account:

1. Set a **Queue Resource Policy** on the SQS queue allowing `sqs:SendMessage` from the SNS topic ARN
2. Subscribe the cross-account SQS queue to the SNS topic

> **Exam rule:** Any time you see "multiple consumers", "decouple and broadcast", or "process the same event in multiple ways" → SNS fan-out to SQS queues.

---

## 8. SNS + Lambda

When Lambda subscribes to an SNS topic:

- SNS **pushes** the message to Lambda (invokes it asynchronously)
- Lambda receives the SNS message wrapped in an event envelope
- Lambda **retries** automatically on failure (up to 2 retries by default for async invocations)
- After retries are exhausted, configure a **Lambda DLQ or Destination** to capture failures

### Lambda event structure from SNS

```json
{
  "Records": [
    {
      "EventSource": "aws:sns",
      "Sns": {
        "TopicArn": "arn:aws:sns:...",
        "Message": "your message body",
        "MessageAttributes": { ... },
        "Timestamp": "2024-01-01T00:00:00.000Z"
      }
    }
  ]
}
```

> **Exam tip:** SNS → Lambda is **async invocation**. Lambda has its own retry + DLQ behavior. This is different from SQS → Lambda (event source mapping, synchronous invocation from Lambda's perspective).

---

## 9. SNS + SQS + Lambda Full Architecture

The most production-ready and exam-relevant architecture:

```
Event Source
    |
    v
SNS Topic  (broadcast)
    |
    |--- SQS Queue 1 ---> Lambda A  (order processor)
    |--- SQS Queue 2 ---> Lambda B  (inventory updater)
    |--- SQS Queue 3 ---> Lambda C  (notification sender)
                |
               DLQ  (failed messages)
```

**Why this beats SNS → Lambda directly:**

||SNS → Lambda|SNS → SQS → Lambda|
|---|---|---|
|Buffering|None|Yes (SQS absorbs spikes)|
|Retry control|Lambda retries only (2x)|Configurable `maxReceiveCount` on DLQ|
|Batch processing|No|Yes (Lambda batch size)|
|Rate limiting|Lambda concurrency limits hit instantly|SQS smooths the load|
|Replay failures|Not easily|Yes (DLQ redrive)|

> **Exam rule:** For resilient, scalable async processing → always prefer **SNS → SQS → Lambda** over direct **SNS → Lambda**.

---

## 10. Message Attributes

SNS message attributes are metadata key-value pairs that travel with the message. Used for:

- **Message filtering** (the primary use case)
- Passing context without modifying the message body

Up to **10 attributes** per message. Supported data types: `String`, `String.Array`, `Number`, `Binary`.

```json
{
  "eventType": {
    "DataType": "String",
    "StringValue": "ORDER_PLACED"
  },
  "amount": {
    "DataType": "Number",
    "StringValue": "149.99"
  }
}
```

---

## 11. Dead Letter Queues (DLQ) for SNS

Unlike SQS where DLQs handle message processing failures, SNS DLQs handle **delivery failures** — when SNS cannot deliver a message to a subscriber endpoint.

### How SNS DLQ works

- Configured **per subscription** (not per topic)
- When SNS fails to deliver to a subscriber after all retries → message goes to the subscription's DLQ (an SQS queue)
- Helps debug delivery issues: endpoint down, permission errors, Lambda throttling

### Configuration

Set the `RedrivePolicy` on the subscription:

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:us-east-1:123456789:MyDLQ"
}
```

### Permissions required

SNS needs permission to write to the DLQ (SQS resource policy):

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "sns.amazonaws.com" },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:us-east-1:123456789:MyDLQ",
  "Condition": {
    "ArnEquals": { "aws:SourceArn": "arn:aws:sns:us-east-1:123456789:MyTopic" }
  }
}
```

> **Exam distinction:** SQS DLQ = failed **processing**. SNS DLQ = failed **delivery**. Both are SQS queues, but they handle different failure modes.

---

## 12. SNS FIFO Topics

For scenarios requiring ordered, deduplicated fan-out.

### Properties

- Strict ordering within a **Message Group**
- Exactly-once delivery (5-minute deduplication window)
- **Only SQS FIFO queues** can subscribe
- Throughput: 300 msg/s (3,000 with batching), matching SQS FIFO

### Message Group ID & Deduplication

Works identically to SQS FIFO:

- `MessageGroupId` — groups messages for ordering
- `MessageDeduplicationId` or content-based deduplication — prevents duplicates

### Naming

Topic name must end with `.fifo`:

```
MyOrderTopic.fifo
```

### Use case

Ordered fan-out: e.g., stock price updates that must be delivered in order to multiple downstream systems.

```
Stock Price Publisher
        |
        v
SNS FIFO Topic (MyPrices.fifo)
        |
        |--- SQS FIFO Queue A.fifo --> Analytics Service
        |--- SQS FIFO Queue B.fifo --> Trading Engine
        |--- SQS FIFO Queue C.fifo --> Audit Logger
```

---

## 13. Security & Encryption

### Encryption In Transit

All SNS API calls and deliveries use **HTTPS/TLS** by default.

### Encryption At Rest (SSE)

SNS supports **Server-Side Encryption** using AWS KMS.

- Encrypts messages stored temporarily before delivery
- Set `KmsMasterKeyId` on the topic
- Uses the same CMK model as SQS

|Key|Description|
|---|---|
|`alias/aws/sns`|AWS-managed key (no extra cost)|
|Custom CMK|Full control, cost per API call|

### Cross-service encryption note

If an **encrypted SNS topic** fans out to an **encrypted SQS queue**, both must use KMS and the KMS key policy must allow both SNS and SQS to use it.

---

## 14. Access Control (IAM + Resource Policies)

### IAM Policies

Control which identities can perform SNS actions.

Common IAM actions:

- `sns:Publish`
- `sns:Subscribe`
- `sns:Unsubscribe`
- `sns:CreateTopic`
- `sns:DeleteTopic`
- `sns:ListTopics`
- `sns:GetTopicAttributes`
- `sns:SetTopicAttributes`

### SNS Resource Policy (Topic Policy)

Required for **cross-account access** or allowing **AWS services** to publish.

**Allow S3 to publish to SNS (S3 Event Notification):**

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "s3.amazonaws.com" },
  "Action": "sns:Publish",
  "Resource": "arn:aws:sns:us-east-1:123456789:MyTopic",
  "Condition": {
    "ArnLike": { "aws:SourceArn": "arn:aws:s3:::my-bucket" }
  }
}
```

**Allow another account to publish:**

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::999999999:root" },
  "Action": "sns:Publish",
  "Resource": "arn:aws:sns:us-east-1:123456789:MyTopic"
}
```

> **Rule:** Same account → IAM policy. Cross-account or AWS service → Topic Resource Policy.

---

## 15. Message Delivery Retries

When SNS fails to deliver to an endpoint, it retries with an **exponential backoff** policy.

### Default retry policy (HTTP/S endpoints)

|Phase|Retries|Delay|
|---|---|---|
|Immediate|3|Instant|
|Pre-backoff|2|1 second|
|Backoff|10|20s → 20 min (exponential)|
|Post-backoff|100,000|20 minutes|

### Retry behavior by protocol

|Protocol|Retries|Notes|
|---|---|---|
|HTTP/S|Yes (configurable)|Retry policy applies|
|Lambda|Yes|Lambda async retry (2 retries)|
|SQS|Yes|SQS is highly available; rare delivery failure|
|Email/SMS|Limited|Best-effort|

### After all retries fail

- If a **DLQ is configured** on the subscription → message lands in DLQ
- If no DLQ → **message is lost**

> **Exam tip:** To ensure no message is lost when a subscriber is down → attach an **SQS queue** as subscriber (or configure a DLQ on the subscription).

---

## 16. SNS vs SQS — Key Differences

|Property|SNS|SQS|
|---|---|---|
|Model|Pub/Sub (push)|Queue (pull)|
|Consumers|Many subscribers simultaneously|One consumer at a time per message|
|Message persistence|No (delivered immediately, then gone)|Yes (up to 14 days)|
|Message ordering|Best-effort (FIFO topic for strict)|Best-effort (FIFO queue for strict)|
|Delivery guarantee|At-least-once (Standard)|At-least-once (Standard)|
|Consumer model|Pushed to all subscribers|Polled by one consumer|
|Use case|Broadcast / fan-out / notify|Decoupling / buffering / async work|
|Failed delivery|Message lost (unless DLQ on subscription)|Message stays in queue until processed or expired|

**One-liner:** SNS broadcasts to many at once. SQS holds messages for one consumer to pick up.

---

## 17. Limits & Quotas Cheat Sheet

|Limit|Value|
|---|---|
|**Message size**|256 KB|
|**Topics per account**|100,000 (soft limit)|
|**Subscriptions per topic**|12,500,000 (Standard), 100 (FIFO)|
|**Pending subscriptions**|100 per topic|
|**Message attributes**|10 per message|
|**Filter policy size**|256 KB|
|**FIFO throughput**|300 msg/s (3,000 with batching)|
|**Standard throughput**|Unlimited|
|**Subscription confirmation expiry**|3 days|
|**FIFO deduplication window**|5 minutes|

---

## 18. Common Exam Scenarios & Traps

### Scenario 1: Fan-out to multiple consumers

**Requirement:** When an order is placed, the inventory service, billing service, and notification service must all react.

**Solution:** Publish to an **SNS topic** with each service subscribing via its own **SQS queue** (fan-out pattern).

---

### Scenario 2: Different messages to different subscribers

**Requirement:** The same SNS topic must send JSON to an SQS queue, a plain text email to the ops team, and a formatted SMS.

**Solution:** Use **per-protocol message structure** with `MessageStructure = "json"` and a JSON body with `"sqs"`, `"email"`, `"sms"`keys.

---

### Scenario 3: Only certain subscribers should get certain events

**Requirement:** The payments team only wants `eventType = PAYMENT_FAILED` messages; the shipping team only wants `eventType = ORDER_SHIPPED`.

**Solution:** Use **SNS Message Filtering** — each subscription gets a filter policy on the `eventType` attribute.

---

### Scenario 4: Messages being lost when Lambda subscriber is throttled

**Symptom:** SNS publishes to Lambda, Lambda is throttled, messages are lost.

**Solution:** Add an **SQS queue between SNS and Lambda** (SNS → SQS → Lambda). SQS buffers the messages during throttling.

---

### Scenario 5: Ordered fan-out

**Requirement:** Price updates must be delivered in order to all downstream services.

**Solution:** **SNS FIFO topic** fanning out to **SQS FIFO queues** (one per service).

---

### Scenario 6: S3 event triggers multiple services

**Requirement:** When a file is uploaded to S3, trigger a transcoding service, a metadata indexing service, and an audit logger.

**Solution:** S3 Event Notification → **SNS topic** → three **SQS queues** (one per service). Do NOT subscribe Lambda directly without a queue buffer.

---

### Scenario 7: Cross-account notification

**Requirement:** Account A's application publishes to Account B's SNS topic.

**Solution:** Add a **Topic Resource Policy** on Account B's SNS topic granting `sns:Publish` to Account A's IAM role ARN.

---

### Scenario 8: Ensure no message is lost for an HTTP subscriber that goes down

**Solution:** Configure a **DLQ (SQS queue)** on the SNS subscription. Failed deliveries land in the DLQ for later reprocessing.

---

### Scenario 9: Mobile push notifications

**Requirement:** Send push notifications to iOS and Android apps.

**Solution:** SNS **Mobile Push** with **Platform Applications** (APNS for iOS, FCM for Android). Publish to individual device endpoints or to a topic subscribed by many device endpoints.

---

### Scenario 10: Alerting on CloudWatch alarm

**Requirement:** Trigger an email and a Lambda function when a CloudWatch alarm fires.

**Solution:** CloudWatch Alarm → **SNS topic** → email subscription + Lambda subscription.

---

## 19. Quick-Reference Summary Table

|Feature|Standard Topic|FIFO Topic|
|---|---|---|
|Ordering|Best-effort|Strict (per group)|
|Delivery|At-least-once|Exactly-once|
|Throughput|Unlimited|300/s (3,000 batched)|
|Subscribers|SQS, Lambda, HTTP, Email, SMS, Kinesis Firehose|SQS FIFO only|
|Deduplication|No|Yes (5-min window)|
|Message filtering|Yes|Yes|
|DLQ support|Yes (per subscription)|Yes|
|Name suffix|None|Must end in `.fifo`|

---

## Key APIs to Know

|API|Description|
|---|---|
|`CreateTopic`|Create a new topic|
|`DeleteTopic`|Delete a topic and all its subscriptions|
|`Publish`|Send a message to a topic|
|`Subscribe`|Subscribe an endpoint to a topic|
|`Unsubscribe`|Remove a subscription|
|`ConfirmSubscription`|Confirm HTTP/email subscription|
|`ListTopics`|List all topics in the account|
|`ListSubscriptions`|List all subscriptions|
|`ListSubscriptionsByTopic`|List subscriptions for a specific topic|
|`GetTopicAttributes`|Get topic metadata and settings|
|`SetTopicAttributes`|Update topic settings|
|`SetSubscriptionAttributes`|Update subscription (e.g., filter policy, DLQ)|

---

## Architecture Pattern Cheat Sheet

```
[Basic notification]
CloudWatch Alarm --> SNS Topic --> Email + SMS + Lambda

[Fan-out (most common exam pattern)]
Event --> SNS Topic --> SQS Queue A --> Consumer A
                    --> SQS Queue B --> Consumer B
                    --> SQS Queue C --> Consumer C

[Resilient async processing]
Event --> SNS --> SQS --> Lambda --> DynamoDB
                  |
                 DLQ (failed messages)

[Ordered fan-out]
Event --> SNS FIFO Topic --> SQS FIFO Queue A --> Service A
                          --> SQS FIFO Queue B --> Service B

[S3 event fan-out]
S3 Upload --> S3 Event --> SNS Topic --> SQS (transcoder)
                                     --> SQS (indexer)
                                     --> SQS (auditor)

[Mobile push]
Backend --> SNS Topic --> Platform Application (APNS/FCM) --> Mobile Devices
```

---

> **Final exam advice:** SNS is almost always the right answer when you see "notify multiple systems", "fan-out", or "broadcast". Remember: SNS = push to many NOW; SQS = hold for one consumer to pull later. The golden combo is **SNS + SQS** — SNS for broadcast, SQS for durability and buffering. And when ordering matters end-to-end, go **SNS FIFO → SQS FIFO**.

---

_Guide covers the AWS Developer Associate (DVA-C02) exam objectives for Amazon SNS._
