# NFR and Estimation Roadmaps

Do them in this order. Each roadmap has 3 to 5 small steps, so you can take one, finish it, and cross it off.

---

## Roadmap 0: The map (1 day)

Learn the 10 NFRs as a checklist you run in every interview and design review:

1. Performance (latency, throughput)
2. Scalability
3. Availability
4. Reliability / fault tolerance
5. Consistency
6. Durability
7. Security
8. Observability
9. Maintainability / extensibility
10. Cost (plus compliance and data retention)

**Done when:** you can list them from memory in 30 seconds.

---

## Roadmap 1: Turning vague business words into NFRs (the core skill)

This is your real gap, so do it first.

1. **Learn the 4-step translation:** vague word → clarifying question → measurable number → design consequence.
    - "Fast" → fast for which action? → p99 under 200 ms → cache or read replica.
    - "Reliable" → what happens if it is down 1 hour? → 99.95% availability → redundancy and failover.
    - "Big" → how many users, and how much growth? → 50M DAU → partitioning.
    - "Real-time" → how stale can data be? → under 1 s → push or streaming, not polling.
    - "Secure" → who are the attackers and what is the data? → auth, encryption, audit logs.
2. **Learn the "money and pain" questions.** Ask what the business loses if it is slow, down, wrong, or lost. The answer tells you which NFR matters most.
3. **Learn the trigger words per NFR.** Examples are "payments" or "balance" (consistency, durability), "launch event" or "flash sale" (scalability, spikes), and "telecom" or "24/7" (availability).
4. **Practice:** take 5 one-line business requirements and rewrite each as 3 to 5 measurable NFRs.

**Done when:** you never say "it should be fast or scalable" without attaching a number.

---

## Roadmaps 2 to 10: One per NFR

Each has the same 4 steps: **Concept → Numbers to ask → Detect signals → Solutions toolbox.**

### Roadmap 2: Performance

- Concept: latency vs throughput, percentiles (p50, p95, p99), tail latency, Little's Law (you already know it).
- Numbers to ask: target p99, peak RPS.
- Solutions: caching layers, indexes, CDN, async work, batching, connection pooling, denormalization.

### Roadmap 3: Scalability

- Concept: vertical vs horizontal scaling, stateless services, bottleneck identification.
- Numbers to ask: growth rate, peak vs average ratio, spike shape.
- Solutions: load balancing, sharding (you already studied it), read replicas, queues to absorb spikes, autoscaling.

### Roadmap 4: Availability

- Concept: the nines (99.9% vs 99.99% as downtime per year), SLI, SLO, SLA, single points of failure.
- Numbers to ask: allowed downtime, cost of downtime.
- Solutions: redundancy, multi-AZ and multi-region, health checks, failover, graceful degradation.

### Roadmap 5: Reliability and fault tolerance

- Concept: failure modes, blast radius, retries amplifying failures.
- Numbers to ask: acceptable error rate, recovery time (RTO) and recovery point (RPO).
- Solutions: timeouts, retries with backoff, idempotency, circuit breakers, bulkheads, DLQ, sagas. You already know most of these, so this one is mainly about naming them under the right NFR.

### Roadmap 6: Consistency

- Concept: CAP and PACELC, strong vs eventual, read-your-writes, isolation levels.
- Numbers to ask: how stale can data be, and what is the cost of a wrong read?
- Solutions: quorum, single leader, transactions, outbox, conflict resolution.

### Roadmap 7: Durability

- Concept: durability vs availability (they are not the same), WAL, replication.
- Numbers to ask: can we lose any acknowledged write? RPO.
- Solutions: synchronous replication, backups, point-in-time recovery, write-ahead logging.

### Roadmap 8: Security and compliance

- Concept: authn vs authz, threat modeling at a basic level, data classification.
- Numbers to ask: what data, who has access, which regulations apply, how long to retain.
- Solutions: JWT/OAuth (you are already on it), encryption in transit and at rest, rate limiting, audit logs, least privilege, PII handling.

### Roadmap 9: Observability and maintainability

- Concept: logs, metrics, traces; deployability; extensibility.
- Numbers to ask: time to detect, time to recover, release frequency.
- Solutions: OpenTelemetry (already studied), alerting on SLOs, feature flags, CI/CD, clear service boundaries.

### Roadmap 10: Cost

- Concept: cost drivers (compute, storage, egress, per-request pricing).
- Numbers to ask: budget per user or per request.
- Solutions: caching, tiered storage, right-sizing, async batching, choosing managed vs self-hosted.

**Done when, for each one:** you can answer "what question do I ask, what number do I get, and what component do I add?"

---

## Roadmap 11: The NFR trade-offs (1 to 2 days)

NFRs conflict, and seniors are judged on how they handle that.

1. Learn the common conflicts: consistency vs availability, latency vs durability, cost vs availability, security vs performance, flexibility vs speed.
2. Learn to say, "Given requirement X, I prioritize A over B because the business loses Y."
3. Practice by ranking NFRs for 3 systems (payments, social feed, chat).

---

## Roadmap 12: Back-of-envelope estimation

1. **Memorize the numbers (2 days).**
    - Powers of 2 and 10 (KB, MB, GB, TB).
    - Latency table: memory, SSD, network in a datacenter, cross-region.
    - Seconds per day is about 100K.
    - Typical limits: one DB node handles roughly 1K to 10K QPS, and one app server roughly 1K to 5K RPS.
2. **Learn the formulas (2 days).**
    - DAU → average QPS = requests per user per day × DAU ÷ 100K.
    - Peak QPS = average × 2 to 5.
    - Storage = records per day × size × retention.
    - Bandwidth = QPS × payload size.
    - Cache size ≈ 20% of the daily hot data.
    - Servers = peak QPS ÷ per-server capacity.
3. **Learn to turn numbers into decisions (1 day).** For example, if the result is "under 1K QPS and under 1 TB," a single Postgres is enough, and "100K QPS" means caching and sharding.
4. **Practice (ongoing).** Do 10 problems: URL shortener, Twitter feed, chat, file storage, rate limiter, notification system, video streaming. Always end with "so what does this number change in my design?"

**Done when:** you can estimate any system in under 5 minutes and round aggressively without freezing.

---

## Roadmap 13: Combine everything (the interview routine)

Use this order in every design:

1. Functional requirements (your strength).
2. Run the NFR checklist, converting vague words to numbers.
3. Estimate with back-of-envelope.
4. Pick the design and name which NFR each component serves.
5. State the trade-offs.

**Practice:** repeat this on your own work too, such as your BFF and the OCS migration. Ask what the NFRs were and whether you ever wrote them down with numbers.

---

## Suggested order

Roadmap 0 → 1 → 12 → 2, 3, 4 → 5, 6, 7 → 8, 9, 10 → 11 → 13.

Pick the roadmap you want to start with and I'll write it as a full markdown guide (concept, numbers, signals, solutions, interview phrasing). I'd suggest Roadmap 1, since everything else depends on it.