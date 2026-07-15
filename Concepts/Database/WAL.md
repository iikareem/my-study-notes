---
tags:
  - database
  - postgresql
  - replication
  - wal
---

## **Write-Ahead Logging — The Idea That Makes Databases Survive Everything**

This one underlies almost every database you have ever used. PostgreSQL, MySQL, SQLite, Kafka, etcd — they all depend on this single idea for durability and recovery. It's one of those concepts where once you understand it, you realize it's doing an enormous amount of invisible work on your behalf every single day.

---

## Start from zero

Imagine you're building a database from scratch. You need two things to be true:

1. **Fast writes** — users can't wait minutes for every insert
2. **Durability** — if the power goes out mid-write, data isn't corrupted or lost

These two goals are in direct tension with each other.

The reason is physics. Your database's actual data lives on disk — in pages, in B-tree nodes, in heap files. To update a row, you need to:

- Find the right page on disk
- Load it into memory
- Modify it
- Write the whole page back to disk

That's slow. And if the power dies between loading and writing back — your page is half-updated. Your data is corrupted. The database is broken.

The naive solution is to write every change synchronously to disk before confirming to the user. This is correct but so slow it's unusable at any real scale.

**Write-Ahead Logging is the solution to this exact tension.**

---

## The core idea

Before you modify anything in the actual data files, you first write a description of the change to a **separate, append-only log file.** Only after that log entry is safely on disk do you confirm the write to the user.

The actual data pages get updated lazily — in the background, whenever convenient.

```
User: INSERT INTO orders (id, amount) VALUES (42, 100)

Step 1: Write to WAL log on disk:
        "Transaction 7: INSERT row (42, 100) into orders table"
        [fsync - force to disk]

Step 2: Confirm to user: "INSERT successful"

Step 3: (sometime later, in background)
        Update actual orders table data page in memory
        Eventually flush that page to disk
```

The user gets a fast confirmation. The data is safe. The actual data files are updated lazily.

---

## Why this works for recovery

When the database restarts after a crash, it does two things:

**Redo:** Scan the WAL log forward from the last checkpoint. For every committed transaction whose changes hadn't made it to the actual data files yet — replay those changes. The log tells you exactly what to write where.

**Undo:** For every transaction that was in-progress when the crash happened — that never committed — roll back its partial changes using the log entries in reverse.

The WAL log is the **ground truth.** The actual data files are just a cache of what the log says should be there. If they disagree, the log wins.

This is why WAL gives you durability: as long as the log entry made it to disk before the crash, the change is recoverable. It doesn't matter that the actual data page was still dirty in memory when the power went out.

---

## Why appending is fast

The WAL log is **append-only.** You only ever write to the end of it. You never seek to a random position and update something in the middle.

This is enormously significant for performance:

On a spinning disk, random writes are slow because the disk head has to physically move to different locations. Sequential writes — always appending to the end — are 10-100x faster because the head stays in one place.

On SSDs, random writes cause write amplification and wear out cells faster. Sequential writes are cleaner and faster.

The WAL trades random writes to data pages (slow) for sequential appends to the log (fast), then does the random writes lazily in the background when the system has time. This is the fundamental performance trick.

---

## Checkpoints — keeping the log from growing forever

If you only ever append to the log, it grows forever. And on recovery, you'd have to replay the entire history of the database from the beginning — which would take forever.

The solution is **checkpoints.** Periodically, the database:

1. Flushes all dirty pages from memory to the actual data files
2. Writes a checkpoint record to the log: "as of this point, all previous log entries are reflected in the data files"
3. The log entries before the checkpoint are now redundant — they can be archived or deleted

On recovery, you only need to replay the log **from the last checkpoint forward.** Everything before that is already in the data files.

```
Log: [old entries] [CHECKPOINT] [entry] [entry] [entry] [CRASH]
                       ↑
              Only replay from here
```

PostgreSQL calls this process **CHECKPOINT** and runs it automatically. You can tune how frequently it runs — more frequent means faster recovery but more background I/O during normal operation.

---

## WAL beyond single-node durability: replication

Once you have a WAL, you get **replication almost for free.**

If you're already writing every change to a sequential log — just send that log to another machine. The replica reads the log entries and applies them in order, keeping its data files in sync with the primary.

This is exactly how PostgreSQL streaming replication works. The primary ships WAL segments to replicas. Replicas replay them. The replica is essentially doing the same recovery process the primary would do after a crash — just continuously, in real time, on a live system.

```
Primary:  [write to WAL] → [send WAL to replica] → [update data files]
Replica:  [receive WAL]  → [replay WAL]           → [update own data files]
```

The replica is always slightly behind — it's replaying a log that the primary is still writing to. How far behind is the **replication lag.** Under heavy write load, the replica can fall behind faster than it can catch up — a failure mode worth monitoring explicitly.

---

## WAL beyond databases: Kafka

Kafka is architecturally a distributed WAL. That's not a metaphor — it's literally what it is.

Producers append messages to a log. Consumers read from the log at their own pace using offsets. The log is the source of truth. Nothing is ever updated in place.

The reason Kafka is fast for the same reason databases are fast with WAL: sequential disk writes are dramatically faster than random writes. Kafka is essentially a system built entirely around the performance property of append-only logs — extended to a distributed, replicated, multi-consumer setting.

Understanding WAL makes Kafka's design feel inevitable rather than arbitrary.

---

## The deeper principle

WAL is an instance of a pattern that appears everywhere in systems design:

**Convert random operations into sequential ones by adding an indirection layer.**

Random writes to data pages → sequential appends to a log, with pages updated lazily. Random updates to state → sequential log of deltas, with state reconstructed from the log.

You see this same structure in:

- **Event sourcing** — store events (the log), derive current state by replaying them
- **CRDT operation logs** — broadcast operations, apply them in any order
- **Blockchain** — an append-only ledger where current state is derived from history
- **Git** — a DAG of immutable commits; current state is the result of replaying them

The common thread: **immutable, sequential history is easier to make durable, replicable, and recoverable than mutable, randomly-updated state.**

WAL is where this insight lives closest to the metal — the place where the tradeoff between fast writes and durable storage gets resolved in the foundation of almost every data system you will ever use.