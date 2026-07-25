---
tags:
  - nodejs
  - v8
  - javascript
  - performance
---

# V8 Hidden Classes — Complete Notes

## 1. Start from the basics: how does data live in memory?

When you write this in JavaScript:

```js
const user = { name: "Alice", age: 25 };
```

This object has to live somewhere in RAM. The machine needs to store those values and be able to find them later. The question is — **how does it find them?**

---

## 2. How static languages solve this

In a static language like C++, the compiler knows the exact layout of every object **at compile time**:

```cpp
struct User {
  string name;  // always at offset 0
  int age;      // always at offset 8
};
```

So accessing `user.age` compiles down to:

```
read memory at [user_pointer + 8]
```

One instruction. No searching. No hashing. The CPU just does arithmetic on a pointer and reads the value directly.

---

## 3. The problem with JavaScript

JavaScript is **dynamically typed**. You can add properties in any order, at any time:

```js
const user = {};
user.name = "Alice";
user.age = 25;
```

The naive solution is a **hashmap** — store key-value pairs, hash the string `"name"` to find where it lives. This works but it's slow. You're doing string hashing and bucket lookups on every single property access.

V8 doesn't want to do that. It wants property access to be a raw memory read, just like C++.

---

## 4. V8's solution: Hidden Classes

V8 introduces something called a **hidden class** — an internal blueprint that records:

- What properties the object has
- What **order** they were added in
- What **memory offset** each property lives at

It's called "hidden" because it's invisible to you as a developer. V8 manages it automatically.

When shapes are consistent, V8 can compile property access down to a hardcoded offset:

```
Object in memory: [ hidden_class_pointer | "Alice" | 25 ]
                    offset 0               offset 8   offset 16

user.name  →  read [user + 8]    ← one instruction
user.age   →  read [user + 16]   ← one instruction
```

---

## 5. How hidden classes are built: the transition chain

V8 builds hidden classes **incrementally**, one property at a time:

```js
const user = {};        // HC0: {}
user.name = "Alice";    // HC1: { name @ offset 8 }
user.age = 25;          // HC2: { name @ offset 8, age @ offset 16 }
```

Each step produces a transition:

```
HC0 --[ add name ]--> HC1 --[ add age ]--> HC2
```

This chain is **cached**. If you create another object and add properties in the same order, V8 walks the same chain and reuses HC1 and HC2. No new hidden classes created. Maximum efficiency.

---

## 6. What breaks it: diverging shapes

```js
const u1 = {};
u1.name = "Alice";   // HC0 → HC1 { name }
u1.age = 25;         // HC1 → HC2 { name @ 8, age @ 16 }

const u2 = {};
u2.age = 25;         // HC0 → HC3 { age }         ← different branch
u2.name = "Alice";   // HC3 → HC4 { age @ 8, name @ 16 }  ← different offsets!
```

Now you have two objects that look identical from JavaScript's perspective, but internally they have **different hidden classes with different memory layouts**:

```
u1: [ HC2 | "Alice" | 25 ]    name=offset 8,  age=offset 16
u2: [ HC4 | 25 | "Alice" ]    age=offset 8,   name=offset 16
```

---

## 7. How V8 and the CPU work together

This is important to understand correctly. V8 and the CPU operate in **two completely separate phases**. They never talk to each other:

```
Phase 1: V8 runs first
         ↓
         V8 watches your code run a few times (profiling)
         ↓
         JIT compiler produces machine code instructions with hardcoded offsets
         ↓
Phase 2: CPU runs after
         ↓
         CPU executes those instructions blindly, no thinking involved
```

Think of it like a chef and a waiter:

- **V8 (chef)** preps everything ahead of time: _"name is always at offset 8, so I'll write the instruction: go to address + 8"_
- **CPU (waiter)** just follows the script blindly: _"go to address + 8, grab the value"_

The waiter never asks the chef anything. The chef wrote the instructions ahead of time.

---

## 8. Monomorphic, Polymorphic, Megamorphic

When a function receives objects of different shapes, V8 has to produce increasingly complex instructions:

### Monomorphic — 1 shape (fastest)

```js
// Every user always has the same hidden class
function processUser(user) {
  return user.age * 2;
}
```

V8 compiles this to:

```
read [user + 16]    ← one hardcoded instruction, nothing else
```

No checks. No comparisons. One memory read and done.

---

### Polymorphic — 2 to 4 shapes (okay)

Now imagine your function receives two kinds of user objects with different hidden classes. V8 can't hardcode one offset anymore — `age` might be at +8 or at +16 depending on which shape arrived.

So V8 builds a **small inline cache (lookup table)** — a short list of known shapes and their corresponding offsets, compiled ahead of time:

```
┌─────────────────────────────────────────┐
│  Shape  │  offset of "age"              │
├─────────────────────────────────────────┤
│  HC2    │  +16                          │
│  HC4    │  +8                           │
└─────────────────────────────────────────┘
```

The compiled code V8 produces looks roughly like:

```
check: is this object HC2?  → yes → read [user + 16] → done
check: is this object HC4?  → yes → read [user + 8]  → done
```

**Why is this still okay?** Because the table is small and fixed. V8 wrote it out ahead of time — the CPU just runs through 2–4 simple comparisons and jumps to the right read. That's a handful of instructions total, and they're all in the CPU's cache because they're so small. Still very fast.

**Why does it cost more than monomorphic?** Those extra comparisons have a cost. More instructions means more work per property access. And if the table has 4 entries, you might have to check all 4 before finding a match. It adds up across thousands of calls.

---

### Megamorphic — 5+ shapes (slow)

At 5+ shapes, V8 says: _"I can't keep track of all these shapes in a small table anymore. It's not worth trying."_

It throws away the optimized path entirely and falls back to a **generic hashmap lookup in software**. This is what it looks like conceptually:

```
Step 1: take the string "age"
Step 2: run it through a hash function → produces a number (e.g. 4827)
Step 3: go to bucket 4827 in the property table
Step 4: check if the key there is actually "age" (hash collisions exist)
Step 5: if yes → read the offset stored there
Step 6: go to that offset and read the value
```

**Why is this so costly?**

- It's **many instructions** instead of one. Every property access is now a small algorithm.
- The property table lives in memory. Reading it means **multiple RAM accesses** — going from the object, to the table, to the bucket, then finally to the value. Each jump risks a **CPU cache miss**, which stalls the CPU while it waits for RAM (RAM is ~100x slower than CPU cache).
- This happens **on every single property access**, not just once. If you read `user.name`, `user.age`, and `user.premium` in a loop of 10,000 users, that's 30,000 hashmap lookups instead of 30,000 direct reads.
- This state is **permanent** — once V8 marks a function as megamorphic, it never goes back.

```
Monomorphic  →  1 read instruction                        ← CPU stays in cache
Polymorphic  →  2–4 comparisons + 1 read                  ← small, still cached
Megamorphic  →  hash → table lookup → multiple RAM reads   ← cache misses, slow
```

The jump from polymorphic to megamorphic is not gradual — it's a cliff.

---

## 9. This happens at runtime with real data too

This isn't just a concern for hand-written objects. It applies to **every object your app ever creates** — including data from APIs, databases, user input, anything:

```js
// Data from your API
const user = await fetch("/api/user").then(r => r.json());

// V8 still assigns a hidden class to this object
// Still tracks its shape
// Still goes megamorphic if shapes are inconsistent
```

A real world scenario that hurts you — your API sometimes returns objects in different orders:

```js
{ name: "Alice", age: 25 }    // HC2
{ age: 30, name: "Bob" }      // HC4  ← different property order
{ name: "Carol" }             // HC5  ← missing property entirely
```

Your function goes megamorphic and stays there forever.

**The fix** — normalize data into a consistent shape as early as possible:

```js
function normalizeUser(raw) {
  return {
    name: raw.name ?? null,   // always present, always first
    age:  raw.age  ?? null,   // always present, always second
  };
}

const user = normalizeUser(await fetch("/api/user").then(r => r.json()));
```

Now every user object has the same hidden class, forever.

---

## 10. The design pattern that solves this: Factory Pattern

The **Factory Pattern** is the design pattern that directly addresses this problem. A factory is a function whose only job is to create objects — and because all creation goes through **one place**, the shape is always consistent:

```js
// The factory — one place, one shape, always
function createUser(name, age) {
  return {
    name: name ?? null,
    age:  age  ?? null,
  };
}

const u1 = createUser("Alice", 25);
const u2 = createUser("Bob", 30);
const u3 = createUser("Carol", null);  // missing age? still same shape, still HC2
```

Every object that comes out of this factory has the same hidden class. V8 sees the same shape every time → stays monomorphic forever.

Without the pattern:

```js
// Different developers, different files, different shapes
const u1 = { name: "Alice", age: 25 };
const u2 = { age: 30, name: "Bob" };   // ← flipped order → different hidden class
const u3 = { name: "Carol" };          // ← missing property → different hidden class
```

Three hidden classes. Megamorphic. Slow.

### Other patterns that help

|Pattern|How it helps|
|---|---|
|**Factory**|One creation point = guaranteed consistent shape|
|**Builder**|Constructs object step by step but always in the same order|
|**Schema / DTO**|Enforces a fixed structure at the boundary of your system when normalizing API data|

---

## 11. The Delete Disaster

There is one operation in JavaScript that completely destroys hidden class optimization — the `delete` keyword.

### What delete does to an object

```js
const user = { name: "Alice", age: 28, premium: false };
// V8 assigned HC2: { name @ 8, age @ 16, premium @ 24 }

delete user.premium;
// HC2 is now wrong — the object no longer matches its blueprint
```

V8 built HC2 assuming `premium` always exists at offset 24. Now it's gone. V8 can't just patch the hidden class — other objects might share it. So instead of fixing the blueprint, V8 does something drastic:

**It converts the object to dictionary mode.**

### What is dictionary mode?

Dictionary mode means V8 completely abandons the hidden class system for this object and converts it to a plain hashmap — exactly the slow structure V8 was trying to avoid in the first place.

```
Before delete (hidden class mode):
[ HC2_pointer | "Alice" | 28 | false ]
  offset 0      offset 8  offset 16  offset 24
  → property access = one memory read

After delete (dictionary mode):
{ "name" → "Alice", "age" → 28 }   ← actual hashmap in memory
  → property access = full hash lookup every time
```

Every property access on this object is now a hashmap lookup forever. Not just for `premium` — for `name`, `age`, everything. The entire object is downgraded.

### Why this is catastrophic

```js
function processUser(user) {
  return user.name + " is " + user.age;  // innocent-looking
}
```

If `user` was created with `delete`, every access in this function — `user.name`, `user.age` — is a hashmap lookup. Call this function 10,000 times and you've done 20,000 unnecessary hashmap lookups.

And unlike going megamorphic, dictionary mode doesn't even benefit from V8's inline cache system. It's completely outside the optimization pipeline.

### The fix: never delete, use null instead

```js
// ❌ DON'T — converts object to dictionary mode permanently
delete user.premium;

// ✅ DO — keeps the hidden class intact, slot still exists
user.premium = null;

// ✅ ALSO FINE — same, slot exists, value is just undefined
user.premium = undefined;
```

Setting a property to `null` or `undefined` keeps the hidden class intact. The slot at offset 24 still exists — it just holds a null value. V8's blueprint is still valid. No dictionary mode, no performance cliff.

### The mental model

Think of the hidden class as a **fixed form with printed fields**. The form has a box labeled "premium." You're allowed to write anything in that box — `false`, `null`, `undefined`, `true`. The form stays valid.

But `delete` is like **ripping that field off the form entirely**. Now the form doesn't match the template anymore. V8 can't use the template, so it throws the whole form away and switches to a freeform notepad — which is the hashmap.

### Summary of the rule

|Operation|Effect|
|---|---|
|`user.premium = null`|✅ Keeps hidden class, slot holds null|
|`user.premium = undefined`|✅ Keeps hidden class, slot holds undefined|
|`delete user.premium`|❌ Converts entire object to dictionary mode|

> One `delete` anywhere in your codebase can silently slow down an entire object's property access for the rest of the application's lifetime.

---

## 12. Rules to follow

- Always declare all properties upfront, in the same order, every time
- Use `null` for missing values instead of skipping the property entirely
- **Never use `delete`** — set properties to `null` or `undefined` instead
- Normalize external data (API responses, DB results) at the boundary before it flows into your app
- Centralize object creation behind a factory function

---

## Summary

|Concept|What it means|
|---|---|
|**Hidden class**|V8's internal blueprint: which properties exist and at what memory offsets|
|**Transition chain**|The sequence of hidden classes built as you add properties one by one|
|**Monomorphic**|All objects at a call site share one hidden class → V8 hardcodes the offset → fastest|
|**Polymorphic**|2–4 hidden classes → V8 compiles a small lookup table → still okay|
|**Megamorphic**|5+ hidden classes → V8 gives up → generic hashmap lookup → slow|
|**The rule**|Always add properties in the same order, preferably all at once in a factory|
|**Factory pattern**|Centralizes object creation so shape is always consistent|
|**Dictionary mode**|What V8 converts an object to after `delete` — full hashmap, all optimizations gone|
|**The delete rule**|Never `delete` a property — use `null` or `undefined` to keep the hidden class intact|

> V8 is trying to make a dynamic language behave like a statically typed one — where property offsets are known ahead of time. Your job as a developer is to make that easy for it by keeping your object shapes consistent.

---

## Related

- [[Node.js Event Loop & Internals]]
- [[Mechanical Sympathy]]
- Blog: https://kareemdev.hashnode.dev

← [[Concepts MOC]]
