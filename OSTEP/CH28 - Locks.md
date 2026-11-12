# CH28 - Locks

## Big Picture
- **Lock**: a variable around a critical section → it runs as if it were a single atomic instruction.
- How to build an efficient lock? Needs **hardware** (atomic instructions) and **OS** support.
- Evaluation criteria: **mutual exclusion** (correctness), **fairness** (no starvation), **performance** (uncontended, single-CPU contended, multi-CPU contended).

---

## 1. Basic Idea
```c
lock_t mutex;   // globally-allocated lock 'mutex'
lock(&mutex);
balance = balance + 1;
unlock(&mutex);
```
- State: **available** (unlocked/free) or **acquired** (locked/held, by exactly one **owner**).
- Pthreads: **mutex** (mutual exclusion); different locks for different data → **fine-grained** locking (more concurrency) vs one big **coarse-grained** lock.

## 2. Controlling Interrupts (single CPU)
```c
void lock()   { DisableInterrupts(); }
void unlock() { EnableInterrupts(); }
```
- ✅ Simple. ❌ Requires trusting apps with a **privileged** operation (could monopolize CPU); ❌ doesn't work on **multiprocessors**; ❌ lost interrupts; ❌ slow on modern CPUs.
- Used only inside the OS in limited contexts.

## 3. A Failed Attempt: Just Load/Store
```c
void lock(lock_t *mutex) {
    while (mutex->flag == 1) ; // spin-wait (do nothing)
    mutex->flag = 1;           // now SET it!
}
```
- ❌ Correctness: both threads can see flag = 0 (interrupt between test and set) → both enter.
- ❌ Performance: **spin-waiting** wastes CPU.

## 4. Test-And-Set (Atomic Exchange)
```c
int TestAndSet(int *old_ptr, int new) {   // executed atomically by hardware
    int old = *old_ptr;
    *old_ptr = new;
    return old;
}
void lock(lock_t *lock) {
    while (TestAndSet(&lock->flag, 1) == 1) ; // spin
}
void unlock(lock_t *lock) { lock->flag = 0; }
```
- **Spin lock**. Needs a **preemptive scheduler** on single CPU (otherwise spinning thread never yields).
- x86: `xchg`; SPARC: `ldstub`.

| Criterion | Spin lock |
|---|---|
| Correctness | ✅ |
| Fairness | ❌ may spin forever (starvation) |
| Performance (1 CPU) | ❌ painful: waiters spin entire time slices |
| Performance (multi-CPU, threads ≈ CPUs) | ✅ reasonably well |

## 5. Compare-And-Swap
```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int original = *ptr;
    if (original == expected) *ptr = new;
    return original;
}
void lock(lock_t *lock) { while (CompareAndSwap(&lock->flag, 0, 1) == 1) ; }
```
- More powerful than test-and-set (e.g., lock-free synchronization). x86: `cmpxchg`.

## 6. Load-Linked / Store-Conditional (MIPS, ARM, PowerPC)
```c
void lock(lock_t *lock) {
    while (1) {
        while (LoadLinked(&lock->flag) == 1) ; // spin until it's zero
        if (StoreConditional(&lock->flag, 1) == 1) return; // success
        // otherwise: try again
    }
}
```
- SC succeeds only if **no store** to the address happened since the LL.

## 7. Fetch-And-Add → Ticket Lock
```c
int FetchAndAdd(int *ptr) { int old = *ptr; *ptr = old + 1; return old; }
typedef struct { int ticket; int turn; } lock_t;
void lock(lock_t *lock) {
    int myturn = FetchAndAdd(&lock->ticket);
    while (lock->turn != myturn) ; // spin
}
void unlock(lock_t *lock) { lock->turn = lock->turn + 1; }
```
- ✅ **Fairness**: every thread eventually gets its turn (FIFO order).

## 8. Too Much Spinning: What Now?
- Thread holding the lock is preempted → N−1 threads spin for full slices.
### Simple Approach: Just Yield
```c
void lock() { while (TestAndSet(&flag, 1) == 1) yield(); }  // give up the CPU
```
- Better on 1 CPU, but with 100 threads: lots of switches; starvation still possible.

### Using Queues: Sleeping Instead of Spinning (Solaris `park()` / `unpark()`)
```c
typedef struct { int flag; int guard; queue_t *q; } lock_t;
void lock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1) ; // acquire guard by spinning
    if (m->flag == 0) { m->flag = 1; m->guard = 0; }       // lock acquired
    else { queue_add(m->q, gettid()); setpark(); m->guard = 0; park(); }
}
void unlock(lock_t *m) {
    while (TestAndSet(&m->guard, 1) == 1) ;
    if (queue_empty(m->q)) m->flag = 0;      // let go of lock; no one wants it
    else unpark(queue_remove(m->q));         // hold lock (for next thread!)
    m->guard = 0;
}
```
- `guard` spin is short (only a few instructions).
- **Wakeup/waiting race**: thread is about to park, lock released before it parks → sleeps forever. Fix: **`setpark()`** — if unpark happens before park, park returns immediately.
- Flag is **not** reset on handoff: lock passed directly to the woken thread.

## 9. Linux Futex
- `futex_wait(address, expected)`: sleep if `*address == expected`; `futex_wake(address)`: wake one waiter.
- Linux mutex uses one integer: **high bit** = locked, **other bits** = number of waiters. Fast path (uncontended) uses only an atomic op, no syscall.

## 10. Two-Phase Locks
- Phase 1: **spin a while** hoping to get the lock soon; phase 2: **sleep** (futex).
- Hybrid, as Linux's lock does (spin once). Works well when locks are released soon.

> 💡 Peterson's / Dekker's algorithms achieve mutual exclusion with only loads/stores, but don't work on modern hardware with **relaxed memory consistency** without fences.
