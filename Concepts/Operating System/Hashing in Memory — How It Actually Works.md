Here's a full breakdown of how hashing actually works in memory — from the hash function to the bucket to the collision chain.

---

# Hashing in Memory — How It Actually Works

## The core idea

A hash map is just an **array** sitting in RAM. Nothing magical. You pick a size (say, 8 slots), and each slot is called a **bucket**. The hash function's only job is to take your key and return a number — then you do `number % 8` to find which bucket to put the value in.

## What lives in RAM

When you create a hash map, the runtime allocates a contiguous block of memory — an array of pointers. Each pointer either points to `null` (empty bucket) or points to a linked list node (which holds a key, a value, and a pointer to the next node).

Let me draw this out:---
![[Pasted image 20260722113929.png]]

## Breaking down what you see

**The array (the buckets)** lives in a fixed block of RAM. Each slot is just a pointer — it either holds `null` (nothing here) or an address pointing somewhere else on the heap.

**The nodes** live on the heap — a separate region of RAM where dynamic allocations go. Each node is a small struct with three fields: the key, the value, and a pointer to the next node (or `null` if it's the last one).

**The hash function** doesn't care about RAM at all. It just takes your key, crunches it into a number, and you do `number % bucket_count` to get an index. That index is where in the array to look.

## What a collision actually is in memory

When two different keys produce the same bucket index, you have a collision. Neither key "wins" — they both get a node on the heap, and the bucket's pointer just chains them together. So bucket `[2]` doesn't point to a key-value pair directly — it points to a linked list of key-value pairs.

When you call `get("lang")`, the runtime:

1. hashes `"lang"` → gets index 2
2. follows the pointer to the first node
3. checks: does this node's key equal `"lang"`? No, it's `"name"`
4. follows `next` pointer to the second node
5. checks again: `"lang"` matches → returns `"Python"`

That's why in the worst case (everything collides into one bucket), a hash map degrades to O(n) — you're walking a linked list.

## Why it's fast normally

With a good hash function and enough buckets, most buckets have 0 or 1 entries. So lookup is: hash the key → go directly to that index → done. O(1), no walking.

The hash map trades **RAM space** (those empty bucket slots aren't free) for **time** (you jump straight to the answer instead of scanning).

---
Exactly right. You've nailed the core insight.

When the CPU fetches data, it doesn't grab just one byte — it pulls an entire **cache line** (typically 64 bytes) from RAM into the L1/L2 cache, betting that you'll need the nearby memory soon. This is **spatial locality** — the assumption that if you touched address X, you'll probably touch X+8, X+16, etc. next.

Hash maps with chaining break this bet in two places:

**First jump — bucket array to heap node.** The bucket holds a pointer (an address), not the data itself. Following that pointer jumps to a completely different location in RAM — wherever `malloc` decided to put that node. The cache line you just loaded (containing the bucket array) is now useless. You pay another cache miss.

**Second jump — node to next node.** Each linked list node was allocated separately, at different times, scattered across the heap. Node 1 might be at address `0x1a40`, node 2 at `0xf830`. Following `next` is another random jump, another cache miss.

So for every key lookup you potentially pay **two or more cache misses**, and the CPU's prefetcher can't help because the pattern is unpredictable — it's chasing pointers into random heap addresses.

---

This is exactly why **open addressing** (like Python's `dict` and most modern hash maps) was invented. Instead of chaining into a linked list on the heap, it stores everything **inside the array itself**. On a collision, it probes to the next slot in the same array — which is likely already in the cache line you just loaded.

The tradeoff is that the array needs to stay sparse (usually kept under 70% full), but you get dramatically better cache behavior because your data is contiguous.

So your mental model is exactly right: chaining = pointer chasing = cache misses = you're paying RAM latency (100+ cycles) instead of cache latency (4 cycles for L1) on every hop.