---
tags:
  - os
  - concurrency
  - synchronization
---

# Concurrency & Synchronization Primitives — Code Level Guide

> The same race condition you fight at the database level exists at the thread level. Same problem, different scope. The primitives map 1-to-1 to what you already know.

---

## Quick Navigation

| Section                                                           | What you'll get                            |
| ----------------------------------------------------------------- | ------------------------------------------ |
| [1. Core Problem](#1-the-core-problem)                            | Why races exist (single-core & multi-core) |
| [2. Layer Stack](#2-the-layer-stack)                              | Where this guide sits                      |
| [3–7. Primitives](#3-mutex)                                       | Mutex → Semaphore → RWLock → CAS → Monitor |
| [8. Deadlock](#8-deadlock)                                        | Coffman conditions + prevention            |
| [9. Visibility & Cost](#9-memory-visibility--context-switch-cost) | Caches, barriers, context switches         |
| [10. Mapping & Choice](#10-code-vs-db-mapping)                    | DB equivalents + decision tree             |
| [11. Interview](#11-interview-playbook)                           | Q&A + red flags                            |
| [Cheat Sheet](#cheat-sheet)                                       | One-table summary                          |

---

## 1. The Core Problem

Every race has the same shape at every layer:

```
1. Read   →  "what is the current state?"
2. Check  →  "is it valid / available?"
3. Act    →  "do the thing"
```

The gap between step 1 and step 3 is the **vulnerability window**.

```
Time →     Thread 1                  Thread 2
───────────────────────────────────────────────────
T=0    Read counter = 1         Read counter = 1
T=1    counter > 0? YES         counter > 0? YES
T=2    counter = counter - 1    counter = counter - 1
T=3    Write counter = 0        Write counter = 0
───────────────────────────────────────────────────
Result: counter = 0 but TWO threads "won" ← RACE
```

Even `counter--` is three CPU instructions:

```
LOAD  counter → register    ; read
SUB   register, 1           ; modify
STORE register → counter    ; write
```

### Single-core vs multi-core

| | How the race happens | Extra problem |
|---|---|---|
| **Single-core** | OS preempts mid-operation | — |
| **Multi-core** | True simultaneous execution | Cache visibility (stale L1) |

**Single-core interleaving:**

```
Core 1: Thread A
          LOAD counter → register
          ← OS switches to Thread B
Core 1: Thread B
          LOAD / SUB / STORE   (same stale value)
          ← OS switches back to A
Core 1: Thread A
          SUB / STORE          ← overwrites B's write
```

**Multi-core true parallelism:**

```
Core 1: Thread A ──────────────────────►
Core 2: Thread B ──────────────────────►
        both can read the same address at the same nanosecond
```

Multi-core makes races more frequent, harder to detect, and adds **memory visibility** (each core has its own cache). Synchronization primitives solve both: ordering **and** visibility.

---

## 2. The Layer Stack

```
┌──────────────────────────────────────────────────────┐
│  DATABASE     Transactions, SELECT FOR UPDATE, dist. lock │
├──────────────────────────────────────────────────────┤
│  SERVICE      Redis / Redlock / fencing token             │
├──────────────────────────────────────────────────────┤
│  THREAD  ←    Mutex, Semaphore, RWLock, CAS, Monitor      │
├──────────────────────────────────────────────────────┤
│  CPU/MEM      Atomic instructions, memory barriers        │
└──────────────────────────────────────────────────────┘
```

Understand one level → you understand all of them.

---

## 3. Mutex

**One thread in at a time. Everyone else blocks.**

DB equivalent: `SELECT FOR UPDATE`.

```
A: acquire → GRANTED → critical section
B: acquire → BLOCKED
C: acquire → BLOCKED

A: release
B: GRANTED
C: still BLOCKED
```

### Critical section

Everything between acquire and release is protected — your check-then-act lives here:

```
LOCK
  check condition
  modify state
  do related work
UNLOCK
```

### Reentrant mutex

A regular mutex **deadlocks** if the same thread acquires twice (blocks on itself). A reentrant mutex tracks a per-thread count:

```
Regular:   acquire → acquire again → DEADLOCK
Reentrant: acquire (count=1) → acquire (count=2) → release → release (count=0)
```

Use when a lock-holding function calls another that needs the same lock.

### Tradeoffs

| Pros | Cons |
|---|---|
| Simple, predictable | Serializes all threads |
| Prevents all races in the section | Deadlock risk with multiple mutexes |
| Works for any multi-step logic | **Context switch** when a thread blocks on the lock |

> **Cost to remember:** every time a thread hits a held mutex, the OS puts it to sleep → **context switch out**. When the lock is released → **context switch back in**. That is ~1,000–10,000 ns per switch — often more expensive than the work inside a short critical section. See [§9](#9-memory-visibility--context-switch-cost).

---

## 4. Semaphore

**Internal counter. Enter while > 0. Block at 0.**

```
Semaphore(3):
  A enters → 2
  B enters → 1
  C enters → 0
  D tries  → BLOCKED
  A exits  → 1 → D enters → 0
```

### Binary semaphore vs mutex

They look the same (N=1) but differ on **ownership**:

| | Mutex | Binary semaphore |
|---|---|---|
| Ownership | Only locker can unlock | Any thread can signal/release |
| Purpose | Protect a resource | Cross-thread signaling |

**Mutex — protect a resource:**

```
A: LOCK → work → UNLOCK     (only A can unlock)
B: blocked until A releases
```

**Binary semaphore — signal between threads:**

```
A (consumer):  wait()     ← blocks at 0
B (producer):  work → signal()   ← B releases what A waits on
A:             continues
```

Ownership would make producer→consumer signaling impossible.

### Mental model

```
Mutex        = "I own this resource until I'm done"
Semaphore(N) = "How many slots are available?"
Binary Sem   = "Has the event happened? (0=no, 1=yes)"
```

### When to use

| Need | Tool | Why |
|---|---|---|
| Protect shared data (multi-step) | Mutex | Ownership required |
| Limit DB connections to 10 | Semaphore(10) | N-slot pool |
| Producer signals consumer | Binary semaphore | Cross-thread signal |
| Max 5 concurrent API calls | Semaphore(5) | Throughput control |

### Tradeoffs

| Pros | Cons |
|---|---|
| Flexible N-way concurrency | No ownership → easy to misuse |
| Cross-thread signaling | Harder to reason about than mutex |
| Resource pool management | Controls throughput, not a single value |

---

## 5. Read-Write Lock

**Many readers at once OR one exclusive writer. Never both.**

DB equivalent: shared (S) vs exclusive (X) locks.

```
A,B,C (read) → GRANTED  readers=3
D (write)    → BLOCKED until readers=0
D gets lock  → exclusive
E (read)     → BLOCKED while writer holds
```

### Why it exists

Mutex serializes readers unnecessarily. RWLock lets concurrent reads through:

```
Mutex:   R1 → R2 → R3 → R4     (serialized)
RWLock:  R1, R2, R3 concurrent  (higher throughput)
```

### Writer starvation

Continuous readers can starve writers forever. Fix: **writer-priority** — once a writer waits, new readers queue behind it.

### Tradeoffs

| Pros | Cons |
|---|---|
| High read throughput | Writer starvation under heavy reads |
| Right for read-heavy / write-rare | More complex than mutex |
| Matches DB S/X semantics | Read→write upgrade not atomic |

---

## 6. Atomic Operations (CAS)

**Compare-and-Swap:** one CPU instruction — read, compare, write atomically. No locks. No blocking.

DB equivalent: optimistic locking with a version column.

### Operation

```
CAS(address, expected, new_value):
  if *address == expected:
    *address = new_value
    return SUCCESS
  else:
    return FAILURE   ← retry
```

### Retry loop (optimistic pattern)

```
loop:
  current = read(counter)
  if current <= 0: return SOLD_OUT
  if CAS(counter, current, current - 1): return BOOKED
  else: retry
```

### Go reference

```go
import "sync/atomic"

var counter int64 = 100

func bookTicket() bool {
    for {
        current := atomic.LoadInt64(&counter)
        if current <= 0 {
            return false
        }
        if atomic.CompareAndSwapInt64(&counter, current, current-1) {
            return true
        }
    }
}
```

### ABA problem

Value goes A → B → A; CAS thinks nothing changed.

```
T1: reads 10
T2: 10 → 9 → 10
T1: CAS(10, 9) → SUCCESS  ← world changed underneath
```

**Fix:** CAS on `(value, version)`. Version always increases: `(A,1) → (B,2) → (A,3)`.

### Why atomic is often better than a mutex (context switch)

Mutex path under contention always involves the OS scheduler:

```
Thread B tries mutex → held by A
  ↓
OS removes B from CPU          ← CONTEXT SWITCH OUT  (~1–10 µs)
OS runs something else
  ↓
A releases mutex
OS puts B back on CPU          ← CONTEXT SWITCH IN   (~1–10 µs)
B finally enters critical section
```

**Two context switches per wait.** Each switch saves/restores registers, stack pointer, program counter, and may invalidate cache lines.

CAS path — thread **never sleeps**:

```
Thread B:  read → CAS fail → retry → CAS fail → retry → CAS success
           stays on the CPU the whole time
           ❌ no context switch
           ⚠️ burns CPU cycles while retrying
```

| | Mutex (blocking) | Atomic CAS |
|---|---|---|
| On conflict | Sleep + wake via OS | Stay on CPU, retry in user space |
| Context switch? | **Yes** (usually 2) | **No** |
| Typical cost (low contention) | Lock + unlock + possible switches → **µs** | One hardware instruction → **ns** |
| Typical cost (high contention) | Queued waiters — CPU free for other work | Retry storm — **CPU burn** |
| Best when | Multi-step critical section, or lock held a while | Single value, short update, low/medium contention |

> **Takeaway:** Atomic wins when the critical work is a **single value** and contention is not extreme — you skip the entire sleep/wake round-trip. Mutex wins when many threads hammer the same spot (retries waste more than sleeping) or you need a **multi-step** critical section CAS cannot express.

### CAS vs mutex (contention)

| Contention | Winner | Why |
|---|---|---|
| Low | **CAS** | No sleep, no context switch — finishes in nanoseconds |
| High | **Mutex** | CAS retry storm burns CPU; mutex lets waiters sleep |

### Tradeoffs

| Pros | Cons |
|---|---|
| **No context switch** — never blocks | CPU burn under high contention |
| No deadlock | ABA without versioning |
| Hardware atomicity | Only single-value ops |
| Fastest under low contention | Cannot wrap multi-step logic |

---

## 7. Monitor & Condition Variable

**Mutex + efficient wait.** Sleep until a condition is true — no spin/poll.

```
Consumer:                 Producer:
  lock                      lock
  while !condition:         make condition true
    wait()  ← release+sleep signal()
  do work                   unlock
  unlock
```

`wait()` atomically releases the lock and sleeps — avoids the race between check and sleep.

### WHILE, not IF

Spurious wakeups exist (OS can wake without a signal). Always recheck:

```
WRONG:  if tickets == 0: wait()   → may proceed with tickets still 0
RIGHT:  while tickets == 0: wait() → guaranteed tickets > 0 after loop
```

### notify vs notify_all

| | Behavior | Use when |
|---|---|---|
| `notify()` | Wakes **one** waiter | Exactly one thread should proceed |
| `notify_all()` | Wakes **all** | Multiple waiters may each satisfy the condition |

### Classic: producer–consumer

```
Producer: while buffer full  → wait; add item; signal consumers
Consumer: while buffer empty → wait; take item; signal producers
```

### Tradeoffs

| Pros | Cons |
|---|---|
| Efficient sleep, no CPU burn | Hardest to reason about |
| Any condition expressible | Must handle spurious wakeups |
| Lock released while waiting | notify vs notify_all bugs |

---

## 8. Deadlock

Threads wait for each other in a cycle. No one proceeds.

### Four Coffman conditions

All four must hold. Remove any one → deadlock impossible.

1. **Mutual exclusion** — resource held by one thread
2. **Hold and wait** — holds one while waiting for another
3. **No preemption** — can't forcibly take resources
4. **Circular wait** — T1↔T2 cycle

### Classic case

```
T1: lock(A) → tries lock(B) → BLOCKED
T2: lock(B) → tries lock(A) → BLOCKED
→ DEADLOCK
```

### Prevention

| Strategy | Breaks | How |
|---|---|---|
| **Lock ordering** | Circular wait | Always acquire A then B, never reverse |
| **Try-lock + timeout** | Hold and wait | Fail → release all → retry |
| **Reduce scope** | Hold and wait | Short holds, fewer locks at once |

```
✅ T1 & T2: lock(A) → lock(B)
❌ T2:      lock(B) → lock(A)
```

---

## 9. Memory Visibility & Context Switch Cost

### Visibility (multi-core caches)

```
Core 1: counter = 1  → L1 only
Core 2: reads counter → own L1 → sees 0 (stale)
```

Mutex lock/unlock and atomics include **memory barriers**. Unlock → next lock establishes *happens-before* — B sees everything A wrote before unlocking.

### volatile

| Guarantees | Does NOT guarantee |
|---|---|
| Writes flush to main memory | Atomicity |
| Reads from main memory, not cache | Thread-safety of `counter++` |

Use only for a single flag/signal. For read-modify-write → locks or atomics.

### What a context switch actually costs

Blocking locks (mutex, semaphore, RWLock, monitor) put a waiting thread to **sleep**. Sleeping requires the OS to switch the CPU to another thread:

```
What the OS does on each switch:
  1. Save current thread: registers, stack pointer, program counter
  2. Load next thread:    registers, stack pointer, program counter
  3. Cache may go cold — new thread touches different memory

Typical cost: ~1,000 to 10,000 nanoseconds  (1–10 µs)
```

Under high contention with short critical sections, threads can spend **more time switching than working**.

### Blocking vs non-blocking (highlighted)

| Primitive | On contention | Context switch? | Pays with |
|---|---|---|---|
| **Mutex** | Sleep and wait | ✅ **Yes** (out + in) | Scheduler latency |
| **Semaphore** | Sleep and wait | ✅ **Yes** | Scheduler latency |
| **RWLock** | Sleep and wait | ✅ **Yes** | Scheduler latency |
| **Monitor / CondVar** | Sleep and wait | ✅ **Yes** | Scheduler latency |
| **Spinlock** | Busy-wait (spin) | ❌ No | CPU cycles |
| **Atomic CAS** | Busy-wait (retry loop) | ❌ No | CPU cycles |

```
BLOCKING (Mutex, Sem, RWLock, Monitor)
  conflict → sleep → CONTEXT SWITCH → wake → CONTEXT SWITCH
  ✅ CPU free for other work while waiting
  ❌ expensive round-trip — bad for tiny critical sections

NON-BLOCKING (CAS, Spinlock)
  conflict → stay on CPU → retry / spin → succeed
  ✅ no context switch — often much faster under low contention
  ❌ burns the core while waiting — bad under heavy contention
```

> **Rule of thumb:** prefer **atomic / spin** when the update is tiny and contention is low (context switch would dominate). Prefer **mutex** when the critical section is longer or many threads contend (spinning / retrying wastes more than sleeping).

---

## 10. Code vs DB Mapping

```
CODE                              DB / SERVICE EQUIVALENT
────────────────────────────────────────────────────────────
Mutex                          →  SELECT FOR UPDATE
Atomic CAS + retry             →  Optimistic lock (version)
Semaphore(N)                   →  Connection pool / rate limit
Binary semaphore               →  Event flag / condition signal
Read-Write Lock                →  Shared (S) / Exclusive (X)
Monitor + CondVar              →  Queue + blocked wait (SKIP LOCKED)
volatile / barrier             →  Visibility (e.g. read committed)
Try-lock + timeout             →  lock_wait_timeout
Deadlock detection             →  DB detector + victim rollback
────────────────────────────────────────────────────────────
Distributed lock (Redis)       →  Mutex across services
Fencing token                  →  CAS version at the resource
Message queue (Kafka/SQS)      →  Monitor + producer-consumer
```

### Decision tree

```
Need?
├── Exclusive multi-step access
│   ├── One owner must release     → Mutex
│   ├── Same thread re-enters      → Reentrant Mutex
│   └── Ultra-short hot path       → Spinlock
├── Control how many can enter
│   ├── N concurrent slots         → Semaphore(N)
│   └── Signal thread → thread     → Binary Semaphore
├── Read-heavy, write-rare         → Read-Write Lock
├── Single value, no multi-step    → Atomic CAS
└── Wait until condition true      → Monitor + CondVar
```

### Performance snapshot

```
Low contention speed:   CAS > Mutex > RWLock > Monitor
                        ↑ no context switch — often wins by 10–1000× on tiny updates
High contention:        Mutex > CAS (retry storm burns more than sleeping)
Read-heavy throughput:  RWLock >> Mutex
Complexity (easy→hard): Mutex → RWLock → Monitor → CAS (ABA) → Binary Sem
```

---

## 11. Interview Playbook

### Opening line

> "The core problem is always read-modify-write / check-then-act with a vulnerability window. I pick the primitive by need: exclusive multi-step (mutex), N slots (semaphore), read/write split (RWLock), or one atomic value (CAS)."

### Key Q&A

**Mutex vs binary semaphore?**  
Same "one at a time" look. Mutex has ownership (only locker unlocks). Binary semaphore does not — any thread can signal. Mutex = protect resource. Binary sem = cross-thread signal.

**CAS over mutex?**  
CAS: single value, low contention — **no sleep, no context switch**, finishes in nanoseconds. Mutex: multi-step sections or high contention — CAS retries burn CPU; sleeping + waking is then cheaper.

**Prevent deadlock?**  
Break one Coffman condition. Practical default: global lock ordering → no circular wait.

**What does volatile do?**  
Visibility only (flush / read from memory). Not atomicity. Compound ops still need locks or atomics.

### Red flags

| Don't say | Say instead |
|---|---|
| "Binary sem and mutex are the same" | "Look alike; mutex has ownership" |
| "volatile makes it thread-safe" | "Visibility only — RMW still races" |
| "Add more locks" | "More locks → deadlock risk; check ordering" |
| "CAS is always faster" | "Wins low contention; loses under retry storms" |

---

## Cheat Sheet

| Primitive | Concurrency | Ownership | Best for | Key risk |
|---|---|---|---|---|
| **Mutex** | 1 | Yes | Multi-step exclusive | Deadlock |
| **Semaphore(N)** | N | No | Pools, rate limits | Any thread can release |
| **Binary Sem** | 1 slot | No | Cross-thread signal | Confused with mutex |
| **RWLock** | N readers XOR 1 writer | Per mode | Read-heavy | Writer starvation |
| **Atomic CAS** | Non-blocking | No | Counter / flag | ABA, retry storm — but **no context switch** |
| **Monitor + CondVar** | 1 + sleepers | Yes (inner mutex) | Producer–consumer | Spurious wakeups |

> **Meta-lesson:** Same tradeoff every layer — correctness vs throughput. Mutex = max safety, min concurrency. CAS = max concurrency, simple values only. Everything else sits in between.

---

## Related

- [[Concurrency Complete Guide]]
- [[Dealing with Contention in System Design]]
- [[Distributed Lock]]

← [[Concepts MOC]]
