---
type: moc
aliases:
  - Code
  - Code Craft
  - Code Optimizations
tags:
  - code
  - moc
  - code-craft
  - algorithms
---

# Code MOC

Practical code craft, patterns, and algorithm techniques. Start here, then open any note below.

## How to use

1. Pick a **practice tip** under Code craft when reviewing how you write day-to-day code.
2. Use **patterns** when designing features (state machines, rules, resource lifecycle).
3. Use **performance & abstraction** when reasoning about recursion, laziness, or call cost.
4. Use **problem-solving** for interview / DSA drills.

## Code craft (readability & structure)

| Note | Focus |
|------|--------|
| [[Enforce the "Single Level of Abstraction" Principle (SLAP)]] | One abstraction level per function |
| [[Flatten Your Code Using Guard Clauses]] | Early returns; avoid arrow-shaped nesting |
| [[Tell, Don't Ask (Encapsulation on Steroids)]] | Tell objects what to do; don’t pull state and decide outside |
| [[Treat Data as Immutable by Default]] | Prefer copies / new values over silent mutation |
| [[Stop Writing "What" Comments—Only Write "Why" Comments]] | Comments explain intent, not the obvious |
| [[Treat Third-Party Dependencies as Untrusted]] | Adapter boundary around vendors |

## Patterns

| Note | Focus |
|------|--------|
| [[The Specification Pattern]] | Composable business rules |
| [[The State Pattern]] | Behavior that changes with state |
| [[Context Manager Pattern]] | Setup/teardown of resources (files, locks, DB) |
| [[Implement an In-Memory Object Pool]] | Reuse expensive objects |

## Performance & evaluation

| Note | Focus |
|------|--------|
| [[Memoization via Closures]] | Cache pure results in a closure |
| [[Lazy Evaluation, Iterators & Generators]] | Defer work until needed |
| [[Function Call Overhead — What Actually Happens Behind the Scenes]] | Iteration vs recursion & call cost |
| [[Tail Recursion]] | Tail calls, TCO, and language support |

## Problem solving (algorithms)

| Note | Focus |
|------|--------|
| [[Two Pointers]] | Opposite ends, fast/slow, sliding window |
| [[Binary Search]] | Search on sorted / monotonic space |
| [[Prefix Sum and Difference Array]] | Range sums & range updates |
| [[Backtrack]] | Explore + prune search trees |

## Reference

- [[Go Project Folder Structure]] — Go layout notes (with source link)

---

*This folder is self-contained — safe to publish on its own.*
