# Company-Wise PYQs: Amazon OS Interview Questions

## 🏢 Company Profile & Interview Focus
Amazon / AWS systems interviews focus heavily on **Hypervisors (KVM vs Xen, AWS Nitro)**, **Virtual Machine Memory Overcommit**, **Storage I/O Bottlenecks (EBS, NVMe)**, **High-Concurrency Thread Pools**, and **Distributed Systems OS Tuning**.

---

## Q1. AWS Nitro Architecture vs. Traditional Hypervisors: How does Nitro offload OS virtualization?
* **Difficulty**: FAANG Advanced
* **Topics**: Hypervisors, Virtualization, Hardware Offloading

### 30-Second Interview Pitch
Traditional hypervisors (like Xen or QEMU) run I/O virtualization (network, storage, security) in software on the host CPU, consuming up to 30% of system performance. AWS Nitro offloads networking, storage, and management onto dedicated PCIe hardware cards (Nitro Cards), leaving nearly 100% of host CPU and RAM for customer Virtual Machines.

### Deep Technical Comparison

```
Traditional Hypervisor (Xen/KVM Software)     AWS Nitro Architecture
+------------------------------------+        +------------------------------------+
|  Guest VMs (EC2 Customer Instances)|        |  Guest VMs (100% Host CPU & RAM)  |
+------------------------------------+        +------------------------------------+
|  Hypervisor Software (QEMU/Xen)    |        |  Minimal KVM Core (Ring 0)         |
|  - Virtual Network Driver          |        +====================================+
|  - Virtual Storage Driver          |        | PCIe Offload Hardware (Nitro Cards)|
|  - Security & Monitoring           |        | - Nitro Card for VPC Networking    |
+------------------------------------+        | - Nitro Card for EBS Storage       |
|  Host CPU & Physical RAM           |        | - Nitro Security Chip              |
+------------------------------------+        +------------------------------------+
```

---

## Q2. What is Memory Overcommit in Cloud Virtualization, and how does the OS handle memory pressure?
* **Difficulty**: Hard
* **Topics**: Virtual Memory, Hypervisors, Swapping

### Detailed Answer
* **Memory Overcommit**: Hypervisors allocate more total virtual RAM to guest VMs than the physical RAM installed on the host hardware, relying on the fact that not all VMs use 100% of allocated memory simultaneously.
* **Reclaim Mechanisms**:
  1. **Memory Ballooning (`virtio-balloon`)**: The hypervisor instructs a guest OS kernel module to allocate physical pages inside the guest, returning those physical frames to the hypervisor host pool.
  2. **Page Sharing (KSM - Kernel Samepage Merging)**: The host kernel scans RAM for identical physical pages across VMs and merges them into a single Copy-on-Write (COW) page.
  3. **Host Swap**: Swaps cold guest memory pages to NVMe storage.

---

## Q3. How do you tune Linux Kernel parameters for high-throughput AWS EBS Storage I/O?
* **Difficulty**: Hard
* **Topics**: Storage I/O, Linux Sysctl Tuning

### Essential Tuning Configurations
```bash
# 1. Increase read-ahead buffer size for block device /dev/nvme0n1
blockdev --setra 4096 /dev/nvme0n1

# 2. Switch I/O Scheduler to none (bypasses elevator for NVMe drives)
echo none > /sys/block/nvme0n1/queue/scheduler

# 3. Increase max open file descriptors in /etc/security/limits.conf
* soft nofile 1048576
* hard nofile 1048576

# 4. Tune dirty page flush ratios in sysctl.conf
sysctl -w vm.dirty_background_ratio=5
sysctl -w vm.dirty_ratio=10
```
