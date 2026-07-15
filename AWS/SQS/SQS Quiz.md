---
tags:
  - aws
  - certification
  - messaging
  - practice
  - sqs
---

**Hub:** [[AWS MOC]] · **Role:** Quiz
**Also:** [[SQS]]

# AWS SQS – Developer Associate Exam Practice

> **Instructions:** Read each scenario carefully and choose the best answer.  
> Answers are provided at the bottom of the document.

---

## Questions

---

**Question 1**

A company runs an e-commerce platform. When a customer places an order, a Lambda function processes the order and sends a confirmation email. During a flash sale, thousands of orders arrive simultaneously and the Lambda function starts throwing throttling errors.

Which SQS configuration change will **best** protect the Lambda function from being overwhelmed?

- A) Switch from a Standard Queue to a FIFO Queue
- B) Enable Long Polling on the queue
- C) Set a **concurrency limit** on Lambda and let SQS buffer the messages
- D) Increase the **Message Retention Period** to 14 days

---

**Question 2**

A developer is building an order-processing system. Orders from the same customer **must be processed in the exact sequence they were received**, and duplicate processing must be avoided.

Which queue type and feature should the developer use?

- A) Standard Queue with Dead Letter Queue enabled
- B) FIFO Queue with Content-Based Deduplication enabled
- C) Standard Queue with Long Polling enabled
- D) FIFO Queue with a Visibility Timeout of 0 seconds

---

**Question 3**

An application reads messages from an SQS queue and processes them. Occasionally, a message causes the consumer to crash before it can delete the message. After recovery, the consumer processes the same message again, causing duplicate side effects in the database.

What is the **most appropriate** solution?

- A) Decrease the Visibility Timeout to 0
- B) Enable a Dead Letter Queue and set `maxReceiveCount` to 1
- C) Make the consumer **idempotent** so processing the same message twice has no additional effect
- D) Switch to a FIFO Queue

---

**Question 4**

A media company uploads large video files to S3. After each upload, a message is sent to SQS containing the S3 object key. A fleet of EC2 workers polls the queue and transcodes the videos. Some videos take up to 45 minutes to transcode. Workers are frequently picking up messages that are already being processed by another worker.

What should the developer change?

- A) Enable Long Polling with a wait time of 20 seconds
- B) Increase the **Visibility Timeout** to be greater than the maximum processing time (e.g., 3600 seconds)
- C) Enable FIFO ordering to prevent duplicate processing
- D) Set the **Message Retention Period** to 1 minute

---

**Question 5**

A developer notices that their SQS consumers are making hundreds of empty API calls per second because the queue is usually empty during off-peak hours. This is increasing costs unnecessarily.

What is the **cheapest** fix?

- A) Increase the number of consumer instances
- B) Enable **Long Polling** by setting `WaitTimeSeconds` to 20
- C) Enable **Short Polling** with a 1-second delay
- D) Move to a FIFO queue

---

**Question 6**

An application sends 300 KB messages to SQS, but SQS has a maximum message size of 256 KB.

What is the **recommended AWS approach** to handle this?

- A) Split the message into multiple smaller messages
- B) Use the **SQS Extended Client Library** to store the payload in S3 and send a reference in the SQS message
- C) Compress the message and send it as a Base64-encoded string
- D) Use SNS instead of SQS for large payloads

---

**Question 7**

A payment service sends messages to SQS. A downstream consumer fails to process a message after **5 attempts**. The team wants to investigate these problematic messages without losing them and without blocking the queue.

Which feature should they configure?

- A) Increase the Visibility Timeout
- B) Enable a **Dead Letter Queue (DLQ)** and set `maxReceiveCount` to 5
- C) Enable FIFO ordering
- D) Delete the messages manually after each failure

---

**Question 8**

A developer is designing a fanout architecture. When a new user registers, the system must:

- Send a welcome email
- Create a user profile in the database
- Log the event to an analytics service

All three actions should happen **independently and in parallel**.

What is the **best** AWS architecture?

- A) Three separate SQS queues, each polled by a different service
- B) One SQS FIFO queue with three consumer groups
- C) **SNS topic** that fans out to three separate SQS queues (SNS → SQS × 3)
- D) EventBridge with a single SQS queue

---

**Question 9**

A company processes financial transactions using SQS. A message is picked up by a consumer and the consumer crashes mid-process after 30 seconds. The Visibility Timeout is set to **20 seconds**.

What happens to the message?

- A) The message is permanently deleted from the queue
- B) The message is moved to the Dead Letter Queue immediately
- C) The message becomes **visible again** in the queue after 20 seconds and can be picked up by another consumer
- D) The message stays invisible until manually released

---

**Question 10**

A developer wants to delay the delivery of messages in SQS for exactly **10 minutes** after they are sent, so downstream consumers only receive them after that window.

Which feature supports this?

- A) Visibility Timeout set to 600 seconds
- B) Message Retention Period set to 600 seconds
- C) **Delay Queue** with `DelaySeconds` set to 600
- D) Long Polling with `WaitTimeSeconds` set to 600

---

**Question 11**

A team uses a FIFO queue. A producer sends the following messages with the same `MessageGroupId`:

```
MSG-1 → MSG-2 → MSG-3
```

MSG-2 fails processing and is retried multiple times. What happens to MSG-3?

- A) MSG-3 is processed immediately regardless of MSG-2
- B) MSG-3 is automatically moved to the DLQ
- C) MSG-3 is **blocked** until MSG-2 is either successfully processed or moved to the DLQ
- D) MSG-3 is discarded after MSG-2 fails

---

**Question 12**

An application sends duplicate messages to an SQS FIFO queue due to a network retry. The developer wants to ensure **exactly-once delivery** without writing custom deduplication logic.

Which feature should be enabled?

- A) Long Polling
- B) **Content-Based Deduplication**
- C) Dead Letter Queue with `maxReceiveCount` = 1
- D) Visibility Timeout set to 0

---

## ✅ Answers

---

| #   | Answer | Explanation                                                                                                                                                                          |
| --- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | **C**  | SQS acts as a buffer. Setting a concurrency limit on Lambda prevents throttling while SQS holds the backlog. FIFO adds overhead and doesn't solve the rate issue.                    |
| 2   | **B**  | FIFO guarantees ordering per `MessageGroupId`. Content-Based Deduplication prevents duplicates using a SHA-256 hash of the body.                                                     |
| 3   | **C**  | Making the consumer idempotent is the **correct architectural pattern**. A DLQ with `maxReceiveCount=1` would just discard the message after one failure — not safe for payments.    |
| 4   | **B**  | The Visibility Timeout must be **longer than the processing time**. If a worker takes 45 min, set Visibility Timeout to ≥ 3600 seconds to prevent another worker from picking it up. |
| 5   | **B**  | **Long Polling** (up to 20 sec) waits for a message to arrive before returning, drastically reducing empty responses and API call costs.                                             |
| 6   | **B**  | The **SQS Extended Client Library** (Java) or equivalent pattern stores the payload in S3 and puts a pointer in the SQS message, handling payloads beyond 256 KB.                    |
| 7   | **B**  | A **Dead Letter Queue** with `maxReceiveCount=5` automatically moves messages that fail 5 times, allowing debugging without blocking normal processing.                              |
| 8   | **C**  | **SNS → SQS fanout** is the canonical pattern. SNS publishes once; each SQS queue delivers independently to its consumer. This decouples all three services.                         |
| 9   | **C**  | Because the Visibility Timeout (20s) expired before the consumer finished (30s), the message **reappears** in the queue and can be reprocessed.                                      |
| 10  | **C**  | **Delay Queues** postpone delivery of new messages. `DelaySeconds` (max 900s = 15 min) is set at queue or message level. Visibility Timeout only hides already-received messages.    |
| 11  | **C**  | In FIFO queues, messages within the **same MessageGroupId are strictly ordered**. A stuck message blocks all subsequent messages in that group.                                      |
| 12  | **B**  | **Content-Based Deduplication** hashes the message body and ignores duplicates sent within the 5-minute deduplication window — no custom code needed.                                |

---

> 💡 **Scoring Guide**
> 
> - 11–12 correct → Excellent — you're exam-ready on SQS!
> - 8–10 correct → Good — review the topics you missed
> - Below 8 → Revisit SQS core concepts: visibility timeout, FIFO vs Standard, DLQ, and polling types
