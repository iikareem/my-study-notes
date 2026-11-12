# How Go Runs Your Code: Goroutines, the GMP Scheduler, and a Node.js Comparison

## Introduction

Every backend developer eventually hits the same question: _how do I handle thousands of things happening at once?_

The old answer was one OS thread per connection. It's simple, but it falls apart quickly, because threads are heavy and the kernel can only juggle so many. Two modern answers emerged. **Node.js** said: use a single thread and make the _developer_write async code so the thread never waits. **Go** said: let the developer write plain, blocking-looking code, and make the _runtime_ so good at juggling tasks that waiting becomes almost free.

Both approaches work, and both use the same OS mechanism (epoll) for network I/O. But they feel completely different to write, fail in different ways, and have very different answers to "what happens when something takes too long?"

This article explains how Go actually runs your code. We'll cover why OS threads are a bad fit for massive concurrency, what a goroutine really is, and how the **GMP scheduler** (Goroutines, Machines, Processors) spreads work across your CPU cores. Along the way, every idea is mapped to its Node.js equivalent, so if you know the event loop, you'll have a mental bridge into Go's runtime.

**By the end you'll be able to answer:**

- Why are goroutines cheap and OS threads expensive?
- What are G, M, and P, and why does P exist?
- What are the local and global queues, and how does the scheduler stay fair?
- What happens when a goroutine blocks on a channel, the network, a file, or a CPU loop?
- When should you pick Go over Node, and when not?

---

## 1. Two philosophies

**Node.js: one thread, and you keep it busy.**  
Your JavaScript runs on a single thread. To avoid wasting it, you write async code (callbacks, promises, `async`/`await`) so that every wait hands control back to the event loop. The mental model is _one cook who must never stand still_.

**Go: many tasks, and the runtime keeps them moving.**  
You write code that looks blocking: `conn.Read()`, `<-ch`. Underneath, the runtime parks the waiting task and runs another one. It also spreads tasks across all your cores by default. The mental model is _many cooks, and a manager who moves them around_.

In short, Node asks you to write code in an async shape. Go lets you write sequential code and makes waiting cheap enough that you don't care.

### Cooperative vs preemptive scheduling

This is the lens for understanding everything else in the article.

- In **cooperative** scheduling, the _task itself_ decides when to stop (it yields). If it never yields, everyone else starves. Shared state is safe because nobody can interrupt you mid-operation.
- In **preemptive** scheduling, the _scheduler_ forces a task to stop after a time slice. Nothing can hog the CPU, but you can be interrupted anywhere, so shared state is risky.

**Node is purely cooperative.** A callback runs until it returns or hits `await`, and nothing can interrupt it. That's why plain JS variables have no data races, and also why a single CPU-heavy loop freezes every request.

**Go is cooperative by default, with a preemptive safety net.** Goroutines yield naturally when they block on a channel, lock, or network call. If one runs too long (about 10 ms), the runtime forces it off the CPU. That keeps Go fair, and it's also why you need locks.

> The OS always preempts _threads_. The real question is whether the runtime **above** the OS preempts your _tasks_. Node's doesn't. Go's does.

---

## 2. What each process looks like

**A Node process:**

```
┌──────────────────────── Node process ────────────────────────┐
│ Main thread: V8 + event loop (libuv)                         │
│   ├─ runs ALL your JS                                        │
│   └─ network I/O via epoll/kqueue/IOCP                       │
│                                                              │
│ libuv threadpool (default 4): fs, dns.lookup, crypto, zlib   │
│ worker_threads (optional): separate V8 isolate + own loop    │
└──────────────────────────────────────────────────────────────┘
```

**A Go process:**

```
┌───────────────────────── Go process ─────────────────────────┐
│ Go runtime (compiled into your binary, no VM)                │
│                                                              │
│   Goroutines (G)  ──scheduled by Go scheduler──►             │
│   OS threads (M)  ──scheduled by the kernel────► CPU cores   │
│                                                              │
│ Netpoller: epoll/kqueue/IOCP under the hood                  │
└──────────────────────────────────────────────────────────────┘
```

The key differences:

- **Threads running your code:** Node runs exactly one. Go runs up to `GOMAXPROCS` (default: your CPU core count) in parallel.
- **Network I/O:** the same mechanism on both sides. Node uses epoll via libuv, and Go uses epoll via its netpoller.
- **File I/O:** Node offloads to a fixed pool of 4 threads. Go lets the thread block and hands its processor to another thread (explained in section 6).
- **Execution:** Node's V8 uses a JIT compiler. Go compiles ahead of time to native code, with the runtime built into your binary and no VM.
- **Data races:** impossible on plain JS variables (single thread), but possible in Go.

---

## 3. What is a goroutine?

A goroutine is a **user-space lightweight thread** managed by the Go runtime, not by the OS.

```go
go handleRequest(conn)   // starts a goroutine, returns immediately
```

To see why Go bothers with its own tasks, look at what's wrong with using OS threads for everything.

### Why OS threads are expensive

**1. Context switches are costly.**  
Switching threads means trapping into the kernel, saving and restoring registers, and running the kernel scheduler. That direct cost is around 1-2 µs. The hidden cost is often bigger: the new thread's data isn't in the CPU caches, so it starts slow and runs cold for a while. At thousands of threads, the CPU spends a real share of its time just switching.

**2. Every thread carries a big stack.**  
Each thread reserves roughly 1-8 MB of stack, plus kernel-side structures. Most of that space is never used, but it still counts against address space and limits. This is why you can run thousands of threads, not hundreds of thousands.

**3. Creating a thread is a syscall.**  
Spawning one goes through the kernel and allocates kernel structures. That's too heavy to do per request or per small task, so systems end up needing thread pools.

**4. Blocking wastes a whole thread.**  
A thread waiting on a socket sits idle in the kernel, holding its stack and doing nothing. In a thread-per-connection server, 10,000 mostly idle connections means 10,000 parked threads. This is the classic C10K problem.

**5. The kernel doesn't know your app.**  
The OS scheduler sees only threads. It can't tell that one is waiting on a channel or that another holds a lock your whole app needs. It may preempt a thread at a bad moment, such as while it holds a mutex.

**6. More threads than cores means time slicing.**  
Once threads outnumber cores, they take turns, adding switch overhead without adding throughput.

### How goroutines fix this

||OS thread|Goroutine|
|---|---|---|
|Initial stack|~1-8 MB|~2 KB, grows and shrinks|
|Creation|Syscall|Cheap function call|
|Context switch|Kernel, ~1-2 µs|User-space, ~100-200 ns|
|Practical count|Thousands|Hundreds of thousands+|

The speedup comes from three design choices:

- **Small, growable stacks.** The runtime grows a goroutine's stack only when needed, so memory per task stays tiny.
- **User-space switching.** Switching goroutines saves just a few registers. There's no kernel trap and no kernel scheduler involved.
- **The runtime knows when a goroutine waits.** It sees the channel, lock, or network call directly, so it parks that goroutine and runs another on the same thread. Nothing sits idle, and the OS never has to intervene.

### Node mapping

A goroutine is roughly one in-flight `async` function, with one key difference:

- **Node `async` is stackless.** At `await`, state is saved to the heap and the stack unwinds.
- **A goroutine is stackful.** It keeps its own stack, and the scheduler just swaps stacks.

The result is that Go has no "function coloring" problem. Any function can block, and any function can be launched with `go`.

---

## 4. The GMP scheduler

Go uses **M:N scheduling**: many goroutines (M) are multiplexed onto a smaller number of OS threads (N), all in user space. The scheduler has three moving parts:

- **G (goroutine):** your function as a task, with its own stack and saved state. You create these with `go f()`.
- **M (machine):** a real **OS thread**, and the only thing that actually executes instructions. The runtime creates Ms on demand.
- **P (processor):** a runtime struct that holds a **permit to run Go code** plus a **local run queue** of goroutines. It's not hardware and not an OS process. The runtime creates all Ps at startup.

```
        Global run queue
                │
   ┌────────────┼────────────┐
   ▼            ▼            ▼
  P0           P1           P2       ← count = GOMAXPROCS
 [G G G]      [G G]        [G G G]   ← local queues
   │            │            │
   M0           M1           M2      ← OS threads (an M needs a P)
   │            │            │
 Core 0       Core 1       Core 2
```

**The chain:** a `G` runs on an `M` (while that M holds a `P`), and the OS runs the `M` on a CPU core.

> **Think of a desk and a worker.** The P is a desk with a tray of tasks. The M is a worker. A worker can only do Go work while sitting at a desk, and a desk is never used by two workers at once. Workers come and go, but the desk and its tray stay and get handed to whichever worker is free.

### The mandatory rules

1. **An M must hold a P to run Go code.** No P, no running goroutines.
2. **The number of Ps equals `GOMAXPROCS`** (default: CPU cores). This caps how many goroutines run **in parallel**.
3. **An M holds the whole P exclusively**, along with its queue.
4. **Each P runs one goroutine at a time**, pulled from its local queue.
5. **Ms can outnumber Ps.** An M blocked in a syscall doesn't hold a P.
6. **The OS only knows Ms.** It has no idea Ps or Gs exist.

### Why does P exist? Why not just G → M?

- **It limits parallelism.** Only `GOMAXPROCS` threads run Go code at once, so you never have more running threads than cores.
- **It removes the global lock.** With one shared queue, every thread would fight over the same lock. Per-P queues mean threads rarely contend.
- **It enables syscall handoff.** If an M blocks in the kernel, the runtime gives its P to another M, so queued goroutines keep running.
- **It enables work stealing.** An idle P can take half of another P's queue.

### Cores vs Ps

With 4 cores and `GOMAXPROCS=4`, four goroutines truly run in parallel. With fewer Ps than cores you waste hardware, and with more Ps than cores you gain no speed, only switching overhead. Leaving the default alone is almost always right.

---

## 5. Local queue, global queue, and fairness

Each P has a **local run queue** that holds up to 256 goroutines. It's lock-free for its owner, so it's fast, and it's where goroutines normally live.

There is also one **global run queue** for the whole runtime. It's shared by all Ps and protected by a mutex, so it's slower. It exists for overflow and fairness.

### When does a goroutine land on the global queue?

1. **Local queue overflow.** When a full local queue gets another goroutine, half of it plus the new one move to the global queue in one batch (one lock acquisition instead of about 129).
2. **Preemption.** A goroutine that exceeds its time slice goes to the global queue, so a CPU hog can't keep jumping back onto its own P.
3. **No P available.** For example, a goroutine readied by a thread returning from a syscall that can't get a P right away.

### How a P finds work

In order:

1. The `runnext` slot (the goroutine just spawned or woken, kept for cache locality), then the **local queue**
2. The **global queue**
3. The **netpoller** (goroutines whose I/O is now ready)
4. **Steal half** of another P's local queue

### How fairness is kept

- **Every 61st scheduling tick**, a P checks the global queue _first_, even if its local queue has work. Without this, global goroutines could starve while busy Ps feed themselves locally. (61 is prime so it doesn't sync up with workload patterns.)
- **When its local queue is empty**, a P grabs a _batch_ from the global queue, so it doesn't need the lock again soon.
- **Time-slice preemption** stops CPU hogs from monopolizing a P.
- **Work stealing** evens out load across Ps.

One detail worth remembering: it's the **P** that counts ticks and checks the global queue. The M just executes on behalf of whichever P it holds.

---

## 6. What happens when a goroutine waits?

**Channel, mutex, `time.Sleep` (cooperative yield).**  
The goroutine is **parked**: set aside, not in any run queue. The M immediately runs the next goroutine. This is the equivalent of `await`, without the keyword. When the goroutine is ready again, it goes back on a queue.

**Network I/O (cooperative yield).**

```
G calls conn.Read() ─► not ready ─► fd registered in epoll, G parked
                                    M runs other goroutines
data arrives ─► epoll event ─► G marked runnable ─► some P runs it
```

This is the libuv model, done by the runtime. Your code looks like a plain blocking `Read`.

**Blocking syscall (file I/O, cgo).**  
Regular files can't use epoll, which is the same reason libuv has a threadpool. Here the **M itself** blocks in the kernel, so the runtime takes its P away and hands it to another M. Other goroutines keep running. Node solves this with a fixed pool of 4 threads, while Go spawns or reuses threads dynamically.

**CPU-bound loop (where the two models really differ).**

```js
// Node (cooperative): freezes EVERYTHING
while (true) {}
```

```go
// Go (preemptive safety net): other goroutines still run
for {}
```

Other Ps keep running on other cores. Even on a single P, a background monitor thread called **sysmon** notices the goroutine running past about **10 ms** and preempts it with a signal. This is **asynchronous preemption**, available since **Go 1.14**.

A few facts worth knowing about preemption:

- The ~10 ms threshold is fixed, with no setting to change it.
- It's a **pause, not a kill**. The goroutine resumes from the exact same instruction later.
- If no other goroutine is waiting, the cost is tiny.
- The real knob for CPU-heavy work is `GOMAXPROCS`.

---

## 7. Shared state and communication

Because Node is cooperative and single-threaded, your code runs until the next `await` and nobody touches your state. Go gives up that guarantee: you can be interrupted anywhere, and other goroutines run in parallel. Data races are real, so run tests with `go test -race`, and protect shared state with channels, `sync.Mutex`, or `atomic`.

```go
ch := make(chan int, 10)          // buffered channel = bounded queue
go func() { ch <- compute() }()
result := <-ch                    // parks the goroutine until a value arrives
```

> **Go proverb:** _"Don't communicate by sharing memory; share memory by communicating."_

A buffered channel gives you **built-in backpressure**. When it's full, the producer blocks, much like `highWaterMark` and `drain`in Node streams.

---

## 8. Concept mapping: Node → Go

|Node.js|Go|Note|
|---|---|---|
|Event loop|GMP scheduler|N loops with work stealing|
|libuv epoll|Netpoller|Same OS mechanism|
|libuv threadpool|Extra Ms + P handoff|Dynamic, not fixed at 4|
|`async` fn / Promise|Goroutine|Stackful vs stackless|
|`await`|Any blocking op|Implicit|
|Microtasks / macrotasks|None|One flat scheduler|
|`Promise.all`|`sync.WaitGroup` / `errgroup`||
|`p-limit`|Worker pool / semaphore channel||
|Stream backpressure|Buffered channel||
|`Promise.race`|`select`||
|`AbortController`|`context.Context`||
|`worker_threads`, `cluster`|Not needed|Already multi-core|
|Rejected promise / `try-catch`|`error` values; `panic`|An unrecovered panic in **any** goroutine kills the process|

The same task in both worlds, with bounded concurrency:

```js
// Node
const limit = pLimit(10);
await Promise.all(items.map(i => limit(() => process(i))));
```

```go
// Go
jobs := make(chan Item)
var wg sync.WaitGroup
for w := 0; w < 10; w++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for it := range jobs { process(it) }
    }()
}
for _, it := range items { jobs <- it }
close(jobs)
wg.Wait()
```

---

## 9. When to use which

**Node.js shines when:**

- The work is I/O-bound with light CPU: API gateways, BFFs, GraphQL, proxies
- You have many idle connections (WebSockets, SSE)
- You're gluing services together and shuffling JSON
- You want one language (TypeScript) across frontend and backend

**Node.js struggles when:**

- CPU-bound work blocks the loop for _every_ request
- One sync mistake spikes p99 for everyone (`readFileSync`, a bad regex, a huge `JSON.parse`)
- You need many cores, since workers and cluster add memory and complexity
- The 4-thread libuv pool becomes a bottleneck under heavy fs, crypto, or zlib work

**Go shines when:**

- You're building high-concurrency network services
- Workloads mix CPU and I/O
- You're writing infra and tooling: proxies, sweepers, pipelines, CLIs, load generators
- You want predictable latency, low memory use, and a single static binary

**Go hurts when:**

- **Data races** creep in (the price of preemptive and parallel)
- **Goroutine leaks** happen (goroutines blocked forever on a channel)
- **Unbounded `go`** floods a dependency: 100k goroutines hitting a DB or API will exhaust connections or rate limits, so always bound with a pool
- A goroutine panics without `recover` and crashes the whole process
- Error handling gets verbose, or the ecosystem is thinner for some CRUD-heavy business domains

The mental load differs too. In Node, the rule is "don't block the loop." In Go, it's "protect shared state and bound your concurrency." A blocking bug in Node hurts the whole loop, while in Go its blast radius is usually one goroutine or one P.

---

## 10. Mandatory takeaways

1. **G** is the task, **M** is the OS thread, and **P** is the permit plus a local queue.
2. **Ps = `GOMAXPROCS`**, which is the maximum number of goroutines running in parallel.
3. An **M needs a P** to run Go code, and a blocked M loses its P.
4. Each P runs **one goroutine at a time** from its local queue (256 max, with overflow going to the global queue).
5. The **global queue** handles overflow and preempted goroutines, and each P checks it every 61 ticks for fairness.
6. A goroutine is **parked** when it waits on channels, locks, or the network, and **preempted** after about 10 ms of CPU.
7. **Idle Ps steal** half of another P's queue.
8. On a **blocking syscall**, Go hands the P to another thread. Node uses the libuv threadpool instead.
9. Both runtimes use **epoll** for network I/O. The difference is who turns "wait for data" into "do other work": libuv plus your callbacks, or the netpoller plus parked goroutines.
10. **The biggest mindset shift from Node:** you lose single-thread safety, so races, leaks, and unbounded goroutines become your problems.

**Rule of thumb:** for I/O glue that benefits from a shared TypeScript ecosystem, choose Node. For concurrency-heavy, CPU-touching, long-running services and tooling, choose Go.

---

## Interview-ready summary

> Go uses an M:N scheduler inside the process. Goroutines (G) are lightweight tasks. Ps are logical processors, defaulting to the number of CPU cores, and each P owns a local run queue. An M, which is an OS thread, must hold a P to run goroutines, and it runs them one at a time from that P's queue. Local queues avoid lock contention. A global queue handles overflow and preempted goroutines, and each P checks it every 61 ticks to keep scheduling fair. When a goroutine blocks on a channel, lock, or network call, it's parked and the M runs the next one. If it hogs the CPU for about 10 ms, it's preempted. Idle Ps steal work from others, and if an M blocks in a syscall, its P is handed off to another thread.

---

I cut the tables down to two, the goroutine-vs-thread numbers and the Node→Go mapping, since those are the places where a table helps most. I can also save this as a downloadable `.md` file if you'd like.