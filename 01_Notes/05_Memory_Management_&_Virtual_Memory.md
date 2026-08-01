# Operating System Notes: Module 5 — Memory Management & Virtual Memory

## 1. Physical Memory Management & Allocation Strategies

Memory allocation maps process requests to physical RAM partitions.

### 1.1 Contiguous Allocation & Fragmentation
* **Internal Fragmentation**: Occurs when fixed-size memory blocks are allocated to a process that requires less space than the block size (unused memory *inside* the block).
* **External Fragmentation**: Occurs when total free memory is sufficient to satisfy a request, but the free memory is split across non-contiguous holes (*outside* allocated blocks).

```
Dynamic Partition Allocation Strategies:
+-------------------------------------------------------------------+
| Free Hole: 100 KB  | Free Hole: 500 KB  | Free Hole: 200 KB       |
+-------------------------------------------------------------------+
Allocation request of 150 KB:
- First Fit: Allocates in 500 KB hole (First hole large enough).
- Best Fit:  Allocates in 200 KB hole (Smallest hole large enough -> Leaves 50 KB hole).
- Worst Fit: Allocates in 500 KB hole (Largest hole -> Leaves 350 KB hole).
```

### 1.2 Kernel Buddy Allocator & Slab Allocator
* **Buddy System**: Divides memory into power-of-2 size frames ($2^k$). When a $64\text{ KB}$ block is freed, the allocator checks if its "buddy" block is free, merging them into a $128\text{ KB}$ frame.
* **Slab Allocator**: Eliminates internal fragmentation for small kernel objects (e.g., `task_struct`, `struct mm_struct`) by creating pre-allocated caches of fixed-size object slabs.

---

## 2. Virtual Memory Architecture & Paging

Virtual memory abstracts physical RAM, giving each process the illusion of an isolated, contiguous address space.

```
Virtual Address (CPU)                   Physical Address (RAM)
+------------------------+              +------------------------+
| Page Number | Offset   |              | Frame Number | Offset  |
|     (p)     |   (d)    |              |     (f)      |   (d)   |
+------+-----------------+              +------+-----------------+
       |                                       ^
       v                                       |
+----------------------------------------------+----+
| Page Table Entry (PTE) Lookup                     |
| Index p -> Frame Number f                         |
| Bits: [ P | R/W | U/S | Dirty | Accessed | PFN ]  |
+---------------------------------------------------+
```

### 2.1 Multi-Level Paging (x86_64 4-Level Paging)
To map a 64-bit address space ($48$-bit active virtual address) without incurring massive flat page table sizes:
$$48\text{-bit Virtual Address} = \text{PML4 }(9\text{ bits}) + \text{PDPT }(9\text{ bits}) + \text{PD }(9\text{ bits}) + \text{PT }(9\text{ bits}) + \text{Offset }(12\text{ bits})$$

```
CR3 Register ---> PML4 Table ---> PDPT Table ---> Page Directory ---> Page Table ---> Physical 4KB Page Frame
```

### 2.2 Translation Lookaside Buffer (TLB)
The **TLB** is an associative high-speed hardware cache on the CPU that stores recent Virtual-to-Physical address translations.

$$\text{Effective Access Time (EAT)} = (H \times T_{\text{TLB}}) + ((1 - H) \times (T_{\text{TLB}} + 2 \times T_{\text{RAM}}))$$
where $H$ = TLB Hit Ratio, $T_{\text{TLB}}$ = TLB Lookup Time, $T_{\text{RAM}}$ = Physical Memory Access Time.

---

## 3. Demand Paging & Page Fault Handling

```
Process CPU Execution
+--------------------+
| Read Address       |
+---------+----------+
          |
          v
    [ In TLB? ] ----(Yes)----> Compute Physical Address & Read RAM
          |
        (No)
          v
[ Page Table Present Bit == 1? ] ----(Yes)----> Update TLB & Read RAM
          |
        (No)
          v
TRAP: CPU Page Fault Exception (#PF)
          |
          v
Kernel Page Fault Handler:
1. Validate Virtual Address (Check VMA permissions).
2. Find Free Physical Frame in RAM (or run Page Replacement Algorithm if full).
3. Issue Disk Read to fetch Page from Storage Swap/File into Allocated Frame.
4. Update Page Table Entry: Set Frame Number, set Present Bit = 1.
5. Invalidate / Update TLB.
6. Return from Trap: CPU retries faulting instruction.
```

---

## 4. Page Replacement Algorithms & Worked Problems

Consider the reference string of page accesses:
$$\mathbf{7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1}$$
Frame Allocation Size = $3$ Frames.

---

### 4.1 First-In First-Out (FIFO) Page Replacement

| Ref | F1 | F2 | F3 | Fault? |
| :--- | :--- | :--- | :--- | :--- |
| **7** | 7 | - | - | **Fault (1)** |
| **0** | 7 | 0 | - | **Fault (2)** |
| **1** | 7 | 0 | 1 | **Fault (3)** |
| **2** | 2 | 0 | 1 | **Fault (4)** (Replaces 7) |
| **0** | 2 | 0 | 1 | Hit |
| **3** | 2 | 3 | 1 | **Fault (5)** (Replaces 0) |
| **0** | 2 | 3 | 0 | **Fault (6)** (Replaces 1) |
| **4** | 4 | 3 | 0 | **Fault (7)** (Replaces 2) |
| **2** | 4 | 2 | 0 | **Fault (8)** (Replaces 3) |
| **3** | 4 | 2 | 3 | **Fault (9)** (Replaces 0) |
| **0** | 0 | 2 | 3 | **Fault (10)** (Replaces 4) |
| **3** | 0 | 2 | 3 | Hit |
| **2** | 0 | 2 | 3 | Hit |
| **1** | 0 | 1 | 3 | **Fault (11)** (Replaces 2) |
| **2** | 0 | 1 | 2 | **Fault (12)** (Replaces 3) |
| **0** | 0 | 1 | 2 | Hit |
| **1** | 0 | 1 | 2 | Hit |
| **7** | 7 | 1 | 2 | **Fault (13)** (Replaces 0) |
| **0** | 7 | 0 | 2 | **Fault (14)** (Replaces 1) |
| **1** | 7 | 0 | 1 | **Fault (15)** (Replaces 2) |

* **Total FIFO Page Faults = 15**

> **Belady's Anomaly**: For FIFO, increasing physical frame allocation can sometimes *increase* the number of page faults (e.g., string `1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5` yields 9 faults with 3 frames, but 10 faults with 4 frames).

---

### 4.2 Least Recently Used (LRU) Page Replacement

| Ref | F1 | F2 | F3 | Fault? |
| :--- | :--- | :--- | :--- | :--- |
| **7** | 7 | - | - | **Fault (1)** |
| **0** | 7 | 0 | - | **Fault (2)** |
| **1** | 7 | 0 | 1 | **Fault (3)** |
| **2** | 2 | 0 | 1 | **Fault (4)** (7 least recent) |
| **0** | 2 | 0 | 1 | Hit |
| **3** | 2 | 0 | 3 | **Fault (5)** (1 least recent) |
| **0** | 2 | 0 | 3 | Hit |
| **4** | 4 | 0 | 3 | **Fault (6)** (2 least recent) |
| **2** | 4 | 0 | 2 | **Fault (7)** (3 least recent) |
| **3** | 4 | 3 | 2 | **Fault (8)** (0 least recent) |
| **0** | 0 | 3 | 2 | **Fault (9)** (4 least recent) |
| **3** | 0 | 3 | 2 | Hit |
| **2** | 0 | 3 | 2 | Hit |
| **1** | 1 | 3 | 2 | **Fault (10)** (0 least recent) |
| **2** | 1 | 3 | 2 | Hit |
| **0** | 1 | 0 | 2 | **Fault (11)** (3 least recent) |
| **1** | 1 | 0 | 2 | Hit |
| **7** | 1 | 0 | 7 | **Fault (12)** (2 least recent) |
| **0** | 1 | 0 | 7 | Hit |
| **1** | 1 | 0 | 7 | Hit |

* **Total LRU Page Faults = 12**

---

### 4.3 Optimal Page Replacement (OPT / Belady's OPT)
* Replaces the page that will not be used for the longest period of time in the future.
* **Total OPT Page Faults = 9** (Theoretical baseline).

---

## 5. Thrashing & Working Set Model

**Thrashing** occurs when a system spends more time paging (swapping pages in and out of disk) than executing actual process instructions.

```
CPU Utilization vs Percentage of Processes (Degree of Multiprogramming)
CPU Util
  100% |        /-----\
       |       /       \  <- Thrashing Threshold
       |      /         \
       |     /           \
    0% +---------------------------> Degree of Multiprogramming
```

### 5.1 Working Set Model
* Working Set $\Delta$: Parameter defining the working-set window (e.g., last 10,000 page references).
* $WSS_i$: Working set size of Process $P_i$ (total distinct pages referenced in window $\Delta$).
* Total demand $D = \sum WSS_i$.
* **If $D > \text{Total Available Physical RAM}$**: Thrashing occurs! The OS must suspend/swap out entire processes to restore CPU utilization.

---

## 6. Summary Checklist & Flash Cards

* **What is the difference between Paging and Segmentation?** Paging divides memory into fixed-size physical frames; Segmentation divides memory into logical variable-length code/data segments.
* **What is TLB Shootdown?** Multi-core event where one CPU modifying a page table must send Inter-Processor Interrupts (IPIs) to force all other cores to flush their local TLBs.
* **Why does LRU not suffer from Belady's Anomaly?** Because LRU belongs to the class of **Stack Algorithms** where the set of pages in $N$ frames is always a subset of pages in $N+1$ frames.
