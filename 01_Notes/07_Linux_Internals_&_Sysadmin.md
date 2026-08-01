# Operating System Notes: Module 7 — Linux Internals, Async I/O & Troubleshooting

## 1. Filesystem Hierarchy Standard (FHS)

```
/ (Root Directory)
├── bin -> usr/bin        # Essential user command binaries (ls, bash, cp)
├── boot                  # Bootloader files, GRUB config, vmlinuz kernel, initramfs
├── dev                   # Device files (e.g. /dev/sda, /dev/nvme0n1, /dev/null, /dev/zero)
├── etc                   # System-wide configuration files (etc/passwd, etc/fstab, etc/systemd)
├── home                  # User home directories (/home/alice)
├── lib -> usr/lib        # Shared system libraries (libc.so) required by binaries in /bin
├── proc                  # Virtual filesystem exporting Kernel runtime process data
├── sys                   # Virtual filesystem exporting Kernel hardware driver state
├── tmp                   # Temporary files (cleared on reboot)
├── usr                   # Secondary hierarchy for user read-only applications (/usr/bin, /usr/lib)
└── var                   # Variable data files (logs in /var/log, databases in /var/lib)
```

---

## 2. Linux Memory Internals & Tuning

### 2.1 Out-Of-Memory (OOM) Killer
When physical RAM and swap are exhausted, the Linux kernel invokes the `OOM Killer` (`mm/oom_kill.c`).

* **Victim Selection (`oom_score`)**:
  * Each process is assigned an `oom_score` (0–1000) proportional to its RAM consumption.
  * Adjusting bias via `/proc/[pid]/oom_score_adj`:
    * Range: `-1000` (Never kill, e.g., sshd, database master) to `+1000` (Kill first).

```bash
# Protect critical process (PID 1234) from OOM Killer
echo -1000 > /proc/1234/oom_score_adj
```

### 2.2 HugePages (Transparent HugePages vs. Static HugePages)
* **Standard Page Size**: $4\text{ KB}$.
* **HugePage Sizes**: $2\text{ MB}$ or $1\text{ GB}$.
* **Benefit**: Reduces the number of page table entries by $512\times$, massively increasing TLB hit ratio for large memory workloads (PostgreSQL, Redis, Oracle DB).

---

## 3. High-Performance I/O Multiplexing: `select` vs. `poll` vs. `epoll` vs. `io_uring`

```
Evolution of Linux I/O Multiplexing Architecture:
+-----------------------------------------------------------------------+
|  select()   : O(N) linear array scan. Limited to 1024 File Descriptors|
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
|  poll()     : O(N) linear array scan. Removes 1024 FD hard limit      |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
|  epoll()    : O(1) event-driven notification. Uses Red-Black Tree    |
|               in kernel + Ready List (epoll_create, epoll_wait)       |
+-----------------------------------------------------------------------+
                                   |
                                   v
+-----------------------------------------------------------------------+
|  io_uring   : Asynchronous Zero-Syscall Ring Buffers (Linux 5.1+)     |
|               Submission Queue (SQ) + Completion Queue (CQ)           |
+-----------------------------------------------------------------------+
```

### 3.1 Detailed I/O Multiplexing Comparison

| Characteristic | `select()` | `poll()` | `epoll()` | `io_uring` |
| :--- | :--- | :--- | :--- | :--- |
| **Search Complexity** | $O(N)$ | $O(N)$ | $O(1)$ | $O(1)$ |
| **Max Descriptors** | 1024 (`FD_SETSIZE`) | Unlimited (`nfds_t`) | Unlimited | Unlimited |
| **Kernel-User Copy** | Copies entire `fd_set` every call | Copies `pollfd` array every call | Copies **only active event list** | **Zero Copy** (Shared Ring Buffer) |
| **Syscall Frequency** | 1 syscall per poll cycle | 1 syscall per poll cycle | 1 syscall per ready batch | **0 syscalls** in Polled Mode |

---

## 4. Systematic Production Debugging Toolkit

### 4.1 Scenario 1: Disk Space at 100% but `du -sh /*` Shows Only 60% Used
* **Cause**: Deleted files that are still held open by running processes. The filesystem unlinks the filename from directory, but inode and disk blocks cannot be freed until file descriptor is closed!
* **Resolution Steps**:
```bash
# 1. Identify deleted open files using lsof
lsof +L1

# 2. Restart or reload the owning service to release the file descriptor
systemctl reload nginx
```

### 4.2 Scenario 2: Tracing Application Failures with `strace`
```bash
# Trace system calls executed by a crashing binary (PID 4321)
strace -f -e trace=file,network -p 4321

# Count time spent per syscall
strace -c ./my_app
```

---

## 5. Summary Checklist & Flash Cards

* **Difference between Hard Link and Bind Mount?** A hard link shares an inode within the same filesystem; a bind mount mirrors an entire directory tree across mount points or filesystems.
* **Why is `epoll` preferred over `select` for web servers like Nginx?** `epoll` avoids $O(N)$ file descriptor array copies and scans; kernel notifies active ready connections in $O(1)$ time.
* **How to view kernel crash ring buffer logs?** Use `dmesg -T` or `journalctl -k`.
