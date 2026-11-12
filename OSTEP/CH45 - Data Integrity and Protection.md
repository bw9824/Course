# CH45 - Data Integrity and Protection

## Big Picture
- Beyond fail-stop: modern disks fail **partially** — a few blocks become unreadable or **silently corrupt**.
- Crux: ensure the data we read is the data we wrote, despite unreliable storage.
- Tools: **redundancy** (for latent sector errors), **checksums** (for corruption), plus physical IDs, scrubbing.

---

## 1. Disk Failure Modes
- **Fail-partial** model (Bairavasundaram et al.):
  - **LSE (Latent Sector Error)**: sector damaged (head crash, cosmic rays); detected by in-disk **ECC** → read returns an error.
  - **Block corruption**: wrong contents returned **silently** (buggy firmware, bad bus transfer, **misdirected write**, **lost write**).

| | Cheap (SATA) | Costly (SCSI/FC) |
|---|---|---|
| LSEs | 9.40% of drives | 1.40% |
| Corruption | 0.50% | 0.05% |

- Findings: costly drives better but not perfect; errors increase with age and size; LSEs/corruption show **spatial and temporal locality**; **disk scrubbing** useful.

## 2. Handling Latent Sector Errors
- Easy to detect (disk reports error) → use **redundancy** (mirror / RAID parity) to reconstruct.
- RAID-4/5 problem: a full-disk failure + an LSE during **reconstruction** → data loss. Fix: **RAID-DP** (NetApp, two parity disks).

## 3. Detecting Corruption: The Checksum
- **Checksum**: small summary (4–8 bytes) of a data block; store, then recompute on read and compare.

| Function | How | Strength / cost |
|---|---|---|
| **XOR** | XOR every 4-byte chunk | Cheap; misses e.g. two flipped bits in the same position |
| **Addition** (2's complement, ignore overflow) | Sum chunks | Fast; misses shifted data |
| **Fletcher** | s1 = (s1 + d_i) mod 255; s2 = (s2 + s1) mod 255 | Detects all single-bit, double-bit, many burst errors |
| **CRC** (cyclic redundancy check) | Treat block as big binary number, divide by k, remainder | Efficient in hardware; widely used in networking |

- **Collisions** are always possible (summary is smaller than data); goal: make likely errors detectable.

### Checksum Layout
```
| C[D0] | D0 | C[D1] | D1 | ...       ← 520-byte sectors (512 + 8) on some drives
| C[D0] C[D1] C[D2] ... | D0 | D1 | D2 | ...   ← packed checksum sector (extra write on update)
```

### Using Checksums
- Read block D → read stored checksum Cs(D) → compute Cc(D) → mismatch → corruption → use a redundant copy or return error.

## 4. Misdirected Writes
- Disk writes correct data to the **wrong location** (wrong disk or wrong block).
- Fix: add a **physical identifier (physical ID)** to the checksum: disk number + block offset.
```
| C[D0] disk=1 block=0 | D0 | C[D1] disk=1 block=1 | D1 | ...
```

## 5. Lost Writes
- Device reports success but the write never persisted → old block with **valid** checksum.
- Fixes: **write verify / read-after-write** (slow, doubles I/O); checksums stored elsewhere (e.g., in the inode / parent pointer — **ZFS** keeps checksums in pointers, so a stale block won't match).

## 6. Scrubbing
- Checking only on access leaves rarely-read data unchecked → **disk scrubbing**: periodically read every block and verify checksums (nightly/weekly).

## 7. Overheads
- **Space**: e.g., 8 bytes per 4 KB block = 0.19% — small.
- **Time**: CPU to compute on every read/write; I/O for separately stored checksums and scrubbing. Combine copying and checksumming into one pass; schedule scrubbing off-peak.

> 💡 No free lunch: more protection costs space and time; pick the level of protection the system needs.
