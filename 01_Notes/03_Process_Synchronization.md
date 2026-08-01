# Operating System Notes: Module 3 — Process Synchronization & Concurrency

## 1. The Critical Section Problem

When multiple concurrent processes/threads execute shared data modifications, a **Race Condition** occurs if the final outcome depends on the exact execution sequence. The code segment mutating shared data is the **Critical Section**.

```
do {
    // 1. Entry Section: Request permission to enter
    CRITICAL SECTION (Mutate shared data)
    // 2. Exit Section: Release lock & signal other threads
    REMAINDER SECTION (Execute non-shared code)
} while (TRUE);
```

### 1.1 The 3 Mandatory Requirements for CS Solutions
1. **Mutual Exclusion**: If process $P_i$ is executing in its critical section, no other process $P_j$ can execute in its critical section simultaneously.
2. **Progress**: If no process is executing in its critical section and some processes wish to enter, only those processes not executing in their remainder section can participate in deciding who enters next, and this selection cannot be postponed indefinitely.
3. **Bounded Waiting**: There must be a bound on the number of times other processes are allowed to enter their critical sections after a process has made a request to enter and before that request is granted (prevents **Starvation**).

---

## 2. Hardware Primitives for Synchronization

Modern CPUs provide atomic instructions to achieve hardware mutual exclusion.

### 2.1 Test-and-Set Instruction
```c
// Executed atomically by CPU hardware
bool TestAndSet(bool *target) {
    bool rv = *target;
    *target = true;
    return rv;
}

// Mutual Exclusion Implementation
bool lock = false;
void process() {
    while (TestAndSet(&lock)) ; // Busy wait (Spinlock)
    // CRITICAL SECTION
    lock = false; // Release lock
}
```

### 2.2 Compare-and-Swap (CAS) Instruction
```c
// Executed atomically by CPU hardware
int CompareAndSwap(int *value, int expected, int new_value) {
    int temp = *value;
    if (temp == expected)
        *value = new_value;
    return temp; // Returns original value
}

// Lockless CAS loop pattern
void atomic_increment(int *counter) {
    int current;
    do {
        current = *counter;
    } while (CompareAndSwap(counter, current, current + 1) != current);
}
```

---

## 3. High-Level Synchronization Primitives

### 3.1 Mutex vs. Spinlock vs. Semaphore vs. Monitor

| Primitive | Mechanism | Behavior when Blocked | Best Used For |
| :--- | :--- | :--- | :--- |
| **Spinlock** | Atomic CPU polling loop (`while(CAS)`) | **Busy Waiting** (Consumes 100% CPU core) | Very short critical sections (< 1 µs), kernel interrupt handlers |
| **Mutex** | Binary ownership lock | **Sleep/Block** (Context switches process to Waiting) | Protecting shared variables, heap structures in user space |
| **Semaphore** | Integer counter ($S \ge 0$) | **Sleep/Block** via wait queue | Resource pooling (e.g., $N$ database connection slots) |
| **Monitor** | High-level language construct (Java `synchronized`) | Automatic lock + Condition Variable wait queues | Encapsulated object-oriented concurrent thread safety |

---

## 4. Semaphores: Definition & Atomic Operations

A **Semaphore** $S$ is an integer variable accessed only via two atomic operations: `wait()` (or `P()`) and `signal()` (or `V()`).

```c
void wait(Semaphore *S) {
    S->value--;
    if (S->value < 0) {
        // add this process to S->queue;
        block();
    }
}

void signal(Semaphore *S) {
    S->value++;
    if (S->value <= 0) {
        // remove a process P from S->queue;
        wakeup(P);
    }
}
```

---

## 5. Classical Synchronization Problems (With C Code)

### 5.1 Producer-Consumer (Bounded Buffer Problem)

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <stdlib.h>

#define BUFFER_SIZE 5
int buffer[BUFFER_SIZE];
int in = 0, out = 0;

sem_t empty_slots; // Counts empty buffer slots
sem_t full_slots;  // Counts filled buffer slots
pthread_mutex_t mutex; // Ensures mutual exclusion for buffer insertion

void* producer(void* arg) {
    for (int i = 0; i < 10; i++) {
        int item = rand() % 100;
        
        sem_wait(&empty_slots);            // Decrement empty slot count
        pthread_mutex_lock(&mutex);        // Lock critical section
        
        buffer[in] = item;
        printf("Producer produced: %d at index %d\n", item, in);
        in = (in + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);      // Unlock critical section
        sem_post(&full_slots);             // Increment filled slot count
    }
    return NULL;
}

void* consumer(void* arg) {
    for (int i = 0; i < 10; i++) {
        sem_wait(&full_slots);             // Decrement filled slot count
        pthread_mutex_lock(&mutex);        // Lock critical section
        
        int item = buffer[out];
        printf("Consumer consumed: %d from index %d\n", item, out);
        out = (out + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);      // Unlock critical section
        sem_post(&empty_slots);            // Increment empty slot count
    }
    return NULL;
}

int main() {
    pthread_t prod, cons;
    sem_init(&empty_slots, 0, BUFFER_SIZE);
    sem_init(&full_slots, 0, 0);
    pthread_mutex_init(&mutex, NULL);

    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);

    pthread_join(prod, NULL);
    pthread_join(cons, NULL);

    sem_destroy(&empty_slots);
    sem_destroy(&full_slots);
    pthread_mutex_destroy(&mutex);
    return 0;
}
```

---

### 5.2 Readers-Writers Problem (Fair Priority Implementation)

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>

int read_count = 0;
pthread_mutex_t rmutex;
sem_t rw_mutex;
sem_t queue_mutex; // Guarantees FIFO fairness (no writer starvation)

void* reader(void* arg) {
    sem_wait(&queue_mutex); // Wait in queue
    pthread_mutex_lock(&rmutex);
    read_count++;
    if (read_count == 1) {
        sem_wait(&rw_mutex); // First reader locks out writers
    }
    pthread_mutex_unlock(&rmutex);
    sem_post(&queue_mutex); // Release queue

    // READING CRITICAL SECTION
    printf("Reader %ld is reading\n", (long)arg);

    pthread_mutex_lock(&rmutex);
    read_count--;
    if (read_count == 0) {
        sem_post(&rw_mutex); // Last reader unlocks writers
    }
    pthread_mutex_unlock(&rmutex);
    return NULL;
}

void* writer(void* arg) {
    sem_wait(&queue_mutex); // Wait in queue
    sem_wait(&rw_mutex);    // Lock exclusive writer access
    sem_post(&queue_mutex);

    // WRITING CRITICAL SECTION
    printf("Writer %ld is writing\n", (long)arg);

    sem_post(&rw_mutex);
    return NULL;
}
```

---

### 5.3 Dining Philosophers Problem (Deadlock-Free Resource Hierarchy)

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

#define N 5
pthread_mutex_t forks[N];

void* philosopher(void* num) {
    long id = (long)num;
    int left_fork = id;
    int right_fork = (id + 1) % N;

    // Resource Hierarchy: Always acquire smaller indexed fork first
    int first_fork = (left_fork < right_fork) ? left_fork : right_fork;
    int second_fork = (left_fork < right_fork) ? right_fork : left_fork;

    for (int i = 0; i < 3; i++) {
        // Thinking
        pthread_mutex_lock(&forks[first_fork]);
        pthread_mutex_lock(&forks[second_fork]);

        // Eating
        printf("Philosopher %ld is eating using forks %d and %d\n", id, first_fork, second_fork);

        pthread_mutex_unlock(&forks[second_fork]);
        pthread_mutex_unlock(&forks[first_fork]);
    }
    return NULL;
}
```

---

## 6. Advanced Concurrency Challenges

### 6.1 Priority Inversion & Priority Inheritance Protocol (PIP)
* **Problem**: Low-priority thread $L$ acquires Mutex $M$. High-priority thread $H$ preempts medium-priority threads, but becomes blocked waiting for Mutex $M$. Medium-priority thread $M_{med}$ (which doesn't need $M$) preempts $L$, causing $H$ to wait indefinitely for $M_{med}$!
* **Solution (Priority Inheritance)**: When thread $H$ blocks on Mutex $M$ held by thread $L$, thread $L$ temporarily inherits thread $H$'s high priority until it unlocks $M$.

### 6.2 Cache Line Bouncing & False Sharing
* **False Sharing**: Occurs when two independent threads running on different CPU cores modify separate variables that happen to reside on the same 64-byte L1 CPU Cache Line.
* **Impact**: CPU hardware cache coherence protocols (MESI) invalidate the entire cache line across cores continuously, severely degrading multi-threaded throughput.
* **Fix**: Align thread-private atomic variables to cache line boundaries:
  `alignas(64) std::atomic<uint64_t> thread_counter;`

---

## 7. Summary Checklist & Flash Cards

* **What are Peterson's solution requirements?** Works for 2 processes using turn variable and flag array; satisfies all 3 CS criteria.
* **Spinlock vs Mutex?** Spinlock busy-waits (good for microsecond critical sections); Mutex sleeps process (good for longer locks).
* **What is a Futex?** Fast Userspace Mutex in Linux — attempts lock acquisition in user space via CAS; only calls kernel `sys_futex` if lock is contested.
* **How to prevent Deadlock in Dining Philosophers?** Asymmetric lock ordering (even pick left first, odd pick right first) or resource hierarchy (lower index first).
