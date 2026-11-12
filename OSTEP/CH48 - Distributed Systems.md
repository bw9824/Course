# CH48 - Distributed Systems

## Big Picture
- Distributed systems (Google, Facebook, client/server) must keep working despite **failures** — machines, disks, networks, software.
- Crux: build systems that work when components fail. Also: **performance** (minimize messages) and **security**.
- Central problem: **communication** — networks are fundamentally **unreliable**.

---

## 1. Communication Basics
- Packets get lost, corrupted (bit flips), or **dropped** because of insufficient **buffering** in routers/switches/endpoints.
- Packet loss is **fundamental**.

## 2. Unreliable Communication Layers
- **UDP/IP**: send/receive datagrams; no delivery guarantees.
```c
int UDP_Open(int port);                                   // create socket, bind
int UDP_Write(int sd, struct sockaddr_in *addr, char *buffer, int n);  // sendto()
int UDP_Read(int sd, struct sockaddr_in *addr, char *buffer, int n);   // recvfrom()
```
- Checksums detect corruption — not loss. Some apps (e.g. streaming) handle loss themselves → **end-to-end argument**: reliability is best provided at the application level.

## 3. Reliable Communication Layers
- **Acknowledgment (ack)**: receiver replies on receipt.
- **Timeout / retry**: sender keeps a copy; resend if no ack within the timeout.
- Lost **ack** → duplicate message → receiver must detect **duplicates** for **exactly-once** semantics:
  - **Sequence counter**: sender and receiver agree on a starting value; message carries counter N; receiver compares to its expected value (seen → ack again, don't pass up).
- Timeout tuning: too small → wasted resends; too big → slow. **Exponential back-off** under congestion (avoid overloading a server).
- **TCP/IP** provides all of this (plus congestion control, multiple outstanding requests).

## 4. Communication Abstractions
### Distributed Shared Memory (DSM)
- Processes on different machines share a virtual address space; page fault → fetch page from a remote machine.
- ❌ Failure handling: a machine fails → parts of address space vanish. ❌ Performance: a single memory access may be a remote page fetch → rarely used.

### Remote Procedure Call (RPC)
- Make remote execution look like a **local function call**.
- **Stub generator** (protocol compiler): from the interface, generates:
  - **Client stub**: create buffer → **marshal** (serialize) args + function ID → send → wait → **unmarshal** reply → return.
  - **Server stub**: unpack → call the real function → package result → reply.
- **Run-time library** issues:
  | Issue | Approach |
  |---|---|
  | Naming | Hostname + port (DNS) |
  | Transport | Often built on **UDP** with its own reliability (avoid TCP's overhead) |
  | Long-running calls | Client periodically asks "still working?"; server sends **explicit acks** |
  | Large arguments | **Fragmentation** + reassembly |
  | Byte ordering | **Endianness** (big vs little); e.g., XDR / Sun RPC defines a format |
  | Sync vs async | **Asynchronous RPC**: return immediately, collect results later |
  | Server concurrency | Thread pool for concurrent requests |

> 💡 Use **checksums** for integrity; and remember **end-to-end**: lower-layer reliability can't guarantee the app's goal (e.g., a file transfer still needs an end-to-end checksum).
