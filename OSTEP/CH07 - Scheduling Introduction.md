# CH07 - Scheduling: Introduction

## Big Picture
- LDE gives the **mechanism**; now the **policy**: which process runs next?
- Build up from unrealistic **workload assumptions**, relaxing them one by one.
- Two metric families pull against each other: **turnaround time** (performance) vs. **response time** / fairness.

---

## 1. Workload Assumptions (to be relaxed)
1. Each job runs for the **same amount of time**.
2. All jobs **arrive at the same time**.
3. Once started, each job **runs to completion**.
4. All jobs **only use the CPU** (no I/O).
5. The **run-time of each job is known**.

## 2. Metrics
- **Turnaround time**: `T_turnaround = T_completion − T_arrival`
- **Response time**: `T_response = T_firstrun − T_arrival`
- Fairness (e.g., Jain's Fairness Index) often trades off against performance.

## 3. FIFO (First In, First Out)
- A, B, C each 10s, arrive at 0 → finish at 10, 20, 30 → avg turnaround **20**.
- Relax (1): A = 100s, B = C = 10s → finish 100, 110, 120 → avg **110**.
- **Convoy effect**: short jobs stuck behind a heavyweight one (like a grocery line).

## 4. SJF (Shortest Job First)
- Run the shortest first: B, C, A → finish 10, 20, 120 → avg **50**.
- **Optimal** for avg turnaround if all jobs arrive at once.
- Relax (2): A arrives at 0 (100s), B and C at 10 → still convoy (A is already running, SJF is **non-preemptive**) → avg (100 + 100 + 110)/3 = **103.33**.

## 5. STCF (Shortest Time-to-Completion First) / PSJF
- Relax (3): **preemptive** — when a new job arrives, run the one with least remaining time.
- Same example: A runs 0–10, B and C run 10–30, A finishes at 120 → avg (120 + 10 + 20)/3 = **50**.
- Optimal for turnaround given the assumptions.

## 6. Response Time and Round Robin (RR)
- Time-shared users want **interactivity** → STCF is bad for response time.
- **RR**: run each job for a **time slice** (scheduling **quantum**, multiple of timer interrupt period), then switch.
- A, B, C (5s each) arrive at 0, 1s slice: avg response **1** (RR) vs **5** (SJF).
- Time slice length trade-off:
  - Shorter → better response time, but **context-switch cost** dominates (also cache/TLB/branch predictor state flushed).
  - Longer → **amortize** the switch cost.
- RR is great for response time but **among the worst for turnaround** (stretches every job out). Fair policies hurt turnaround — an inherent trade-off.

| Policy | Turnaround | Response |
|---|---|---|
| SJF / STCF | Good | Bad |
| RR | Bad | Good |

## 7. Incorporating I/O
- Relax (4): a job issuing I/O is **blocked**; scheduler runs another job; when I/O completes, job becomes ready.
- Example: A = 50ms CPU in 10ms bursts with I/O between; B = 50ms pure CPU.
  - Treat each 10ms sub-job of A as an independent job → under STCF, A's sub-job runs, then B fills in while A does I/O → **overlap** → better utilization.

## 8. No More Oracle
- Assumption (5) is the worst: the OS rarely knows job length.
- Next chapter: **MLFQ** uses the **recent past to predict the future**.
