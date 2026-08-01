# Master PYQs: Module 3 — Memory & Virtual Memory

## Q1. Walk through every step of a Page Fault handling lifecycle in Linux.
* **Asked in**: Google, Amazon, Meta, Microsoft, Apple
* **Difficulty**: FAANG Advanced

### Detailed Step-by-Step Answer
1. **CPU Memory Reference**: CPU attempts to execute an instruction referencing virtual address $VA$.
2. **TLB Lookup Miss**: TLB checks tag array; $VA$ is not present (TLB Miss).
3. **Page Table Walk**: MMU traverses Page Tables (PML4 -> PDPT -> PD -> PT).
4. **PTE Flag Inspection**: MMU finds the Page Table Entry (PTE) for $VA$, but the **Present Bit is 0**.
5. **CPU Exception Generation**: CPU hardware pauses execution and fires **Page Fault Exception (`#PF`, Vector 14)**, saving faulting address into `CR2` register.
6. **Kernel `#PF` Handler Invocation**: CPU switches to Ring 0 kernel stack, jumping to `do_page_fault()`.
7. **Address & VMA Validation**: Kernel checks if address falls within a valid Virtual Memory Area (`vma_struct`) for the process. If invalid -> sends `SIGSEGV` (Segmentation Fault).
8. **Physical Frame Allocation**: Kernel's Buddy Allocator allocates a free physical RAM page frame $PFN$. (If RAM is full -> runs Page Replacement Algorithm).
9. **Storage Swap / File Read**: If page was swapped out or belongs to an executable file on disk, kernel issues I/O request to disk controller to fetch page data into $PFN$. Process state set to `TASK_UNINTERRUPTIBLE`.
10. **Page Table Update**: Once I/O completes, kernel updates PTE: writes $PFN$, sets Present Bit = 1, sets Dirty/Accessed bits.
11. **TLB Invalidation**: Updates local TLB entry.
12. **Instruction Retry**: Kernel returns from interrupt via `IRET`. CPU retries the exact instruction that faulted.

---

## Q2. Prove why FIFO suffers from Belady's Anomaly while LRU does not.
* **Asked in**: Google, Microsoft, Amazon
* **Difficulty**: Medium to Hard

### Proof Summary
Belady's Anomaly occurs when increasing the number of allocated physical frame buffers causes an increase in total page faults.

* **Stack Algorithms Property**: An algorithm belongs to the class of **Stack Algorithms** if the set of pages loaded in $N$ memory frames is guaranteed to be a strict subset of the set of pages loaded in $N+1$ memory frames at any step $t$:
$$S(N, t) \subseteq S(N+1, t)$$
* **LRU & OPT**: Both LRU and Optimal algorithms satisfy the Stack Property. Because $S(N, t) \subseteq S(N+1, t)$, increasing frame allocation size can **never** evict a page that would have been retained with fewer frames. Therefore, LRU and OPT **never** exhibit Belady's Anomaly.
* **FIFO**: FIFO does not satisfy the stack property because its eviction criteria depends strictly on arrival order rather than locality or future access.

---

## Q3. What is Thrashing? How does an OS use the Working Set Model to prevent it?
* **Asked in**: Meta, Amazon, VMware
* **Difficulty**: Hard

### Detailed Explanation
Thrashing is a severe system state where physical RAM is oversubscribed, causing processes to spend more time executing page swaps than actual instructions.

**Working Set Model Solution**:
* Defines working-set window $\Delta$ (e.g., last 10,000 page references).
* Computes Working Set Size $WSS_i$ for each process $P_i$.
* Total RAM Demand $D = \sum WSS_i$.
* **Admission Control**: If $D > \text{Total Physical Memory Frames } M$, the OS scheduler suspends (swaps out) an entire process to disk, freeing its frames for remaining processes until $D \le M$.
