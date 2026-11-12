# APX - Virtual Machine Monitors

## Big Picture
- **VMM / hypervisor**: a thin layer that virtualizes the machine itself so several **OSes** run on one computer, each thinking it owns the hardware.
- IBM mainframes (1960s) → revived by Stanford's **Disco** and **VMware** (late 1990s).
- Same principle as the OS: **limited direct execution** — run the guest OS directly on the CPU, but the VMM keeps control of the machine.

---

## 1. Motivation
- **Server consolidation**: many lightly-used servers → VMs on fewer machines (cost, ease of administration).
- Desktop: run Windows + Linux apps on one machine.
- Testing / debugging new OSes.
- **Transparency**: the guest OS must run unmodified (as far as possible).

## 2. Virtualizing the CPU
- To "boot" a guest: jump to the OS's first instruction.
- Switching between VMs = **machine switch**: save all machine state of one OS (registers, PC, privileged hardware state), restore another's.
- Problem: the guest OS tries **privileged** operations (update TLB, install trap table) — it can't be allowed to really do them.

### System Calls (trap-and-emulate)
| Process | Guest OS | VMM |
|---|---|---|
| System call: trap into OS | | |
| | | Hardware traps into the **VMM**; VMM calls the **OS trap handler** (it knows the address because the OS tried to install its trap table, a privileged op the VMM recorded) |
| | OS trap handler: decode trap, execute syscall; return-from-trap (privileged → traps) | |
| | | VMM does the real return-from-trap |
| Resume execution | | |
- Without a VM, a syscall needs one trap; with a VM, more mode switches → **slower**.
- Guest OS can't run in kernel mode → on MIPS, a **supervisor mode** (between user and kernel) lets the OS access some extra memory; on others, run the OS in user mode with protection tricks (x86 had extra rings).

> 💡 Use **implementing indirection**: the VMM is another layer of indirection on top of the hardware.

## 3. Virtualizing Memory
- Three levels: **virtual** (process) → **"physical"** (what the guest OS thinks) → **machine** (real memory, VMM-managed).
- Guest OS page table: VA → physical. VMM's per-VM table: physical → machine.

### TLB Miss on a Software-Managed TLB (MIPS-style)
| Process | Guest OS | VMM |
|---|---|---|
| Load from memory: TLB miss → trap | | |
| | | VMM TLB miss handler: call OS TLB handler (reducing privilege) |
| | OS TLB miss handler: VPN → PFN from its page table; try to install VPN→PFN in TLB (privileged → trap) | |
| | | VMM traps; installs **VPN → MFN** (machine frame) instead; jump back to OS |
| | Return from trap (traps again) | |
| | | Return from trap |
| Resume: retry instruction, TLB hit | | |
- Many more traps → TLB misses are much more expensive. Mitigation: VMM keeps a **software TLB** of recent VPN→MFN mappings.
- Hardware-managed TLBs: VMM keeps a **shadow page table** (VA → MA) that the hardware walks; must watch guest page-table updates.

## 4. The Information Gap
- VMM doesn't know what the guest OS is doing:
  - Guest's **idle loop** spins → VMM wastes CPU (it can't tell idle from work).
  - Guest **zeroes pages** for security → VMM already zeroed them → double work.
- Solutions: **implicit information** (infer guest behavior), or **para-virtualization** — slightly modify the guest OS to cooperate (e.g., Xen); **balloon driver** to reclaim memory from a guest.

> 💡 VMMs are widely used: cloud computing (each tenant gets VMs), and hardware support for virtualization (Intel VT, AMD-V; nested page tables) reduces the overheads above.
