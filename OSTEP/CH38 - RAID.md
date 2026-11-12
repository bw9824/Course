# CH38 - Redundant Arrays of Inexpensive Disks (RAID)

## Big Picture
- **RAID**: use multiple disks together to build a faster, bigger, more reliable disk.
- **Transparent**: looks like one big disk to the host (deployable without software changes).
- Evaluate on **capacity**, **reliability**, **performance**.

---

## 1. Interface and Internals
- Logical I/O → RAID computes which physical disk(s) to access → issues physical I/Os.
- Hardware RAID: microcontroller, DRAM buffer, sometimes **NVRAM**, parity logic → a specialized computer.

## 2. Fault Model
- **Fail-stop**: a disk is either working or completely failed; failure is immediately detected.
- (Silent corruption, latent sector errors → later chapter.)

## 3. Evaluation
- N disks, each B blocks. Single-disk sequential bandwidth **S** MB/s, random **R** MB/s (S ≫ R).
- Example: 10 MB sequential, 7ms seek, 3ms rotation, 50 MB/s → S = 47.62 MB/s; 10 KB random → R = 0.981 MB/s.

## 4. RAID-0: Striping
```
Disk 0  Disk 1  Disk 2  Disk 3
  0       1       2       3      ← stripe
  4       5       6       7
```
- **Chunk size**: small → more parallelism for one file, more positioning time; big → less intra-file parallelism.
- Mapping: `Disk = A % N`, `Offset = A / N`.
- Capacity **N·B**; reliability: any disk failure → data loss.

## 5. RAID-1: Mirroring
```
Disk 0  Disk 1  Disk 2  Disk 3
  0       0       1       1
  2       2       3       3
```
- RAID-10 (stripe of mirrors) vs RAID-01.
- Capacity **N·B/2**; tolerates 1 failure for sure (up to N/2 with luck).
- Write must update **both** copies; not atomic → **consistent-update problem** → use a **write-ahead log** in NVRAM.

## 6. RAID-4: Parity
```
Disk 0  Disk 1  Disk 2  Disk 3  Disk 4
  0       1       2       3      P0
  4       5       6       7      P1
```
- Parity = **XOR** of the stripe's data blocks (even number of 1s).
- Reconstruct a lost block: XOR the remaining blocks + parity.
- **Small write** update:
  - **Additive**: read all other data blocks, recompute parity.
  - **Subtractive**: `P_new = (C_old ⊕ C_new) ⊕ P_old` → 2 reads + 2 writes.
- ❌ **Small-write problem**: parity disk is a **bottleneck** — all writes serialize on it.
- Capacity **(N−1)·B**; tolerates 1 failure.

## 7. RAID-5: Rotating Parity
```
Disk 0  Disk 1  Disk 2  Disk 3  Disk 4
  0       1       2       3      P0
  5       6       7      P1       4
 10      11      P2       8       9
```
- Parity spread across disks → removes the bottleneck → random writes in parallel.

## 8. Comparison
| | RAID-0 | RAID-1 | RAID-4 | RAID-5 |
|---|---|---|---|---|
| Capacity | N·B | N·B/2 | (N−1)·B | (N−1)·B |
| Reliability | 0 | 1 (for sure), N/2 (if lucky) | 1 | 1 |
| Sequential read | N·S | (N/2)·S | (N−1)·S | (N−1)·S |
| Sequential write | N·S | (N/2)·S | (N−1)·S | (N−1)·S |
| Random read | N·R | N·R | (N−1)·R | N·R |
| Random write | N·R | (N/2)·R | **½·R** | **(N/4)·R** |
| Latency read | T | T | T | T |
| Latency write | T | T | 2T | 2T |

- Mirroring halves sequential read bandwidth: each disk skips the blocks its mirror serves (head still rotates past them).
- Rules of thumb: performance only → RAID-0; random I/O + reliability → RAID-1; capacity + reliability → RAID-5.

## 9. Other Topics
- RAID-2/3, RAID-6 (2 failures); **hot spares**; rebuilding; latent faults; software RAID.
