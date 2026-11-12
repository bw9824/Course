# CH21 - Beyond Physical Memory: Mechanisms

## Big Picture
- Relax the assumption that all address spaces fit in physical memory.
- Add a level to the memory hierarchy: **swap space** on disk (slower, bigger).
- Illusion of a **large** virtual memory — programmers don't manage overlays by hand.

---

## 1. Swap Space
- Disk space reserved for moving pages in and out; OS remembers each page's **disk address**.
- Example: 4 physical frames, 8 swap blocks; Proc 0/1/2 partly in memory, Proc 3 fully swapped.
- Code pages may be paged back from the **binary on disk** instead of swap.

## 2. The Present Bit
- On TLB miss, hardware walks the page table; PTE **present bit**:
  - 1 → page in memory, proceed.
  - 0 → page on disk → **page fault**.

> 💡 "Page fault" is a slight misnomer — accessing a non-present page is legal; the OS handles it.

## 3. The Page Fault
- Handled by the **OS** (even with hardware-managed TLBs): disk is slow anyway, and hardware doesn't understand swap space.
- PTE bits normally holding the PFN can store the **disk address** of the page.
- OS issues the disk I/O; process is **blocked** meanwhile → another process runs (overlap).
- I/O done → update PTE (present = 1, PFN), **retry** instruction → TLB miss → then TLB hit.

## 4. What If Memory Is Full?
- Must **page out** one or more pages first → **page-replacement policy** (next chapter).

## 5. Page Fault Control Flow
### Hardware
```c
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success) { ... normal ... }
else {
    PTE = AccessMemory(PTBR + VPN * sizeof(PTE))
    if (PTE.Valid == False)            RaiseException(SEGMENTATION_FAULT)
    else if (!CanAccess(PTE.ProtectBits)) RaiseException(PROTECTION_FAULT)
    else if (PTE.Present == True)      { TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits); RetryInstruction() }
    else if (PTE.Present == False)     RaiseException(PAGE_FAULT)
}
```
### OS (page-fault handler)
```c
PFN = FindFreePhysicalPage()
if (PFN == -1)              // no free page found
    PFN = EvictPage()       // replacement algorithm
DiskRead(PTE.DiskAddr, PFN) // sleep (waiting for I/O)
PTE.present = True
PTE.PFN     = PFN
RetryInstruction()
```

| Case | Present | Valid | Result |
|---|---|---|---|
| In memory, TLB miss | 1 | 1 | Hardware fills TLB |
| Swapped out | 0 | 1 | Page fault → OS loads page |
| Invalid address | – | 0 | Segfault → kill process |

## 6. When Replacements Really Occur
- Not lazily at the last moment: OS keeps a small amount of memory free.
- **High watermark (HW)** and **low watermark (LW)**: when free pages < LW, a background **swap daemon / page daemon** evicts until HW pages are free, then sleeps.
- Lets the OS **cluster/group** writes to swap → fewer seeks, better disk efficiency.
- Page-fault handler: if no free pages, wake the daemon and wait.

> 💡 Do work in the **background** when possible: batching, idle-time use (e.g., buffered file writes).
