# Read Replicas: Shared Storage (Neon) vs. Physical Storage (Classic)

## 1. What is a read replica?

A read replica is another database instance that serves **read-only queries**, so the primary can focus on writes. Its goal is to scale reads horizontally.

There are two very different ways to build one. The difference is **what the replica owns**:

||Type A: Compute-only replica|Type B: Full physical replica|
|---|---|---|
|Owns its own CPU/RAM|Yes|Yes|
|Owns its own copy of the data on disk|**No** (shares storage with the primary)|**Yes** (full copy)|
|Example|Neon, Aurora|Classic PostgreSQL streaming replication, RDS read replicas|

---

## 2. Type A: Compute-only replica (separated compute and storage)

### Architecture

In Neon, storage and compute are separate layers:

```
        ┌───────────┐     ┌───────────┐     ┌───────────┐
        │  Primary  │     │ Replica 1 │     │ Replica 2 │   <- Compute (CPU + RAM
        │ (compute) │     │ (compute) │     │ (compute) │      + buffer cache)
        └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
              │ WAL             │ WAL stream      │
              ▼                 ▼                 ▼
        ┌─────────────────────────────────────────────┐
        │   Safekeepers (durable WAL)                 │
        └──────────────────────┬──────────────────────┘
                               ▼
        ┌─────────────────────────────────────────────┐
        │   Pageservers + object storage              │  <- ONE shared copy of data
        │   (pages rebuilt from WAL at any LSN)       │
        └─────────────────────────────────────────────┘
```

### Key idea

A replica is **just a new compute node**. It can have a different CPU size and RAM than the primary. There is no new storage and no data copy. It reads the _same_ data the primary uses.

### How it stays in sync

1. The primary writes WAL (Write-Ahead Log). WAL goes to the safekeepers.
2. The replica receives the **WAL stream** from the safekeepers.
3. The replica uses that WAL to keep its **buffer cache** correct:
    - If a changed page is **in the replica's buffer cache**, the WAL is applied to it (or the cached page is invalidated) so it stays fresh.
    - If the page is **not in the cache**, nothing is done. The WAL is not applied locally, because the storage layer already has it.
4. When a query needs a page that is not cached, the replica asks the pageserver: _"give me page X as of LSN Y"_. LSN is the log position the replica has replayed up to, so the replica sees a consistent snapshot.

So "synced using WAL" is correct, but the WAL only keeps the **cache** consistent. The replica never rewrites data files.

### Properties

- Starts in seconds, with no base backup and no data copy
- Cheap: you pay for compute, not for another copy of the data
- Can scale to zero when idle
- Easy to size independently (for example a big replica for analytics)

---

## 3. Type B: Full physical replica (separate storage)

### Architecture

```
   ┌───────────────┐   WAL stream   ┌───────────────┐
   │    Primary    │ ─────────────► │    Replica    │
   │  CPU + RAM    │                │  CPU + RAM    │
   │  ┌─────────┐  │                │  ┌─────────┐  │
   │  │  Disk   │  │                │  │  Disk   │  │
   │  │ (full   │  │                │  │ (own    │  │
   │  │  data)  │  │                │  │  full   │  │
   │  └─────────┘  │                │  │  copy)  │  │
   └───────────────┘                │  └─────────┘  │
                                    └───────────────┘
```

### Key idea

The replica has its **own disk with a complete copy** of the database. It receives WAL from the primary and **replays it onto its own data files**, so it stays a byte-for-byte physical copy of the primary.

### How it works

1. Initial setup: take a **base backup** (copy the whole database) to the replica.
2. The primary streams WAL to the replica continuously.
3. The replica **replays every WAL record**, changing its own data files.
4. Queries read from the replica's own disk and its own buffer cache.

### Properties

- Reads are fast and predictable: data is local, with no network hop per page miss
- The replica is independent. If the primary's storage dies, the replica still has the data and can be promoted
- Setup is slow for large databases (a full copy is needed)
- Storage cost grows by one full copy per replica

---

## 4. Side-by-side comparison

|Aspect|Compute-only (Neon-style)|Physical (classic)|
|---|---|---|
|Storage per replica|None (shared)|Full copy|
|Storage cost|~1x regardless of replica count|N x data size|
|Provisioning time|Seconds|Minutes to hours (base backup)|
|Sync mechanism|WAL keeps the cache fresh, missing pages fetched from the pageserver|WAL replayed onto local data files|
|Cold start|Cold cache, so early queries are slow|Warm disk, only the cache is cold|
|Read latency on cache miss|Network call to the pageserver|Local disk read|
|Independence from primary|Shares the storage layer|Fully independent copy|
|Failover/DR|Storage layer must be durable|Replica can be promoted to primary|
|Scale to zero|Yes|Not usually|
|Replica sizing|Fully flexible|Usually similar to the primary (disk must hold everything)|

---

## 5. Bottlenecks

### Compute-only replica bottlenecks

- **Shared storage is the choke point.** Every replica hits the same pageservers, so adding replicas does not add storage throughput.
- **Cache-miss latency.** A miss means a network round trip, so a replica with a small or cold cache can be slow.
- **Cold starts.** A new or restarted replica has an empty buffer cache.
- **Replication lag.** The replica can fall behind the WAL stream, so reads can be slightly stale.
- **Write rate of the primary.** A heavy write workload creates a lot of WAL for every replica to process.
- **Shared failure domain.** If the storage layer has a problem, all replicas are affected.

### Physical replica bottlenecks

- **Single-threaded WAL replay.** The replica must apply every change. Under heavy writes it falls behind, and lag grows.
- **Disk I/O contention.** Replay writes and read queries compete for the same disk.
- **Recovery conflicts.** Replay may need to remove rows that a long-running read query still needs, so the query gets cancelled or replay is delayed.
- **Storage cost and size.** Every replica needs the full dataset on fast disks.
- **Slow provisioning.** A big database makes a new replica slow to create.
- **Load on the primary.** More replicas means more WAL senders and more WAL retained on the primary.

---

## 6. When to use which

### Choose compute-only (Neon-style) when:

- You want **cheap, fast** read scaling without paying for copies of the data
- Replicas are **spiky or temporary** (reporting, analytics, preview environments)
- You want **different replica sizes** (for example a big-RAM replica for heavy queries)
- Your read workload is **cache-friendly** (hot working set fits in RAM)
- You value **scale-to-zero** and fast provisioning

### Choose physical replicas when:

- You need **consistently low read latency**, with no network hop per page miss
- Your working set is **larger than RAM**, so many reads hit disk and you want local disk
- You need **disaster recovery**: an independent copy you can promote
- You need **a replica in another region** that survives problems with the primary's storage
- You run **self-managed or traditional** infrastructure with no storage/compute separation

---

## 7. Mental model

> **Compute-only replica** = another _brain_ looking at the _same notebook_. **Physical replica** = another brain with its _own photocopy of the notebook_, which it must keep updating line by line.

Neither is "better". The choice is about **cost, speed to create, and latency versus independence and resilience**.

---

## 8. Quick note on terminology

- **WAL**: Write-Ahead Log. Every change is written here first, and it is the basis for replication.
- **Buffer cache**: pages held in RAM, so the replica avoids going to storage.
- **LSN**: Log Sequence Number, a position in the WAL. It gives a consistent view of the data.
- **Pageserver**: Neon's storage service that serves page versions on request.
- **Safekeeper**: Neon's service that durably stores WAL.