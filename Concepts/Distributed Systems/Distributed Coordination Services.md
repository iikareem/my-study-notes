# Coordination Services (etcd, ZooKeeper): Must-Know Notes

## 1. What they are

A **small, strongly consistent, replicated key-value store** (3 or 5 nodes) built on consensus (etcd uses **Raft**, ZooKeeper uses **ZAB**).

- A write is accepted only when a **majority (quorum)** stores it.
- Lose the majority and writes stop. They choose **consistency over availability**.
- They don’t make services scale by themselves. They give many nodes a **consistent, fault-tolerant source of truth** (who is leader, who is alive, what the config is), replicated so no single node failure loses it, and that agreement is what makes horizontal scaling safe
- Use them for **small, critical metadata** (kilobytes), never for bulk data or logs.

## 2. Why not a normal DB or Redis?

| Setup                        | Problem                                                                                                                                                                                        |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Single node                  | Single point of failure                                                                                                                                                                        |
| Async replication + failover | The primary acknowledges a write, crashes before replicating it, and the promoted replica doesn't have it. This means **lost acknowledged writes** and possible **split brain** (two leaders). |
| Sync replication             | Writes block if the replica is down, and you still need a safe way to choose the new primary                                                                                                   |
| **Consensus (Raft/ZAB)**     | Safe, but slower, and unavailable without a majority                                                                                                                                           |

## 3. The four primitives

Learn them in this order:

|#|Primitive|One-line meaning|
|---|---|---|
|1|**Compare-and-swap (CAS)**|Atomic "write only if condition holds", so only one racer wins|
|2|**Lease / ephemeral key**|A key with a timer. It is deleted if its owner stops renewing (owner died).|
|3|**Watch**|Get **notified** when a key changes instead of polling|
|4|**Versioning (revision / zxid)**|A number that only goes up, used for ordering and **fencing tokens** to reject stale clients|

**Lease vs watch:** the lease _causes_ the event (key disappears), and the watch _delivers_ it to others.

## 4. Patterns built from them

| Pattern                                   | Primitives                                                                                   |
| ----------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Config** (change live on all instances) | Watch                                                                                        |
| **Leader election**                       | CAS (one winner) + lease (dies with leader). Watch is optional and only makes failover fast. |
| **Distributed lock**                      | CAS + lease, plus a **fencing token** if correctness matters                                 |
| **Service discovery / membership**        | Lease (register) + watch (observe)                                                           |

**Fencing token idea:** a lock holder can pause past its lease, wake up, and still think it holds the lock. Each grant gets an increasing number, and the resource rejects writes with an older number.

## 5. etcd vs ZooKeeper

Same ideas, different mechanics.

|Idea|etcd|ZooKeeper|
|---|---|---|
|Data model|Flat key-value|Tree of znodes|
|CAS|`txn` (if condition, then put)|`create` and `setData` with expected version|
|Auto-expiry|**Lease** (separate object)|**Ephemeral znode** (tied to the session)|
|Watch|Continuous stream, resumable from a revision|One-time trigger, re-register after firing|
|Ordering|Revision|zxid|

## 6. How to pick: the decision guide

**Start from what you need, and use the simplest tool that is safe.**

|Need|Pick|
|---|---|
|1 instance, static config|Env vars or a config file|
|Many instances, config rarely changes|Kubernetes ConfigMap or **SSM Parameter Store**, then restart or redeploy|
|Config that must change **live everywhere**|etcd or ZooKeeper watch, or AppConfig|
|Singleton job or worker where the **work lives in a DB**|**Conditional `UPDATE ... WHERE`** in that DB (CAS + `locked_until` as lease + `version` as fencing). `FOR UPDATE SKIP LOCKED` for job queues.|
|Lock where a duplicate is **harmless** (efficiency only)|Redis lock|
|Lock where a duplicate causes **real damage**(double refund)|Consensus-backed lock or **DynamoDB conditional write**, plus a **version check and idempotency key** at the resource|
|Leader election on Kubernetes|**Kubernetes Lease** (etcd underneath)|
|Coordinating services with **no shared DB**, or sub-second failover, or electing a DB primary|etcd or ZooKeeper|

**Key rules:**

1. If the work already lives in a database, use a conditional UPDATE in that database as the lock.
2. Run lock queries on the **primary**, never on an async replica, and use the **DB clock**, not app-server clocks.
3. A lock is only one layer. Make operations **idempotent** and check **versions** at the resource, so correctness doesn't depend on the lock alone.
4. One instance with static config means a coordination service is overkill. You'd just move the SPOF to etcd unless you run a 3-node cluster.

## 7. AWS note

**SSM Parameter Store is only a config store.** It has no lease, no CAS, no watch, and no ordering. It is great for config and secrets, but **you can't build locks or leader election on it**. For those, use **DynamoDB conditional writes** (CAS, TTL as lease, version as fencing token) or Kubernetes Leases.

## 8. You mostly use it indirectly

- **Kubernetes** stores all cluster state in etcd.
- **Kafka** uses it (ZooKeeper, or KRaft in newer versions) for controller election and membership.
- **Postgres HA tools** (Patroni) use etcd to decide the single primary.

## 9. Interview lines

> **Definition:** "A coordination service gives a cluster a strongly consistent source of truth for small metadata, using consensus like Raft. Through CAS, leases, watches, and revisions, it enables leader election, locks, config distribution, and service discovery. It favors consistency over availability."

> **Choosing:** "I don't usually run etcd myself. I use it through Kubernetes or Kafka. For my own services, if the work lives in a DB, I use a conditional update as the lock. I use Redis when the lock is only an optimization, and consensus or DynamoDB conditional writes plus fencing and idempotency when duplicates would cause real damage."

> **Locks:** "A lock is CAS plus a lease, but a paused client can outlive its lease, so correctness comes from fencing tokens and idempotency at the resource."

If you'd like, I can turn this into a downloadable file, or quiz you on it with a few interview-style questions.