# Company-Wise PYQs: Qualcomm & Nvidia Hardware/Embedded OS

## 🏢 Profile & Interview Focus
Qualcomm and Nvidia systems interviews focus on **Embedded RTOS (FreeRTOS, VxWorks)**, **Hardware Interrupt Handling (ISR vs Tasklets/Bottom-Halves)**, **Direct Memory Access (DMA)**, **Memory-Mapped I/O (MMIO)**, and **Cache Line Invalidations**.

---

## Q1. Interrupt Service Routine (ISR) Top-Half vs. Bottom-Half Processing
* **Difficulty**: Medium to Hard

### Technical Breakdown
* **Top Half (Hard ISR)**: Runs immediately when hardware interrupt fires. Must execute in microseconds. Disables interrupts, acknowledges hardware IRQ, schedules bottom half work, and returns.
* **Bottom Half (Softirqs / Tasklets / Workqueues)**: Runs asynchronously with interrupts enabled. Performs heavy processing (parsing network packets, allocating memory, waking user processes).

---

## Q2. Explain Cache Line Alignment, False Sharing, and Memory Barriers in Embedded Driver Code.
* **Difficulty**: Hard
* **Topics**: CPU Cache Coherence, Embedded C

```c
// Preventing False Sharing in Embedded C
struct alignas(64) PerCoreData {
    uint64_t rx_bytes;
    uint64_t tx_bytes;
};
```
