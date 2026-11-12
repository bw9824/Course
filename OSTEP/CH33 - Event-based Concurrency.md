# CH33 - Event-based Concurrency (Advanced)

## Big Picture
- Alternative to threads, used in GUI apps and servers (**node.js**): **event-based concurrency**.
- Problems with threads: hard to get right (locks, deadlocks) and little control over scheduling.
- Idea: wait for an event → figure out its type → do the small amount of work (**event handler**). No threads, no locks on a single CPU.

---

## 1. The Event Loop
```c
while (1) {
    events = getEvents();
    for (e in events)
        processEvent(e);
}
```
- When a handler runs, it's the **only** activity → explicit control over scheduling.

## 2. select() / poll()
```c
int select(int nfds, fd_set *restrict readfds, fd_set *restrict writefds,
           fd_set *restrict errorfds, struct timeval *restrict timeout);
```
- Checks which fds are ready for reading/writing; returns the count. Timeout NULL → block; 0 → return immediately.
```c
while (1) {
    fd_set readFDs; FD_ZERO(&readFDs);
    for (int fd = minFD; fd < maxFD; fd++) FD_SET(fd, &readFDs);
    int rc = select(maxFD + 1, &readFDs, NULL, NULL, NULL);
    for (int fd = minFD; fd < maxFD; fd++)
        if (FD_ISSET(fd, &readFDs)) processFD(fd);
}
```
> 💡 Blocking vs **non-blocking** (synchronous vs asynchronous) interfaces: non-blocking returns immediately and works in the background — essential for event-based design.

## 3. Why Simpler? No Locks Needed
- Single CPU: one event at a time → no interruption by another thread → no concurrency bugs of the thread kind.

## 4. A Problem: Blocking System Calls
- A handler issuing a **blocking** call (e.g., `open()`/`read()` on disk) blocks the entire server.
> ⚠️ Rule: **no blocking calls are allowed** in event-based systems.

## 5. Asynchronous I/O
```c
struct aiocb {
    int aio_fildes; off_t aio_offset; volatile void *aio_buf; size_t aio_nbytes;
};
int aio_read(struct aiocb *aiocbp);     // issue; returns immediately
int aio_error(const struct aiocb *aiocbp); // poll: 0 = done, EINPROGRESS otherwise
```
- Checking many outstanding I/Os by polling is painful → some systems use **interrupts / UNIX signals** to notify completion.
- Without async I/O: hybrid — **thread pools** for disk I/O, events for network (Flash web server).

## 6. Another Problem: State Management
- Thread: state lives on its stack. Event handler: must save state explicitly for when I/O completes → **manual stack management**.
- Solution: **continuations** — record needed info (e.g., the socket fd) in a structure (hash table keyed by the I/O's fd), look it up when the event completes.
```c
int rc = read(fd, buffer, size);   // thread version: then write(sd, buffer, size)
// event version: issue aio_read; store (fd → sd) mapping; on completion, look up sd and write
```

## 7. What Is Still Difficult
- **Multiple CPUs**: run handlers in parallel → back to locks.
- **Paging**: a page fault in a handler blocks the server implicitly.
- **Evolving code**: a non-blocking routine later becoming blocking requires splitting handlers.
- Async disk I/O and network I/O don't integrate cleanly (`select()` for network + AIO calls for disk).

> 💡 Threads and events are both valid; pick based on the problem (Ousterhout: "Why Threads Are A Bad Idea"; von Behren: "Why Events Are A Bad Idea").
