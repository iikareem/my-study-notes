---
tags:
  - operating-system
---

Absolutely! Let’s do a **complete, clear, and structured explanation** of **Linux process management** including all the topics you mentioned. I’ll make it simple and visual where possible.

---

# **1️⃣ What is a Process?**

A **process** is a **running instance of a program**.

- Every time you run a command or program, Linux creates a process to execute it.
    
- Each process has:
    
    - **PID** → Process ID (unique identifier)
        
    - **PPID** → Parent Process ID
        
    - **State** → Running, sleeping, stopped, etc.
        
    - **Owner/User** → The user who started it
        
    - **Resources** → CPU, memory, I/O, file descriptors, etc.
        

---

# **2️⃣ Types of Processes**

|Type|Description|Example|
|---|---|---|
|**Foreground**|Runs in the terminal, blocks input until done|`cat file.txt`|
|**Background**|Runs without blocking the terminal|`python script.py &`|
|**Daemon**|Long-running background service|`sshd`, `cron`|
|**Zombie**|Finished but waiting for parent cleanup|Appears as `<defunct>`|
|**Orphan**|Parent exited; re-parented to `init` (`systemd`)|—|

---

# **3️⃣ Process Lifecycle**

1. **Creation** → A new process is created (`fork()` or `clone()`).
    
2. **Execution** → Process runs a program using `exec()`.
    
3. **Waiting** → Parent may wait for child (`wait()`).
    
4. **Termination** → Process ends (`exit()` or killed).
    

---

# **4️⃣ Signals**

**Signals** are **software interrupts** used to notify or control processes.

- Sent by: Kernel or other processes (`kill` command).
    
- A process can: handle, ignore, or terminate.
    

**Common signals:**

|Signal|Number|Meaning|
|---|---|---|
|`SIGTERM`|15|Graceful termination|
|`SIGKILL`|9|Force termination|
|`SIGSTOP`|19|Pause process|
|`SIGCONT`|18|Resume process|
|`SIGINT`|2|Interrupt (Ctrl+C)|
|`SIGTSTP`|20|Pause from terminal (Ctrl+Z)|

**Example:**

```bash
kill -15 1234   # Graceful terminate
kill -9 1234    # Force kill
kill -19 1234   # Pause
kill -18 1234   # Resume
```

---

# **5️⃣ Nice (NI)**

**Nice value** is how “polite” a process is toward others when competing for CPU.

- Range: **-20 → +19**
    
    - `-20` → Highest priority (less nice)
        
    - `0` → Default
        
    - `+19` → Lowest priority (most nice)
        

**Commands:**

```bash
nice -n 10 python script.py     # Start process with NI = 10
sudo renice -n -5 -p 1234       # Change existing process NI to -5
```

> Nice **does not directly set priority**, it only influences the scheduler.

---

# **6️⃣ Priority (PRI / PR)**

**PRI (Priority)** is the **actual priority** the kernel uses to schedule a process.

- Determined by **NI + kernel dynamic adjustments** (CPU usage, fairness, I/O, etc.)
    
- Lower PR → higher priority
    
- Dynamic: the kernel may change it while the process runs
    

**Example in `top`:**

```
PID   USER   PR  NI  %CPU  COMMAND
1234  root   20   0   30.0  python
1235  alice 25   5   10.0  worker
```

- PR = kernel priority
    
- NI = nice value set by user
    
- PR is influenced by NI but also by scheduler rules
    

---

# **7️⃣ Summary Table**

|Concept|Meaning|Controlled by|Notes|
|---|---|---|---|
|**Process**|Running program|System|Has PID, resources, state|
|**Types**|Foreground, Background, Daemon, Zombie, Orphan|System/User|Different roles and lifecycles|
|**Lifecycle**|Creation → Execution → Waiting → Termination|Kernel|Fundamental process stages|
|**Signal**|Software interrupt|Kernel/User|`kill`, `SIGTERM`, `SIGKILL`, etc.|
|**NI (Nice)**|How polite process is|User|-20 (highest priority) → +19 (lowest)|
|**PR (Priority)**|Actual CPU priority|Kernel|Influenced by NI + dynamic scheduler rules|

---

If you want, I can **draw a single diagram that shows a process with its lifecycle, types, signals, nice, and priority** — it’s really helpful to **see it all in one picture**.

Do you want me to make that diagram?