# Company-Wise PYQs: Uber & Netflix Systems Questions

## 🏢 Profile & Interview Focus
Uber and Netflix engineering interviews focus on **High-Throughput Socket I/O Tuning**, **Zero-Copy Data Transfer (`sendfile`, `splice`)**, **TCP Buffer Optimization**, **File Descriptor Limits**, and **Low-Latency IPC**.

---

## Q1. How does Zero-Copy (`sendfile` / `splice`) work, and why does Netflix use it for high-throughput video streaming?
* **Difficulty**: FAANG Advanced

### Traditional File-to-Socket Transfer (4 Context Switches, 4 Data Copies)
1. Application calls `read()`: Disk -> Page Cache (DMA) -> User Buffer (CPU copy).
2. Application calls `write()`: User Buffer -> Socket Buffer (CPU copy) -> NIC (DMA).

### Zero-Copy `sendfile()` Transfer (2 Context Switches, 2 DMA Copies, 0 CPU Copies!)
```
Disk Drive ---> [DMA Copy] ---> Kernel Page Cache ---> [DMA Copy] ---> NIC Hardware
                                      |
                         (Passes Descriptor Pointer Only)
```

* **Result**: Eliminates CPU memory copy overhead completely and frees CPU cycles for handling hundreds of thousands of concurrent video streams per edge proxy server.
