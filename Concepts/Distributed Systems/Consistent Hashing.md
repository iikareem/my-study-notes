---
tags:
  - caching
  - consistent-hashing
  - distributed-systems
  - sharding
  - system-design
---

## **Consistent Hashing — Why Rescaling Your System Shouldn't Destroy Everything**

This one is foundational to how almost every large distributed system works under the hood — Redis Cluster, Cassandra, DynamoDB, CDNs, load balancers. Most engineers use these systems without understanding the elegant idea that makes them rescalable without catastrophic data movement.

---

## Start from zero

Say you have a cache with 3 nodes and a million keys distributed across them. The classic way to assign a key to a node is **modulo hashing:**

```
node = hash(key) % number_of_nodes
```

Simple. Fast. Works perfectly — until you add or remove a node.

You go from 3 nodes to 4. Now:

```
hash(key) % 3  →  completely different from  hash(key) % 4
```

Almost every key maps to a different node. You've **invalidated nearly your entire cache simultaneously.** Every request misses. Every miss hits your database. Your database falls over.

This isn't hypothetical — it's a real failure mode that has taken down production systems during what should have been a routine scaling operation.

The problem: modulo hashing has **no locality.** Adding one node reshuffles everything.

---

## The insight behind consistent hashing

What if instead of mapping keys to nodes directly, you mapped **both keys and nodes** onto the same abstract space — and assigned each key to the nearest node in that space?

Imagine a circle — a ring — numbered from 0 to 2³²-1 (the range of a 32-bit hash).

```
        0
    ┌───────┐
    │       │
2³²─┤       ├─ 1
    │       │
    └───────┘
      2³²/2
```

You hash each **node** to a position on this ring. You hash each **key** to a position on this ring. A key belongs to whichever node is next **clockwise** from the key's position.

```
Ring positions:
  Node A at 14°
  Node B at 130°
  Node C at 247°

Key X hashes to 80°  → belongs to Node B (next clockwise)
Key Y hashes to 200° → belongs to Node C (next clockwise)
Key Z hashes to 300° → belongs to Node A (next clockwise, wrapping around)
```

---

## Why this survives adding and removing nodes

**Adding a node D at 190°:**

```
Before: Key Y at 200° → Node C at 247°
After:  Key Y at 200° → Node D at 190°... wait, D is at 190, Y is at 200°
        so next clockwise from 200° is still Node C at 247°

Keys between 130° and 190° that were going to C now go to D.
Everything else is completely unchanged.
```

Only the keys in the arc **between the new node and its predecessor** need to move. Everything else stays exactly where it is.

Remove a node: only the keys that were assigned to that node need to redistribute — to the next node clockwise. Nothing else moves.

In modulo hashing, adding 1 node to 3 reshuffles ~75% of keys. In consistent hashing, adding 1 node to 3 reshuffles ~25% of keys — exactly the right fraction and no more.

---

## The problem with basic consistent hashing: uneven distribution

If you just hash node identifiers to ring positions, you get unlucky clustering. Three nodes might land near each other, leaving one huge arc and two tiny ones. One node handles 70% of traffic, two handle 15% each.

The fix is **virtual nodes (vnodes).**

Instead of placing each physical node once on the ring, you place it **many times** — each physical node gets say 150 virtual positions on the ring, each with a different hash.

```
Node A: positions at 14°, 67°, 203°, 318°, ...  (150 positions)
Node B: positions at 41°, 89°, 176°, 290°, ...  (150 positions)
Node C: positions at 22°, 115°, 251°, 337°, ... (150 positions)
```

Now the ring has 450 points spread across it. Each physical node owns roughly equal arcs in aggregate — even if individual arcs vary, they average out across 150 positions.

Virtual nodes also make **heterogeneous clusters** easy. A more powerful node gets more virtual positions — proportionally more of the ring — so it handles proportionally more traffic. A node with 2x RAM gets 2x the vnodes.

---

## Where you actually see this

**Cassandra** uses consistent hashing with vnodes as its core data distribution mechanism. When you add a node to a Cassandra cluster, it claims virtual positions on the ring and receives data from neighboring nodes for those positions. The rest of the cluster is untouched.

**Redis Cluster** uses a variant called **hash slots** — 16,384 fixed slots around the ring, with nodes owning ranges of slots. Adding a node means migrating some slots to it. The slot abstraction is consistent hashing with a fixed granularity.

**CDNs** use consistent hashing to decide which edge cache node handles a given URL. When a cache node goes down, only the keys assigned to it need to be re-fetched from origin — not the entire cache.

**DynamoDB** uses consistent hashing internally for partition assignment, combined with additional mechanisms for load balancing hot partitions.

---

## The subtler guarantee: predictable impact

Beyond the efficiency of data movement, consistent hashing gives you something more valuable: **predictability.**

With modulo hashing, you cannot reason about what a scaling operation will do to your system. It reshuffles everything and you just hope the database survives the cache miss storm.

With consistent hashing, you can calculate exactly how much data moves when you add or remove a node. You can pre-warm the new node before switching traffic. You can rate-limit the data migration. You can reason about the operation before you execute it.

This is the difference between a scaling operation being a controlled procedure versus a dangerous gamble.

---

## The thing most engineers miss

Consistent hashing is often taught as a performance optimization — fewer cache misses during rescaling. That framing undersells it.

The deeper value is that it **decouples the number of nodes from the key assignment logic.** In modulo hashing, the number of nodes is baked into every key's location — change the node count and the entire mapping is invalid. In consistent hashing, nodes are just points on a ring, and adding or removing points has only local effects.

This is the same principle as **loose coupling** applied to infrastructure topology. Your data distribution strategy shouldn't be globally sensitive to every topology change. Consistent hashing gives you locality of impact — the system property that lets distributed systems actually scale gracefully in production instead of theoretically on a whiteboard.