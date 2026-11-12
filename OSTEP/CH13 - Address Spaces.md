# CH13 - The Abstraction: Address Spaces

## Big Picture
- Early machines: no memory abstraction — OS at physical address 0, one program in the rest.
- Multiprogramming + time sharing → many processes in memory at once → need **protection**.
- Abstraction: the **address space** — the running program's **view of memory**. Every address a program sees is **virtual**.

---

## 1. Evolution
| Era | Memory |
|---|---|
| Early systems | OS (0–64KB) + current program (rest) |
| Multiprogramming | Switch on I/O → better CPU utilization |
| Time sharing | Interactivity; first attempt: save whole memory to disk on switch → **too slow**; instead leave processes in memory and switch between them |

## 2. The Address Space
```
0KB   | Program Code |  static, at the top
1KB   | Heap         |  grows downward (malloc)
      |    ↓         |
      |   (free)     |
      |    ↑         |
15KB  | Stack        |  grows upward (locals, args, return values)
16KB
```
- Code, heap, stack — heap and stack at opposite ends so both can grow.
- (With threads, multiple stacks — later.)
- A process loaded at address 0 in its own view may actually be at physical 320KB → **virtualizing memory**.

> 💡 **Isolation** is a key principle: one process can't affect another; the OS is protected from processes. Microkernels push this further.

## 3. Goals of Virtual Memory
| Goal | Meaning |
|---|---|
| **Transparency** | Program doesn't notice memory is virtualized |
| **Efficiency** | Time (not slow) and space (small structures); needs hardware like TLBs |
| **Protection** | Processes can't touch others' memory or the OS — isolation |

## 4. Every Address You See Is Virtual
```c
printf("location of code : %p\n", main);
printf("location of heap : %p\n", malloc(100e6));
int x = 3;
printf("location of stack: %p\n", &x);
```
```
location of code : 0x1095afe50
location of heap : 0x1096008c0
location of stack: 0x7fff691aea64
```
- Code first, then heap, stack at the far end — all **virtual**; only the OS and hardware know the physical addresses.
