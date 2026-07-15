---
tags:
  - algorithms
  - backtracking
  - code
  - problem-solving
---

# Backtracking

**Backtrack Concept**  
Backtracking is a problem-solving strategy used to find solutions by trying out possibilities and undoing them if they lead to dead ends. It’s like exploring a maze: you try a path, and if it doesn’t work, you backtrack to the last decision point and try another path. The process systematically tests all possible options, keeping track of choices and reversing when necessary, until a solution is found or all possibilities are exhausted.

Key aspects:

- **Incremental Approach**: Builds a solution step-by-step, adding one piece at a time.
- **Decision Points**: At each step, you choose from a set of options.
- **Pruning**: If a choice leads to an invalid or dead-end state, you backtrack to the previous step and try a different option.
- **Exhaustive Search**: It explores all feasible combinations, making it suitable for problems with multiple possible solutions.

**Use Cases**

1. **Puzzles and Games**:
    - Solving puzzles like Sudoku, crosswords, or the N-Queens problem, where you place queens on a chessboard so none threaten each other.
    - Finding valid moves in games like chess or generating valid configurations for a Rubik’s Cube solver.
2. **Combinatorial Problems**:
    - Generating all permutations or combinations of a set (e.g., arranging books on a shelf or selecting a team).
    - Solving problems like the knapsack problem, where you choose items to maximize value within a weight limit.
3. **Pathfinding and Mazes**:
    - Navigating a maze by trying different paths and backtracking when hitting a dead end.
    - Finding routes in graphs, like solving a traveling salesman problem for a valid tour.
4. **Constraint Satisfaction Problems**:
    - Scheduling tasks (e.g., assigning time slots for classes without conflicts).
    - Assigning resources, like allocating employees to shifts while meeting availability constraints.
5. **Parsing and Validation**:
    - Validating expressions in compilers or interpreters, where backtracking ensures correct syntax by trying different parsing paths.
    - Matching patterns in text, like in some regular expression engines.

### **Correct: Backtracking does _not always_ explore all paths.**

Backtracking is **designed to avoid unnecessary work** by:

- Abandoning paths as soon as it becomes clear they can’t lead to a solution.
    
- This is called **pruning** — cutting off branches of the search tree early.

### ⚖️ Trade-Off

- **Brute-force** explores all paths no matter what.
    
- **Backtracking** explores **only promising paths** and abandons bad ones early.
    

That's what makes backtracking much more efficient than brute-force — especially when combined with **smart heuristics** or **constraints**.

## See also

- [[Code MOC]]
