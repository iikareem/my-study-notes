---
tags:
  - concurrency
  - operating-system
---

---

## The Full Cost of a Context Switch

### 1. Kernel Mode Transition (Ring Switch)

Before the OS can even touch registers, the CPU must switch from **user mode (ring 3)** to **kernel mode (ring 0)**. This is not free.

```
User Space (Ring 3)
    ↓  [timer interrupt fires]
    ↓  CPU saves minimal state to kernel stack
    ↓  CPU changes privilege level
Kernel Space (Ring 0)
    ↓  OS interrupt handler runs
    ↓  OS saves full register state
    ↓  OS scheduler picks next task
    ↓  OS restores next task's registers
    ↓  CPU changes privilege level back
User Space (Ring 3)
    ↓  Task B resumes
```

Every context switch requires **two ring transitions** — into the kernel and back out. Each transition forces the CPU to validate permissions, switch stack pointers, and flush parts of the processor pipeline.

---

### 2. Pipeline Flush

Modern CPUs do not execute one instruction then wait for the result. They use **pipelining** — fetching, decoding, and executing many instructions simultaneously, overlapping stages.

```
Normal pipelined execution:

Cycle:     1    2    3    4    5    6    7
Instr 1: [Fetch][Decode][Execute][Write]
Instr 2:        [Fetch][Decode][Execute][Write]
Instr 3:               [Fetch][Decode][Execute][Write]
Instr 4:                      [Fetch][Decode][Execute]
```

The CPU also uses **branch prediction** — it guesses which way an `if` will go and starts executing that path speculatively before the condition is even evaluated.

When a timer interrupt fires mid-execution, the CPU has instructions at various stages of the pipeline — some fetched, some decoded, some half-executed speculatively. All of that in-flight work is wrong context. It must be **flushed entirely**.

```
Interrupt fires here:
                              ↓
Cycle:     1    2    3    4    5    6
Instr 1: [Fetch][Decode][Execute][Write]
Instr 2:        [Fetch][Decode][FLUSH]
Instr 3:               [Fetch][FLUSH]
Instr 4:                      [FLUSH]

All in-flight instructions are discarded.
Pipeline must refill from scratch with kernel interrupt handler code.
```

This flush and refill costs **10–20 cycles** at minimum. On a 3 GHz CPU that is several nanoseconds of pure waste, and it happens twice per context switch (into kernel, back out).

---

### 3. TLB Flush (The Expensive One)

This is the biggest hidden cost and the one most explanations skip.

The CPU has a component called the **MMU (Memory Management Unit)** that translates virtual addresses (what your program sees) to physical addresses (actual RAM locations). Every memory access — every variable read, every function call — goes through this translation.

Because translation is slow if done from scratch every time, the CPU caches recent translations in the **TLB (Translation Lookaside Buffer)** — a small, extremely fast on-chip cache, typically 64–1024 entries.

```
Virtual Address → [TLB lookup] → Physical Address
                     ↓
              TLB hit: ~1 cycle
              TLB miss: ~100-300 cycles (must walk page tables in RAM)
```

When you switch between **processes**, their virtual address spaces are completely different. The TLB entries from Task A are meaningless for Task B. The OS must **invalidate the TLB**.

```
Before context switch:   TLB is warm, full of Task A's translations
Context switch happens:  TLB is flushed (all entries invalidated)
Task B starts running:   TLB is cold — every memory access is a miss
                         Each miss: ~100-300 cycles to refill from page tables
                         Until TLB warms up again: severe slowdown
```

A warm TLB means ~1 cycle per memory access. A cold TLB after a flush means ~100–300 cycles per access until it fills back up. For a task that touches many memory locations at startup, this translates to thousands of wasted cycles just getting back to normal speed.

**Important nuance — threads vs processes:**

Threads within the same process share the same virtual address space, so the TLB does **not** need to be fully flushed when switching between threads of the same process. This is one of the core reasons threads are cheaper to switch between than processes.

```
Thread switch (same process):   partial or no TLB flush needed
Process switch (different):     full TLB flush required
```

Modern CPUs partially mitigate this with **ASID (Address Space Identifiers)** — tags that let the TLB hold entries from multiple address spaces simultaneously, reducing how much gets flushed. But it is still a real cost, especially under load.

---

### 4. CPU Cache Eviction (L1/L2/L3)

Beyond the TLB, the CPU has larger caches — L1 (fastest, ~32KB), L2 (~256KB), L3 (~several MB) — that hold recently used data and instructions.

```
Access cost:
L1 cache hit:   ~4 cycles
L2 cache hit:   ~12 cycles
L3 cache hit:   ~40 cycles
RAM access:     ~200-300 cycles
```

When Task A was running, the caches filled up with _Task A's_ data and code. When Task B resumes, those caches are full of the wrong data. Task B's memory accesses miss the cache and go all the way to RAM until the caches warm back up.

```
Task B resumes after switch:
[access data] → L1 miss → L2 miss → L3 miss → RAM (200+ cycles)
[access data] → L1 miss → L2 miss → L3 miss → RAM (200+ cycles)
... (repeats until caches fill with Task B's data)
[access data] → L1 hit  (4 cycles) ← finally warm
```

This **cache warming penalty** can account for thousands of wasted cycles per context switch, and it compounds — the more tasks you have competing for cache space, the colder everyone's cache stays.

---

### The Complete Picture

```
Context Switch Full Cost Breakdown:

1. Timer interrupt + ring transition    →   pipeline flush, ~10-20 cycles lost
2. Save Task A registers to kernel      →   ~10s of cycles, cheap
3. OS scheduler runs                    →   hundreds of cycles
4. Restore Task B registers             →   ~10s of cycles, cheap
5. Ring transition back to user space   →   pipeline flush again
6. TLB flush (if different process)     →   subsequent misses cost 100-300 cycles each
7. CPU cache cold start                 →   hundreds to thousands of wasted cycles
                                             until caches warm up

Total overhead visible as: 1,000 – 10,000+ CPU cycles per switch
```

The register save/restore (steps 2 and 4) is genuinely cheap. Everything else — the ring transitions, pipeline flushes, TLB invalidation, and cache warming — is where the real cost lives.

---

### Why This Is Why M:N and Async Exist

Once you see this full picture, the motivation for every alternative becomes obvious:

- **M:N user-space task switching** avoids the ring transition and pipeline flush entirely — switching between user tasks is just saving/restoring registers in user space, no kernel involvement, TLB stays warm (same process)
- **Async/event loop** avoids switching altogether — one thread, one TLB state, caches stay warm, no flush ever
- **Thread pools** reduce switch frequency — fewer threads means fewer opportunities for this cascade of costs to accumulate

The hardware overhead is not an implementation detail. It is the fundamental reason the entire field of concurrent programming looks the way it does.