# CH17 - Free-Space Management

## Big Picture
- Managing **variable-sized** free space (malloc library, OS with segmentation) → **external fragmentation**.
- Fixed-size units (paging) are easy; variable sizes are hard.
- Crux: minimize fragmentation while keeping time/space overheads low.

---

## 1. Assumptions
- Interface: `void *malloc(size_t size)`, `void free(void *ptr)` — free takes no size.
- Managed region = the **heap**; free space tracked in a **free list**.
- Once handed out, memory **can't be relocated** (no compaction) — the C program owns the pointer.
- Fixed-size heap (for simplicity).

## 2. Low-Level Mechanisms
### Splitting and Coalescing
```
30-byte heap: [free 0-9][used 10-19][free 20-29]
free list: head → (addr 0, len 10) → (addr 20, len 10) → NULL
```
- **Splitting**: request 1 byte → split a free chunk; return 1 byte, keep 9 on the list.
- **Coalescing**: on free, merge with adjacent free chunks → (addr 0, len 30) instead of three 10-byte chunks.

### Tracking the Size of Allocated Regions
- A **header** just before the returned pointer:
```c
typedef struct { int size; int magic; } header_t;   // magic = integrity check
void free(void *ptr) {
    header_t *hptr = (header_t *) ptr - 1;
    assert(hptr->magic == 1234567);
    // free region size = hptr->size + sizeof(header_t)
}
```
- `malloc(N)` really searches for **N + header size**.

### Embedding a Free List
- The list lives **inside the free space** itself:
```c
typedef struct __node_t { int size; struct __node_t *next; } node_t;
node_t *head = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_ANON|MAP_PRIVATE, -1, 0);
head->size = 4096 - sizeof(node_t);
head->next = NULL;
```
- Allocation of 100 bytes → header (8B) + 100B carved from the front; remaining chunk shrinks.
- Free → chunk inserted back at the head; without coalescing the list becomes **fragmented**. Fix: go through the list and merge neighbors (keeping the list **sorted by address** helps).

### Growing the Heap
- Out of space → return NULL, or call `sbrk()` to get more from the OS.

## 3. Basic Strategies
| Strategy | How | Pros / Cons |
|---|---|---|
| **Best fit** | Smallest chunk ≥ request | Less waste; full search is slow |
| **Worst fit** | Largest chunk | Tries to keep big leftovers; full search; studies show it's bad |
| **First fit** | First chunk big enough | Fast; pollutes start of list with small chunks (address-ordered list helps coalescing) |
| **Next fit** | First fit but starting where the last search ended | Spreads searches across the list |

Example: free list 10 → 30 → 20, request 15:
- Best fit → 20 (leaves 10 → 30 → 5); worst fit → 30 (leaves 10 → 15 → 20); first fit → 30.

## 4. Other Approaches
### Segregated Lists
- Separate lists for **popular sizes** (e.g., kernel objects); general allocator for the rest.
- **Slab allocator** (Bonwick, Solaris): object caches per kernel object type (locks, inodes); slabs taken from the general allocator; objects kept **pre-initialized** → avoids init/destroy cost.

### Buddy Allocation
- Free memory = one big 2^N space; a request **recursively halves** until the block just fits.
```
64KB → 32KB + 32KB → 16KB + 16KB → 8KB + 8KB   (request 7KB → 8KB block)
```
- ✅ **Coalescing is easy**: a block's buddy differs in **only one bit** of the address; if buddy free → merge, recursively up.
- ❌ **Internal fragmentation** (only power-of-two sizes).

### Other Ideas
- Searching lists is slow → use balanced binary trees, splay trees, partially-ordered trees.
- **Multi-thread scalability**: per-thread arenas (Hoard, **jemalloc**, glibc).

> 💡 There's no perfect allocator: fast, space-efficient, scalable, works for any workload — pick trade-offs.
