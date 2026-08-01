# Company-Wise PYQs: Microsoft OS Interview Questions

## 🏢 Company Profile & Interview Focus
Microsoft interviews evaluate **Windows Architecture vs Linux Architecture**, **Windows NT Kernel (HAL, Object Manager, Executive)**, **Windows I/O Request Packets (IRPs)**, **Thread Scheduling & Quantum Adjustments**, and **Memory Compression**.

---

## Q1. Windows NT Kernel Architecture vs. Linux Kernel Architecture
* **Difficulty**: Medium to Hard

### Architectural Comparison Matrix

| Feature | Windows NT Architecture | Linux Architecture |
| :--- | :--- | :--- |
| **Kernel Classification** | Hybrid Kernel (`ntoskrnl.exe`) | Monolithic Kernel (`vmlinuz`) |
| **Hardware Abstraction** | HAL (Hardware Abstraction Layer `.dll`) | Inline architecture drivers in kernel |
| **Object System** | Unified Object Manager (Handles, Access Masks) | VFS Inodes & File Descriptors |
| **I/O Subsystem** | Packet-driven asynchronous IRPs | Synchronous & Asynchronous File Descriptors |
| **GUI Subsystem** | Integrated in kernel (`win32k.sys`) | Isolated User-Space (X11 / Wayland) |

---

## Q2. How does Windows manage Thread Priorities and Quantum Adjustments?
* **Difficulty**: Medium

### Detailed Explanation
* **Priority Levels**: Windows defines 32 thread priorities (0–31):
  * Priority 0: Zero-Page Thread (Clears RAM pages).
  * Priorities 1–15: Dynamic Priority Range (User applications).
  * Priorities 16–31: Real-Time Priority Range (Kernel tasks, audio drivers).
* **Dynamic Quantum Adjustment**:
  * Foreground window threads receive a **Quantum Boost** (longer CPU time slices) to keep desktop UI responsive.
  * When a thread wakes up from an I/O wait, its priority is temporarily boosted based on device type (e.g., sound driver gets +8 priority boost, disk gets +1).
