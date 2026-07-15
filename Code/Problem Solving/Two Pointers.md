---
tags:
  - algorithms
  - code
  - problem-solving
  - sliding-window
  - two-pointers
---

# Two Pointers

The **two pointers technique** is a programming pattern where **you use two indexes** (pointers) to iterate through a data structure — usually an array or string — to solve problems **more efficiently**, often reducing **O(n²)** brute-force time to **O(n)** or **O(n log n)**.

### **When to Use Two Pointers**

Use it when:

- You are dealing with **sorted arrays or strings**.
    
- You need to find a **pair/triplet that satisfies a condition** (e.g., sum, palindrome).
    
- You are required to **modify or compare** elements in-place without using extra space.
    
- You're trying to **optimize** brute force algorithms with nested loops.

	### **Common Two Pointers Patterns**

| Pattern            | Description                                          | Use Cases                                               |
| ------------------ | ---------------------------------------------------- | ------------------------------------------------------- |
| **Opposite Ends**  | Start pointers at the beginning and end, move inward | Palindrome check, 2Sum in sorted array                  |
| **Fast and Slow**  | One pointer moves faster than the other              | Cycle detection in linked lists, removing duplicates    |
| **Sliding Window** | Two pointers define a window that expands/shrinks    | Subarrays, substrings, longest/shortest window problems |

### What Two Pointers Usually Outperform

- **Brute Force / Nested Loops (O(n²))**  
    Many problems that naively require checking all pairs or subarrays use nested loops, resulting in quadratic time complexity.
    
- **Two pointers often reduce these to O(n) or O(n log n)** (if sorting is involved first).

### When You _Can_ Use Two Pointers Without a Sorted Array

In these cases, two pointers manage a **window, relationship, or condition** — not a value comparison that depends on sorted order:

- **Sliding window** — longest substring without repeats, minimum window covering a set
- **Fast / slow** — linked-list cycle detection, middle of a list
- **Partition / in-place rewrite** — move zeros, Dutch national flag, remove duplicates

### Mental checklist

1. Can nested loops shrink to one pass with two indexes?
2. Do the pointers always move in one direction (no reset to the start)?
3. Is the invariant clear (what the window / pair always means)?

## See also

- [[Binary Search]]
- [[Prefix Sum and Difference Array]]
- [[Code MOC]]
