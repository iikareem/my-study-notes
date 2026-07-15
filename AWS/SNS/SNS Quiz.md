---
tags:
  - aws
  - certification
  - messaging
  - practice
  - sns
---

**Hub:** [[AWS MOC]] · **Role:** Quiz
**Also:** [[SNS]]

# AWS SNS – Developer Associate Exam Practice

> **Instructions:** Read each scenario carefully and choose the best answer.  
> Answers are provided at the bottom of the document.

---

## Questions

---

**Question 1**

A company wants to send a notification to **multiple systems simultaneously** when a new order is placed: an inventory service, a shipping service, and an analytics service. Each service must receive every message independently.

What is the **best** AWS architecture?

- A) One SQS queue shared by all three services
- B) One SNS topic with three SQS queues subscribed to it
- C) Three separate Lambda functions polling one SQS queue
- D) EventBridge rule triggering one SQS queue

---

**Question 2**

A developer publishes a message to an SNS topic. The topic has the following subscribers:

- An SQS queue
- A Lambda function
- An HTTP endpoint

The HTTP endpoint is **temporarily unavailable**. What happens to the message delivery for the HTTP endpoint?

- A) The message is lost immediately
- B) SNS retries delivery to the HTTP endpoint using a **retry policy with exponential backoff**
- C) SNS moves the message to a Dead Letter Queue automatically
- D) SNS pauses delivery to all subscribers until the HTTP endpoint recovers

---

**Question 3**

A retail company uses an SNS topic to notify subscribers about product discounts. They want **only the subscribers interested in "Electronics"** to receive discount messages for that category, without creating separate topics per category.

Which SNS feature should they use?

- A) SNS FIFO Topic
- B) SNS **Message Filtering** using subscription filter policies
- C) SNS Message Attributes with a Lambda transformer
- D) A separate SQS queue per category

---

**Question 4**

A developer is building a payment system. Payment events must be delivered to downstream services in **strict order** and **without duplicates**.

Which SNS configuration is appropriate?

- A) Standard SNS Topic with SQS Standard Queue
- B) Standard SNS Topic with SQS FIFO Queue
- C) **SNS FIFO Topic** subscribed to an **SQS FIFO Queue**
- D) SNS Standard Topic with Lambda and deduplication logic

---

**Question 5**

A security team wants to receive **SMS alerts** on their phones whenever a critical CloudWatch alarm is triggered (e.g., CPU > 90% for 5 minutes).

What is the **simplest** architecture?

- A) CloudWatch → SQS → Lambda → SNS SMS
- B) CloudWatch → EventBridge → SQS → SMS gateway
- C) **CloudWatch Alarm** action set to publish to an **SNS topic** with an SMS subscription
- D) CloudWatch → Lambda → Twilio API

---

**Question 6**

A developer publishes a message to an SNS topic that has an SQS queue as a subscriber. When the message arrives in SQS, the consumer notices the message body is **wrapped in extra JSON metadata** (e.g., `"Message"`, `"Subject"`, `"Timestamp"` fields) instead of the raw payload.

How can the developer receive **only the original message body** in SQS?

- A) Set the SQS message format to "raw" in queue settings
- B) Enable **Raw Message Delivery** on the SNS subscription
- C) Use a Lambda function to strip the metadata before sending to SQS
- D) Publish using the `MessageStructure` parameter set to "json"

---

**Question 7**

A developer wants to send **different message content** to different subscriber types (email subscribers get a human-readable summary, SQS subscribers get JSON) from the **same SNS publish call**.

Which feature enables this?

- A) SNS Message Filtering
- B) SNS Subscription Attributes
- C) **SNS Message Structure** with `MessageStructure=json` and per-protocol payloads
- D) SNS FIFO with per-group formatting

---

**Question 8**

An application publishes messages to an SNS topic. A subscribed Lambda function intermittently fails to process messages. The team wants to ensure **failed messages are not silently lost** and can be investigated later.

What should the developer configure?

- A) Enable Long Polling on the SNS topic
- B) Add a **Dead Letter Queue (DLQ)** to the Lambda function's SNS event source mapping
- C) Increase the SNS topic's message retention period
- D) Add a retry policy to the SNS topic subscription

---

**Question 9**

A company has a multi-region setup. They publish events from `us-east-1` to an SNS topic and want a Lambda function in `eu-west-1` to be triggered by those events.

What is the **correct approach**?

- A) SNS topics natively support cross-region Lambda triggers
- B) Use SNS topic in `us-east-1` → SQS queue in `eu-west-1` via cross-region subscription → Lambda trigger
- C) Replicate the SNS topic to `eu-west-1` using S3 replication
- D) Use **EventBridge** with a cross-region event bus rule to trigger the Lambda in `eu-west-1`

---

**Question 10**

A developer publishes a 300 KB JSON payload to an SNS topic. The publish call fails.

What is the **most likely reason**?

- A) SNS does not support JSON payloads
- B) The SQS subscriber queue is full
- C) SNS has a **maximum message size of 256 KB** and the payload exceeds it
- D) The IAM role does not have `sns:Publish` permission

---

**Question 11**

A company uses SNS to send promotional emails to 500,000 subscribers. They notice that some subscribers are receiving **duplicate emails** occasionally.

Which characteristic of SNS explains this behavior?

- A) SNS FIFO does not support email subscriptions
- B) SNS Standard topics provide **at-least-once delivery**, which can result in duplicates
- C) The email protocol does not support deduplication IDs
- D) The SNS topic has Content-Based Deduplication disabled

---

**Question 12**

A developer needs to **fan out** S3 event notifications to both an SQS queue and a Lambda function whenever a new object is uploaded to a bucket. S3 event notifications only support **one destination** per event type.

What is the correct architecture?

- A) Configure two separate S3 event notification rules for the same event type
- B) Use S3 → Lambda → manually forward to SQS
- C) **S3 → SNS topic** → SNS fans out to both SQS queue and Lambda function
- D) Use EventBridge with two targets configured for the S3 event

---

## ✅ Answers

---

| #   | Answer | Explanation                                                                                                                                                                                                                                      |
| --- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | **B**  | SNS **fanout pattern** — one publish reaches all subscribers independently. A shared SQS queue means only one consumer gets each message, not all three.                                                                                         |
| 2   | **B**  | SNS uses a **managed retry policy with exponential backoff** for HTTP/HTTPS endpoints (up to 23 retries over hours). SQS and Lambda retries are handled differently. Only HTTP/S gets this automatic retry.                                      |
| 3   | **B**  | **Subscription Filter Policies** let each subscriber declare which messages it wants based on message attributes. No need for separate topics.                                                                                                   |
| 4   | **C**  | **SNS FIFO → SQS FIFO** is the only combination that guarantees strict ordering and exactly-once delivery end-to-end. Standard topics/queues provide best-effort ordering only.                                                                  |
| 5   | **C**  | CloudWatch Alarms can directly trigger an SNS topic action. Adding an SMS subscription is the simplest and most native path — no Lambda or third-party needed.                                                                                   |
| 6   | **B**  | **Raw Message Delivery** on the SNS-to-SQS subscription strips the SNS envelope and delivers only the original message body to the SQS queue.                                                                                                    |
| 7   | **C**  | Setting `MessageStructure=json` allows the developer to provide a different message string per protocol (e.g., `"sqs"`, `"email"`, `"lambda"`) in a single publish call.                                                                         |
| 8   | **B**  | A **DLQ on the Lambda function** captures events that fail after all retries are exhausted. SNS itself doesn't have a retention period — messages not delivered are gone.                                                                        |
| 9   | **D**  | **EventBridge** is the correct service for cross-region event routing. SNS cross-region subscriptions to Lambda are not natively supported in the same way. EventBridge supports cross-region event buses.                                       |
| 10  | **C**  | SNS enforces a hard **256 KB message size limit**. For larger payloads, use the S3 + SNS pointer pattern (store payload in S3, send the S3 reference via SNS).                                                                                   |
| 11  | **B**  | SNS Standard topics guarantee **at-least-once delivery** — duplicates are possible by design. If exactly-once is needed, use FIFO topics (which only support SQS FIFO and Lambda subscribers, not email).                                        |
| 12  | **C**  | **S3 → SNS → fanout** is the canonical solution. S3 publishes one event to SNS, and SNS fans it out to both SQS and Lambda simultaneously. EventBridge (option D) also works but SNS fanout is the more classic DVA-C02 answer for this pattern. |

---

> 💡 **Scoring Guide**
> 
> - 11–12 correct → Excellent — SNS is locked in!
> - 8–10 correct → Good — revisit filtering, FIFO, and delivery retries
> - Below 8 → Review SNS core concepts: fanout, filter policies, raw delivery, FIFO vs Standard, and DLQ behavior
