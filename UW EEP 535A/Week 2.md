# Boolean Algebra, Timing, Intro to Verilog

## 1. Basic

| Origin               | Simplify           | Description                                  |
| -------------------- | ------------------ | -------------------------------------------- |
| X+~X                 | 1                  |                                              |
| X~X                  | 0                  |                                              |
| X+0, X+X, XX, X(X+Y) | X                  |                                              |
| X+YZ                 | (X+Y)(X+Z)         |                                              |
| X+~XY                | X+Y                | Using the previous formulas<br>= (X+~X)(X+Y) |
| ~(X + Y)<br>~(X * Y) | ~X * ~Y<br>~X + ~Y | DeMorgan’s Law                               |

| Law          | OR form                     | AND form                  |
| ------------ | --------------------------- | ------------------------- |
| Commutative  | `X + Y = Y + X`             | `XY = YX`                 |
| Associative  | `X + (Y + Z) = (X + Y) + Z` | `X(YZ) = (XY)Z`           |
| Distributive | `X(Y + Z) = XY + XZ`        | `X + YZ = (X + Y)(X + Z)` |

## 2. Absorption Laws
| Expression        | Result    | Why                    |
| ----------------- | --------- | ---------------------- |
| `X + XY`          | **X**     | `X(1 + Y) = X`         |
| `XY + XY'`        | **X**     | `X(Y + Y') = X`        |
| `X + X'Y`         | **X + Y** | `(X + X')(X + Y)`      |
| `X(X + Y)`        | **X**     | `XX + XY = X + XY = X` |
| `(X + Y)(X + Y')` | **X**     | `X + YY' = X`          |
| `X(X' + Y)`       | **XY**    | `XX' + XY = XY`        |

## 3. Worked Simplifications
**① `F = XYZ + X'Y + XYZ'`**
```
XYZ + XYZ' = XY(Z + Z') = XY
F = XY + X'Y = Y(X + X') = Y
```

**② `F = X'YZ + XZ`**
```
F = Z(X'Y + X) = Z(X + Y)      ← X + X'Y = X + Y
  = XZ + YZ
```

**③ `F = (X + Y' + XY')(XY + X'Z + YZ)`**
```
X + Y' + XY' = X + Y'                               (XY' absorbed)
F = (X + Y')(XY + X'Z + YZ)
  = XY + XX'Z + XYZ + XY'Y + X'Y'Z + Y'YZ
  = XY + XYZ + X'Y'Z                                (XX' = YY' = 0)
  = XY + X'Y'Z                                      (XYZ absorbed by XY)
```

## 4. DeMorgan's Law
```
(X + Y)' = X' · Y'        NOR  = AND of inverted inputs
(X · Y)' = X' + Y'        NAND = OR of inverted inputs
```

| X | Y | X' | Y' | (X+Y)' | X'·Y' | (X·Y)' | X'+Y' |
|---|---|---|---|---|---|---|---|
| 0 | 0 | 1 | 1 | 1 | 1 | 1 | 1 |
| 0 | 1 | 1 | 0 | 0 | 0 | 1 | 1 |
| 1 | 0 | 0 | 1 | 0 | 0 | 1 | 1 |
| 1 | 1 | 0 | 0 | 0 | 0 | 0 | 0 |

- `Z = A'B'C + A'BC + AB'C + ABC'`
- `Z' = (A + B + C')(A + B' + C')(A' + B + C')(A' + B' + C)` → SOP (Sum of Products) turns into POS (Product of Sums).

**Example**: `F = (XY + Z)(Y' + X'Z)(XY' + Z')`, find F'.
```
F' = (XY + Z)' + (Y' + X'Z)' + (XY' + Z')'
   = (X' + Y')Z'  +  Y(X + Z')  +  (X' + Y)Z
   = X'Z' + Y'Z' + XY + YZ' + X'Z + YZ
   = X'(Z' + Z) + Y(Z' + Z) + Y'Z' + XY
   = X' + Y + Y'Z' + XY
   = X' + Y + Z'              (Y + Y'Z' = Y + Z'; XY absorbed by Y)
→  F = X Y' Z
```

## 5. Equations → Circuits
- `F = X'Y'Z + X'Y + XYZ'` → three ANDs (`X'Y'Z`, `X'Y`, `XYZ'`) feeding one OR; inverters make `X'`, `Y'`, `Z'`.
- `F = XY + X(WZ + WZ')` = `XY + XWZ + XWZ'` → simplifies to `XY + XW = X(Y + W)`: two 2-input ANDs + OR, or one OR + one AND.

## 6. Circuit Timing
- **Simple model: each gate reacts after a fixed delay.**
- Example: `E = A·B·C`, `F = E + D`, `Out = F'`, with B = C = 1, D = 0. A goes 0 → 1:
```
A    : ___|‾‾‾‾‾‾‾‾‾‾
E    : _____|‾‾‾‾‾‾‾‾      +1 gate delay
F    : _______|‾‾‾‾‾‾      +2
Out  : ‾‾‾‾‾‾‾‾‾|_____     +3
```
- Delay accumulates **per level** of logic → why fewer levels = faster circuits.

## 7. Hazards / Glitches
Autopilot Engaged logic: `AE = CAR·PIC' + PAR·PIC`
(CAR = copilot autopilot request, PIC = pilot in charge, PAR = pilot autopilot request)
- CAR = PAR = 1 → AE should **stay 1** no matter what PIC does.
- PIC goes 1 → 0:
	- A rises after **1** delays
	- B = `CAR·PIC'` rises after 2 delay (inverter + AND).
	- C = `PAR·PIC` falls after **1** delays.
	- For one gate delay, **both** AND outputs are 0 → **AE glitches to 0**. The autopilot disengages briefly. ⚠️
- **Fix**: add the redundant (consensus) term `CAR·PAR`: `AE = CAR·PIC' + PAR·PIC + CAR·PAR`. Logically identical, but it holds AE at 1 during the transition.

> 💡 This is why synchronous designs sample signals only on **clock edges** after everything has settled (flip-flops, Week 4).

## 8. Verilog
- **Hardware description language (HDL)**: describes hardware, it's not a program that runs line by line.
	- Simulate behavior **before** spending time implementing → find bugs early.
	- Tools automatically create the implementation (synthesis).
- Syntax similar to C; VHDL (the other HDL) is similar to Ada.
- **SystemVerilog** = modern superset, cleaner. Used in this class.

## 9. Three Ways to Write the Same AOI Gate
`F = ((A·B) + (C·D))'` (AND-OR-INVERT)
```systemverilog
// 1. Expression (structural/dataflow)
module AOI (F, A, B, C, D);
	output logic F;
	input  logic A, B, C, D;
	
	assign F = ~((A & B) | (C & D));
endmodule

// 2. Internal wires: "logic" names each internal node
module AOI (F, A, B, C, D);
	output logic F;
	input  logic A, B, C, D;
	logic AB, CD, O;
	
	assign AB = A & B;
	assign CD = C & D;
	assign O  = AB | CD;
	assign F  = ~O;
endmodule

// 3. Gate level: built-in primitives, output first
module AOI (F, A, B, C, D);
	output logic F;
	input  logic A, B, C, D;
	logic AB, CD, O;
	
	and a1 (AB, A, B);      // a1 = instance name
	and a2 (CD, C, D);
	or  o1 (O, AB, CD);
	not n1 (F, O);
endmodule
```

| Operator         | Meaning                                                                                                                            |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `&` `\|` `~` `^` | AND, OR, NOT, XOR                                                                                                                  |
| `assign`         | Continuous assignment: output always equals the expression (it's a wire, not a one-time copy) Example: A=0; F=A; A=1; (Output F=1) |

## 10. Hierarchy: 2:1 Mux from AOI
- **Mux**: `V = I` when SEL = 1, `V = J` when SEL = 0 → `V = I·SEL + J·SEL'`.
```systemverilog
module MUX2 (V, SEL, I, J);
  output logic V;
  input  logic SEL, I, J;
  logic SELB, VB;
  not G1 (SELB, SEL);
  AOI G2 (.F(VB), .A(I), .B(SEL), .C(SELB), .D(J));   // named port connections
  not G3 (V, VB);
endmodule
```
- AOI outputs `(I·SEL + J·SEL')'`; the final inverter undoes it.
- **Named ports** `.port(signal)` are safer than positional order. `.V` is shorthand for `.V(V)`.
```systemverilog
AOI G2 (VB, I, SELB, SEL, J); // The order incorrectly, result will be wrong
AOI G2 (.F(VB), .A(I), .B(SEL), .C(SELB), .D(J));
AOI G2 (.F(VB), .C(SELB), .D(J), .A(I), .B(SEL));  // Different order, same result
```

## 11. Testbenches
```systemverilog
module MUX2TEST;            // no ports: it's the top-level test
	logic SEL, I, J, V;

	initial begin             // stimulus
		SEL = 1; I = 0; J = 1;
		#10 I = 1;              // #10 = wait 10 time units
		#10 SEL = 0; J = 0;
		#10 J = 1;
	end

	MUX2 M (.V, .SEL, .I, .J);   // device under test
endmodule
```

| Time | SEL | I | J | V |
|---|---|---|---|---|
| 0–10 | 1 | 0 | 1 | 0 (= I) |
| 10–20 | 1 | 1 | 1 | 1 (= I) |
| 20–30 | 0 | 1 | 0 | 0 (= J) |
| 30– | 0 | 1 | 1 | 1 (= J) |
- Test vectors → DUT → check outputs (waveforms in ModelSim).
- Naming convention: `name_testbench` / `name_tb`.

## 12. NAND and NOR
| X | Y | NAND `(XY)'` | NOR `(X+Y)'` |
|---|---|---|---|
| 0 | 0 | 1 | 1 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 0 |
- Symbol: AND / OR with a small circle (= inversion) on the output.
- By DeMorgan: NAND = OR with inverter on the inputs; NOR = AND with inverter on the inputs.

> 💡 In CMOS, NAND and NOR are the natural gates (fewest transistors); AND = NAND + inverter.
