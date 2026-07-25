---
tags:
  - os
  - cpu
  - mechanical-sympathy
---

# CPU Pipeline & Branch Prediction

## The Problem: Doing Things One at a Time is Slow

A CPU executes instructions. Every single instruction goes through multiple stages:

1. **Fetch** — grab the instruction from memory
2. **Decode** — figure out what it means
3. **Execute** — do the actual work
4. **Write-back** — store the result

If the CPU handled one instruction completely before touching the next, it would look like this:

```
Cycle:        1      2      3      4      5      6      7      8
Instruction 1: Fetch → Decode → Execute → Write
Instruction 2:                                   Fetch → Decode → Execute → Write
```

While instruction 1 is being decoded, the fetch unit sits completely idle. While it's executing, both fetch and decode sit idle. **You have expensive hardware doing nothing most of the time.**

At 3GHz — 3 billion cycles per second — this waste compounds into a massive performance problem.

---

## The Solution: Pipelining

The fix is to overlap the stages. The moment instruction 1 moves from Fetch to Decode, instruction 2 jumps into Fetch. Nothing sits idle.

```
Cycle:        1       2       3       4       5
Instruction 1: Fetch → Decode → Execute → Write
Instruction 2:         Fetch  → Decode  → Execute → Write
Instruction 3:                  Fetch   → Decode  → Execute → Write
Instruction 4:                           Fetch    → Decode  → Execute → Write
```

Read it **column by column** — that's one clock cycle. At cycle 4, all four stages are busy simultaneously, each working on a different instruction.

This is the **conveyor belt model**. Each stage hands its result to the next, then immediately picks up new work. The dependency flows forward in sequence — but the stages run in parallel in time.

> The CPU isn't making one instruction faster.  
> It's completing far more instructions per second.

---

## The New Problem: Branches

Pipelining works perfectly when instructions flow one after another in a straight line. But code is never a straight line.

```js
if (user.isLoggedIn) {
    // path A — hundreds of instructions
} else {
    // path B — hundreds of instructions
}
```

This is a **branch** — a point where execution can go one of two directions depending on a condition.

Here's the problem: by the time the Execute stage resolves `user.isLoggedIn`, the Fetch stage has already moved **3 or 4 instructions ahead** — because that's what pipelining does, it always runs ahead.

**Which instructions did it fetch?** It had to pick a direction before knowing the answer.

The CPU has two options:

|Option|What happens|
|---|---|
|**Stall**|Stop the pipeline and wait for the branch to resolve. Safe, but destroys all the parallelism you just built.|
|**Guess**|Pick a direction, keep the pipeline full, throw away the work if wrong.|

Modern CPUs always guess. That guessing mechanism is called **branch prediction**.

---

## What Happens When the Guess is Wrong

If the CPU predicted the wrong path:

1. It **flushes the pipeline** — throws away everything fetched and decoded on the wrong path
2. Jumps to the correct path and starts fresh
3. Loses roughly **10–20 clock cycles** of work — the **misprediction penalty**

At 3GHz inside a tight loop, this adds up fast.

---

## How the Predictor Guesses

It doesn't guess randomly — it uses history.

The CPU keeps a record of how each branch behaved in the past: _"the last 8 times we hit this branch, it went left."_ It uses that pattern to predict the next outcome.

This is why **sorted data runs faster than unsorted data** in a loop with a branch condition. Sorted data creates a predictable pattern — false for a while, then true for a while. The predictor learns it. Unsorted data is random — the predictor keeps getting it wrong.

Modern predictors reach **95–99% accuracy** on typical code. The remaining 1–5% is where performance dies.

---

## Summary

```
Sequential execution
└── Stages run one at a time → hardware sits idle → slow

Pipelining
└── Stages overlap across instructions → hardware always busy → fast
    └── But now Fetch runs ahead of Execute
        └── Branches create uncertainty about what to fetch next
            └── Branch Prediction: guess the path, keep the pipeline full
                └── Wrong guess = flush pipeline = misprediction penalty
```

---

## Related

- [[Mechanical Sympathy]]
- [[Memory Hierarchy Closest to CPU]]

← [[Concepts MOC]]
