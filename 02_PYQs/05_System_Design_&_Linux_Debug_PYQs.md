# Master PYQs: Module 5 — System Design & Production Linux Debugging

## Q1. Incident Debugging: Disk space shows 100% full (`df -h`), but `du -sh /*` accounts for only 60%. How do you fix this without restarting the server?
* **Asked in**: Amazon, Netflix, Meta, Google (SRE / Systems Role)
* **Difficulty**: FAANG Advanced

### 30-Second Interview Pitch
This discrepancy occurs when large log files were deleted (`rm`) while still actively held open by a running process. The directory entry is deleted, but the OS cannot free the inode and disk blocks until all open file descriptors referencing it are closed.

### Step-by-Step Resolution Command Sequence
```bash
# 1. Identify open deleted files using lsof
lsof +L1 | grep deleted

# Output example:
# mysqld  1234  mysql   5u   REG  253,0  42949672960  1048578 /var/log/mysql/queries.log (deleted)

# 2. Truncate the file via /proc filesystem without stopping the process!
echo > /proc/1234/fd/5

# 3. Verify disk space is reclaimed immediately
df -h /var
```

---

## Q2. Incident Debugging: Server Load Average is 15.0 on a 4-Core CPU, but CPU utilization is only 8%. What is happening?
* **Asked in**: Uber, Google, Stripe, Cloudflare
* **Difficulty**: FAANG Advanced

### Detailed Explanation
In Linux, **Load Average** measures the number of processes in the `TASK_RUNNING` (using or waiting for CPU) AND `TASK_UNINTERRUPTIBLE` (blocked waiting for I/O disk/network) states.

* **Diagnosis**: Low CPU utilization combined with a high Load Average indicates severe **I/O Bottleneck / Disk Stalling**. Processes are stuck waiting for disk reads/writes or NFS network storage.
* **Investigation Commands**:
```bash
# Check process states - look for processes in 'D' state (Uninterruptible Sleep)
ps aux | awk '$8 ~ /D/'

# Monitor I/O wait percentages and disk queue depths
iostat -xz 1
```

---

## Q3. How does the Linux Out-Of-Memory (OOM) Killer select victim processes? How do you protect critical services?
* **Asked in**: Amazon, Meta, Salesforce, ByteDance
* **Difficulty**: Medium to Hard

### Victim Selection Algorithm
When RAM + Swap is exhausted, the kernel invokes `badness()` (`mm/oom_kill.c`) to compute an `oom_score` (0–1000) for every process based on:
1. Percentage of physical RAM consumed by the process.
2. Root processes get a small score discount ($3\%$).
3. `oom_score_adj` value added from `/proc/[pid]/oom_score_adj`.

### How to Protect MySQL / Redis from OOM Killer
```bash
# Set oom_score_adj to -1000 (Never Kill)
echo -1000 > /proc/$(pgrep mysqld)/oom_score_adj
```
