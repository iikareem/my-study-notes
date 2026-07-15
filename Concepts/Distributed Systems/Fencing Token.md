---
tags:
  - distributed-lock
  - distributed-systems
  - fencing-token
  - locking
---

# Fencing Tokens Explained

Fencing tokens are a security mechanism used to prevent **race conditions and unauthorized state changes** in distributed systems. Let me break this down:

## The Problem They Solve

Imagine you're building a system where multiple processes or servers might try to modify the same resource at the same time. Without protection, you could get inconsistent or corrupted data.

**Classic example:** Two servers both think they can delete a file:

1. Server A checks: "Does file exist?" → Yes
2. Server B checks: "Does file exist?" → Yes
3. Server A deletes the file
4. Server B tries to delete the same file → ERROR or unexpected behavior

This is a **race condition** — the outcome depends on unpredictable timing.

## How Fencing Tokens Work

A fencing token is a **unique, monotonically increasing number** issued by a central authority. It works like a numbered ticket at the DMV:

1. **Resource holder gets a token**: When a process acquires a resource (like a lock), it receives a unique token number (e.g., token #42)
2. **Token must be presented**: When that process tries to modify the resource, it must include its token
3. **Resource checks the token**: The resource rejects any write operation with an old/invalid token number
4. **New holders get higher tokens**: If the lock is released and re-acquired, the new holder gets a higher token (#43)

## Visual Example

```
Process A acquires lock → gets token #42
  ↓
Process A holds token #42, starts operation
  ↓
Process A crashes/disconnects → lock expires
  ↓
Process B acquires lock → gets token #43
  ↓
Resource receives write from Process A with token #42
  ↓
Resource rejects it (old token) ✗
  ↓
Resource accepts write from Process B with token #43 ✓
```

## Real-World Problem This Solves

**"Zombie processes"** — what if a process thinks it still owns a resource, but the system has already moved on?

Scenario without fencing:

- Process holds a lock but gets delayed (network latency, GC pause)
- Lock automatically expires (timeout)
- Another process takes over and modifies the resource
- Original process finally responds and overwrites the new changes → DATA CORRUPTION

Scenario with fencing tokens:

- Process holds token #42
- Lock expires, new process gets token #43
- Original process tries to write with token #42
- System rejects it → SAFE ✓

## Common Use Cases

- **Distributed databases** (HBase, HDFS, Cassandra)
- **Cloud storage systems** (AWS S3, Google Cloud)
- **Lease-based systems** (Zookeeper, etcd)
- **Distributed file systems** (NFS, GFS)

## Key Insight

Fencing tokens assume there's a **trusted central authority** that hands out monotonically increasing tokens. This prevents "confused" processes from accidentally corrupting shared state, even if they're out of sync with the rest of the system.

Does this clarify the concept? Would you like me to dive deeper into a specific use case or implementation?
