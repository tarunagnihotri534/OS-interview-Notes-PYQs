# Operating System Notes: Module 4 — Deadlocks

## 1. System Model & Definition

A **Deadlock** is a state where a set of processes are permanently blocked because each process holds a resource needed by another process in the set, creating an unresolvable dependency cycle.

```
Process P1 holds Resource R1 ----> Process P1 requests Resource R2
                                           |
                                           v
Process P2 holds Resource R2 <---- Process P2 requests Resource R1
```

---

## 2. The 4 Necessary & Sufficient Coffman Conditions

A deadlock can occur **if and only if** all four of the following conditions hold simultaneously in a system:

1. **Mutual Exclusion**: At least one resource must be held in a non-shareable mode (only one process can use the resource at a time).
2. **Hold and Wait**: A process must currently hold at least one resource and be waiting to acquire additional resources currently held by other processes.
3. **No Preemption**: Resources cannot be forcibly preempted from a process; a resource can only be released voluntarily by the process holding it after completing its task.
4. **Circular Wait**: A closed chain of processes exists $\{P_0, P_1, \dots, P_n\}$ such that $P_0$ waits for a resource held by $P_1$, $P_1$ waits for $P_2$, and $P_n$ waits for $P_0$.

---

## 3. Resource Allocation Graph (RAG)

A directed graph $G = (V, E)$ representing processes ($V_P$) and resources ($V_R$).
* **Request Edge** $P_i 	o R_j$: Process $P_i$ is waiting for resource $R_j$.
* **Assignment Edge** $R_j 	o P_i$: Resource $R_j$ is allocated to process $P_i$.

```
Single-Instance Resource Graph (Cycle = Deadlock)
   +------+          +------+
   |  P1  |--------->|  R1  |
   +------+          +------+
      ^                 |
      |                 v
   +------+          +------+
   |  R2  |<---------|  P2  |
   +------+          +------+
```

### 3.1 Cycle Rule for RAG
* **If single instance per resource type**: A cycle in RAG is a **necessary and sufficient** condition for deadlock.
* **If multiple instances per resource type**: A cycle in RAG is a **necessary condition**, but NOT sufficient (system may or may not be deadlocked).

---

## 4. Deadlock Handling Strategies

```
                       +---------------------------------------+
                       |    Deadlock Handling Strategies       |
                       +-------------------+-------------------+
                                           |
    +------------------+-------------------+-------------------+------------------+
    |                  |                                       |                  |
+---v----+         +---v----+                              +---v----+         +---v----+
| Ostrich|         | Deadlock|                             | Deadlock|        | Deadlock|
| Algo   |         | Prevention                            | Avoidance        | Detect &|
| Ignore |         | (Break 1 of 4 conditions)             | (Banker's)       | Recover |
+--------+         +--------+                              +--------+         +--------+
```

### 4.1 Deadlock Prevention (Breaking Coffman Conditions)
* **Eliminate Mutual Exclusion**: Make resources shareable (e.g., read-only files). *Not possible for write-only devices like printers.*
* **Eliminate Hold and Wait**: Require processes to request all resources at once before starting, or release current resources before requesting new ones. *Causes low resource utilization and starvation.*
* **Eliminate No Preemption**: If a process requesting a resource cannot get it, preempt all resources it currently holds. *Difficult for stateful resources like printers or memory transactions.*
* **Eliminate Circular Wait**: Enforce a strict global total ordering of all resource types ($F: R 	o \mathbb{N}$). A process can only request resource $R_j$ if $F(R_j) > F(R_i)$ for all resources $R_i$ it holds. **(Most practical prevention technique)**.

---

## 5. Deadlock Avoidance: Banker's Algorithm

Banker's Algorithm dynamically analyzes resource allocations to ensure the system remains in a **Safe State** (a state where there exists a sequence of process executions $\langle P_1, P_2, \dots, P_n angle$ such that every process can satisfy its maximum resource claims).

### 5.1 Mathematical Data Structures
* $N$ = Number of processes, $M$ = Number of resource types.
* $	ext{Available}[M]$: Vector of available instances of each resource type.
* $	ext{Max}[N][M]$: Matrix defining maximum demand of each process.
* $	ext{Allocation}[N][M]$: Matrix of resources currently allocated to each process.
* $	ext{Need}[N][M] = 	ext{Max}[N][M] - 	ext{Allocation}[N][M]$: Remaining resource claim per process.

---

### 5.2 Step-by-Step Solved Problem: Banker's Algorithm

Consider a system with 5 processes ($P_0, P_1, P_2, P_3, P_4$) and 3 resource types ($A, B, C$). Total system resources: $A=10, B=5, C=7$.

#### Given Allocation and Max Matrices at time $t_0$:

| Process | Allocation ($A B C$) | Max ($A B C$) |
| :--- | :--- | :--- |
| **P0** | 0 1 0 | 7 5 3 |
| **P1** | 2 0 0 | 3 2 2 |
| **P2** | 3 0 2 | 9 0 2 |
| **P3** | 2 1 1 | 2 2 2 |
| **P4** | 0 0 2 | 4 3 3 |

---

#### Step 1: Calculate Available Vector
$$	ext{Total Allocated } A = 0+2+3+2+0 = 7$$
$$	ext{Total Allocated } B = 1+0+0+1+0 = 2$$
$$	ext{Total Allocated } C = 0+0+2+1+2 = 5$$
$$	ext{Available} = (10 - 7, 5 - 2, 7 - 5) = (3, 3, 2)$$

---

#### Step 2: Compute Need Matrix ($	ext{Need} = 	ext{Max} - 	ext{Allocation}$)

| Process | Need ($A B C$) |
| :--- | :--- |
| **P0** | $(7-0, 5-1, 3-0) = \mathbf{7\ 4\ 3}$ |
| **P1** | $(3-2, 2-0, 2-0) = \mathbf{1\ 2\ 2}$ |
| **P2** | $(9-3, 0-0, 2-2) = \mathbf{6\ 0\ 0}$ |
| **P3** | $(2-2, 2-1, 2-1) = \mathbf{0\ 1\ 1}$ |
| **P4** | $(4-0, 3-0, 3-2) = \mathbf{4\ 3\ 1}$ |

---

#### Step 3: Run Safety Algorithm

Initialize $	ext{Work} = 	ext{Available} = (3, 3, 2)$, $	ext{Finish}[P_i] = 	ext{False}$ for all $i$.

1. **Check P0**: $	ext{Need}_0 (7, 4, 3) \le 	ext{Work} (3, 3, 2)$ ? **No** ($7 \le 3$ is false). Skip P0.
2. **Check P1**: $	ext{Need}_1 (1, 2, 2) \le 	ext{Work} (3, 3, 2)$ ? **Yes!**
   * P1 can execute to completion.
   * $	ext{Work} = 	ext{Work} + 	ext{Allocation}_1 = (3, 3, 2) + (2, 0, 0) = \mathbf{(5, 3, 2)}$.
   * $	ext{Finish}[P1] = 	ext{True}$.
3. **Check P3**: $	ext{Need}_3 (0, 1, 1) \le 	ext{Work} (5, 3, 2)$ ? **Yes!**
   * P3 can execute to completion.
   * $	ext{Work} = 	ext{Work} + 	ext{Allocation}_3 = (5, 3, 2) + (2, 1, 1) = \mathbf{(7, 4, 3)}$.
   * $	ext{Finish}[P3] = 	ext{True}$.
4. **Check P4**: $	ext{Need}_4 (4, 3, 1) \le 	ext{Work} (7, 4, 3)$ ? **Yes!**
   * P4 can execute to completion.
   * $	ext{Work} = 	ext{Work} + 	ext{Allocation}_4 = (7, 4, 3) + (0, 0, 2) = \mathbf{(7, 4, 5)}$.
   * $	ext{Finish}[P4] = 	ext{True}$.
5. **Check P0**: $	ext{Need}_0 (7, 4, 3) \le 	ext{Work} (7, 4, 5)$ ? **Yes!**
   * P0 can execute to completion.
   * $	ext{Work} = 	ext{Work} + 	ext{Allocation}_0 = (7, 4, 5) + (0, 1, 0) = \mathbf{(7, 5, 5)}$.
   * $	ext{Finish}[P0] = 	ext{True}$.
6. **Check P2**: $	ext{Need}_2 (6, 0, 0) \le 	ext{Work} (7, 5, 5)$ ? **Yes!**
   * P2 can execute to completion.
   * $	ext{Work} = 	ext{Work} + 	ext{Allocation}_2 = (7, 5, 5) + (3, 0, 2) = \mathbf{(10, 5, 7)}$.
   * $	ext{Finish}[P2] = 	ext{True}$.

#### Result:
The system is in a **Safe State** with Safe Sequence: **$\langle P_1, P_3, P_4, P_0, P_2 angle$**.

---

## 6. Deadlock Detection & Recovery

* **Detection Algorithm**: Uses an algorithm similar to Banker's safety check on active allocation and request matrices periodically.
* **Recovery Options**:
  1. **Process Termination**: Abort all deadlocked processes, or abort one process at a time until cycle is broken.
  2. **Resource Preemption**: Rollback process to safe checkpoint and preempt resources (requires state save support).

---

## 7. Summary Checklist & Flash Cards

* **Difference between Safe State and Unsafe State?** A Safe State guarantees no deadlock can occur; an Unsafe State contains the *potential* for a deadlock if maximum claims are requested simultaneously.
* **What is the time complexity of Banker's Algorithm?** $O(M 	imes N^2)$ where $N$ is processes and $M$ is resource types.
* **Why isn't Banker's algorithm used in production OS kernels?** Because processes cannot predict their maximum future resource claims in advance.
