# CH27 - Interlude: Thread API

## Big Picture
- POSIX threads (**pthreads**) interface: create, join, locks, condition variables.
- Compile with `#include <pthread.h>` and link with `-pthread`.

---

## 1. Thread Creation
```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```
- `thread`: handle; `attr`: stack size, scheduling priority (NULL = defaults, init with `pthread_attr_init()`).
- `start_routine`: function pointer taking `void *`, returning `void *` → any argument / return type.
```c
typedef struct { int a; int b; } myarg_t;
void *mythread(void *arg) { myarg_t *args = (myarg_t *) arg; printf("%d %d\n", args->a, args->b); return NULL; }
myarg_t args = { 10, 20 };
int rc = pthread_create(&p, NULL, mythread, &args);
```

## 2. Thread Completion
```c
int pthread_join(pthread_t thread, void **value_ptr);
```
```c
typedef struct { int x; int y; } myret_t;
void *mythread(void *arg) {
    myret_t *rvals = Malloc(sizeof(myret_t));   // heap, not stack!
    rvals->x = 1; rvals->y = 2;
    return (void *) rvals;
}
myret_t *rvals;
Pthread_join(p, (void **) &rvals);
```
> ⚠️ Never return a pointer to something on the thread's **stack** — it's deallocated when the thread returns.
- Simple values: pass/return `long long int` via casting, no malloc.
- Not every program joins: e.g., a web server's worker threads run forever.

## 3. Locks
```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;   // or pthread_mutex_init(&lock, NULL)
pthread_mutex_lock(&lock);
x = x + 1;   // critical section
pthread_mutex_unlock(&lock);
```
- Must **initialize**; check error codes (or wrap: `Pthread_mutex_lock` asserts `rc == 0`).
- Also: `pthread_mutex_trylock()` (fails if held), `pthread_mutex_timedlock()` (gives up after timeout).
- `pthread_mutex_destroy()` when done.

## 4. Condition Variables
```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);
```
```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cond = PTHREAD_COND_INITIALIZER;

// waiting thread
Pthread_mutex_lock(&lock);
while (ready == 0)
    Pthread_cond_wait(&cond, &lock);   // releases lock while sleeping, reacquires before returning
Pthread_mutex_unlock(&lock);

// signaling thread
Pthread_mutex_lock(&lock);
ready = 1;
Pthread_cond_signal(&cond);
Pthread_mutex_unlock(&lock);
```
- Always hold the lock when signaling or modifying the condition.
- Use **`while`**, not `if` — **spurious wakeups** possible; recheck the condition.

> ⚠️ Don't use a simple flag + spin loop instead of a CV: wastes CPU and is error-prone (memory ordering bugs).

## 5. Thread API Guidelines
- **Keep it simple**: tricky thread interaction → bugs.
- **Minimize thread interactions**.
- **Initialize locks and condition variables**.
- **Check return codes**.
- **Be careful with arguments/return values** (no stack pointers).
- **Each thread has its own stack** → locals are private; share via heap/globals.
- **Always use CVs to signal between threads**.
- **Use the manual pages** (Linux man pages are informative).
