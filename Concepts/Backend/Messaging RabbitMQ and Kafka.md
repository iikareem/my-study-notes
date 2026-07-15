---
tags:
  - backend
  - exactly-once
  - kafka
  - messaging
  - rabbitmq
---

# Messaging — RabbitMQ & Kafka

> Merged notes: RabbitMQ operational guidance + Kafka vs RabbitMQ comparison and Kafka deep dive.

## Related

- [[Backpressure]]
- [[HOL]]

---

# Part A — RabbitMQ Operational Guide

![[Pasted image 20260524153855.png]]### **Point 1: Keep Your Queue Short** — Detailed Breakdown

#### **What does "many messages in a queue" mean?**

This refers to the number of messages accumulating in a queue that haven't been consumed yet. If producers are sending 1,000 messages/sec but consumers can only process 500 messages/sec, the queue grows by 500 messages/sec. Over time, this backlog becomes a problem.

#### **The Prerequisites You Need:**

1. **Fast enough consumers** — Your consumer applications must be able to keep up with the producer rate. If they can't, messages pile up automatically.
2. **Monitoring in place** — You need visibility into queue depth (number of ready messages) so you know when it's growing.

#### **Why RAM usage becomes a problem:**

**In RAM** → Messages are indexed in memory, super fast to access (microseconds)  
**In Disk** → Messages must be read from disk first, then loaded into RAM (milliseconds)

When your queue has millions of messages, RabbitMQ can't hold them all in RAM. So it automatically pages out (flushes) messages to disk. But here's the catch: **paging is a blocking operation**. While RabbitMQ is writing thousands of messages to disk, the queue is locked and can't process new messages. This causes a sudden performance cliff.

#### **The restart nightmare:**

If you have a cluster with millions of messages on disk and you restart it:

- Each node must **rebuild its index** — reading all persisted messages from disk and reconstructing the in-memory index. With millions of messages, this can take hours.
- Each node must **sync with other nodes** to ensure all copies of messages are consistent. With millions of messages, this is very slow.

#### **When you CAN'T keep queues short:**

Sometimes queues grow because consumers legitimately can't keep up (processing complex jobs, external API latency). In this case, you use **lazy queues** or **TTL/max-length** (see points below).

![[Pasted image 20260524154048.png]]
### **Point 2: Use Quorum Queues** — Detailed Breakdown

#### **What is the difference?**

**Classic Queues** (the old way):

- One master queue, with optional async copies on other nodes
- If the master crashes before a copy is synced, messages are lost
- Failover is slow because there's no automatic detection

**Quorum Queues** (the new way):

- Uses the Raft consensus algorithm (same one used by Etcd, Consul)
- A message is only acknowledged when written to a **majority** of nodes
- If any node dies, others automatically take over
- Zero data loss guarantee

#### **Prerequisites:**

1. **At least 3 nodes in your cluster** — Quorum needs a majority. With 3 nodes, you need 2 to agree (can lose 1). With 5, you need 3 (can lose 2).
2. **RabbitMQ version 3.8 or newer** — Quorum Queues didn't exist before this.
3. **Accept slightly higher latency** — Quorum writes wait for majority acknowledgment, so they're inherently slower than classic queues. But it's a worthwhile trade-off for safety.

#### **Why migrate from classic?**

The text mentions "design flaws of classic mirrored queues." The main ones:

- Race conditions during failover where messages can be lost
- Slow detection of master failure
- No built-in consensus logic, leading to split-brain scenarios

Quorum Queues use a battle-tested algorithm (Raft), so these problems don't exist.

![[Pasted image 20260524154224.png]]
### **Point 3: Enable Lazy Queues** — Detailed Breakdown

#### **What is a lazy queue?**

A queue mode where messages are written to disk **immediately**, not kept in RAM until it overflows. Only actively delivered messages live in RAM.

#### **The key insight:**

**Eager queues** (default) assume: "Keep messages in the fast layer (RAM) as long as possible." **Lazy queues** assume: "Keep messages stable on disk from the start, read into RAM only when needed."

#### **Prerequisites:**

1. **RabbitMQ 3.6 or newer** — Lazy queues were introduced then.
2. **Accept lower throughput** — Disk I/O is slower than RAM operations.
3. **Fast disk storage** — Lazy queues are only predictable if your disk isn't the bottleneck.

#### **Why use lazy queues?**

You use them when **stability matters more than speed**. Examples:

- Batch jobs that arrive in bursts (1M messages in 1 second, then nothing for an hour)
- External API calls as consumers (slow and unpredictable latency)
- Long-running jobs where queue depth naturally grows

Without lazy queues, RabbitMQ in these scenarios would:

1. Fill RAM quickly with the burst
2. Trigger page-out at the worst moment
3. Block for 10+ seconds while flushing

With lazy queues, there's no surprise — messages go to disk, RAM stays small, performance is predictable.

#### **Important note (RabbitMQ 3.12+):**

Starting with 3.12, **all classic queues behave like lazy queues by default**. So if you're on 3.12+, you don't need to manually enable lazy queue mode anymore.

![[Pasted image 20260524154410.png]]
### **Point 4: TTL and Max-Length** — Detailed Breakdown

#### **What is TTL?**

TTL = Time To Live. A message parameter that says "this message is valid for X milliseconds, then delete it." If a consumer doesn't pick it up within that window, RabbitMQ discards it automatically.

#### **What is Max-Length?**

A hard cap on queue size. Once a queue reaches N messages, the oldest messages are discarded to make room for new ones (FIFO: first in, first out).

#### **Prerequisites:**

1. **Accept message loss** — Both TTL and max-length delete messages. Use only if losing old or excess messages is acceptable.
2. **Understand your access patterns** — For TTL, you need to know how long a message stays relevant. For max-length, you need to pick a number that makes sense for your use case.

#### **Why use them?**

**TTL is for time-sensitive data:**

- Stock price updates (old price is irrelevant)
- Notifications (a 1-hour-old alert is stale)
- Cache invalidation (if data isn't refreshed in X seconds, it's wrong)

**Max-length is for throughput protection:**

- Spike handling (app receives burst of messages, but consumers are slower)
- Memory-bounded systems (you need a hard guarantee that a queue never exceeds 50,000 messages)

#### **The key difference:**

- **TTL** = "Messages expire based on age"
- **Max-Length** = "Queue never grows beyond N, drop oldest when full"

You can use both together: "Keep messages for 30 seconds OR 10,000 items, whichever comes first."

![[Pasted image 20260524154557.png]]

### **Point 5: Number of Queues and Threading** — Detailed Breakdown

#### **The core issue:**

RabbitMQ queues are **single-threaded**. One queue = one thread = can only run on one CPU core at a time. Even if your machine has 16 cores, that queue will max out one core and leave the rest idle.

#### **The goal:**

Create enough queues so each core has its own queue to process. If you have 8 cores, aim for 8+ queues. Each queue runs on its own core in parallel → 8× throughput.

#### **Prerequisites:**

1. **Know your hardware** — Count the CPU cores on each RabbitMQ node.
2. **Design your queue topology** — Create roughly one queue per core.
3. **Distribute consumers** — Ensure your consumer applications can pull from all queues (don't create queues if consumers can't consume from them).

#### **The hidden cost: Management overhead**

The catch is that RabbitMQ's management plugin collects metrics for **every queue and consumer**. With thousands of queues, metric collection becomes a bottleneck itself. Your server spends so much CPU collecting statistics that it has little left for actual message processing.

**Rule of thumb:** Don't go beyond a few thousand queues per cluster. Beyond that, the overhead outweighs the benefits.

#### **How to handle more than cores?**

If you need more queues than cores (e.g., 100 logical queues but only 8 cores), consider:

- **Consistent hash exchange** or **RabbitMQ sharding** (from Point 1) to distribute messages across fewer physical queues
- **Hierarchical queue design** — multiple exchanges feeding into fewer core queues
---

# Part B — RabbitMQ vs Kafka & Kafka Deep Dive

### Kafka
- **Pull-based**: Consumers poll for messages, giving them control over consumption rate.
- **Message Retention**: Messages persist in topics for a configurable retention period (e.g., days or size-based), not deleted after consumption.
- **Offsets**: Consumers track their progress using offsets, stored in Kafka's `__consumer_offsets` topic. This allows replaying or resuming from any point.
- **Delivery Guarantees**:
  - **At-least-once** (default): If a consumer crashes before committing an offset, it may reprocess messages.
  - **Exactly-once**: Achievable with idempotent producers or Kafka transactions, but requires careful configuration (e.g., enabling `enable.idempotence=true` for producers or using transactional APIs).
- **Use Case**: Ideal for event streaming, log aggregation, and high-throughput systems where data replay or historical processing is valuable.

### RabbitMQ
- **Push-based**: Broker pushes messages to consumers, who process them from queues.
- **Message Removal**: Messages are deleted from the queue after a consumer acknowledges (acks) them.
- **Delivery Guarantees**:
  - **At-most-once** (auto-ack): Messages may be lost if the consumer crashes before processing.
  - **At-least-once** (manual ack): Duplicates possible if a consumer crashes before acknowledging.
  - **Exactly-once**: Not natively supported; requires idempotent consumers to handle potential duplicates.
- **Use Case**: Suited for task queues, work distribution, and request/response patterns where messages are transient.

### Key Differences
- **Architecture**: Kafka is a distributed log (event store) with persistent messages, while RabbitMQ is a traditional message queue with transient messages.
- **Message Handling**: Kafka allows replay and retention; RabbitMQ focuses on immediate delivery and removal.
- **Scalability**: Kafka excels in high-throughput, streaming scenarios; RabbitMQ is better for low-latency, task-based workflows.
- **Delivery Semantics**: Both default to at-least-once, but Kafka's exactly-once is more robust with transactions, while RabbitMQ relies on application-level idempotency.

### Additional Notes
- **Kafka** shines in big data pipelines (e.g., feeding data to Spark, Flink) due to its ability to handle massive event streams and replay.
- **RabbitMQ** is simpler for scenarios like microservices communication or job queues (e.g., Celery in Python).
- Neither provides true exactly-once delivery without extra effort, as distributed systems inherently face challenges like network failures or crashes.

If you have a specific use case or need a deeper dive into any aspect (e.g., setting up exactly-once in Kafka or RabbitMQ configuration), let me know!

I assume you're asking for additional information about Apache Kafka beyond what's already been provided in the comparison with RabbitMQ. Since you’ve shared a concise summary of Kafka’s key features, I’ll expand on those points with more details, clarify some aspects, and include additional relevant information about Kafka’s architecture, use cases, and features. If you meant something else by "kafka information" (e.g., a specific aspect like configuration, performance tuning, or a comparison with another system), please clarify, and I’ll tailor the response further.

### Apache Kafka: Detailed Information

**Apache Kafka** is an open-source distributed streaming platform designed for high-throughput, fault-tolerant, and scalable event streaming. It operates as a distributed commit log, storing and processing streams of records in real time. Below is a comprehensive overview of Kafka, building on the points you mentioned:

---

### Core Concepts
1. **Pull-based Model**:
   - Consumers actively poll Kafka brokers for messages from topics, allowing fine-grained control over consumption rate and load balancing.
   - This contrasts with push-based systems (like RabbitMQ), enabling consumers to process messages at their own pace, which is ideal for high-throughput scenarios.

2. **Message Retention**:
   - Messages are stored in **topics** for a configurable retention period (e.g., hours, days, or indefinitely), based on time (`log.retention.hours`) or size (`log.retention.bytes`).
   - Unlike traditional queues, messages aren’t deleted after consumption, enabling replayability and historical data access.
   - Example: A topic can retain data for 7 days, allowing consumers to reprocess events from any point within that period.

3. **Offsets**:
   - Each message in a topic has an **offset**, a unique identifier within a partition that acts like a bookmark.
   - Consumers track their progress by storing offsets, typically in Kafka’s internal `__consumer_offsets` topic.
   - Consumers can reset offsets to replay messages (e.g., `earliest` to start from the beginning or `latest` for new messages).

4. **Delivery Guarantees**:
   - **At-least-once** (default): Ensures no message loss, but a consumer crash before committing an offset may lead to reprocessing (duplicates).
   - **At-most-once**: Possible by disabling retries and auto-committing offsets, but risks message loss if a consumer fails.
   - **Exactly-once**: Supported since Kafka 0.11 (2017) via:
     - **Idempotent producers**: Ensures no duplicate writes using unique message IDs (`enable.idempotence=true`).
     - **Transactions**: Allows atomic writes across multiple partitions/topics, ensuring exactly-once delivery for producer-to-consumer workflows.
     - Use case: Critical applications like financial transactions where duplicates are unacceptable.
   - Exactly-once requires careful configuration (e.g., setting `transactional.id` for producers and enabling transactional consumers).

---

### Architecture
- **Topics**: Logical channels where messages are published. Topics are divided into **partitions** for parallelism and scalability.
- **Partitions**: Physical storage units distributed across brokers. Each partition is an ordered, immutable log of messages.
- **Brokers**: Servers in a Kafka cluster that store and manage data. Each broker holds a subset of partitions.
- **Producers**: Applications that publish messages to topics.
- **Consumers**: Applications that subscribe to topics and process messages. Consumers belong to **consumer groups** for load balancing and fault tolerance.
- **Consumer Groups**: Allow multiple consumers to share the load of processing messages from a topic. Each partition is assigned to one consumer in the group.
- **ZooKeeper/KAFT**:
   - Historically, Kafka relied on ZooKeeper for cluster coordination (e.g., leader election, metadata management).
   - Since Kafka 2.8 (2021), **KRaft** (Kafka Raft) mode replaces ZooKeeper for metadata management, simplifying deployment and improving scalability.
- **Replication**: Partitions are replicated across brokers for fault tolerance. The **replication factor** (e.g., 3) ensures data durability even if brokers fail.

---

### Key Features
1. **Scalability**:
   - Kafka scales horizontally by adding brokers or partitions, handling millions of messages per second.
   - Partitions enable parallel processing, making it suitable for big data workloads.

2. **Fault Tolerance**:
   - Replicated partitions ensure data availability despite broker failures.
   - Leader-follower replication ensures high availability, with followers syncing data from the leader.

3. **High Throughput**:
   - Optimized for sequential disk I/O and batch processing, Kafka achieves low-latency and high-throughput performance.
   - Compression (e.g., gzip, snappy) reduces network overhead.

4. **Kafka Streams and Connect**:
   - **Kafka Streams**: A Java library for building real-time stream processing applications directly on Kafka topics (e.g., filtering, aggregating data).
   - **Kafka Connect**: A framework for integrating Kafka with external systems (e.g., databases, S3) via connectors.

5. **Schema Registry** (often used with Kafka):
   - Manages message schemas (e.g., Avro, JSON) to ensure compatibility and evolution of data formats.
   - Commonly used in production to maintain data consistency.

---

### Use Cases
Kafka excels in scenarios requiring high-throughput, persistent, and scalable event streaming:
- **Event Streaming**: Real-time data pipelines for analytics (e.g., feeding data to Apache Spark, Flink, or Elasticsearch).
- **Log Aggregation**: Centralizing logs from multiple services for monitoring and analysis.
- **Event Sourcing**: Storing a sequence of events to reconstruct application state (e.g., in microservices).
- **Change Data Capture (CDC)**: Capturing database changes and streaming them to other systems.
- **IoT/Telemetry**: Handling massive streams of sensor data.
- **Message Broker**: Replacing traditional message queues in some cases, though it’s not a direct replacement for RabbitMQ in task-queue scenarios.

---

### Strengths
- **High Throughput**: Handles millions of messages per second with low latency.
- **Durability**: Persistent storage ensures data isn’t lost even after consumption.
- **Replayability**: Consumers can reprocess historical data by resetting offsets.
- **Ecosystem**: Integrates with tools like Kafka Streams, Kafka Connect, and Confluent’s Schema Registry.

---

### Limitations
- **Complexity**: Steeper learning curve than traditional message queues like RabbitMQ due to its distributed nature and configuration options.
- **Operational Overhead**: Managing a Kafka cluster (brokers, ZooKeeper/KRaft, partitions) requires expertise, especially at scale.
- **Not Ideal for Short-Lived Tasks**: Kafka’s persistent model is overkill for simple task queues or request/response patterns where RabbitMQ shines.
- **Latency**: While low, Kafka’s batch-oriented design may introduce slight delays compared to RabbitMQ for low-latency use cases.

---

### Comparison with RabbitMQ (Recap)
As you noted:
- **Kafka**: Pull-based, persistent log, consumer-managed offsets. Ideal for streaming, replay, and high-throughput use cases.
- **RabbitMQ**: Push-based, transient queues, broker-managed delivery. Suited for task distribution and low-latency workflows.
- **Delivery Guarantees**: Both default to at-least-once. Kafka’s exactly-once is more robust with transactions, while RabbitMQ requires idempotent consumers.
- **Use Case Fit**: Kafka for event-driven architectures and big data; RabbitMQ for job queues and microservices communication.

---

### Additional Details
- **Performance Tuning**:
  - Increase partitions for parallelism, but too many partitions can degrade performance.
  - Tune `num.io.threads` and `num.network.threads` on brokers for better throughput.
  - Use compression (`compression.type`) to reduce network load.
- **Security**:
  - Supports SSL/TLS for encryption, SASL for authentication, and ACLs for authorization.
- **Monitoring**:
  - Tools like Confluent Control Center, Prometheus, or Grafana are used to monitor cluster health, lag, and throughput.
- **Deployment**:
  - Can be deployed on-premises, in the cloud, or via managed services (e.g., Confluent Cloud, AWS MSK).
  - KRaft mode simplifies single-node setups for development.

---

### Example Configuration (Basic)
Here’s a simple Kafka producer configuration in Java to illustrate setup:
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all"); // Ensure durability
props.put("enable.idempotence", "true"); // For exactly-once
KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("my-topic", "key", "value"));
```

---

If you need specific details (e.g., Kafka configuration, performance optimization, or a deeper comparison with another system), or if you want me to generate a chart (e.g., comparing Kafka and RabbitMQ throughput), let me know! For chart generation, I’d need specific data points, as I don’t assume numbers. Also, if you want me to analyze a Kafka-related post or profile from X or search for real-time info, I can do that too.

Kafka

Pull-based → Consumers request messages.

Messages stay in Kafka for a retention period (not deleted after being read).

Consumers track progress using offsets (like a bookmark).

Offsets are usually stored in Kafka (__consumer_offsets topic).

At-least-once delivery by default → if a consumer crashes before committing offset, it may reprocess the same message.

Exactly-once is possible, but requires special setup (idempotent processing or transactions).

🔹 RabbitMQ

Push-based → Broker pushes messages into consumer queues.

Messages are removed after consumer sends an acknowledgment (ack).

Delivery guarantees:

Auto-ack → at-most-once (risk of loss if consumer crashes).

Manual ack → at-least-once (risk of duplicates if consumer crashes before ack).

Exactly-once is not guaranteed → you need idempotent consumers to handle duplicates safely.

🔹 Key Difference

Kafka: log of events + consumer-managed offsets → great for replay, streaming, high throughput.

RabbitMQ: queue with ack → great for task distribution, job queues, request/response.

Both are at-least-once by default; neither is truly exactly-once without extra effort.

## See also

- [[Message Brokers Guide — RabbitMQ & Kafka]] — competing consumers, exchanges, scaling, prefetch
- [[System Design MOC]] · [[Realtime Updates]]
