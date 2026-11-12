# CH42 - Crash Consistency: FSCK and Journaling

## Big Picture
- FS structures must persist despite **power loss** or **system crashes** mid-update.
- Crux: disk only commits one write at a time → crash between writes leaves on-disk state **inconsistent**.
- Solutions: **fsck** (fix after the fact), **journaling / write-ahead logging** (ext3/ext4, NTFS, XFS), others (soft updates, COW, backpointers, optimistic).

---

## 1. Example: Appending a Block
- Append one data block to a file → three writes:
  1. **Inode** (I[v2]): new pointer, new size.
  2. **Data bitmap** (B[v2]): mark block allocated.
  3. **Data block** (Db).

### Single-Write Crash Scenarios
| Only this written | Result |
|---|---|
| Db | Data on disk but no inode points to it → as if never written (fine for FS consistency) |
| I[v2] | Inode points to garbage; bitmap says free → **inconsistency** |
| B[v2] | Bitmap says used, nobody uses it → **space leak** |

### Two-Write Crash Scenarios
| Written | Result |
|---|---|
| I[v2] + B[v2] | Metadata consistent, but block contains **garbage** |
| I[v2] + Db | Inode/bitmap disagree |
| B[v2] + Db | Leak: block never used |

- Goal: move the FS **atomically** from one consistent state to another (**crash-consistency problem**).

## 2. Solution #1: fsck (File System Checker)
- Run before mounting after a crash; scans everything:

| Check | What |
|---|---|
| Superblock | Sanity (size > blocks allocated) |
| Free blocks | Scan inodes/indirect blocks → rebuild bitmaps (trust inodes) |
| Inode state | Valid type? Clear suspicious inodes |
| Inode links | Recount references by walking the tree → fix link counts; unreferenced → `lost+found` |
| Duplicates | Two inodes pointing to the same block → clear one or copy |
| Bad blocks | Pointers outside the valid range → clear |
| Directory checks | `.` and `..` first, valid inodes, no dir linked twice |

- ❌ **Too slow**: time proportional to disk size (hours for big RAID); wasteful (scan everything to fix 3 blocks).
- ❌ Can't fix "garbage data" case.

## 3. Solution #2: Journaling (Write-Ahead Logging)
- Before updating structures in place, write a note describing the update to a **log (journal)** → after a crash, **replay** it.
- ext3 layout: `| Super | Journal | Group 0 | Group 1 | ... |`

### Data Journaling
```
Journal: | TxB | I[v2] | B[v2] | Db | TxE |
```
- **TxB**: transaction begin (incl. **transaction ID**, final addresses).
- **TxE**: transaction end (commit).
- Logged contents = exact **physical** blocks (**physical logging**); alternative: **logical logging**.
- Then **checkpoint**: write the blocks to their final locations.

### Writing the Journal Safely
- Writing all 5 blocks at once: disk may reorder → TxE written before Db → replay garbage.
- Protocol:
  1. **Journal write**: TxB + contents, wait.
  2. **Journal commit**: write TxE (one **512-byte sector** → atomic), wait.
  3. **Checkpoint**: write to final locations.
- Optimization (ext4): **checksum** of transaction in TxB/TxE → write all at once.
- Disk write caches: need **write barriers** / cache flush (but some disks lie).

### Recovery
- Scan the log; **replay** committed transactions (**redo logging**). Uncommitted → skip.
- Replaying twice is fine (idempotent). Recovery is fast.

### Batching Log Updates
- Creating two files in the same directory updates the same blocks → buffer updates in a **global transaction**, commit every few seconds.

### Making the Log Finite
- **Circular log**; after checkpoint, mark transaction **free** in the **journal superblock**.
1. Journal write → 2. Journal commit → 3. Checkpoint → 4. **Free**.

## 4. Metadata (Ordered) Journaling
- Data journaling writes every data block **twice** → halves write bandwidth.
- **Ordered mode** (ext3 default): journal only **metadata**:
```
Journal: | TxB | I[v2] | B[v2] | TxE |
```
- Rule: **write data before metadata** → data write → journal metadata → commit → checkpoint metadata → free.
  - "Write the pointed-to object before the object that points to it."
- **Unordered (writeback) mode**: data any time → possible garbage after crash.
- Steps 1 (data write) and journal metadata write can be issued concurrently; must complete before commit.

## 5. Tricky Case: Block Reuse
- Directory block (metadata, journaled) freed and reused by a file (data, not journaled) → replay overwrites the new file's data with the old directory contents.
- ext3 fix: **revoke records** — replay skips revoked blocks.

## 6. Timelines
| Data journaling | Metadata journaling |
|---|---|
| TxB, contents (metadata + data) issue → complete | TxB, metadata issue; **data** issue → all complete |
| TxE issue → complete | TxE issue → complete |
| Checkpoint metadata + data | Checkpoint metadata |

## 7. Other Approaches
| Approach | Idea |
|---|---|
| **Soft Updates** (FFS) | Carefully **order** all writes so on-disk state is never inconsistent; complex, needs FS knowledge |
| **Copy-On-Write (COW)** | Never overwrite in place; write new copies, then flip a root pointer (ZFS, LFS) |
| **Backpointer-based consistency** (BBC) | Each data block points back to its inode; check both directions |
| **Optimistic crash consistency** | Issue as many writes as possible, use checksums + **asynchronous durability notifications** |
