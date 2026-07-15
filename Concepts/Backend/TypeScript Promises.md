---
tags:
  - backend
  - typescript
---

Let's build this up layer by layer.

---

## 1. The Problem That Promises Solve

When you run JavaScript, most things are **synchronous** — line 2 runs after line 1 finishes. But some operations take unknown time: reading a file, making an HTTP request, querying a database.

The old solution was **callbacks**:

```javascript
readFile("data.txt", function(err, data) {
    // This runs later, when the file is ready
});
// Code here runs BEFORE the file is ready
```

Callbacks work, but they have a structural problem: you can't `return` a value from them, you can't `try/catch` them naturally, and nesting them gets ugly fast.

**A Promise is a solution to this:** it's an object that represents a value that doesn't exist yet but will at some point.

---

## 2. What a Promise Actually Is

A Promise is an object with **one internal state** that can only move forward — never backward:

```
pending  ──→  fulfilled (has a value)
         ──→  rejected  (has an error)
```

Once it moves from `pending`, it's frozen forever. A fulfilled promise never becomes rejected. A rejected promise never becomes fulfilled.

When you write:

```javascript
const p = fetch("https://api.example.com/data");
```

`p` is immediately a Promise object — right now, in this line — but it's in the `pending` state. The HTTP request is running in the background. At some future point, it will transition to `fulfilled` or `rejected`.

---

## 3. How You Consume a Promise

```javascript
p.then(value  => { /* runs if fulfilled */ })
 .catch(error => { /* runs if rejected  */ });
```

Or with `async/await`, which is just cleaner syntax for the same thing:

```javascript
try {
    const value = await p; // waits for fulfillment
} catch (error) {          // catches rejection
}
```

This is the **consumer** side. Now let's look at the **producer** side — where the promise is created and controlled.

---

## 4. The Promise Constructor: `resolve` and `reject`

When you create a Promise yourself, you pass it a function called the **executor**. The executor receives two arguments: `resolve` and `reject`.

```javascript
const p = new Promise((resolve, reject) => {
    // executor runs immediately, synchronously
    // resolve and reject are functions given to you by the JS engine
});
```

These two functions are your **controls** over the promise's state:

- Calling `resolve(value)` → transitions the promise from `pending` to `fulfilled` with that value.
- Calling `reject(error)` → transitions the promise from `pending` to `rejected` with that error.

A practical example:

```javascript
const p = new Promise((resolve, reject) => {
    setTimeout(() => {
        resolve(42); // after 1 second, fulfill with 42
    }, 1000);
});

const result = await p; // waits 1 second
console.log(result);    // 42
```

The executor sets up an async operation (`setTimeout`), and when it completes, it calls `resolve`. The promise transitions, and the `await` unblocks.

---

## 5. The Critical Insight: `resolve` and `reject` Are Just Functions

This is the part most people don't internalize.

`resolve` and `reject` are plain JavaScript functions. You can store them in a variable. You can pass them to another function. You can store them in an object. You can call them from anywhere — even from completely unrelated code.

The promise doesn't care _where_ or _when_ `resolve` or `reject` is called. It only cares _that_ they're called.

```javascript
let storedResolve;
let storedReject;

const p = new Promise((resolve, reject) => {
    storedResolve = resolve; // save them, don't call them yet
    storedReject  = reject;
    // executor returns without calling either — promise stays pending
});

// ... completely separate code, maybe an event handler, maybe a different file ...

storedResolve("done"); // NOW the promise fulfills
```

`p` was pending from the moment it was created. It stayed pending until `storedResolve` was called from outside. The `await p` would have been blocking for exactly that duration.

---

## 6. The Deferred Pattern

**The Deferred pattern** is what you get when you formalize the above idea into a reusable structure.

The word "deferred" means: "the resolution is _deferred_ — it will happen later, from the outside, when conditions are met."

A minimal implementation:

```javascript
function createDeferred() {
    let resolve, reject;

    const promise = new Promise((res, rej) => {
        resolve = res;
        reject  = rej;
    });

    return { promise, resolve, reject };
}
```

Usage:

```javascript
const deferred = createDeferred();

// Pass deferred.promise to whoever needs to WAIT
// Pass deferred.resolve/reject to whoever needs to SIGNAL

// Somewhere else:
deferred.resolve("result"); // triggers the waiting side
```

The key structural property: **the creator of the promise and the settler of the promise are decoupled**. Different pieces of code can hold each side.

---

## 7. Why This Matters for WorkerPool

In liteQ's WorkerPool, here's the communication model:

```
Main thread              Worker thread
    │                        │
    │── postMessage(job) ───►│
    │                        │ (runs the job)
    │◄─ postMessage(result) ─│
    │                        │
```

This is **event-driven** — you send a message and eventually receive one back via an event listener. There's no built-in way to `await` this.

The deferred pattern bridges the gap:

```javascript
const pendingJobs = new Map(); // jobId → { resolve, reject }

function sendJobToWorker(jobId, work) {
    return new Promise((resolve, reject) => {
        pendingJobs.set(jobId, { resolve, reject }); // store them
        worker.postMessage({ jobId, work });          // fire and forget
    });
    // Promise is pending. resolve/reject are sitting in the Map.
}

// Registered ONCE when the worker is spawned:
worker.on("message", ({ jobId, result, error }) => {
    const deferred = pendingJobs.get(jobId);
    pendingJobs.delete(jobId);

    if (error) deferred.reject(new Error(error));
    else       deferred.resolve(result);
    // NOW the promise returned by sendJobToWorker fulfills or rejects
});
```

From the caller's perspective:

```javascript
const result = await sendJobToWorker(42, { type: "resize-image" });
// clean, linear, no event listener visible here
```

Under the hood, the promise was held in `pending` state for the entire duration of the worker thread execution. The event handler, registered once on the worker, holds the key (`resolve`/`reject`) to settle it when the reply arrives.

---

## 8. Summary of the Three Concepts

|Concept|What it is|
|---|---|
|`resolve`|A function that transitions a promise from `pending` → `fulfilled` with a value|
|`reject`|A function that transitions a promise from `pending` → `rejected` with an error|
|**Deferred**|A pattern where `resolve`/`reject` are extracted from the executor and stored, so the promise can be settled from outside, later, by different code|

The deferred pattern comes up any time you need to wrap a **non-promise async mechanism** (event emitters, callbacks, message passing) into a clean `Promise` that callers can `await`.