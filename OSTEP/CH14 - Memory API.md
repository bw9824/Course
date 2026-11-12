# CH14 - Interlude: Memory API

## Big Picture
- Two kinds of memory in C: **stack** (automatic, managed by the compiler) and **heap** (explicit, managed by the programmer).
- Interfaces: `malloc()` / `free()`. Many common bugs come from misusing them.

---

## 1. Types of Memory
```c
void func() {
    int x;                                   // stack: freed on return
    int *p = (int *) malloc(sizeof(int));    // pointer on stack, int on heap
}
```

## 2. malloc()
```c
#include <stdlib.h>
void *malloc(size_t size);
double *d = (double *) malloc(sizeof(double));
```
- Returns a pointer to new space, or **NULL** on failure.
- `sizeof` is a **compile-time** operator:
```c
int *x = malloc(10 * sizeof(int));
printf("%d\n", sizeof(x));        // 4 or 8 — size of the pointer, not 40!
int y[10];
printf("%d\n", sizeof(y));        // 40 — static array, compiler knows
```
- Strings: `malloc(strlen(s) + 1)` — **+1 for the end-of-string character**.
- Cast `(int *)` is not needed in C; just tells the reader.

## 3. free()
```c
int *x = malloc(10 * sizeof(int));
...
free(x);   // no size: the library tracks it
```

## 4. Common Errors
| Error | Example / Effect |
|---|---|
| Forgetting to allocate | `char *dst; strcpy(dst, src);` → segfault. Fix: `malloc` first, or `strdup()` |
| Not allocating enough (**buffer overflow**) | `malloc(strlen(src))` — off by one; may "work" but corrupts memory; security hole |
| Forgetting to initialize | Read uninitialized heap → **uninitialized read** (garbage) |
| Forgetting to free | **Memory leak** — deadly in long-running programs / the OS |
| Freeing before done | **Dangling pointer** → later use crashes or overwrites |
| Freeing repeatedly | **Double free** → undefined behavior |
| Calling free incorrectly | **Invalid free** — passing a pointer not from malloc |

> 💡 "It compiled" or "it ran" ≠ correct. Use tools: **purify**, **valgrind**.

> 💡 Short-lived program: OS reclaims all memory on exit (two levels of memory management: OS → process, malloc/free → within process). Still, freeing is a good habit.

## 5. Underlying OS Support
- `malloc`/`free` are **library calls**, not system calls.
- Library asks the OS for memory via:
  - **`brk` / `sbrk`**: move the program's **break** (end of heap). Never call directly.
  - **`mmap()`**: create an anonymous region (swap-backed) usable as heap.

## 6. Other Calls
- `calloc()`: allocates **and zeroes**.
- `realloc()`: grows/shrinks a region (may copy to a new place).
