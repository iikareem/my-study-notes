---
tags:
  - call-stack
  - code
  - code-craft
  - iteration
  - performance
  - recursion
---

# Iteration vs Recursion & Function Call Overhead

## Iteration, Abstracted

```
LOOP_START:
   do work
   update state
   if condition still true: jump back to LOOP_START
   else: continue past loop
```

- There is **one** execution context the whole time.
- "Progress" is tracked by **overwriting variables in place** (a counter, an accumulator).
- The CPU never leaves its current function — it just moves the instruction pointer backward to repeat instructions.
- Nothing is stacked. There is no "waiting" — each step is fully finished before the next begins.

**Mental model:** A person walking in a circle, updating a tally on a notepad each lap. One person, one notepad, just walking the loop again and again.

---

## Recursion, Abstracted

```
FUNCTION f(state):
   if base case: return known answer
   else: return combine(state, f(next_state))
```

- Each call creates a **new, independent execution context** (its own copy of local variables).
- The new context is **stacked on top of** the one that called it — the caller is paused, not finished, because it's waiting on a result it needs to _combine_ with.
- Nothing actually resolves until the **base case** is hit at the bottom — then results flow back up, each paused caller resuming and combining as the stack unwinds.
- This only works because of the **call stack**: a structure that remembers, in order, every paused caller and where to return to.

**Mental model:** A line of people, each asking the person behind them for an answer before they can finish their own sentence. Person 5 asks person 4, who asks person 3, who asks person 2, who asks person 1. Person 1 (base case) finally answers directly. Then person 2 finishes their sentence using that answer, then person 3, and so on back up the line. _Everyone_ is still standing there, mid-sentence, until the unwinding completes.

---

## The Core Difference, Abstracted

|Concept|Iteration|Recursion|
|---|---|---|
|**What represents "progress"**|Values in variables, overwritten in place|The _call stack itself_ — its depth and order|
|**How many execution contexts exist at once**|1|n (one per unresolved call)|
|**When is a step "done"**|Immediately, before moving to the next|Not until everything _below_ it (deeper calls) finishes first|
|**Direction of work**|Forward only|Forward (calling down) then backward (returning up)|
|**What "remembers where you are"**|Nothing extra needed — it's just a variable|The stack frame + saved return address (see sections below)|

---

## The Deepest Abstraction

Both are really doing the same underlying thing: **maintaining state across repeated work**. The difference is _where that state lives_:

- Iteration keeps state in **a fixed, reused slot** (variables that get overwritten).
- Recursion keeps state in **a growing chain of paused contexts**, where each context's "memory" of where it was is implicit in _being on the stack at all_.

This is why any recursive function _can_ be rewritten as a loop with an explicit stack (you manually do what the call stack was doing for free) — and why tail-recursive functions (where there's nothing left to "combine" after the recursive call returns) can be automatically converted into a loop by the compiler: there's no real need to keep the caller paused if it has nothing left to do.

---

### What a function call actually does (the overhead)

When you call `dfs(neighbor, visited)` recursively, the CPU has to:

1. **Save registers** — the current state of CPU registers needs to be preserved so the parent call can resume correctly later
2. **Push a return address** — where to jump back to when this call finishes
3. **Push parameters** (`neighbor`, `visited`) onto the frame
4. **Allocate space for local variables** in the new frame
5. **Jump** to the function's code
6. ...later, on return: **pop the frame**, **restore registers**, **jump back** to the return address

Function calls are simply heavier instructions than array operations — more CPU cycles, more memory touched (registers, frame metadata), even though both scale at O(n) overall.

**Concretely:** in languages like Python, C, or Java, you'll typically see iterative DFS run **noticeably faster** than naive recursive DFS on large inputs — not because of _less memory_, but because of _less per-step work_. The gain shows up as a constant-factor speedup, not a change in Big-O complexity.

One caveat: the difference is often small enough that for moderate-size inputs, recursive DFS is preferred anyway for its much simpler, more readable code — you'd only switch to iterative if you're hitting actual performance bottlenecks or risking stack overflow on deep graphs.

## See also

- [[Tail Recursion]]
- [[Code MOC]]
