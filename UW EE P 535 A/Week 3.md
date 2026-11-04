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
- **Sum of Products (SOP)**: OR of AND terms, inversions only on the inputs ( **NO**`(YZ)'`). e.g. `F = X' + YZ' + XYZ (→ X' + Y, X' + X·(YZ) = X' + YZ)`.
- Implementation: inverters on inputs → AND level → OR level (= NAND-NAND).

| X \ YZ | 00  | 01  | 11  | 10  |
| ------ | --- | --- | --- | --- |
| 0      | 1   | 1   | 1   | 1   |
| 1      | 0   | 0   | 1   | 1   |
- K-Maps
	- Circle 0: The entire column where X = 0 is 1, so it is **X’**
	- Circle 1: The columns YZ = 11 and 10 (i.e., Y = 1) are both 1, so it is **Y**
	- F = X’ + Y
- **On-set**: input patterns where F = 1. **Off-set**: patterns where F = 0.
	- SOP is written from the on-set; product-of-sums (POS) from the off-set.

## 6. The Uniting Theorem
`A(B' + B) = A` → `AB' + AB = A`
- If two on-set rows differ in **exactly one variable**, that variable drops out.
- `F = AB' + AB = A` (B changes inside the on-set → eliminated)
- `G = A'B' + AB' = B'` (A changes → eliminated)
- K-maps arrange the truth table so these "differ by one" rows are **physically adjacent**.

## 7. K-Map Layout
- Order rows/columns in **Gray code**: `00, 01, 11, 10` → neighbors differ in one bit.
- Column labels AB = `00 01 11 10`; the **A** bracket covers the right two columns, the **B** bracket the middle two.
- Rows: 1 variable (C) = `0, 1`; 2 variables (CD) = `00, 01, 11, 10`.
- **Adjacency wraps around**: left edge ↔ right edge, top ↔ bottom (a torus). The 4 corners are adjacent too.
- More than 4 variables → computer tools.

**Rules**
1. Circle groups of **1, 2, 4, 8** adjacent 1s (rectangles, powers of 2).
2. Make groups **as large as possible**; as **few** as possible.
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

## 9. K-Map Simplification (filled in)
| Map                                             | Result                                      |
| ----------------------------------------------- | ------------------------------------------- |
| 2-var: column A all 1s                          | **F = A**                                   |
| 2-var: row B' all 1s                            | **G = B'**                                  |
| Full-adder carry                                | **Cout = AB + ACin + BCin**                 |
| 3-var: columns `11` and `10` all 1s             | **F = A**                                   |
| `F = A'B'C' + AB'C' + AB'C + ABC`               | **F = B'C' + AC** (`B'C'` wraps left↔right) |
| `F'` simply replace 1's with 0's and vice versa | **F' = BC' + A'C**                          |
- For F' just group the **0s** of F instead of the 1s.

**4-variable example**: `F = A'D + BD + B'C + AB'D'`
```
AB→     00 01 11 10
CD=00    0  0  0  1
CD=01    1  1  1  0
CD=11    1  1  1  1
CD=10    1  0  0  1
F = A'D + BD + B'C + AB'D'      (already minimal, 4 groups)
```

**`F = (A ⊕ C)·D + AC'D' + A'BCD'`**
```
AB→     00 01 11 10
CD=00    0  0  1  1
CD=01    0  0  1  1
CD=11    1  1  0  0
CD=10    0  1  0  0
F = AC' + A'CD + A'BC   = AC' + A'C(B + D)
```

**Multiple solutions**
```
AB→     00 01 11 10
CD=00    1  1  0  0
CD=01    1  1  0  0
CD=11    0  1  1  1
CD=10    0  0  1  1
F = A'C' + AC + A'BD    or    F = A'C' + AC + BCD
```
- The `A'BCD` cell can join either neighbor → both answers are minimal.

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
- **Y** has no big groups (checkerboard → XOR-like): `Y = A ⊕ C ⊕ (BD)`, If there are an odd number of 1s, Y = 1; if there are an even number of 1s, Y = 0.; as SOP `Y = AC'D' + A'CD' + AB'C' + A'B'C + A'BC'D + ABCD`
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
- Outputs: **Tie** = 1 if same move; **Winner** = ID of the winner (here 1 = player CD wins; don't care when tied).
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

