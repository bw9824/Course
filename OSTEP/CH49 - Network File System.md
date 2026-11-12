# CH49 - Sun's Network File System (NFS)

## Big Picture
- **Client/server distributed FS**: clients access files stored on a **server** over the network.
- Benefits: easy **sharing**, centralized **administration** (backup), **security**.
- **NFSv2** (Sun): open protocol; key design goal **simple and fast server crash recovery** → **stateless** protocol.

---

## 1. Basic Distributed FS
```
Client App → client-side FS → network → file server → server FS → disks
```
- **Transparent**: apps use normal `open()`/`read()`/`write()`.

## 2. Focus: Simple and Fast Server Crash Recovery
- Stateful protocols (e.g., server tracks open fds) break on server crash — server loses the state; client crashes leave state dangling.
- NFSv2: **stateless** — server keeps no info about clients; **every request contains everything needed** to complete it.

## 3. Key: File Handles
- **File handle** = volume ID + **inode number** + **generation number** (detects inode reuse).
```
NFSPROC_GETATTR  file handle → attributes
NFSPROC_SETATTR  file handle, attributes → –
NFSPROC_LOOKUP   directory fh, name → fh of file/dir, attributes
NFSPROC_READ     fh, offset, count → data, attributes
NFSPROC_WRITE    fh, offset, count, data → attributes
NFSPROC_CREATE   directory fh, name, attributes → –
NFSPROC_REMOVE   directory fh, name → –
NFSPROC_MKDIR    directory fh, name, attributes → fh, attributes
NFSPROC_RMDIR    directory fh, name → –
NFSPROC_READDIR  directory fh, count, cookie → directory entries, cookie
```
- Reads/writes carry the **offset** explicitly.

## 4. From Protocol to Distributed FS
- Client keeps the state: `open("/foo.txt")` → client sends `LOOKUP(root fh, "foo.txt")` → gets fh → client-side open-file table stores fh + current offset.
- `read(fd, buf, 4096)` → `READ(fh, offset, 4096)`; client advances offset.
- `close()` → nothing sent to server (in basic design).
- Long path `/home/remzi/foo.txt` → three LOOKUPs.

## 5. Handling Server Failure with Idempotent Operations
- Request lost / server down / reply lost → client just **retries** after timeout.
- Works because most operations are **idempotent** (doing it twice = doing it once): LOOKUP, READ, WRITE (with explicit offset).
- Not idempotent: `MKDIR` twice → second fails ("already exists"). Life isn't perfect.

## 6. Improving Performance: Client-Side Caching
- Cache file data and metadata in client memory; also **write buffering** (return before writes hit the server).
- Creates the **cache consistency problem**:
  - **Update visibility**: when do other clients see a client's update?
  - **Stale cache**: client has an old copy after another client updated the server.

## 7. Assessing NFS Cache Consistency
| Problem | NFSv2 fix |
|---|---|
| Update visibility | **Flush-on-close** (close-to-open consistency): on close, write all dirty pages to the server |
| Stale cache | Before using cached content, send **GETATTR**; if modified time newer → invalidate |
- GETATTR flood → **attribute cache** with timeout (e.g., 3 s) → semantics become murky (stale data possible within the window).

## 8. Implications on Server-Side Write Buffering
- Server must **commit writes to stable storage** before replying success. Otherwise: client writes a, b, c; server acks b from memory, crashes before writing it → file has a, old-b, c.
- Write performance bottleneck → use **battery-backed memory** or FSs designed for fast sync writes (e.g., WAFL).

> 💡 Idempotency is a powerful idea: makes retry-based failure handling simple (also useful in other systems).
> ⚠️ NFS also needed security (originally weak: trusted client UIDs) — later Kerberos integration.
