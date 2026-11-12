# CH39 - Interlude: Files and Directories

## Big Picture
- Persistent storage virtualized via two abstractions:
  - **File**: linear array of bytes; low-level name = **inode number**.
  - **Directory**: list of (user-readable name, inode number) pairs.
- **Directory tree / hierarchy** from the **root** `/`; **absolute pathname** e.g. `/foo/bar.txt`.

---

## 1. Creating Files
```c
int fd = open("foo", O_CREAT | O_WRONLY | O_TRUNC, S_IRUSR | S_IWUSR);
```
- `O_CREAT` create if missing, `O_WRONLY` write only, `O_TRUNC` truncate to 0 bytes. Permissions: owner read+write.
- **File descriptor**: private per-process integer, a **capability** to perform operations. Kept in `struct proc`'s `ofile[NOFILE]` array.

## 2. Reading and Writing
```
prompt> strace cat foo
open("foo", O_RDONLY|O_LARGEFILE) = 3
read(3, "hello\n", 4096)           = 6
write(1, "hello\n", 6)             = 6
read(3, "", 4096)                  = 0     ← EOF
close(3)                           = 0
```
- fds 0, 1, 2 already open (stdin, stdout, stderr) → first `open` returns **3**.
- **`strace`** traces system calls.

## 3. Non-Sequential: lseek()
```c
off_t lseek(int fildes, off_t offset, int whence);  // SEEK_SET, SEEK_CUR, SEEK_END
```
- Each open file has a **current offset**, updated by read/write or set by `lseek`.
- `lseek` does **not** perform a disk seek — it only changes a variable.

## 4. Open File Table
```c
struct file { int ref; char readable; char writable; struct inode *ip; uint off; };
struct { struct spinlock lock; struct file file[NFILE]; } ftable;   // system-wide
```
- Each `open()` of the same file → separate entry with **independent offsets**.
- **Shared entries** (`ref` count > 1):
  - `fork()`: child shares parent's open file entries → shared offset.
  - `dup()` / `dup2()`: new fd referring to the **same** open file (used for shell redirection).

## 5. fsync()
- `write()` is buffered (5–30 s) for performance.
- **`fsync(fd)`**: force all dirty data of the file to disk. Also **fsync the directory** when creating a file to ensure the entry is durable.

## 6. Renaming
- `rename(old, new)` is **atomic** w.r.t. crashes. Editor pattern:
```c
int fd = open("foo.txt.tmp", O_WRONLY|O_CREAT|O_TRUNC, S_IRUSR|S_IWUSR);
write(fd, buffer, size);
fsync(fd);
close(fd);
rename("foo.txt.tmp", "foo.txt");   // atomically swap in the new version
```

## 7. Metadata: stat()
```c
struct stat {
    dev_t st_dev; ino_t st_ino; mode_t st_mode; nlink_t st_nlink;
    uid_t st_uid; gid_t st_gid; off_t st_size; blksize_t st_blksize; blkcnt_t st_blocks;
    time_t st_atime; time_t st_mtime; time_t st_ctime;
};
```
- Stored in the **inode**.

## 8. Removing Files
- `rm` calls **`unlink()`** — removes a name ↔ inode link (see hard links).

## 9. Directories
- `mkdir()`: new directory has two entries — `.` (itself) and `..` (parent).
- Can't write directories directly (FS metadata) — only via create/delete/rename.
- Reading: `opendir()`, `readdir()`, `closedir()`; `struct dirent { char d_name[256]; ino_t d_ino; off_t d_off; unsigned short d_reclen; unsigned char d_type; }`.
- `rmdir()`: only if **empty** (only `.` and `..`).

## 10. Hard Links
```
prompt> ln file file2      # link(): new name → same inode
prompt> ls -i file file2
67158084 file
67158084 file2
```
- Creating a file = make an inode + **link** a name to it.
- Inode keeps a **reference / link count**; `unlink()` decrements; at **0** the inode and data blocks are freed.
- Can't hard-link **directories** (cycles) or across **file systems** (inode numbers aren't unique).

## 11. Symbolic (Soft) Links
```
prompt> ln -s file file2
```
- A different **file type**: holds the **pathname** of the target.
- Size = length of the pathname. Removing the target → **dangling reference**.

## 12. Permission Bits and ACLs
```
-rw-r--r--  1 remzi wheel  0 Aug 24 16:29 foo.txt
```
- Type (`-` file, `d` dir, `l` symlink) + **owner / group / other** × **r w x**; changed with `chmod` (e.g. `chmod 600`).
- Directory `x` = can `cd` into it (and with `w`, create files).
- **ACLs** (AFS etc.): explicit per-user lists.
- **TOCTTOU** (Time Of Check To Time Of Use) bug: attacker swaps a file (e.g., symlink) between check and use. Mitigation: `O_NOFOLLOW`, transactional FS.

## 13. Making and Mounting a File System
- **`mkfs`**: write an empty FS (root dir) onto a device/partition.
- **`mount`**: attach it at a **mount point** in the existing tree → unified hierarchy.
```
prompt> mount -t ext3 /dev/sda1 /home/users
```
