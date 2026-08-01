# Operating System Notes: Module 2 — Process & Thread Management

## 1. Process vs. Program vs. Thread

* **Program**: Passive entity stored on disk containing machine code instructions, initialized data, and metadata (ELF format on Linux, PE format on Windows).
* **Process**: Active entity executing a program in RAM. Possesses its own isolated Virtual Address Space, Process ID (PID), memory mappings (text, data, bss, heap, stack), file descriptor table, and CPU execution context.
* **Thread**: The smallest unit of execution schedulable by the OS. Threads belonging to the same process share the process's code, data, heap, and open files, but maintain private stacks, CPU register states, and Program Counters (PC).

```
Process Virtual Address Space (Isolated per process)
+-------------------------------------------------------+
|  Kernel Space Mapping (Protected, Ring 0)             |
+-------------------------------------------------------+
|  User Stack (Thread 1)   |   User Stack (Thread 2)    |  <- Thread-Private
+--------------------------+----------------------------+
|  Shared Dynamic Heap (malloc / new)                   |  <- Process-Shared
+-------------------------------------------------------+
|  BSS Segment (Uninitialized Global Variables)         |  <- Process-Shared
+-------------------------------------------------------+
|  Data Segment (Initialized Global/Static Variables)   |  <- Process-Shared
+-------------------------------------------------------+
|  Text Segment (Compiled Machine Code Instructions)    |  <- Process-Shared
+-------------------------------------------------------+
```

---

## 2. Process Control Block (PCB)

The **Process Control Block (PCB)** is a kernel data structure representing a process.

```
+---------------------------------------------------+
|               Process Control Block (PCB)         |
+---------------------------------------------------+
| Process State (Running, Ready, Waiting, Zombie)   |
| Process ID (PID) & Parent PID (PPID)             |
| Program Counter (PC / RIP)                        |
| CPU Registers (RAX, RBX, RCX, RSP, RBP, RFLAGS)   |
| Memory Management Info (Page Table Pointer / CR3)|
| CPU Scheduling Info (Priority, Nice Value, Quantum)|
| Accounting Info (CPU Time Used, Execution Limits) |
| I/O & File Descriptor Table (Pointers to files)   |
| Signal Handling Masks & Handlers                  |
+---------------------------------------------------+
```

---

## 3. Process State Transitions & Lifecycle

```
                    +-------------------+
                    |       NEW         |
                    +---------+---------+
                              | Admitted
                              v
                    +-------------------+  Interrupt (Time Slice Out)
        +---------->|       READY       |-------------------+
        |           +---------+---------+                   |
        |                     |                             |
        | I/O or Event        | Scheduler                   |
        | Completion          | Dispatch                    |
        |                     v                             v
+-------+-----------+-------------------+          +-------------------+
|      WAITING      |<------------------|          |      RUNNING      |
|     (BLOCKED)     | I/O or Event Wait |          +---------+---------+
+-------------------+-------------------+                    |
                                                             | Exit
                                                             v
                                                   +-------------------+
                                                   |    TERMINATED     |
                                                   |     (ZOMBIE)      |
                                                   +-------------------+
```

### 3.1 State Description
* **New**: Process is being created, PCB allocated, code loaded from storage into RAM.
* **Ready**: Process resides in RAM ready to execute, waiting for CPU dispatch.
* **Running**: Instructions are executing on the CPU core.
* **Waiting / Blocked**: Process is paused waiting for an asynchronous event (I/O, lock acquisition, timer).
* **Terminated / Zombie**: Execution finished; process memory released, but PCB retained until parent reads exit code via `waitpid()`.

---

## 4. CPU Context Switching & Overhead

A **Context Switch** is the procedure where the OS saves the CPU execution state of a running process/thread and restores the saved state of a new process/thread.

```
Running Process P1               OS Kernel / Scheduler               Ready Process P2
+------------------+             +-------------------+               +------------------+
| Executing        |             |                   |               | Ready            |
|                  | Timer Interrupt                 |               |                  |
|                  |------------>| 1. Save P1 State  |               |                  |
|                  |             |    (RIP, Registers|               |                  |
|                  |             |     to PCB1)      |               |                  |
| Idle             |             |                   |               |                  |
|                  |             | 2. Pick P2 via    |               |                  |
|                  |             |    Scheduler      |               |                  |
|                  |             |                   |               |                  |
|                  |             | 3. Swap CR3 (MMU) |               |                  |
|                  |             | 4. Restore P2 Regs|               |                  |
|                  |             |    from PCB2      |               |                  |
|                  |<------------|                   |-------------->| Executes         |
|                  |  Return     |                   |  Return       |                  |
+------------------+             +-------------------+               +------------------+
```

### 4.1 Direct and Indirect Costs of Context Switching
1. **Direct Costs**: Saving/restoring CPU registers (~100–300 ns), executing scheduler code, and loading the memory page directory (`CR3` register write).
2. **Indirect Costs**:
   * **TLB Invalidation**: Swapping page tables flushes the Translation Lookaside Buffer (TLB), causing page table walk misses on subsequent memory accesses.
   * **CPU Cache Pollution**: L1/L2/L3 CPU caches lose locality as Process P2 evicts P1's cached lines.

---

## 5. Thread Execution Models & POSIX Pthreads

### 5.1 Thread Models

| Model | Ratio (User:Kernel) | Description | Advantages | Disadvantages |
| :--- | :--- | :--- | :--- | :--- |
| **Many-to-One (N:1)** | $N$ User threads to $1$ Kernel thread | Managed entirely in user space | Extremely fast context switches | One blocking syscall blocks **all** threads |
| **One-to-One (1:1)** | $1$ User thread to $1$ Kernel thread | **Standard on Linux (NPTL) & Windows** | Parallel execution on multi-core CPUs | Higher kernel overhead for thread creation |
| **Many-to-Many (M:N)** | $M$ User threads to $N$ Kernel threads | Hybrid model (e.g., Go Goroutines scheduler) | Scalable, highly efficient thread pool | Complex two-level thread scheduling logic |

### 5.2 POSIX Thread Creation in C
```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void* thread_function(void* arg) {
    int thread_id = *(int*)arg;
    printf("Hello from Thread %d! Thread ID: %ld\n", thread_id, pthread_self());
    pthread_exit(NULL);
}

int main() {
    pthread_t threads[3];
    int thread_args[3];

    for (int i = 0; i < 3; i++) {
        thread_args[i] = i + 1;
        if (pthread_create(&threads[i], NULL, thread_function, &thread_args[i]) != 0) {
            perror("Failed to create thread");
            return 1;
        }
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }
    printf("All threads joined successfully.\n");
    return 0;
}
```

---

## 6. Process Creation: `fork()`, `execve()`, Copy-on-Write (COW)

### 6.1 `fork()` and Copy-on-Write (COW)
When `fork()` is called:
1. Kernel duplicates the parent's PCB and file descriptor table.
2. **Instead of copying physical RAM pages**, the kernel marks all parent memory pages as **Read-Only** and duplicates parent page tables into child page tables point to the same physical memory frames.
3. If either process attempts to write to a page, a CPU **Page Fault** is triggered.
4. The kernel's `#PF` handler allocates a new physical frame, copies original page contents into it, marks the new page as **Read-Write**, and updates the writing process's page table.

```
Initial State After fork()          After Child Writes to Page 2 (COW Triggered)
Parent PTE2 ----> Physical Frame 2   Parent PTE2 ----> Physical Frame 2 (Read-Only)
                     ^                                                
Child PTE2 ------+ (Read-Only)      Child PTE2 -------> Physical Frame 2' (Read-Write Copy)
```

### 6.2 Zombie vs. Orphan Processes
* **Zombie Process**: A process that has completed execution (`exit()`), but its PCB entry remains in the process table because its parent has not yet called `wait()` or `waitpid()`.
  * *Clean up*: Parent must reap it via `wait()`, or if parent dies, `systemd` reaps it.
* **Orphan Process**: A running process whose parent process terminates before it does.
  * *Adoption*: Adopted immediately by `init` / `systemd` (PID 1), which periodically reaps terminating children.

---

## 7. CPU Scheduling Algorithms & Solved Problems

### 7.1 Core Metric Equations
$$	ext{Turnaround Time (TAT)} = 	ext{Completion Time (CT)} - 	ext{Arrival Time (AT)}$$
$$	ext{Waiting Time (WT)} = 	ext{Turnaround Time (TAT)} - 	ext{Burst Time (BT)}$$
$$	ext{Response Time (RT)} = 	ext{Time of First CPU Execution} - 	ext{Arrival Time (AT)}$$

---

### 7.2 Solved Problem: Comprehensive Comparative Scheduling

Consider 4 processes with the following parameters:

| Process | Arrival Time ($AT$) | Burst Time ($BT$) | Priority (Lower = Higher Priority) |
| :--- | :--- | :--- | :--- |
| **P1** | 0 | 8 | 3 |
| **P2** | 1 | 4 | 1 |
| **P3** | 2 | 9 | 4 |
| **P4** | 3 | 5 | 2 |

---

#### Algorithm 1: First-Come First-Served (FCFS)
* **Gantt Chart**:
  `[ P1 (0-8) | P2 (8-12) | P3 (12-21) | P4 (21-26) ]`

| Process | AT | BT | CT | TAT ($CT-AT$) | WT ($TAT-BT$) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| P1 | 0 | 8 | 8 | 8 | 0 |
| P2 | 1 | 4 | 12 | 11 | 7 |
| P3 | 2 | 9 | 21 | 19 | 10 |
| P4 | 3 | 5 | 26 | 23 | 18 |
| **Average** | | | | **15.25 ms** | **8.75 ms** |

---

#### Algorithm 2: Shortest Remaining Time First (SRTF - Preemptive SJF)
* **Gantt Chart Execution**:
  * $t=0$: P1 arrives ($BT=8$). Executes.
  * $t=1$: P2 arrives ($BT=4$). Compare P1 remaining ($7$) vs P2 ($4$). Preempt P1! P2 executes.
  * $t=2$: P3 arrives ($BT=9$). P2 remaining ($3$). P2 continues.
  * $t=3$: P4 arrives ($BT=5$). P2 remaining ($2$). P2 continues.
  * $t=5$: P2 finishes ($CT=5$). Ready pool: P1 ($7$), P3 ($9$), P4 ($5$). Smallest is P4. P4 executes.
  * $t=10$: P4 finishes ($CT=10$). Ready pool: P1 ($7$), P3 ($9$). Smallest is P1. P1 executes.
  * $t=17$: P1 finishes ($CT=17$). P3 executes.
  * $t=26$: P3 finishes ($CT=26$).
* **Gantt Chart**:
  `[ P1 (0-1) | P2 (1-5) | P4 (5-10) | P1 (10-17) | P3 (17-26) ]`

| Process | AT | BT | CT | TAT ($CT-AT$) | WT ($TAT-BT$) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| P1 | 0 | 8 | 17 | 17 | 9 |
| P2 | 1 | 4 | 5 | 4 | 0 |
| P3 | 2 | 9 | 26 | 24 | 15 |
| P4 | 3 | 5 | 10 | 7 | 2 |
| **Average** | | | | **13.0 ms** | **6.5 ms** |

---

#### Algorithm 3: Preemptive Priority Scheduling
*(Priority 1 is highest)*
* **Gantt Chart Execution**:
  * $t=0$: P1 arrives (Prio 3). Executes.
  * $t=1$: P2 arrives (Prio 1). Preempt P1! P2 executes.
  * $t=5$: P2 completes. Ready: P1 (Prio 3), P3 (Prio 4), P4 (Prio 2). Highest priority is P4. P4 executes.
  * $t=10$: P4 completes. Ready: P1 (Prio 3), P3 (Prio 4). Next highest is P1. P1 executes.
  * $t=17$: P1 completes. P3 executes.
  * $t=26$: P3 completes.
* **Gantt Chart**:
  `[ P1 (0-1) | P2 (1-5) | P4 (5-10) | P1 (10-17) | P3 (17-26) ]`
* Average TAT = $13.0$ ms, Average WT = $6.5$ ms.

---

#### Algorithm 4: Round Robin (RR with Quantum $Q = 3$)
* **Execution Trace**:
  * Ready Queue: `P1`
  * $t=0..3$: P1 executes ($rem=5$). Queue after $t=3$: `P2 (AT=1), P3 (AT=2), P4 (AT=3), P1`.
  * $t=3..6$: P2 executes ($rem=1$). Queue: `P3, P4, P1, P2`.
  * $t=6..9$: P3 executes ($rem=6$). Queue: `P4, P1, P2, P3`.
  * $t=9..12$: P4 executes ($rem=2$). Finished! $CT_{P4}=12$. Queue: `P1, P2, P3`.
  * $t=12..15$: P1 executes ($rem=2$). Queue: `P2, P3, P1`.
  * $t=15..16$: P2 executes ($rem=0$). Finished! $CT_{P2}=16$. Queue: `P3, P1`.
  * $t=16..19$: P3 executes ($rem=3$). Queue: `P1, P3`.
  * $t=19..21$: P1 executes ($rem=0$). Finished! $CT_{P1}=21$. Queue: `P3`.
  * $t=21..24$: P3 executes ($rem=0$). Finished! $CT_{P3}=24$.
* **Gantt Chart**:
  `[ P1 (0-3) | P2 (3-6) | P3 (6-9) | P4 (9-12) | P1 (12-15) | P2 (15-16) | P3 (16-19) | P1 (19-21) | P3 (21-24) ]`

| Process | AT | BT | CT | TAT ($CT-AT$) | WT ($TAT-BT$) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| P1 | 0 | 8 | 21 | 21 | 13 |
| P2 | 1 | 4 | 16 | 15 | 11 |
| P3 | 2 | 9 | 24 | 22 | 13 |
| P4 | 3 | 5 | 12 | 9 | 4 |
| **Average** | | | | **16.75 ms** | **10.25 ms** |

---

## 8. Summary Checklist & Flash Cards

* **Difference between Process and Thread?** Process has isolated virtual memory space; Threads share memory (heap, data, code) within the same process.
* **What triggers Copy-on-Write (COW)?** A write instruction to a read-only shared memory page following `fork()`.
* **What is Convoy Effect?** Short processes waiting behind a long CPU-bound process in FCFS scheduling.
* **What is Starvation and how is it solved?** Low priority processes waiting indefinitely; solved via **Aging** (gradually increasing process priority over time).
* **Why does Round Robin with very small quantum degrade performance?** Context switch overhead dominates CPU time.
