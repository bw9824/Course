# CH30 - Condition Variables

## Big Picture
- Locks aren't enough: a thread often must **wait for a condition** (e.g., child finished, buffer non-empty).
- Spinning on a shared variable wastes CPU (and can be wrong).
- **Condition variable (CV)**: an explicit **queue** threads put themselves on when a condition isn't met; another thread **signals** when it changes.

---

## 1. Definition and Routines
```c
pthread_cond_wait(pthread_cond_t *c, pthread_mutex_t *m);  // atomically release m and sleep; reacquire m on wake
pthread_cond_signal(pthread_cond_t *c);                    // wake one waiting thread
```
### Parent waiting for child (join)
```c
int done = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  c = PTHREAD_COND_INITIALIZER;

void thr_exit() {
    Pthread_mutex_lock(&m);
    done = 1;
    Pthread_cond_signal(&c);
    Pthread_mutex_unlock(&m);
}
void thr_join() {
    Pthread_mutex_lock(&m);
    while (done == 0) Pthread_cond_wait(&c, &m);
    Pthread_mutex_unlock(&m);
}
```
- Why the **state variable `done`**? Without it, if the child runs first and signals before the parent waits → parent sleeps forever.
- Why the **lock**? Without it: parent checks `done == 0`, gets interrupted, child sets done & signals (no one waiting), parent sleeps forever.

> 💡 **Hold the lock when calling signal or wait.**

## 2. The Producer/Consumer (Bounded Buffer) Problem
- Producers put items in a buffer; consumers take them out (e.g., `grep foo file.txt | wc -l`, web server work queues).
```c
int buffer; int count = 0;   // single-slot buffer to start
void put(int value) { assert(count == 0); count = 1; buffer = value; }
int  get()          { assert(count == 1); count = 0; return buffer; }
```

### Broken Solution #1: `if` + single CV
```c
// consumer
Pthread_mutex_lock(&mutex);
if (count == 0) Pthread_cond_wait(&cond, &mutex);
int tmp = get();
...
```
- Two consumers: Tc1 waits; producer fills & signals → Tc1 ready; **Tc2 sneaks in** and consumes; Tc1 runs, buffer empty → assert fails.
- **Mesa semantics**: a signal is only a **hint** the state changed; no guarantee it still holds when the woken thread runs. (**Hoare semantics** guarantees immediate run — rarely implemented.)

### Better, Still Broken: `while`
- `while (count == 0) wait` fixes the above. Rule: **always use while loops** (also handles spurious wakeups).
- But with **one CV**: consumer may wake another **consumer** instead of the producer → all three asleep.

### Single-Buffer Solution: Two CVs
```c
cond_t empty, fill;
void *producer(void *arg) {
    for (int i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        while (count == 1) Pthread_cond_wait(&empty, &mutex);
        put(i);
        Pthread_cond_signal(&fill);
        Pthread_mutex_unlock(&mutex);
    }
}
void *consumer(void *arg) {
    for (int i = 0; i < loops; i++) {
        Pthread_mutex_lock(&mutex);
        while (count == 0) Pthread_cond_wait(&fill, &mutex);
        int tmp = get();
        Pthread_cond_signal(&empty);
        Pthread_mutex_unlock(&mutex);
        printf("%d\n", tmp);
    }
}
```
- Producers wait on `empty`, signal `fill`; consumers the reverse.

### Final Solution: Multi-Slot Buffer
```c
int buffer[MAX]; int fill_ptr = 0; int use_ptr = 0; int count = 0;
void put(int value) { buffer[fill_ptr] = value; fill_ptr = (fill_ptr + 1) % MAX; count++; }
int  get()          { int tmp = buffer[use_ptr]; use_ptr = (use_ptr + 1) % MAX; count--; return tmp; }
// producer: while (count == MAX) wait(&empty) ; consumer: while (count == 0) wait(&fill)
```
- More concurrency, fewer context switches.

## 3. Covering Conditions
- Memory allocator: `allocate(size)` waits until enough free memory; `free()` signals — but which waiter?
```c
void *allocate(int size) {
    Pthread_mutex_lock(&m);
    while (bytesLeft < size) Pthread_cond_wait(&c, &m);
    ... bytesLeft -= size; ...
}
void free(void *ptr, int size) {
    Pthread_mutex_lock(&m);
    bytesLeft += size;
    Pthread_cond_broadcast(&c);   // wake ALL waiters
    Pthread_mutex_unlock(&m);
}
```
- **`pthread_cond_broadcast()`**: wakes all waiters; each rechecks; those that can't proceed go back to sleep.
- **Covering condition** (Lampson & Redell): covers all cases conservatively; cost = needless wakeups.

> 💡 If a program only works with broadcast instead of signal, you probably have a bug — except in true covering-condition cases.
