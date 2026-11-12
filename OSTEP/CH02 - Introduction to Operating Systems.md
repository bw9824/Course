# CH02 - Introduction to Operating Systems

## Big Picture
- A running program does one simple thing: **fetch → decode → execute** instructions (von Neumann model).
- The **OS** is the body of software that makes it easy to run programs, share memory, talk to devices — correctly and efficiently.
- Main technique: **virtualization** — take a physical resource (CPU, memory, disk) and turn it into a more general, easy-to-use **virtual** form.
- The book's three easy pieces: **Virtualization**, **Concurrency**, **Persistence**.

---

## 1. OS Roles
| Role | Meaning |
|---|---|
| Virtual machine | Turns physical CPU/memory/disk into easier virtual versions |
| Standard library | Exposes **system calls** (a few hundred) apps can call |
| Resource manager | Decides who gets CPU, memory, disk and when |

## 2. Virtualizing the CPU
```c
// cpu.c — spin ~1s then print the string, forever
while (1) { Spin(1); printf("%s\n", str); }
```
```
prompt> ./cpu A & ./cpu B & ./cpu C & ./cpu D &
```
- One CPU, but all four run "at once" → illusion of **many virtual CPUs**.
- Raises **policy** questions: if two programs want to run, which goes first?

## 3. Virtualizing Memory
```c
int *p = malloc(sizeof(int));
printf("(%d) address of p: %p\n", getpid(), p);
*p = 0;
while (1) { Spin(1); *p = *p + 1; printf("(%d) p: %d\n", getpid(), *p); }
```
- Two instances print the **same address** but update values **independently**.
- Each process has its own **private virtual address space**; the OS maps it onto shared physical memory.

> 💡 To see identical addresses, disable address-space randomization (ASLR).

## 4. Concurrency
```c
volatile int counter = 0;
void *worker(void *arg) { for (int i = 0; i < loops; i++) counter++; return NULL; }
// main: create 2 threads running worker, join both, print counter
```
- `loops = 1000` → 2000 (fine). `loops = 100000` → sometimes 143012, 137298...
- `counter++` is **three instructions** (load, increment, store), not atomic → interleavings lose updates.
- Problem appears in the OS itself (many things at once) and in multi-threaded apps.

## 5. Persistence
```c
int fd = open("/tmp/file", O_WRONLY | O_CREAT | O_TRUNC, S_IRWXU);
write(fd, "hello world\n", 13);
close(fd);
```
- DRAM is **volatile** → need hardware (HDD/SSD) + software (**file system**) to store data persistently.
- Disk is **not** virtualized per app: files are meant to be **shared**.
- FS must handle: where to put data on disk, metadata, performance (batching writes), and **crashes** (journaling, copy-on-write).

## 6. Design Goals
- **Abstractions**: make the system convenient (assembly → C → OS abstractions).
- **Performance**: minimize overhead (time + space).
- **Protection / isolation** between apps and between apps and the OS.
- **Reliability**: OS must run non-stop.
- Others: energy efficiency, security, mobility.

## 7. A Little History
| Era | Key idea |
|---|---|
| Early: libraries | OS = set of common functions; **batch** processing by a human operator |
| Beyond libraries: protection | **System call** vs procedure call: trap to kernel, raise **hardware privilege level** (user mode → kernel mode), return-from-trap |
| Multiprogramming (minicomputers) | Switch between jobs while one waits for I/O → better CPU utilization; needs memory protection + concurrency care. **UNIX** (Thompson & Ritchie) |
| Modern era (PCs) | Early DOS / Mac OS lacked protection; later Mac OS X (UNIX-based), Windows NT, Linux brought back the good ideas |

> 💡 Mechanism (system call / trap) vs. policy (scheduling) — a separation used throughout the book.
