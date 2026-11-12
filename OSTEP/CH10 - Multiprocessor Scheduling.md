# CH10 - Multiprocessor Scheduling (Advanced)

## Big Picture
- Multicore is everywhere; a single-threaded app doesn't speed up with more CPUs → **multi-threaded** apps.
- Scheduling multiple jobs on multiple CPUs brings new problems: **caches**, **cache coherence**, **synchronization**, **cache affinity**.

---

## 1. Background: Caches
- Each CPU has its own cache, backed by shared main memory.
- **Temporal locality** (reuse soon) and **spatial locality** (nearby addresses).
- Problem: CPU 1 caches and modifies value D at address A (not yet written back); job moves to CPU 2, reads A → stale D.
- **Cache coherence** solved in hardware: **bus snooping** — caches watch the shared bus and invalidate/update their copies.

## 2. Don't Forget Synchronization
- Coherence does not make shared data structures safe — still need **locks**.
```c
typedef struct __Node_t { int value; struct __Node_t *next; } Node_t;
int List_Pop() {
    Node_t *tmp = head;        // two threads may both get the same head
    int value = head->value;
    head = head->next;
    free(tmp);                 // double free!
    return value;
}
```
- Fix: `pthread_mutex_lock(&m)` around the body. But locks hurt performance as #CPUs grows.

## 3. Cache Affinity
- A process builds state in a CPU's cache/TLB → runs faster if it **stays on the same CPU**.
- A multiprocessor scheduler should prefer keeping a process on the same CPU.

## 4. Single-Queue Multiprocessor Scheduling (SQMS)
- Put all jobs in **one queue**; each CPU picks the next job.
- ✅ Simple, reuses single-CPU policy.
- ❌ **Scalability**: lock contention on the single queue.
- ❌ **Cache affinity**: jobs bounce across CPUs:
```
CPU0: A E D C B ...
CPU1: B A E D C ...
```
- Mitigation: affinity mechanism (keep most jobs on the same CPU, migrate a few) → complex.

## 5. Multi-Queue Multiprocessor Scheduling (MQMS)
- **One queue per CPU**; each job is placed in one queue (randomly or by load) and scheduled independently.
```
Q0: A → C        CPU0: A A C C A A C C ...
Q1: B → D        CPU1: B B D D B B D D ...
```
- ✅ Scalable (no shared lock), ✅ natural cache affinity.
- ❌ **Load imbalance**: if C finishes, A gets 2× the CPU of B and D; if A and C finish, CPU0 is idle.

### Migration
- Move jobs between queues to balance load. With A on Q0 and B, D on Q1: continuously migrate one job back and forth.
- **Work stealing**: a queue low on work peeks at another, steals jobs if it's fuller. Peek too often → overhead; too rarely → imbalance (another voo-doo constant).

## 6. Linux Multiprocessor Schedulers
| Scheduler | Approach |
|---|---|
| O(1) | Multiple queues, priority-based (like MLFQ) |
| CFS | Multiple queues, deterministic proportional share |
| BFS | **Single** queue, proportional share (Earliest Eligible Virtual Deadline First) |

> 💡 Single queue = simple, balanced, poor scaling/affinity. Multiple queues = scalable, affinity-friendly, need load balancing.
