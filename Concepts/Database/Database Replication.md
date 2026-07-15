---
tags:
  - async
  - database
  - ha
  - postgresql
  - replication
  - rpo
  - rto
  - scaling
  - sync
  - system-design
  - wal
---

# Database Replication

Replication copies data so you can **scale reads**, survive **primary failure**, or both. Almost every design choice sits on two axes: **how storage is shared**, and **how strongly the secondary waits before a write is ACKed**.

---

## Table of Contents

1. [RPO and RTO — what they mean](#1-rpo-and-rto--what-they-mean)
2. [Mental model — two axes](#2-mental-model--two-axes)
3. [Shared storage (read replicas)](#3-shared-storage-read-replicas)
4. [Separate storage — two roles](#4-separate-storage--two-roles-not-only-durability)
5. [Replication modes — async, sync, semi-sync](#5-replication-modes--async-sync-semi-sync)
6. [WAL — how changes move](#6-wal--how-changes-move)
7. [Problems and solutions (each on its own)](#7-problems-and-solutions-each-on-its-own)
8. [Comparison](#8-comparison)
9. [Decision tree](#9-decision-tree)
10. [Key takeaways](#10-key-takeaways)

---

## 1. RPO and RTO — what they mean

These two metrics answer different questions after a failure. Do not mix them up.

### RPO — Recovery Point Objective

**Question:** *How much data can we afford to lose?*

Measured as **time** (or commit range) between:

- the last write the business still has after recovery, and  
- the last write the client was told succeeded before the crash  

```
Timeline of writes on primary (async):

  ... W1   W2   W3   W4   W5   💥 primary dies
       │              │
       │              └─ ACKed to clients, but WAL not yet on standby
       └─ already on standby

After failover to standby, only W1–W3 exist.
W4 and W5 are gone → you lost those commits.

RPO ≈ “how far behind was the standby?”
If lag was 5 seconds, RPO budget is often spoken as “~5 seconds of writes.”
```

| RPO target | Plain English | Typical approach |
| --- | --- | --- |
| **RPO = 0** | Never lose an ACKed write | Sync replication, shared storage, or equiv. multi-AZ sync |
| **RPO = seconds** | May lose a few seconds of writes | Async same-region |
| **RPO = minutes** | Acceptable for DR / some analytics | Async cross-region, slower shipping |

**Important:** RPO is about **data**, not downtime. A system can be back online quickly and still have lost writes (low RTO, high RPO).

### RTO — Recovery Time Objective

**Question:** *How long can we be down (or read-only) before we’re writable again?*

```
💥 primary dies ──────── detect ── promote ── DNS/LB flip ── apps reconnect ── ✅ writing
|<--------------------------- RTO window ---------------------------->|
```

| RTO target | Plain English | Typical approach |
| --- | --- | --- |
| **Seconds** | Near-seamless | Shared-storage promote; well-tuned Patroni |
| **~1–2 minutes** | Short blip | RDS Multi-AZ style managed failover |
| **Minutes+** | Manual / carefully reviewed promote | Cross-region DR |

### Together

|                 | RPO                             | RTO                                |
| --------------- | ------------------------------- | ---------------------------------- |
| **Asks**        | How much data lost?             | How long until service returns?    |
| **Unit**        | Time / commits lost             | Clock time offline                 |
| **Improved by** | Sync (or shared disk), less lag | Faster detection, promote, reroute |

Example: async cross-region DR might be **RPO = 30s**, **RTO = 15 min**. Sync Multi-AZ might be **RPO = 0**, **RTO = 90s**.

---

## 2. Mental model — two axes

```
                   STORAGE
                     │
          Shared     │     Separate
          (one volume│     (each node
           of truth) │      owns its disk)
──────────────────────────────────────────
  Read replicas      │  Two roles (often on the
  (primary job =     │  same streaming tech):
   SELECTs)          │   1) Failover / durability
  Aurora / AlloyDB   │   2) Hot standby = also reads
COMPUTE              │
  Primary + N read   │  Primary + standby(s)
  nodes mount same   │  each has full copy;
  storage            │  changes must be shipped
```

\*RDS Multi-AZ: separate volumes, storage-layer sync — standby stays current for **failover** (RPO ≈ 0); that standby classically does **not** serve app reads. Separately, RDS “read replicas” are the ones that scale SELECTs.

| Axis | Shared storage | Separate storage |
| --- | --- | --- |
| **How sync works** | Same volume | Ship WAL / binlog |
| **Can serve reads?** | Yes — that’s the point of read replicas | **Optional** — only if configured as hot standby / read replica |
| **Durability / HA?** | Promote another compute on same volume | Promote a standby that has its own copy |
| **Main cost** | Cloud / proprietary model | Lag, RPO trade-offs, harder failover |

**Don’t conflate “replica” types:**

| Name you’ll hear | Storage | Serves reads? | Main reason it exists |
| --- | --- | --- | --- |
| Shared-storage read replica | Shared | Yes | Scale reads |
| **Failover / Multi-AZ standby** | Separate (often sync) | **Usually no** | Durability + fast promote (RPO→0) |
| **Hot standby / async read replica** | Separate | **Yes** | Scale reads (+ can promote if needed) |

So: separate storage is **not** “durability only.” Durability-only is one *role*. The other role — which you already know — is **every replica serves reads**.

---

## 3. Shared storage (read replicas)

```
        ┌─────────────────────────┐
        │     Shared storage      │
        └───────────┬─────────────┘
              ┌─────┴─────┐
         Primary      Read replica ×N
        (R/W)            (reads)
```

- Writes → primary compute → shared volume  
- Reads → any replica (same storage)  
- Tiny “lag” can still come from **replica cache invalidate** (~ms), not shipping a second copy  
- Promote another compute on the **same** volume → typically **RPO = 0**, RTO in seconds  

---

## 4. Separate storage — two roles (not only durability)

```
┌──────────────┐   WAL / binlog    ┌──────────────┐
│   PRIMARY    │ ───────────────▶  │   STANDBY    │
│  own disk    │                   │  own disk    │
└──────────────┘                   └──────────────┘
```

Same shipping mechanism. **How you use the standby** is what differs.

### Role A — Failover replica (durability / HA)

**Job:** stay ready to become primary. Protect against data loss and long outages.

| | |
| --- | --- |
| **Serves app reads?** | Usually **no** (cold/warm standby, or Multi-AZ standby hidden from the app) |
| **Sync style** | Often **sync** (or storage-layer sync) when you want RPO ≈ 0 |
| **Examples** | RDS Multi-AZ standby; a dedicated HA standby behind Patroni that apps never query |
| **You gain** | Durability + promote path |
| **You do not gain** | Extra read capacity — primary still does (almost) all SELECTs |

This is the “failover replica = durability” case you had in mind — and that framing is right **for this role**.

### Role B — Hot standby / read replica (serves reads)

**Job:** take SELECT traffic off the primary. Same separate copy + WAL stream — but apps are allowed to query it.

| | |
| --- | --- |
| **Serves app reads?** | **Yes** — that’s why you added it |
| **Sync style** | Usually **async** (write path stays fast); lag ⇒ stale reads possible |
| **Examples** | Postgres hot standby; MySQL / RDS read replicas; self-managed streaming replicas in a read pool |
| **You gain** | Horizontal **read** scale |
| **Also possible** | Promote one of them on primary death (second benefit, not the only one) |

```
                    ┌─▶ Read replica 1  (SELECT)
 Primary (R/W) ─────┼─▶ Read replica 2  (SELECT)
   WAL out          └─▶ Read replica N  (SELECT)

 Optional separate Multi-AZ / sync standby → failover only (no SELECTs)
```

### Role A vs Role B (memorize this)

| | Failover replica (A) | Hot standby / read replica (B) |
| --- | --- | --- |
| **Why it exists** | Durability + HA | Scale reads |
| **App reads from it?** | No | Yes |
| **Typical ACK mode** | Sync / Multi-AZ | Async |
| **RPO focus** | Often RPO = 0 | Lag OK; RPO > 0 if you promote under async |
| **Lag visible to users?** | N/A (no reads) | Yes — stale SELECTs |

Many production stacks run **both**: sync Multi-AZ (or sync standby) for durability, **plus** a fleet of async read replicas that all serve reads.

### Shared-storage read replicas vs Role B

Both “all serve reads.” Difference is plumbing, not the product goal:

| | Shared storage (Aurora-style) | Separate hot standbys |
| --- | --- | --- |
| **Copy of data** | One shared volume | Each replica has its own disk |
| **How they stay current** | Read same storage (+ tiny cache notify) | Apply shipped WAL |
| **Read lag** | Usually ms (cache) | Seconds possible under load / distance |
| **Product goal** | Same — **SELECTs on many nodes** | Same |

---

## 5. Replication modes — async, sync, semi-sync

### Async

```
Write → primary durable → ACK client
              ↓ (later)
         ship WAL → standby applies
```

- **RPO:** not zero if primary dies before WAL arrives  
- **Write latency:** unchanged  
- **Use:** DR, cross-region, latency-sensitive writes when some loss is OK  

### Sync

```
Write → primary WAL → ship → standby durable ACK → ACK client
```

- **RPO:** 0 for ACKed writes  
- **Write latency:** + RTT every commit  
- **Use:** payments / regulated data; same AZ or nearby  

Postgres wait strength (weaker → stronger): `remote_write` → `on` (fsync) → `remote_apply`.

### Semi-sync

Wait for one standby ACK; **on timeout, fall back to async** so writes don’t block forever. Near-zero loss in the common case; weaker guarantee when standbys are unhealthy.

| Mode | Primary waits? | RPO on primary death |
| --- | --- | --- |
| Async | No | Can lose recent ACKs |
| Sync | Yes | 0 for ACKed writes |
| Semi-sync | Yes until timeout | Usually 0; worse after degrade |

---

## 6. WAL — how changes move

**WAL** / **binlog** = ordered change log. Local durability and replication both use it.

```
Client write → append WAL (LSN) → durable locally → stream → standby replay
```

| Style | Behavior | Lag |
| --- | --- | --- |
| Streaming | Continuous TCP | ms–seconds |
| Archive shipping | Copy WAL files on a schedule | minutes |

Lag ≈ primary sent LSN − standby replay LSN.

---

## 7. Problems and solutions (each on its own)

Treat each problem separately. A system often needs several of these answers at once, but the **solution belongs to that problem**.

---

### Problem 1 — Primary is overloaded by reads

**What goes wrong:** One node serves all SELECTs and writes; CPU/IO saturates; latency climbs.

**Solution (pick by storage model):**

| Approach | How it solves it |
| --- | --- |
| **Shared-storage read replicas** | Extra compute nodes read the same volume — scale reads with almost no replication lag |
| **Separate-storage hot standbys** | Standbys serve SELECTs from their own copy — scale reads, accept possible staleness (async) |

**Not a solution for this problem alone:** sync replication (that’s about RPO, not read capacity).

---

### Problem 2 — Primary can die and take the database with it

**What goes wrong:** Single failure domain; disk or AZ loss → long outage or permanent data risk.

**Solution:**

| Approach | How it solves it | RPO / RTO flavor |
| --- | --- | --- |
| **Standby + failover** | Promote a copy; reroute apps to it | Depends on sync vs async |
| **Shared storage** | Promote another compute on same volume | RPO ≈ 0, RTO usually seconds |
| **Managed Multi-AZ** | Provider detects, fences, promotes, flips DNS | RPO ≈ 0, RTO ~1–2 min |

**Failover steps (separate storage):** detect → pick best standby (highest replay LSN) → promote → update DNS/LB (**name, not IP**) → apps reconnect.

**Postgres timelines:** promote bumps timeline ID so an old primary that “comes back” cannot quietly continue the old WAL and corrupt the cluster.

---

### Problem 3 — We cannot lose ACKed writes (RPO must be 0)

**What goes wrong:** Under async, client got success, standby never got the WAL, primary dies → those commits vanish forever.

**Solution (independent of “how we scale reads”):**

| Approach | Why RPO = 0 |
| --- | --- |
| **Sync replication** | Client ACK only after standby has the WAL durable |
| **Shared storage** | There is only one storage truth — no second copy to lag |
| **Sync Multi-AZ (e.g. RDS)** | Storage-layer sync before success |

**Trade-off you accept:** higher write latency (sync) or cloud shared-storage constraints.

**Not enough alone:** “we have a replica” — if it’s async, RPO is still > 0.

---

### Problem 4 — Sync makes every write too slow

**What goes wrong:** Every commit waits for standby RTT; p99 write latency or throughput collapses (especially cross-region).

**Solution:**

| Approach | How it solves it |
| --- | --- |
| **Async (same or other region)** | Standby off the critical path — accept RPO > 0 |
| **Semi-sync** | Sync-like when healthy; degrade on timeout so writers don’t wedge |
| **Keep sync local only** | Sync standby in same AZ/region; use **async** for remote DR |

**Rule:** never solve cross-region DR with sync if you care about write latency — pay the ocean on every commit.

---

### Problem 5 — Replica reads are stale / “I don’t see my write”

**What goes wrong:** User writes on primary, immediately reads from a lagging standby → missing or old data (read-your-writes violation). Also dashboards/report lag.

**This is replication lag** — separate from RTO. Lag hurts **freshness**; on failover under async it also sets the **RPO window**.

**Causes (diagnose by side):**

| Side | Examples |
| --- | --- |
| Primary | WAL volume > apply capacity; huge transactions |
| Network | Cross-region RTT, bandwidth, loss |
| Standby | Slow disk/CPU; long queries blocking apply; vacuum conflicts |

**Solutions (each targets a cause):**

| Solution | Addresses |
| --- | --- |
| **Faster standby hardware / IO** | Apply can’t keep up |
| **Break huge transactions into batches** | One giant WAL burst |
| **Don’t use remote replicas for freshness-critical reads** | Cross-region lag is structural |
| **`hot_standby_feedback` (Postgres)** | Queries vs apply/vacuum conflicts |
| **Read-your-writes routing** | After a write, read primary (or only replicas past that LSN) for a short window |
| **Alert on lag SLO** | e.g. warn > 30s, page > 5m — fix before failover RPO explodes |

See also [[Scaling Reads]].

---

### Problem 6 — Failover creates two primaries (split-brain)

**What goes wrong:** Network partition → old primary and new primary both accept writes → two datasets. After heal, one side’s data must be discarded.

**Solution (must run before or as part of promote — not after):**

| Mechanism | How it solves it |
| --- | --- |
| **STONITH / fencing** | Hard-disable old primary (power / network / storage) so it cannot accept writes |
| **Quorum (N/2 + 1)** | Only the majority side may own primary — minority cannot promote |
| **Lease (etcd / ZK / Consul)** | Only the process holding the lease is primary (Patroni-style) |

**Do not:** automate failover with “if ping fails, promote” and nothing else — that *causes* split-brain.

---

### Problem 7 — We need DR in another region

**What goes wrong:** Whole region dies; local sync standby dies with it.

**Solution:**

| Approach | Honest trade-off |
| --- | --- |
| **Async replica in the other region** | Survives region loss; **RPO > 0** (lag across the ocean); **RTO** often minutes (promote + DNS carefully) |
| **Accept RPO in writing** | Product/legal know “we may lose N seconds of writes on regional disaster” |

Sync cross-region is usually the wrong tool for this problem (see Problem 4).

---

### Problem 8 — Apps stay pointed at the dead primary after promote

**What goes wrong:** Standby is primary, but clients still use the old IP → errors or split writes.

**Solution:**

| Approach | How it solves it |
| --- | --- |
| **Stable DNS / cluster endpoint** | Name flips to new primary; apps never hardcode IP |
| **Proxy (HAProxy, PgBouncer, MySQL Router)** | Proxy watches who is primary and routes |
| **Pool retry + short DNS TTL** | Survive the flip window; avoid sticky OS DNS caches |

This is an **RTO** problem (time to usable again), not an RPO problem.

---

### Quick map — problem → solution focus

| Problem | Primary lever |
| --- | --- |
| Too many reads | Read replicas / hot standbys |
| Node/AZ death | Failover + fencing |
| Must not lose ACKs | Sync or shared storage (**RPO = 0**) |
| Writes too slow under sync | Async / semi-sync / local-only sync |
| Stale reads / missing my write | Lag fixes + primary read-after-write |
| Two writers after failover | Fence / quorum / lease |
| Region disaster | Async cross-region (**accept RPO**) |
| Apps stuck on old node | DNS / proxy / retry (**RTO**) |

---

## 8. Comparison

| | Shared storage reads | Async separate | Sync separate |
| --- | --- | --- | --- |
| **Examples** | Aurora, AlloyDB | Postgres/MySQL async | Postgres sync, RDS Multi-AZ |
| **Read scale** | Excellent | Yes (stale possible) | Yes |
| **RPO** | 0 | Seconds+ possible | 0 |
| **Write latency** | Low | Low | + standby RTT |
| **Cross-region** | Special products | Natural | Usually bad idea |
| **Split-brain** | Storage is authority | Must fence | Must fence |
| **Best for** | Read-heavy SaaS | DR + cheap read scale | Zero-loss commits |

**Stacks to recognize:** Patroni + etcd (Postgres HA), Galera / Group Replication (MySQL). What matters: **who is primary** and **how fencing works**.

---

## 9. Decision tree

```
Main goal?
├─ Scale reads, managed OK?
│    → Shared storage read replicas
├─ Survive AZ failure with RPO = 0?
│    → Sync standby or RDS Multi-AZ (nearby)
├─ Survive region failure?
│    → Async cross-region (accept RPO > 0)
└─ Self-managed Postgres?
     → Streaming + Patroni
        (sync only if RPO=0 and low RTT; else async)
```

---

## 10. Key takeaways

| Concept | One-liner |
| --- | --- |
| **RPO** | How much ACKed data you may lose on failure |
| **RTO** | How long until you’re writable again |
| Shared storage | Same volume — fast reads, promote, RPO 0 |
| Separate storage | Own copies via WAL — **two roles**: failover-only (durability) **or** hot standby (serves reads) |
| Failover replica | Durability/HA — usually **does not** serve app reads |
| Hot standby / read replica | **Does** serve reads — scale SELECTs; lag possible if async |
| Async | Fast writes; RPO > 0 possible |
| Sync | RPO 0; every write pays standby RTT |
| Semi-sync | Wait, then degrade on timeout |
| Lag | Stale reads now; larger loss window on async failover |
| Split-brain | Prevent with fence / quorum / lease before promote |
| Solve per problem | Don’t use sync to “scale reads”; don’t use async and claim RPO 0 |

## See also

- [[Scaling Reads vs Writes]] · [[Scaling PostgreSQL to 800 Million Users]]
- [[WAL]] · [[Isolation Level and MVCC]] · [[System Design MOC]]
