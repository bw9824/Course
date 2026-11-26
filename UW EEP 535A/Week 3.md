# NAND/NOR Logic, K-Maps, Combinational Design

## 1. Bubble Manipulation
- DeMorgan law:
  - **NOR** = AND with inverted inputs: `(X + Y)' = X'Y'`
  - **NAND** = OR with inverted inputs: `(XY)' = X' + Y'`
- Simplification: `((AB)' + (CD)')' = AB · CD = ABCD`
  → a NAND-NAND-NOR chain can be just one 4-input AND.

> 💡 Trick: push bubbles through a gate and it flips AND ↔ OR. Bubbles at both ends of a wire cancel.

## 2. NAND / NOR Universality
| Gate        | With NAND only                                 | With NOR only                              |
| ----------- | ---------------------------------------------- | ------------------------------------------ |
| **NOT X**   | `NAND(X, X)`                                   | `NOR(X, X)`                                |
| **X AND Y** | `NAND(NAND(X,Y), NAND(X,Y))` — NAND + inverter | `NOR(NOR(X,X), NOR(Y,Y))` = `(X' + Y')'`   |
| **X OR Y**  | `NAND(NAND(X,X), NAND(Y,Y))` = `(X'Y')'`       | `NOR(NOR(X,Y), NOR(X,Y))` — NOR + inverter |
- Since AND, OR, NOT cover everything, NAND (or NOR) alone covers everything.

## 3. Converting Circuits to NAND / NOR
| Gate form               | Equivalent          |
| ----------------------- | ------------------- |
| AND + bubble on output  | **NAND**            |
| OR + bubbles on inputs  | **NAND** (DeMorgan) |
| OR + bubble on output   | **NOR**             |
| AND + bubbles on inputs | **NOR** (DeMorgan)  |
1. Group gates into **levels** from the output back.
2. Insert **double inversions** (two bubbles) on the wires between **alternating levels** ().
	- When a signal skips a level (enters at a level of the "wrong" parity), its bubble doesn't pair up → add an **internal inverter** (or invert that input).
```
	A ─┐                        A ─┐                         
	   AND─o──o┐                   NAND─┐
	B ─┘       │	            B ─┘    │
	           OR ── F	                NAND ── F
	C ─┐       │	            C ─┐    │
	   AND─o──o┘	               NAND─┘
	D ─┘	                    D ─┘
```
## 4. XOR and XNOR
| X   | Y   | XOR `X ⊕ Y` | XNOR `(X ⊕ Y)'` |
| --- | --- | ----------- | --------------- |
| 0   | 0   | 0           | 1               |
| 0   | 1   | 1           | 0               |
| 1   | 0   | 1           | 0               |
| 1   | 1   | 0           | 1               |
- XOR = 1, if an **odd** number of inputs are 1 → `X'Y + XY'` (parity, adder sum).
- XNOR = 1, if an **even** number are 1 → "equality" check: `XY + X'Y'`.

## 5. Two-Level Forms
- **Sum of Products (SOP)**: OR of AND terms; inversions only on the inputs (**NO** `(YZ)'` inside a term).
- Implementation: inverters on inputs → AND level → OR level (= NAND-NAND).
- Example from the slides: `F = X + YZ + XYZ`. The term `XYZ` is crossed out, because it is absorbed by `X` (`X + XYZ = X`) → **F = X + YZ**.

| X | Y | Z | F |
| --- | --- | --- | --- |
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 0 |
| 0 | 1 | 0 | 0 |
| 0 | 1 | 1 | 1 |
| 1 | 0 | 0 | 1 |
| 1 | 0 | 1 | 1 |
| 1 | 1 | 0 | 1 |
| 1 | 1 | 1 | 1 |

| X \ YZ | 00  | 01  | 11  | 10  |
| ------ | --- | --- | --- | --- |
| 0      | 0   | 0   | 1   | 0   |
| 1      | 1   | 1   | 1   | 1   |
- K-Map
	- Circle 1: the whole row X = 1 is 1 → **X**
	- Circle 2: the column YZ = 11 (both rows) is 1 → **YZ**
	- F = X + YZ
- Circuit: Y, Z → AND; that output and X → OR → F (2 levels, no inverters needed).
- **On-set**: input patterns where F = 1. **Off-set**: patterns where F = 0.
	- SOP is written from the on-set; product-of-sums (POS) from the off-set.
- **POS from the off-set (circle the 0s)**: each group becomes one OR term. A variable fixed at 0 in the group is written as-is; a variable fixed at 1 is written inverted.
	- Circle 1: X = 0, YZ = 00, 01 → **(X + Y)**
	- Circle 2: X = 0, YZ = 00, 10 → **(X + Z)**
	- F = (X + Y)(X + Z) = X + YZ (distributive law), the same function as the SOP.

## 6. The Uniting Theorem
`A(B' + B) = A` → `AB' + AB = A`
- If two on-set rows differ in **exactly one variable**, that variable drops out.
- `F = AB' + AB = A` (B changes inside the on-set → eliminated)
- `G = A'B' + AB' = B'` (A changes → eliminated)
- K-maps arrange the truth table so these "differ by one" rows are **physically adjacent**.

## 7. K-Map Layout
- Order rows/columns in **Gray code**: `00, 01, 11, 10` → neighbors differ in one bit.
- Rows: 1 variable (C) = `0, 1`; 2 variables (CD) = `00, 01, 11, 10`.
- **Adjacency wraps around**: left edge ↔ right edge, top ↔ bottom (a torus). The 4 corners are adjacent too.
- More than 4 variables → computer tools.

**Rules**
1. Circle groups of **1, 2, 4, 8** adjacent 1s (rectangles, powers of 2).
2. Make groups **as large as possible**; group number as **few** as possible.
3. Groups may **overlap**.
4. Each group → one product term: keep only variables that **don't change** inside the group.

## 8. Truth Tables → K-Maps
- `F` (2-var, on-set `10`): `F = AB'`
- `G` (3-var, on-set `001 010 011 100`):
```
AB→    00 01 11 10
C=0     0  1  0  1
C=1     1  1  0  0
G = A'B + A'C + AB'C'
```
- `H` (4-var):
```
AB→     00 01 11 10
CD=00    0  1  1  1
CD=01    0  0  0  1
CD=11    1  0  0  1
CD=10    0  1  0  1
H = AB' + B'CD + A'BD' + BC'D'
```

## 9. K-Map Simplification

### Example 1: 4-variable K-map
**Q:** Simplify `F = A'D + BD + B'C + AB'D`.
```
AB→     00 01 11 10
CD=00    0  0  0  0
CD=01    1  1  1  1
CD=11    1  1  1  1
CD=10    1  0  0  1
```
- Group 1: the two middle rows (D = 1), all 8 cells → **D**
- Group 2: the left and right columns (B = 0) in the bottom two rows (C = 1); these wrap around → **B'C**

**Ans:** `F = D + B'C`

### Example 2: K-map from a mixed expression
**Q:** Simplify `F = (A ⊕ B)·D + AC'D' + A'BCD'`.
- First expand to SOP: `A ⊕ B = AB' + A'B`, so `F = AB'D + A'BD + AC'D' + A'BCD'`.
```
AB→     00 01 11 10
CD=00    0  0  1  1
CD=01    0  1  0  1
CD=11    0  1  0  1
CD=10    0  1  0  0
```
- Group 1: row CD = 00, columns 11 and 10 → **AC'D'**
- Group 2: column 10, rows 01 and 11 → **AB'D**
- Group 3: column 01, rows 01 and 11 → **A'BD**
- Group 4: column 01, rows 11 and 10 → **A'BC**

**Ans:** `F = AC'D' + AB'D + A'BD + A'BC`

### Example 3: multiple solutions
**Q:** Find a minimal SOP for this map. `F = (A ⊕ C)·D + AC'D' + A'BCD'`.
```
AB→     00 01 11 10
CD=00    1  1  0  0
CD=01    1  1  0  0
CD=11    0  1  1  1
CD=10    0  0  1  1
```
- Group 1: top-left 2×2 square (A = 0, C = 0) → **A'C'**
- Group 2: bottom-right 2×2 square (A = 1, C = 1) → **AC**
- The cell `A'BCD` (column 01, row 11) is still uncovered. It can pair with either neighbor:
	- with the cell above it → **A'BD**
	- with the cell to its right → **BCD**

**Ans:** `F = A'C' + AC + A'BD` **or** `F = A'C' + AC + BCD`. Both use 3 terms of the same size, so both are minimal.
- Since `A'C' + AC = (A ⊕ C)'` (XNOR), `F = (A ⊕ C)' + A'BD`.

## 10. Design Example: 2-bit Adder
Inputs `AB` (N1) + `CD` (N2) → 3-bit sum `XYZ` (N3).
```
K-map X              K-map Y              K-map Z
AB→ 00 01 11 10      AB→ 00 01 11 10      AB→ 00 01 11 10
00   0  0  0  0      00   0  0  1  1      00   0  1  1  0
01   0  0  1  0      01   0  1  0  1      01   1  0  0  1
11   0  1  1  1      11   1  0  1  0      11   1  0  0  1
10   0  0  1  1      10   1  1  0  0      10   0  1  1  0
```
- **X** (carry out) `= AC + ABD + BCD`
- **Z** (LSB) `= BD' + B'D = B ⊕ D`
- **Y** (checkerboard → XOR-like): `Y = A ⊕ C ⊕ (BD)`, If there are an odd number of 1s, Y = 1; if there are an even number of 1s, Y = 0.; as SOP `Y = AC'D' + A'CD' + AB'C' + A'B'C + A'BC'D + ABCD`
```
      A   B      ← N1
  +   C   D      ← N2
  ---------
  X   Y   Z
```

| A   | C   | BD  | Y   |
| --- | --- | --- | --- |
| 0   | 0   | 0   | 0   |
| 0   | 0   | 1   | 1   |
| 0   | 1   | 0   | 1   |
| 0   | 1   | 1   | 0   |
| 1   | 0   | 0   | 1   |
| 1   | 0   | 1   | 0   |
| 1   | 1   | 0   | 0   |
| 1   | 1   | 1   | 1   |
> 💡 Checkerboard K-map patterns = XOR. K-maps only find the best **two-level** form; XOR gates can be much smaller.

## 11. Don't Cares
- Input combinations that **never happen** (or whose output doesn't matter) → `X`.
- Treat each X as 1 or 0, whichever makes groups bigger.
```
AB→     00 01 11 10
CD=00    0  0  X  0
CD=01    1  1  X  1
CD=11    1  1  0  0
CD=10    0  X  0  0
```

| Choice                               | F                                 |
| ------------------------------------ | --------------------------------- |
| All X = 0                            | `A'D + B'C'D`                     |
| All X = 1                            | `A'D + C'D + ABC' + A'BC` (worse) |
| **Use X smartly** (only `ABC'D` = 1) | **`A'D + C'D`**                   |

## 12. Design Example: Rock-Paper-Scissors
- Player 0 = `AB`, player 1 = `CD`; Rock = 00, Paper = 01, Scissors = 10; `11` unused → **don't care**.
- Outputs: **Tie** = 1 if same move; **Winner** = ID of the winner (here 0,1 = player AB, CD wins; don't care X when tied).
```
Tie                       Winner
AB→  00 01 11 10          AB→  00 01 11 10
00    1  0  X  0          00    X  0  X  1
01    0  1  X  0          01    1  X  X  0
11    X  X  X  X          11    X  X  X  X
10    0  0  X  1          10    0  1  X  X
```
- **Tie = A'B'C'D' + BD + AC**
- **Winner = A'D + AD' + BC**

