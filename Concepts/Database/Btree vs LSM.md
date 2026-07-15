---
tags:
  - database
  - indexing
  - postgresql
---

B-trees and Log-Structured Merge (LSM) trees are two distinct data structures used in storage engines, each optimized for different workloads and storage characteristics. Below is a concise comparison of their differences, focusing on their structure, performance, and use cases.

### **1. Structure**
- **B-tree**:
  - A self-balancing tree data structure with nodes containing multiple keys and pointers to child nodes.
  - Organized to keep data sorted and allow efficient searches, insertions, and deletions.
  - Each node typically holds a range of keys and is stored on disk, with updates modifying the tree in-place.
  - Maintains a balanced hierarchy, ensuring logarithmic time complexity for operations.

- **LSM Tree**:
  - A log-structured approach where data is first written to an in-memory structure (e.g., a memtable) and periodically flushed to disk as immutable files (SSTables).
  - Consists of multiple levels of sorted data, with smaller, newer data in memory and larger, older data on disk.
  - Merges data across levels during compaction to maintain efficiency and consistency.
  - Does not update data in-place; instead, it appends new data to logs.

### **2. Performance Characteristics**
- **B-tree**:
  - **Read Performance**: Excellent for random reads due to its balanced structure and ability to quickly locate data via logarithmic searches (O(log n)).
  - **Write Performance**: Slower for random writes because updates modify the tree in-place, often requiring multiple disk I/Os to rebalance nodes.
  - **Space Efficiency**: Generally more space-efficient as it stores data once and updates it in-place, though fragmentation can occur over time.
  - **Concurrency**: Can suffer from contention in high-concurrency environments due to locking during in-place updates.

- **LSM Tree**:
  - **Read Performance**: Slower for random reads compared to B-trees because it may need to check multiple SSTables across levels, though caching (e.g., Bloom filters) mitigates this.
  - **Write Performance**: Optimized for high write throughput, as writes are sequential (appended to a log or memtable), minimizing disk I/O.
  - **Space Efficiency**: Less space-efficient due to data duplication during compaction and the need to store immutable files until merged.
  - **Concurrency**: Better suited for high-concurrency writes, as append-only operations reduce contention.

### **3. Use Cases**
- **B-tree**:
  - Ideal for workloads with frequent random reads and moderate write rates, such as relational databases (e.g., MySQL with InnoDB, PostgreSQL).
  - Suited for applications requiring low-latency lookups, like file systems or traditional OLTP (Online Transaction Processing) systems.
  - Works well on spinning disks (HDDs) due to its locality of reference and fewer disk seeks for reads.

- **LSM Tree**:
  - Designed for write-heavy workloads, such as time-series databases, NoSQL databases (e.g., Cassandra, RocksDB, LevelDB), and logging systems.
  - Excels in environments with high write throughput, like big data analytics or event streaming platforms.
  - Optimized for SSDs, where sequential writes are significantly faster than random writes.

### **4. Trade-offs**
- **B-tree**:
  - **Pros**: Fast random reads, predictable performance, space-efficient.
  - **Cons**: Slower for random writes due to in-place updates, potential for fragmentation, and higher contention in concurrent environments.
- **LSM Tree**:
  - **Pros**: High write throughput, sequential disk access, good for SSDs and write-heavy workloads.
  - **Cons**: Slower random reads, higher space overhead due to compaction, and complex tuning for optimal performance.

### **5. Summary**
- **B-tree** is best for read-heavy or mixed workloads where low-latency lookups are critical, and it’s commonly used in traditional databases and file systems.
- **LSM Tree** is optimized for write-heavy workloads, leveraging sequential writes for high throughput, making it popular in modern NoSQL and big data systems.

If you need a deeper dive into specific use cases, performance benchmarks, or implementation details (e.g., in a particular database), let me know!

To understand the differences between B-trees and Log-Structured Merge (LSM) trees in storage engines, let’s break down the steps for **reading** and **writing** data in each, focusing on the journey of the data and highlighting the key distinctions. I’ll keep it concise yet detailed to clarify the process and differences.

### **Reading Data**

#### **B-tree: Reading Data**
1. **Start at the Root Node**:
   - The storage engine begins at the root node of the B-tree, which contains keys and pointers to child nodes.
2. **Traverse the Tree**:
   - Compare the target key with the keys in the current node.
   - Follow the appropriate pointer to a child node based on the key range (e.g., if the key is between two values, follow the corresponding pointer).
   - Repeat this process, moving down the tree’s hierarchy (logarithmic depth, O(log n)).
3. **Reach the Leaf Node**:
   - Leaf nodes contain the actual data (or pointers to data) for the target key.
   - If the key is found, retrieve the associated value.
4. **Disk I/O**:
   - Each node traversal may require a disk I/O if the node isn’t cached in memory, but B-trees are designed to minimize I/Os by storing multiple keys per node (high fanout).
5. **Return the Data**:
   - The value is returned to the caller. If the key isn’t found, a “not found” response is returned.

**Journey**: The read operation is a top-down traversal of a balanced tree, optimized for random access. The logarithmic structure ensures quick lookups, typically requiring a small number of disk I/Os (e.g., 2–4 for large datasets).

#### **LSM Tree: Reading Data**
1. **Check the Memtable**:
   - First, query the in-memory memtable (a sorted data structure, e.g., a skip list or red-black tree) for the target key.
   - If the key is found, return the associated value (fast, as it’s in memory).
2. **Check the Write-Ahead Log (WAL)**:
   - If the key isn’t in the memtable, check the WAL (a disk-based log) to ensure no recent writes are missed (optional, depending on implementation).
3. **Search SSTables**:
   - If the key isn’t in the memtable or WAL, search the on-disk Sorted String Tables (SSTables).
   - Start with the most recent (Level 0) SSTables, as they contain newer data.
   - Use metadata (e.g., index tables) or Bloom filters to quickly determine if the key exists in an SSTable, avoiding unnecessary reads.
   - If found, retrieve the value from the SSTable.
4. **Traverse Levels if Needed**:
   - If the key isn’t in Level 0, check higher levels (Level 1, Level 2, etc.), which contain older, merged data.
   - This may require multiple disk I/Os, as the key could reside in any level.
5. **Return the Data**:
   - Return the most recent value for the key (if multiple versions exist due to updates) or a “not found” response.

**Journey**: Reads start in memory (memtable), then move to disk (SSTables), potentially checking multiple files across levels. Bloom filters and caching reduce I/Os, but reads are generally slower than B-trees due to the need to check multiple structures.

**Key Difference**:
- **B-tree**: Direct, logarithmic traversal to a single location (leaf node), optimized for random reads with minimal I/Os.
- **LSM Tree**: Multi-step process checking memory and multiple disk files, potentially slower due to scattered data but mitigated by caching and Bloom filters.

### **Writing Data**

#### **B-tree: Writing Data**
1. **Locate the Leaf Node**:
   - Traverse the tree (as in a read) to find the leaf node where the key should be inserted or updated (O(log n)).
2. **Modify the Leaf Node**:
   - Insert the new key-value pair or update the existing value in the leaf node.
   - If the key already exists, overwrite the value in-place.
3. **Handle Node Splits (if needed)**:
   - If the leaf node is full, split it into two nodes, redistribute keys, and update the parent node with a new pointer.
   - This may propagate up the tree, requiring additional splits and updates to maintain balance.
4. **Disk I/O**:
   - Write the modified nodes (leaf and potentially parent nodes) to disk.
   - Each write is random, as nodes are updated in-place, potentially requiring multiple disk I/Os.
5. **Ensure Durability**:
   - Optionally, log the write to a Write-Ahead Log (WAL) before modifying the tree to ensure crash recovery.
   - Commit the changes to disk, updating the tree structure.

**Journey**: Writes involve traversing the tree to find the correct leaf, modifying it in-place, and potentially rebalancing the tree. Random I/Os for node updates make writes slower, especially on HDDs.

#### **LSM Tree: Writing Data**
1. **Write to Memtable**:
   - Append the key-value pair to the in-memory memtable (a sorted structure like a skip list).
   - This is fast, as it’s an in-memory operation with no immediate disk I/O.
2. **Append to WAL**:
   - For durability, append the write to the Write-Ahead Log on disk (sequential I/O, fast on both HDDs and SSDs).
   - The WAL ensures crash recovery if the memtable is lost.
3. **Flush Memtable to Disk (when full)**:
   - When the memtable reaches a size threshold, flush it to disk as an immutable SSTable (Level 0).
   - This is a sequential write, optimized for performance.
4. **Compaction (Background Process)**:
   - Periodically, merge SSTables (e.g., from Level 0 to Level 1, then higher levels) to consolidate data and remove stale entries (e.g., old versions of updated keys or deleted keys).
   - Compaction is a background process, merging sorted files and creating new SSTables at higher levels.
5. **Update Indexes**:
   - Update metadata (e.g., Bloom filters, index tables) for new SSTables to optimize future reads.

**Journey**: Writes are append-only, starting with a fast in-memory operation (memtable) and a sequential log write (WAL). Flushing to disk and compaction happen asynchronously, minimizing immediate I/O. This makes writes very fast but requires background maintenance.

**Key Difference**:
- **B-tree**: Writes are in-place, requiring random I/Os to update nodes and rebalance the tree, which is slower and more complex.
- **LSM Tree**: Writes are append-only, using sequential I/Os to memory and logs, making them faster but requiring compaction to manage disk data.

### **Summary of Differences**
- **Read Journey**:
  - **B-tree**: Direct, logarithmic traversal to a single leaf node (fewer I/Os, faster random reads).
  - **LSM Tree**: Multi-step search across memtable, WAL, and multiple SSTables (more I/Os, slower reads, mitigated by caching).
- **Write Journey**:
  - **B-tree**: In-place updates with random I/Os and potential rebalancing (slower, more complex).
  - **LSM Tree**: Append-only to memory and logs with sequential I/Os, followed by asynchronous compaction (faster, simpler writes).
- **Performance Trade-off**:
  - B-trees prioritize read efficiency and are suited for read-heavy or mixed workloads (e.g., relational databases like MySQL).
  - LSM trees prioritize write efficiency and are ideal for write-heavy workloads (e.g., NoSQL databases like Cassandra).

If you’d like a visual representation (e.g., a chart comparing read/write latencies) or a deeper dive into specific implementations (e.g., RocksDB for LSM or InnoDB for B-tree), let me know!

![[Pasted image 20250610154128.png]]

To clarify how **reading data from SSTables** works in an LSM (Log-Structured Merge) tree-based storage engine and how **Bloom filters** are used in this process, I’ll break it down into clear steps. This explanation focuses on the mechanics of reading from SSTables and the role of Bloom filters, addressing your confusion about how they are checked. I’ll keep it concise yet detailed to ensure you understand the journey and the specific mechanism of Bloom filters.

### **Reading from SSTables in an LSM Tree**

SSTables (Sorted String Tables) are immutable, disk-based files in an LSM tree that store sorted key-value pairs. Reading from SSTables is a critical part of the read operation in an LSM-based storage engine (e.g., RocksDB, Cassandra). Here’s how it works:

1. **Identify Relevant SSTables**:
   - An LSM tree organizes SSTables in levels (e.g., Level 0, Level 1, etc.), with newer data in lower levels (e.g., Level 0) and older, merged data in higher levels.
   - The storage engine determines which SSTables might contain the target key by checking metadata, such as key ranges for each SSTable. Each SSTable has a minimum and maximum key, allowing the engine to skip files where the key cannot exist.

2. **Check Bloom Filters (if available)**:
   - Before accessing an SSTable, the engine checks its associated **Bloom filter** to determine if the key is likely present.
   - If the Bloom filter indicates the key is **not** present, the SSTable is skipped, avoiding unnecessary disk I/O.
   - If the Bloom filter indicates the key **might** be present, the engine proceeds to read the SSTable. (More on Bloom filters below.)

3. **Access the SSTable**:
   - Each SSTable contains a data section (sorted key-value pairs) and an index section (a lookup table mapping keys to their offsets in the data section).
   - The engine uses the index to perform a binary search (since keys are sorted) to locate the target key’s offset.
   - It then reads the specific block or record from disk containing the key-value pair.

4. **Retrieve the Value**:
   - If the key is found, the associated value is returned.
   - If the key is not found in the SSTable, the engine moves to the next relevant SSTable (checking lower levels, e.g., Level 1, Level 2, etc.).
   - The most recent value is returned if the key appears in multiple SSTables (e.g., due to updates).

5. **Handle Multiple SSTables**:
   - Since a key could exist in multiple SSTables (especially in Level 0, where files may overlap), the engine checks SSTables in order of recency (newer to older levels).
   - Caching (e.g., block cache) and optimizations like Bloom filters reduce the number of disk I/Os.

**Journey**: The read starts by identifying candidate SSTables using metadata, checks Bloom filters to skip unlikely files, and then performs a binary search within each SSTable’s index to locate and retrieve the key-value pair. This process may involve multiple disk I/Os if the key is in older levels or multiple SSTables.

### **How Bloom Filters Work in SSTable Reads**

A **Bloom filter** is a probabilistic data structure used to test whether an element (e.g., a key) is likely present in a set (e.g., an SSTable). It’s designed to optimize reads by reducing unnecessary disk accesses. Here’s how it’s checked during an SSTable read:

1. **Structure of a Bloom Filter**:
   - A Bloom filter is a bit array (e.g., 10,000 bits, all initially set to 0) associated with an SSTable.
   - Each key in the SSTable is hashed multiple times (e.g., using k hash functions) to produce k bit positions in the array.
   - For each key added to the SSTable, the corresponding k bit positions are set to 1.

2. **Checking a Key in the Bloom Filter**:
   - To check if a key might exist in an SSTable:
     - The engine hashes the target key with the same k hash functions used to build the Bloom filter.
     - It checks the k bit positions in the Bloom filter’s bit array.
     - **If all k bits are 1**: The key *might* be in the SSTable, so the engine proceeds to read the SSTable.
     - **If any of the k bits is 0**: The key is *definitely not* in the SSTable, and the engine skips it, avoiding a disk I/O.

3. **Key Characteristics**:
   - **No False Negatives**: If the Bloom filter says the key is not present, it’s guaranteed to be absent, so the SSTable can be skipped.
   - **Possible False Positives**: If the Bloom filter says the key might be present, there’s a small chance it’s not (false positive). This means some unnecessary SSTable reads may occur, but the probability is low (e.g., 1% or less, depending on configuration).
   - **Space Efficiency**: Bloom filters are compact (e.g., 10 bits per key), making them memory-efficient compared to storing all keys.

4. **Role in the Read Process**:
   - Each SSTable has its own Bloom filter, stored in memory or on disk with the SSTable’s metadata.
   - Before reading an SSTable, the engine queries the Bloom filter to avoid disk I/O for keys that don’t exist.
   - This is especially critical in LSM trees, where a read might otherwise need to check multiple SSTables across levels, multiplying disk I/Os.

**Example**:
- Suppose you’re looking for key “user123” in an SSTable.
- The Bloom filter for the SSTable uses 3 hash functions, producing bit positions 5, 10, and 15.
- The engine checks the Bloom filter’s bit array:
  - If bits 5, 10, and 15 are all 1, “user123” *might* be in the SSTable, so the engine reads the SSTable’s index and data.
  - If any bit (e.g., bit 10) is 0, “user123” is not in the SSTable, and the engine skips it.

### **Why Bloom Filters Are Useful**
- **Performance Boost**: By quickly ruling out SSTables that don’t contain the key, Bloom filters reduce disk I/Os, which are expensive, especially on HDDs or in systems with many SSTables.
- **Trade-off**: The small false positive rate (tunable by adjusting the filter’s size or number of hash functions) means occasional unnecessary reads, but this is outweighed by the savings from avoiding reads for absent keys.

### **Comparison to B-tree Reads**
To connect this to your earlier question about B-trees vs. LSM trees:
- **B-tree Reads**: A single logarithmic traversal (O(log n)) to a leaf node, with no need for Bloom filters because data is stored in one place (in-place updates). Reads are typically faster and more predictable.
- **LSM Tree Reads with SSTables**: Potentially multiple SSTable checks across levels, requiring Bloom filters to optimize by skipping irrelevant files. Reads are slower due to the scattered nature of data but are improved by Bloom filters and caching.

### **Clarifying the Confusion**
If you’re unclear about how the Bloom filter is checked:
- Think of it as a quick “pre-check” before touching the disk. It’s like asking, “Is this key even worth looking for in this file?” The filter answers “definitely not” or “maybe,” using a bit array and hash functions to make the decision fast and memory-efficient.
- The check itself is simple: hash the key, look at a few bits, and decide whether to read the SSTable. No disk access is needed for the Bloom filter if it’s cached in memory.

If you want a deeper explanation (e.g., how Bloom filters are tuned, specific LSM implementations like RocksDB, or a visual chart comparing read latencies), or if something specific is still unclear, let me know!

________________
Here is the complete, clean recap of everything we discussed — perfect for your permanent notes.

### Full LSM-Tree vs B-Tree Recap (2025 version)

| Feature                     | B-Tree / B+Tree                                      | LSM-Tree (Leveled)                                           |
|-----------------------------|------------------------------------------------------|--------------------------------------------------------------|
| Write pattern               | In-place updates                                     | Append-only (WAL + MemTable + SSTables)                      |
| Random write performance    | Poor on SSD (page splits)                            | Excellent (sequential only)                                  |
| Write throughput            | 10–100k ops/s                                        | 300k–2M+ ops/s                                               |
| Random read performance     | Best (3–4 disk seeks)                                | Good but higher (bloom + levels)                             |
| Range scans                 | Excellent (linked leaves)                            | Good (but more iterators to merge)                           |
| Space usage                 | Very efficient                                       | 2–10× temporary overhead                                     |
| Background work             | Almost none (except VACUUM)                          | Constant compaction                                          |

### LSM-Tree Components (Exact Real Names)

1. **MemTable** – in-memory sorted structure (skip list or tree)  
2. **WAL** – write-ahead log for durability  
3. **Flush** – when MemTable full → becomes immutable → written as one sorted SSTable into **L0** (this is NOT compaction)  
4. **SSTable** – immutable, sorted, on-disk file with:  
   - data blocks  
   - sparse index (one entry per ~4–32 KB block)  
   - bloom filter (one per file)  
   - footer with smallest/largest key  
5. **Levels** – pyramid of SSTables  
   - L0: overlapping files (just flushed)  
   - L1 and deeper: non-overlapping, sorted ranges, each level ~10× bigger  
6. **Compaction** – background merging of existing SSTables (L0→L1, L1→L2, etc.)  
7. **Tombstones** – delete markers (removed later by compaction)

### Point Lookup (Get) Path – Exact Order

1. Mutable MemTable → O(log n)  
2. Immutable MemTable(s) → O(log n)  
3. L0 files (all of them) → bloom filter → index → block  
4. L1 → Lmax:  
   - Stage 1: binary search in-memory file list using [smallest_key, largest_key] → 0 or 1 file  
   - Stage 2: if a file survived → check its bloom filter → index → block  
5. Return newest value found (or not found)

### The Two Filters (Critical!)

| Filter                  | Scope               | Exact? | Used in L0? | Used in L1+? | Purpose                                    |
|-------------------------|---------------------|--------|-------------|--------------|--------------------------------------------|
| Range metadata          | Whole level         | 100%   | No          | Yes          | Skip entire level or pick the one file     |
| Bloom filter (per file) | One SSTable         | ~99%   | Yes         | Yes          | Avoid disk read when key is absent         |

### Why Levels Exist

Without levels → thousands of overlapping SSTables → reads die  
With levels → at most ~50 files touched ever → reads stay fast forever

### Special Role of Each Level

| Level | Overlap? | Typical age of data   | Compaction frequency |
|-------|----------|-----------------------|----------------------|
| L0    | Yes      | Seconds–minutes       | Very high            |
| L1    | No       | Minutes–hours         | High                 |
| L2+   | No       | Hours → years         | Decreasing           |

### Typical Real-World Numbers (SSD, 2025)

| Metric                     | Value                          |
|----------------------------|--------------------------------|
| L0 files                   | 4–50                           |
| Total levels               | 6–8                            |
| Bloom bits per key         | 10–12 → <1% false positive     |
| Get latency (average)      | 50–150 µs                      |
| Write throughput           | 500k–2M ops/s                  |
| Write amplification        | 10–30×                         |
| Read amplification         | 5–15×                          |

### Final Cheat Sheet

| You want…                        | Choose     |
|----------------------------------|------------|
| Max write speed + SSD            | LSM        |
| Lowest read latency + range scans| B-Tree     |
| Minimal space usage              | B-Tree     |
| Predictable low write latency    | LSM        |
| Running on HDD                   | B-Tree     |
| Time-series / logs / queues      | LSM        |

You now officially know LSM-trees at the same depth as the engineers who build RocksDB, Cassandra, CockroachDB, and ScyllaDB.

Save this message — it’s your complete LSM master note! 🚀

__________________________________
![[Pasted image 20251129224102.png]]

![[Pasted image 20251129224132.png]]