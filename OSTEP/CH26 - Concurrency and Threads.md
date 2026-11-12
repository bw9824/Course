# CH26 - Concurrency: An Introduction

## Big Picture
- **Thread**: like a separate process, but threads **share the same address space**.
- Each thread has its own **PC**, **registers**, and **stack** (thread-local storage).
- Switching threads = context switch, but **no address-space switch** (page table stays).
- Sharing data → **race conditions** → need **mutual exclusion** and **waiting** primitives.

---

## 1. Why Use Threads?
- **Parallelism**: split work across CPUs (parallelization).
- **Avoid blocking on slow I/O**: one thread waits, others compute (like multiprogramming within one program).
- vs processes: threads naturally **share data**; processes are better for logically separate tasks.

## 2. Single- vs Multi-Threaded Address Space
```
0KB  | Program Code |        | Program Code |
1KB  | Heap         |        | Heap         |
     |   (free)     |        |   (free)     |
     |              |        | Stack (2)    |
     |              |        |   (free)     |
16KB | Stack        |        | Stack (1)    |
```
- Each thread's state saved in a **TCB (Thread Control Block)**.

## 3. Thread Creation
```c
void *mythread(void *arg) { printf("%s\n", (char *) arg); return NULL; }
pthread_t p1, p2;
Pthread_create(&p1, NULL, mythread, "A");
Pthread_create(&p2, NULL, mythread, "B");
Pthread_join(p1, NULL);   // wait for threads to finish
Pthread_join(p2, NULL);
```
- Order of "A" and "B" is **not deterministic** — the scheduler decides.

## 4. Why It Gets Worse: Shared Data
```c
static volatile int counter = 0;
void *mythread(void *arg) {
    for (int i = 0; i < 1e7; i++) counter = counter + 1;
    return NULL;
}
```
- Expected 20,000,000; got **19,345,221**, then 19,221,041 — different each run.

## 5. The Heart of the Problem: Uncontrolled Scheduling
```asm
mov 0x8049a1c, %eax   ; load counter
add $0x1, %eax        ; increment
mov %eax, 0x8049a1c   ; store
```
| OS | Thread 1 | Thread 2 | PC | eax | counter |
|---|---|---|---|---|---|
| | before critical section | | 100 | 0 | 50 |
| | mov → eax = 50 | | 105 | 50 | 50 |
| | add → eax = 51 | | 108 | 51 | 50 |
| interrupt: save T1, restore T2 | | | | | |
| | | mov → eax = 50 | 105 | 50 | 50 |
| | | add → 51 | 108 | 51 | 50 |
| | | mov → counter = 51 | 113 | 51 | 51 |
| interrupt: save T2, restore T1 | | | | | |
| | mov → counter = 51 | | 113 | 51 | **51** |
- Two increments, counter only +1.

## 6. Key Concurrency Terms
| Term | Meaning |
|---|---|
| **Critical section** | Code accessing a shared resource (variable/structure) |
| **Race condition** (data race) | Multiple threads enter the critical section at roughly the same time, result depends on timing |
| **Indeterminate** program | Output varies run to run |
| **Mutual exclusion** | Only one thread in the critical section at a time |

## 7. The Wish for Atomicity
- Ideal: a single instruction `memory-add 0x8049a1c, $0x1` that is **atomic** ("all or nothing").
- Not practical for every operation (e.g. a whole B-tree update) → build **synchronization primitives** from a few hardware instructions + OS support.

> 💡 Atomic operations are also key in file systems (journaling, copy-on-write) and databases (**transactions**).

## 8. Another Problem: Waiting for Another
- Not only shared access: one thread must **wait** for another (e.g., I/O completion) → **condition variables** (later).

## 9. Why in OS Class?
- The **OS was the first concurrent program** — interrupts can occur anytime; file allocation, process lists, page tables, etc. all need careful synchronization.
