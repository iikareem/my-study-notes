---
tags:
  - concurrency
  - operating-system
---

# Concurrency & Multitasking: The Complete Guide From OS Fundamentals to Production Systems

> A full, no-shortcut guide: how the kernel, processes, threads, context switches, and schedulers actually work — and why that knowledge decides how you write async code, goroutines, and scalable servers.

---

## Table of Contents

1. [Why This Matters](#1-why-this-matters)
2. [Prerequisites: The OS Layer You Must Understand First](#2-prerequisites-the-os-layer-you-must-understand-first)
   - 2.1 [The Kernel](#21-the-kernel)
   - 2.2 [Processes](#22-processes)
   - 2.3 [Threads](#23-threads)
   - 2.4 [PCB and TCB](#24-pcb-and-tcb)
   - 2.5 [Process Lifecycle](#25-process-lifecycle)
   - 2.6 [Memory Layout, Virtual Memory, and the MMU](#26-memory-layout-virtual-memory-and-the-mmu)
   - 2.7 [The Memory Hierarchy Closest to the CPU](#27-the-memory-hierarchy-closest-to-the-cpu)
3. [Parallelism vs Concurrency](#3-parallelism-vs-concurrency)
4. [CPU-Bound vs I/O-Bound](#4-cpu-bound-vs-io-bound)
5. [How a Single Core Juggles Everything](#5-how-a-single-core-juggles-everything)
   - 5.1 [Preemptive Multitasking and Time Slices](#51-preemptive-multitasking-and-time-slices)
   - 5.2 [Context Switching — What Happens](#52-context-switching--what-happens)
   - 5.3 [The Full Cost of a Context Switch](#53-the-full-cost-of-a-context-switch)
   - 5.4 [Process Switch vs Thread Switch](#54-process-switch-vs-thread-switch)
   - 5.5 [The Scheduler](#55-the-scheduler)
6. [Cooperative Multitasking and Event Loops](#6-cooperative-multitasking-and-event-loops)
7. [Threads, Processes, and Coroutines Compared](#7-threads-processes-and-coroutines-compared)
8. [M:N Scheduling — The Hybrid Model Behind Modern Runtimes](#8-mn-scheduling--the-hybrid-model-behind-modern-runtimes)
   - 8.1 [The Two Worlds: Kernel Space and User Space](#81-the-two-worlds-kernel-space-and-user-space)
   - 8.2 [1:1 — Pure OS Threads](#82-11--pure-os-threads)
   - 8.3 [N:1 — Pure User Threads](#83-n1--pure-user-threads)
   - 8.4 [M:N — The Hybrid Solution](#84-mn--the-hybrid-solution)
   - 8.5 [How M:N Works Internally](#85-how-mn-works-internally)
   - 8.6 [Hard Problems M:N Must Solve](#86-hard-problems-mn-must-solve)
   - 8.7 [M:N vs Async/Await](#87-mn-vs-asyncawait)
   - 8.8 [Where M:N Lives Today](#88-where-mn-lives-today)
9. [Shared State and Synchronization](#9-shared-state-and-synchronization)
10. [Async/Await Demystified](#10-asyncawait-demystified)
11. [Language-Specific Concurrency Models](#11-language-specific-concurrency-models)
12. [Common Pitfalls and Anti-Patterns](#12-common-pitfalls-and-anti-patterns)
13. [Practical Decision Framework](#13-practical-decision-framework)
14. [Quick Reference Cheat Sheet](#14-quick-reference-cheat-sheet)

---

## 1. Why This Matters

Every program you write runs on hardware that can only execute one instruction at a time **per core**. Yet modern computers feel like they run hundreds of things simultaneously — a browser, a music player, a database, your IDE — all at once.

That feeling is not magic. It is a carefully engineered illusion built from a few fundamental ideas:

- The **kernel** schedules work onto CPU cores
- **Processes** and **threads** are the units of that work
- **Context switching** creates the illusion of simultaneity on one core
- **Runtimes** (Go, JVM Loom, Node, asyncio) add another scheduling layer on top

Understanding those ideas is what separates developers who write code that *works* from developers who write code that *scales*.

**What you will understand after this guide:**

- Why `async/await` exists and what it actually does under the hood
- Why adding more threads sometimes makes performance *worse*
- Why Node.js can handle thousands of connections with one thread
- Why Python's GIL matters, and when it doesn't
- Why Go can run millions of goroutines while OS threads cannot
- Why context switches are expensive at the hardware level (TLB, caches, pipelines)
- How to choose the right concurrency tool for the job

This is not a summary. We start from OS prerequisites and build all the way to production decisions.

---

## 2. Prerequisites: The OS Layer You Must Understand First

Before concurrency models make sense, you need a clear mental model of what the operating system is doing underneath your code.

### 2.1 The Kernel

The **kernel** is the core of the OS. It sits between hardware and your programs and is responsible for:

- **Process management** — create, schedule, and terminate processes
- **Memory management** — allocate RAM, isolate address spaces
- **Device management** — talk to disks, network cards, keyboards via drivers
- **File system management** — storage and file access
- **System calls** — the API your program uses to request OS services (read a file, create a thread, sleep, etc.)

Think of the kernel as the conductor: applications never touch the CPU or RAM directly. They ask the kernel, and the kernel decides.

Privilege levels matter here:

- **User mode (ring 3)** — your application code
- **Kernel mode (ring 0)** — OS code with full hardware access

Every time your program blocks on I/O, sleeps, or gets preempted by a timer, the CPU crosses into kernel mode. That transition is part of why concurrency has real cost.

### 2.2 Processes

A **process** is an independent program in execution: its code, data, and system resources (memory, CPU time, open files).

**Key characteristics:**

| Trait | Meaning |
| --- | --- |
| **Isolation** | Each process has its own virtual address space. One process cannot freely read/write another’s memory |
| **Resources** | Owns memory, file handles, sockets, and other OS resources |
| **Overhead** | Creating a process is expensive (separate memory, kernel bookkeeping) |
| **IPC** | Processes talk via pipes, sockets, message queues, or shared memory — never by casually sharing variables |

Examples: your browser, your editor, a database server — each is typically one or more processes.

On Linux, every process has:

- **PID** — unique process ID
- **PPID** — parent process ID
- **State** — running, sleeping, stopped, zombie, etc.
- **Owner** — which user started it
- **Resources** — CPU, memory, I/O, file descriptors

Common process types:

| Type | Description |
| --- | --- |
| **Foreground** | Runs in the terminal, blocks input until done |
| **Background** | Runs without blocking the terminal (`script.py &`) |
| **Daemon** | Long-running service (`sshd`, `cron`) |
| **Zombie** | Finished, waiting for parent to collect exit status |
| **Orphan** | Parent exited; re-parented to `init`/`systemd` |

### 2.3 Threads

A **thread** is the smallest unit of execution within a process. Multiple threads in the same process share the same memory space and resources, but each has its own:

- Program counter (where it is executing)
- Register state
- Stack

**Key characteristics:**

| Trait | Meaning |
| --- | --- |
| **Shared resources** | Memory, file handles, sockets are shared — communication is cheap |
| **Lightweight** | Cheaper to create and switch than processes |
| **Parallelism** | Multiple threads can run on multiple cores at once |
| **Risk** | Shared memory → race conditions and deadlocks if unsynchronized |

Example: a browser process may have threads for UI rendering, network I/O, and JavaScript execution.

Relationship:

```
Process (PCB — process-wide identity and resources)
 ├── Thread 1 (TCB — own stack, PC, registers)
 ├── Thread 2 (TCB)
 └── Thread 3 (TCB)
```

A process can be single-threaded or multi-threaded. Threads share the process; they do not get isolation from each other.

### 2.4 PCB and TCB

The OS tracks running work with control blocks.

#### Process Control Block (PCB)

A **PCB** is the kernel data structure that stores everything needed to manage a process — its “identity card.”

| Field | Description |
| --- | --- |
| **Process ID (PID)** | Unique identifier |
| **Program Counter (PC)** | Address of the next instruction to execute |
| **CPU Registers** | Values to restore when the process resumes |
| **Process State** | New, Ready, Running, Waiting, Terminated |
| **Memory Info** | Address space / page table info |
| **Scheduling Info** | Priority, nice value, queue position |
| **I/O Status** | Open files and devices |
| **Accounting Info** | CPU time used, owner, etc. |

When the OS switches processes, it saves state into the PCB and restores another process’s PCB into the CPU.

#### Thread Control Block (TCB)

A **TCB** stores thread-specific execution state:

- Program counter
- Registers
- Stack pointer
- Thread priority / status

Hierarchy:

- **PCB** = process-wide info (memory map, open sockets, credentials)
- **TCB** = per-thread execution details

Switching between processes (PCB + address space) is heavy. Switching between threads in the same process (TCB only) is lighter because the memory map does not change.

### 2.5 Process Lifecycle

Every process moves through states:

```
[New / Created] → [Ready] → [Running] → [Terminated]
                     ↑          |
                     |     (blocks for I/O / event)
                     |          ↓
                     ←──── [Waiting / Blocked]
```

| State | Meaning |
| --- | --- |
| **New** | Process created; OS allocates a PCB |
| **Ready** | Loaded in memory, waiting in the ready queue for a CPU |
| **Running** | Currently executing on a core (one process/thread per core at a time) |
| **Waiting / Blocked** | Waiting for I/O or an event; cannot run until it completes |
| **Terminated** | Finished or killed; OS reclaims resources |

**Program Counter (PC):** a special CPU register holding the address of the next instruction. After most instructions, the PC advances; on jumps/calls, it is updated to a new address.

**CPU registers:** tiny, extremely fast storage inside the CPU for temporary values, return addresses, stack pointer, flags, etc. During a context switch, register values are saved into the PCB/TCB and later restored.

On Linux, lifecycle is often:

1. **Creation** — `fork()` / `clone()`
2. **Execution** — `exec()` loads a new program image
3. **Waiting** — parent may `wait()` for children
4. **Termination** — `exit()` or a signal (`SIGTERM`, `SIGKILL`, …)

You can influence scheduling with **nice** values (`-20` highest priority … `+19` lowest). Nice is a hint; the kernel’s real priority (`PR`) also depends on dynamic fairness rules.

### 2.6 Memory Layout, Virtual Memory, and the MMU

When a process runs, the OS loads it into RAM with a layout like this:

```
High Address
+----------------------+
|      Stack           |  ← grows down (call frames, locals)
+----------------------+
|      Unused          |
+----------------------+
|      Heap            |  ← grows up (malloc / new)
+----------------------+
|      BSS             |  uninitialized globals
+----------------------+
|      Data            |  initialized globals/statics
+----------------------+
|      Text (Code)     |  machine instructions
+----------------------+
Low Address
```

| Segment | Role |
| --- | --- |
| **Text** | Compiled instructions (usually read-only) |
| **Data** | Initialized global/static variables |
| **BSS** | Uninitialized / zeroed globals |
| **Heap** | Dynamic allocation |
| **Stack** | Function frames, locals, return addresses |

#### Virtual vs Physical Memory

| Term | Meaning |
| --- | --- |
| **Virtual address** | What your program sees (logical address) |
| **Physical address** | Real location in RAM |
| **MMU** | Hardware that maps virtual → physical using page tables |
| **TLB** | Cache of recent virtual→physical translations |

Why virtual memory exists:

1. **Protection** — one process cannot overwrite another’s memory
2. **Isolation** — each process can pretend it owns a full address space starting near zero
3. **Efficient sharing of RAM** — only needed pages stay in physical memory; others can live on disk (swap)

This matters for concurrency because **switching processes invalidates or pollutes address-translation caches**. Switching threads inside one process usually does not.

### 2.7 The Memory Hierarchy Closest to the CPU

From fastest/smallest to slowest/largest:

```
CPU Registers
    ↓
L1 Cache  (per core, ~32KB, ~4 cycles)
    ↓
L2 Cache  (per core or shared, ~256KB, ~12 cycles)
    ↓
L3 Cache  (shared among cores, several MB, ~40 cycles)
    ↓
RAM       (~200–300 cycles)
    ↓
Disk / SSD (orders of magnitude slower)
```

Your concurrency design lives inside this hierarchy. Every time you switch tasks:

- The **pipeline** may flush
- The **TLB** may go cold
- The **L1/L2/L3 caches** may hold the wrong working set

That is not a footnote. It is the physical reason “too many threads” kills throughput.

With these prerequisites in place, the rest of concurrency becomes explainable instead of magical.

---

## 3. Parallelism vs Concurrency

These words are used interchangeably in casual speech. In systems, they mean different things.

### Concurrency

**Definition:** Multiple tasks make *progress* over the same stretch of time, even if only one runs at any instant.

Think of a chef boiling water, chopping vegetables, and checking the oven — not by doing three things at once, but by switching whenever one task waits.

```
Time →
Task A: ████░░░░████░░░░████
Task B: ░░░░████░░░░████░░░░
         (one CPU core, switching between tasks)
```

Concurrency is about *structure* — organizing work so progress continues while some tasks wait.

### Parallelism

**Definition:** Multiple tasks run at the *exact same instant* on different CPU cores.

```
Time →
Core 1: ████████████████████  ← Task A
Core 2: ████████████████████  ← Task B
         (two cores, genuinely simultaneous)
```

Parallelism is about *resources* — you need multiple processing units.

### The Key Insight

| | Concurrent | Parallel |
| --- | --- | --- |
| **One core** | Yes | No |
| **Multiple cores** | Yes | Yes |
| **Requires multi-core hardware** | No | Yes |
| **Handles waiting well** | Yes | Not necessarily |

> **Rule of thumb:** Concurrency is a software concern. Parallelism is a hardware concern. You can have concurrency without parallelism. You cannot have useful parallelism without concurrent design.

Context switching is heavily used for concurrency on one core. Parallelism still uses switching when there are more runnable tasks than cores, but the goal is overlapping real execution.

---

## 4. CPU-Bound vs I/O-Bound

Before choosing any concurrency strategy, answer: **what is my bottleneck?**

### CPU-Bound Tasks

The bottleneck is the processor. The program is computing and wants every cycle.

**Examples:** image/video encoding, ML training, cryptography, sorting huge datasets, physics simulations.

**Strategy:** true parallelism — spread work across cores.

```python
# CPU-bound: use multiprocessing, not threading (especially in CPython)
from multiprocessing import Pool

def compress_image(path):
    ...

with Pool(processes=8) as pool:
    pool.map(compress_image, image_paths)
```

### I/O-Bound Tasks

The bottleneck is waiting — disk, network, database, user input. The CPU is idle most of the time.

**Examples:** HTTP requests, DB queries, file reads/writes.

**Strategy:** concurrency — while one task waits, run another. Extra cores alone will not help if the CPU is already idle.

```python
# I/O-bound: async or threading
import asyncio
import aiohttp

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.json()

async def main():
    async with aiohttp.ClientSession() as session:
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
```

### Why This Distinction Is Critical

| Scenario | Wrong choice | Result |
| --- | --- | --- |
| CPU-bound + threads (CPython) | `threading` | GIL blocks parallelism, little/no speedup |
| I/O-bound + processes | `multiprocessing` | Huge overhead, little benefit |
| I/O-bound + async | `asyncio` | Efficient, low overhead |
| CPU-bound + processes | `multiprocessing` | True parallelism |

---

## 5. How a Single Core Juggles Everything

### 5.1 Preemptive Multitasking and Time Slices

The OS gives each task a **time slice** (typically ~1–10 ms). When the slice expires, a hardware timer fires an interrupt. The current task is forcibly paused and the OS takes control — whether the task cooperated or not.

```
OS Timer fires every ~4ms:

[Task A runs 4ms] → [OS preempts A] → [Task B runs 4ms] → [OS preempts B] → [Task A resumes]
```

This is **preemptive** multitasking. Tasks do not choose when to stop; the OS interrupts them.

**Why developers care:**

- A task can be interrupted *between any two instructions*
- That is why shared data needs locks (or message passing)
- The OS scheduler — not your code — chooses what runs next

### 5.2 Context Switching — What Happens

A **context switch** pauses one process/thread and resumes another by saving and loading execution state.

The **context** includes:

- Program counter
- CPU registers and flags
- Stack pointer
- Memory-related info (for processes: page tables / address space)
- Related OS bookkeeping in the PCB/TCB

High-level steps:

```
1. Interrupt / block / yield occurs
2. Save current task state → PCB / TCB
3. Scheduler picks the next task
4. Restore next task state from its PCB / TCB
5. Resume at the saved program counter
```

Without context switching, there is no multitasking illusion on a single core.

### 5.3 The Full Cost of a Context Switch

People often say “a context switch costs 1–10 microseconds.” That is the wall-clock summary. The real cost is a cascade of hardware effects.

#### 1) Kernel mode transition (ring switch)

Before the OS can touch full register state, the CPU moves from user mode to kernel mode:

```
User Space (Ring 3)
    ↓  timer interrupt
    ↓  CPU saves minimal state, changes privilege
Kernel Space (Ring 0)
    ↓  interrupt handler
    ↓  save full register state
    ↓  scheduler picks next task
    ↓  restore next task registers
    ↓  privilege change back
User Space (Ring 3)
    ↓  Task B resumes
```

Every OS context switch pays **two ring transitions** — in and out. Each transition validates permissions, switches stacks, and disrupts the pipeline.

#### 2) Pipeline flush

Modern CPUs are **pipelined**: fetch, decode, execute, and writeback overlap across many instructions. They also use **branch prediction** and speculative execution.

When an interrupt hits mid-pipeline, in-flight work for the old task is wrong context and must be **flushed**. The pipeline then refills with interrupt-handler code.

Cost: on the order of **10–20+ cycles** minimum, and it happens on the way into the kernel and again on the way out.

#### 3) TLB flush (often the expensive one)

The **MMU** translates every virtual address to a physical address. The **TLB** caches recent translations:

```
Virtual Address → [TLB] → Physical Address
                   hit:  ~1 cycle
                   miss: ~100–300 cycles (page-table walk)
```

Different processes have different address spaces. After a process switch, Task A’s TLB entries are meaningless for Task B. The OS must invalidate (or heavily disrupt) the TLB.

```
Before switch: TLB warm with Task A
After switch:  TLB cold for Task B
               every early memory access can miss
               until TLB warms again
```

**Threads vs processes:**

| Switch type | TLB effect |
| --- | --- |
| Threads in same process | Usually little/no full flush (same address space) |
| Different processes | Full or heavy TLB invalidation |

Modern CPUs mitigate with **ASIDs** (Address Space Identifiers), tagging TLB entries so multiple spaces can coexist. Under load, TLB pressure is still real.

#### 4) CPU cache eviction (L1/L2/L3)

While Task A ran, caches filled with A’s code and data. When Task B resumes, those lines are often the wrong working set:

```
Task B after switch:
access → L1 miss → L2 miss → L3 miss → RAM (200+ cycles)
... until caches warm with Task B’s data
```

This **cache warming penalty** can dominate. More competing tasks → colder caches for everyone.

#### Complete cost picture

```
1. Interrupt + ring transition     → pipeline flush
2. Save Task A registers           → relatively cheap
3. Scheduler runs                  → hundreds of cycles
4. Restore Task B registers        → relatively cheap
5. Ring transition back            → pipeline flush again
6. TLB disruption (esp. processes) → many expensive misses
7. Cache cold start                → hundreds to thousands of wasted cycles

Visible total: often ~1,000 – 10,000+ CPU cycles per switch
```

Register save/restore is the cheap part. Ring transitions, pipeline flushes, TLB disruption, and cache warming are where the money goes.

#### Why this invents M:N and async

Once you see the full cost:

- **User-space M:N switching** avoids kernel entry and keeps TLB/cache warmer (same process)
- **Async / event loops** avoid switching almost entirely — one thread, warm caches
- **Thread pools** reduce switch frequency by bounding runnable threads

Hardware overhead is not an implementation detail. It is why concurrent programming looks the way it does.

### 5.4 Process Switch vs Thread Switch

| | Process context switch | Thread context switch (same process) |
| --- | --- | --- |
| **Saved into** | PCB | TCB |
| **Address space** | Must switch (MMU / page tables) | Shared — no full memory switch |
| **TLB** | Often flushed / disrupted | Usually preserved |
| **Cache** | More likely to go cold | More likely to stay warm (shared data) |
| **Cost** | High | Lower |
| **Typical time** | Microseconds (sometimes more) | Nanoseconds to microseconds |

Same-process thread switches are cheaper. Cross-process switches pay the full memory-context bill.

**Thrashing:** too many runnable threads/processes → the machine spends more time switching than doing useful work.

Tools that reduce this:

- Thread pools
- Async I/O
- Coroutines / green threads / goroutines / virtual threads

### 5.5 The Scheduler

The scheduler decides which runnable task gets the CPU next. It navigates a triangle of trade-offs:

```
           Throughput
          (most work done)
               ▲
              / \
             /   \
            /     \
Fairness ◄─────────► Latency
(every task          (respond fast)
 gets a turn)
```

| Algorithm | How it works | Best for |
| --- | --- | --- |
| **Round Robin** | Equal time slices in rotation | General fairness |
| **Priority Scheduling** | Higher priority first | Real-time / interactive systems |
| **CFS (Completely Fair Scheduler)** | Linux default — tracks virtual runtime, prefers who ran least | General-purpose Linux |
| **Shortest Job First** | Prefer jobs expected to finish soon | Batch / minimize average wait |
| **FIFO** | First come, first served | Simple embedded systems |

What developers need:

- You can influence scheduling with `nice` / priorities
- You generally cannot control *exact* run order
- Assuming order without synchronization is a race condition waiting to happen

---

## 6. Cooperative Multitasking and Event Loops

The OS uses preemptive multitasking for processes and kernel threads. Many modern runtimes add **cooperative multitasking** on top.

### How It Works

Tasks voluntarily yield at explicit points — usually when about to wait. The runtime runs another task, then resumes the first when its wait completes.

```
Cooperative multitasking (event loop):

Task A: ──────await──────────────────────resume──────
               ↓                              ↑
Event Loop:   [pick next task]         [A's I/O done]
               ↓
Task B: ──────────────────────────────await──
```

### The Event Loop

```
while True:
    1. Check: any I/O ready? (timers, sockets, completed reads)
    2. Run callbacks / coroutines for ready events
    3. If nothing ready, sleep until something is
```

### JavaScript / Node.js

JavaScript is single-threaded. Node handles huge concurrency because most work is I/O-bound and yields at `await`.

```javascript
async function handleRequest(req) {
    const user = await db.getUser(req.userId);  // yields
    const data = await fetch(externalApi);       // yields
    return { user, data };
}
```

No OS thread switch at each await — just suspend/resume a function on one thread.

**Danger:** one blocking synchronous call freezes the whole loop:

```javascript
// BAD: blocks the entire event loop
app.get('/api/data', (req, res) => {
    const result = heavyCpuComputation(); // everything else waits
    res.json(result);
});
```

### Go’s Goroutine Scheduler (preview of M:N)

Go multiplexes many goroutines onto fewer OS threads:

```
Goroutines (M):  G1  G2  G3  G4  G5  G6  G7  G8
Processors (P):  [P1]          [P2]          [P3]
OS Threads (N):   T1            T2            T3
```

- Block on I/O → park that goroutine, run another on the same OS thread
- CPU-bound → Go can preempt after ~10ms (since Go 1.14)
- Result: millions of goroutines with low overhead

(Full M:N treatment is in section 8.)

---

## 7. Threads, Processes, and Coroutines Compared

| | **Process** | **Thread** | **Goroutine / Green Thread** | **Coroutine (async)** |
| --- | --- | --- | --- | --- |
| **Memory** | Separate address space | Shared in process | Shared (runtime) | Shared (often one thread) |
| **Creation cost** | High (~ms, MB-scale) | Medium (~100µs, ~1MB stack) | Low (~µs, ~KB stack) | Minimal |
| **Scheduling** | OS | OS | Runtime + OS | Runtime (cooperative) |
| **Communication** | IPC | Shared memory + locks | Channels / shared mem | Shared memory |
| **Crash isolation** | Strong | Crash can take process | Often recoverable | Usually kills that task |
| **Best for** | Isolation + CPU parallel | Parallelism / blocking I/O | Massive concurrency | I/O-bound concurrency |

### Choosing the Right Abstraction

```
Is your bottleneck CPU computation?
├── Yes → Processes (or native threads with true parallelism)
└── No (I/O / waiting)
    ├── Need isolation/safety? → Processes
    ├── Language has good async? → Async / coroutines
    └── Need blocking/legacy APIs? → Threads
```

---

## 8. M:N Scheduling — The Hybrid Model Behind Modern Runtimes

M:N scheduling is how systems get *millions* of concurrent tasks without drowning in OS thread cost.

### 8.1 The Two Worlds: Kernel Space and User Space

#### OS Layer (Kernel Space)

The OS manages **kernel threads** — real threads the CPU scheduler knows about. Creating one means allocating a stack (~1 MB default), registering with the kernel scheduler, and letting the OS decide when it runs.

```
OS Kernel:
┌──────────────────────────────────────┐
│  Kernel Thread 1   Kernel Thread 2   │
│       ↕                   ↕          │
│         CPU Scheduler                │
│         (preemptive)                 │
└──────────────────────────────────────┘
```

#### User Layer (User Space)

Your runtime can also manage lightweight units of work without the OS knowing each one: **green threads**, **fibers**, **coroutines**, **goroutines**, **virtual threads**. Same idea: the *runtime* schedules them.

```
Your Program:
┌──────────────────────────────────────┐
│  Task 1   Task 2   Task 3   Task 4   │
│       ↕                              │
│    Runtime Scheduler                 │
└──────────────────────────────────────┘
```

How these layers combine is exactly what **1:1**, **N:1**, and **M:N** mean.

### 8.2 1:1 — Pure OS Threads

**1:1:** one user task = one OS kernel thread.

```
User Tasks:    T1    T2    T3    T4
               |     |     |     |
OS Threads:   KT1   KT2   KT3   KT4
```

Historical default for C/C++/Java (pre-21).

**Works at small scale:**

- True multi-core parallelism
- Simple mental model
- A blocking syscall only blocks that one thread

**Breaks at large scale:**

1. **Creation cost** — 10,000 threads × ~1 MB stack ≈ 10 GB before app data
2. **Context switch overhead** — 1,000–10,000 instructions; TLB/cache effects compound
3. **Scheduler pressure** — OS schedulers are not designed for 100,000 runnable threads per process

This is the heart of the **C10K problem**: how do you serve 10,000 concurrent clients with 1:1 threads?

### 8.3 N:1 — Pure User Threads

**N:1:** many user tasks on **one** OS thread. Runtime schedules everything in user space.

```
User Tasks:   T1   T2   T3   T4   T5   T6
                 \   |   |   |   |   /
                  Runtime Scheduler
                          |
                 One OS Kernel Thread
```

**Attractive:**

- Cheap creation, tiny stacks
- Fast switches (no kernel)
- Huge task counts, tiny memory

**Why it was largely abandoned as a solo model:**

1. **No true parallelism** — 16 cores, you use 1
2. **One blocking syscall freezes everything** on that OS thread

```
T1 calls blocking read() → OS thread blocks → T2..T6 all frozen
```

You had to make *every* I/O non-blocking manually — fragile and library-hostile.

### 8.4 M:N — The Hybrid Solution

**M:N:** M user tasks across N OS threads, with **M >> N**.

Take the good of both:

- From **1:1**: multiple OS threads → parallelism; one block does not freeze the world
- From **N:1**: lightweight user tasks → scale without 1 MB stacks and kernel thrash

```
User Tasks (M):   T1  T2  T3  T4  T5  T6  T7  T8
                   \  |   |  /     \  |   |  /
Runtime:          [Worker 1]      [Worker 2]
                      |               |
OS Threads:          KT1             KT2
                      |               |
                   Core 1          Core 2
```

| Model | User tasks | OS threads | Parallelism | Blocking safe? |
| --- | --- | --- | --- | --- |
| **1:1** | N | N | Yes | Yes |
| **N:1** | Many | 1 | No | No |
| **M:N** | Many | Few (≈ cores) | Yes | Yes (if runtime handles it) |

Typical setup: N ≈ CPU core count; M = thousands or millions of tasks.

**Why it was invented:**

1. Internet scale needed thousands of connections (1:1 could not)
2. Multi-core CPUs arrived (pure N:1 / single-threaded async left cores idle)

M:N keeps OS threads small and stable while letting concurrent tasks grow.

### 8.5 How M:N Works Internally

```
┌─────────────────────────────────────────────────────────┐
│                    User Space                           │
│  Run Queue:  [T3] → [T7] → [T1] → [T9] → ...          │
│  Worker 1: currently running T5                         │
│  Worker 2: currently running T2                         │
│  Worker 3: currently running T8                         │
│  Parked: T4 (network), T6 (sleeping)                    │
└─────────────────────────────────────────────────────────┘
```

- **Run queue** — ready tasks not yet running
- **Workers** — fixed OS thread pool: pick → run → yield/block → pick next
- **Parked tasks** — waiting on I/O/timer/lock; no CPU until ready again

#### Task lifecycle

```
[Created] → [Run Queue] → [Running] → [Finished]
                ↑              |
                |         (yield / block)
                |              ↓
            [Run Queue] ←── [Parked / Waiting]
                  (I/O completes)
```

#### Yielding in user space

When a task waits:

1. Save task context (registers, stack pointer)
2. Mark parked for event X
3. Pop next ready task
4. Restore and resume

Same *shape* as an OS context switch, but:

- No kernel call (~100 ns vs ~1–10 µs)
- No TLB flush (same process)
- Tiny stacks (few KB)
- Runtime controls timing

#### Handling truly blocking syscalls

If a task does a hard blocking `read()`, that *worker OS thread* stalls.

Runtimes respond with:

**A) Convert to async I/O** — intercept and use epoll / kqueue / IOCP; park + resume  
**B) Spawn a temporary OS thread** — keep worker count stable while one is stuck in the kernel

Most real systems use both depending on I/O type.

### 8.6 Hard Problems M:N Must Solve

#### Work stealing

Idle Worker 1 steals from busy Worker 2’s queue so load balances without a central bottleneck.

#### Stack growth

User tasks start tiny (4–8 KB). Deep call chains need more: segmented stacks or stack copying.

#### Preemption of CPU-bound tasks

Pure cooperation starves others if a task never yields. Runtimes add timer-based forced yields (partial preemption).

#### Syscall detection

The runtime must notice blocking syscalls (libc wrapping, or owning the stdlib I/O path). That is why M:N works best when the language runtime owns I/O.

### 8.7 M:N vs Async/Await

Same *problem* (concurrency without 1:1 cost), different levels.

| | Async/Await | M:N Scheduling |
| --- | --- | --- |
| **Expression** | Explicit `await` in source | Spawn tasks; runtime multiplexes |
| **Mechanism** | Compiler → state machine | Runtime scheduler + workers |
| **Threads** | Often N:1 event loop unless you add workers | Multiple OS threads by default |
| **Blocking** | You must never block | Runtime can handle / compensate |
| **Yield** | Explicit | Can be implicit / preemptive |

Many systems combine both: M:N underneath, `await` as explicit cheap yield points.

### 8.8 Where M:N Lives Today

| Runtime | User tasks | OS threads | Notes |
| --- | --- | --- | --- |
| **Go** | Goroutines | `GOMAXPROCS` (≈ cores) | Work stealing; coop + timer preemption |
| **Java Virtual Threads** (21+) | Virtual threads | Platform threads | Existing blocking code often works |
| **Erlang/BEAM** | Processes | ≈ cores | Soft real-time; early M:N pioneer |
| **Haskell GHC** | Haskell threads | Capabilities | Very cheap threads |
| **Tokio (Rust)** | Tasks | Worker pool | Async + work stealing |
| **.NET ThreadPool** | Tasks | Dynamic pool | Work stealing (OS threads, not classic green threads) |
| **Node.js** | Async + libuv pool | Partial | Not full M:N; thread pool for blocking I/O |

**History (compressed):**

```
1993 — Solaris M:N threading
1995 — Erlang lightweight processes
2000s — Linux NPTL makes 1:1 “good enough”; many abandon M:N
2009 — Go brings M:N back cleanly
2012 — Node popularizes async I/O (different approach, same C10K pressure)
2021+ — Java Loom virtual threads
```

Linux 1:1 got fast enough that some ecosystems paused M:N. M:N returned because **1:1 still cannot scale to millions of concurrent tasks**, no matter how cheap one thread becomes.

**Mental model:** restaurant floor — OS threads are tables you can serve at once; user tasks are customers (can vastly outnumber tables); the runtime is the floor manager seating people efficiently.

---

## 9. Shared State and Synchronization

When tasks share memory, bugs become timing-dependent and painful.

### Race Conditions

Outcome depends on interleaving:

```python
counter = 0

def increment():
    global counter
    temp = counter      # read
    temp = temp + 1     # modify
    counter = temp      # write
```

Two threads can both read `0` and both write `1`. Final value is wrong. This is a **read-modify-write race**.

### Deadlocks

```
Thread A: holds Lock 1, waits for Lock 2
Thread B: holds Lock 2, waits for Lock 1
→ forever
```

**Prevention:**

1. Acquire locks in a global consistent order
2. Use timeouts on lock acquisition
3. Prefer channels/queues/actors over manual locking

### Synchronization Primitives

| Primitive | Purpose | Watch out for |
| --- | --- | --- |
| **Mutex / Lock** | One thread in a critical section | Deadlocks; forgot unlock |
| **RWLock** | Many readers OR one writer | Writer starvation |
| **Semaphore** | Cap concurrency at N | Mismatched acquire/release |
| **Condition Variable** | Wait until a condition is true | Spurious wakeups |
| **Atomics** | Lock-free RMW | Hard to reason about |
| **Channel / Queue** | Pass data safely | Blocking on full/empty |

### The Golden Rule

> **Do not share mutable state. Communicate instead.**

Message passing cannot race on state you never share.

---

## 10. Async/Await Demystified

`async/await` is syntactic sugar over coroutines / state machines.

### What `async` does

Marks a function so it returns a coroutine / Promise / Future. The body does not run until scheduled/awaited.

### What `await` does

Suspends the current coroutine and returns control to the event loop. When the waited operation completes, the loop resumes exactly where it left off.

### Under the hood (simplified)

```python
# This...
async def fetch_user(id):
    data = await db.query(f"SELECT * FROM users WHERE id={id}")
    return data

# ...is roughly a state machine:
def fetch_user_state_machine(id):
    future = db.query(...)
    yield future          # suspend
    data = future.result()  # resume
    return data
```

### Async is NOT free

Async makes **waiting** efficient. It does not make CPU work faster.

```python
# WRONG for CPU-bound work
async def resize_images(paths):
    for path in paths:
        await resize(path)  # if resize is CPU-bound, no win

# RIGHT — offload CPU work
async def resize_images(paths):
    loop = asyncio.get_event_loop()
    for path in paths:
        await loop.run_in_executor(None, resize, path)
```

---

## 11. Language-Specific Concurrency Models

### Python

CPython’s **GIL** allows only one thread to execute Python bytecode at a time.

| Use case | Tool | Why |
| --- | --- | --- |
| I/O-bound | `asyncio` | Event loop; GIL less relevant |
| I/O-bound (simple) | `threading` | Threads release GIL around I/O |
| CPU-bound | `multiprocessing` | Separate processes bypass GIL |
| CPU-bound (native) | C extensions / numpy | Can release GIL |

Python 3.13+ has optional free-threaded builds (`--disable-gil`) — evolving.

### JavaScript / Node.js

Single-threaded event loop. Never block it. Use `Promise.all` for concurrent I/O; `worker_threads` for CPU parallelism.

### Go

Goroutines + channels (CSP): share memory by communicating.

```go
ch := make(chan int)
go func() { ch <- heavyComputation() }()
result := <-ch
```

### Java / JVM

Java 21+ **virtual threads** (Project Loom): lightweight M:N-style threads that park on blocking I/O.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var task : tasks) {
        executor.submit(task);
    }
}
```

### Rust

Ownership prevents data races at compile time. Tokio for async; OS threads when needed.

```rust
let handle = tokio::task::spawn(async {
    fetch_data().await
});
let result = handle.await?;
```

---

## 12. Common Pitfalls and Anti-Patterns

### Thread-per-request at scale

```python
# Breaks around thousands–tens of thousands of concurrent users
thread = Thread(target=process, args=(request,))
thread.start()
```

**Fix:** thread pool, async I/O, or virtual threads / goroutines.

### Mixing blocking calls into async code

```python
async def handler():
    time.sleep(5)         # WRONG — blocks the loop
    await asyncio.sleep(5)  # RIGHT — yields
```

### Assuming compound operations are atomic

`counter += 1` is read/modify/write — not atomic. Use locks, atomics, or queues.

### Unbounded fan-out

```go
// Unbounded
for _, url := range millionURLs {
    go fetch(url)
}

// Bounded
sem := make(chan struct{}, 100)
for _, url := range millionURLs {
    sem <- struct{}{}
    go func(u string) {
        defer func() { <-sem }()
        fetch(u)
    }(url)
}
```

### Ignoring cancellation

Always propagate deadlines/contexts so workers stop when callers give up — otherwise you leak work.

### Lock contention collapsing parallelism

If every worker fights one lock, you get serial execution with extra overhead. Prefer finer locks, sharding, or lock-free / message-passing designs.

---

## 13. Practical Decision Framework

```
What is your bottleneck?
│
├── CPU computation
│   ├── Same machine → multiprocessing / native threads
│   ├── Multiple machines → distributed compute (Celery, Ray, Dask, …)
│   └── Extreme perf → Rust/C++/GPU
│
└── I/O / waiting
    ├── Simple / low scale → threads
    ├── High concurrency → async/await or M:N runtime (Go, Loom)
    ├── Need isolation → processes
    └── Mixed I/O + CPU → async + executor/thread pool for CPU parts
```

### Scaling checklist

- [ ] Profiled the bottleneck (not guessed)?
- [ ] CPU-bound or I/O-bound?
- [ ] Shared state protected (or eliminated)?
- [ ] Worker count bounded?
- [ ] Async paths actually non-blocking?
- [ ] Cancellation and timeouts handled?
- [ ] Metrics/tracing to see runtime behavior?

### Complexity budget

```
Simple sync
  → async/await for I/O concurrency
    → thread pool for mixed I/O + CPU
      → multiprocessing for CPU parallelism
        → distributed systems beyond one machine
```

Each step solves a real problem and adds real complexity. Do not jump ahead of evidence.

---

## 14. Quick Reference Cheat Sheet

### Core Concepts

| Concept | One-line definition |
| --- | --- |
| **Kernel** | OS core that schedules CPU, memory, devices, syscalls |
| **Process** | Isolated running program with its own address space |
| **Thread** | Execution unit inside a process; shares memory |
| **PCB / TCB** | Kernel structs storing process / thread state |
| **Concurrency** | Multiple tasks making progress over time |
| **Parallelism** | Multiple tasks running at the same instant |
| **Preemptive multitasking** | OS forcibly interrupts on a timer |
| **Cooperative multitasking** | Tasks yield at `await` / explicit points |
| **Context switch** | Save one task’s CPU state, load another’s |
| **TLB / cache cost** | Why switches are expensive beyond register save |
| **Event loop** | Runtime loop driving async tasks when I/O is ready |
| **1:1 / N:1 / M:N** | How user tasks map onto OS threads |
| **Race condition** | Outcome depends on nondeterministic ordering |
| **Deadlock** | Circular wait for resources — forever |
| **GIL** | CPython lock allowing one bytecode thread at a time |

### When To Use What

| Situation | Tool |
| --- | --- |
| Many network requests | `async/await` or M:N runtime |
| CPU-heavy (Python) | `multiprocessing` |
| CPU-heavy (Go/Rust/Java) | Threads / goroutines / native parallel |
| Simple background work | Thread pool |
| Safe handoff between tasks | Queue / channel |
| Limit concurrent access | Semaphore |
| Short critical section | Mutex / Lock |

### The One Sentence To Keep

> Separate the **unit of concurrency** (task / coroutine / goroutine) from the **unit of parallelism** (OS thread / core). OS context switches are expensive because of rings, pipelines, TLBs, and caches — modern runtimes exist to avoid paying that cost on every wait.

---

*This guide unifies OS fundamentals (kernel, processes, threads, PCB/TCB, virtual memory, context-switch hardware cost) with concurrency models (preemptive vs cooperative, async, and M:N scheduling) and production decision-making. Master this foundation and every higher-level framework — Node, Go, Loom, Tokio, asyncio — becomes a variation on the same ideas instead of a new religion.*
