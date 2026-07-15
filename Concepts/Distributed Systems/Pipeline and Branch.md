---
tags:
  - distributed-systems
---

## What is a Branch?

A branch is an instruction that can alter the normal sequential flow of program execution. Instead of executing the next instruction in memory, a branch can jump to a different location (conditional branch like `if`, `for`, `while`) or unconditionally jump (like `goto` or function calls).

```javascript
// Simple example
if (x > 5) {
  doSomething();  // Branch target (taken)
} else {
  doSomethingElse();  // Alternative path (not taken)
}
```

At the CPU level, this becomes a conditional branch instruction. The problem: **the CPU doesn't know which path to take until it evaluates the condition**, but by then it's already started executing future instructions.

## The Pipelining Problem

Modern CPUs use instruction pipelining — they fetch, decode, and execute instructions in parallel stages:

```
Cycle:    1  2  3  4  5  6  7
Instr A: [F][D][E][M][W]
Instr B:    [F][D][E][M][W]
Instr C:       [F][D][E][M][W]
(F=Fetch, D=Decode, E=Execute, M=Memory, W=Write)
```

With a branch, the CPU can't know which instruction comes next until the branch resolves (in the Execute stage, or even later). **If the CPU just stalls and waits, the pipeline empties and performance tanks.**

## Branch Prediction to the Rescue

Instead of stalling, the CPU _guesses_ which direction the branch will take and speculatively executes down that path. If the guess is right, great—no wasted cycles. If wrong, the CPU throws away the speculative work and rewinds to execute the correct path.

### How Branch Prediction Works

**1. Pattern-Based Predictors**

Early CPUs used simple heuristics:

- **Forward branches default to "not taken"** (backward branches default to "taken")
- **Always-taken / Always-not-taken** — static prediction based on instruction bits

**2. Dynamic Predictors (Modern CPUs)**

Modern CPUs use sophisticated predictors that learn from history:

**Branch History Table (BHT) / 1-bit predictor:**

- For each branch address, store 1 bit: taken (T) or not taken (N)
- If the branch was taken last time, predict it will be taken this time
- Problem: poor at alternating patterns (T-N-T-N)

**2-bit Saturating Counter:**

```
State:  Predict   On Taken   On Not-Taken
00      Not-Taken    01          00
01      Not-Taken    11          00
10      Taken        11          00
11      Taken        11          10
```

Needs two mispredictions to flip the prediction—more stable.

**Pattern History Table (PHT):**

- Track the _history_ of recent branch outcomes (e.g., last 4 branches)
- Use that as an index into a table of 2-bit counters
- Captures correlation: "If the last 3 branches went (T, T, N), then this branch typically goes T"

**Gshare / Global History Predictor:**

- Combines global branch history (last N taken/not-taken decisions across all branches) with the branch address
- XOR them together to form an index into a table of counters
- Captures global patterns across different branches

**Tournament Predictor (Modern CPUs like Intel Skylake, AMD Zen):**

- Combine multiple predictors (e.g., local, global, bimodal)
- Use a _meta-predictor_ to decide which one to trust
- If global predictor is accurate for this branch, use it; if local is better, switch to that

### Example: 2-bit Counter in Action

```javascript
for (let i = 0; i < 100; i++) {
  if (data[i] > threshold) {   // Branch A
    process(data[i]);
  }
}
```

Assuming `data` is mostly random:

- Initial prediction state: 01 (weakly "not taken")
- Cycle 1: data[0] > threshold? No. Correct! Stay in 01.
- Cycle 2: data[1] > threshold? Yes. Misprediction! Go to 11.
- Cycle 3+: Mostly yes, so stay in 11 (predict "taken")
- ...but if data[50] isn't > threshold: misprediction! Go to 10.

The predictor learns and adapts over a few iterations, then stabilizes.

## Branch Misprediction Penalty

This is where performance really matters.

**The cost of a misprediction:**

When the branch resolves and the prediction was wrong:

1. All speculatively executed instructions are flushed (wasted work)
2. The pipeline must be refilled with the correct instructions
3. Modern CPUs: **10–20+ cycles of stall** (Skylake: ~15 cycles in typical cases)

Compare this to a correct prediction: **0 cycles of penalty** (branch resolved, next correct instruction already in flight).

### Real Performance Impact

```javascript
// Case 1: Predictable branch (best case)
let sum = 0;
for (let i = 0; i < 1000; i++) {
  sum += i;  // No branch, or branch always taken/not-taken
}
// ~1000 cycles

// Case 2: Unpredictable branch (worst case)
let sum = 0;
for (let i = 0; i < 1000; i++) {
  if (Math.random() > 0.5) {  // Random branch
    sum += i;
  }
}
// ~1000 + (1000 * 0.5 * 15) = ~8500 cycles!
// 8.5x slower due to random mispredictions
```

## Branch Misprediction Patterns

**When predictions fail:**

1. **True randomness:** `if (Math.random() > 0.5)` — no pattern to learn
    
2. **Long periods of one outcome, then flip:**
    
    ```
    T T T T T T T T T N N N N N N
    ```
    
    The predictor trains to "always T", then when it finally goes N, boom—misprediction. Then it retrains to "always N", rinse repeat.
    
3. **Data-dependent branches with unpredictable data:**
    
    ```javascript
    for (let x of data) {
      if (x > someValue) {  // Depends on data pattern
        process(x);
      }
    }
    ```
    
    If `data` is effectively random, predictions are random.
    
4. **Mixed patterns:**
    
    ```
    T N T N T N T N...  // Alternating
    ```
    
    A 1-bit predictor fails every time. Even a 2-bit can struggle.
    

## Strategies to Avoid Mispredictions

**1. Reduce branches in hot code paths**

```javascript
// Bad: Branch inside tight loop
for (let i = 0; i < 1000000; i++) {
  if (arr[i] < 0) {
    result = 0;
  } else {
    result = arr[i];
  }
}

// Better: Branch prediction (CMOV on x86)
for (let i = 0; i < 1000000; i++) {
  result = arr[i] < 0 ? 0 : arr[i];  // Might compile to conditional move, not branch
}
```

**2. Sort data to make branches more predictable**

```javascript
// Bad: Random if data is random
let sum = 0;
for (let x of unsortedData) {
  if (x >= 128) sum += x;
}

// Better: Sorted data = predictable branch
sortedData.sort((a, b) => a - b);
let sum = 0;
for (let x of sortedData) {
  if (x >= 128) sum += x;  // Branch is T T T ... N N N ... predictable!
}
```

The sorted version can be 6–10x faster due to branch prediction!

**3. Avoid deep pipelines or speculative execution in unpredictable branches**

```javascript
// Better: Move branch out of critical loop
let count = 0;
for (let x of data) {
  if (someCondition) count++;
}
// Easier to optimize, branch is outside tight loop
```

**4. Use data-level parallelism instead of branch-level control flow**

```javascript
// Avoid branch
let result = values.map(v => v > threshold ? v * 2 : v);
// SIMD or vectorization may handle this without branches
```

## What Else Do You Want to Explore?

Are you interested in:

- **How branch prediction relates to Spectre/Meltdown?** (Speculative execution is the culprit)
- **Measuring mispredictions on real hardware?** (Using performance counters: `perf stat` on Linux)
- **Deep dive into specific CPU predictors?** (Gshare, TAGE, etc.)
- **How to profile and detect branch mispredictions in your code?**
- **Return stack buffers** (special case for function returns)?
- **Indirect branches and indirect predictor tables?**

Let me know where you'd like to go deeper!