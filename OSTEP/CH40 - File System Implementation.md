# CH40 - File System Implementation

## Big Picture
- **vsfs** (Very Simple File System): a simplified UNIX file system.
- Two aspects: **data structures** (on-disk layout) and **access methods** (how open/read/write map onto those structures).
- Pure software — no hardware help needed.

---

## 1. Overall Organization
- Disk divided into **blocks** (4 KB). Tiny example: 64 blocks.
```
| S | i | d | I I I I I | D D D ... D (56 data blocks) |
  0   1   2   3 ...  7    8 ... 63
```
| Region | Meaning |
|---|---|
| **S** — Superblock | FS info: # inodes, # data blocks, inode table start, magic number |
| **i** — Inode bitmap | Free/used inodes |
| **d** — Data bitmap | Free/used data blocks |
| **I** — Inode table | 5 blocks × 16 inodes (256B each) = 80 inodes max |
| **D** — Data region | User data |
- Mounting: OS reads the superblock first.

## 2. The Inode (index node)
- Referred to by **i-number**. Location computed directly:
```
blk    = (inumber * sizeof(inode_t)) / blockSize;
sector = ((blk * blockSize) + inodeStartAddr) / sectorSize;
```
- E.g., inode 32 → offset 32 × 256 = 8KB into the inode table → 12KB + 8KB = 20KB.

| ext2 inode field | Meaning |
|---|---|
| mode | Read/write/execute permission, file type |
| uid, gid | Owner, group |
| size | Bytes in file |
| time, ctime, mtime, dtime | Access, create, modify, delete times |
| links_count | Hard links |
| blocks | Number of blocks |
| block[15] | Disk pointers (12 direct + indirect) |

### Multi-Level Index
- **Direct pointers** (e.g., 12) → data blocks.
- **Indirect pointer** → a block full of pointers (4KB / 4B = 1024 pointers).
  - Max size = (12 + 1024) × 4K = **4144 KB**.
- **Double indirect** → 1024² more blocks (~4 GB); **triple indirect** → ~4 TB.
- Imbalanced tree, because **most files are small** (~2KB typical); a few big files use most space.
- Alternatives: **extents** (pointer + length; ext4, XFS) — compact but less flexible; **linked list / FAT** (DOS: in-memory table of next pointers; no hard links).

## 3. Directory Organization
```
inum | reclen | strlen | name
 5      12        2     .
 2      12        3     ..
 12     12        4     foo
 13     12        4     bar
 24     36       28     foobar_is_a_pretty_longname
```
- Directory = a special file (type in inode = directory), data blocks hold entries.
- Deleting: mark inum 0 (reserved); **reclen** allows reusing space.
- XFS uses B-trees for large directories.

## 4. Free Space Management
- **Bitmaps** (one for inodes, one for data). Some FSs use free lists or B-trees.
- **Pre-allocation**: ext2/ext3 look for a run of free blocks (e.g., 8) to keep files contiguous.

## 5. Access Paths: Reading a File
- `open("/foo/bar", O_RDONLY)`: **traverse the pathname**:
  - Read **root inode** (well-known i-number **2**) → read root dir data → find `foo` → read foo's inode → foo's data → find `bar` → read bar's inode → permission check → fd in per-process table.
- `read()`: read inode, read data block, **write inode** (update last-access time), update offset.

| Operation | data bitmap | inode bitmap | root inode | foo inode | bar inode | root data | foo data | bar data |
|---|---|---|---|---|---|---|---|---|
| open(bar) | | | read | read | read | read | read | |
| read() | | | | | read, write | | | read |

- I/Os proportional to the **length of the path**; big directories may need many block reads.

## 6. Writing to Disk
- Each write that **allocates** a block: read data bitmap, write bitmap, read inode, write inode, write data → **5 I/Os**.
- **Creating** a file (`open` with O_CREAT): read inode bitmap, write inode bitmap, write new inode (init), write directory data, read/write directory inode (+ more if the directory grows) → ~10 I/Os.

## 7. Caching and Buffering
- Read path: cache popular blocks in DRAM. Early: **fixed-size** cache (~10% of memory) → static partitioning.
- Modern: **unified page cache** — dynamic partitioning between VM pages and FS pages.
- Write buffering: **delay writes** (5–30 s):
  - Batch updates (e.g., multiple bitmap updates → 1 I/O).
  - Schedule I/Os better.
  - **Avoid** writes entirely (file created then deleted).
- Trade-off: **durability vs performance**; databases use `fsync()`, **direct I/O**, or raw disk.

> 💡 Static vs dynamic partitioning: static is predictable; dynamic achieves better utilization.
