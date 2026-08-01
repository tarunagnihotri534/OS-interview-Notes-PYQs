# Company-Wise PYQs: Meta OS Interview Questions

## 🏢 Company Profile & Interview Focus
Meta systems interviews emphasize **High-Performance Async I/O (`io_uring`)**, **PSI (Pressure Stall Information)**, **Cgroup v2 Resource Management**, **Transparent HugePages Tuning**, and **Multi-Threaded Lock-Free C++ Concurrency**.

---

## Q1. How does `io_uring` achieve Zero-Syscall Asynchronous I/O, and how does it compare to `epoll`?
* **Difficulty**: FAANG Advanced

### 30-Second Interview Pitch
`io_uring` uses two ring buffers shared directly between user space and kernel space: a **Submission Queue (SQ)** and a **Completion Queue (CQ)**. Applications push I/O operations into the SQ and read completed results from the CQ without issuing CPU system calls for each request.

```
User Space Process                                   Linux Kernel
+-------------------+                               +-------------------+
| Submission Queue  |===(Lockless Ring Buffer)=====>| Kernel I/O Worker |
| (SQ Ring)         |                               | Threads           |
+-------------------+                               +-------------------+
| Completion Queue  |<===(Lockless Ring Buffer)=====| Completes Disk /  |
| (CQ Ring)         |                               | Network Requests  |
+-------------------+                               +-------------------+
```

---

## Q2. What is Linux PSI (Pressure Stall Information) and how does Meta use it for automated server health management?
* **Difficulty**: FAANG Advanced

### Detailed Answer
PSI quantifies system resource shortages across CPU, Memory, and I/O.
* Metrics exported via `/proc/pressure/{cpu,memory,io}`:
  * `some`: Percentage of time some tasks were stalled waiting for a resource.
  * `full`: Percentage of time **all** non-idle tasks were stalled waiting for a resource.
* **Meta Application**: Meta's daemon `oomd` monitors `/proc/pressure/memory`. If `some` memory pressure exceeds 20% over 10 seconds, `oomd` proactively terminates low-priority cgroups before the system enters a thrashing lockup!
