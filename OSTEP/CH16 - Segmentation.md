# CH16 - Segmentation

## Big Picture
- Base/bounds wastes the big free space between heap and stack and can't fit an address space bigger than memory.
- **Segmentation**: a base/bounds pair **per logical segment** (code, heap, stack) → place each segment independently in physical memory.

---

## 1. Generalized Base/Bounds
| Segment | Base | Size |
|---|---|---|
| Code | 32K | 2K |
| Heap | 34K | 3K |
| Stack | 28K | 2K |
- Code at VA 100 → 32K + 100 = **32868**.
- Heap at VA 4200 → offset = 4200 − 4096 = 104 → 34K + 104 = **34920**.
- Address beyond the segment → **segmentation violation / segfault**.

## 2. Which Segment Are We Referring To?
- **Explicit approach**: top bits of the VA select the segment.
```
14-bit VA:  | seg (2) | offset (12) |   00 = code, 01 = heap, 11 = stack
```
```c
Segment = (VirtualAddress & SEG_MASK) >> SEG_SHIFT;   // SEG_MASK 0x3000, SHIFT 12
Offset  =  VirtualAddress & OFFSET_MASK;              // 0xFFF
if (Offset >= Bounds[Segment]) RaiseException(PROTECTION_FAULT);
else PhysAddr = Base[Segment] + Offset;
```
- 2 bits for 3 segments wastes one; some systems put code with heap → 1 bit.
- **Implicit approach**: hardware infers from how the address was formed (PC → code, stack/base pointer → stack, else heap).

## 3. The Stack (grows backward)
- Hardware needs a **direction bit** (1 = grows positive).
- Stack VA 15KB → segment 11, offset 3KB; max size 4KB → negative offset = 3KB − 4KB = −1KB → PA = 28KB − 1KB = **27KB**. Check |−1KB| ≤ size (2KB).

## 4. Support for Sharing
- **Protection bits** per segment (Read / Write / Execute).
- Code segment **Read-Execute** → can be shared across processes safely.

| Segment | Base | Size (max 4K) | Grows positive? | Protection |
|---|---|---|---|---|
| Code₀₀ | 32K | 2K | 1 | Read-Execute |
| Heap₀₁ | 34K | 3K | 1 | Read-Write |
| Stack₁₁ | 28K | 2K | 0 | Read-Write |

## 5. Fine-Grained vs Coarse-Grained
- **Coarse-grained**: few segments (code, stack, heap).
- **Fine-grained**: thousands of segments (Multics, Burroughs B5000) → needs a **segment table** in memory.

## 6. OS Support
- Context switch: save/restore segment registers.
- Segment grow: `malloc` → `sbrk()` → OS grows the heap segment (or rejects).
- Managing free space: segments have variable sizes → physical memory fills with small holes → **external fragmentation**.
  - **Compaction**: stop processes, copy segments to be contiguous, update registers → expensive, makes growth harder.
  - **Free-list algorithms**: best-fit, worst-fit, first-fit, **buddy** (next chapter). Fragmentation always exists; algorithms only minimize it.

## 7. Summary
- ✅ Supports sparse address spaces, fast translation, code sharing.
- ❌ External fragmentation; still not flexible enough for a large sparse heap (whole segment must be resident).
- → Next: **paging**.
