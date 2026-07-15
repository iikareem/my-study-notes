---
tags:
  - caching
  - companion
  - interview
  - invalidation
  - problem-8
  - problems
  - system-design
  - thundering-herd
---

# Companion — Cache Stampede, Avalanche & Invalidation

**Domain:** Caching & Memory Architecture  
**Topics:** stampede · avalanche · invalidation · TTL jitter

**Pairs with:** [[We added a cache and our system got slower and more broken than before]] · [[Problems MOC]]

Caching often fails for three different reasons that feel the same from the outside (DB overload or wrong data). Name them correctly and the fixes become obvious.

---

## 1. Cache stampede (thundering herd on one key / synchronized expiry)

**What it is:** Many clients miss at once and all rebuild the same value from the DB.

**Typical cause:** Fixed TTL → many keys set together → expire together (e.g. Monday 9am after a quiet night).

**Symptom:** Short, sharp DB spike tied to a popular key or a synchronized expiry wave.

**Fixes (pick by complexity):**

| Approach | Idea |
|----------|------|
| **TTL jitter** | `ttl = base ± random` so expiries smear over time |
| **Single-flight / mutex** | Only one request rebuilds; others wait or serve stale |
| **Probabilistic early expire (XFetch)** | Soft-expire early so one client refreshes before hard miss |

**Rule:** Never ship a high-cardinality cache with a single fixed TTL and no jitter.

---

## 2. Cache avalanche (entire cache layer gone)

**What it is:** *All* (or most) keys disappear at once → every request falls through to the DB.

**Typical cause:** `FLUSHDB`, Redis restart without persistence, cluster failover that drops memory, “clear cache” ops in prod.

**Symptom:** Platform-wide overload, not just one endpoint.

**Fixes:**

| Approach | Idea |
|----------|------|
| **Never flush prod blindly** | Prefer key prefixes + targeted delete |
| **Staggered warm** | Rebuild hot keys gradually after empty cache |
| **Circuit breaker** | If DB error rate spikes, fail fast / degrade instead of hammering |
| **Replica / HA for cache** | Reduce “empty everything” failure modes |

**Stampede vs avalanche**

```
Stampede:  one key (or synced subset) · many concurrent rebuilds
Avalanche: whole cache missing     · full traffic hits DB
```

Same outward symptom (DB dies). Different blast radius and playbook.

---

## 3. Cache invalidation (stale after writes)

**What it is:** DB changed; cache still serves old value until TTL.

**Hard part:** Knowing *which* keys to drop for every write path.

**Strategies:**

| Strategy | Behavior | Tradeoff |
|----------|----------|----------|
| **TTL only** | Eventual freshness | Easy; stale windows |
| **Write-through** | Update DB + cache together | Fresh; more write cost |
| **Write-around** | Write DB; fill cache on next read | Simple; cold after writes |
| **Active delete on write** | App deletes keys it knows | Couples writers to cache layout |
| **Event-driven invalidation** | Domain event → invalidator deletes / warms | Decoupled; small lag |

**Rule:** Prefer invalidation tied to **domain events** when many services/read models share cache keys.

---

## 4. Read / write mental model

```
READ
├── Cache-aside (lazy)   → app checks cache; miss → DB → fill
└── Read-through         → cache library loads DB on miss

WRITE
├── Write-around         → DB only
├── Write-through        → DB + cache
└── Write-behind         → cache first; async DB (fast, riskier)
```

Match strategy to **read/write ratio** and **staleness tolerance**.

---

## 5. Quick checklist before “just add Redis”

1. What is the key design? (user? tenant? query hash?)
2. TTL fixed or jittered?
3. Who invalidates on write — and how do they know the keys?
4. What happens if Redis is empty or down? (circuit breaker / degrade)
5. Is cache treated as **truth** or as **acceleration**? (DB must remain truth)

---

## Key takeaways

| Lesson | Rule |
|--------|------|
| TTL jitter | Spread expiry; avoid synchronized misses |
| Stampede vs avalanche | One-key herd vs full-layer wipe |
| Invalidation | Hardest part — design write→key mapping early |
| Circuit breaker | Protect DB when cache fails |
| `FLUSHDB` | Not a prod ops tool for “refresh” |
| Source of truth | Database stays authoritative |

## Next

- Scenario: [[We added a cache and our system got slower and more broken than before]]
- Back to hub: [[Problems MOC]]
