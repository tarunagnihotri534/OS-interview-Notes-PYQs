# Company-Wise PYQs: Google OS Interview Questions

## 🏢 Company Profile & Interview Focus
Google systems and backend interviews place heavy emphasis on **Linux Kernel Internals**, **Concurrency & Lock-Free Data Structures**, **Containers/Namespaces (Cgroups v2)**, **eBPF Tracing**, and **High-Scalability I/O (`epoll`)**.

---

## Q1. How do Linux Namespaces and Cgroups enable Container Isolation (Docker / Kubernetes / Borg)?
* **Difficulty**: FAANG Advanced
* **Topics**: Linux Kernel, Virtualization, Containers

### 30-Second Interview Pitch
Namespaces provide **isolation** by restricting what a process can *see* (PID, network interfaces, mount points, hostnames). Control Groups (Cgroups) provide **resource limiting** by restricting how much hardware a process group can *consume* (CPU shares, RAM limits, I/O bandwidth).

### Deep Technical Explanation
* **Linux Namespaces** (7 Key Types):
  1. `pid`: Isolates process tree (Process inside container sees itself as PID 1).
  2. `net`: Isolates network stacks (own loopback, IP addresses, routing tables, `iptables`).
  3. `mnt`: Isolates filesystem mount points (`chroot` / `pivot_root`).
  4. `ipc`: Isolates System V IPC and POSIX message queues.
  5. `uts`: Isolates hostname and NIS domain name.
  6. `user`: Maps container root user (UID 0) to an unprivileged user UID on the host.
  7. `cgroup`: Isolates cgroup root directory views.
* **Cgroups v2 (Control Groups)**:
  * Organizes processes into a unified hierarchical tree.
  * Controllers: `cpu.max` (bandwidth limits via CFS scheduler), `memory.max` (hard RAM limit triggering OOM), `io.max` (disk IOPS/BPS limits).

---

## Q2. What is eBPF (Extended Berkeley Packet Filter) and how does it execute safely inside the Linux Kernel?
* **Difficulty**: FAANG Advanced
* **Topics**: Linux Kernel, Security, Tracing

### Detailed Answer
eBPF allows developers to run sandboxed byte-code programs inside the Linux kernel dynamically without modifying kernel source code or loading kernel modules (`kmods`).

* **Safety Verification**: Before loading, the kernel **eBPF Verifier** checks the code for:
  1. No uninitialized memory accesses.
  2. Guaranteed termination (no infinite loops; bounded loops only).
  3. Strict register type checks and array out-of-bound checks.
* **JIT Compilation**: Converts verified eBPF bytecode into native x86_64/ARM machine instructions for near-zero execution overhead.
* **Use Cases at Google**: High-performance packet filtering (Cilium), dynamic kernel tracing (`kprobes`/`uprobes`), and container security monitoring.

---

## Q3. Explain the Linux EEVDF (Earliest Eligible Virtual Deadline First) CPU Scheduler.
* **Difficulty**: FAANG Advanced
* **Topics**: CPU Scheduling, Linux Kernel 6.6+

### Detailed Answer
EEVDF replaced the Completely Fair Scheduler (CFS) in Linux Kernel 6.6+.
* **Problem with CFS**: CFS prioritized process fairness based on virtual runtime (`vruntime`), but struggled with latency-sensitive tasks (e.g., audio, UI rendering) that need immediate CPU bursts.
* **EEVDF Core Logic**:
  1. **Eligibility**: A process is eligible if its virtual runtime is not ahead of the average system virtual time.
  2. **Virtual Deadline**: Each process specifies a latency request (`lag`). EEVDF calculates a virtual deadline:
     $$\text{Virtual Deadline} = \text{Virtual Time} + \frac{\text{Slice}}{\text{Weight}}$$
  3. **Selection**: Among all *eligible* processes, EEVDF dispatches the process with the **earliest virtual deadline**.
