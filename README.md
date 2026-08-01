# Operating System Interview Notes & Company-Wise PYQs 🚀

[![Operating System](https://img.shields.io/badge/Subject-Operating%20System-blue.svg?style=for-the-badge&logo=linux)](https://github.com/tarunagnihotri534/OS-interview-Notes-PYQs)
[![FAANG Interview Ready](https://img.shields.io/badge/Level-Advanced%20FAANG%20Prep-orange.svg?style=for-the-badge)](https://github.com/tarunagnihotri534/OS-interview-Notes-PYQs)
[![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-brightgreen.svg?style=for-the-badge)](https://github.com/tarunagnihotri534/OS-interview-Notes-PYQs/pulls)
[![License](https://img.shields.io/badge/License-MIT-purple.svg?style=for-the-badge)](LICENSE)

> A comprehensive, battle-tested, interview-focused repository containing **Deep OS Notes**, **100+ Master Previous Year Questions (PYQs)**, **Solved Numerical Problems**, **Linux Kernel Internals**, and **Company-Wise Question Banks** for top tech companies (Google, Amazon, Microsoft, Meta, Apple, Uber, Qualcomm, Nvidia, Goldman Sachs, etc.).

---

## 👤 Author & Maintainer

**Tarun Agnihotri** ([@tarunagnihotri534](https://github.com/tarunagnihotri534))
* **GitHub**: [tarunagnihotri534](https://github.com/tarunagnihotri534)
* **Repository**: [OS-interview-Notes-PYQs](https://github.com/tarunagnihotri534/OS-interview-Notes-PYQs)

---

## 🏛️ High-Level Operating System Architecture

```
+-------------------------------------------------------------------+
|                        User Applications                          |
|             (Web Browsers, Databases, IDEs, Shells)               |
+-------------------------------------------------------------------+
|                       System API & Libraries                      |
|                (POSIX, glibc, Win32 API, vDSO)                    |
+-------------------------------------------------------------------+
=================== USER SPACE / KERNEL SPACE BOUNDARY ===================
+-------------------------------------------------------------------+
|                       System Call Interface                       |
|           (sys_fork, sys_execve, sys_read, sys_write)             |
+-------------------------------------------------------------------+
|                          Kernel Modules                           |
|  +-------------------+  +------------------+  +----------------+  |
|  | Process Scheduler |  | Memory Manager   |  | File System    |  |
|  +-------------------+  +------------------+  +----------------+  |
|  | Device Drivers    |  | Network Protocol |  | Security/IPC   |  |
|  +-------------------+  +------------------+  +----------------+  |
+-------------------------------------------------------------------+
|                    Hardware Abstraction Layer (HAL)              |
+-------------------------------------------------------------------+
=========================== HARDWARE LAYER ==========================
|       CPU (x86_64/ARM)  |  RAM / Memory  | Storage & Peripherals  |
+-------------------------------------------------------------------+
```

---

## 📚 Master Directory Structure & Navigation Index

### 1. 📖 Comprehensive OS Notes (`01_Notes/`)
| Module File | Key Topics Covered |
| :--- | :--- |
| 📚 [01. OS Architecture & Kernels](01_Notes/01_OS_Architecture_&_Kernels.md) | Dual-Mode (Ring 0 vs Ring 3), Traps vs Interrupts, Monolithic vs Microkernel vs Hybrid, Boot Process (BIOS/UEFI -> GRUB -> PID 1), vDSO, Meltdown & KPTI. |
| 📚 [02. Process & Thread Management](01_Notes/02_Process_&_Thread_Management.md) | PCB Structure, Process States, Context Switch Overhead, Pthreads, `fork()`, `execve()`, Copy-on-Write, Solved FCFS, SJF, SRTF, RR & Priority Scheduling. |
| 📚 [03. Process Synchronization](01_Notes/03_Process_Synchronization.md) | Critical Section Problem, Test-and-Set, CAS, Mutex, Spinlock, Semaphores, Monitors, C Code for Producer-Consumer, Readers-Writers, Dining Philosophers. |
| 📚 [04. Deadlocks](01_Notes/04_Deadlocks.md) | 4 Coffman Conditions, Resource Allocation Graphs, Prevention, Avoidance, Solved Banker's Algorithm with multi-resource matrices. |
| 📚 [05. Memory Management & Virtual Memory](01_Notes/05_Memory_Management_&_Virtual_Memory.md) | Paging, Segmentation, TLB Hit Rate, Page Table Bits, Page Fault Lifecycle, Solved FIFO, LRU, Optimal Page Replacement, Belady's Anomaly, Thrashing. |
| 📚 [06. Storage, I/O & File Systems](01_Notes/06_Storage_I_O_&_File_Systems.md) | VFS Layer, Inode Direct/Indirect Blocks, File Table, Solved SSTF, SCAN, C-SCAN Disk Scheduling, DMA Controller, MMIO vs PMIO, I/O Schedulers. |
| 📚 [07. Linux Internals & Sysadmin](01_Notes/07_Linux_Internals_&_Sysadmin.md) | Linux FHS Directory Hierarchy, systemd, OOM Killer (`oom_score_adj`), HugePages, `select` vs `poll` vs `epoll` vs `io_uring`, `strace` & `lsof` debugging. |

---

### 2. 📝 Master PYQs Collection (`02_PYQs/`)
| Module File | Key Questions Answered |
| :--- | :--- |
| 📝 [01. OS Core & Kernel PYQs](02_PYQs/01_OS_Core_&_Kernel_PYQs.md) | Deep dive into `fork()` + `execve()` execution, Monolithic vs Microkernel trade-offs, Traps vs Interrupts, vDSO syscall speedups, KPTI Meltdown mitigations. |
| 📝 [02. Concurrency & Sync PYQs](02_PYQs/02_Process_Concurrency_&_Sync_PYQs.md) | `fork() && fork() \|\| fork()` execution tree analysis, Mutex vs Semaphore vs Spinlock comparison, Priority Inversion & PIP, Runnable C code for Readers-Writers. |
| 📝 [03. Memory & VM PYQs](02_PYQs/03_Memory_&_Virtual_Memory_PYQs.md) | Complete 12-step Page Fault lifecycle, Mathematical proof of Belady's Anomaly for FIFO vs LRU, Working Set Model calculation for Thrashing. |
| 📝 [04. Storage & I/O PYQs](02_PYQs/04_Storage_I_O_&_File_Systems_PYQs.md) | Inode exhaustion troubleshooting ("No space left on device"), `epoll` $O(1)$ scalability internals, DMA Cycle Stealing & Cache Coherence handling. |
| 📝 [05. System Design & Debug PYQs](02_PYQs/05_System_Design_&_Linux_Debug_PYQs.md) | Production incident: Deleted open files consuming 100% disk space, High Load Average with low CPU utilization debugging, OOM Killer protection. |

---

### 3. 🏢 Company-Wise Interview Questions (`03_Company_Wise_PYQs/`)
| Company Guide | Primary Systems Focus |
| :--- | :--- |
| 🎯 [01. Google OS Questions](03_Company_Wise_PYQs/01_Google_OS_Interview_Questions.md) | Linux Namespaces, Cgroups v2, eBPF Tracing, EEVDF Scheduler, Lock-Free Concurrency. |
| 🎯 [02. Amazon / AWS OS Questions](03_Company_Wise_PYQs/02_Amazon_OS_Interview_Questions.md) | AWS Nitro Hypervisor Offloading, VM Memory Overcommit, EBS NVMe Kernel Sysctl Tuning. |
| 🎯 [03. Microsoft OS Questions](03_Company_Wise_PYQs/03_Microsoft_OS_Interview_Questions.md) | Windows NT Kernel vs Linux, HAL, IRP Packets, Dynamic Thread Quantum Adjustments. |
| 🎯 [04. Meta OS Questions](03_Company_Wise_PYQs/04_Meta_OS_Interview_Questions.md) | Zero-Syscall `io_uring`, Pressure Stall Information (PSI), Memory Reclaim, HugePages. |
| 🎯 [05. Apple OS Questions](03_Company_Wise_PYQs/05_Apple_OS_Interview_Questions.md) | XNU Kernel Architecture (Mach + BSD + I/O Kit), Mach IPC, Apple Silicon UMA Zero-Copy. |
| 🎯 [06. Uber & Netflix OS Questions](03_Company_Wise_PYQs/06_Uber_Netflix_Systems_Questions.md) | High-Throughput Zero-Copy (`sendfile`), Socket Buffers, Network Kernel Tuning. |
| 🎯 [07. Qualcomm & Nvidia Hardware OS](03_Company_Wise_PYQs/07_Qualcomm_Nvidia_Hardware_OS.md) | RTOS, ISR Top/Bottom-Half Processing, DMA Bus Drivers, Cache Line Invalidations. |
| 🎯 [08. Goldman Sachs & Enterprise OS](03_Company_Wise_PYQs/08_GoldmanSachs_Atlassian_Enterprise.md) | Low Latency High-Frequency Trading Tuning, CPU Affinity (`taskset`), NUMA Node Opt. |
| 📊 [09. Company Preparation Matrix](03_Company_Wise_PYQs/09_Company_Wise_OS_Matrix.md) | Master cross-company matrix table mapping top 25 tech companies to OS topics. |

---

## ⚡ 10-Minute Rapid Revision Cheat Sheet

### 1. The 4 Coffman Conditions for Deadlock
1. **Mutual Exclusion**: Non-shareable resource access.
2. **Hold and Wait**: Process holding resources while requesting more.
3. **No Preemption**: Resources cannot be forcibly taken.
4. **Circular Wait**: Closed chain of processes waiting for each other.

### 2. Core Scheduling Formulas
$$\text{Turnaround Time (TAT)} = \text{Completion Time (CT)} - \text{Arrival Time (AT)}$$
$$\text{Waiting Time (WT)} = \text{Turnaround Time (TAT)} - \text{Burst Time (BT)}$$

### 3. Effective Access Time (EAT) for Virtual Memory
$$\text{EAT} = (H \times T_{\text{TLB}}) + ((1 - H) \times (T_{\text{TLB}} + 2 \times T_{\text{RAM}}))$$

### 4. Linux x86_64 Syscall ABI Registers
* Syscall Number: `RAX`
* Arguments: `RDI`, `RSI`, `RDX`, `R10`, `R8`, `R9`
* Return Value: `RAX`

---

## 🛠️ How to Use This Repository

1. **For Concept Mastery**: Read through the 7 modules in [`01_Notes/`](01_Notes/). Pay special attention to ASCII architectural diagrams and C code implementations.
2. **For Interview Problem Solving**: Practice the worked numericals and conceptual questions in [`02_PYQs/`](02_PYQs/).
3. **For Targeted Company Revision**: Navigate directly to your target company in [`03_Company_Wise_PYQs/`](03_Company_Wise_PYQs/) before your interview round!

---

## 🤝 Contributing

Contributions, additional company questions, and corrections are welcome! Feel free to open a Pull Request or Issue.

---
*Created & Maintained with ❤️ by **Tarun Agnihotri** ([@tarunagnihotri534](https://github.com/tarunagnihotri534)) for Systems Engineers & Software Developers worldwide.*
