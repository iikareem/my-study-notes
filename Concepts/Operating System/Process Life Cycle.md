---
tags:
  - operating-system
---

### **New (Created)**

- The process is **created**.
    
- OS allocates a **Process Control Block (PCB)** — stores process ID, memory pointers, register values, etc.

PCB: A **PCB** is a **data structure used by the operating system** to **track all the information about a running process**.

You can think of it like the **"profile" or "identity card"** for a process — containing everything the OS needs to manage and switch between processes.

The OS uses the PCB to:

- Know **which process is running** or needs to be resumed.
    
- Save a process's **state** during context switching.
    
- Track each process's **resources and status**.

### 🔍 What’s inside a PCB (at a high level)?

| Field                    | Description                                          |
| ------------------------ | ---------------------------------------------------- |
| **Process ID (PID)**     | Unique identifier for the process                    |
| **Program Counter (PC)** | Address of next instruction to execute               |
| **CPU Registers**        | Values that need to be restored when process resumes |
| **Process State**        | Ready, Running, Waiting, etc.                        |
| **Memory Info**          | Base and limit of the process's address space        |
| **Scheduling Info**      | Priority, queue position, etc.                       |
| **I/O Status**           | List of files or devices being used                  |
| **Accounting Info**      | Time used, process owner, etc.                       |

### **Program Counter (PC):**

- A **special CPU register** that holds the **memory address of the next instruction** to execute.
- The **Program Counter** (also called **Instruction Pointer** in some architectures) is a **special-purpose CPU register** that **holds the memory address of the next instruction** to be executed.
- - The CPU uses the PC to **know where to fetch the next instruction** from RAM.
- - After executing an instruction, the PC is usually **incremented** to point to the next one.
    
- In case of a jump or function call, the PC is updated to point to a different address.
### **CPU Registers:**

- **Small, fast memory inside the CPU**.
    
- Used to store **temporary data**, like:
    
    - Current instruction's data
        
    - Function return values
        
    - Loop counters
        
    - Memory addresses
        
    - Stack pointer, etc.

### 2. **Ready**

- The process is **loaded into memory** and is ready to be assigned to a CPU.
    
- It is **waiting** in the ready queue for the CPU to become available.

### 3. **Running**

- The process is **assigned to a CPU** and is currently executing instructions.
    
- Only **one process per core** can be in this state at a time.

### 4. **Waiting (Blocked)**

- The process is **waiting for I/O or an external event** (e.g., file read, user input).
    
- It can’t continue until the event is completed.
    

---

### 5. **Terminated (Exit)**

- The process **has completed** or has been **killed**.
    
- OS cleans up and **reclaims resources**.

Hierarchy: A process (with its PCB) can contain multiple threads, each with its own TCB. The PCB holds process-wide information (e.g., memory map), while TCBs manage thread-specific execution details.

Context Switching: Switching between processes (PCB) is heavier because it involves changing memory spaces. Switching between threads (TCB) within the same process is lighter since they share the same memory space.

- **PCB**: Essential for process isolation and resource management. Used by the OS scheduler to allocate CPU time and resources to processes.
- **TCB**: Critical for multithreading, enabling efficient concurrent execution within a process. Used by the thread scheduler to manage thread execution.

- **Example Scenario**: In a multi-threaded web server (one process), the PCB tracks the server’s memory and open sockets, while each client-handling thread has a TCB to manage its execution state (e.g., processing a specific HTTP request).

| Component | Role                                                                            |
| --------- | ------------------------------------------------------------------------------- |
| **RAM**   | Holds the process’s code, data, heap, stack (loaded by OS).                     |
| **CPU**   | Executes instructions fetched from RAM using registers.                         |
| **PCB**   | Stores the state of the process, including memory info and CPU register values. |

### PCB Register vs CPU Register in Data Flow

- **PCB registers** (on the circuit board) often act as **buffers or temporary storage** holding data or instructions _before_ the CPU processes them.
    
- The CPU **loads data from these PCB registers into its own internal registers** to perform calculations or execute instructions.