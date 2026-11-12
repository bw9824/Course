# CH04 - The Abstraction: The Process

## Big Picture
- **Process** = a running program. The program is lifeless bytes on disk; the OS brings it to life.
- Crux: how to provide the illusion of **many CPUs** with only a few?
- Answer: **time sharing** the CPU — run one process, stop it, run another.
- Needs **mechanisms** (low-level: context switch) + **policies** (high-level: scheduling decisions).

---

## 1. Machine State of a Process
| Part | Contents |
|---|---|
| Memory (address space) | Instructions, data the program reads/writes |
| Registers | General registers, **PC/IP**, **stack pointer**, frame pointer |
| I/O info | List of open files |

## 2. Process API (what any OS offers)
- **Create**, **Destroy** (kill), **Wait**, **Miscellaneous control** (suspend/resume), **Status** (how long it ran, state).

## 3. Process Creation
1. **Load** code + static data from disk (executable format) into the address space.
   - Early OSes: **eagerly** all at once. Modern: **lazily** (load pieces when needed, via paging/swapping).
2. Allocate the **run-time stack** (local vars, params, return addresses); fill in `argc`/`argv`.
3. Allocate some **heap** (`malloc`/`free`, grows on demand).
4. I/O setup: on UNIX, open fds **0, 1, 2** (stdin, stdout, stderr).
5. Jump to the entry point `main()` → OS hands the CPU to the process.

## 4. Process States
```
          scheduled
  Ready ------------> Running
   ^    <------------   |
   |     descheduled    | I/O: initiate
   |                    v
   +----- I/O: done --- Blocked
```
- **Running**: executing instructions.
- **Ready**: could run, but the OS chose not to right now.
- **Blocked**: waiting for some event (e.g., disk I/O) → another process can use the CPU.
- Other states: **initial** (being created), **final / zombie** (exited but not yet cleaned up; lets the parent read the return code via `wait()`).

### Example: two processes, one does I/O
| Time | P0 | P1 | Notes |
|---|---|---|---|
| 1–3 | Running | Ready | |
| 4 | Blocked | Running | P0 starts I/O → switch to P1 |
| 5–6 | Blocked | Running | |
| 7 | Ready | Running | I/O done |
| 8 | Ready | Running | P1 done |
| 9–10 | Running | – | |

> 💡 Scheduler decisions (switch on I/O? resume P0 right away when I/O completes?) are **policy** — later chapters.

## 5. Data Structures
- **Process list** (task list): one entry per process.
- **PCB (Process Control Block)** — e.g. xv6 `struct proc`:
```c
struct context { int eip, esp, ebx, ecx, edx, esi, edi, ebp; };   // saved registers
enum proc_state { UNUSED, EMBRYO, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
struct proc {
    char *mem; uint sz; char *kstack;      // memory, size, kernel stack
    enum proc_state state; int pid;
    struct proc *parent; void *chan;       // chan: what it's sleeping on
    int killed; struct file *ofile[NOFILE]; struct inode *cwd;
    struct context context;                // swapped on context switch
    struct trapframe *tf;
};
```
- **Register context**: when stopped, registers are saved here; restoring them resumes the process (**context switch**).
