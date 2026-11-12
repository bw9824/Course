# CH18 - Paging: Introduction

## Big Picture
- Segmentation (variable chunks) → fragmentation. Alternative: chop space into **fixed-sized pieces**.
- **Page**: fixed-size unit of the virtual address space. **Page frame**: fixed-size slot in physical memory.
- **Page table**: per-process structure mapping **virtual page → physical frame**.
- ✅ Flexibility (no assumption on how heap/stack grow), ✅ simple free-space management (free list of pages).
- ❌ Can be **slow** (extra memory access) and **big** (page table size).

---

## 1. Simple Example
- 64-byte address space, 16-byte pages → 4 virtual pages; 128-byte physical memory → 8 frames.
- VP0 → PF3, VP1 → PF7, VP2 → PF5, VP3 → PF2.

## 2. Address Translation
```
VA (6 bits):  | VPN (2) | offset (4) |
VA 21 = 01 0101 → VPN 1, offset 5
VPN 1 → PFN 7 (111) → PA = 111 0101 = 117
```
- Only the **VPN** is translated; the **offset** stays the same.

## 3. Where Are Page Tables Stored?
- Can be **huge**: 32-bit address space, 4KB pages → 20-bit VPN → 2^20 entries × 4 bytes = **4MB per process**. 100 processes → 400MB.
- So stored **in memory** (kernel physical memory, or even OS virtual memory/swapped).

## 4. What's in the Page Table?
- Simplest form: **linear page table** — an array indexed by VPN, each entry a **PTE**.

| PTE bit | Meaning |
|---|---|
| **Valid** | Is this translation valid? Unused space marked invalid → no frames needed (sparse address spaces) |
| **Protection** | Read / write / execute allowed? |
| **Present** | In physical memory or swapped to disk? |
| **Dirty** | Modified since brought into memory? |
| **Reference / Accessed** | Accessed recently? (for replacement) |

### x86 PTE (32-bit)
```
| PFN (20) | G | PAT | D | A | PCD | PWT | U/S | R/W | P |
```
- P = present, R/W, U/S = user/supervisor, PWT/PCD/PAT = caching, A = accessed, D = dirty, G = global.

## 5. Paging Is Also Too Slow
```
VPN     = (VirtualAddress & VPN_MASK) >> SHIFT
PTEAddr = PageTableBaseRegister + (VPN * sizeof(PTE))
PTE     = AccessMemory(PTEAddr)
if (PTE.Valid == False)                RaiseException(SEGMENTATION_FAULT)
else if (CanAccess(PTE.ProtectBits) == False) RaiseException(PROTECTION_FAULT)
else {
    offset   = VirtualAddress & OFFSET_MASK
    PhysAddr = (PTE.PFN << PFN_SHIFT) | offset
    Register = AccessMemory(PhysAddr)
}
```
- **Every** memory reference needs **one extra memory access** to fetch the PTE → ~2× slower.

## 6. A Memory Trace
```c
int array[1000];
for (i = 0; i < 1000; i++) array[i] = 0;
```
```asm
1024: movl $0x0,(%edi,%eax,4)   ; array[i] = 0
1028: incl %eax
1032: cmpl $0x03e8,%eax          ; i < 1000?
1036: jne  0x1024
```
- Each iteration: 4 instruction fetches + 1 store = 5 accesses, each needing a page-table lookup → **10 memory accesses** per loop iteration.

> 💡 Two problems to solve next: speed → **TLB** (CH19); size → **smaller page tables** (CH20).
