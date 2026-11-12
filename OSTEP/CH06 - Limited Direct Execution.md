# CH06 - Mechanism: Limited Direct Execution

## Big Picture
- Goal: virtualize the CPU with **performance** and **control**.
- **Direct execution**: just run the program on the CPU → fast.
- But two problems: (1) how to let it do **restricted operations** safely? (2) how to **stop it** and switch processes?
- Answer: **Limited** Direct Execution (LDE) — hardware support: **user/kernel mode**, **traps**, **timer interrupt**.

---

## 1. Plain Direct Execution
| OS | Program |
|---|---|
| Create entry in process list, allocate memory, load program, set up stack (argc/argv), clear registers, call `main()` | |
| | Run `main()`, execute `return` |
| Free memory, remove from list | |
- No limits → program could do anything; OS is just a library.

## 2. Problem #1: Restricted Operations
- **User mode**: restricted (e.g., can't issue I/O). Violation → exception → OS likely kills the process.
- **Kernel mode**: OS runs with full privileges.
- **System call**: program executes a **trap** instruction → jumps into kernel, raises privilege → OS does work → **return-from-trap** lowers privilege.
- Hardware saves registers (PC, flags) on a per-process **kernel stack**; return-from-trap pops them.
- **Trap table** set up at **boot** (in kernel mode): tells hardware where the **trap handlers** are. Setting it is privileged.
- **System-call number**: user code puts it in a register; the OS validates it → user can't jump to arbitrary kernel addresses.

### LDE protocol (two phases)
| OS @ boot (kernel) | Hardware |
|---|---|
| Initialize trap table | Remember address of syscall handler |

| OS @ run (kernel) | Hardware | Program (user) |
|---|---|---|
| Create process entry, alloc memory, load, set up stack with argv, fill kernel stack with reg/PC, **return-from-trap** | Restore regs from kernel stack, move to user mode, jump to `main` | |
| | | Run `main()` ... call syscall, **trap** into OS |
| | Save regs to kernel stack, move to kernel mode, jump to trap handler | |
| Handle trap, do work, **return-from-trap** | Restore regs, move to user mode, jump to PC after trap | |
| | | ... return from `main`, trap (`exit()`) |
| Free memory, remove from list | | |

> ⚠️ The OS must check syscall arguments carefully — e.g., a `write()` buffer address must not point into kernel memory.

## 3. Problem #2: Switching Between Processes
- If a process is running, the OS **isn't** — how does it regain control?
- **Cooperative approach**: wait for a syscall / illegal operation; processes call `yield()`. Infinite loop → only fix is **reboot**.
- **Non-cooperative approach**: **timer interrupt** every few ms → hardware halts the process, runs the OS's **interrupt handler**. Starting the timer is privileged (done at boot).

## 4. Context Switch
- Scheduler decides to switch → OS runs low-level `switch()`:
  - Save current process's registers (into its PCB / `proc` struct), restore the next one's, switch kernel stacks.
- Two kinds of register save:
  1. **Hardware** saves **user** regs on the kernel stack on interrupt.
  2. **OS** saves **kernel** regs into the process structure on switch.

### Timer interrupt + switch (A → B)
| OS | Hardware | Program |
|---|---|---|
| | | Process A running |
| | Timer interrupt: save regs(A) → k-stack(A), kernel mode, jump to handler | |
| Handle trap, call `switch()`: save regs(A) → proc_t(A), restore regs(B) ← proc_t(B), switch to k-stack(B), return-from-trap (into B) | | |
| | Restore regs(B) ← k-stack(B), user mode, jump to B's PC | |
| | | Process B running |

```asm
# xv6 swtch(struct context **old, struct context *new)
movl 4(%esp), %eax     # old
movl 8(%esp), %edx     # new
pushl %ebp; pushl %ebx; pushl %esi; pushl %edi   # save callee-saved regs
movl %esp, (%eax)      # save old stack pointer
movl %edx, %esp        # switch stacks
popl %edi; popl %esi; popl %ebx; popl %ebp
ret
```

## 5. Concurrency Concerns
- Interrupt during a syscall/interrupt handler? → **disable interrupts** during handling (not too long — lost interrupts), plus **locks** in the kernel.

## 6. How Long Things Take
- lmbench: syscall ~ **microseconds** in 1996 → **sub-microsecond** now; context switch similar.
- OS performance doesn't track CPU speed (memory bandwidth bound).

> 💡 **Reboot is useful**: returns software to a known, tested state; reclaims leaked resources.
