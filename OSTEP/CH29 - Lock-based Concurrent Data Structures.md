# CH29 - Lock-based Concurrent Data Structures

## Big Picture
- Adding locks makes a data structure **thread safe**. Crux: add locks so it's **correct** and **performs well** (scales).
- Start simple: one big lock; improve only if there's a performance problem.

---

## 1. Concurrent Counters
### Simple but Not Scalable
```c
typedef struct { int value; pthread_mutex_t lock; } counter_t;
void increment(counter_t *c) { Pthread_mutex_lock(&c->lock); c->value++; Pthread_mutex_unlock(&c->lock); }
```
- Correct, but 1M increments: 1 thread ~0.03s, 2 threads ~5s on 4 CPUs → **poor scaling**.
- Ideal: **perfect scaling** — N threads on N CPUs as fast as one thread on one CPU.

### Approximate (Sloppy) Counter
- One **local counter per CPU** (each with its own lock) + one **global counter** (with lock).
- Thread increments its local counter; when local reaches threshold **S**, add to global and reset local.
```c
void update(counter_t *c, int threadID, int amt) {
    int cpu = threadID % NUMCPUS;
    pthread_mutex_lock(&c->llock[cpu]);
    c->local[cpu] += amt;
    if (c->local[cpu] >= c->threshold) {     // transfer to global
        pthread_mutex_lock(&c->glock);
        c->global += c->local[cpu];
        pthread_mutex_unlock(&c->glock);
        c->local[cpu] = 0;
    }
    pthread_mutex_unlock(&c->llock[cpu]);
}
int get(counter_t *c) { /* lock glock; return global (approximate) */ }
```
- **S** trade-off: small → accurate but slow; large → scales well, global lags by up to CPUs × S.
- S = 1024: 4 threads × 1M increments as fast as 1 thread.

## 2. Concurrent Linked Lists
```c
int List_Insert(list_t *L, int key) {
    node_t *new = malloc(sizeof(node_t));     // malloc outside the lock
    if (new == NULL) { perror("malloc"); return -1; }
    new->key = key;
    pthread_mutex_lock(&L->lock);             // lock only the critical section
    new->next = L->head;
    L->head = new;
    pthread_mutex_unlock(&L->lock);
    return 0;
}
```
- Restructure to minimize code under the lock and reduce **error-path unlock bugs** (single return path in lookup).

### Hand-Over-Hand Locking (Lock Coupling)
- A lock **per node**; acquire the next node's lock before releasing the current.
- Enables concurrency in theory, but lock overhead per node usually makes it **slower** than one big lock.

> 💡 More concurrency isn't necessarily faster. Beware of control flow changes (returns, errors) that forget to release locks.

## 3. Concurrent Queues
- **Michael & Scott queue**: two locks — **head lock** (dequeue) and **tail lock** (enqueue) — plus a **dummy node** to separate them.
```c
void Queue_Enqueue(queue_t *q, int value) {
    node_t *tmp = malloc(sizeof(node_t)); tmp->value = value; tmp->next = NULL;
    pthread_mutex_lock(&q->tail_lock);
    q->tail->next = tmp; q->tail = tmp;
    pthread_mutex_unlock(&q->tail_lock);
}
int Queue_Dequeue(queue_t *q, int *value) {
    pthread_mutex_lock(&q->head_lock);
    node_t *tmp = q->head; node_t *new_head = tmp->next;
    if (new_head == NULL) { pthread_mutex_unlock(&q->head_lock); return -1; } // empty
    *value = new_head->value; q->head = new_head;
    pthread_mutex_unlock(&q->head_lock);
    free(tmp);
    return 0;
}
```
- Enqueue and dequeue run **concurrently**. A **bounded** queue (waiting when empty/full) needs CVs (next chapter).

## 4. Concurrent Hash Table
- Built from concurrent lists: **one lock per bucket** (not one per table).
```c
#define BUCKETS (101)
typedef struct { list_t lists[BUCKETS]; } hash_t;
int Hash_Insert(hash_t *H, int key) { return List_Insert(&H->lists[key % BUCKETS], key); }
```
- Scales **excellently** compared to a single-lock list.

## 5. Summary
- Be careful with lock acquire/release around control flow changes.
- Enabling more concurrency doesn't always increase performance.
- Fix performance only when it's a problem — **avoid premature optimization** (Knuth).
- Many OSes started with one **big kernel lock** (early Linux, BSD) and later moved to finer-grained locks as multicore made it a bottleneck.
