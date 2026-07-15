---
tags:
  - operating-system
---

![[Pasted image 20250524144731.png]]

**Registers:**

These are the smallest and fastest memory units, located directly within the CPU. They hold data and instructions that are actively being used by the CPU

**Cache Memory (L1, L2, L3):**

This is a small, fast memory that stores frequently accessed data and instructions, acting as a buffer between the CPU and main memory. L1 is the fastest and closest, followed by L2 and L3

L1 => PER CORE
L2 => Usually inside CPU chip (per core or shared)
L3 Cache => On CPU chip, shared among cores

- **Main Memory (RAM):**
    
    This is the primary workspace of the computer, where applications and data in use are stored. It's larger and slower than cache memory but faster than secondary storage.
	Main Memory (RAM):

|Segment|Description|
|---|---|
|**Text Segment** (Code Segment)|Contains the compiled **machine code instructions** of the program (read-only).|
|**Data Segment**|Stores **global and static variables** initialized by the programmer.|
|**BSS Segment**|Holds **global and static variables** that are **uninitialized or initialized to zero**.|
|**Heap**|A memory area used for **dynamic memory allocation** (e.g., using `malloc` in C or `new` in Java/C++). Grows **upward** in memory.|
|**Stack**|Stores **function call frames**, **local variables**, **return addresses**, and supports **recursive calls**. Grows **downward** in memory.|

High Memory Address
+----------------------+
|      Stack           |  ← grows down
+----------------------+
|      Unused          |
+----------------------+
|      Heap            |  ← grows up
+----------------------+
|      BSS             |
+----------------------+
|      Data            |
+----------------------+
|      Text (Code)     |
+----------------------+
Low Memory Address

### Summary:

- All these segments are loaded into **RAM** when a process runs.
    
- The **Operating System’s loader** sets up this layout during program startup.
    
- The CPU uses **virtual memory addressing**, and the **MMU (Memory Management Unit)** translates virtual addresses to physical RAM addresses. =>

## Virtual vs Physical Memory

| Term                             | Meaning                                                                        |
| -------------------------------- | ------------------------------------------------------------------------------ |
| **Virtual Address**              | What your program sees (e.g., 0x1000) — fake/logical address                   |
| **Physical Address**             | Real location in RAM (e.g., 0xFA0020)                                          |
| **MMU (Memory Management Unit)** | Hardware inside CPU that **maps** virtual → physical address using page tables |

The **main reason** we use **virtual memory + MMU** is to let **different processes safely and efficiently share physical RAM** without interfering with each other.

---

## ✅ Why this system exists:

### 1. **Memory Protection**

- Every process thinks it has its own private memory space.
    
- One process **cannot access or overwrite** another process’s memory.
    
- If it tries? 🔒 The OS blocks it or kills the process — this is enforced by the **MMU**.

### **Isolation**

- Virtual memory makes it **look like each process starts at address 0x0000**, even though they are mapped to different places in physical RAM.
    
- This keeps processes from “seeing” each other’s memory.

- **Secondary Storage (Hard Drives, SSDs):**
    
    This provides large storage capacity at a lower cost but is significantly slower than the above layers. It's used for long-term storage of data. 
    
- **Tertiary Storage (Optical Disks, Magnetic Tapes):**
    
    This is used for archival and backup purposes