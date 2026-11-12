# CH19 - Paging: Faster Translations (TLBs)

## Big Picture
- Paging needs an extra memory lookup per access → too slow.
- **TLB (Translation-Lookaside Buffer)**: hardware **cache of VPN → PFN translations** inside the MMU.
- Most accesses hit the TLB → paging becomes practical.

---

## 1. Basic TLB Algorithm
```c
VPN = (VirtualAddress & VPN_MASK) >> SHIFT
(Success, TlbEntry) = TLB_Lookup(VPN)
if (Success == True) {                         // TLB hit
    if (CanAccess(TlbEntry.ProtectBits)) {
        Offset = VirtualAddress & OFFSET_MASK
        PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
        Register = AccessMemory(PhysAddr)
    } else RaiseException(PROTECTION_FAULT)
} else {                                       // TLB miss
    PTEAddr = PTBR + (VPN * sizeof(PTE))
    PTE = AccessMemory(PTEAddr)
    if (PTE.Valid == False) RaiseException(SEGMENTATION_FAULT)
    else if (!CanAccess(PTE.ProtectBits)) RaiseException(PROTECTION_FAULT)
    else { TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits); RetryInstruction() }
}
```

## 2. Example: Accessing an Array
- 10 ints (40 bytes) starting at VA 100, 16-byte pages, 8-bit VA.
- Access pattern: miss, hit, hit | miss, hit, hit, hit | miss, hit, hit → **70% hit rate**, thanks to **spatial locality**.
- Bigger pages → fewer misses. Re-running the loop soon → all hits (**temporal locality**).

> 💡 Caching is one of the most fundamental performance techniques. Caches must be small to be fast — making them big defeats the purpose.

## 3. Who Handles the TLB Miss?
| | Hardware-managed (CISC, x86) | Software-managed (RISC, MIPS/SPARC) |
|---|---|---|
| Who walks the table | Hardware, via **page-table base register** (CR3) | OS **trap handler** |
| Page table format | Fixed by hardware | Any structure the OS wants |
| Notes | | Return-from-trap must **retry** the faulting instruction (not continue after it); avoid infinite miss loops (keep handler in unmapped memory or **wired** TLB entries) |

## 4. TLB Contents
```
| VPN | PFN | other bits |
```
- **Fully associative**: any translation can be anywhere; hardware searches all entries in parallel.
- Other bits: **valid** (≠ PTE valid), protection, address-space ID, dirty...

## 5. Context Switches
- TLB entries belong to one process; after a switch they're meaningless.
- Option 1: **flush** the TLB on every switch (set all valid bits to 0) → every switch causes misses.
- Option 2: **ASID (Address Space Identifier)** field (like a PID, ~8 bits) → entries from multiple processes coexist.
- Two processes can **share** a page (two entries, different VPN/ASID, same PFN) — e.g., code pages.

## 6. Replacement Policy
- **LRU**: exploits locality.
- **Random**: simple, avoids corner cases (e.g., loop over n+1 pages with an n-entry TLB under LRU → every access misses).

## 7. Real TLB: MIPS R4000
```
| VPN (19) | G | ASID (8) | PFN (24) | C (3) | D | V |
```
- 32-bit addresses, 4KB pages → 20-bit VPN, but half the space reserved for kernel → 19 bits.
- **G** global bit (shared across processes, ignore ASID), C coherence, D dirty, V valid.
- 32 or 64 entries; some **wired** for the OS.
- Privileged instructions: `TLBP` (probe), `TLBR` (read), `TLBWI` (write indexed), `TLBWR` (write random).

## 8. Culler's Law
- **RAM isn't always RAM**: accessing many pages randomly can exceed **TLB coverage** → many misses.
- Fix: **larger pages** (used by DBMSs).
- Physically-indexed caches make the TLB a bottleneck on the critical path → **virtually-indexed caches** (with their own issues).
