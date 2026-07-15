---
tags:
  - logging
  - observability
---

# Structured Logging — From Scratch

## The Problem: Plain Text Logs

When you first learn programming, logging looks like this:

```python
print("User logged in")
print("Order placed for user 42, amount 99.99")
print("ERROR: Payment failed for order 123")
```

Or using a basic logger:

```python
import logging
logging.info("User logged in")
logging.warning("Payment failed")
```

This feels fine at first. But once your application grows — multiple services, thousands of users, production incidents — **plain text logs become a nightmare.**

---

## What Goes Wrong with Plain Text Logs

### 1. You Can't Search or Filter Effectively

Imagine you want to find all failed payments for user `42` in the last hour.

Your logs look like:

```
2024-01-15 10:23:11 Payment failed for user 42 order 99
2024-01-15 10:24:55 User 42 logged in
2024-01-15 10:25:01 Payment failed for user 87 order 100
2024-01-15 10:26:33 Payment failed for user 42 order 101
```

You'd have to write fragile regex like:

```bash
grep "Payment failed for user 42" app.log
```

This breaks the moment someone changes the log message wording.

---

### 2. Log Formats Are Inconsistent

Different developers write logs differently:

```
User 42 payment failed
Payment error - userId=42
ERROR: [user:42] could not charge card
Failed payment. User: 42. Reason: insufficient funds
```

Same event. Four different formats. No machine can reliably parse all of these.

---

### 3. You Can't Aggregate or Analyze

- How many payments failed per hour?
- Which users are hitting errors the most?
- What's the average response time by endpoint?

With plain text logs, answering these questions means writing custom parsers — fragile, expensive, and slow.

---

### 4. Correlation Across Services Is Impossible

In a microservices architecture, a single user request touches 5–10 services. Each service logs its own plain text. When something breaks, tracing the full request across services is like trying to read 10 books at once where the pages are shuffled together.

---

## The Solution: Structured Logging

**Structured logging** means writing logs as **data** (usually JSON) instead of free-form text.

Instead of:

```
Payment failed for user 42
```

You log:

```json
{
  "timestamp": "2024-01-15T10:23:11Z",
  "level": "ERROR",
  "event": "payment_failed",
  "user_id": 42,
  "order_id": 99,
  "amount": 199.99,
  "reason": "insufficient_funds",
  "service": "payment-service",
  "trace_id": "abc-123-xyz"
}
```

Every piece of information has a **named key**. The log is a record, not a sentence.

---

## Problems Structured Logging Solves

### ✅ 1. Precise Filtering & Search

With structured logs, your log aggregation tool (Elasticsearch, Loki, Splunk, Datadog) can query like a database:

```sql
SELECT * FROM logs
WHERE user_id = 42
  AND event = 'payment_failed'
  AND timestamp > now() - interval '1 hour'
```

No regex. No guessing. Exact results.

---

### ✅ 2. Consistent, Machine-Readable Format

Because every log is JSON (or another structured format), any tool can parse it reliably — regardless of which developer wrote the log line, or which service emitted it.

---

### ✅ 3. Aggregation & Metrics

You can now answer operational questions instantly:

- Count `payment_failed` events grouped by `reason` → find the most common failure cause
- Average `response_time_ms` grouped by `endpoint` → find slow routes
- Count errors per `user_id` → detect abuse or broken accounts

These are just GROUP BY queries over your log data.

---

### ✅ 4. Request Tracing Across Services

Every log entry includes a `trace_id` (generated at the edge when a request arrives). All services pass this ID along and include it in their logs.

Now you can find all logs for a single request across 10 services in one query:

```
trace_id = "abc-123-xyz"
```

This is the foundation of **distributed tracing**.

---

### ✅ 5. Alerting & Monitoring

With structured logs, you can set up alerts based on field values:

- Alert when `level = ERROR` and `service = payment-service` exceeds 10 events/minute
- Alert when `response_time_ms > 2000` for any endpoint

Plain text logs make this nearly impossible without brittle regex.

---

## Structured Logging in Practice

### Python — using `structlog`

```python
import structlog

log = structlog.get_logger()

log.info(
    "payment_failed",
    user_id=42,
    order_id=99,
    amount=199.99,
    reason="insufficient_funds"
)
```

Output (JSON):

```json
{
  "event": "payment_failed",
  "user_id": 42,
  "order_id": 99,
  "amount": 199.99,
  "reason": "insufficient_funds",
  "level": "info",
  "timestamp": "2024-01-15T10:23:11Z"
}
```

---

### Node.js — using `pino`

```javascript
const pino = require('pino');
const log = pino();

log.error({
  event: 'payment_failed',
  userId: 42,
  orderId: 99,
  amount: 199.99,
  reason: 'insufficient_funds'
});
```

---

### Go — using `zap`

```go
logger.Error("payment_failed",
    zap.Int("user_id", 42),
    zap.Int("order_id", 99),
    zap.Float64("amount", 199.99),
    zap.String("reason", "insufficient_funds"),
)
```

---

## Key Fields to Always Include

|Field|Why|
|---|---|
|`timestamp`|When did this happen?|
|`level`|DEBUG / INFO / WARN / ERROR|
|`event`|A stable, machine-readable event name|
|`service`|Which service emitted this log|
|`trace_id`|Links logs across services for one request|
|`user_id`|Who was involved|
|`environment`|production / staging / dev|

---

## Structured vs Unstructured — Side-by-Side

||Unstructured|Structured|
|---|---|---|
|Format|Free text|JSON / key-value|
|Searchable|Fragile regex|Exact field queries|
|Consistent|No|Yes|
|Aggregatable|Hard|Easy|
|Cross-service tracing|Nearly impossible|Built-in via `trace_id`|
|Alerting|Fragile|Reliable|
|Onboarding new devs|Hard to know format|Self-documenting|

---

## Common Pitfalls

**❌ Logging sensitive data** Never log passwords, tokens, full credit card numbers, or PII unless required and encrypted.

```json
// BAD
{ "event": "login", "password": "hunter2" }

// GOOD
{ "event": "login", "user_id": 42 }
```

**❌ Using dynamic event names**

```python
# BAD — breaks aggregation
log.info(f"user {user_id} failed payment")

# GOOD — stable key, dynamic value
log.info("payment_failed", user_id=user_id)
```

**❌ Over-logging** Don't log every variable in a tight loop. Log meaningful events at meaningful boundaries (start of request, end of request, errors, important state changes).

---

## Summary

Structured logging transforms your logs from a wall of text into **queryable, aggregatable, alertable data**. It is the foundation of observability — your ability to understand what your system is doing in production.

The shift is conceptual: stop thinking of a log as a message for a human to read, and start thinking of it as a **record of an event** that both humans and machines can process.

> **Rule of thumb:** If you'd want to search by it, filter by it, or alert on it — make it a field, not part of a sentence.