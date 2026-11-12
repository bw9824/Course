# CH36 - I/O Devices

## Big Picture
- Programs without input/output are pointless; how should I/O be integrated into systems?
- Topics: system architecture (buses), a canonical device + protocol, **interrupts**, **DMA**, device interaction methods, **device drivers**.

---

## 1. System Architecture
```
CPU ── Memory          (memory bus — fastest, shortest)
  │
General I/O bus (PCI) ── graphics, high-performance devices
  │
Peripheral bus (SCSI, SATA, USB) ── disks, mice, keyboards (slowest)
```
- Hierarchy because of **physics and cost**: faster buses must be shorter; high-performance buses are expensive.
- Modern (Intel Z270): CPU ↔ memory, CPU ↔ graphics via **PCIe**; **I/O chip (DMI)** connects disks (eSATA), USB, and **PCIe NVMe** SSDs.

## 2. A Canonical Device
- **Interface**: registers presented to the system — **status**, **command**, **data**.
- **Internals**: micro-controller, memory (DRAM/SRAM), firmware, other chips (implementation-specific).

## 3. Canonical Protocol
```c
While (STATUS == BUSY) ;           // wait until device is not busy (polling)
Write data to DATA register
Write command to COMMAND register  // starts the device
While (STATUS == BUSY) ;           // wait until device is done with your request
```
- CPU moves data itself → **programmed I/O (PIO)**.
- ❌ **Polling** wastes CPU time.

## 4. Lowering CPU Overhead with Interrupts
- Issue the request, put the process to **sleep**, switch to another; device raises a **hardware interrupt** → **ISR (interrupt handler)** wakes the process.
- Allows **overlap** of computation and I/O.
- ⚠️ Not always better:
  - Fast device → interrupt overhead/context switch can exceed polling. **Hybrid**: poll a little, then interrupt (**two-phased**).
  - Networks: flood of interrupts → **livelock** (only processing interrupts); polling gives control.
  - **Interrupt coalescing**: device waits a bit to batch several completions into one interrupt (latency trade-off).

## 5. More Efficient Data Movement with DMA
- PIO copying a big block occupies the CPU.
- **DMA (Direct Memory Access) engine**: OS programs it with where data lives, how much, which device → DMA copies; raises an interrupt when done. CPU free meanwhile.

## 6. Methods of Device Interaction
| Method | How |
|---|---|
| **Explicit I/O instructions** | x86 `in` / `out` — **privileged** |
| **Memory-mapped I/O** | Device registers appear as memory addresses; ordinary load/store |
- No strong advantage either way; both still used.

## 7. Fitting Into the OS: Device Drivers
- Keep the OS **device-neutral** via **abstraction**: the **device driver** encapsulates device specifics.
```
Application
  ↓  POSIX API (open, read, write, close)
File System             | raw interface
  ↓  generic block interface (block read/write)
Generic Block Layer
  ↓  specific block interface
Device Driver [SCSI, ATA, ...]
```
- **Raw interface**: direct block access (used by fsck, defragmenters).
- Downside: rich device features (e.g., SCSI error reporting) hidden behind a lowest-common-denominator interface.
- Drivers are **~70% of Linux kernel code**; and the main source of kernel **bugs**.

## 8. Case Study: xv6 IDE Disk Driver
- Registers: control (0x3F6), command block (0x1F0–0x1F7: data, error, sector count, LBA, drive, status/command).
- Protocol: wait for ready → write parameters (sector count, LBA, drive) → start I/O (READ/WRITE command) → data transfer (for writes) → handle interrupt → error handling.
```c
static int ide_wait_ready() {
    int r;
    while (((r = inb(0x1f7)) & IDE_BSY) || !(r & IDE_DRDY)) ; // loop until drive isn't busy
}
void ide_rw(struct buf *b) {
    struct buf **pp;
    acquire(&ide_lock);
    for (pp = &ide_queue; *pp; pp = &(*pp)->qnext) ;   // walk queue
    *pp = b;                               // add request to end
    if (ide_queue == b) ide_start_request(b); // send req to disk
    while ((b->flags & (B_VALID|B_DIRTY)) != B_VALID)
        sleep(b, &ide_lock);               // wait for completion
    release(&ide_lock);
}
void ide_intr() {
    // if READ: insl(0x1f0, b->data, 512/4)  — get data
    // set B_VALID, clear B_DIRTY, wakeup(b); start next request if queue non-empty
}
```

## 9. Historical Notes
- Interrupts and DMA date back to the 1950s (UNIVAC, DYSEAC, IBM SAGE).
