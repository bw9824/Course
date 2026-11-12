# CH50 - The Andrew File System (AFS)

## Big Picture
- CMU (Satyanarayanan et al., 1980s). Main goal: **scale** — how many clients can one server support?
- Unlike NFS: **whole-file caching** on the client's **local disk**, simple **user-visible** consistency semantics.

---

## 1. AFS Version 1
- On `open()`: fetch the **entire file** from the server, store on local disk; reads/writes go to the local copy.
- On `close()`: if modified, **flush** the whole file back.
- Next `open()`: client sends **TestAuth** to check whether the file changed; if not, use the local copy.

| AFSv1 protocol | |
|---|---|
| TestAuth | Test whether file changed (validate cache) |
| GetFileStat | Get stat info |
| Fetch | Fetch file contents |
| Store | Store file on server |
| SetFileStat | Set stat info |
| ListDir | List directory contents |

## 2. Problems with Version 1
- **Path-traversal costs too high**: client sends the full pathname; server walks it → server CPU bottleneck.
- **Too many TestAuth messages** (like NFS GETATTRs), mostly saying "not changed".
- Also: load imbalance across servers; one process per client on the server (context-switch overhead).
- Server could support only ~20 clients.

> 💡 **Measure, then build**: AFS designers measured where the time went before redesigning.

## 3. Improving the Protocol (AFSv2)
- **Callback**: server promises to **inform the client** when its cached file is modified → client no longer polls. State moves to the server.
- **File identifier (FID)** (like an NFS file handle) = volume ID + file ID + "uniquifier" (for reuse).
- Client walks paths itself, caching directory contents and setting up callbacks on them.
```
open("/home/remzi/notes.txt"):
  Fetch(home) → cache, callback   → Fetch(remzi) → cache, callback   → Fetch(notes.txt) → cache, callback
later reads/writes: purely local; close: Store if dirty
next open: if callbacks still valid → no server interaction at all
```

## 4. Cache Consistency
- **Across machines**: updates visible on the server (and to others) **at close**; server **breaks callbacks** for other clients caching the file → they refetch on next open.
- **Same machine**: local processes see writes **immediately** (UNIX semantics).
- **Last writer wins** (last closer): whole file replaced; no mixing of blocks like NFS.

| Timeline | Behavior |
|---|---|
| P1 on A, P2 on B both open, write, close | Whichever closes last overwrites the server copy entirely |

## 5. Crash Recovery
- **Client crash/reboot**: may have missed callback breaks → treat cache as **suspect**, send TestAuth on next access.
- **Server crash**: callbacks are in memory → lost. Clients must realize (heartbeat messages or server says "treat caches as suspect") and revalidate.
- More complex than NFS's stateless approach.

## 6. Scale and Performance of AFSv2
- ~50 clients per server (vs ~20).
- Client performance often close to local (local disk cache).

| Workload | NFS | AFS | AFS/NFS |
|---|---|---|---|
| Small file, sequential read | N_s·L_net | N_s·L_net | 1 |
| Small file, re-read | N_s·L_mem | N_s·L_mem | 1 |
| Large file, re-read | N_L·L_net (memory too small) | N_L·L_disk | L_disk / L_net |
| Large file, sequential overwrite | N_L·L_net | 2·N_L·L_net (fetches first) | 2 |
| Large file, single read in middle | L_net | N_L·L_net (whole file) | N_L |

- AFS wins on large-file re-reads (local disk cache); loses on overwriting files and accessing small parts of large files.

## 7. Other AFS Improvements
- **Global namespace** (same paths on every client).
- **Security**: authentication; **ACLs** (flexible access control).
- **Volume** management: move volumes between servers for load balancing.

> 💡 Lessons: design for scale from the start; simple semantics users understand (whole-file, close-to-open). AFS is less widely used today; NFS (and its evolutions, NFSv4 adopting some AFS ideas) dominates.
