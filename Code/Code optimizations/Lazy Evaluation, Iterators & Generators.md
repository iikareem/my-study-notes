---
tags:
  - code
  - code-craft
  - generators
  - iterators
  - lazy-evaluation
  - performance
---

# Lazy Evaluation, Iterators & Generators

> **The core problem**: By default, code decides how much work to do. The caller has no say.

---

## 1. The Problem: Eager Evaluation

When you chain array methods, JavaScript is **eager** — it finishes each step completely before moving to the next.

```js
data.filter(...).map(...).slice(0, 5)
```

Under the hood this is three separate passes over the data, each producing a brand new array in memory. The chaining syntax hides this from you, but those temporary arrays are real.

This creates two independent costs:

**Cost 1 — Intermediate array waste (memory)** Every `.filter()` and `.map()` allocates a new array. If you chain three methods on 100k records, three separate 100k-item arrays exist in heap simultaneously — even if you only need the final result. They're created, used as stepping stones, then thrown away. The GC cleans them up eventually, but the spike already happened.

This cost hits you **even if you consume all results**. It's purely about intermediate allocation.

**Cost 2 — Useless processing (CPU)** If you filter a million items, map all survivors, then take only the first 5 — the first two operations had no idea you only needed 5. They processed everything eagerly before handing off.

This cost only hits you **when you stop early**. If you consume everything, no CPU was wasted.

These are two distinct problems with different triggers. They often appear together, which is why they get conflated.

---

## 2. The Root Cause

Both problems come from the same place: **the producer controls the pace**.

A function runs to completion and hands you everything at once. You, the caller, get no say in when to stop. The only tool you have is to not look at part of the result — but the work was already done.

What you actually want is the ability to say:

> _"Give me one item. Now give me another. Okay, I have enough — stop."_

That's the Iterator pattern.

---

## 3. The Iterator Pattern

The Iterator pattern's idea:

> Decouple _how data is stored or produced_ from _how it is consumed_, by exposing a uniform one-item-at-a-time interface.

The contract is simple: an iterator is any object with a `.next()` method that returns `{ value, done }`.

- `value` — the current item
- `done` — `true` when there's nothing left

The caller drives the pace. The producer does exactly one unit of work per `.next()` call, then waits.

You already used this pattern manually in BFS/DFS — managing a stack or queue, deciding when traversal ends, exposing one node at a time. You were hand-rolling the Iterator pattern. Generators just give you this for free.

---

## 4. Generators: The Iterator Pattern as a Language Feature

A generator is a special function that can **pause itself mid-execution**, hand you a value, and wait until you ask for the next one. The caller holds the remote control.

Two pieces of syntax:

- `function*` — marks the function as a generator
- `yield` — the pause button; hands a value to the caller and suspends execution

### The three states of a generator

```
Not started → Running → Suspended (at yield) → Completed
                  ↑______________|
                    .next() resumes it
```

In the suspended state the generator is **alive** — local variables intact, position remembered — but consuming zero CPU. It only runs when you call `.next()`.

### How `for...of` fits in

`for...of` doesn't create a generator. It's a consumer — it calls `.next()` in a loop and stops when `done: true`. The generator already exists; `for...of` is just clean syntax for driving it.

```
generator function  →  produces an iterator
for...of            →  consumes any iterator
```

Because `for...of` works with _any_ iterator — not just generators — it also works on arrays, strings, Maps, Sets, and any object you give a `.next()` method. Generators are just the most convenient way to produce one.

---

## 5. How This Solves Both Problems

When an item flows through a generator pipeline, it goes **all the way through every step before the next item is touched**. Nothing accumulates.

```
Eager:     filter ALL → map ALL → take 5  (3 full passes, 3 arrays)
Generator: item flows filter → map → taken, then next item, then next...
```

**Memory**: no intermediate arrays are ever built. At any moment, only the current item exists between steps. Memory footprint is flat regardless of dataset size.

**CPU**: the moment you `break` out of a `for...of`, the generator freezes. Items after that point are never processed — not filtered, not mapped, nothing.

---

## 6. What It Doesn't Fix

If you consume **all** results without breaking, you still get the memory benefit but not the CPU one. Every item still gets processed — the generator just avoided the intermediate array allocations.

|Situation|Memory saved|CPU saved|
|---|---|---|
|Consume all results|✅ Yes|❌ No|
|Break early|✅ Yes|✅ Yes|

The memory win is unconditional. The CPU win depends on whether the caller stops early.

---

## 7. Async Generators: The Same Idea Over I/O

In backend work, the data source is often not an in-memory array but a database, a file, or an external API. Async generators apply the exact same principle to asynchronous work.

```
sync generator:   yield one computed value, pause
async generator:  yield one awaited value, pause
```

Real scenarios where this matters:

**Streaming database results** — instead of loading a million rows into an array, a cursor yields one row at a time. Memory stays flat no matter how large the table is.

**Paginating an API** — instead of fetching all pages up front, yield each item from each page. If you find what you need on page 2, pages 3 through N are never fetched.

In both cases the caller retains full control — it can stop at any point and no unnecessary work happens after that.

---

## 8. The Conceptual Hierarchy

```
Iterator Pattern              ← abstract concept: uniform .next() interface
        ↓
JavaScript Iterator Protocol  ← language spec: .next() → { value, done }
        ↓
Generators                    ← language feature that implements the protocol
        ↓
Async Generators              ← same idea, each yield can await async work
```

When you write `function*` and `yield`, JavaScript builds the iterator object for you — the `.next()` method, the `{ value, done }`shape, the suspended execution state. You're not inventing anything new; you're getting the Iterator pattern for free.

---

## 9. Key Takeaways

**Eager vs lazy** is about who controls the pace — the producer or the consumer. Generators hand that control to the caller.

**Two separate costs** come from eager evaluation: intermediate array allocations (always) and useless processing (only when stopping early). Know which one you're solving.

**Generators are the Iterator pattern** built into the language. The same idea you used in BFS/DFS, without the boilerplate.

**`yield` is a pause button** — the function stays alive with all its state intact until you resume it.

**`break` is meaningful** on a generator — it stops all further work immediately.

**Async generators** extend the same model to I/O, giving you flat memory and early-stop control over databases, files, and external APIs.

---

## 10. One-Sentence Mental Model

> A generator is a function that does **exactly as much work as you ask for**, remembers where it stopped, and waits until you ask for more.

## See also

- [[Memoization via Closures]]
- [[Code MOC]]
