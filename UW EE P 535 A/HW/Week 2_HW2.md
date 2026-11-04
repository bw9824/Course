# HW 2

## Problem 1
**Q:** For the following truth tables, provide the corresponding Boolean equation. Do not simplify the equations.

| A B C | Output1 | Output2 | A B | Output3 |
| ----- | ------- | ------- | --- | ------- |
| 0 0 0 | 0       | 1       | 0 0 | 1       |
| 0 0 1 | 1       | 0       | 0 1 | 0       |
| 0 1 0 | 0       | 0       | 1 0 | 1       |
| 0 1 1 | 1       | 1       | 1 1 | 1       |
| 1 0 0 | 0       | 1       |     |         |
| 1 0 1 | 0       | 0       |     |         |
| 1 1 0 | 0       | 0       |     |         |
| 1 1 1 | 1       | 0       |     |         |
**Ans:** Sum of minterms (one AND term per row with output 1):
- `Output1 = A'B'C + A'BC + ABC`
- `Output2 = A'B'C' + A'BC + AB'C'`
- `Output3 = A'B' + AB' + AB`

## Problem 2
**Q:** For the following equations, create the circuit diagrams and the truth tables. Do not simplify the equations.
- `F = X·Y' + X'·(Y + Z)'`
- `G = A'B + A·B'·C' + A'·C' + BC' + C`
- `H = AD·(C + B)'·(D' + B) + E`

**Ans:**
- F = 1 for XYZ = 000, 100, 101 (else 0)
![[EEP535_HW2_P2_F.png|450]]
- G = **1 for every input** (the `+ C` and `C'` terms cover everything)
![[EEP535_HW2_P2_G.png|450]]
- H = **E** (the AD… term is always 0: `(C+B)'` needs B = 0, then `(D'+B)` needs D = 0, but AD needs D = 1)
![[EEP535_HW2_P2_H.png|450]]

## Problem 3
**Q:** Use DeMorgan's to compute the inverse of the three expressions in problem 2. Then, simplify the equations.
**Ans:**
```
F' = (X·Y')' · (X'·(Y+Z)')'
   = (X' + Y)(X + Y + Z)
   = X'Y + X'Z + XY + Y + YZ
   = Y + X'Z
G' = 0            (G = 1)
H' = (AD(C+B)'(D'+B))' · E' = E'     (first term is always 0)
```
- DeMorgan: flip AND ↔ OR and invert every literal.

## Problem 4
**Q:** Using the theorems of Boolean algebra, simplify the following functions.
- `F = AB + A·B'·C' + AC + A'`
- `G = ( X(Y' + Z')(X + Y) + Y(X + Z') )'`
- `H = XY' + XYZ' + YZ' + XYZ + XZ' + X'YZ`

**Ans:**
```
F = A(B + B'C' + C) + A' = A(B + C' + C) + A' = A + A' = 1

G = ( X(Y'+Z') + Y(X+Z') )'         ← X(X+Y) = X
  = ( XY' + XZ' + XY + YZ' )'
  = ( X + YZ' )'
  = X'(Y' + Z)
  = X'Y' + X'Z

H = X(Y' + YZ' + YZ + Z') + YZ' + X'YZ
  = X + YZ' + X'YZ
  = X + YZ' + YZ                     ← X + X'A = X + A
  = X + Y
```

## Problem 5
**Q:** Convert the following circuit to a Boolean equation. Then, simplify this equation.
![[EEP535_HW2_P5_circuit_q.png|450]]
**Ans:** `F = (AB + A'C) · (BC' + AB')'` → **`F = A'C + BC`**
```
F = (AB + A'C)(BC' + AB')'
(BC' + AB')' = (B' + C)(A' + B) = A'B' + A'C + BC
F = (AB + A'C)(A'B' + A'C + BC)
  = ABC + A'B'C + A'C + A'BC
  = ABC + A'C
  = A'C + BC
```

## Problem 6
**Q:** For the following Verilog code, draw the corresponding circuit diagram for "bigMod". Your circuit should only involve standard gates (AND, OR, NAND, NOR, XOR, Inverter), and the variable names Val, X, Y, and Z. DO NOT SIMPLIFY THE CIRCUIT. Note: all of the code is legal Verilog, with no errors.
```systemverilog
module tstMod(F, A, B, C);
	output logic F;
	input  logic A, B, C;
	assign F = A | (B & ~C);
endmodule

module bigMod(Val, X, Y, Z);
	output logic Val;
	input  logic X, Y, Z;
	logic        T;
	tstMod T1 (.F(T),   .A(X), .B(1'b1), .C(Y));
	tstMod T2 (.F(Val), .A(Z), .B(T),    .C(X));
endmodule
```
**Ans:** Substitute the named ports of each instance:
- T1: `T = X + (1 · Y')`
- T2: `Val = Z + (T · X')`
- Draw two copies of the tstMod gates (NOT + AND + OR); keep the AND with constant 1 (no simplifying).
![[EEP535_HW2_P6_bigMod.png|450]]
