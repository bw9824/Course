# CH08 - Scheduling: The Multi-Level Feedback Queue

## Big Picture
- **MLFQ** (Corbató, CTSS 1962, Turing Award): optimize **turnaround** (like SJF) and **response time** (like RR) **without knowing job lengths**.
- Learns from **history**: observes how a job behaves and adjusts its priority.

---

## 1. Basic Rules
- Several **queues**, each with a priority level. A ready job sits in exactly one queue.
- **Rule 1**: If Priority(A) > Priority(B), A runs.
- **Rule 2**: If Priority(A) = Priority(B), A & B run in **RR**.
- Priority set by **observed behavior**: relinquishes CPU often (waiting for keyboard) → high priority; uses CPU intensively → lower.

## 2. Attempt #1: Changing Priority
- **Rule 3**: A new job enters at the **highest** priority.
- **Rule 4a**: If it uses up a whole time slice → priority **reduced** (moves down one queue).
- **Rule 4b**: If it gives up the CPU before the slice ends → stays at the **same** priority.
- Effect: since MLFQ doesn't know if a job is short, it **assumes it is**; long jobs sink → approximates SJF.
- Interactive job (I/O-bound) stays high → good response time.

### Problems with Attempt #1
1. **Starvation**: too many interactive jobs → long-running jobs never get CPU.
2. **Gaming**: issue an I/O just before the slice ends → stay at top priority, monopolize CPU.
3. **Behavior changes**: a CPU-bound job may become interactive later, but is stuck at the bottom.

## 3. Attempt #2: Priority Boost
- **Rule 5**: After some period **S**, move **all** jobs to the topmost queue.
- Solves starvation and changing behavior.
- **S** is a **voo-doo constant** (Ousterhout): too high → starvation; too low → interactive jobs don't get a fair share.

## 4. Attempt #3: Better Accounting
- Replace 4a/4b with:
- **Rule 4**: Once a job uses up its **time allotment** at a level (regardless of how many times it gave up the CPU), its priority is reduced.
- Defeats gaming.

## 5. Tuning MLFQ
- How many queues? Slice per queue? How often to boost?
- Common: **high-priority queues → short slices** (interactive, 10ms or less); **low-priority → long slices** (CPU-bound, 100s of ms).
- **Solaris TS class**: table-driven (default 60 queues, slices 20ms → few hundred ms, boost ~ every 1s); admins can edit the table.
- **FreeBSD**: formula based on recent CPU usage (with **decay-usage**).
- Some reserve top priorities for OS work; some allow user **advice** (`nice`).

> 💡 Use **advice** where possible: the OS can't know everything; hints help (nice, madvise, informed prefetching).

## 6. Final MLFQ Rules
1. Priority(A) > Priority(B) → A runs (B doesn't).
2. Priority(A) = Priority(B) → A & B run in RR using the queue's time slice.
3. New job → topmost queue.
4. Job uses up its time allotment at a level → moves down one queue.
5. After period S → all jobs to topmost queue.

- Used (with variants) in BSD UNIX, Solaris, Windows NT and later.
