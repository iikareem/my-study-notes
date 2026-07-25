---
tags:
  - system-design
  - queues
  - async
  - workers
---

# Long Running Tasks — Full System Design Guide

## Table of Contents

1. [What Are Long Running Tasks?](#1-what-are-long-running-tasks)
2. [Why Synchronous Processing Fails](#2-why-synchronous-processing-fails)
3. [The Async Job Queue Pattern](#3-the-async-job-queue-pattern)
4. [Core Components Deep Dive](#4-core-components-deep-dive)
5. [Queue Technologies Compared](#5-queue-technologies-compared)
6. [Worker Design](#6-worker-design)
7. [Status Tracking & Delivery Patterns](#7-status-tracking--delivery-patterns)
8. [Handling Failures](#8-handling-failures)
9. [Multi-Step Workflows & Orchestration](#9-multi-step-workflows--orchestration)
10. [Scaling the System](#10-scaling-the-system)
11. [Observability & Monitoring](#11-observability--monitoring)
12. [Real-World Example: Video Processing Pipeline](#12-real-world-example-video-processing-pipeline)
13. [Interview Cheat Sheet](#13-interview-cheat-sheet)

---

## 1. What Are Long Running Tasks?

A **long running task** is any operation that takes longer than a user is willing to wait for a synchronous HTTP response — typically anything over a few seconds.

### Common Examples

|Domain|Example Tasks|
|---|---|
|Media|Video transcoding, image resizing, PDF generation|
|Data|Report generation, CSV exports, ETL pipelines|
|AI/ML|Model inference, batch predictions, fine-tuning jobs|
|Communication|Sending bulk emails/SMS, push notifications|
|Finance|End-of-day reconciliation, invoice generation|
|DevOps|CI/CD builds, deployments, database migrations|

### The Rule of Thumb

> If an operation takes **more than ~2 seconds** or its duration is **unpredictable**, it should be handled asynchronously.

---

## 2. Why Synchronous Processing Fails

Understanding the failure modes is essential before jumping to solutions.

### Problem 1: HTTP Timeout

Web clients and proxies have timeout limits (commonly 30–60 seconds). A long operation simply gets cut off.

```
Client ──── POST /export ────► Server
                                 │ (processing for 5 minutes)
Client ◄── 504 Gateway Timeout ──┘
```

### Problem 2: Blocked Web Server Threads

Web servers have a finite thread/connection pool. A slow request holds a thread for its entire duration.

```
Thread Pool (8 threads)

Thread 1: → Processing video (10 min) ← BLOCKED
Thread 2: → Processing video (10 min) ← BLOCKED
Thread 3: → Processing video (10 min) ← BLOCKED
...
Thread 8: → Processing video (10 min) ← BLOCKED

New Request: ← REJECTED (no available threads)
```

Even 8 concurrent heavy requests can bring down a server entirely.

### Problem 3: No Retry on Failure

If the server crashes mid-operation, the work is silently lost. There is no way to resume or retry.

### Problem 4: No Progress Visibility

The user sees a spinner with no feedback. They cannot tell if the system is working, stuck, or dead.

### Problem 5: Resource Contention

CPU/memory-intensive jobs compete with fast API requests for the same resources, degrading the experience for all users.

---

## 3. The Async Job Queue Pattern

The universal solution is to **decouple request acceptance from request processing**.

### The Flow

```
                        ┌─────────────────────────────────────────────┐
                        │                                             │
  Client                │   API Server         Queue        Worker   │
    │                   │       │                │              │     │
    │  POST /jobs       │       │                │              │     │
    │──────────────────►│       │                │              │     │
    │                   │       │  Enqueue job   │              │     │
    │                   │       │───────────────►│              │     │
    │  202 + { jobId }  │       │                │  Dequeue job │     │
    │◄──────────────────│       │                │─────────────►│     │
    │                   │       │                │              │     │
    │  GET /jobs/:id    │       │                │   (working)  │     │
    │──────────────────►│       │                │              │     │
    │  { status: "processing" } │                │              │     │
    │◄──────────────────│       │                │              │     │
    │                   │       │                │   Complete   │     │
    │  GET /jobs/:id    │       │                │◄─────────────│     │
    │──────────────────►│       │                │              │     │
    │  { status: "done", result: "..." }         │              │     │
    │◄──────────────────│       │                │              │     │
                        └─────────────────────────────────────────────┘
```

### The Three Golden Steps

1. **Accept** — Validate the request and immediately return `202 Accepted` with a `jobId`
2. **Enqueue** — Persist the job to a durable queue
3. **Process** — A separate worker picks it up, executes it, and updates the job status

### Why This Works

- The API server is never blocked — it just writes to a queue and returns
- Workers can run on separate machines, scaled independently
- If a worker crashes, the job remains in the queue and can be retried
- The user has a `jobId` to check status or receive a callback

---

## 4. Core Components Deep Dive

### 4.1 The API Layer

Responsibilities:

- Validate the incoming request (auth, input sanity)
- Create a job record in the database with status `pending`
- Push the job ID onto the queue
- Return `202 Accepted` with the job ID

```
POST /videos/transcode
{
  "source_url": "s3://bucket/input.mp4",
  "format": "hls",
  "notify_url": "https://yourapp.com/webhook"
}

→ 202 Accepted
{
  "job_id": "job_abc123",
  "status": "pending",
  "status_url": "https://api.example.com/jobs/job_abc123"
}
```

### 4.2 The Job Store (Database)

A persistent record of every job and its current state.

```
jobs table
──────────────────────────────────────────────────────
id          | job_abc123
type        | video_transcode
status      | processing          ← pending → processing → done / failed
payload     | { source_url, format, notify_url }
result      | null                ← filled when done
error       | null                ← filled on failure
created_at  | 2024-01-15 10:00:00
updated_at  | 2024-01-15 10:00:05
started_at  | 2024-01-15 10:00:05
completed_at| null
retries     | 0
```

### 4.3 The Queue

A buffer between producers (API servers) and consumers (workers). The queue holds job IDs (not the full payload — the worker looks that up from the DB).

```
Queue
┌──────────────────────────────────┐
│  job_abc123  │  job_def456  │ … │
└──────────────────────────────────┘
  ▲ Enqueue                   Dequeue ▼
  API Server                   Worker
```

**Why store only the job ID in the queue?**

- Queue messages have size limits (SQS: 256KB)
- Payload changes (e.g. cancellation) are reflected immediately from the DB
- Simpler re-enqueue logic on retry

### 4.4 The Worker

A long-running process that:

1. Polls or subscribes to the queue
2. Locks/acknowledges the message (prevents double processing)
3. Fetches the full job payload from the DB
4. Executes the actual work
5. Updates job status to `done` or `failed`
6. Sends a callback if configured

Workers are stateless — they can be scaled horizontally without coordination.

---

## 5. Queue Technologies Compared

### Redis (BullMQ / Sidekiq)

```
Pros:
✓ Simple to set up and operate
✓ Very fast (in-memory)
✓ Rich job lifecycle: delay, priority, rate limiting, concurrency
✓ Good dashboards (Bull Board, Sidekiq Web)

Cons:
✗ Data loss risk if Redis crashes without persistence configured
✗ Limited message retention/replay
✗ Not ideal for very high throughput (millions/sec)

Best for: Startups, internal tools, moderate traffic, where simplicity matters
```

### Amazon SQS

```
Pros:
✓ Fully managed — no infrastructure to run
✓ At-least-once delivery guaranteed
✓ Dead-letter queues (DLQ) built in
✓ Visibility timeout prevents double processing
✓ Scales automatically

Cons:
✗ No replay — once consumed, messages are gone
✗ At-least-once means workers must be idempotent
✗ 256KB message size limit
✗ No ordering guarantees (standard queue)

Best for: Cloud-native AWS workloads, production systems needing managed reliability
```

### Apache Kafka

```
Pros:
✓ Durable log — messages retained for configurable time (days/weeks)
✓ Consumers can replay from any offset
✓ Very high throughput (millions of messages/sec)
✓ Multiple consumer groups can independently read the same stream
✓ Ordered within a partition

Cons:
✗ Complex to operate (ZooKeeper / KRaft, brokers, replication)
✗ Operational overhead is significant
✗ Overkill for simple job queues

Best for: High-volume event pipelines, audit logs, fan-out to multiple consumers, replay requirements
```

### Quick Decision Guide

```
Simple job queue, small team, fast iteration → Redis (BullMQ)
Production cloud workload, reliability needed → SQS
High volume, replay needed, multiple consumers → Kafka
```

---

## 6. Worker Design

### 6.1 The Basic Worker Loop

```
while true:
    job = queue.dequeue(visibility_timeout=30s)
    if no job: sleep(1s); continue

    db.update(job.id, status="processing", started_at=now())
    try:
        result = execute(job.payload)
        db.update(job.id, status="done", result=result, completed_at=now())
        send_callback(job.notify_url, job.id, "done")
    except Exception as e:
        db.update(job.id, status="failed", error=str(e))
        if job.retries < MAX_RETRIES:
            queue.enqueue(job.id, delay=backoff(job.retries))
            db.update(job.id, retries=job.retries+1)
        else:
            db.update(job.id, status="dead")
            queue.send_to_dlq(job.id)
```

### 6.2 Idempotency

Workers **must** be idempotent — running the same job twice should produce the same result without side effects.

Why? At-least-once delivery queues (SQS, Kafka) can deliver the same message more than once (network glitches, timeouts, etc.).

Strategies:

- Use the `job_id` as an idempotency key for downstream operations
- Check `status` before processing: skip if already `done`
- Use database transactions to atomically check-and-update

### 6.3 Visibility Timeout / Message Locking

When a worker dequeues a message, the queue hides it from other workers for a configurable window (e.g. 30 seconds). If the worker doesn't acknowledge it in time, the message becomes visible again and another worker picks it up.

```
Worker A dequeues job_abc123
  ↓ (visibility timeout: 30s)
  ↓ (worker extends timeout every 25s while working)
  ↓ (job completes)
Worker A deletes message from queue ← critical step
```

**Always delete the message after successful processing.** Failing to do so causes the job to run again after the timeout expires.

### 6.4 Graceful Shutdown

Workers must handle `SIGTERM` gracefully: finish the current job, then exit.

```
on SIGTERM:
    stop_accepting_new_jobs = true
    wait_for_current_job_to_finish()
    exit(0)
```

This is critical in containerized environments (Kubernetes rolling deploys, ECS task replacements).

---

## 7. Status Tracking & Delivery Patterns

Once a job is submitted, the client needs to know when it finishes. Three patterns:

### Pattern 1: Polling

The client repeatedly calls `GET /jobs/:id` until the status changes.

```
Client                          API Server
  │  GET /jobs/job_abc123           │
  │─────────────────────────────►  │
  │  { status: "processing" }       │
  │◄─────────────────────────────  │
  │  (wait 5 seconds)               │
  │  GET /jobs/job_abc123           │
  │─────────────────────────────►  │
  │  { status: "done", result: … }  │
  │◄─────────────────────────────  │
```

**Pros:** Simple, works everywhere, no inbound requirements for the client  
**Cons:** Wasteful (many empty responses), adds latency (up to 1 poll interval)  
**Use when:** Client is a browser or third party that can't receive webhooks

Implement with **exponential backoff**: poll at 1s, 2s, 4s, 8s… to avoid hammering the server.

---

### Pattern 2: Webhooks / Callbacks

The client provides a `notify_url` when submitting the job. The server POSTs to it on completion.

```
Client                          API Server
  │  POST /jobs                     │
  │  { notify_url: "https://…" }    │
  │─────────────────────────────►  │
  │  202 { job_id }                 │
  │◄─────────────────────────────  │
  │                                 │
  │  (time passes, job completes)   │
  │                                 │
  │◄─────── POST /webhook ─────── Worker
  │  { job_id, status: "done" }
```

**Pros:** Real-time, no polling overhead  
**Cons:** Client must expose a public HTTPS endpoint; must handle retries if the client is down  
**Use when:** Server-to-server integrations, B2B APIs

Webhook delivery should include:

- Retries with exponential backoff
- A signature header (HMAC-SHA256) for verification
- Idempotency key so the client can ignore duplicates

---

### Pattern 3: WebSockets / Server-Sent Events (SSE)

The client opens a persistent connection and receives real-time progress updates.

```
Client                          API Server
  │  GET /jobs/:id/stream (SSE)     │
  │─────────────────────────────►  │
  │                                 │
  │◄── data: { progress: 10% } ──  │
  │◄── data: { progress: 45% } ──  │
  │◄── data: { progress: 90% } ──  │
  │◄── data: { status: "done" } ── │
```

**Pros:** Real-time progress bars, best UX  
**Cons:** Requires persistent connection management; harder to scale  
**Use when:** Rich browser UIs where progress matters (video uploads, AI generation)

---

## 8. Handling Failures

Failures are inevitable. Design for them explicitly.

### 8.1 Retry with Exponential Backoff + Jitter

Don't retry immediately — back off to avoid thundering herd problems.

```
Attempt 1: failed → wait 2s
Attempt 2: failed → wait 4s
Attempt 3: failed → wait 8s
Attempt 4: failed → wait 16s + random(0–4s)  ← jitter prevents synchronized retries
Attempt 5: failed → Dead Letter Queue
```

### 8.2 Dead Letter Queue (DLQ)

After `N` failed retries, move the job to a DLQ — a separate queue for jobs that could not be processed.

```
Main Queue → (5 retries fail) → Dead Letter Queue
                                        ↓
                                  Alert on-call engineer
                                  Inspect payload
                                  Requeue after fix
```

The DLQ is not a graveyard — it's a quarantine. Jobs there should be investigated and replayed once the underlying bug is fixed.

### 8.3 Poison Message Detection

A message that always causes worker crashes (e.g. malformed payload causing a segfault) is a **poison message**. It will cycle endlessly through retry → crash → retry.

Defense:

- Track per-message delivery count
- Move to DLQ after threshold (SQS does this automatically with `maxReceiveCount`)

### 8.4 Timeouts

Workers must enforce their own execution timeout — a job that hangs forever should be killed and retried.

```
job_timeout = 10 minutes
if execution_time > job_timeout:
    raise TimeoutError("Job exceeded max execution time")
```

---

## 9. Multi-Step Workflows & Orchestration

Simple queues work for single-step jobs. But many real tasks are **pipelines of dependent steps**.

### Example: Video Processing Pipeline

```
Upload → Validate → Transcode → Generate Thumbnails → Update CDN → Notify User
```

A failure at step 4 shouldn't re-run steps 1–3. Each step's output is the next step's input.

### Naive Approach: Chained Queues

Each worker publishes to the next queue when done.

```
[Upload Queue] → Worker → [Transcode Queue] → Worker → [Thumbnail Queue] → ...
```

**Problems:**

- State scattered across queues — hard to debug
- No central view of pipeline progress
- Retry logic must be reimplemented at every step
- Branching logic becomes spaghetti

### Better: Workflow Orchestration

Dedicated systems manage the entire lifecycle of a multi-step workflow.

---

### Temporal

Code-native workflow engine. You write workflows as regular code; Temporal handles durability, retries, and state.

```python
@workflow.defn
class VideoProcessingWorkflow:
    @workflow.run
    async def run(self, input: VideoInput) -> str:
        validated = await workflow.execute_activity(
            validate_video, input, start_to_close_timeout=timedelta(minutes=5)
        )
        transcoded = await workflow.execute_activity(
            transcode_video, validated, start_to_close_timeout=timedelta(hours=1)
        )
        thumbnails = await workflow.execute_activity(
            generate_thumbnails, transcoded, start_to_close_timeout=timedelta(minutes=10)
        )
        await workflow.execute_activity(notify_user, thumbnails)
        return transcoded.output_url
```

If the process crashes mid-execution, Temporal replays the workflow from where it left off — the code itself is the state machine.

**Strengths:** Code feels natural, full replay on failure, excellent visibility  
**Use when:** Complex workflows in Python/Go/Java/TypeScript codebases

---

### AWS Step Functions

Visual, serverless state machine orchestration. Define your workflow as a JSON state machine; AWS runs it.

```
[Validate] → [Transcode] → [Thumbnails] → [Notify]
                ↓ on error
           [Handle Failure]
```

**Strengths:** Fully managed, native AWS integration, visual debugger, no servers  
**Use when:** AWS-first teams, teams wanting a low-ops solution

---

### When to Use What

|Situation|Recommendation|
|---|---|
|Single, independent job|Plain queue (Redis/SQS)|
|2–3 sequential steps, simple logic|Chained queues|
|Complex pipeline with branching, retries, long duration|Temporal or Step Functions|
|Existing AWS infrastructure|Step Functions|
|Code-first preference, language flexibility|Temporal|

---

## 10. Scaling the System

### Scaling Workers

Workers are stateless and horizontally scalable. Add more workers to process more jobs concurrently.

```
Queue
  │
  ├──► Worker 1
  ├──► Worker 2
  ├──► Worker 3
  └──► Worker N  (auto-scale based on queue depth)
```

**Auto-scaling trigger:** Queue depth (number of messages waiting). If depth > threshold, add workers. If depth = 0, scale down.

On AWS: use SQS queue depth as a CloudWatch metric to trigger ECS or EC2 Auto Scaling.

### Prioritization

Not all jobs are equal. Use **priority queues** for different job tiers:

```
Critical Queue  → Workers (always allocated)      ← user-facing, real-time
High Queue      → Workers (scaled aggressively)   ← paid tier, SLA-bound
Normal Queue    → Workers (standard scaling)      ← free tier
Batch Queue     → Workers (low priority, spot)    ← background ETL, reports
```

### Rate Limiting

Prevent a single customer from flooding the queue and starving others:

- Track per-customer job submission rate
- Reject or queue-delay submissions that exceed the limit
- Use token bucket or leaky bucket algorithms

### Partitioning / Sharding

For very high throughput, shard queues by customer, job type, or geography:

```
video-transcode-us-east
video-transcode-eu-west
video-transcode-ap-south
```

---

## 11. Observability & Monitoring

A job queue system is only as reliable as your ability to see inside it.

### Key Metrics

|Metric|What It Tells You|
|---|---|
|**Queue depth**|Backlog size — are workers keeping up?|
|**Job latency** (enqueue → start)|How long jobs wait in queue|
|**Job duration** (start → complete)|How long processing takes|
|**Error rate**|% of jobs failing|
|**DLQ depth**|Jobs that exhausted retries — needs investigation|
|**Worker utilization**|Are workers idle or maxed out?|
|**Throughput**|Jobs completed per minute|

### Alerting Rules

```
ALERT if queue_depth > 1000 for 5 minutes   → workers not keeping up
ALERT if dlq_depth > 0                       → failed jobs need investigation
ALERT if job_latency_p99 > 60s              → SLA breach risk
ALERT if worker_count = 0 and queue_depth > 0 → all workers down
```

### Distributed Tracing

Propagate a `trace_id` from the original HTTP request through the queue message and into the worker. This lets you trace a single job's entire journey across services in tools like Jaeger, Datadog APM, or AWS X-Ray.

```
HTTP Request (trace_id: abc) 
  → Queue message (trace_id: abc)
    → Worker execution (trace_id: abc)
      → Downstream service calls (trace_id: abc)
```

---

## 12. Real-World Example: Video Processing Pipeline

Let's walk through a complete, concrete design for a video transcoding service.

### Requirements

- Users upload videos; they should be available in HLS format within 10 minutes
- Users can check progress; they receive a notification when done
- System must handle 1000 concurrent transcoding jobs
- Failed jobs must be retried up to 3 times

### Architecture

```
                                                ┌──────────────┐
User ──► Upload Service ──► S3 (raw video)      │   Workers    │
              │                                  │  (EC2/ECS)   │
              ▼                                  └──────┬───────┘
         API Server                                     │
              │ POST /jobs                              │ Poll
              │ → 202 { job_id }                       │
              ▼                                         │
         Job Store ◄──────────────────────────── Update Status
         (Postgres)                                     │
              │                                         │
              ▼                                         │
         SQS Queue ──────────────────────────────────► │
              │                                   ┌─────┴──────┐
              │                                   │ FFmpeg      │
              │                                   │ Transcode   │
              ▼                                   │ → S3 (HLS)  │
         DLQ (failed jobs)                        └────────────┘
```

### Job Lifecycle

```
1. User uploads to S3 via presigned URL
2. Client calls POST /jobs/transcode { s3_key, notify_url }
3. API creates job record (status: pending), enqueues job_id to SQS, returns 202
4. Worker picks up job_id, fetches payload from DB
5. Worker updates status → processing
6. Worker downloads raw video from S3, runs FFmpeg
7. Worker uploads HLS segments to S3/CDN
8. Worker updates status → done, result = { hls_url }
9. Worker POSTs to notify_url
10. CDN URL is live; user receives notification
```

### Failure Scenarios

|Failure|Response|
|---|---|
|Worker crashes mid-transcode|SQS visibility timeout expires → job requeued → new worker picks it up|
|FFmpeg fails (corrupted video)|Worker marks job `failed`, retries with backoff|
|All retries exhausted|Job moved to DLQ, alert fires, on-call investigates|
|Notification webhook fails|Retry webhook delivery independently; use separate retry queue|
|S3 upload fails|Retry at step 7 only (idempotent; overwrites same key)|

---

## 13. Interview Cheat Sheet

When a system design interview asks about long running tasks, cover these points in order:

### Step 1: Identify the Problem

> "This operation takes too long for synchronous execution. Web servers will block, users will timeout, and failures can't be retried. We need async processing."

### Step 2: Propose the Pattern

> "Accept the request, persist the job, enqueue the job ID, return 202 with a job ID. Workers process asynchronously."

### Step 3: Choose a Queue — and Justify It

> - "Redis/BullMQ if we want simplicity and speed"
> - "SQS if we're on AWS and need managed retries and DLQ"
> - "Kafka if we need replay, high throughput, or fan-out to multiple consumers"

### Step 4: Address Reliability

> "Workers must be idempotent. We need visibility timeouts to prevent double processing. Failed jobs go to a dead-letter queue after N retries with exponential backoff."

### Step 5: Discuss Status & Delivery

> "Polling is simplest. Webhooks are better for server-to-server. SSE or WebSockets for real-time progress in a browser."

### Step 6: Handle Workflows

> "If the job has multiple dependent steps, a plain queue becomes unmanageable. I'd use Temporal or Step Functions for orchestration, retry-at-step-level, and visibility."

### Step 7: Scaling

> "Workers are stateless — scale horizontally. Auto-scale on queue depth. Use priority queues for different job tiers."

### Step 8: Observability

> "Monitor queue depth, job latency, error rate, and DLQ depth. Alert on queue backlog and DLQ growth. Propagate trace IDs for end-to-end debugging."

---

### Common Trade-off Questions

**Q: Why not just use a bigger timeout?**  
A: Doesn't scale. Ties up server threads. Doesn't handle retries. Not a real solution.

**Q: What if the worker crashes after completing the job but before deleting the queue message?**  
A: Job runs again — this is why workers must be idempotent.

**Q: How do you prevent a customer from abusing the queue?**  
A: Per-customer rate limiting at the API layer. Priority queues for fairness.

**Q: How do you handle a job that takes longer than the visibility timeout?**  
A: Workers extend the visibility timeout periodically (heartbeat) while they're still working.

**Q: When would you use Temporal over Step Functions?**  
A: Temporal is code-first (better for complex logic, easier to test, not AWS-specific). Step Functions is better if you want zero infra and you're already on AWS.

---

## Related

- [[System Design MOC]]
- [[Message Brokers Guide — RabbitMQ & Kafka]] · [[Backpressure]]
- [[Large Blobs]] — async post-processing after upload
- [[Thundering Herd Problem]]

← [[System Design MOC]] · [[Concepts MOC]]
