# CH43 - Log-structured File Systems (LFS)

## Big Picture
- Rosenblum & Ousterhout (Berkeley, early 1990s). Motivations:
  - **Memory growing** → more reads served by cache → disk traffic dominated by **writes**.
  - Huge gap between **random and sequential** I/O performance.
  - Existing FSs (FFS) do many small writes for a simple operation (inode, bitmap, dir, data...).
  - FSs are not **RAID-aware** (small-write problem).
- Idea: **buffer all updates** (data + metadata) in memory into a **segment**, write it **sequentially** to free space. **Never overwrite in place**.

---

## 1. Writing Sequentially
```
| D | I |      ← data block at A0, inode pointing to A0, written together
```
- Not enough: writes to consecutive addresses separated in time miss a rotation.

## 2. Writing Effectively: Write Buffering
- Collect updates into a large **segment** (e.g., a few MB) and write it in one go.
```
| D[j,0] | D[j,1] | D[j,2] | D[j,3] | Inode[j] | D[k,0] | Inode[k] |
```

## 3. How Much to Buffer?
- Positioning overhead T_position, transfer rate R_peak, target fraction F of peak:
```
D = F / (1 − F) × R_peak × T_position
```
- 10 ms positioning, 100 MB/s, F = 90% → **9 MB** per segment; 99% → 99 MB.

## 4. Problem: Finding Inodes
- Inodes are scattered all over the log and keep moving.
- **Inode map (imap)**: inode number → latest inode disk address.
- imap itself written **into the log** next to the new inodes:
```
| D | I[k] | imap |
```

## 5. Checkpoint Region (CR)
- Fixed location on disk with pointers to the latest **imap pieces**.
- Updated only periodically (~every 30 s) → no big performance hit.

## 6. Reading a File from Disk
- Read CR → read entire imap into memory (cache it) → then a read works like a normal FS: imap lookup → inode → data.
- Same number of I/Os as a typical FS when imap is cached.

## 7. What About Directories?
- Directory data = (name, inode number) pairs, written to the log like files.
- **Recursive update problem**: a moved inode would require updating its directory, then the parent... → avoided because directories store **inode numbers** and the **imap** handles the indirection.

## 8. Garbage Collection
- Old versions of data/inodes become **garbage**.
- (Alternatively keep old versions → **versioning file systems**.)
- **Cleaner** works **segment-by-segment**: read M old segments, keep **live** blocks, write them compactly into N new segments (N < M), free the M.

### Determining Block Liveness
- **Segment summary block** (at the segment head): for each data block, its inode number and offset.
```c
(N, T) = SegmentSummary[A];
inode  = Read(imap[N]);
if (inode[T] == A)  // block D is alive
else                // block D is garbage
```
- Shortcut: **version number** per file in the imap and summary → mismatch → dead immediately.

### Which Blocks to Clean, and When?
- When: periodically, during idle time, or when the disk is full.
- Which: **hot** segments (frequently overwritten) → wait longer before cleaning; **cold** segments → clean sooner.

## 9. Crash Recovery and the Log
- CR update is atomic-ish: **two CRs** at both ends of the disk, alternately written, with timestamps at head and tail → pick the most recent consistent one.
- Segments written after the last CR: **roll forward** — read the log from the last checkpoint, recover valid updates.

## 10. Legacy
- Ideas live on in **WAFL** (NetApp; turns old versions into **snapshots**), **ZFS**, **btrfs**, and **flash FTLs** (log-structured).

> 💡 Turning flaws into virtues: LFS's garbage (old versions) became a feature — snapshots.
