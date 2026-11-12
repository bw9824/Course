# CH09 - Scheduling: Proportional Share

## Big Picture
- **Proportional-share** (fair-share) scheduler: instead of optimizing turnaround/response, guarantee each job a **certain percentage** of CPU time.
- Three approaches: **lottery** (randomized), **stride** (deterministic), **Linux CFS** (virtual runtime).

---

## 1. Tickets
- **Tickets** represent a process's share. A has 75, B has 25 → A should get 75% of the CPU.
- **Lottery**: every time slice, pick a random winning ticket in [0, total); winner runs.
- Probabilistic: correct **on average**, over longer runs.

> 💡 Randomness is good: avoids corner-case worst behavior (cf. LRU), little state per process, fast decisions.

## 2. Ticket Mechanisms
| Mechanism | Meaning |
|---|---|
| **Ticket currency** | A user allocates tickets in their own currency; the system converts to global value (A1: 500 of A's 1000 → 50 global) |
| **Ticket transfer** | Client temporarily hands tickets to a server to speed up work on its behalf |
| **Ticket inflation** | A process raises/lowers its own tickets — only in trusted (cooperative) settings |

## 3. Implementation
```c
int counter = 0;
int winner = getrandom(0, totaltickets);   // pick winner
node_t *current = head;
while (current) {                           // walk the list
    counter = counter + current->tickets;
    if (counter > winner) break;            // found the winner
    current = current->next;
}
// 'current' is the winner: schedule it
```
- Just a random number generator + a list. Sorting by descending tickets → fewer iterations.

## 4. Fairness Metric
- **Unfairness U** = completion time of first job / completion time of second (two jobs, same tickets & run time). Perfect = 1.
- Short jobs → U well below 1; U → 1 as job length grows.

## 5. How to Assign Tickets?
- Open problem ("ticket-assignment problem"); one option: let users decide.

## 6. Stride Scheduling (deterministic)
- **stride = BIG_NUMBER / tickets** (e.g. 10000 / {100, 50, 250} → 100, 200, 40).
- Each process has a **pass** value; run the process with the **lowest pass**, then `pass += stride`.
```c
curr = remove_min(queue);   // lowest pass value
schedule(curr);
curr->pass += curr->stride;
insert(queue, curr);
```
- Exactly proportional each cycle. Downside: **global state** — a new job's pass? (set to 0 → monopolizes). Lottery has no such problem.

## 7. Linux Completely Fair Scheduler (CFS)
- Goal: fairly divide the CPU, **very efficiently** (scheduling ~5% of datacenter CPU time).
- Each process accumulates **vruntime** while running; pick the process with **lowest vruntime**.
- **sched_latency** (e.g. 48ms): time slice = sched_latency / n.
- **min_granularity** (e.g. 6ms): lower bound on slice so too many processes don't cause too much switching.
- Uses a periodic timer interrupt (e.g. 1ms) → slices may not be perfectly precise.

### Weighting (niceness)
- `nice` from −20 to +19 (default 0); negative = higher priority. Mapped to **weights** (nice 0 → 1024, nice −5 → 3121, ...).
```
time_slice_k = weight_k / Σ weight_i  × sched_latency
vruntime_i  = vruntime_i + weight_0 / weight_i × runtime_i
```
- Table is constructed so **ratios** are preserved: the difference in nice values matters, not the absolute values.

### Red-Black Trees
- Running/runnable processes kept in a **red-black tree** keyed by vruntime → O(log n) insert/delete, find min quickly.
- Sleeping processes are removed from the tree.

### Dealing with I/O and Sleep
- A job that slept a long time would have tiny vruntime and monopolize the CPU on wake-up.
- CFS sets the woken job's vruntime to the **minimum value in the tree**. Cost: jobs that sleep briefly often may not get their fair share.

> ⚠️ Proportional share doesn't mesh well with I/O, and ticket/priority assignment is hard → not general purpose; best when shares are easy to define (e.g., VM CPU shares in a virtualized datacenter).
