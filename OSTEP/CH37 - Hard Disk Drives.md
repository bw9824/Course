# CH37 - Hard Disk Drives

## Big Picture
- HDDs have been the main persistent storage for decades; file-system design depends on how they behave.
- Performance dominated by **mechanical** costs: **seek** and **rotation**.
- Key lesson: **sequential access ≫ random access**; schedule I/Os to minimize positioning time.

---

## 1. The Interface
- Disk = array of **sectors** (512 bytes), addressed **0 to n−1**.
- Only a single 512-byte write is **atomic** (guaranteed). Power loss mid large write → **torn write**.
- Unwritten contract: nearby blocks are faster to access than far ones; **sequential** is fastest.

## 2. Basic Geometry
| Part | Meaning |
|---|---|
| **Platter** | Circular hard surface, magnetic; two **surfaces** |
| **Spindle** | Rotates platters at fixed **RPM** (7,200 – 15,000) |
| **Track** | Concentric circle of sectors on a surface |
| **Disk head / arm** | Reads/writes; one head per surface, all move together |

- 10,000 RPM → one rotation = **6 ms**.

## 3. A Simple Disk Drive
- **Rotational delay**: wait for the sector to rotate under the head (avg R/2).
- **Seek**: move the arm to the right track — acceleration → coasting → deceleration → **settling** (0.5–2 ms).
- **Transfer**: read/write as the sector passes.

### Other Details
- **Track skew**: sectors offset across tracks so sequential reads crossing a track boundary don't miss the next block after a head switch.
- **Multi-zoned**: outer tracks have more sectors.
- **Track buffer / cache** (8–16 MB): read whole track; write caching:
  - **Write-back** (immediate reporting): ack when in cache — fast but dangerous for ordering.
  - **Write-through**: ack after written to disk.

## 4. I/O Time
```
T_I/O = T_seek + T_rotation + T_transfer
R_I/O = Size_transfer / T_I/O
```
| | Cheetah 15K.5 | Barracuda |
|---|---|---|
| Capacity | 300 GB | 1 TB |
| RPM | 15,000 | 7,200 |
| Avg seek | 4 ms | 9 ms |
| Max transfer | 125 MB/s | 105 MB/s |

| Workload | Cheetah | Barracuda |
|---|---|---|
| **Random** 4KB: T_I/O | 6 ms (4 + 2 + 0.03) | 13.2 ms |
| Random R_I/O | **0.66 MB/s** | **0.31 MB/s** |
| **Sequential** 100MB: R_I/O | **125 MB/s** | **105 MB/s** |

- ~200–300× difference → **transfer data sequentially** whenever possible.
- Average seek ≈ **1/3 of full seek** (derived from average distance N/3).

## 5. Disk Scheduling
- Disk scheduler can estimate each request's time → **greedy SJF-like**.

| Algorithm | Idea | Problem |
|---|---|---|
| **SSTF** (Shortest Seek Time First) | Pick request on nearest track | OS doesn't know geometry → use **NBF** (nearest block first); **starvation** |
| **SCAN (elevator)** | Sweep across tracks in one direction, serve requests in order | Ignores rotation |
| F-SCAN | Freeze queue during a sweep | |
| **C-SCAN** | Sweep only outer → inner, then reset | Fairer to inner/outer tracks |
| **SPTF / SATF** (Shortest Positioning Time First) | Consider seek **and** rotation | Done inside the drive (knows geometry) |

- Example: head on middle track at sector 30, requests 16 (middle track) vs 8 (outer): if seek ≫ rotation → SSTF fine; if seek ≈ rotation, the farther track may be faster.

## 6. Other Scheduling Issues
- Modern: OS picks a few best requests, the **disk** schedules them internally (SPTF).
- **I/O merging**: requests for blocks 33, 8, 34 → merge 33 + 34.
- **Work-conserving** (issue immediately) vs **non-work-conserving** (**anticipatory scheduling**: wait a bit for a better request to arrive).
