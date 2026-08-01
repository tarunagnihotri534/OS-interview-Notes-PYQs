# Company-Wise PYQs: Goldman Sachs & Atlassian Enterprise Questions

## 🏢 Profile & Interview Focus
Goldman Sachs and Atlassian evaluate **Low-Latency OS Tuning**, **CPU Affinity (`taskset`)**, **NUMA (Non-Uniform Memory Access) Node Optimization**, **POSIX Shared Memory (`shm_open`)**, and **Lockless Ring Buffers**.

---

## Q1. What is NUMA (Non-Uniform Memory Access) and how do you optimize OS processes for NUMA Nodes?
* **Difficulty**: Hard

### Detailed Explanation
In multi-socket server architectures, physical RAM is divided into NUMA nodes attached locally to specific CPU sockets.
* **Local Access**: CPU accessing its local NUMA RAM node (~50 ns).
* **Remote Access**: CPU accessing RAM attached to a remote socket over QPI/UPI bus (~120 ns latency penalty).
* **OS Optimization Command**:
```bash
# Pin high-frequency trading binary to CPU socket 0 and local NUMA node 0
numactl --cpunodebind=0 --membind=0 ./hft_trading_engine
```
