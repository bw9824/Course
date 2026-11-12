# CH32 - Common Concurrency Problems

## Big Picture
- Study of real bugs (Lu et al., ASPLOS'08) in MySQL, Apache, Mozilla, OpenOffice: **105 bugs** — **74 non-deadlock**, **31 deadlock**.
- Most non-deadlock bugs are **atomicity violations** or **order violations**.

---

## 1. Non-Deadlock Bugs
| Application | Non-deadlock | Deadlock |
|---|---|---|
| MySQL | 14 | 9 |
| Apache | 13 | 4 |
| Mozilla | 41 | 16 |
| OpenOffice | 6 | 2 |
| **Total** | **74** | **31** |

### Atomicity Violation
- Desired serializability of a code region is violated.
```c
// Thread 1
if (thd->proc_info) { fputs(thd->proc_info, ...); }
// Thread 2
thd->proc_info = NULL;
```
- T1 passes the check, is interrupted, T2 sets NULL → T1 dereferences NULL.
- Fix: lock around **both** accesses.

### Order Violation
- Desired order between two accesses is flipped.
```c
// Thread 1
void init() { mThread = PR_CreateThread(mMain, ...); }
// Thread 2
void mMain(...) { mState = mThread->State; }   // assumes mThread already set
```
- Fix: enforce ordering with a **CV** (`mtInit` flag + wait/signal).

- 97% of non-deadlock bugs are one of these two.

## 2. Deadlock Bugs
```c
// Thread 1          // Thread 2
lock(L1);            lock(L2);
lock(L2);            lock(L1);
```
- Cycle in the **dependency graph** → both wait forever.
- Why they occur: large code bases with complex dependencies (VM ↔ FS), **encapsulation** hides locking (e.g., Java `v1.AddAll(v2)` locks v1 then v2; another thread calls `v2.AddAll(v1)`).

### Four Conditions for Deadlock (all must hold)
| Condition | Meaning |
|---|---|
| **Mutual exclusion** | Threads claim exclusive control of resources |
| **Hold-and-wait** | Hold resources while waiting for more |
| **No preemption** | Resources can't be forcibly taken |
| **Circular wait** | Circular chain of threads, each waiting on the next |

## 3. Prevention
### Circular Wait
- **Total ordering** of lock acquisition (always L1 before L2), or **partial ordering** (Linux memory mapping code documents ~10 groups).
- Trick: **lock address ordering**:
```c
if (m1 > m2) { lock(m1); lock(m2); }
else         { lock(m2); lock(m1); }
```
### Hold-and-Wait
```c
pthread_mutex_lock(prevention);   // begin acquisition
pthread_mutex_lock(L1);
pthread_mutex_lock(L2);
pthread_mutex_unlock(prevention); // end
```
- ❌ Must know all locks up front; reduces concurrency.

### No Preemption
```c
top:
    pthread_mutex_lock(L1);
    if (pthread_mutex_trylock(L2) != 0) { pthread_mutex_unlock(L1); goto top; }
```
- ❌ **Livelock**: both threads keep retrying and failing → add random delay.
- ❌ Must release other resources acquired in between.

### Mutual Exclusion → Lock-Free / Wait-Free
```c
void AtomicIncrement(int *value, int amount) {
    do { int old = *value; } while (CompareAndSwap(value, old, old + amount) == 0);
}
void insert(int value) {           // lock-free list insert
    node_t *n = malloc(sizeof(node_t)); n->value = value;
    do { n->next = head; } while (CompareAndSwap(&head, n->next, n) == 0);
}
```

## 4. Avoidance via Scheduling
- If the scheduler knows which locks each thread needs, schedule them so deadlock can't occur (e.g., don't run T1 and T2 concurrently if both need L1 and L2).
- **Banker's algorithm** (Dijkstra) — limited applicability; requires full knowledge, reduces concurrency.

## 5. Detect and Recover
- Allow deadlocks occasionally, detect (build resource graph, look for cycles), then recover (restart / kill).
- Common in databases.

> 💡 Tom West's Law: **not everything worth doing is worth doing well** — if deadlock is rare and harmless (reboot), don't over-engineer.
