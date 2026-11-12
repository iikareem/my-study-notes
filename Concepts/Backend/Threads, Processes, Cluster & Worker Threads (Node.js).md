

## 1. Races and locks: threads vs processes

- The **thread** is the unit the OS schedules and executes. A **process** is a container (address space, file descriptors).
- A race condition needs **two executing threads + a shared resource + at least one write + no coordination**.
- **Same process:** threads share heap and globals, so races are easy.
- **Different processes:** memory is isolated, but they can still race on shared external things:
    - files
    - DB rows
    - shared memory / mmap
    - sockets, pipes, Redis keys

|Scope|Lock type|
|---|---|
|Threads in one process|mutex, semaphore, atomics|
|Across processes / machines|`flock`, named semaphores, `SELECT ... FOR UPDATE`, Redis `SET NX`|

**Node angle:**

- JS runs on one thread, so there are no mid-statement races in memory.
- Races still happen across `await` gaps (check-then-act).
- Races also happen across multiple Node processes/pods hitting the same DB or Redis key. Use transactions or a distributed lock.

---

## 2. Communication

### Between threads (same process)

- Shared memory, fast, no copy.
- You must synchronize yourself. Races and deadlocks come from here.

### Between processes (IPC)

|Mechanism|Idea|
|---|---|
|Pipes|One-way byte stream|
|Unix domain sockets|Two-way, local, fast|
|TCP/UDP sockets|Works across machines|
|Shared memory|Fastest, needs locks|
|Message queues|OS-managed queue|
|Signals|Tiny notifications|
|Files|Simple but slow|

**Trade-off:**

- Threads = fast and simple to share, but one bug can crash everything.
- Processes = isolated and safer, but communication is slower.

---

## 3. `worker_threads` memory model

A worker is a real OS thread in the same process, but it has its **own V8 isolate** (own heap, own event loop, own globals). JS objects are **not shared** by default.

|Way|Copy?|Shared at the same time?|
|---|---|---|
|`postMessage(obj)`|Yes (structured clone)|No|
|`postMessage(buf, [buf])`|No (transferred)|No, ownership moves|
|`SharedArrayBuffer`|No|Yes, needs `Atomics`|

- Cannot be cloned: functions, sockets. Class instances lose their methods.
- Races only appear if you opt in with `SharedArrayBuffer`.

---

## 4. Cluster vs Worker Threads

||Cluster|Worker threads|
|---|---|---|
|Unit|Processes|Threads|
|Memory|Isolated|Isolated (or `SharedArrayBuffer`)|
|Goal|Handle **more requests** (throughput)|Offload **heavy CPU work**|
|Communication|IPC (`process.send`)|`postMessage`|
|Who accepts connections|Each process|Main thread only|

**Why a single Node process is limited:** one event loop = one core. At some request rate that thread hits 100% CPU and requests queue.

**Cluster = "zero to hero":** each process is a full Node instance (own V8, event loop, libuv pool, memory). It accepts and handles requests end to end. There are many front doors, so throughput scales.

**Worker = a back room:** the main thread still accepts and parses every request, then forwards to workers (copy cost) and sends results back. The main thread stays the bottleneck for many light requests. Workers are for keeping the event loop **free** from heavy work.

Both use multiple cores. The difference is **where the work is split**, not core usage.

**Rule of thumb:**

- Many requests, each light → **cluster** (or more pods)
- Few requests, each CPU-heavy → **worker threads** (pool, e.g. `piscina`)
- Waiting on DB/network → `async/await` already handles it
- Combine: cluster/pods for throughput + worker pool inside each for heavy tasks

> Cluster doesn't make a single request faster. It increases how many can be handled at once.

---

## 5. libuv: what is shared and what is not

libuv has two parts:

1. **Event loop:** one **per worker thread** (and per process). Not shared.
2. **libuv thread pool** (default 4, `UV_THREADPOOL_SIZE`): used for `fs`, `dns.lookup`, `crypto`, `zlib`. **One per process**, shared by the main thread and all its workers.

||Event loop|libuv thread pool|
|---|---|---|
|Worker threads (same process)|Own each|**Shared**|
|Child process / cluster|Own each|**Own each**|

**Consequence:**

- Cluster with 8 processes = 8 pools of 4, no contention.
- 8 worker threads = they share one pool of 4, so heavy `fs`/`crypto` can bottleneck. Raise `UV_THREADPOOL_SIZE` if needed.

---

child process 

||Clone (OS `fork()`)|From scratch (`fork()` + `exec()`, or Node's `child_process`/cluster)|
|---|---|---|
|**Starts as**|A copy of the parent's state (COW)|A fresh program, no inherited state|
|**Startup cost**|Very cheap, nothing is copied until written|Higher: boot a new runtime, load code, rebuild connections|
|**Inherits**|Memory, open file descriptors, state|Almost nothing, only what you pass (args, env, IPC)|

**1. "Fast" is only about startup.** After the fork, each process is fully independent and "dedicated" too. A write in one is never visible in the other. The clone just gets there faster.

**2. Node only gives you the from-scratch option.** `child_process.fork()` and `cluster` start a new `node` process. Nothing is cloned from your running heap. This is why each cluster worker opens its own DB pool and loads its own modules, and why startup is slower than a raw OS `fork()`.

Some systems use the clone approach on purpose. For example, a "pre-fork" server loads everything once, then forks workers that share the loaded read-only pages through COW, which saves memory and startup time. Node doesn't work that way.

___________

## Quick summary

- Race = shared resource + concurrent write, between threads or across processes.
- Threads share memory (fast, risky). Processes are isolated (need IPC).
- `worker_threads`: own isolate and loop, `postMessage` copies, `SharedArrayBuffer` shares.
- Cluster = many front doors → throughput. Worker = back room → keep the loop free.
- Event loop per worker/process. libuv thread pool per **process**.