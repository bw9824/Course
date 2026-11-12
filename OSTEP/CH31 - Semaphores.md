# CH31 - Semaphores

## Big Picture
- **Semaphore** (Dijkstra): an integer with two routines; one primitive usable as **both lock and condition variable**.
- POSIX: `sem_wait()` (P) and `sem_post()` (V).

---

## 1. Definition
```c
#include <semaphore.h>
sem_t s;
sem_init(&s, 0, 1);   // initial value 1; 0 = shared among threads of one process

int sem_wait(sem_t *s) { /* decrement s by one; wait if value of s is negative */ }
int sem_post(sem_t *s) { /* increment s by one; if there are waiting threads, wake one */ }
```
- When negative, the value = **number of waiting threads**.
- Both are executed **atomically**.

## 2. Binary Semaphores (Locks)
```c
sem_t m;
sem_init(&m, 0, 1);   // initial value X = 1
sem_wait(&m);
// critical section
sem_post(&m);
```
| Val | Thread 0 | Thread 1 |
|---|---|---|
| 1 | call `sem_wait()` | |
| 0 | returns, in critical section | |
| 0 | | call `sem_wait()` |
| −1 | | decrement, sleep |
| −1 | `sem_post()` | |
| 0 | increment, wake T1 | |
| 0 | | returns, in critical section |

## 3. Semaphores for Ordering
```c
sem_t s;
void *child(void *arg) { printf("child\n"); sem_post(&s); return NULL; }
int main() {
    sem_init(&s, 0, 0);    // X = 0
    Pthread_create(&c, NULL, child, NULL);
    sem_wait(&s);          // wait here for child
    printf("parent: end\n");
}
```
- Works whether child runs first (value → 1, parent's wait returns immediately) or parent first (parent sleeps at −1).

> 💡 Initial value rule of thumb: **how many resources you're willing to give away right after init**. Lock → 1; ordering → 0.

## 4. Producer/Consumer (Bounded Buffer)
```c
sem_t empty, full, mutex;
void *producer(void *arg) {
    for (int i = 0; i < loops; i++) {
        sem_wait(&empty);
        sem_wait(&mutex);    // lock INSIDE
        put(i);
        sem_post(&mutex);
        sem_post(&full);
    }
}
void *consumer(void *arg) {
    for (int i = 0; i < loops; i++) {
        sem_wait(&full);
        sem_wait(&mutex);
        int tmp = get();
        sem_post(&mutex);
        sem_post(&empty);
    }
}
// sem_init(&empty, 0, MAX); sem_init(&full, 0, 0); sem_init(&mutex, 0, 1);
```
- Without `mutex` and MAX > 1: two producers race in `put()` → overwrite data.
> ⚠️ Putting the mutex **outside** `empty`/`full` waits → **deadlock** (consumer holds mutex, sleeps on full; producer can't get mutex).

## 5. Reader-Writer Locks
```c
typedef struct { sem_t lock; sem_t writelock; int readers; } rwlock_t;
void rwlock_acquire_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers++;
    if (rw->readers == 1) sem_wait(&rw->writelock);   // first reader acquires writelock
    sem_post(&rw->lock);
}
void rwlock_release_readlock(rwlock_t *rw) {
    sem_wait(&rw->lock);
    rw->readers--;
    if (rw->readers == 0) sem_post(&rw->writelock);   // last reader releases
    sem_post(&rw->lock);
}
void rwlock_acquire_writelock(rwlock_t *rw) { sem_wait(&rw->writelock); }
void rwlock_release_writelock(rwlock_t *rw) { sem_post(&rw->writelock); }
```
- Many readers concurrently, writers exclusive. ❌ **Writer starvation** possible; extra overhead → often not faster than a simple lock.

> 💡 Simple and dumb can be better (Hill's Law).

## 6. The Dining Philosophers
- 5 philosophers, 5 forks between them; each needs **left and right** forks to eat.
```c
int left(int p)  { return p; }
int right(int p) { return (p + 1) % 5; }
```
- Naive: everyone grabs left then right → **deadlock** (all hold left, wait for right).
- Fix: **break the dependency** — philosopher 4 grabs **right first**.
```c
void get_forks(int p) {
    if (p == 4) { sem_wait(&forks[right(p)]); sem_wait(&forks[left(p)]); }
    else        { sem_wait(&forks[left(p)]);  sem_wait(&forks[right(p)]); }
}
```

## 7. Thread Throttling
- Limit how many threads do something at once (e.g., memory-intensive region) → semaphore initialized to the max count.

## 8. Implementing Semaphores (Zemaphores)
```c
typedef struct { int value; pthread_cond_t cond; pthread_mutex_t lock; } Zem_t;
void Zem_wait(Zem_t *s) {
    Mutex_lock(&s->lock);
    while (s->value <= 0) Cond_wait(&s->cond, &s->lock);
    s->value--;
    Mutex_unlock(&s->lock);
}
void Zem_post(Zem_t *s) {
    Mutex_lock(&s->lock);
    s->value++;
    Cond_signal(&s->cond);
    Mutex_unlock(&s->lock);
}
```
- One lock + one CV + a value. Value never goes below 0 (matches Linux; differs from Dijkstra's definition).
- Building CVs from semaphores is much harder.
