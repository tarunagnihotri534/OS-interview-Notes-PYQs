# Operating System Notes: Module 1 — OS Architecture & Kernel Internals

## 1. Introduction to Operating Systems & Core Principles

An **Operating System (OS)** is system software that acts as an intermediary between computer application programs and hardware components. It abstracts complex hardware mechanisms, allocates system resources efficiently, and enforces security and process isolation.

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

### 1.1 The 5 Core Problems Solved by an OS
1. **Resource Allocation & Arbitrage**: Manages finite hardware resources (CPU cycles, RAM pages, disk blocks, network bandwidth) fairly and efficiently among competing processes.
2. **Hardware Abstraction**: Hides heterogeneous physical hardware details behind uniform interfaces (e.g., viewing a disk sector, SSD block, or NVMe drive as a unified Virtual File System `vfs`).
3. **Process Isolation & Protection**: Prevents malicious or bug-ridden user applications from accessing or overwriting memory allocated to other applications or the kernel core.
4. **Concurrency & Execution Management**: Enables simultaneous execution of multiple processes via hardware CPU multiplexing (time-slicing) and multi-core thread scheduling.
5. **Security & Access Control**: Enforces user privileges, capabilities, system call limits, and file access policies across system users.

---

## 2. Hardware CPU Modes & User Space vs. Kernel Space

Modern CPUs implement privilege rings to restrict hardware access based on execution context.

```
       +---------------------------------------------+
       | Ring 3: User Space (Applications, Shells)   |  <- Least Privileged
       |   +-------------------------------------+   |
       |   | Ring 1 & 2: Device Drivers (Rare)   |   |
       |   |   +-----------------------------+   |   |
       |   |   | Ring 0: Kernel Space (OS)   |   |   |  <- Most Privileged
       |   |   +-----------------------------+   |   |
       |   +-------------------------------------+   |
       +---------------------------------------------+
```

### 2.1 Privilege Levels (Ring 0 vs. Ring 3)
* **User Space (Ring 3)**: Code executes with restricted instructions. Applications cannot directly access hardware registers, modify page tables, trigger port I/O, or disable CPU interrupts.
* **Kernel Space (Ring 0)**: Code executes with unconstrained CPU access. Can execute privileged instructions (`cli`, `sti`, `hlt`, `lgdt`, `mov cr3`), directly access physical RAM, and manage peripheral devices.

### 2.2 Dual-Mode Switching Mechanism
When a user application requests a kernel service (e.g., reading a file), the CPU undergoes a **Mode Switch**:

```
User Mode (Ring 3)                                 Kernel Mode (Ring 0)
+------------------+                               +------------------+
| Executing App    |                               |                  |
| Code             |                               |                  |
|                  |                               |                  |
| sys_read() called|                               |                  |
| Load args into   |                               |                  |
| RAX, RDI, RSI    |                               |                  |
|                  |                               |                  |
| SYSCALL Instruction ---> 1. Save User SS/RSP --->| Trap Handler     |
| (Hardware Shift) |    2. Switch to Kernel Stack  | Executed         |
|                  |    3. Save RIP, RFLAGS        |                  |
|                  |    4. Load IA32_LSTAR -> RIP  | Lookup Syscall   |
|                  |                               | Table [RAX]      |
|                  |                               |                  |
|                  |                               | Execute sys_read |
|                  |                               |                  |
|                  |<--- SYSRET Instruction <------- Return Result    |
| Resumes in User  |    1. Restore User Stack      | in RAX           |
| Code             |    2. Restore RIP, RFLAGS     |                  |
+------------------+                               +------------------+
```

### 2.3 Micro-steps of a Mode Transition (`syscall` / `sysret`)
1. **User Request**: User code prepares system call argument values in specific CPU registers (x86_64: `RAX` = syscall number, `RDI` = arg1, `RSI` = arg2, `RDX` = arg3, `R10` = arg4, `R8` = arg5, `R9` = arg6).
2. **CPU Exception/Trap Entry**: Executes the `SYSCALL` instruction.
3. **Privilege Shift**:
   * Hardware loads the RIP (Instruction Pointer) from Model Specific Register `IA32_LSTAR`.
   * Switch CPU privilege mode from Ring 3 to Ring 0.
   * Saves user space `RSP` and swaps to the process's Kernel Stack.
4. **Kernel Execution**: Kernel dispatches control to `system_call_entry()`, validates arguments, looks up the system call table, and executes the kernel routine.
5. **Return (`SYSRET`)**: Restores user registers, switches CPU mode back to Ring 3, and jumps to user instruction following the syscall.

---

## 3. Interrupts, Traps, and Exceptions

The CPU alters normal instruction execution flow in response to events categorized into three types:

```
                          +-------------------------+
                          |   Hardware / Software   |
                          |     Interrupt Event     |
                          +------------+------------+
                                       |
           +---------------------------+---------------------------+
           |                           |                           |
+----------v----------+     +----------v----------+     +----------v----------+
| Hardware Interrupt  |     |   Software Trap     |     |   CPU Exception     |
| (Asynchronous)      |     | (Synchronous)       |     | (Synchronous)       |
+---------------------+     +---------------------+     +---------------------+
| External device IRQ |     | User-triggered call |     | CPU detects error   |
| (Keyboard, Timer,   |     | (int 0x80, syscall, |     | (Page fault, Divide |
| Disk DMA, NIC)      |     | breakpoint trap)    |     | by zero, GPF)       |
+---------------------+     +---------------------+     +---------------------+
```

### 3.1 Classification Matrix

| Feature | Hardware Interrupt | Software Trap | Exception |
| :--- | :--- | :--- | :--- |
| **Origin** | External Hardware (I/O device, NIC, Timer) | Software Instruction (`syscall`, `int 0x80`) | Internal CPU Error during instruction execution |
| **Synchronicity** | **Asynchronous** (Can occur at any time) | **Synchronous** (Occurs at specific instruction) | **Synchronous** (Occurs at specific instruction) |
| **Example** | Timer interrupt, Keyboard key press | System call invocation, `int 3` breakpoint | Divide-by-zero, Page Fault (`#PF`), Segment Fault |
| **Handling** | Interrupt Controller (APIC) -> IDT Lookup | Trap Handler in IDT -> Syscall Table | Fault Handler -> Resolves fault or sends `SIGSEGV` |

### 3.2 Interrupt Handling Mechanism (IDT)
1. Device signals **Interrupt Request (IRQ)** line to the **Advanced Programmable Interrupt Controller (APIC)**.
2. APIC maps IRQ to an Interrupt Vector Number (0–255) and signals CPU `INTR` pin.
3. CPU completes current instruction, pushes `RFLAGS`, `CS`, and `RIP` onto the kernel stack.
4. CPU indexes the **Interrupt Descriptor Table (IDT)** using vector number to retrieve the **Interrupt Service Routine (ISR)** function pointer.
5. ISR executes, handles device event, acknowledges APIC (sending End of Interrupt - EOI), and calls `IRET` (Interrupt Return) to resume user process.

---

## 4. Kernel Architecture Taxonomy

Kernels are structured based on how OS services (file systems, memory managers, networking, drivers) are organized and executed.

```
Monolithic Kernel                    Microkernel Architecture
+-------------------------------+    +-------------------------------+
| User Apps (Web, Database)     |    | User Apps | Drivers | FileSys |
+-------------------------------+    +-------------------------------+
| KERNEL SPACE (Ring 0)         |    | USER SPACE (Ring 3)           |
| - Process Scheduler           |    +===============================+
| - Virtual Memory Manager      |    | KERNEL SPACE (Ring 0)         |
| - Virtual File System (VFS)   |    | - IPC Mechanism               |
| - Network Protocols (TCP/IP)  |    | - Basic Thread Scheduling     |
| - Device Drivers              |    | - Low-level Address Space Mgmt|
+-------------------------------+    +-------------------------------+
```

### 4.1 Monolithic Kernel
* **Architecture**: All core OS services run together inside a single large binary image in Kernel Space (Ring 0).
* **Examples**: Linux, FreeBSD, OpenBSD.
* **Pros**: High execution performance (services communicate via direct in-memory C function calls).
* **Cons**: Poor fault isolation (a bug in a third-party GPU driver can crash the entire operating system via Kernel Panic).

### 4.2 Microkernel
* **Architecture**: Moves non-essential services (device drivers, file systems, protocol stacks) out of kernel space into User Space processes. The kernel only provides IPC, basic thread scheduling, and physical memory management.
* **Examples**: seL4, L4, Minix 3, QNX (used in automotive/embedded systems).
* **Pros**: High reliability and modular security (if the File System crashes, it can be restarted without crashing the system).
* **Cons**: IPC Performance Overhead (message passing between user-space services requires repeated mode switches and context switches).

### 4.3 Hybrid Kernel
* **Architecture**: Combines microkernel modularity with monolithic performance by running key user-space service subsystems inside kernel space.
* **Examples**: Windows NT (NTOSKRNL.EXE), macOS / iOS XNU (Mach microkernel + BSD subsystem + I/O Kit).

### 4.4 Comprehensive Kernel Taxonomy Comparison

| Dimension | Monolithic Kernel | Microkernel | Hybrid Kernel | Exokernel |
| :--- | :--- | :--- | :--- | :--- |
| **Component Location** | All in Ring 0 | Minimal in Ring 0, rest in Ring 3 | Microkernel core + key services in Ring 0 | Minimal protection layer in Ring 0 |
| **Communication** | Direct C function call | Message Passing IPC | Direct Calls & Shared Memory | Library OS (LibOS) direct HW access |
| **Crash Vulnerability** | High (Driver crash = Panic) | Low (Driver crash = Process restart) | Moderate | Extremely Low |
| **IPC Overhead** | Minimal ($0$ switch penalty) | High (Multiple mode switches) | Low to Moderate | Zero (Application manages HW) |

---

## 5. Operating System Boot Process

The boot sequence transitions system control from physical hardware reset to a fully operational userspace.

```
+------------------+      +------------------+      +------------------+      +------------------+
|  1. Firmware     | ---> |  2. Bootloader   | ---> |  3. Kernel Exec  | ---> |  4. Init/Systemd |
|  POST / BIOS/UEFI|      |  MBR/GPT -> GRUB |      |  vmlinuz -> RAM  |      |  PID 1 Launch    |
+------------------+      +------------------+      +------------------+      +------------------+
```

### 5.1 Step-by-Step Execution Sequence
1. **Power-On & Firmware Initialization (POST)**:
   * Power Supply Stabilization -> CPU RESET line released.
   * CPU begins executing code at reset vector address (`0xFFFFFFF0` on x86).
   * **BIOS / UEFI** runs **Power-On Self-Test (POST)** to verify hardware (RAM, GPU, Drives).
2. **Boot Device Selection & Bootloader Loading**:
   * **Legacy BIOS**: Reads first 512-byte sector (**MBR - Master Boot Record**) of boot device into RAM at `0x7C00` and executes Stage 1 bootloader.
   * **Modern UEFI**: Reads **EFI System Partition (ESP)** formatted with FAT32, identifies NVRAM boot targets, loads UEFI application binary (`grubx64.efi` or `systemd-boot`).
3. **Bootloader Execution (GRUB Stage 2)**:
   * Initializes basic display, filesystem drivers, loads kernel image (`vmlinuz`) and initial RAM disk (`initramfs.img`) into RAM.
   * Passes kernel boot parameters (e.g., `root=/dev/nvme0n1p2 ro quiet init=/sbin/init`) to kernel entry point.
4. **Kernel Initialization**:
   * Kernel unpacks `initramfs` (a temporary in-memory root filesystem).
   * Initializes CPU page tables, memory allocators (Buddy System, Slab Allocator), Interrupt Descriptor Table (IDT), device drivers, and mounts real root filesystem.
5. **Userspace Startup (PID 1)**:
   * Kernel executes `/sbin/init` (or `systemd`) as **Process ID 1 (PID 1)** in User Space.
   * `systemd` parses target files (`multi-user.target`, `graphical.target`), spawns system daemons (`sshd`, `dbus`, `networkd`), and launches login prompt / graphical UI.

---

## 6. System Calls & Linux vDSO Architecture

### 6.1 Linux System Call Registers (x86_64 ABI)

| Argument Position | CPU Register |
| :--- | :--- |
| **Syscall Number** | `RAX` |
| **Argument 1** | `RDI` |
| **Argument 2** | `RSI` |
| **Argument 3** | `RDX` |
| **Argument 4** | `R10` |
| **Argument 5** | `R8` |
| **Argument 6** | `R9` |
| **Return Value** | `RAX` (Negative value indicating `-errno` on failure) |

### 6.2 Fast System Calls via vDSO (virtual Dynamic Shared Object)
* **Problem**: Executing a full hardware privilege switch (`SYSCALL` / `SYSRET`) for frequently queried, non-mutating system calls like `gettimeofday()` or `clock_gettime()` imposes CPU pipeline flush costs (~100 CPU cycles).
* **Solution (vDSO)**: The Linux kernel exports a small shared memory page containing kernel-provided code directly mapped into every user process's address space.
* **Working**: When an application calls `clock_gettime()`, glibc invokes the function inside the mapped `vDSO` page. The code reads time structure data updated periodically by kernel hardware timers directly from user space **without issuing a CPU `SYSCALL` instruction**.

---

## 7. Meltdown & Spectre Security Vulnerabilities

### 7.1 Speculative Execution & Side-Channel Attacks
* **Meltdown (CVE-2017-5754)**: Exploited out-of-order execution in Intel CPUs. A user-space instruction attempted to read a kernel-space memory address. While the CPU eventually raised a Page Fault exception, the CPU speculatively executed subsequent instructions using the leaked data before the exception was committed, caching data in L1 Cache. Attacked processes could measure cache timing (Flush+Reload) to extract arbitrary kernel memory secrets.
* **KPTI Fix (Kernel Page Table Isolation)**:
  * Prior to KPTI, user page tables contained full kernel space mappings (marked Supervisor-only).
  * KPTI splits page tables into two sets:
    1. **User Page Table**: Contains user space mappings and only minimal kernel entries (trap vectors, syscall entry code).
    2. **Kernel Page Table**: Full system memory mapping, loaded into `CR3` register only when a CPU privilege switch to Ring 0 occurs.
  * **Cost**: Every system call requires swapping `CR3` registers, causing TLB flushes and a 5–30% CPU overhead on older hardware.

---

## 8. Summary Checklist & Flash Cards

* **What is Ring 0 vs Ring 3?** Ring 0 is Kernel Mode (unrestricted hardware access); Ring 3 is User Mode (restricted instructions, isolated memory).
* **Difference between Trap and Interrupt?** Interrupts are asynchronous hardware signals; Traps are synchronous software execution events (syscalls).
* **Why do microkernels suffer from performance issues?** Due to high IPC overhead requiring frequent privilege ring transitions and context switches between isolated user-space driver modules.
* **What is PID 1?** The first user-space process spawned by the kernel during boot (`init` or `systemd`), parent to all orphan processes.
* **How does vDSO optimize system calls?** Maps kernel-updated read-only data into user space, allowing syscall execution like `gettimeofday()` without Ring 3 -> Ring 0 mode switching.
