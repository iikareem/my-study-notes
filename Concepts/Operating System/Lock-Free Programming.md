# Lock-Free Programming

## Common Misconception: "Mutexes Always Cause Context Switches"

This is one of the most repeated oversimplifications in systems programming — and it is wrong. Understanding _why_ it is wrong is what lets you reason clearly in interviews and technical discussions.

### Where the misconception comes from

People see this chain and assume it is always true:

```
mutex → blocking → context switch → TLB flush → slow
```

But that chain only describes **one specific case**: a contested mutex where the waiting thread decides to sleep. It is not what always happens.

### The root cause to understand

A context switch happens when **the OS scheduler gets involved**. The OS gets involved when **a thread asks to sleep**. A thread asks to sleep when **it is waiting for something and has nothing else to do**.

A mutex by itself does not cause a context switch. **A sleeping thread causes a context switch.**

The mutex is just the reason the thread decided to sleep. But sleeping is a choice — not all mutex implementations always sleep.

### The full picture

**Uncontested mutex** — the lock is free when you arrive:

- Acquired with a single atomic check in user space
- OS never involved
- No context switch
- No TLB flush
- Essentially free

**Contested mutex, thread spins** — lock is taken, but thread keeps checking in a loop:

- Still no OS involvement
- No context switch
- Burns CPU cycles while waiting
- Used when the wait is expected to be very short

**Contested mutex, thread sleeps** — lock is taken, wait is expected to be long:

- Thread tells the OS: "wake me when this is free"
- OS suspends the thread → saves its registers, flushes pipeline
- TLB may flush → memory accesses get slower until it refills
- Cache lines go cold
- Kernel transition: user space ↔ kernel space
- Later, when lock is released, OS wakes the thread → same cost in reverse
- This is the expensive case everyone warns about

Most modern mutexes do both: **spin briefly first, then sleep if the lock is still held**. This is called a hybrid or adaptive mutex.

### So what is lock-free actually solving?

Not "context switches in general." The precise problem it solves is:

> A thread holding a mutex can be preempted by the OS at any moment. While it sleeps, every other thread waiting for that lock is blocked — even if the work was almost done.

Lock-free removes this risk entirely. The thread **never sleeps, ever**, regardless of contention. It just retries. The OS scheduler cannot block your progress by preempting someone else at the wrong moment.

The real guarantee lock-free gives you is: **no thread can be the bottleneck for other threads by holding a lock while sleeping.**

### How to explain this in an interview

> "Mutexes don't always cause context switches — only when the thread chooses to sleep while waiting. An uncontested mutex is just an atomic check in user space, essentially free. The cost appears when the lock is held and the waiting thread sleeps, which brings in the OS, causes a context switch, and potentially flushes the TLB and caches. Lock-free programming avoids this not by avoiding mutexes specifically, but by ensuring threads never sleep at all — they always keep running, retrying on conflict instead of blocking."

---

## Why Does This Even Exist?

The common claim is "mutexes cause context switches." That is an oversimplification. The real picture:

**Uncontested mutex** — no one else holds it → acquired immediately in user space (Linux futex). No kernel involvement, no context switch. Cheap.

**Contested mutex** — someone holds it → the thread has two options:

- **Spin:** keep checking in a loop. No context switch, but burns CPU while waiting.
- **Sleep:** ask the OS to wake it when the lock is free. This is the context switch — thread goes sleeping, OS scheduler runs something else, then wakes the thread later.

Most production mutexes do both: spin briefly, then sleep. Called a **hybrid/adaptive mutex**.

**So the real cost of a contested, sleeping mutex:**

- Context switch — saves register state, flushes the pipeline
- TLB may flush — every memory access re-walks the page table until it refills
- L1/L2/L3 cache lines go cold
- Kernel transition: user space ↔ kernel space — hundreds of nanoseconds
- Then the reverse when the thread wakes up

**Lock-free programming says:** what if the thread never sleeps — ever?  
No sleep → no context switch → no TLB flush → no kernel involvement.  
The thread always keeps running, just retrying if it loses a CAS race.

The real advantage is not "avoids context switches in general" — it is: **guarantees the thread never sleeps, eliminating tail latency spikes and OS scheduler interference**, which matters most on hot paths and latency-sensitive work.

---

## The Core Idea

Instead of locking a shared resource and letting other threads wait, lock-free design uses **atomic operations** — hardware instructions that are guaranteed to complete as a single, uninterruptible unit.

The most important one is:

### Compare-And-Swap (CAS)

```
CAS(memory_location, expected_value, new_value) → bool
```

It does this atomically:

```
if (*ptr == expected) {
    *ptr = new_value;
    return true;
} else {
    return false;
}
```

If another thread modified `*ptr` between when you read it and when you tried to write it — CAS fails. You retry. No lock needed. The CPU guarantees this check-and-write is one indivisible operation.

Modern CPUs expose this as a single instruction (`CMPXCHG` on x86, `LDREX/STREX` on ARM).

---

## The Spectrum: Lock-Free vs Wait-Free vs Obstruction-Free

|Term|Guarantee|
|---|---|
|**Obstruction-free**|A thread makes progress if it runs _alone_ (no contention)|
|**Lock-free**|At least _one_ thread in the system always makes progress|
|**Wait-free**|_Every_ thread makes progress in a bounded number of steps|

Most practical systems aim for **lock-free**. Wait-free is harder to implement and often has higher constant overhead, but gives stronger real-time guarantees.

---

## The Mental Model: Optimistic Concurrency

Lock-based thinking is **pessimistic**: "Someone might modify this, so I'll lock it first."

Lock-free thinking is **optimistic**: "I'll do my work assuming no conflict, then check. If there was a conflict, I'll redo the work."

```
loop:
    old = read(shared_value)       // snapshot
    new = compute(old)             // do your work on the snapshot
    if CAS(shared_value, old, new) // try to commit
        break                      // success
    // else: someone else changed it, retry
```

The key insight: **retrying is cheap** because you never blocked. You just burn CPU cycles instead of sleeping, but for short critical sections this is faster than a kernel sleep-and-wake cycle.

---

## What Lock-Free Is Good At (Where It Shines)

Lock-free is a good fit when:

1. **The critical section is short** — a counter increment, a pointer swap, a queue push. The "optimistic work" you redo on retry must be cheap.
    
2. **Contention is low-to-medium** — under high contention, CAS loops spin and waste CPU. A mutex might actually be better at very high contention because it sleeps the thread.
    
3. **Latency matters more than throughput** — lock-free avoids tail latency spikes caused by a thread holding a lock while preempted.
    
4. **You're building primitives** — queues, stacks, ring buffers, counters, reference counting (like `Arc` in Rust or `shared_ptr` in C++).
    
5. **You're on a hot path** — e.g., metrics collection, event queuing, a BFF request counter. Anything called millions of times per second.
    

---

## The Core Primitives You'll Actually Use

You don't usually write raw CAS loops in application code. You use these building blocks:

### Atomic Counter

```js
// Node.js doesn't have shared-memory atomics for single process,
// but in a SharedArrayBuffer (worker threads) context:
const counter = new Int32Array(new SharedArrayBuffer(4));
Atomics.add(counter, 0, 1); // atomic increment
```

### Lock-Free Queue (Concept)

The classic is the **Michael-Scott Queue** (two-pointer: head + tail).  
Push: CAS on the tail pointer.  
Pop: CAS on the head pointer.  
Both operations never block — they retry if CAS fails.

### Lock-Free Stack

```
push:
    new_node.next = top
    CAS(top, new_node.next, new_node)  // retry if top changed
    
pop:
    old_top = top
    CAS(top, old_top, old_top.next)    // retry if top changed
```

---

## Why Retry? And What Does Retry Actually Mean?

This is the most important question to understand deeply.

When CAS fails, it means: **the world changed between when you read it and when you tried to write it.**

So what do you retry? **You redo the entire loop — read again, compute again, then CAS again.**

```
loop:
    old = read(shared_value)       // 1. read the CURRENT state again
    new = compute(old)             // 2. recompute based on what's there NOW
    if CAS(shared_value, old, new) // 3. try to commit
        break
    // CAS failed → someone changed it → go back to step 1
```

You are NOT retrying "hoping it's different." You are retrying **because the input changed**, so you recompute with the new input.

### A concrete example: atomic counter increment

Say two threads both want to increment a counter from 5 to 6.

```
Thread A:  reads 5 → computes 6 → CAS(ptr, 5, 6) ✓ succeeds → counter is now 6
Thread B:  reads 5 → computes 6 → CAS(ptr, 5, 6) ✗ fails (it's already 6, not 5 anymore)
Thread B:  reads 6 → computes 7 → CAS(ptr, 6, 7) ✓ succeeds → counter is now 7
```

Thread B didn't retry "hoping 5 is back." It re-read 6, recomputed 7, and committed correctly.

The retry is meaningful — it's not a spin hoping for luck. **It's re-running the logic on fresh data.**

---

## Why the Work Inside the Loop Must Be Short

You are also exactly right about this. Here is why it matters:

Every retry throws away the work you did in the previous iteration. If your "compute" step takes 1 millisecond, and you retry 10 times under contention, you just wasted 10 milliseconds of CPU — and still produced only one successful write.

The failure cost = (cost of one iteration) × (number of retries)

So the rule is: **the work inside the CAS loop must be so cheap that retrying is negligible.**

Good candidates (cheap work):

- Increment a number
- Append a pointer to a linked list
- Swap one pointer
- Toggle a flag

Bad candidates (expensive work):

- Sort a list
- Compute a hash of a large object
- Do a network call
- Run a complex state machine

If your operation is complex, you have two choices:

1. Break it into smaller atomic steps, each with its own CAS
2. Accept that a mutex is the right tool here

Lock-free is a scalpel, not a hammer. It is precise and fast only when the operation it protects is tiny.

---

## The Hard Problems

### 1. The ABA Problem

CAS only checks "is the value what I expected?" It cannot detect whether the value changed and came back.

Example with a lock-free stack:

```
Thread 1:  reads top = A           → plans to pop it
           (gets preempted here)

Thread 2:  pops A
Thread 2:  pops B
Thread 2:  pushes A back

Thread 1:  resumes → CAS(top, A, A.next) → SUCCEEDS
           But A.next is now wrong — B was already gone
```

CAS saw `A` and said "nothing changed" — but everything changed. A was removed and re-inserted. The pointer looks the same, the history is completely different.

**Fix: attach a version counter to the value.**

Instead of storing just the pointer, store `(pointer, version)` together. The version increments every time the value is touched — it only ever goes up.

```
Start:             top = (A, version=1)

Thread 2 pops A:   top = (B, version=2)
Thread 2 pops B:   top = (C, version=3)
Thread 2 pushes A: top = (A, version=4)  ← pointer is A again, but version is 4

Thread 1 resumes:
  tries  CAS(top,  expected=(A, version=1),  new=...)
  actual top is    (A, version=4)
  (A,1) ≠ (A,4)  →  CAS FAILS ✓  → Thread 1 retries with fresh data
```

The version counter is a "how many times has anything changed here" counter. Even if the pointer cycles back to the same address, the version never lies.

### 2. Memory Ordering

CPUs and compilers reorder instructions for performance. A write in your code may not be visible to another CPU core immediately.

You must use **memory barriers** (fences) or atomic operations with the right ordering semantics:

- `Acquire` — "I'm reading, ensure I see all prior writes"
- `Release` — "I'm writing, ensure my writes are visible before this"
- `SeqCst` — strongest, total order across all threads (slowest)

In Node.js / V8 land, `Atomics` operations handle this for you. In C/C++/Rust, you choose explicitly.

### 3. Lock-Free ≠ Wait-Free

Lock-free guarantees _system_ progress, not _your thread's_ progress. Under very high contention, one unlucky thread could retry thousands of times while others succeed. For hard real-time systems, this matters.

---

## Criteria: When Should You Use Lock-Free?

Ask yourself these questions:

|Question|If Yes → Lock-Free Makes Sense|
|---|---|
|Is the critical section < ~10 instructions?|✓|
|Is contention low-medium (< ~8 threads fighting)?|✓|
|Do you need predictable low latency?|✓|
|Is this a hot path called millions of times/sec?|✓|
|Are you building a queue, stack, counter, or ring buffer?|✓|
|Is the shared data a single pointer or integer?|✓|

|Question|If Yes → Consider a Mutex Instead|
|---|---|
|Is the critical section large / complex?|✗|
|Is contention very high (many threads, long waits)?|✗|
|Does correctness require multiple operations to be atomic together?|✗|
|Are you modifying a complex data structure (tree, hash map)?|✗|

---
## Summary

```
Problem:    Locks → OS involvement → context switch → TLB flush → latency
Solution:   Atomic hardware instructions (CAS) → no blocking → no OS → no flush

Core idea:  Read → compute → CAS to commit → retry if conflict
            (Optimistic: assume no conflict, handle it if it happens)

Sweet spot: Short critical sections, low-medium contention, hot paths, primitives

Hard parts: ABA problem, memory ordering, fairness under high contention
```

Lock-free is not a silver bullet — it's a precision tool for situations where the overhead of traditional locking is measurably a bottleneck.