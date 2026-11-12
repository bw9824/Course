# CH22 - Beyond Physical Memory: Policies

## Big Picture
- Memory under pressure → OS must choose **which page to evict**: the **replacement policy**.
- Main memory = a **cache** of the virtual pages; goal: minimize **cache misses**.

---

## 1. Cache Management
- **AMAT = T_M + (P_Miss × T_D)**
  - T_M: memory access cost (~100ns), T_D: disk access cost (~10ms), P_Miss: miss rate.
- Example: 90% hit rate → AMAT ≈ 1ms; 99.9% → 10.1µs. Disk cost **dominates** — even a tiny miss rate matters.

## 2. Optimal Replacement (Belady's MIN)
- Evict the page that will be accessed **furthest in the future**.
- Not implementable (needs the future) but a **baseline** for comparison.
- Trace 0 1 2 0 1 3 0 3 1 2 1, cache size 3 → 6 hits, 5 misses (3 **compulsory/cold-start** misses).

| Miss type (3 C's) | Cause |
|---|---|
| Compulsory | First reference to the item |
| Capacity | Cache too small, had to evict |
| Conflict | Hardware set-associativity limits (not in OS page cache — fully associative) |

## 3. Simple Policies
| Policy | Idea | Same trace hit rate |
|---|---|---|
| **FIFO** | Evict the oldest-arrived page | 36.4% (vs 54.5% optimal) |
| **Random** | Evict a random page | Varies; sometimes optimal, sometimes bad |

### Belady's Anomaly
- FIFO trace 1,2,3,4,1,2,5,1,2,3,4,5: cache 3 → 9 misses, cache 4 → **10** misses — bigger cache, worse hit rate.
- **Stack property** policies (LRU) don't suffer this: a cache of size N+1 always contains the contents of size N.

## 4. Using History: LRU / LFU
- **Principle of locality**: spatial and temporal.
- **LRU** (Least Recently Used): evict the page not used for the longest → matches optimal on the example trace.
- **LFU** (Least Frequently Used).
- Opposites MRU / MFU usually perform badly.

## 5. Workload Examples (100 unique pages, 10,000 accesses)
| Workload | Result |
|---|---|
| **No locality** (random) | All policies (except optimal) the same; hit rate depends only on cache size |
| **80-20** (80% refs to 20% hot pages) | LRU better than FIFO/Random; optimal best |
| **Looping-sequential** (0→49 repeatedly) | **LRU and FIFO worst case**: 0% hits with cache < 50; Random does fine |

## 6. Implementing Historical Algorithms
- Perfect LRU needs updating a data structure on **every memory reference** — too costly.
- Hardware help: a time field per frame → still need to scan all pages (4GB / 4KB = 1M pages).

## 7. Approximating LRU: Clock Algorithm
- Hardware **use (reference) bit** per page, set to 1 on access; OS clears it.
- **Clock**: pages in a circle, a hand points to one:
  - use bit 1 → clear to 0, advance.
  - use bit 0 → evict it.
- Performs close to LRU, much cheaper.

## 8. Considering Dirty Pages
- Evicting a **dirty** (modified) page requires writing it to disk; clean pages are free to evict.
- Clock variant: prefer pages that are **unused and clean**, then unused dirty, etc. Hardware provides a **dirty / modified bit**.

## 9. Other VM Policies
- **Page selection**: when to bring a page in — **demand paging** (on access) or **prefetching** (e.g., page P+1 with P).
- **Clustering / grouping** writes to disk.

## 10. Thrashing
- Memory **oversubscribed**: working sets of running processes exceed memory → constant paging.
- **Admission control**: run a subset of processes whose **working sets** fit.
- Linux **out-of-memory killer**: kill a memory-intensive process (may kill X server!).

> 💡 Best solution to excessive paging is often simple: **buy more memory**.
