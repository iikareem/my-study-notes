# Deployment Strategies, API Versioning & WebSocket Gateways

Study notes from a conversation about keeping APIs, databases, and connections consistent during deployments.

---

## 1. Core Idea

Deployment strategy and API versioning solve **two different problems**. Keep them independent.

- **Deployment strategy** (blue-green, rolling, canary): _how do I swap running code safely?_
- **API versioning**: _how do old and new clients keep working while the contract changes?_

During any deployment, **old and new versions of the code run at the same time**. In rolling deployments the overlap lasts minutes. In blue-green there is still a moment of overlap (in-flight requests, cached clients).

> **Rule:** Every change must be compatible with the version running next to it.

---

## 2. Changes to an Existing API Version

### Safe (additive) changes, no new version needed

- Add a response field
- Add an optional request field (the server must have a default when it is missing)
- Add a new endpoint

### Unsafe (breaking) changes, need a new version

- Remove or rename a field
- Change a field's type or meaning
- Make an optional field required
- Change error formats or status codes that clients rely on

### Tolerant reader rule

Each client reads only the fields it knows and **ignores unknown fields**. This is what lets an old client survive a new key. Write it into your API guidelines from day one.

---

## 3. The New-Endpoint Problem in Rolling Deployments

Mid-rollout, some instances have the new endpoint and some do not:

|Instance|Has existing endpoints|Has new endpoint / `/v2`|
|---|---|---|
|Old instances|yes|no|
|New instances|yes|yes|

- Existing endpoints work on every instance, so existing clients are unaffected.
- A request to the new endpoint can hit an old instance and return 404.

**Solution: backend first, client second, gated by a feature flag.**

1. Deploy the backend with the new endpoint or version.
2. Wait until the rollout is healthy (for example `kubectl rollout status` succeeds).
3. The pipeline flips the feature flag.
4. Clients start calling the new endpoint.
5. If something goes wrong, turn the flag **off first**.

### Rolling vs blue-green

|Strategy|Mixed state?|Flag needed?|
|---|---|---|
|Rolling|Yes, for the whole rollout|**Required**|
|Blue-green|No (whole environment is ready before the switch)|Optional but recommended, as an instant off-switch|

Caveat for blue-green: requests already in flight during the switch can still hit the old side.

### Good habit

Use the flag with **both** strategies. The flow is identical; only the reason differs.

---

## 4. Where the Feature Flag Lives

The value lives in a **server-side shared store**. The client reads it **through your backend**, never directly from Redis or etcd.

```
Pipeline ──writes──▶ Flag store (Redis / DB / config service / etcd)
                           │
                           ▼
Client ──GET /config──▶ Backend ──reads──▶ returns { "export_v2": true }
```

### Flag store options

- **Redis**: simple and fast
- **Database table**: easy to audit, fine for small systems
- **etcd / Consul / ZooKeeper**: for many services and change notifications
- **Managed services** (LaunchDarkly, Unleash, Flagsmith, AWS AppConfig): UI, per-user targeting, percentage rollouts, audit logs

### Why not talk to Redis from the client

- Security: Redis and etcd must not be exposed to clients
- The backend can decide per user or per client version
- You can change the store later without changing clients

### Practical points

- Cache flags on the client with a short TTL (30 to 60 seconds). A flipped flag takes up to that TTL to reach everyone.
- **Safe default:** if the fetch fails, treat the flag as "off" and keep the old behavior.
- Cache flags in backend memory for a few seconds instead of reading Redis on every request.
- Mobile apps update slowly, so ship the client code early and let the **server-controlled** flag decide when it is used.
- Optional safety net: the client handles a 404 on the new endpoint by falling back to old behavior (not a replacement for the flag).

---

## 5. API Versioning

### Approaches

|Approach|Example|Trade-off|
|---|---|---|
|URL versioning|`/v1/users`, `/v2/users`|Most common, simple, easy to route at the gateway|
|Header versioning|`Accept: application/vnd.app.v2+json`|Cleaner URLs, harder to debug and cache|
|Date-based|`API-Version: 2026-11-10` (Stripe style)|Very flexible, more engineering|

### Key points

- **One build serves multiple versions.** `v1` and `v2` live in the same codebase and are deployed together, so blue-green and rolling work as usual.
- During a rollout, treat `/v2/*` as "just new endpoints": `/v1` works everywhere, `/v2` only on new instances, so clients wait for the flag.
- **Sequence for a new version:**
    1. Deploy a build supporting both v1 and v2
    2. Wait for 100% rollout
    3. Flip the flag so clients use v2
    4. Migrate clients over time
    5. **Later**, once metrics show nobody uses v1, remove it in a separate deployment
- Track usage per version and announce deprecation with `Deprecation` and `Sunset` headers.

### Rollback caveat

Once clients use v2, you cannot roll the backend back to a build without v2. Either:

- Turn the client flag off first (clients return to v1), then roll back the backend, or
- Roll forward with a fix.

### Do you need a custom gateway?

Usually no.

- **Without a gateway:** the app routes `/v1/*` and `/v2/*` internally. Fine for most teams.
- **With an off-the-shelf gateway** (Kong, NGINX, AWS API Gateway, Envoy): useful for routing versions to different services, per-version rate limits and auth, canary traffic splitting, and per-version metrics.
- Building a _custom_ gateway is rarely worth it.

---

## 6. Database Changes: Expand / Contract

Old and new code share one database, so every schema change must work for both.

1. **Expand:** add the new column or table (nullable or with a default). Old code ignores it.
2. **Deploy** code that writes to both old and new and reads from the new (falling back to the old).
3. **Backfill** existing data.
4. **Contract:** after no running version uses the old column, drop it in a later release.

Example, renaming `name` to `full_name`:

- Release 1: add `full_name`, write to both columns
- Release 2: backfill, read from `full_name`
- Release 3: drop `name`

Never rename or drop in the same release that depends on it. Blue-green's "instant rollback" is only real if the schema stays backward compatible.

---

## 7. WebSocket Gateway Pattern

### Problem

WebSocket connections are stateful. If business services hold the sockets, every deployment disconnects users, which is not acceptable in production.

### Solution: separate connections from business logic

```
Business services ──publish──▶ Redis (Pub/Sub or Streams) ──▶ WS Gateway ──▶ Client sockets
        ▲                                                          │
        └──────────────── client messages (via gateway) ───────────┘
```

- **Business logic** changes often, so it lives in stateless services you can deploy freely.
- **Connections** are expensive to drop, so they live in a thin gateway that changes rarely.

### Gateway responsibilities

**Should:**

- Authenticate the connection
- Keep the socket registry (socket ID to user)
- Subscribe to events and push them to the right sockets
- Forward client messages inbound

**Should not:**

- Contain business rules or interpret payloads
- Treat messages as opaque envelopes (`{ userId, type, payload }`) so new event types need no gateway redeploy

### Production-grade refinements

1. **The gateway still gets deployed sometimes.** Design for it:
    - **Graceful draining:** stop accepting new connections, tell clients to reconnect, close old sockets gradually
    - **Client reconnect with exponential backoff and jitter** (avoids a thundering herd)
    - Roll the gateway slowly, a small fraction at a time
2. **Pub/Sub is fire-and-forget.** Messages are lost if the user is disconnected or the gateway restarts. Options:
    - **Redis Streams** (or Kafka): keep messages, so a reconnecting client says "last seen event 1042" and the gateway replays what it missed
    - Store important events in the database, and let the client fetch missed ones through REST after reconnecting (Pub/Sub is just a "wake up and refresh" signal)
3. **Routing events to the right gateway instance:**
    - **Per-user channel** (`user:{id}`): simple, many subscriptions
    - **Presence map** (`user -> gatewayInstance` in Redis): more efficient at scale, needs TTL, heartbeats, and cleanup on disconnect
4. **Multiple devices and tabs:** map a user to a **set** of sockets.
5. **Protocol versioning applies too.** Keep messages additive, include a version in the handshake (`?v=2`), and ignore unknown message types (tolerant readers again).
6. **Scaling and limits:** autoscale on connection count (not only CPU), and use ping/pong heartbeats to detect dead sockets.

---

## 8. Master Checklist

1. Make changes **additive** whenever possible (no version bump needed).
2. Make clients **tolerant readers**.
3. Deploy **backend before client**.
4. Gate new endpoints and versions behind a **server-controlled feature flag**, flipped by the pipeline after the rollout is healthy.
5. Use **expand/contract** for database changes.
6. Version only for true breaking changes, and keep all supported versions in one build.
7. Track per-version usage, announce deprecation, then retire old versions in a later release.
8. Add **contract tests** (for example Pact) to CI to catch compatibility breaks before deploying.
9. Keep WebSocket gateways thin and stateful, business services stateless, and plan for graceful draining, jittered reconnects, and event replay.

---

## 9. Quick Reference

|Situation|What to do|
|---|---|
|Add response field or optional request field|Just add it, no version needed|
|Add new endpoint (rolling deploy)|Deploy backend, wait for rollout, flip flag|
|Add new endpoint (blue-green)|Switch traffic, verify, flip flag (optional but recommended)|
|Rename, remove, or retype a field|New API version (`/v2`), leave `/v1` untouched|
|Rename a DB column|Expand, dual-write, backfill, contract (3 releases)|
|Roll back after clients use v2|Flip the flag off first, then roll back|
|Deploy business logic with WebSockets|Publish via Redis to the gateway, no sockets dropped|
|Deploy the gateway itself|Drain gracefully, jittered reconnect, replay missed events|

---

## 10. Possible Next Topics

- Concrete example of the client flag check plus the pipeline step that flips it after `rollout status`
- Reconnect-with-replay flow using Redis Streams
- An expand/contract migration in your specific stack