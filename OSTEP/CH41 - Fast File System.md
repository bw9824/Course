# CH41 - Locality and The Fast File System (FFS)

## Big Picture
- The **old UNIX file system** (Thompson) was simple but delivered only **2% of disk bandwidth**.
- Problem: it treated the disk like **random-access memory** → data scattered, expensive positioning.
- **FFS** (Berkeley, McKusick et al.): make the FS **disk aware** — same API, new on-disk structures/policies.

---

## 1. The Problem: Poor Performance
```
| S | Inodes | Data |
```
- Inodes far from data → seek per file access.
- **Fragmentation**: free list leads to a file's blocks spread across the disk.
```
A1 A2 B1 B2 C1 C2 D1 D2  → delete B, D →  A1 A2 __ __ C1 C2 __ __
new file E (4 blocks): A1 A2 E1 E2 C1 C2 E3 E4   ← not contiguous
```
- Small block size (**512 B**): little internal waste but bad transfer efficiency.
- **Defragmentation** tools reorganize data later.

## 2. FFS: Disk Awareness Is the Solution
- Divide the disk into **cylinder groups** (cylinder = tracks at the same distance from center across surfaces). Modern FSs (ext2/3/4) use **block groups**.
- Each group: **superblock copy**, **inode bitmap**, **data bitmap**, **inodes**, **data blocks**.
```
| S | ib | db | Inodes | Data |   (per group)
```

## 3. Policies: How to Allocate Files and Directories
- Mantra: **keep related stuff together** (and unrelated stuff far apart).
- **Directories**: place in a group with **few allocated directories** and **many free inodes** → balance.
- **Files**:
  1. Data blocks in the **same group as its inode**.
  2. Files in the same directory in the **same group as the directory**.
- Example: `/a/c`, `/a/d`, `/a/e`, `/b/f` → a's files together in one group, b's in another.

## 4. Measuring File Locality
- SEER traces: **path difference** (distance in the tree between consecutive accesses).
- Roughly 40% of accesses go to the same file or a file in the same directory (distance 0 or 1) → FFS's locality assumption holds for a large share of accesses; still, a good chunk has no such locality.

## 5. The Large-File Exception
- A big file would fill a whole group and hurt locality of related files.
- FFS: after the direct blocks (12), place each subsequent **chunk** in a **different group**.
- Cost: seeks between chunks, amortized by large chunks:
  - Goal: 50% of peak bandwidth with 10 ms positioning at 40 MB/s → chunk of **409.6 KB**; 90% → 3.69 MB; 99% → 40.6 MB.
- FFS actually used the **inode structure**: each indirect block's pointed-to blocks (4 MB) in a separate group.

## 6. Other FFS Innovations
| Innovation | Purpose |
|---|---|
| **Sub-blocks** (512 B) | Small files don't waste a whole 4KB block; later copied into a full block. Mostly avoided by buffering writes in libc |
| **Parameterized placement** | Skip every other block so the next sequential block isn't missed during processing; modern disks use **track buffers** instead |
| **Long file names** | Beyond 8 chars |
| **Symbolic links** | Introduced |
| **Atomic rename()** | Introduced |

> 💡 Make the system usable: FFS's usability improvements (long names, symlinks, rename) mattered as much as performance.
