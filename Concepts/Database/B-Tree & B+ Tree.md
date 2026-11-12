# B-Tree & B+ Tree — Complete Notes

---

## What Are They?

Both **B-Tree** and **B+ Tree** are self-balancing tree data structures used to store and retrieve data efficiently — especially in databases and file systems where data lives on disk.

---

## Components of a B-Tree

A B-Tree is built from 4 core components:

### 1. Node

A node is a single "box" in the tree that holds keys and pointers. Every node in a B-Tree holds actual data.

### 2. Keys

The search values inside a node, always sorted in ascending order. Keys tell you which direction to go when searching.

```
  [ 20 | 50 ]

  Left child  → values < 20
  Middle child → values between 20 and 50
  Right child  → values > 50
```

### 3. Data Pointers

Every key is paired with a pointer to the actual record on disk (e.g. a row in a database table). In a B-Tree, every node at every level holds data pointers — you can find data without going all the way to the bottom.

### 4. Child Pointers

A node with `n` keys has exactly `n+1` child pointers — one for each gap between (and around) the keys.

```
  [ 20 | 50 ]   ← 2 keys = 3 child pointers
   /    |    \
  P0   P1    P2
```

---

## Node Types

Both B-Tree and B+ Tree share the same 3 node types. The difference is in what each node **holds**.

### Root Node

- The very **top node** — entry point for every search
- There is always exactly **one** root
- Has at least 1 key

### Internal Nodes

- All nodes in the **middle layers** between root and leaves
- Hold keys to **guide the search** (routing)
- Have child pointers pointing down to the next layer
- Can have **many layers** depending on how much data is stored

### Leaf Nodes

- Always at the **very bottom** of the tree
- Have **no child pointers** — the tree ends here
- All leaf nodes sit at the **same depth** (this is what keeps the tree balanced)

```
            [ Root ]                    ← 1 root node
           /        \
      [ Internal ] [ Internal ]         ← middle layer(s)
       /    \        /    \
   [Leaf] [Leaf] [Leaf] [Leaf]          ← always the bottom
```

> The tree can have many internal layers. Root and internal nodes are both just routing. Leaves are always the bottom.

---

## The n Keys → n+1 Children Rule

A node with `n` keys always has `n+1` children.

Think of keys as **fences** and children as the **gaps between them** — there is always one more gap than fences.

```
        [ 20 | 50 ]       ← 2 keys → 3 children
        /    |    \
       P0   P1    P2

  P0 → values < 20
  P1 → values between 20 and 50
  P2 → values > 50
```

|Keys|Children|
|---|---|
|1|2|
|2|3|
|3|4|
|n|n + 1|

---

## B-Tree vs B+ Tree — Structure

### B-Tree

- **Every node** (root, internal, leaf) holds keys + data
- Data can be found at **any level** — search can stop early
- Leaf nodes are just the bottom nodes with no children — nothing special about them
- Leaf nodes are **not linked** to each other

```
              [ 40 ]               ← Root (holds data)
             /       \
          [ 20 ]    [ 60 ]         ← Internal (holds data)
          /    \    /    \
       [10]  [30] [50]  [70]       ← Leaf (holds data, no children)
        ↓     ↓    ↓     ↓
      data  data  data  data
```

### B+ Tree

- **Only leaf nodes** hold actual data
- Root and internal nodes hold keys only — they are purely for routing
- Leaf nodes are **unique and special** — all data lives here
- Leaf nodes are **linked together** as a chain (doubly linked list)

```
              [ 40 ]               ← Root (keys only, no data)
             /       \
          [ 20 ]    [ 60 ]         ← Internal (keys only, no data)
          /    \    /    \
       [10]  [30] [50]  [70]       ← Leaf (data lives here, all linked)
        ↓     ↓    ↓     ↓    
      data  data  data  data
        ↔     ↔    ↔     ↔         ← linked list chain
```

---

## Leaf Nodes: The Key Difference

||B-Tree Leaf|B+ Tree Leaf|
|---|---|---|
|**Role**|Just the bottom — a dead end|The most important part of the tree|
|**Holds data?**|Yes (like every other node)|Yes — the ONLY place with data|
|**Linked?**|No|Yes — all connected in a chain|
|**Special?**|Nothing unique about them|Unique and central to the whole design|

---

## Why B+ Tree is Better for Range Queries and Sequential Scans

This is where B+ Tree truly shines — and it's all because of the **linked leaf nodes**.

### Range Query Example: find all records where `age` is between 20 and 50

**In a B-Tree:**

The data is scattered across all levels — root, internal, and leaf nodes all hold data. To collect everything in a range, you have to traverse the entire tree up and down, visiting many nodes across different levels. There is no shortcut.

```
  Must visit nodes across ALL levels to collect range results:

              [ 40 ]      ← maybe data here
             /       \
          [ 20 ]    [ 60 ]   ← maybe data here too
          /    \    /    \
       [10]  [30] [50]  [70]  ← and here
```

**In a B+ Tree:**

All data is at the leaf level and leaves are linked. So you just:

1. Search down the tree to find the **first leaf** that matches (e.g. leaf with 20)
2. **Walk the chain** left to right until you pass 50
3. Done — no need to go back up the tree at all

```
  Step 1: search down to find leaf[20]
  Step 2: walk the chain →

  [10] ↔ [20] ↔ [30] ↔ [40] ↔ [50] ↔ [60] ↔ [70]
              ↑_________________________↑
              collect everything in this range
```

### Sequential Scan Example: read ALL records in order

**In a B-Tree:** You have to do a full **in-order traversal** of the entire tree — going up and down through every level repeatedly.

**In a B+ Tree:** Just go to the **leftmost leaf** and walk the entire chain from left to right. Every record, in order, without ever going back up the tree.

```
  [10] → [20] → [30] → [40] → [50] → [60] → [70]
  Just walk the chain. Simple, fast, linear.
```

### Why This Matters in Databases

Queries like these are extremely common:

```sql
WHERE age BETWEEN 20 AND 50          -- range query
ORDER BY id                          -- sequential scan
WHERE created_at > '2024-01-01'      -- range query
```

B+ Tree handles all of them by just **walking the leaf chain** — no tree traversal needed after finding the start point. This is why almost every real database (MySQL, PostgreSQL, SQLite, Oracle) uses B+ Tree internally.

---

## Quick Comparison

|Feature|B-Tree|B+ Tree|
|---|---|---|
|Data location|Every node|Leaf nodes only|
|Leaf nodes|Dead end, no children|Linked chain, holds all data|
|Internal nodes|Keys + data|Keys only (routing)|
|Range queries|Slow — traverse whole tree|Fast — walk the leaf chain|
|Sequential scan|Slow — full tree traversal|Fast — walk left to right|
|Point lookups|Can stop early (any level)|Always goes to leaf|
|Used in databases|Rarely|Almost always|

---

## Summary

- A B-Tree stores data at **every node** — root, internal, and leaf.
- A B+ Tree stores data **only at leaf nodes**; everything above is just routing.
- **Leaf nodes in B-Tree** = just the bottom, nothing special.
- **Leaf nodes in B+ Tree** = where all data lives, and they're all linked together.
- A node with `n` keys always has `n+1` children.
- B+ Tree dominates for **range queries and sequential scans** because the linked leaf chain lets you collect sorted data by just walking left to right — no tree traversal needed.

---

## The Simple Rule to Remember

> **B-Tree** = "I want exactly this one thing, find it fast" **B+ Tree** = "I want everything between here and there, give it to me in order"

**B-Tree is better** when you're looking for a **single or minimum value** — data lives at every level so you might find it at the root or an internal node and stop right there. No need to go all the way down.

**B+ Tree is better** when you need **many values in a range or in order** — the linked leaf chain lets you walk left to right and collect everything without climbing up and down the tree.

And that's exactly why real databases almost always pick B+ Tree — most real queries are not just `WHERE id = 5`. They're more like `WHERE created_at > '2024-01-01'` or `ORDER BY age` — range and sequential scan queries where the linked leaf list makes all the difference.