# CH44 - Flash-based SSDs

## Big Picture
- **SSD**: no moving parts; built from transistors, retains data without power (unlike DRAM).
- Technology: **NAND flash**. Odd properties:
  - To write a **page**, must first **erase** a larger **block** — expensive.
  - Writing too often → **wear out**.
- Crux: build a fast, reliable SSD despite expensive erases and wear-out → the **FTL** (Flash Translation Layer).

---

## 1. Storing a Single Bit
| Type | Bits per cell |
|---|---|
| **SLC** (single-level cell) | 1 — fastest, most durable, most expensive |
| **MLC** (multi-level) | 2 |
| **TLC** (triple-level) | 3 |
| QLC | 4 |

## 2. From Bits to Banks/Planes
- Flash chip → **banks/planes** → **blocks** (128 KB or 256 KB) → **pages** (4 KB).
- Block = erase unit; page = read/program unit.

## 3. Basic Flash Operations
| Op | Unit | Latency (SLC / MLC / TLC) | Notes |
|---|---|---|---|
| **Read** (page) | Page | 25 / 50 / ~75 µs | Random access device |
| **Program** (page) | Page | 200–300 / 600–900 / 900–1350 µs | Only on erased page; 1 → 0 |
| **Erase** (block) | Block | 1.5–2 / ~3 / ~4.5 ms | Sets all bits to 1; destroys block contents |

- Page states: **INVALID → (erase) → ERASED → (program) → VALID**.
- To overwrite a page: copy out other live pages in the block, erase, reprogram.

## 4. Flash Performance and Reliability
- **Wear out**: each erase/program cycle adds charge → MLC ~10,000 P/E cycles, SLC ~100,000.
- **Disturbance**: reading/programming may flip bits in **neighboring** pages (**read disturb**, **program disturb**) → program pages **in order** within a block.

## 5. From Raw Flash to Flash-Based SSDs
- SSD = flash chips + volatile memory (SRAM cache) + **control logic**.
- **FTL** maps logical block reads/writes → physical page reads/programs/erases.
- Goals: **performance** (parallelism across chips, low **write amplification** = flash bytes written / client bytes written) and **reliability** (**wear leveling**, sequential programming within a block).

## 6. A Bad Approach: Direct-Mapped FTL
- Logical page N → physical page N.
- Every write: read whole block, erase, program old + new pages → **terrible performance**, hot blocks **wear out quickly**.

## 7. A Log-Structured FTL
- Write to the **next free page** in the current block (like LFS); keep a **mapping table** (logical → physical) in memory.
- Example: write(100, a1), write(101, a2), write(2000, b1), write(2001, b2) → physical pages 0–3, mapping 100→0, 101→1, 2000→2, 2001→3.
- Mapping persisted (out-of-band area + periodic checkpoints) for **power loss**.

## 8. Garbage Collection
- Overwrites leave **dead** (garbage) pages.
- GC: find a block with dead pages, copy **live** pages elsewhere, **erase** the block.
- Liveness: store each page's logical address in the block; check the mapping table.
- Ideal: blocks with **only dead pages** → just erase, no migration.
- **Overprovisioning**: extra flash capacity → GC can run in the background.
- **TRIM**: FS tells the SSD a block is no longer needed → avoid copying dead data.

## 9. Mapping Table Size
- 1 TB SSD, 4KB pages, 4B entries → **1 GB** of mapping memory → too big.
### Block-Based Mapping
- Map per **block** (like bigger pages) → small table, but **small writes** force copying whole blocks → write amplification.
### Hybrid Mapping
- **Log blocks**: a few blocks page-mapped (**log table**); the rest block-mapped (**data table**).
- Periodically **merge** log blocks into data blocks:
  - **Switch merge** (best): log block contains all pages of a logical block in order → just switch pointer.
  - **Partial merge**: copy remaining pages from old block.
  - **Full merge** (worst): gather pages from many blocks.
### Page Mapping + Caching
- Keep only the **active** part of the page map in memory (exploits locality); misses cost extra flash reads.

## 10. Wear Leveling
- Spread erase/program load evenly; **long-lived data** blocks never get rewritten → FTL periodically **migrates** them so their blocks share the load.

## 11. SSD Performance and Cost
| | Random reads | Random writes | Seq reads | Seq writes |
|---|---|---|---|---|
| Samsung 840 Pro SSD | 103 MB/s | 287 MB/s | 421 MB/s | 384 MB/s |
| Seagate 600 SSD | 84 | 252 | 424 | 374 |
| Intel SSD 335 | 39 | 222 | 344 | 354 |
| Seagate Savvio 15K.3 HDD | 2 | 2 | 223 | 223 |

- SSDs: random I/O **orders of magnitude** faster; random **writes beat random reads** (log-structured FTL turns them sequential).
- Sequential gap is much smaller → HDDs still win on **cost per GB** (~10×) for bulk storage.
