---
tags:
  - database
  - indexing
  - postgresql
---

Great — let’s break each of these `pgstatindex` output fields down **in detail**, so you can understand what they mean, why they matter, and how to interpret them:

---

### 📦 `index_size` (in bytes)

- **What it is**: The total size of the index on disk.
    
- **Why it matters**: A very large index might indicate:
    
    - It's on a high-cardinality column (many unique values).
        
    - It's bloated (e.g., due to many updates/deletes).
        
- **What to do**:
    
    - Use `pg_size_pretty()` to make it human-readable.
        
    - Consider `REINDEX` if it's much larger than expected.
        

---

### 🌿 `avg_leaf_density` (in %)

- **What it is**: The **average fullness** of the leaf pages in the B-Tree.
    
- **Ideal value**: **70–90%**
    
- **Why it matters**:
    
    - Low density (< 50%) means pages are half-full → waste of space and more I/O.
        
    - This happens with many UPDATEs or DELETEs, where old entries are just marked dead.
        
- **What to do**:
    
    - If this is low → `REINDEX` or `VACUUM FULL` the table or index.
        

---

### 🔧 `leaf_fragmentation` (in %)

- **What it is**: The percentage of leaf pages that are **not physically ordered**.
    
- **Ideal value**: **Close to 0%**
    
- **Why it matters**:
    
    - Fragmented pages = more random I/O = slower index scans.
        
    - Especially hurts performance on **range queries** (e.g., `BETWEEN`, `>`, `<`).
        
- **What to do**:
    
    - If fragmentation is high (> 20–30%) → consider `REINDEX`.
        

---

### 🌲 `btree_level`

- **What it is**: The number of **levels** in the B-Tree structure.
    
- **Ideal value**: **2 or 3**
    
- **Why it matters**:
    
    - Each level adds extra reads per lookup.
        
    - Very high levels (4+) are rare but may hurt performance.
        
- **What to do**:
    
    - If the index gets too deep, you may be indexing a **very large dataset** or the key is too wide.
        
    - Rebuild may help in some cases.
        

---

### 📄 `num_leaf_pages`

- **What it is**: The count of **leaf pages** (bottom level of the B-Tree, where actual index entries are).
    
- **Why it matters**:
    
    - Helps you estimate density and space usage.
        
    - Used in combination with `avg_leaf_density` to assess index bloat.
        

---

### ✅ Example Interpretation

|Metric|Value|Interpretation|
|---|---|---|
|`index_size`|128 MB|A moderately large index|
|`avg_leaf_density`|45%|⚠️ Low → many pages are underutilized|
|`leaf_fragmentation`|35%|⚠️ High → slow range scans|
|`btree_level`|3|✅ Good — typical for medium-sized index|
|`num_leaf_pages`|8000|Means the index has many leaf blocks|

📌 **Conclusion**: The index is bloated and fragmented → should be **REINDEXed**.

---

Let me know if you'd like a **custom SQL** to find all indexes with low density or high fragmentation automatically.