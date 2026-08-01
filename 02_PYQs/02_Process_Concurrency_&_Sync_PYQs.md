# Master PYQs: Module 2 — Process Concurrency & Synchronization

## Q1. How many total child processes are created by `fork() && fork() || fork()`?
* **Asked in**: Microsoft, Amazon, Adobe, Goldman Sachs
* **Difficulty**: Medium (Classic Interview Puzzle)

### Detailed Analysis
Let's trace the short-circuit evaluation logic of `&&` and `||` operators in C:

```c
fork() [F1] && fork() [F2] || fork() [F3];
```

```
               P0 (Parent Initial)
              /                 \
       F1 (fork 1)            F1 (returns 0)
        /        \                 \
   P0 (ret>0)   P1 (Child 1)       P1 evaluates F1 as 0
      |              |              |
   F2 (fork 2)   F2 (fork 2)       P1 skips F2 (due to && short-circuit)
   /        \     /        \        |
P0(r>0)   P2    P1(r>0)   P3       P1 executes F3 (due to ||) -> Spawns P4!
  |        |      |        |
Skips F3  F3     Skips F3 F3
(ret>0)   |     (ret>0)    |
          v                v
         P5               P6
```

### Process Count Breakdown
1. **Initial Process**: $P_0$.
2. **First `fork()` [F1]**:
   * Executed by $P_0$. Spawns $P_1$.
   * $P_0$ gets return value $>0$ (True).
   * $P_1$ gets return value $0$ (False).
3. **Second `fork()` [F2]**:
   * Evaluated **only** by processes where F1 returned True ($P_0$ only, due to `&&` short-circuit).
   * $P_0$ executes F2, spawning $P_2$.
   * $P_0$ gets return $>0$ (True). $P_0$ `(F1 && F2)` is True -> skips F3 due to `||` short-circuit.
   * $P_2$ gets return $0$ (False). $P_2$ `(F1 && F2)` is False -> **executes F3**, spawning $P_5$.
4. **Third `fork()` [F3]**:
   * $P_1$ (F1 was False -> skipped F2) evaluates `(False || F3)`. Executes F3, spawning $P_4$.
   * $P_4$ is created.
   * $P_2$ executes F3, spawning $P_5$.
   * $P_1$ after F3 spawning $P_4$: $P_4$ also finishes.

* **Total Processes Created (excluding parent $P_0$) = 6 Child Processes.**
* **Total Processes Running in System = 7 Processes.**

---

## Q2. Mutex vs. Binary Semaphore vs. Counting Semaphore vs. Spinlock
* **Asked in**: Google, Qualcomm, Nvidia, Meta
* **Difficulty**: Medium

### Comprehensive Difference Matrix

| Feature | Spinlock | Mutex | Binary Semaphore | Counting Semaphore |
| :--- | :--- | :--- | :--- | :--- |
| **Ownership** | Owned by locking thread | Owned by locking thread | **No Ownership** (Any thread can signal) | **No Ownership** |
| **Waiting State** | Busy-waiting CPU loop | Sleep / Blocked state | Sleep / Blocked state | Sleep / Blocked state |
| **Initial Value** | 0 / 1 | Unlocked | 0 or 1 | $N$ (Resource count) |
| **Use Case** | Low-latency kernel ISRs | Exclusive resource access | Inter-thread Signaling | Managing $N$ connection slots |

---

## Q3. What is Priority Inversion and how does Priority Inheritance Protocol (PIP) solve it?
* **Asked in**: Apple, Lockheed Martin, Qualcomm, Boeing (Embedded & Systems)
* **Difficulty**: FAANG Advanced

### 30-Second Interview Pitch
Priority Inversion occurs when a high-priority thread $H$ is blocked waiting for a lock held by a low-priority thread $L$, while medium-priority threads $M$ (which don't need the lock) preempt thread $L$, causing thread $H$ to be indirectly delayed by lower priority threads. Priority Inheritance Protocol (PIP) solves this by temporarily boosting thread $L$'s priority to match thread $H$'s priority until $L$ releases the lock.

---

## Q4. Implement a Thread-Safe Reader-Writer Lock in C
* **Asked in**: Meta, Amazon, Uber
* **Difficulty**: Hard

```c
#include <pthread.h>
#include <stdio.h>

typedef struct {
    pthread_mutex_t lock;
    pthread_cond_t read_phase;
    pthread_cond_t write_phase;
    int readers;
    int writers_waiting;
    int writer_active;
} rwlock_t;

void rwlock_init(rwlock_t *rw) {
    pthread_mutex_init(&rw->lock, NULL);
    pthread_cond_init(&rw->read_phase, NULL);
    pthread_cond_init(&rw->write_phase, NULL);
    rw->readers = 0;
    rw->writers_waiting = 0;
    rw->writer_active = 0;
}

void rwlock_read_lock(rwlock_t *rw) {
    pthread_mutex_lock(&rw->lock);
    while (rw->writer_active || rw->writers_waiting > 0) {
        pthread_cond_wait(&rw->read_phase, &rw->lock);
    }
    rw->readers++;
    pthread_mutex_unlock(&rw->lock);
}

void rwlock_read_unlock(rwlock_t *rw) {
    pthread_mutex_lock(&rw->lock);
    rw->readers--;
    if (rw->readers == 0) {
        pthread_cond_signal(&rw->write_phase);
    }
    pthread_mutex_unlock(&rw->lock);
}

void rwlock_write_lock(rwlock_t *rw) {
    pthread_mutex_lock(&rw->lock);
    rw->writers_waiting++;
    while (rw->readers > 0 || rw->writer_active) {
        pthread_cond_wait(&rw->write_phase, &rw->lock);
    }
    rw->writers_waiting--;
    rw->writer_active = 1;
    pthread_mutex_unlock(&rw->lock);
}

void rwlock_write_unlock(rwlock_t *rw) {
    pthread_mutex_lock(&rw->lock);
    rw->writer_active = 0;
    if (rw->writers_waiting > 0) {
        pthread_cond_signal(&rw->write_phase);
    } else {
        pthread_cond_broadcast(&rw->read_phase);
    }
    pthread_mutex_unlock(&rw->lock);
}
```
