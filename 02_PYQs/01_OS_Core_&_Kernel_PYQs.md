# Master PYQs: Module 1 — OS Core & Kernel Architecture

## Q1. What happens under the hood when a program calls `fork()` followed by `execve()`?
* **Asked in**: Google, Amazon, Microsoft, Meta
* **Difficulty**: FAANG Advanced

### 30-Second Interview Pitch
`fork()` creates a copy of the parent process using Copy-on-Write (COW), duplicating page tables and file descriptors without copying physical RAM. `execve()` then replaces the process address space entirely by releasing old page tables, loading a new ELF executable image into RAM, and jumping to its entry point (`_start`).

### Detailed Technical Explanation
1. **`fork()` Phase**:
   * Kernel allocates a new PCB (`struct task_struct`) and assigns a new PID.
   * Copies parent process context (file descriptor table, signal handlers, environment).
   * Maps child page tables to the **same physical RAM page frames** as parent, marking entries **Read-Only**.
   * Sets return value: $0$ in child process, child PID in parent process.
2. **`execve()` Phase**:
   * Releases existing virtual address space mappings (text, data, bss, heap, stack).
   * Parses the target binary ELF header (verifies magic bytes `\x7fELF`, architecture, permissions).
   * Maps binary segments (`PT_LOAD`) into virtual address space.
   * Sets up new user stack containing command line arguments (`argv`) and environment variables (`envp`).
   * Sets CPU Program Counter (`RIP`) to ELF entry point address (`_start` in glibc).

---

## Q2. Monolithic Kernel vs. Microkernel: Architecture & Trade-offs
* **Asked in**: Apple, Qualcomm, Microsoft, VMware
* **Difficulty**: Medium

### 30-Second Interview Pitch
Monolithic kernels run all OS services (filesystem, IPC, drivers, networking) in Ring 0 for maximum speed via direct function calls, but a single driver crash takes down the OS. Microkernels run only minimal services (IPC, scheduling, low-level memory) in Ring 0 and isolate drivers/filesystems in Ring 3 user processes, sacrificing IPC performance for rock-solid reliability.

### Architectural Comparison Matrix

| Dimension | Monolithic Kernel (Linux) | Microkernel (seL4 / QNX) |
| :--- | :--- | :--- |
| **Ring 0 Components** | Scheduler, VFS, Memory Mgmt, Drivers, Network | Scheduler, Low-level IPC, Page table primitives |
| **Fault Isolation** | Poor (Driver bug = Kernel Panic) | Excellent (Driver crash = Restart process) |
| **Performance** | Maximum (In-memory C function calls) | Overhead (Message passing requires mode switches) |
| **Code Base Size** | 30+ Million Lines of Code | ~10,000 Lines of Code (Formally Verified) |

---

## Q3. Explain the exact difference between Hardware Interrupts, Software Traps, and CPU Exceptions.
* **Asked in**: Qualcomm, Nvidia, Cisco, Intel
* **Difficulty**: Medium

### Detailed Answer
* **Hardware Interrupt (Asynchronous)**: Initiated by external peripheral hardware (keyboard, NIC, disk controller) pushing a signal on the IRQ line to the APIC. Can interrupt CPU execution at *any arbitrary instruction boundary*.
* **Software Trap (Synchronous)**: Triggered intentionally by software executing a specific trap instruction (`SYSCALL`, `INT 0x80`). Used by applications to request kernel services.
* **CPU Exception (Synchronous)**: Generated internally by the CPU execution unit when an error condition is encountered during instruction decoding/execution (Divide-by-zero, Page Fault `#PF`, Invalid Opcode).

---

## Q4. What is Linux `vDSO` (virtual Dynamic Shared Object) and how does it optimize system calls?
* **Asked in**: Meta, Netflix, Google, Uber
* **Difficulty**: FAANG Advanced

### 30-Second Interview Pitch
`vDSO` is a small kernel-provided shared library mapped directly into every user process's address space. It allows read-only system call queries (like `clock_gettime()` or `gettimeofday()`) to run entirely in Ring 3 user space by reading kernel-updated shared memory pages without executing expensive CPU privilege switches.

---

## Q5. Walk through the complete OS boot process from physical reset to `systemd` PID 1.
* **Asked in**: Amazon, Red Hat, Cisco, VMware
* **Difficulty**: Medium to Hard

### Step-by-Step Execution Sequence
1. **Power-On & POST**: Power supply signals `POWER_GOOD` -> CPU RESET line cleared -> CPU loads reset vector `0xFFFFFFF0` -> BIOS/UEFI executes Power-On Self-Test (POST).
2. **Bootloader Selection**: UEFI reads EFI System Partition (ESP) -> executes GRUB bootloader (`grubx64.efi`).
3. **Kernel Loading**: GRUB loads `vmlinuz` kernel image and `initramfs` disk image into RAM -> passes kernel command line params -> jumps to kernel entry point.
4. **Kernel Init**: Kernel unpacks `initramfs`, initializes CPU page tables, device drivers, IDT table, memory allocators -> mounts real root filesystem (`/`).
5. **Userspace Launch**: Kernel spawns `/sbin/init` (or `systemd`) as **PID 1** in Ring 3 user space.

---

## Q6. What are Meltdown and Spectre? How does Kernel Page Table Isolation (KPTI) resolve Meltdown?
* **Asked in**: Intel, AMD, Google, Apple
* **Difficulty**: FAANG Advanced

### Detailed Answer
Meltdown exploited out-of-order CPU speculative execution to read kernel memory addresses from user mode. Even though the CPU eventually triggered a Page Fault, speculative instructions fetched data into CPU L1 Cache before the fault was committed, allowing attackers to leak kernel memory via cache timing (Flush+Reload).

**KPTI Solution**: Splitting user page tables and kernel page tables completely. When running in User Mode (Ring 3), kernel memory addresses are unmapped from the page table entirely (except minimal syscall stubs), preventing speculative reads of kernel space.
