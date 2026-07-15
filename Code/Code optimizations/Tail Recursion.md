---
tags:
  - code
  - code-craft
  - performance
  - recursion
  - tail-recursion
  - tco
---

# Tail Recursion

**Tail recursion** is a specific kind of recursion where the **recursive call is the last operation** in the function. In other words, there is **nothing left to do after the recursive call returns**—no computations, no additions, no function calls.

### Example of Tail Recursion (pseudocode)

```
function factorial(n, acc = 1):
    if n == 0:
        return acc
    else:
        return factorial(n - 1, acc * n)
```

Here, `factorial(n - 1, acc * n)` is the last operation — this is tail recursion.

### Non-tail recursive version

```
function factorial(n):
    if n == 0:
        return 1
    else:
        return n * factorial(n - 1)
```

In this case, the multiplication (`n * ...`) happens _after_ the recursive call returns — so it is **not** tail-recursive.
### Why Tail Recursion Matters

- **Optimization**: Some compilers and interpreters can **optimize tail-recursive functions** to avoid growing the call stack. This is known as **tail call optimization (TCO)**.
    
- **Prevents Stack Overflow**: Especially useful in functional languages or when dealing with deep recursion.

### 🔄 So how does TCO work?

When a function is **tail-recursive**, there's no need to **remember previous stack frames**, because the function doesn't need to come back and do more work.

f(3)
 └─ f(2)
     └─ f(1)
         └─ f(0)

With TCO, the compiler or interpreter can **reuse the current stack frame** like this:

f(3) → f(2) → f(1) → f(0)  (all in same frame)

### ✅ So yes — TCO makes tail recursion work like iteration.

It gives you:

- The **readability** of recursion
    
- The **efficiency** of a loop (no stack growth)

| Language      | TCO Support | Notes                                                           |
| ------------- | ----------- | --------------------------------------------------------------- |
| Scheme        | ✅           | Fully supported by design                                       |
| Haskell       | ✅           | Functional language, highly optimized recursion                 |
| Scala         | ✅           | Use `@tailrec` to enforce and check TCO                         |
| JavaScript    | ❌           | ES6 spec allows, but no engines implement                       |
| Python        | ❌           | Recursion is limited; use loops instead                         |
| Java          | ❌           | JVM doesn’t optimize tail calls                                 |
| Rust          | ⚠️ Partial  | Might optimize via LLVM, not guaranteed                         |
| C (GCC/Clang) | ⚠️ Optional | Can be optimized with flags (`-O2`, `-foptimize-sibling-calls`) |

## See also

- [[Function Call Overhead — What Actually Happens Behind the Scenes]]
- [[Code MOC]]
