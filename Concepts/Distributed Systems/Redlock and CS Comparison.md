Redlock and a coordination service both use "majority", so the difference is in **who runs the majority and what it produces**.

## The core difference: who coordinates

**Redlock: the client counts the votes.** The 5 Redis instances don't know about each other. Your client asks each one separately and counts the OKs itself.

```
        Client ──► Redis 1   (independent, no talking between them)
        Client ──► Redis 2
        Client ──► Redis 3
        Client ──► Redis 4
        Client ──► Redis 5
```

**etcd / ZooKeeper: the servers run consensus among themselves.** The client sends one request to the cluster. The servers replicate it through a leader, agree on one ordered log, and only then reply.

```
        Client ──► Leader ──► Follower 2
                         └──► Follower 3     (servers agree with each other)
```

Everything else follows from that.

## What this changes

||Redlock|etcd / ZooKeeper|
|---|---|---|
|**Majority is decided by**|The client, by counting replies|The servers, through consensus|
|**Order of events**|No global order. Each instance only knows its own key|One agreed log, so every lock grant has a position|
|**Fencing token**|None available|Revision / zxid / sequence number|
|**Expiry**|Each instance expires the key by its **own clock**|Expiry is itself a replicated write, so all nodes agree it happened and in what order|
|**Safety depends on clocks**|Yes, on bounded drift and no clock jumps|No for safety (leases still use timeouts for liveness)|
|**Instance restarts**|A restarted Redis can come back empty and forget its lock|Data is written to disk and replicated before the reply|
|**Contention**|Split votes are possible (A gets 2, B gets 2, nobody wins), so clients retry with random delay|The leader serializes requests, so there is one winner|
|**Waiting for a lock**|Polling and retrying|Watches notify the next waiter|

## The failure that shows the difference

**Redlock:** one instance restarts and forgets.

```
1. A locks Redis 1, 2, 3 → majority, A is working
2. Redis 3 crashes and restarts empty (no persistence, or the last writes weren't flushed)
3. B asks all 5: Redis 3, 4, 5 say OK → B also has a majority

A and B both hold the lock.
```

Nothing in the algorithm stops this, because the instances can't coordinate. The mitigation is operational (persist with fsync, delay restarts for at least one TTL), not structural.

**etcd:** the same crash.

```
1. A's lock is committed on a majority of the nodes' disks
2. One node crashes and restarts
3. It rejoins, catches up from the leader, and agrees on the same history
4. B's request: denied, because the log says A holds it
```

The cluster can't disagree with its own history, because the servers agree before they reply.

## What the move gives you, in one list

1. **A reliable fencing token** (revision), which Redlock can't produce.
2. **Safety that doesn't depend on clocks.**
3. **No lost locks** from restarts or failover.
4. **Watches** instead of polling.
5. **Leader election and membership** as built-in recipes.

## What it does not change

A paused client still loses its lease without knowing, so you still fence at the resource. Both systems have this problem equally.

## When the transition makes sense

- You need **fencing tokens** and Redlock can't give them.
- You saw or fear **double holders** after restarts or clock changes.
- You need **leader election** or watches.
- You're running 5 independent Redis servers only for Redlock. That is about the same operational burden as a 3-node etcd cluster, so you might as well get the stronger guarantees.

If duplicates are harmless or already caught by an idempotency key, none of this is needed, and a single Redis lock or the DB claim is enough.



 that's the right summary. Three small precisions:

**1. "Relies on process memory" is slightly too strong.** Redis can persist to disk (AOF or RDB snapshots), but by default it writes asynchronously, so recent writes can be lost in a crash. Even with `appendfsync always`, which makes each instance durable, you've only fixed the "forgot after restart" case. The instances still don't agree with each other, so there is no shared history or order.

**2. Durability is only half of the difference.** The other half is **agreement**:

||Redlock|etcd / ZooKeeper|
|---|---|---|
|Durability|Optional, you configure it|Built in: a write is on disk before it's acknowledged|
|Agreement between nodes|None, the client counts votes|Consensus (Raft / ZAB) among the servers|
|Global order of events|No|Yes, which gives fencing tokens|

**3. "Sync up with the leader's history" is right.** A node that crashes and restarts reloads its log from disk and catches up from the leader, so it can't contradict what the cluster already committed. A Redlock instance has nobody to catch up from.

So your last sentence is the best way to say it: **in Redlock the client is the coordinator, and in a coordination service the servers coordinate themselves.** Everything else (lost locks after restarts, no fencing tokens, clock dependence) follows from that one design choice.