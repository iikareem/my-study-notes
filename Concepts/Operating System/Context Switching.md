---
tags:
  - concurrency
  - operating-system
---

**Context switching** is the process where the operating system (OS) **pauses one process or thread** and **resumes another**, by saving and loading their execution state.

It lets the CPU **quickly switch between tasks**, giving the illusion that multiple programs are running **simultaneously** — even on a single-core processor.

### What is the "context"?

The **context** means the **entire CPU state** of a process or thread, including:

- **Program Counter (PC)** – where to continue execution
    
- **CPU Registers** – working data
    
- **Stack Pointer**, **Flags**
    
- **Memory-related info** (from PCB)
    

---

### 🔁 Steps in Context Switching:

1. **OS interrupts** the currently running process (e.g., time slice ends or I/O occurs).
    
2. **Saves the current context** (PC + registers) into that process's **PCB**.
    
3. **Selects the next process** to run (from the scheduler).
    
4. **Loads the context** of that new process from its PCB into the CPU.
    
5. CPU **resumes execution** from the new process’s PC.

### 💡 Why is context switching important?

- Enables **multitasking**.
    
- Ensures **fair use** of CPU.
    
- Supports **responsive systems**.
    

---

### ⚠️ Downsides?

- **Overhead**: It takes CPU time to save/load state — this is called **context switch cost**.
    
- Too many switches → less actual work done.

### Tools that help reduce this:

- **Thread pools** (to reuse threads)
    
- **Async programming** (to avoid blocking)
    
- **Coroutines/lightweight threads** (like in Go or Kotlin)

| Downside                    | Description                                                                                                                  |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 🧠 **CPU Overhead**         | Saving and loading process/thread state takes CPU time — during this, **no useful work is done**.                            |
| 🕒 **Latency**              | Frequent switching can increase **response time**, especially if tasks are short or switching is excessive.                  |
| 🧵 **Thread Thrashing**     | Too many threads cause **frequent switches** without making progress. This is called **thrashing**.                          |
| 🗂️ **Cache Misses**        | Switching between processes or threads can cause **CPU cache** (L1/L2/L3) to be invalidated or flushed, hurting performance. |
| 🧩 **Complexity in Design** | Requires the OS to **manage scheduling, priorities, and fairness**, which adds complexity.                                   |
| 💾 **Memory Usage**         | More processes/threads mean more PCBs, stacks, and kernel resources in memory.                                               |

### Concurrency vs. Parallelism:

|Concept|Description|Context Switching?|
|---|---|---|
|**Concurrency**|Multiple tasks progress over time (maybe not at the same time)|✅ Yes, heavily used|
|**Parallelism**|Multiple tasks run truly at the same time (on multiple cores)|✅ Still used, but less switching|

---

### ⚠️ In highly concurrent systems:

- Too many threads = frequent context switches.
    
- This leads to **CPU overhead** (saving/loading states).
    
- Known as **"context switching overhead"** or **"thrashing"**.

**context switching in process and threads ?**

**Context switching** is the process by which an operating system (OS) saves the state of a currently executing process or thread and restores the state of another to resume its execution. It’s a core mechanism for multitasking, allowing multiple processes or threads to share CPU resources. Below, I explain context switching for **processes** and **threads**, highlighting their differences and key aspects.

### **Context Switching in Processes**

- **Definition**: Context switching between processes involves saving the complete state of one process (via its **Process Control Block, PCB**) and loading the state of another process to allow the CPU to execute it.
- **Steps Involved**:
    1. **Save Current Process State**: Store the process’s state in its PCB, including:
        - Program counter (PC)
        - CPU registers (general-purpose, status registers)
        - Memory management info (e.g., page tables, base/limit registers)
        - I/O status (open files, devices)
        - Accounting info (CPU time used)
    2. **Update Process State**: Mark the current process as "ready" or "blocked" in the scheduler.
    3. **Select New Process**: The OS scheduler selects the next process to run based on priority or scheduling algorithm.
    4. **Restore New Process State**: Load the new process’s PCB data into the CPU, including its memory context, registers, and program counter.
    5. **Switch Memory Context**: Update the memory management unit (MMU) to point to the new process’s address space (e.g., load page tables).
    6. **Resume Execution**: Start executing the new process at its saved program counter.
- **Overhead**:
    - **High Cost**: Process context switching is expensive because it involves:
        - Saving and restoring a large amount of state (PCB is comprehensive).
        - Switching the entire memory address space (e.g., updating page tables or TLB flush).
        - Potential cache invalidation, reducing performance.
    - **Time**: Typically takes microseconds to milliseconds, depending on the OS and hardware.
- **When It Happens**:
    - Time slice expires (preemptive scheduling).
    - Process blocks for I/O or waits for an event.
    - Higher-priority process becomes ready.
- **Example**: Switching from a web browser process to a text editor process when you alt-tab between applications.

### **Context Switching in Threads**

- **Definition**: Context switching between threads (within the same process or across processes) involves saving the state of one thread (via its **Thread Control Block, TCB**) and loading the state of another thread.
- **Steps Involved**:
    1. **Save Current Thread State**: Store the thread’s state in its TCB, including:
        - Program counter (PC)
        - CPU registers
        - Stack pointer (thread-specific stack)
        - Thread priority or status
    2. **Update Thread State**: Mark the current thread as "ready" or "blocked."
    3. **Select New Thread**: The scheduler selects the next thread to run (from the same or different process).
    4. **Restore New Thread State**: Load the new thread’s TCB data into the CPU (registers, stack pointer, PC).
    5. **Resume Execution**: Start executing the new thread.
    6. **Memory Context (if needed)**: If the threads belong to different processes, switch the memory context (similar to process switching). If within the same process, no memory context switch is needed.
- **Overhead**:
    - **Lower Cost (Same Process)**: Thread context switching within the same process is faster because:
        - Threads share the same memory address space, so no MMU or page table updates are needed.
        - Only thread-specific state (TCB) is saved/restored (smaller than PCB).
        - Cache hits are more likely since threads share data.
    - **Higher Cost (Different Processes)**: If threads belong to different processes, the overhead is similar to process context switching due to memory context changes.
    - **Time**: Typically faster than process switching (nanoseconds to microseconds for same-process threads).
- **When It Happens**:
    - Thread time slice expires.
    - Thread blocks (e.g., waiting for I/O or synchronization like a mutex).
    - Higher-priority thread becomes ready.
- **Example**: In a web browser process, switching between a UI-rendering thread and a JavaScript execution thread when a user clicks a button.

![[Pasted image 20250524150455.png]]