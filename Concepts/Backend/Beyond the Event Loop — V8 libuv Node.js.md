# Beyond the Event Loop: How V8, libuv, and Node.js Actually Work Together

Most tutorials teach you the event loop — timers, poll, check, close — and stop there. That's useful, but it skips the part that actually explains _why_ the event loop behaves the way it does. The event loop is not the whole story. It's one component living inside libuv, which itself is only half of Node.js. The other half is V8, and the two run **at the same time**, not in sequence.

This article builds the mental model from the ground up: what V8 is, how it hands work off to libuv, how libuv splits work between OS-level mechanisms and a thread pool, and how a completed task finally makes its way back onto the call stack. Once you see that whole picture, the event loop's phases stop looking like arbitrary rules and start looking like the obvious consequence of the architecture underneath them.

---

## 1. The Big Misconception

> ❌ "JavaScript is single-threaded, so Node.js does one thing at a time." ✅ "JS _execution_ is single-threaded. Node.js, as a whole, is not."

Node.js is a runtime built from three cooperating layers:

| Layer                | Language | Role                                             |
| -------------------- | -------- | ------------------------------------------------ |
| **V8**               | C++      | Compiles and executes your JavaScript            |
| **libuv**            | C        | Event loop, async I/O, thread pool, OS interface |
| **Node.js bindings** | C++      | The bridge connecting V8 to libuv                |

![Node.js Runtime Architecture](assets/nodejs-runtime-architecture.png)

V8 is single-threaded, so only one JS callback ever runs at a time. libuv is multi-threaded and event-driven, so I/O, timers, and background work all happen in parallel while V8 is busy. Instead of spawning a thread per request (like traditional multi-threaded servers), Node keeps one JS thread free by **delegating** blocking work — either to the OS's event notification system (for network I/O) or to libuv's thread pool (for file I/O, crypto, DNS lookups, compression).

Keep that delegation idea in mind — it's the thread that ties every section below together.

---

## 2. The V8 Engine

V8 is Google's JavaScript/WebAssembly engine (also used in Chrome). It parses your source code and compiles it into machine code that runs directly on the CPU. It is the execution core of Node.js — but it only knows how to run JavaScript. It has no idea what a file system or a socket is; that's libuv's job.

### 2.1 Ignition and TurboFan — the JIT pipeline

V8 is a **Just-In-Time** compiler: it blends interpretation and compilation at runtime instead of compiling everything ahead of time like C or Rust would.

|Stage|Job|
|---|---|
|**Ignition**|Interpreter. Turns source into bytecode immediately for fast startup, and collects profiling data (which functions run often, what types they see) as it goes.|
|**TurboFan**|Optimizing compiler. Once Ignition marks a function as "hot," TurboFan uses the collected profile to emit highly optimized machine code for it.|

```
JS source → Parser → AST → Ignition (bytecode + profiling)
                              ↓ function becomes "hot"
                            TurboFan (optimized machine code)
```

This pipeline is why Node.js starts fast (interpreter) and gets faster the longer a hot path runs (JIT-compiled code).

### 2.2 Hidden classes and inline caching

JavaScript objects look like free-form key/value bags, but V8 doesn't treat them that way internally. It builds **hidden classes** — blueprints describing an object's property layout in memory.

```js
const user = {};       // hidden class C0 — no properties
user.name = "Alice";   // → C1, name at a fixed offset
user.age = 30;         // → C2, name + age at fixed offsets
```

Objects that add the same properties in the same order share hidden classes, so V8 can look up properties by memory offset instead of by name. The first time a property is accessed, V8 patches the call site with a shortcut ("inline cache"):

- **Monomorphic** — same shape every time → the cached shortcut is reused → near-native speed.
- **Polymorphic / megamorphic** — shape keeps changing → the shortcut is abandoned → V8 falls back to slower, generic property lookup.

Practical takeaway: initialize object properties in a consistent order, and avoid deleting properties or reassigning them to different types on hot objects.

### 2.3 Memory: stack, heap, and generational GC

```
Stack   — call frames, primitives, pointers to heap objects. LIFO, very fast.
Heap    — objects, arrays, closures.
  New Space (Young Gen)  — short-lived objects, frequent "minor GC"
      ↓ survivors get promoted
  Old Space (Old Gen)    — long-lived data, rarer but heavier "major GC"
```

Most objects die young — request-scoped data, temporary buffers — and get cleaned up quickly in New Space. Objects that survive several minor GC cycles (long-lived caches, config objects) get promoted to Old Space, which is swept less often but at higher cost per pass. This is why allocating large temporary objects inside hot paths creates GC pressure, and why long-lived caches belong in structures that don't get recreated per request.

### 2.4 The call stack

The call stack tracks what's currently executing, LIFO:

```js
function greet(name) { return `Hello, ${name}`; }
function main() {
  const msg = greet('Ali');
  console.log(msg);
}
main();
```

```
push main() → push greet() → pop greet() → push console.log() → pop → pop main()
```

**The rule everything else depends on:** the event loop can only push a callback onto the call stack when the stack is _completely empty_. While synchronous code is running, the stack is occupied — even if libuv finished an I/O operation ten seconds ago, nothing can be delivered until V8 is done with what it's currently doing.

---

## 3. How V8 Talks to libuv: Delegation and Yielding

This is the part most explanations skip. When your code calls something asynchronous, here's the actual sequence:

```
① JS calls fs.readFile() / net.connect() / setTimeout()
② The Node.js binding hands the operation to libuv
③ libuv routes it: OS kernel (network) or thread pool (file/crypto/dns.lookup)
④ Control returns to V8 immediately — the call stack is now free
⑤ V8 moves on to the next line without waiting
⑥ Whenever libuv finishes, it queues the callback in the right event-loop queue
⑦ The event loop delivers that callback to the call stack the moment it's empty
```

Two things to internalize:

**V8 and libuv are not sequential — they run in parallel from the moment the process starts.**

- ❌ **Wrong:** V8 finishes all sync code → *then* the event loop starts → *then* libuv starts.
- ✅ **Right:** V8 execution, libuv's event loop, and its thread pool all start together and run simultaneously for the entire lifetime of the process.

| Timeline | What happens |
| --- | --- |
| **0–50 ms** | V8 runs synchronous code (call stack busy). libuv finishes I/O in parallel at ~50 ms. |
| **50–100 ms** | libuv is done; the callback sits in a queue. V8 is still running sync code. |
| **100 ms** | Call stack empties → event loop delivers the waiting callback. |

libuv finished its work at 50 ms but the callback sat in a queue for 50 ms — not because libuv was slow, but because V8's call stack was still occupied. **libuv itself never waits on a completed task; it drops the result in a queue and moves on immediately. It's the queued callback that waits, not libuv.**

**`setTimeout(fn, 0)` proves the point:**

```js
console.log('A');
setTimeout(() => console.log('C'), 0);
console.log('B');
// Output: A, B, C — always, even with a 0ms delay
```

`setTimeout` registers a timer with libuv and returns instantly — it does not pause execution. `A` and `B` are synchronous and guaranteed to run first; `C` only runs once the stack is empty and the timer phase gets a turn.

**Yielding, concretely:** every time V8 hits an async API, it doesn't "wait" — it registers the work with libuv and _yields_ the call stack back to whatever comes next in the script. This is the entire reason Node can serve thousands of concurrent connections on one JS thread: it never blocks itself waiting on I/O; it always yields and lets libuv report back later.

---

## 4. libuv: The Parallel Engine

libuv is the C library that gives Node its async I/O, its event loop, and its thread pool. It starts running the moment the Node process starts, in parallel with V8, and it never stops until there's nothing left pending.

### 4.1 The confusion almost everyone has: thread pool vs. OS threads

These are **not the same thing**, and conflating them is the single most common source of confusion about Node's concurrency model.

| Aspect | **libuv Thread Pool** | **OS-level concurrency (epoll/kqueue/IOCP)** |
| --- | --- | --- |
| What it is | A fixed pool of worker threads (default 4, configurable up to 1024) that libuv manages internally | The kernel's native ability to watch many file descriptors and report only the ready ones |
| What runs there | Blocking C calls that have no true async OS API: file I/O, `dns.lookup`, `crypto.pbkdf2`/`scrypt`, `zlib` | Network I/O - TCP/UDP sockets, pipes |
| Why it's needed | Most operating systems don't expose a real non-blocking file-read API, so libuv fakes async by handing the blocking call to a spare thread | The kernel already supports watching sockets non-blockingly and natively, so no thread is needed at all |
| Scales by | Number of pool threads (bounded, default 4) | Thousands of file descriptors on a single call (`epoll_wait`) |
| Configuration | `UV_THREADPOOL_SIZE` (must be set before the process starts) | Not configurable - it's how the kernel already works |

The reason this table matters: **network I/O never touches the thread pool.** A TCP connection doesn't consume one of your 4 threads. Only file I/O, DNS lookups via `dns.lookup`, crypto, and zlib do. `dns.resolve` (as opposed to `dns.lookup`) also skips the thread pool entirely because it uses c-ares to talk to the network directly instead of calling the blocking `getaddrinfo`.

![libuv I/O Paths](assets/libuv-io-paths.png)

This is also why thread pool saturation is a distinct failure mode from event-loop blocking. If you fire off 5 concurrent `crypto.pbkdf2` calls against a default pool of 4, the 5th one simply waits its turn — and so does every `fs.readFile` or `dns.lookup` behind it, because they all share the same 4 threads. Meanwhile the JS call stack is completely free. This looks like "Node is slow," but the event loop isn't blocked at all — the thread pool is. The fix there is `UV_THREADPOOL_SIZE`; for genuinely CPU-bound JavaScript, the fix is `worker_threads`, which spins up separate V8 isolates with their own event loops rather than sharing the 4-thread pool.

### 4.2 The event loop lives inside libuv

The event loop isn't a separate thing bolted onto libuv — it _is_ libuv's core scheduling mechanism. It starts running the instant the process starts (not after sync code finishes), and it cycles through six fixed phases forever:

```
① timers        → expired setTimeout / setInterval callbacks
② pending cbs    → deferred I/O error callbacks (mostly internal)
③ idle/prepare   → internal libuv housekeeping, not exposed to JS
④ poll ⭐        → I/O callbacks; waits for new I/O when idle
⑤ check          → setImmediate callbacks
⑥ close cbs      → .on('close', ...) handlers
   ↺ back to ① — forever, until nothing is pending
```

Microtask queues (`process.nextTick`, then Promises) are fully drained **between every single phase**, not just once per tick. What changes when your synchronous code finishes isn't that the event loop "starts" — it was already running. What changes is that the call stack becomes free, so the event loop finally gets to push something onto it.

If there's nothing left pending in libuv (no open timers, no in-flight I/O, no active handles) the process exits. This is why a script with only `console.log` statements exits instantly, while one with a live `setTimeout` or an open server socket keeps running.

### 4.3 The poll phase: where epoll turns into a JS callback

The poll phase is where Node spends most of its time, and it's the clearest place to see the FD → callback mapping in action. On Linux, libuv uses `epoll`; macOS/BSD use `kqueue`; Windows uses IOCP — same idea, different kernel API.

**Registering interest (done once, when a socket is created):**

```
server.listen() → libuv creates a socket, kernel assigns fd=12
                → epoll_ctl(epfd, EPOLL_CTL_ADD, fd=12, EPOLLIN)
                → kernel now watches fd=12 for the server's lifetime
```

**What actually lands in the poll queue, and when.** The article so far has said "the poll queue is drained" without saying who puts things into it — there are exactly two sources, and nothing else ever pushes a callback there:

1. **Network I/O**, via `epoll_wait` (step ③ below): the kernel reports a file descriptor is ready, libuv reads/accepts on it, and queues the matching JS callback.
2. **Thread pool completions**: when a worker thread finishes a blocking call — `fs.readFile`, `crypto.pbkdf2`, `dns.lookup`, `zlib` — it hands the result back to libuv's main loop, which queues the callback in this same poll queue. It doesn't matter whether the finished work came from the kernel or from a thread; both land in the same place.

So step ① in the flow below — "is anything already sitting in the poll queue?" — is really asking: _did either of those two sources finish and queue something while V8 was busy running synchronous code?_ If yes, that work gets run before libuv even considers calling `epoll_wait` again.

**The poll phase's actual decision flow:**

```
Enter poll phase
  ① Anything already sitting in the poll queue?
        YES → run all of them, one by one → loop back to ①
        NO  → go to ②
  ② Is a setImmediate() pending?
        YES → leave poll early → go straight to ⑤ check phase
        NO  → go to ③
  ③ epoll_wait(epfd, events, MAX_EVENTS, timeout)
        timeout = time remaining until the next timer fires
        kernel puts the thread to sleep until:
          - a watched FD becomes readable/writable  → wake up → back to ①
          - the timeout expires                      → leave poll → ① timers
```

The check in step ① exists because libuv may have already finished I/O while V8 was busy running sync code — those callbacks are sitting ready. Calling `epoll_wait` (which can block) when work is already waiting would waste time; libuv drains what's ready first.

**From a ready FD to your callback:**

```
Client sends an HTTP request
        ↓
Kernel marks the server's socket FD as readable
        ↓
epoll_wait() wakes up, returns [fd=12]
        ↓
libuv: accept()/read() on that FD — copies data from kernel space to user space
        ↓
The matching JS callback is placed in the poll queue
        ↓
Poll phase: call stack is empty → callback pushed → V8 executes your handler
```

This is the concrete answer to "how does libuv delegate to the call stack when it's free": epoll doesn't call your JavaScript directly. It returns a list of ready file descriptors to libuv; libuv resolves each FD to the callback that was registered against it, queues that callback, and the event loop only pushes it onto the V8 call stack once the stack has nothing else running. File I/O completions from the thread pool are delivered through the exact same poll queue — a thread does the blocking `read()`, and when it's done, the callback joins the same queue as network callbacks and runs through the same mechanism.

### 4.4 Microtasks vs. macrotasks — the priority order

Start with the two categories, because the names alone cause confusion:

- **Macrotasks** are the six phases you just saw — timers, pending callbacks, poll, check, close. One phase = one type of macrotask. The event loop moves through them in a fixed cycle.
- **Microtasks** are not a phase. They are a checkpoint that runs **in between everything else** — after any single callback finishes, and before the loop is allowed to move on to the next callback or the next phase.

There are exactly two microtask queues, and they always drain in this fixed order:

1. `process.nextTick()` callbacks — all of them, first.
2. Promise reactions (`.then`, `.catch`, `.finally`, and what happens after an `await`) — all of them, second.

"Drain completely" is the important phrase: if a `nextTick` callback schedules another `nextTick`, that new one also runs before the loop moves on. Both microtask queues must go fully empty before V8 is allowed to touch the next macrotask.

**Where this checkpoint happens — literally everywhere:**

```
run one synchronous script  →  [drain nextTick, then drain Promises]
run one timer callback      →  [drain nextTick, then drain Promises]
run one poll callback       →  [drain nextTick, then drain Promises]
run one setImmediate cb     →  [drain nextTick, then drain Promises]
   ...and so on, after every single callback, not just once per loop tick
```

So the real mental model isn't "microtasks are a queue somewhere in the cycle" — it's "V8 never hands control back to libuv without first emptying both microtask queues."

**Priority order, slowest to fastest to actually run:**

|Order|What|Category|Runs in|
|---|---|---|---|
|1|Synchronous code|—|Immediately, on the call stack|
|2|`process.nextTick()`|Microtask|Right after the current callback, before anything else|
|3|`Promise.then` / `await` continuations|Microtask|Right after `nextTick` fully drains|
|4|`setImmediate()`|Macrotask|Check phase|
|5|`setTimeout(fn, 0)`|Macrotask|Timers phase (~1ms minimum, not truly 0)|
|6|I/O callbacks|Macrotask|Poll phase|

**A worked example, traced step by step:**

```js
setTimeout(() => {
  console.log('timer 1');
  process.nextTick(() => console.log('nextTick inside timer'));
  Promise.resolve().then(() => console.log('promise inside timer'));
}, 0);

setTimeout(() => console.log('timer 2'), 0);
```

Walk through it:

1. Both timers are registered with libuv; the script has no more sync code, so the timers phase runs.
2. The first timer callback fires: `'timer 1'` prints. Inside it, a `nextTick` and a `.then` get scheduled — but they don't run yet, they just get queued.
3. Before the event loop is allowed to run the **second** timer callback, it must drain the microtask queues completely: `nextTick` first (`'nextTick inside timer'`), then Promises (`'promise inside timer'`).
4. Only now does the timers phase move on to its next entry: `'timer 2'` prints.

```
timer 1
nextTick inside timer
promise inside timer
timer 2
```

The takeaway: microtasks queued _during_ a macrotask always run before the _next_ macrotask of the same type gets its turn — even if that next macrotask was already sitting in the same phase's queue first.

**The one rule worth memorizing:** inside an I/O callback specifically, `setImmediate` always wins over `setTimeout(fn, 0)`. That's not a coincidence — it falls directly out of the phase order: poll flows straight into check, and check comes before the loop wraps back around to timers.

```js
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
// Always: "immediate" then "timeout" — because check runs before the loop reaches timers again
```

Outside of an I/O callback — for example, if both calls sit at the top level of a script — there's no poll-phase context to anchor the ordering, so which one fires first between `setImmediate` and `setTimeout(fn, 0)` is not guaranteed; it depends on how fast the process finished its startup work relative to the 1ms timer threshold.

### 4.5 Critical example: yielding a CPU-heavy task with `setImmediate`

Here's where the poll queue and the check phase actually become useful together, not just theoretical. Say you need to process 100,000 items with a synchronous, CPU-heavy transform. Doing it in one pass blocks the call stack for the entire duration — no other request gets served until it's done:

```js
function processAllAtOnce(items) {
  for (const item of items) heavyCompute(item); // call stack occupied the whole time
}
```

The fix is to break the work into chunks and hand control back to the event loop between chunks, using `setImmediate` as the handoff:

```js
function processChunk(items, index, onDone) {
  const CHUNK_SIZE = 1000;
  const end = Math.min(index + CHUNK_SIZE, items.length);

  for (let i = index; i < end; i++) {
    heavyCompute(items[i]); // still synchronous — blocks only for this slice
  }

  if (end < items.length) {
    setImmediate(() => processChunk(items, end, onDone)); // yield here
  } else {
    onDone();
  }
}
```

Trace what happens on each iteration, in terms of the phases you already know:

1. `processChunk` runs one slice synchronously. For those 1,000 items, the call stack **is** occupied — this trick doesn't make the computation itself non-blocking.
2. `setImmediate(...)` queues the next slice's callback into the **check phase's queue** and returns immediately. The function exits, the call stack is now empty.
3. Because the stack is empty, the event loop is free to advance through its phases _before_ your next chunk runs. Critically, **poll comes before check** — so if a network request was already accepted by `epoll_wait`, or a thread-pool job (a file read, a DNS lookup) finished while your chunk was running, those callbacks are sitting in the poll queue and get to run now, ahead of your next chunk.
4. Only once poll has nothing left does the loop reach the check phase and run the next `processChunk` slice.
5. Repeat until every chunk is processed.

The net effect: your CPU-bound work still runs on the same single JS thread — there's no parallelism gained, and the total CPU time is the same or slightly higher due to the repeated scheduling overhead. What you _do_ get is that other pending work — incoming requests, finished file reads — gets interleaved between chunks instead of being frozen out for the entire duration. The server stays responsive in small slices rather than going silent for one long stretch.

One honest caveat: this is a cooperative-yielding trick, not real concurrency. If the computation is genuinely heavy (seconds, not milliseconds), chunking with `setImmediate` just spreads the pain out — it doesn't remove it. At that point `worker_threads` (see §4.1) is the correct fix, because it actually moves the work to a separate thread with its own event loop instead of sharing time with the main one.

---

## 5. What `await` Actually Does

`await` is often described as "pausing" execution, which is misleading. Here's the real sequence for:

```js
async function getData() {
  console.log('A');
  const res = await fetch('https://api.example.com/data');
  console.log('B');
}
getData();
console.log('C');
```

1. `fetch()` runs synchronously — it creates a socket, hands the connection to libuv (which registers the FD with epoll), and immediately returns a pending Promise.
2. `await` sees a pending Promise and **suspends** `getData()`, removing its frame from the call stack. The stack is now free.
3. `console.log('C')` runs — this is why `C` prints before `B`. Awaiting a pending promise doesn't block anything; it frees the stack for whatever comes next in the script.
4. Later, the kernel marks the socket readable, `epoll_wait` wakes up, libuv reads the response and resolves the fetch Promise. The continuation of `getData()` after `await` is queued as a **Promise microtask**.
5. Once the microtask queue gets its turn, `getData()` resumes and `console.log('B')` finally runs.

**Output: `A`, `C`, `B`.**

The key correction to the common mental model: `await` does not hand the Promise itself to the event loop. The Promise is just a JS object living in V8's heap. What actually gets delegated to libuv is the underlying I/O operation (the socket); the Promise is merely the handle that gets resolved once libuv reports the result back.

---

## 6. Common Pitfalls

**Blocking the event loop with CPU-bound work.** A synchronous heavy computation occupies the call stack completely — every other request served by that process freezes for the duration, regardless of how much I/O is pending elsewhere. Fix: move the work to `worker_threads`, not a bigger thread pool (the thread pool is for libuv's own blocking C calls, not your JS).

**Recursive `process.nextTick`.** Because the nextTick queue must fully drain before the loop can proceed to any phase, a function that keeps re-scheduling itself via `nextTick` will starve timers, I/O, and `setImmediate` indefinitely. Use `setImmediate` for recursive deferrals instead — it yields back to the event loop between iterations.

**Confusing thread-pool saturation with event-loop blocking.** These produce similar symptoms (things feel slow) but have different causes and different fixes — see §4.1. Check whether your JS call stack is actually busy versus whether libuv's 4 worker threads are all occupied.

**Assuming `setTimeout(fn, 0)` means "immediately."** It's coerced to a ~1ms minimum and, more importantly, it can't run until the current synchronous code, all microtasks, and any prior phases finish. It's "as soon as possible," not "right now."

**Ignoring hidden class stability.** Changing an object's shape unpredictably (deleting properties, adding them out of order, reusing an object for different data shapes) degrades V8's inline caches from monomorphic to polymorphic/megamorphic, which is a real, measurable slowdown in hot paths.

---

## 7. The Mental Model, Compressed

If you remember nothing else:

- **V8** executes JS on a single call stack and manages the heap/GC. It knows nothing about I/O.
- **Node.js bindings** are the bridge that lets V8 hand work to libuv and get results back.
- **libuv** runs continuously, in parallel with V8, from process start to exit. It owns the event loop, the thread pool, and the OS event-demultiplexer interface.
- **Network I/O** goes straight to the OS kernel via epoll/kqueue/IOCP — no thread pool involved, ever.
- **File I/O, crypto, `dns.lookup`, and zlib** go to libuv's fixed-size thread pool because the OS doesn't offer a true non-blocking API for them.
- The **poll phase** is where epoll's list of ready file descriptors gets turned into queued JS callbacks — and where thread-pool completions land through the same queue.
- The event loop can only ever push work onto the call stack **when the stack is empty** — that single rule explains almost every "why does this run in this order" question in Node.js.