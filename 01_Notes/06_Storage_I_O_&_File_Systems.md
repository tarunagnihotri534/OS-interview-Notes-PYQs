# Operating System Notes: Module 6 — Storage, I/O & File Systems

## 1. Virtual File System (VFS) Architecture

The **Virtual File System (VFS)** is a kernel abstraction layer allowing client applications to access different physical filesystems (ext4, XFS, NTFS, NFS) via uniform POSIX system calls (`read`, `write`, `open`).

```
User Space Application
       |
       v
System Call Interface (open, read, write)
       |
       v
Virtual File System (VFS Layer - struct inode, struct dentry, struct file)
       |
       +-----------------------+-----------------------+
       |                       |                       |
       v                       v                       v
Ext4 File System        XFS File System         NFS (Network File System)
       |                       |                       |
       +-----------------------+-----------------------+
                               |
                               v
                     Block Device Driver
                               |
                               v
                    NVMe / SATA SSD / Hard Disk
```

---

## 2. Inode Structure & File Descriptor Tables

### 2.1 Inode Architecture (Indexed Allocation)
An **Inode (Index Node)** is a filesystem structure storing metadata for a file (size, permissions, owner, timestamps, block pointers).

```
Inode Block Pointer Structure:
+------------------------------------+
| Inode Attributes (mode, size, uid) |
+------------------------------------+
| Direct Block 0 -------------> Disk Block 102
| Direct Block 1 -------------> Disk Block 103
| ...                                       
| Direct Block 11 ------------> Disk Block 113
| Single Indirect Pointer ----> Block of Block Pointers (256 Pointers)
| Double Indirect Pointer ----> Block of Single Indirect Block Pointers
| Triple Indirect Pointer ----> Block of Double Indirect Block Pointers
+------------------------------------+
```

### 2.2 File System Table Relationships

```
User Process FD Table               System-Wide Open File Table            Inode Table
+-----------------------+           +-----------------------------+        +----------------------+
| FD 0 (stdin)          |           | File Struct 1               |        | Inode 4082           |
| FD 1 (stdout)         |           | - File Offset (e.g. 1024)   |------->| - File Size: 40 KB   |
| FD 2 (stderr)         |           | - Open Flags (O_RDWR)       |        | - Permissions: 0644  |
| FD 3 -----------------+---------->| - Inode Pointer ------------+        | - Block Pointers...  |
+-----------------------+           +-----------------------------+        +----------------------+
```

---

## 3. Disk Scheduling Algorithms & Solved Problems

Disk Request Queue: $\mathbf{98, 183, 37, 122, 14, 124, 65, 67}$
Initial Disk Read/Write Head Position = $\mathbf{53}$ (Track range: $0–199$).

---

### 3.1 Solved Calculations

#### 1. FCFS (First-Come First-Served)
Order: $53 \to 98 \to 183 \to 37 \to 122 \to 14 \to 124 \to 65 \to 67$
$$\text{Head Movement} = |98-53| + |183-98| + |37-183| + |122-37| + |14-122| + |124-14| + |65-124| + |67-65|$$
$$= 45 + 85 + 146 + 85 + 108 + 110 + 59 + 2 = \mathbf{640\text{ tracks}}$$

---

#### 2. SSTF (Shortest Seek Time First)
Order: $53 \to 65 \to 67 \to 37 \to 14 \to 98 \to 122 \to 124 \to 183$
$$\text{Head Movement} = |65-53| + |67-65| + |37-67| + |14-37| + |98-14| + |122-98| + |124-122| + |183-124|$$
$$= 12 + 2 + 30 + 23 + 84 + 24 + 2 + 59 = \mathbf{236\text{ tracks}}$$

---

#### 3. SCAN (Elevator Algorithm - Moving towards 0)
Order: $53 \to 37 \to 14 \to \mathbf{0} \to 65 \to 67 \to 98 \to 122 \to 124 \to 183$
$$\text{Head Movement} = (53 - 0) + (183 - 0) = 53 + 183 = \mathbf{236\text{ tracks}}$$

---

#### 4. C-SCAN (Circular SCAN - Moving towards 199)
Order: $53 \to 65 \to 67 \to 98 \to 122 \to 124 \to 183 \to \mathbf{199} \to \mathbf{0} \to 14 \to 37$
$$\text{Head Movement} = (199 - 53) + (199 - 0) + (37 - 0) = 146 + 199 + 37 = \mathbf{382\text{ tracks}}$$

---

## 4. Hardware I/O Architecture & DMA

### 4.1 Programmed I/O vs. Interrupt-Driven I/O vs. DMA

```
Programmed I/O:      CPU constantly polls device status register in tight loop.
Interrupt-Driven:    CPU issues command, switches process, wakes up when device fires IRQ.
Direct Memory Access:CPU programs DMA controller (src, dest, len), DMA transfers blocks directly between device and RAM via Bus Mastering, fires 1 IRQ when entire transfer completes.
```

```
DMA Transfer Lifecycle:
1. CPU writes DMA Command (RAM Address, Disk Address, Byte Count) to DMA Controller.
2. DMA Controller initiates transfer with Disk Controller via system bus.
3. Disk Controller streams data bytes directly to RAM (Cycle Stealing mode).
4. When transfer finishes, DMA Controller sends Hardware Interrupt (IRQ) to CPU.
```

### 4.2 Memory-Mapped I/O (MMIO) vs. Port-Mapped I/O (PMIO)
* **Port-Mapped I/O (PMIO)**: Special CPU instructions (`in`, `out` on x86) access a separate I/O port address space.
* **Memory-Mapped I/O (MMIO)**: Device control registers are mapped directly into physical memory address space; standard memory pointers (`mov`, `load`, `store`) read/write device registers.

---

## 5. Summary Checklist & Flash Cards

* **What happens if a filesystem runs out of inodes?** No new files can be created, even if gigabytes of free disk storage space remain!
* **Why do SSDs use `none` or `mq-deadline` schedulers instead of `SCAN`?** SSDs have zero rotational latency and mechanical seek overhead; elevator scheduling adds unnecessary CPU queue overhead.
* **What is Hard Link vs Soft Link (Symlink)?** A hard link is an additional directory entry pointing directly to an existing inode number; a soft link is a distinct file containing a path string to another file.
