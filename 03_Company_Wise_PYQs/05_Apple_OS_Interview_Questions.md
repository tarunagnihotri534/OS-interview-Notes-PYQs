# Company-Wise PYQs: Apple OS Interview Questions

## 🏢 Company Profile & Interview Focus
Apple OS interviews evaluate **XNU Kernel Architecture (Mach Microkernel + BSD Layer + I/O Kit)**, **Mach IPC (Ports & Messages)**, **Apple Silicon Unified Memory Architecture (UMA)**, **Cache Line MESI Coherence**, and **Real-Time Audio/Display Scheduling**.

---

## Q1. Explain Apple XNU Kernel Architecture (Mach + BSD + I/O Kit).
* **Difficulty**: FAANG Advanced

```
+-------------------------------------------------------------+
|             User Applications (macOS / iOS / iPadOS)        |
+-------------------------------------------------------------+
========================= USER / KERNEL BOUNDARY =========================
|                       XNU Kernel Architecture               |
| +---------------------+ +--------------------+ +----------+ |
| | Mach Microkernel    | | BSD Layer          | | I/O Kit  | |
| | - Tasks & Threads   | | - POSIX API      | | - Driver | |
| | - Mach IPC Ports    | | - VFS & Inodes   | |   Object | |
| | - Virtual Memory    | | - BSD Signals    | |   C++    | |
| +---------------------+ +--------------------+ +----------+ |
+-------------------------------------------------------------+
|                 Hardware (Apple Silicon M-Series / A-Series) |
+-------------------------------------------------------------+
```

* **Mach Layer**: Handles low-level primitives: threads, tasks, Mach IPC ports, thread scheduling, virtual memory management.
* **BSD Layer**: Provides POSIX compliance, BSD process model, Pthreads, VFS filesystems, networking stack, process credentials, and signal delivery.
* **I/O Kit**: Object-oriented C++ framework for high-performance hardware device drivers.

---

## Q2. How does Apple Silicon Unified Memory Architecture (UMA) alter CPU-GPU Memory Management?
* **Difficulty**: FAANG Advanced

### Detailed Answer
In traditional PC architectures, CPU and discrete GPU have separate physical RAM pools connected via PCIe bus (requiring expensive memory copies over PCIe). Apple Silicon UMA integrates CPU, GPU, and Neural Engine onto a single SoC sharing a single unified high-bandwidth LPDDR5 RAM pool.
* **OS Impact**: Eliminates zero-copy memory transfers between CPU and GPU! Operating system allocates memory buffers (`MTLBuffer`) accessible directly by both CPU and GPU shaders simultaneously without PCI-e page pin overhead.
