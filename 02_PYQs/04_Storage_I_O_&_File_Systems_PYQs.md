# Master PYQs: Module 4 — Storage, I/O & File Systems

## Q1. What is an Inode? What happens if a Linux filesystem runs out of Inodes even though 50% disk capacity is free?
* **Asked in**: Amazon (AWS), Red Hat, Netflix, Uber
* **Difficulty**: Medium

### 30-Second Interview Pitch
An Inode (Index Node) is a fixed-size kernel data structure on disk storing metadata (file size, owner, permissions, block pointers) for a single file/directory. If all pre-allocated inodes on a filesystem are exhausted (e.g., millions of tiny 1-byte files created), the filesystem will throw `No space left on device` errors when creating new files, even if gigabytes of storage space remain free!

---

## Q2. Detailed Comparison: `select()`, `poll()`, `epoll()`, and `io_uring`
* **Asked in**: Meta, Google, Uber, Netflix
* **Difficulty**: FAANG Advanced

### In-Depth Architectural Comparison

```
+------------------------------------------------------------------------------------+
| Feature        | select()             | poll()             | epoll()               |
+----------------+----------------------+--------------------+-----------------------+
| Data Structure | Bitmasks (FD_SET)    | Array of pollfd    | Red-Black Tree + List |
| Complexity     | O(N) linear scan     | O(N) linear scan   | O(1) event callback   |
| Max FDs        | 1024 (FD_SETSIZE)    | Unlimited          | Unlimited             |
| Memory Copy    | Copy mask every call | Copy array on call | Zero copy event list  |
+------------------------------------------------------------------------------------+
```

---

## Q3. How does Direct Memory Access (DMA) work? Explain Cycle Stealing and Cache Coherency issues.
* **Asked in**: Nvidia, Qualcomm, Apple, Intel
* **Difficulty**: FAANG Advanced

### Detailed Answer
DMA transfers data directly between high-speed peripheral devices (NICs, NVMe SSDs) and physical RAM without consuming CPU cycles for every byte transferred.

* **Cycle Stealing**: The DMA Controller temporarily takes control of the memory bus from the CPU for one bus cycle to transfer a data word, pausing CPU memory access briefly.
* **Cache Coherency Issue**: If the DMA Controller writes new data directly into physical RAM, but the CPU holds stale cached copies of those memory lines in its L1/L2 cache, the CPU will read invalid stale data.
* **Resolution**: Hardware Bus Snooping (MESI protocol) or explicit Kernel DMA Buffer Flushing (`dma_sync_single_for_cpu()`).
