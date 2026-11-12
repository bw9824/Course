# CH15 - Mechanism: Address Translation

## Big Picture
- Like LDE for the CPU: run directly on hardware, OS steps in at key points.
- **Hardware-based address translation**: every memory access (fetch, load, store) is translated **virtual → physical** by hardware.
- OS sets up the hardware and **manages memory** (which parts are free/used).
- First mechanism: **base and bounds** (dynamic relocation).

---

## 1. Assumptions (for now)
- Address space placed **contiguously** in physical memory.
- Address space smaller than physical memory; all address spaces the same size.

## 2. Example
```c
void func() { int x = 3000; x = x + 3; }
```
```asm
128: movl 0x0(%ebx), %eax   ; load from address 15KB
132: addl $0x03, %eax
135: movl %eax, 0x0(%ebx)   ; store back
```
- Program thinks it's at 0–16KB; OS wants to put it at 32KB–48KB **transparently**.

## 3. Dynamic (Hardware-Based) Relocation
- Two registers per CPU in the **MMU**: **base** and **bounds** (limit).
```
physical address = virtual address + base
if (virtual address >= bounds) → exception (out of bounds)
```
- Example: base = 32KB, VA 128 → PA 32896; VA 15KB → PA 47KB.
- **Static relocation** (loader rewrites addresses) has no protection and is hard to move later.
- Bounds can hold the **size** (check VA) or the **end PA** (check after adding).

| VA | PA (base 16KB, bounds 4KB) |
|---|---|
| 0 | 16KB |
| 1KB | 17KB |
| 3000 | 19384 |
| 4400 | **Fault** (out of bounds) |

## 4. Hardware Requirements
| Requirement | Why |
|---|---|
| Privileged (kernel) mode | Prevent user processes from privileged ops |
| Base/bounds registers | Per-CPU, for translation + bounds checks |
| Translate VA and check bounds | Circuitry in the MMU |
| Privileged instructions to update base/bounds | OS sets them before running a process |
| Privileged instructions to register exception handlers | OS tells hardware what to run on exceptions |
| Raise exceptions | On illegal access or privileged instruction in user mode |

## 5. OS Responsibilities
| When | OS action |
|---|---|
| Process creation | Find space in physical memory (**free list**) |
| Process termination | Reclaim memory to the free list |
| Context switch | Save/restore base & bounds (in PCB) |
| Exception | Handler: usually kill the offending process |

- To **move** an address space: deschedule, copy memory, update saved base.

## 6. LDE + Dynamic Relocation (timeline)
| OS @ run | Hardware | Program |
|---|---|---|
| Allocate entry in proc table, memory, set base/bounds, return-from-trap into A | Restore regs, user mode, jump to A's PC | |
| | | Process A runs, fetches instruction |
| | Translate VA, perform fetch; load/store → check bounds, translate, access | |
| | Timer interrupt → kernel mode, handler | |
| Save regs(A) incl. base/bounds, restore B's, return-from-trap | Restore regs(B), user mode | |
| | | Process B runs, executes a bad load |
| | Load out of bounds → trap | |
| Kill B, free its memory | | |

## 7. Problem: Internal Fragmentation
- Space between heap and stack inside the fixed-size slot is **wasted** (allocated but unused).
- Leads to: **segmentation** (next chapter).
