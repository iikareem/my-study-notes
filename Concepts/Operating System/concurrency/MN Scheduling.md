---
tags:
  - concurrency
  - operating-system
---

# M:N Scheduling — From Scratch

> Why it was invented, what problem it solves, and how it actually works — language-agnostic.
[[Concurrency & Multitasking From Zero to Production-Ready]]
---

## Table of Contents

1. [Start Here: The Two Worlds](#1-start-here-the-two-worlds)
2. [The Problem With Pure OS Threads (1:1)](#2-the-problem-with-pure-os-threads-11)
3. [The Problem With Pure User Threads (N:1)](#3-the-problem-with-pure-user-threads-n1)
4. [M:N — The Hybrid Solution](#4-mn--the-hybrid-solution)
5. [How M:N Scheduling Works Internally](#5-how-mn-scheduling-works-internally)
6. [The Hard Problems M:N Must Solve](#6-the-hard-problems-mn-must-solve)
7. [M:N vs. Async/Await — Are They the Same?](#7-mn-vs-asyncawait--are-they-the-same)
8. [Where M:N Scheduling Lives Today](#8-where-mn-scheduling-lives-today)
9. [Quick Summary](#9-quick-summary)

---

## 1. Start Here: The Two Worlds

To understand M:N scheduling you first need to understand that there are two completely separate layers where "threads" can live:

### The OS Layer (Kernel Space)

The operating system manages **kernel threads** — real threads the CPU scheduler knows about and runs. Creating one asks the OS to allocate a stack (~1 MB by default), register the thread in the kernel's scheduler, and from that point on, the OS is in full control of when it runs.

```
OS Kernel:
┌──────────────────────────────────────┐
│  Kernel Thread 1   Kernel Thread 2   │
│       ↕                   ↕          │
│         CPU Scheduler                │
│         (preemptive)                 │
└──────────────────────────────────────┘
```

### The User Layer (User Space)

Your program (or its runtime) can also manage its own lightweight units of work — entirely in user space, without the OS knowing about them. These go by many names: **green threads**, **fibers**, **coroutines**, **goroutines**, **virtual threads**. They are all variations of the same idea: a task that the runtime, not the OS, is responsible for scheduling.

```
Your Program:
┌──────────────────────────────────────┐
│  Task 1   Task 2   Task 3   Task 4   │
│       ↕                              │
│    Runtime Scheduler                 │
│    (user space, your process)        │
└──────────────────────────────────────┘
```

These two layers can be combined in three different ways. That is exactly what 1:1, N:1, and M:N describe.

---

## 2. The Problem With Pure OS Threads (1:1)

**1:1 means: one user task = one OS kernel thread.**

Every task you create maps directly to one real OS thread. This is the simplest model and what most languages defaulted to historically (C, C++, Java pre-21).

```
User Tasks:    T1    T2    T3    T4
               |     |     |     |
               ↕     ↕     ↕     ↕
OS Threads:   KT1   KT2   KT3   KT4
```

### Why It Works Well at Small Scale

- True parallelism on multiple cores
- Simple mental model — each task is an independent OS-managed thread
- Blocking syscalls (file read, network wait) only block that one thread

### Why It Breaks at Large Scale

**Problem 1: Cost of creation**

Each OS thread needs ~1 MB of stack memory reserved upfront. Spawning 10,000 concurrent connections means:

```
10,000 threads × 1 MB stack = 10 GB of memory
```

That is before any of your actual application data. Most servers cannot afford this.

**Problem 2: Context switching overhead**

Every switch between OS threads requires the kernel to:

1. Save the running thread's registers, stack pointer, and memory mappings
2. Restore the next thread's state
3. Flush CPU caches and TLBs (memory translation caches)

This costs roughly **1,000–10,000 CPU instructions** per switch. With 10,000 threads all runnable, the system can spend more time switching than working.

**Problem 3: Scheduler pressure**

The OS scheduler was not designed to handle 100,000 runnable threads. Its data structures and algorithms work well in the range of tens to hundreds of threads per process. Saturating it with thousands causes scheduling latency and unpredictable behavior.

**The wall:** 1:1 threading hits a hard scalability wall somewhere between 1,000 and 10,000 threads depending on the system. This is a well-known problem with a name — the **C10K problem** (how do you serve 10,000 concurrent clients?).

---

## 3. The Problem With Pure User Threads (N:1)

Seeing the limits of OS threads, early runtimes went to the opposite extreme: **N:1 means many user tasks, all running on one single OS thread.**

The runtime manages all task scheduling itself, in user space, without involving the OS for each switch.

```
User Tasks:   T1   T2   T3   T4   T5   T6
               \    |    |    |    |   /
                \   |    |    |    |  /
                 Runtime Scheduler
                        |
                        ↕
               One OS Kernel Thread
```

### Why This Was Attractive

- Extremely cheap task creation (no OS call, tiny stack)
- Fast task switching (pure user-space function calls, no kernel involvement)
- Thousands of tasks use negligible memory

### Why It Was Abandoned

**Problem 1: No true parallelism**

Everything runs on one OS thread. On a machine with 16 CPU cores, your program uses exactly one. You cannot use the other 15, regardless of how much work you have.

**Problem 2: One blocking call stops everything**

If any task makes a blocking system call — reading from disk, waiting on a socket — the _entire_ OS thread blocks. Every other task in the program freezes, even if they have work to do.

```
T1 calls read() → OS thread blocks → T2, T3, T4, T5 all frozen
                  even though they are ready to run
```

This made N:1 threading fragile and difficult to use correctly. You had to manually convert every blocking call to a non-blocking one — which is complex, error-prone, and limits what libraries you can use.

---

## 4. M:N — The Hybrid Solution

**M:N means: M user tasks running across N OS threads, where M >> N.**

The idea is to take the good parts of both models and eliminate the bad:

- From **1:1**: use multiple OS threads to get real parallelism and to avoid one blocking call freezing everything
- From **N:1**: manage lightweight tasks in user space to avoid the cost and scalability limits of OS threads

```
User Tasks (M):   T1  T2  T3  T4  T5  T6  T7  T8
                   \  |   |  /     \  |   |  /
                    \ |   | /       \ |   | /
Runtime Scheduler: [Worker 1]      [Worker 2]
                        |               |
                        ↕               ↕
OS Kernel Threads:     KT1             KT2
                        |               |
                 CPU Core 1       CPU Core 2
```

### The Numbers

|Model|User tasks|OS threads|Parallelism|Blocking safe?|
|---|---|---|---|---|
|**1:1**|N|N|✅ Yes|✅ Yes|
|**N:1**|M (many)|1|❌ No|❌ No|
|**M:N**|M (many)|N (small)|✅ Yes|✅ Yes|

A typical M:N runtime creates N OS threads equal to the number of CPU cores (e.g. 8 on an 8-core machine), then multiplexes thousands or millions of user tasks across them.

### Why This Was Invented

The combination of two trends in the late 1990s and 2000s made M:N necessary:

1. **The internet scaled**: servers began needing to handle thousands of simultaneous connections, not dozens. 1:1 threading could not scale this far.
    
2. **Multi-core CPUs arrived**: pure async/N:1 solutions could not use multiple cores. Servers had 4, 8, 16 cores sitting idle.
    

M:N scheduling was the engineering response: keep the OS thread count small (match cores), but allow the number of concurrent tasks to grow as large as needed.

---

## 5. How M:N Scheduling Works Internally

Understanding the mechanism makes the whole model click.

### The Core Components

```
┌─────────────────────────────────────────────────────────┐
│                    User Space                           │
│                                                         │
│  Run Queue:  [T3] → [T7] → [T1] → [T9] → ...          │
│                                                         │
│  Worker 1 (OS Thread):   currently running T5           │
│  Worker 2 (OS Thread):   currently running T2           │
│  Worker 3 (OS Thread):   currently running T8           │
│                                                         │
│  Parked Tasks: T4 (waiting on network), T6 (sleeping)  │
└─────────────────────────────────────────────────────────┘
```

**Run queue:** A shared list of tasks that are ready to run but not yet assigned to a worker.

**Workers:** A fixed pool of OS threads. Each worker loops: pick a task from the run queue → run it → when it yields or blocks, put it back or park it → pick the next task.

**Parked tasks:** Tasks waiting for something (I/O, a timer, a lock). They are not in the run queue. They consume no CPU. When their wait is complete, they are moved back into the run queue.

### The Task Lifecycle

```
[Created] → [Run Queue] → [Running] → [Finished]
                ↑              |
                |         (yields / blocks)
                |              ↓
            [Run Queue]   [Parked / Waiting]
                ↑              |
                └──────────────┘
                  (I/O completes)
```

### The Key Operation: Yielding

When a task hits a wait point (network call, sleep, waiting for a lock), the runtime does this entirely in user space:

1. Save the task's current execution state (registers, stack pointer) — this is called the **task context**
2. Mark the task as "parked, waiting for X"
3. Pick the next task from the run queue
4. Restore that task's context and resume it

This is structurally identical to an OS context switch, but:

- No kernel call required (much faster — ~100ns vs ~1–10µs)
- No TLB flush (same process memory)
- Stack can be tiny (a few KB, not 1 MB)
- The runtime controls timing, not the OS

### Handling Blocking Syscalls

The trickiest part of M:N is what happens when a task makes a _truly blocking_ OS call (like a blocking `read()` on a file). The OS thread blocks — which means that worker can no longer run other tasks.

Runtimes handle this in a few ways:

**Option A: Convert to async I/O** The runtime intercepts the call and uses non-blocking I/O under the hood (epoll on Linux, kqueue on macOS, IOCP on Windows), turning the blocking call into a park + resume.

**Option B: Spawn a temporary OS thread** When a worker blocks on a syscall, the runtime spawns a new OS thread to temporarily replace it, keeping the number of active workers constant.

```
Before blocking call:    Worker 1 is available
During blocking call:    Worker 1 is stuck in OS
                         Runtime spawns Worker 1' to compensate
After call returns:      Worker 1 resumes, Worker 1' is retired
```

Most real M:N runtimes use both options depending on the type of I/O.

---

## 6. The Hard Problems M:N Must Solve

M:N is powerful but genuinely difficult to implement correctly. Here are the non-trivial problems any M:N runtime must tackle:

### Problem 1: Work Stealing

If Worker 1's run queue is empty and Worker 2's queue has 50 tasks, Worker 1 is idle while Worker 2 is overloaded.

**Solution: Work stealing.** Idle workers "steal" tasks from the tail of busy workers' queues. This balances load automatically without central coordination.

```
Worker 1 (idle):  empty queue → steals tasks from Worker 2
Worker 2 (busy):  [T3][T7][T1][T9][T5] → Worker 1 takes [T9][T5]
```

### Problem 2: Stack Growth

OS threads get a large fixed stack (~1 MB) because the OS cannot easily grow it. User tasks start with tiny stacks (4–8 KB) to save memory, but some tasks need more stack as they call deeper into functions.

**Solution:** Segmented stacks or stack copying. When a task's stack overflows its current allocation, the runtime allocates more and either links the segments together or copies the whole stack to a larger allocation.

### Problem 3: Preemption of CPU-Bound Tasks

In cooperative scheduling, a task that never yields (pure CPU computation) never lets other tasks run on that worker — starving them.

**Solution:** The runtime installs a timer that fires periodically and forces a yield, even for CPU-bound tasks. This makes the user-space scheduler partially preemptive, not just cooperative.

### Problem 4: Syscall Detection

The runtime needs to know when a task is about to make a blocking syscall so it can take action (park the task, compensate with a new thread). This typically requires either:

- Intercepting standard library functions (libc wrapping)
- Compiling the language's I/O layer to always go through the runtime

This is why M:N scheduling is most practical when the runtime controls the language's standard library — it is very hard to bolt onto existing codebases.

---

## 7. M:N vs. Async/Await — Are They the Same?

This is a common source of confusion. They solve the same _problem_ (efficient concurrency without 1:1 thread cost) but at different levels.

### Async/Await (Coroutines)

- Concurrency is expressed in the **source code** — you write `await` at yield points
- The compiler transforms your function into a state machine
- Scheduling is typically N:1 (one event loop thread) unless you explicitly add worker threads
- Blocking a thread blocks all coroutines on that thread — you must never call blocking code

### M:N Scheduling

- Concurrency is expressed as **tasks** — you spawn a task and the runtime handles the rest
- No special syntax required for yield points (the runtime can preempt or detect blocking calls)
- Multiple OS threads are used automatically
- Blocking calls can be handled transparently by the runtime

```
Async/Await:
  You are responsible for never blocking.
  Cooperative yield is explicit in your code (await).
  Simpler runtime, harder to use correctly.

M:N Scheduling:
  The runtime handles blocking transparently.
  Yield can be implicit (runtime preempts you).
  More complex runtime, easier to use correctly.
```

### The Relationship

Many modern systems combine both. An M:N runtime runs user tasks across multiple OS threads. Within each task, async/await syntax is used to express yield points. The runtime handles the multiplexing, and `await` gives it explicit hints about where switching is cheap and safe.

---

## 8. Where M:N Scheduling Lives Today

M:N scheduling is not a niche idea — it powers some of the most widely used software in the world.

### Runtimes That Use M:N

|Runtime|User tasks|OS threads|Notes|
|---|---|---|---|
|**Go runtime**|Goroutines (millions possible)|GOMAXPROCS (default: CPU count)|Work-stealing scheduler, cooperative + timer-based preemption|
|**Java Virtual Threads** (Project Loom, Java 21+)|Virtual threads|Platform threads (CPU count)|Transparent — existing blocking code works unchanged|
|**Erlang/BEAM**|Processes (millions)|CPU count|Preemptive, soft-real-time guarantees, pioneered many M:N ideas|
|**Haskell GHC**|Haskell threads|Capabilities (CPU count)|Preemptive with very cheap threads|
|**Node.js** (partial)|Worker threads + async tasks|libuv thread pool|Not a full M:N model, but uses a thread pool for blocking I/O|
|**Tokio (Rust)**|Tasks|Worker threads (CPU count)|Cooperative, async/await based, work-stealing|
|**.NET ThreadPool**|Tasks|Dynamic thread pool|Work-stealing, but uses larger OS threads not green threads|

### Historical Context

The idea is not new. Here is a brief timeline:

```
1993 — Solaris 2 introduces M:N threading as the default model
1995 — Erlang launches with lightweight processes (M:N pioneering)
2000s — Linux NPTL (Native POSIX Thread Library) makes 1:1 good enough
         → Many systems abandoned M:N due to implementation complexity
2009 — Go launches, brings M:N back with a cleaner design
2012 — Node.js popularizes async I/O (not M:N, but solves the same scaling problem)
2021 — Java Project Loom delivers virtual threads to the JVM ecosystem
```

An interesting historical note: Linux's 1:1 threading (NPTL) became so efficient in the early 2000s that many runtimes (including early versions of Go's predecessor and some JVMs) temporarily abandoned M:N in favor of 1:1. M:N came back because 1:1 still cannot scale to millions of concurrent tasks regardless of how cheap individual thread creation gets.

---

## 9. Quick Summary

### The Three Models at a Glance

```
1:1 (OS Threads Only)
├── Pro: Simple, true parallelism, blocking is safe
└── Con: ~1 MB per thread, OS scheduler overload at thousands of threads

N:1 (User Threads Only)
├── Pro: Millions of tasks, fast switching, tiny memory
└── Con: One core only, one blocking call freezes everything

M:N (Hybrid)
├── Pro: Millions of tasks + true parallelism + blocking handled transparently
└── Con: Complex runtime to implement correctly
```

### The One Mental Model to Keep

Think of M:N like a restaurant:

- **OS threads (workers)** = the number of tables being served simultaneously (fixed, matches your resources)
- **User tasks** = the number of customers (can be huge, far exceeds workers)
- **The runtime scheduler** = the floor manager routing customers to available tables

The floor manager is far more efficient than having one waiter permanently assigned to every customer regardless of whether that customer is eating, waiting for food, or just thinking about dessert.

### When M:N Matters to You as a Developer

You do not need to implement M:N scheduling — but you should understand it when:

- **Choosing a language/runtime** for a high-concurrency server: Go and Java 21 give you M:N for free
- **Debugging performance problems**: knowing why goroutine or virtual thread count alone does not hurt you helps you understand what actually costs money
- **Understanding async limitations**: async/await in most languages is N:1 underneath — it cannot use multiple cores without explicit parallelism
- **Evaluating framework claims**: "lightweight concurrency" and "millions of tasks" are only meaningful if the runtime underneath is genuinely M:N

---

_The key invention behind M:N is separating the unit of concurrency (the task) from the unit of parallelism (the OS thread). Once you see that separation, every modern concurrency primitive starts to make much more sense._