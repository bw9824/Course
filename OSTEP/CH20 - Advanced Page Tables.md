# CH20 - Paging: Smaller Tables

## Big Picture
- Linear page tables are **too big**: 32-bit space, 4KB pages, 4-byte PTE → **4MB per process**; mostly invalid entries.
- Ways to shrink them: **bigger pages**, **paging + segmentation hybrid**, **multi-level page tables**, **inverted page tables**, swapping tables to disk.

---

## 1. Bigger Pages
- 16KB pages → 18-bit VPN → 2^18 × 4B = **1MB** (4× smaller).
- ❌ **Internal fragmentation** inside pages. Most systems use 4KB (or 8KB) normally, with optional **large pages** (e.g., for DBMSs, to reduce TLB pressure).

## 2. Hybrid: Paging and Segments
- One page table **per segment** (code, heap, stack). Base register points to the segment's page table; bounds = number of valid pages.
```
32-bit VA: | Seg (2) | VPN (18) | Offset (12) |
SN      = (VirtualAddress & SEG_MASK) >> SN_SHIFT
VPN     = (VirtualAddress & VPN_MASK) >> VPN_SHIFT
AddressOfPTE = Base[SN] + (VPN * sizeof(PTE))
```
- Unallocated pages between stack and heap no longer occupy the table.
- ❌ Still segmentation: assumes a usage pattern; big sparse heap still wastes; page tables of arbitrary size → **external fragmentation** again.

## 3. Multi-Level Page Tables
- Chop the linear page table into **page-sized units**; if an entire page of PTEs is invalid, **don't allocate it**.
- **Page directory**: one **PDE** per page of the page table (valid bit + PFN).
```
Linear:     PTBR → [PT page 0][PT page 1 (all invalid)][PT page 2 (all invalid)][PT page 3]
Multi-level: PDBR → [Page Directory] → PT page 0, PT page 3  (pages 1, 2 not allocated)
```
- ✅ Space proportional to the address space **actually used**; ✅ each piece fits in a page → easy to manage (no contiguous allocation needed).
- ❌ **Time–space trade-off**: TLB miss → **two** memory loads (PDE + PTE); ❌ more complex.

### Detailed Example
- 16KB address space, 64-byte pages → 14-bit VA, 8-bit VPN, 6-bit offset; 256 PTEs × 4B = 1KB = 16 pages of PTEs (16 PTEs/page).
```
| PD index (4) | PT index (4) | offset (6) |
PDEAddr = PageDirBase + (PDIndex * sizeof(PDE))
PTEAddr = (PDE.PFN << SHIFT) + (PTIndex * sizeof(PTE))
```
- Program uses VPN 0, 1 (code), 4, 5 (heap), 254, 255 (stack) → only **2** page-table pages + 1 directory = **3 pages** instead of 16.
- VA 0x3F80 (11 1111 1000 0000) → PD index 15, PT index 14 → PFN 55 → PA = 55×64 + 0 = 0x0DC0.

### More Than Two Levels
- Goal: **every piece fits within one page** (including the directory).
- 30-bit VA, 512-byte pages, 4B PTE → 128 PTEs/page → 21-bit VPN = 7-bit PT index + 14-bit PD index → directory spans 128 pages → add another level:
```
| PD index 0 (7) | PD index 1 (7) | PT index (7) | offset (9) |
```

### Translation with TLB
```c
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success) { ... normal hit ... }
else {
    PDE = AccessMemory(PDBR + PDIndex * sizeof(PDE))
    if (PDE.Valid == False) RaiseException(SEGMENTATION_FAULT)
    PTE = AccessMemory((PDE.PFN << SHIFT) + PTIndex * sizeof(PTE))
    if (PTE.Valid == False) RaiseException(SEGMENTATION_FAULT)
    ... check protection, TLB_Insert, RetryInstruction()
}
```

## 4. Inverted Page Tables
- **One table for the whole system**: one entry per **physical page** → which process + VPN uses it.
- Lookup needs a search → use a **hash table** (PowerPC).

## 5. Swapping Page Tables to Disk
- Some systems (VAX/VMS) put page tables in **kernel virtual memory** → can be swapped out under memory pressure.

> 💡 Page tables are just **data structures** — trade time for space. Memory-constrained → small structures; plenty of memory and many TLB misses → bigger, faster tables.
