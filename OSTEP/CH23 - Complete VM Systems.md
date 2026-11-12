# CH23 - Complete Virtual Memory Systems

## Big Picture
- Put the pieces together through two real systems: **VAX/VMS** (1970s, clean design) and **Linux** (modern).
- Many "modern" ideas (COW, demand zeroing, lazy allocation) date back to VMS.

---

## 1. VAX/VMS
### Hardware
- 32-bit VA, **512-byte** pages (small!) → 23-bit VPN + 9-bit offset.
- Upper 2 bits of the VPN select the segment → **hybrid of paging + segmentation**.
```
0       | P0: user code, heap (grows down) |
2^30    | P1: user stack (grows up)        |
2^31    | S:  system (kernel) — shared     |
2^32
```
- Page tables for P0/P1 placed in **kernel virtual memory** (S) → can be swapped out.
- Only one page table for S, shared by all processes.

### Address Space Details
- **Page 0 is invalid** → null-pointer dereference traps.
- Kernel mapped into every address space (protection bits prevent user access) → easy kernel access to user memory, no TLB flush of kernel on switch.

### Replacement: Segmented FIFO
- No reference bit in VAX hardware!
- Each process has a **resident set size (RSS)** limit; pages kept in a FIFO list.
- Evicted pages go to two global **second-chance lists**: **clean-page free list** and **dirty-page list**.
- If a process faults on a page that's still in these lists → reclaim it (no disk I/O) → approximates LRU.
- **Clustering**: group many dirty pages → one big write to disk.

### Other Neat Tricks
| Trick | Meaning |
|---|---|
| **Demand zeroing** | On heap growth, mark page inaccessible; zero it only on first access |
| **Copy-on-write (COW)** | Share a page read-only between address spaces; copy only on a write. Makes UNIX `fork()` + `exec()` cheap |

## 2. The Linux Virtual Memory System
### Linux Address Space (x86 32-bit)
```
0x00000000 | user code, heap, ..., stack   (user portion)
0xC0000000 | kernel logical memory          (kernel portion, shared)
           | kernel virtual memory
```
- **Kernel logical addresses**: `kmalloc`; directly mapped to physical memory (VA − 0xC0000000 = PA) → contiguous physically → good for **DMA**; can't be swapped.
- **Kernel virtual addresses**: `vmalloc`; not necessarily contiguous physically; used for large buffers.

### Page Table Structure (x86-64)
- Hardware-managed, **multi-level** (4 levels; 5 coming).
```
| unused (16) | P1 (9) | P2 (9) | P3 (9) | P4 (9) | offset (12) |
```
- 48 bits used today; top level indexed by P1.

### Large Page Support
- **2MB / 1GB huge pages** → fewer TLB misses, faster TLB miss path, shorter allocation.
- Initially explicit (`mmap`/`shmget`); now **transparent huge pages**.
- ❌ Internal fragmentation, swap cost.

### The Page Cache
- Unified cache for: **memory-mapped files**, file data/metadata, **anonymous** memory (heap/stack, backed by swap).
- Dirty data written back by background threads (`pdflush`).
- Replacement: **2Q** — **inactive list** and **active list**; first access → inactive; re-access → active. Evict from inactive. Defeats cyclic large-file access that kills LRU. Approximated with clock-like scanning.

### Security and Buffer Overflows
| Attack / Defense | Idea |
|---|---|
| Buffer overflow | Overwrite return address with injected code |
| **NX bit** (no-execute) | Mark stack/heap non-executable |
| **ROP** (return-oriented programming) | Chain **gadgets** of existing code |
| **ASLR** | Randomize code/stack/heap placement (also **KASLR** for kernel) |
| **Meltdown / Spectre** | Speculative execution leaks kernel memory |
| **KPTI** (kernel page-table isolation) | Remove most kernel mappings from user page tables → switch tables on entry; costs performance |

> 💡 Laziness is a virtue: demand zeroing, COW — postpone work until needed (maybe never).
