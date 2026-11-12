# Deployment Strategies for Backend Engineers

## Why This Matters

Every time you deploy, your running instances get replaced. If you do this naively, two things break:

- **In-flight requests die mid-execution** — a user's transaction is halfway through, your process gets killed, data is left in an inconsistent state
- **Users experience downtime** — even for a few seconds, that's unacceptable in production

Deployment strategies exist to solve exactly this: **how do you swap old code for new code without users feeling it?**

But the strategy alone isn't enough. Your application also needs to know _how to die cleanly_. That's where **OS signals**come in — specifically `SIGTERM`, which is the system's way of telling your process "shut down now, but do it gracefully." If your app ignores it, it gets force-killed mid-request.

So before any deployment strategy can give you true zero downtime, three things must be true:

1. **The strategy routes traffic safely** — users never hit a dead instance
2. **The app handles SIGTERM** — it finishes what it's doing before exiting
3. **The DB migration is backward-compatible** — old and new code can run side by side without breaking each other

Everything in this guide builds on those three ideas.

---

## Tier 1 — Must Know Deeply

### 1. Rolling Deployment

**What it is:** Gradually replace old instances with new ones, a few at a time. At no point is the entire service down. This is the **default deployment strategy in Kubernetes** and what most teams use for routine releases.

**How it actually works step by step:**

You have 4 pods running v1. You deploy v2. Kubernetes doesn't kill everything at once — it follows the rules you set:

```
Desired pods: 4
maxSurge: 1         → can temporarily have 5 pods during rollout
maxUnavailable: 0   → never drop below 4 healthy pods

Step 1: spin up 1 new v2 pod → now 5 pods total [v1 v1 v1 v1 v2]
        wait for v2 to pass health check...
        send SIGTERM to 1 old v1 pod → it finishes in-flight requests and exits
        now back to 4 pods [v1 v1 v1 v2]

Step 2: spin up 1 new v2 pod → [v1 v1 v1 v2 v2]
        wait for health check...
        SIGTERM to old v1 → [v1 v1 v2 v2]

Step 3: [v1 v1 v2 v2 v2] → [v1 v2 v2 v2]
Step 4: [v1 v2 v2 v2 v2] → [v2 v2 v2 v2] ✅ done
```

**The health check is critical here.** Kubernetes won't terminate an old pod until the new one passes its `readinessProbe`. If the new pod never becomes ready (a bug, a bad config), the rollout stops and your old pods stay up. This is the built-in safety net.

**The DB compatibility problem:**

During the rollout window, v1 and v2 are both live and both hitting the same database. This means:

- If v2 adds a new column, v1 must not crash when it sees it
- If v2 renames a column, v1 is now broken
- You can never do destructive schema changes in the same deploy

This is why rolling deployments force you to write **backward-compatible migrations** — schema changes and code changes must be decoupled across separate deploys.

**Rollback:**

There is no instant rollback. If v2 is bad, you redeploy v1 — which triggers another rolling cycle. This takes time. If you need instant rollback, you need Blue-Green.

**Tradeoffs:**

|Pro|Con|
|---|---|
|Zero downtime|v1 and v2 serve traffic simultaneously during rollout|
|No extra infrastructure|No instant rollback|
|Built-in safety via health checks|DB must be backward-compatible|
|Default in Kubernetes|Rollback = another full rolling cycle|

**When to use:** Routine releases with low risk and backward-compatible schema changes.

---

### 2. Blue-Green Deployment

**What it is:** Run two complete, identical environments in parallel — Blue (current live) and Green (new version). Traffic stays 100% on Blue while you build and validate Green. When ready, you flip the load balancer. All traffic moves to Green instantly.

**How it actually works:**

```
Phase 1 — Build Green while Blue serves all traffic:

Users → Load Balancer → [Blue v1] [Blue v1] [Blue v1] [Blue v1]  ← 100% live
                        [Green v2] [Green v2] [Green v2] [Green v2] ← being deployed, tested

Phase 2 — Validate Green (smoke tests, health checks, QA):
  Green is fully up but receives zero real traffic
  You run your test suite against Green directly
  You verify DB migrations ran correctly

Phase 3 — Flip:
Users → Load Balancer → [Blue v1] [Blue v1] [Blue v1] [Blue v1]  ← idle, kept alive
                        [Green v2] [Green v2] [Green v2] [Green v2] ← 100% live ✅

Phase 4 — Rollback if needed:
  Something wrong? Flip back to Blue in seconds.
  Blue is still running — nothing was torn down yet.

Phase 5 — Teardown:
  After confidence period (hours/days), shut down Blue.
  SIGTERM sent to Blue pods — they drain and exit.
```

**The DB migration problem is harder here:**

With rolling, you have a gradual overlap. With Blue-Green, you flip instantly — but your database is shared between both environments. This means:

- If Green runs a migration that renames a column, and you flip back to Blue — Blue breaks because the column it expects is gone
- The safest approach: **only run additive migrations before the flip** (add columns, add tables), and **never destructive ones** (drop, rename) until Blue is fully retired
- This is why Blue-Green is paired with the expand/contract pattern

**Why it's ideal for urgent hotfixes:**

- You build the fix in Green
- You test it fully before any user sees it
- You flip — instant, clean cutover
- If the fix has a bug, you flip back to Blue in seconds — no re-deploy cycle

**Cost tradeoff:**

You need double the infrastructure during the transition window. For large-scale systems this matters. For most mid-size teams this is acceptable for the safety it buys.

**Tradeoffs:**

|Pro|Con|
|---|---|
|Instant rollback (flip back to Blue)|Double infrastructure cost during transition|
|No mixed-version traffic|DB migrations require extra care|
|Full validation before any user sees new version|Requires environment parity (Blue = Green in config/infra)|
|Clean, binary cutover|More complex setup than rolling|

**When to use:** Urgent hotfixes, high-risk releases, or any time you need the confidence of instant rollback.

---

### ⚠️ The Golden Rule for Rolling & Blue-Green — Non-Destructive Migrations Only

This is a rule that comes out of both strategies, not specific to either one. Any time you have two versions of your code running — whether simultaneously (rolling) or with a potential flip-back (blue-green) — **your database must be able to serve both versions at the same time.**

This means one absolute rule:

> **Never run a destructive SQL command as part of a deployment. Ever.**

Destructive commands are anything that removes or changes the shape of existing data:

```sql
-- ❌ NEVER do these in the same deploy as your code change
ALTER TABLE orders RENAME COLUMN total TO total_amount;
ALTER TABLE orders DROP COLUMN legacy_status;
ALTER TABLE users DROP TABLE sessions;
```

If you do any of these and need to roll back — your old code is now broken because the column or table it depends on is gone.

**The pattern that solves this is Expand / Contract (also called shadow migration):**

Instead of changing the schema in one step, you split it across multiple deploys:

```
Deploy 1 — EXPAND:
  Add the new column (nullable, no constraints yet)
  ALTER TABLE orders ADD COLUMN total_amount NUMERIC;
  Both old code (uses total) and new code (uses total_amount) work fine ✅

Deploy 2 — MIGRATE:
  Backfill data into the new column
  UPDATE orders SET total_amount = total WHERE total_amount IS NULL;
  Run this as a background job for large tables

Deploy 3 — SWITCH:
  New code now reads/writes total_amount only
  Old code is fully retired, no more pods running v1

Deploy 4 — CONTRACT:
  Now it's safe to drop the old column — nobody uses it
  ALTER TABLE orders DROP COLUMN total;
```

**The rules in plain English:**

| Rule                                                    | Why                                      |
| ------------------------------------------------------- | ---------------------------------------- |
| Never DROP or RENAME in the same deploy as code changes | Old pods will break instantly            |
| Never add a NOT NULL column without a default           | Old code inserting rows will fail        |
| Always add columns as nullable first                    | Both versions can coexist                |
| Only drop after all old pods are fully gone             | Safe to remove what nobody reads anymore |

**This applies to both strategies:**

- **Rolling** — v1 and v2 pods are live simultaneously during rollout window
- **Blue-Green** — Blue must stay functional in case you flip back

The expand/contract pattern is what makes zero-downtime deployments actually safe at the database level.

---

### 3. Feature Flags

**What it is:** Decouple **deployment** from **release**. You ship the code to production — but the feature is off. A flag in your system controls whether the code path runs. You turn it on independently, per user, per segment, or by percentage — without touching your deployment pipeline at all.

**The core idea:**

Without feature flags, deployment = release. The moment you deploy, users see it.

With feature flags:

```
Deploy v2 (flag OFF) → nobody sees the new feature
                     → internal team tests it in production
                     → enable for 5% of users → monitor errors/latency
                     → enable for 50% → monitor
                     → enable for 100% → fully released
                     → remove the flag from code (cleanup)
```

**How the flag check works in code:**

```js
// The flag value comes from your flag service (DB, Redis, LaunchDarkly, etc.)
const flag = await flagService.isEnabled('new-checkout-flow', { userId: user.id });

if (flag) {
  return newCheckoutFlow(cart);  // v2 path
} else {
  return oldCheckoutFlow(cart);  // v1 path — still in the codebase
}
```

The key thing: **both code paths exist in production simultaneously**. The flag decides which one runs.

**Types of flag targeting:**

|Target|Example use|
|---|---|
|All users OFF|Just deployed, not ready|
|Internal users only|QA and team testing in prod|
|Percentage rollout|5% → 20% → 100% gradual release|
|Specific user IDs|Beta testers, specific accounts|
|Per region|Release in KSA before global|

**The kill switch:**

If something goes wrong in production — latency spike, error rate increase — you turn the flag OFF. No rollback, no re-deploy. The old code path is already there, already running. This is the most powerful thing about feature flags.

**The tech debt problem:**

Every flag is a branch in your code. If you never clean it up:

```js
// 6 months later, nobody knows if this flag is still needed
if (await flagService.isEnabled('new-checkout-flow')) { ... }
```

Flags that aren't removed become landmines. The rule: **once a flag is at 100% and stable, remove the flag and the old code path within the same sprint.**

**Tradeoffs:**

|Pro|Con|
|---|---|
|Ship anytime, release when ready|Dead flags accumulate as tech debt|
|Instant kill switch without re-deploy|Code has conditional branches everywhere|
|Per-user / per-segment control|Requires a flag management system|
|Test in production safely|Both code paths must be maintained until flag is removed|

**When to use:** Every significant feature. This is the standard in modern engineering — deployment and release are separate events.

**Tools:** LaunchDarkly (managed), Unleash (self-hosted open source), or a simple Redis/DB-backed key-value config for small teams.

---

## Tier 2 — Know Conceptually, Understand the Tradeoffs

### 4. Canary Deployment

**What it is:** Release the new version to a small percentage of real users first, monitor, then gradually increase to 100%.

```
100% users → [v1]

After canary release:
 95% users → [v1]
  5% users → [v2] ← watch error rate, latency, metrics

If healthy → increase to 20% → 50% → 100%
If problem → route back to v1
```

**Tradeoffs:**

|Pro|Con|
|---|---|
|Real-world validation before full rollout|Requires good observability to catch issues|
|Gradual blast radius|More complex traffic routing|
|Easy rollback|Old and new versions run simultaneously|

**When to use:** Risky or high-impact changes where you want real traffic validation before full rollout.

**Key depth:** Know when to choose canary over rolling. The key difference: rolling replaces instances progressively, canary splits _traffic_ by percentage.

---

### 5. A/B Deployment

**What it is:** Like canary, but intentional — split traffic between two variants to _measure_ which performs better, not just whether the new version is stable.

```
50% users → [Variant A] (current behavior)
50% users → [Variant B] (new behavior)

Measure: conversion rate, click rate, revenue impact
```

**Key distinction from Canary:**

- Canary = "is this safe?" → stability concern
- A/B = "which is better?" → product/metrics concern

**When to use:** Product experiments, UI/UX changes, algorithm comparisons. Usually a product team concern but you'll implement the backend split.

---

### 6. Recreate (In-Place)

**What it is:** Stop all instances of the old version, then deploy the new version. Causes downtime.

```
v1 running → STOP all v1 → deploy v2 → START v2
                  ↑
             downtime here
```

**When to use:** Only for non-critical internal services, scheduled maintenance windows, or very early-stage products where downtime is acceptable.

**Key depth:** Know this exists and why you'd avoid it in production.

---

## Tier 3 — Awareness Only

### 7. Shadow Deployment

Route real production traffic to the new version in parallel, but discard its responses. Users only see v1's responses. Used to test v2 under real load without any risk.

**Use case:** Load testing a new service with real traffic patterns before switching.

---

## The One Concept That Ties Everything Together

### Database Migrations Are the Hardest Part

No matter which strategy you choose, if your schema changes, you must handle **backward compatibility** — because during any deployment, old and new code run simultaneously.

**The Expand / Contract Pattern:**

```
Step 1 — Expand:   Add new_column (nullable). Old code ignores it. ✅
Step 2 — Migrate:  Backfill data into new_column.
Step 3 — Switch:   New code reads/writes new_column. Old code still uses old_column.
Step 4 — Contract: Remove old_column once all old instances are gone.
```

**Rules:**

- ❌ Never drop or rename a column in the same deploy that removes its reference in code
- ❌ Never add a NOT NULL column without a default value
- ✅ Always make schema changes backward-compatible with the previous version of your code

---

## Rolling vs Blue-Green — What Actually Makes Them Different

They both achieve zero downtime, but the _mechanism_ is completely different.

### Rolling — Controlled Percentage Expansion

Rolling is not "replace everything at once." It's a **gradual, configurable expansion** controlled by two Kubernetes parameters:

- **`maxSurge`** — how many _extra_ pods can be added above your desired count during the rollout
- **`maxUnavailable`** — how many pods can be down at the same time during the rollout

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # spin up 1 extra new pod at a time
    maxUnavailable: 0  # never take down an old pod until new one is healthy
```

**What this means in practice** — if you have 4 pods running v1:

```
Start:   [v1] [v1] [v1] [v1]

Step 1:  [v1] [v1] [v1] [v1] [v2]   ← new pod added (maxSurge: 1)
         wait for v2 to be healthy...
         [v1] [v1] [v1] [v2]         ← one old pod removed

Step 2:  [v1] [v1] [v1] [v2] [v2]   ← another new pod added
         [v1] [v1] [v2] [v2]         ← another old pod removed

... and so on until:

Done:    [v2] [v2] [v2] [v2]
```

At every step, SIGTERM is sent to the old pod being replaced — and it shuts down gracefully while the new one is already serving traffic. **This is why graceful shutdown is non-negotiable with rolling deployments.**

---

### Blue-Green — Full Environment, Instant Flip

Blue-Green is not gradual at all. You build the **entire new environment** first, test it, then flip all traffic at once.

```
Before flip:
Users → Load Balancer → [Blue v1] [Blue v1] [Blue v1] [Blue v1]  ← live
                        [Green v2] [Green v2] [Green v2] [Green v2] ← idle, ready

After flip:
Users → Load Balancer → [Blue v1] [Blue v1] [Blue v1] [Blue v1]  ← idle (kept for rollback)
                        [Green v2] [Green v2] [Green v2] [Green v2] ← live ✅
```

No gradual steps. No mixed traffic. It's a binary switch.

---

### Side by Side

||Rolling|Blue-Green|
|---|---|---|
|How traffic shifts|Gradually, pod by pod|All at once (instant flip)|
|Old + new serving traffic together|Yes — during the rollout window|No — only one environment is live at a time|
|Rollback|Slow — must re-deploy old version|Instant — flip back to Blue|
|Infrastructure cost|Normal|Double (two full environments)|
|SIGTERM / graceful shutdown|Critical — happens on every replaced pod|Happens once — when Blue is finally shut down|
|Best for|Routine releases|Major releases, risky changes|

---

## Graceful Shutdown & OS Signals

During any deployment, your running instances will be terminated. How they die matters.

### The Signals You Need to Know

|Signal|Meaning|Default Behavior|
|---|---|---|
|`SIGTERM`|"Please shut down cleanly"|Sent by Kubernetes/Docker before killing the process|
|`SIGKILL`|"Die immediately, no choice"|Sent after grace period expires — cannot be caught|
|`SIGINT`|"Interrupt" (Ctrl+C)|Sent during local dev|

### What Happens During a Rolling/Blue-Green Deploy

```
Kubernetes decides to terminate a pod:

1. Pod removed from Service endpoints → no new traffic routed to it
2. SIGTERM sent to the process        → you have time to finish up
3. Grace period (default: 30s)        → your app should shut down by now
4. SIGKILL sent                       → process is force-killed regardless
```

If you don't handle SIGTERM, step 3 becomes instant death mid-request.

### What Graceful Shutdown Means in Practice

When SIGTERM arrives, your app should:

1. **Stop accepting new requests** — close the HTTP server
2. **Finish in-flight requests** — wait for active requests to complete
3. **Close connections cleanly** — DB pool, Redis, Kafka consumer, message queue
4. **Exit with code 0**

### Node.js Example

```js
const server = app.listen(3000);

process.on('SIGTERM', async () => {
  // 1. Stop accepting new connections
  server.close(async () => {
    // 2. In-flight requests are done at this point

    // 3. Close DB, Redis, Kafka, BullMQ workers, etc.
    await db.end();
    await redisClient.quit();
    await kafkaConsumer.disconnect();

    // 4. Exit cleanly
    process.exit(0);
  });
});
```

> **In Kubernetes:** set `terminationGracePeriodSeconds` (default 30s) to give your app enough time. If your requests can take up to 10s, 30s is fine. If you have long-running jobs, increase it.

### Why This Matters Per Strategy

| Strategy   | Shutdown Risk                                                |
| ---------- | ------------------------------------------------------------ |
| Rolling    | Every replaced pod gets SIGTERM — must handle it             |
| Blue-Green | Old environment is killed after traffic flip — same          |
| Canary     | Pods being drained of traffic still get SIGTERM              |
| Recreate   | All pods killed at once — worst case if no graceful shutdown |
|            |                                                              |

---

## Quick Reference

| Strategy      | Downtime | Rollback Speed   | Complexity | When to Use               |
| ------------- | -------- | ---------------- | ---------- | ------------------------- |
| Rolling       | None     | Slow (re-deploy) | Low        | Default for most releases |
| Blue-Green    | None     | Instant (flip)   | Medium     | Major/risky releases      |
| Feature Flags | None     | Instant (toggle) | Low        | Every significant feature |
| Canary        | None     | Fast             | Medium     | High-risk changes         |
| A/B           | None     | Fast             | Medium     | Product experiments       |
| Recreate      | Yes      | N/A              | None       | Non-critical / dev only   |
| Shadow        | None     | N/A              | High       | Load validation           |

## Highlights in the existing migration content

Mark these lines in the Golden Rule and Expand/Contract sections:

> 🔆 **Expand before you contract.** Add new columns as nullable, with no constraints yet. The schema must support the oldest running version and the newest version you might roll back to.

> 🔆 **Backfill in batches.** A single large `UPDATE` holds locks and bloats the write-ahead log. Make the job resumable and idempotent.

> 🔆 **Enforce constraints safely.** On Postgres, add `CHECK (... IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT`, and set `lock_timeout` on migrations.

> 🔆 **Contract only after a rollback window.** Dropping the old column removes your rollback path. Confirm that no running version, batch job, report, or downstream consumer still reads it.

---

## Notes

These notes go deeper on three places where deployment strategies get hard in practice: long-lived connections, API contracts during overlap, and shared databases. They sit on top of everything above — especially graceful shutdown, expand/contract, and the idea that old and new code must coexist.

---

### Notes: WebSocket Gateway

HTTP is request/response — a deploy kills a process, the client retries, life goes on. WebSockets and SSE are different: the connection *is* the session. Every pod replacement drops real users unless you design for it.

**Why separate the gateway from business services**

```
Clients ──WebSocket──► [Gateway pods] ──events──► [Business services]
                         ↑ holds connections         ↑ deploy often
                         ↑ deploy rarely             ↑ no connection state
```

- The gateway's job is hold connections, authenticate once, route events, and deliver. It should not contain checkout logic, scoring, or domain rules.
- Separating them means you can ship business logic daily without tearing down every open socket.
- The gateway still ships for protocol bumps, auth changes, TLS/library CVEs, and infra upgrades — those deploys need the same drain discipline as any other pod.

**What happens when a gateway pod dies**

1. Load balancer / K8s removes the pod from endpoints → no new connections land there
2. `SIGTERM` → stop accepting new sockets, stop publishing, finish in-flight sends
3. Local connection map is gone — every user on that pod must reconnect elsewhere
4. Clients reconnect (with backoff + jitter), land on a healthy gateway, resume from last-seen event ID

If you skip drain + reconnect design, a routine rolling deploy looks like a mass disconnect.

**Connection state lives on the gateway**

Each gateway instance knows which users/sockets it owns. That map is **not** shared by default. So:

- You cannot "move" a live WebSocket to another pod — the client must reconnect
- Sticky sessions help *during* a connection's life; they do not survive pod death
- Presence ("user is online") must be rebuilt after reconnect, usually via Redis

**Pub/Sub vs Streams — pick by loss tolerance**

| Mechanism | Durability | Resume | Good for | Bad for |
|---|---|---|---|---|
| Redis Pub/Sub | None — fire-and-forget | No | Typing, presence, ephemeral UI signals | Anything the user must see |
| Redis Streams | Persisted, consumer groups | Yes — from last ID | Chat messages, notifications, order events | Pure throwaway signals (overkill) |
| Kafka / similar log | Persisted, partitioned | Yes — offset | High-volume, multi-consumer event buses | Tiny low-traffic apps |

Rule of thumb: if missing an event during a gateway restart is unacceptable, use Streams (or Kafka). If it's a "nice to know" signal, Pub/Sub is fine.

**Fan-out at scale**

Do not subscribe every gateway to every channel. Subscribe only to channels that have local sockets, or use a user/room → gateway routing layer. Blind global fan-out burns CPU and Redis bandwidth as you add pods.

**Delivery semantics**

In practice you get **at-least-once**: a gateway may crash after receiving an event but before the client ACKs it. Therefore:

- Every event needs a stable ID (or sequence number)
- Clients dedupe on ID
- Resumption is "give me everything after `lastSeenId`" — not "replay the last N blindly"

**SSE vs WebSocket resumption**

| | SSE | WebSocket |
|---|---|---|
| Built-in resume | `Last-Event-ID` header | None — you invent it |
| Typical pattern | Server stores events; client sends last ID on reconnect | Client sends `{ resumeFrom: seq }` on connect; server replays or points at a catch-up API |
| Deploy impact | Same — connection drops; resume must work | Same |

**Client reconnect contract (treat as part of the product)**

```
Disconnect → wait (exp backoff + jitter) → reconnect
          → send lastSeenEventId
          → receive missed events (or a "too old, full sync" signal)
          → resume normal stream
```

Without jitter, a rolling deploy of 10 gateway pods can stampede your auth and Redis layers at once.

**Heartbeats and load balancers**

- Use WebSocket ping/pong or SSE comments on an interval (e.g. 15–30s)
- Set LB / proxy idle timeout **above** that interval, or idle connections get killed and look like random flakiness
- On `SIGTERM`, stop heartbeats only after you've closed the accept path and drained writes

**Deploy checklist for a WebSocket gateway**

| Step | Why |
|---|---|
| Drain: stop new accepts, finish sends | Avoid half-delivered frames |
| Publish "server going away" if your protocol supports it | Clients reconnect sooner and cleaner |
| Grace period ≥ longest drain you expect | Default 30s may be too short under load |
| Client backoff + jitter | Prevent reconnect stampede |
| Resume via event ID / Streams offset | No silent message loss |
| Keep business logic out of the gateway | Logic deploys ≠ connection drops |

---

### Notes: API Versioning and Compatibility

During rolling, canary, and blue-green (while Blue still exists for rollback), **two API implementations serve traffic against one shared world** (DB, queues, caches). Compatibility is not optional — it is the deployment strategy.

**Backward vs forward compatibility**

| Direction | Meaning | Example |
|---|---|---|
| Backward compatible | New server still works with old clients | Adding an optional response field |
| Forward compatible | Old server still works with new clients (or ignores unknown fields) | Client sends a new optional field; old server ignores it |

You usually need both during a rollout window: old app binaries, cached mobile builds, and new pods all overlap.

**Safe vs unsafe contract changes**

| Safe (do in one deploy) | Unsafe (needs a multi-step plan) |
|---|---|
| Add optional response field | Rename or remove a field |
| Add optional request field with a default | Make an optional field required |
| Add a new endpoint | Change field meaning/type |
| Widen an enum carefully (unknown → ignore) | Narrow an enum / reject old values |

Mobile and third-party clients may lag weeks behind your server. "We'll just ship a breaking change" often means weeks of 4xx/5xx for someone.

**WebSocket / SSE payloads are APIs**

Treat event schemas like REST: additive fields, version or capability negotiation at connect time, never silently reinterpret an existing field. Example handshake:

```
Client → { protocolVersion: 2, supports: ["receipts", "typing"] }
Server → { protocolVersion: 2, enabled: ["receipts"] }
```

If the server only speaks v1, the client either downgrades or fails closed with a clear error — not a cryptic parse crash.

**Idempotency during rollouts**

Retries spike when pods die mid-request. Without idempotency:

- Double charges, double notifications, duplicate rows
- "It worked on my machine" becomes "it worked twice in prod"

Patterns that hold under deploy churn:

- Client sends `Idempotency-Key` on POSTs that create side effects
- Server stores key → result for a TTL and returns the same response on replay
- Consumers of queues/streams also dedupe on event ID

**Breaking changes as a multi-deploy process**

Same shape as expand/contract:

```
Deploy 1 — Support both shapes (old + new)
Deploy 2 — Migrate consumers / clients to the new shape
Deploy 3 — Remove the old shape only after metrics show zero old usage
```

Feature flags help here: serve the new response shape to internal users first, then percentage roll out, then delete the old serializer.

**Practical rules**

- Prefer additive changes; treat renames as "add new + dual-write/read + drop old"
- Version in the URL (`/v1`, `/v2`) or via headers when you truly need a hard cut — but still run both during migration
- Log/metric "old shape still in use" before you delete anything
- Document the deprecation window; do not leave dual support forever without a kill date

---

### Notes: Database Migration

The database is the shared mutable surface between every version you have running and every version you might roll back to. Strategies give you zero downtime at the *process* layer; expand/contract is what makes that safe at the *data* layer.

**Principle: append-only schema history**

People sometimes say "immutable infrastructure." For schemas, the useful idea is narrower: **every running version must be able to read the current schema.** You extend the schema; you do not reshape it under live readers. Contract only when those readers are gone.

**Full expand → contract sequence (with the missing steps)**

```
1. EXPAND     Add nullable columns / new tables / new indexes
              Old code ignores them. New code may dual-write.

2. MIGRATE    Backfill in batches. Resumable. Idempotent.
              Measure throughput on a prod-sized copy first.

3. ENFORCE    Add constraints safely (e.g. Postgres NOT VALID → VALIDATE)
              Set lock_timeout so a blocked migration fails fast instead of hanging traffic.

4. SWITCH     New code reads/writes the new shape only
              Ideally behind a feature flag so you can flip reads back without redeploying schema.

5. ROLLBACK   Keep the old column through a confidence window
 WINDOW      Blue still exists / old pods might still appear / reports still lag.

6. CONTRACT   DROP / rename-away only after:
              - no app version reads it
              - no batch job / CDC / warehouse / BI extract reads it
              - rollback window has passed
```

**Renames are expand/contract in disguise**

```sql
-- ❌ one-shot rename under live traffic
ALTER TABLE orders RENAME COLUMN total TO total_amount;

-- ✅ multi-step
-- expand:  ADD total_amount
-- dual-write both; backfill
-- switch reads to total_amount
-- contract: DROP total
```

**Decouple migrations from app deploys**

| Change | When it runs |
|---|---|
| Expand migration | *Before* (or at least independently of) the code that requires the new column |
| App deploy that writes/reads new shape | After expand is done everywhere |
| Contract migration | Separate change, after rollback window — never bundled with the feature PR |

Shipping "migration + code that depends on it + drop old column" in one pipeline is how you invent an unrecoverable rollback.

**Consumers you do not own**

Before any `DROP`:

- Analytics warehouses / dbt models
- CDC streams (Debezium, etc.) and downstream search indexes
- Admin reports, Metabase/Looker queries
- Third-party integrations reading via API or DB replicas
- Forgotten cron jobs and one-off scripts

If any of them still select `total`, contract breaks them even when the app is clean.

**Backfill discipline**

- Batch by primary key ranges (or another stable cursor), not one giant `UPDATE`
- Make each batch idempotent (`WHERE new_col IS NULL`)
- Checkpoint progress so a killed job resumes
- Watch replication lag, lock waits, and WAL growth
- Rehearse on a production-sized clone — row count and index shape change everything

**Postgres constraint pattern (safe enforce)**

```sql
-- add without validating existing rows (avoids long exclusive locks on huge tables)
ALTER TABLE orders
  ADD CONSTRAINT orders_total_amount_not_null
  CHECK (total_amount IS NOT NULL) NOT VALID;

-- validate in the background-friendly way
ALTER TABLE orders VALIDATE CONSTRAINT orders_total_amount_not_null;

-- only then consider a real NOT NULL if you still need it
SET lock_timeout = '2s';  -- fail fast if blocked
```

**Payments / money-adjacent extras**

Schema safety is not enough when the domain is money:

- Idempotency keys on every charge/refund/transfer
- Ledger-style append-only entries rather than "update balance in place" when you can
- Reconciliation jobs that compare ledger totals vs external PSP / bank reports
- Dual-write periods must leave an audit trail of which path wrote what

**Tie-back to strategies**

| Strategy | Why migrations still bite |
|---|---|
| Rolling | v1 and v2 hit the same DB for the whole rollout |
| Blue-Green | Flip-back to Blue fails if Green already contracted |
| Canary | Small % on v2 still shares the DB with 95% on v1 |
| Feature flags | Code paths diverge; schema must support both paths until the flag is removed |

Bottom line: **deployment strategy chooses how traffic moves; expand/contract chooses whether the database survives that movement.**