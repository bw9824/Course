# Intro to FPGAs and Digital Logic

## 1. FPGA
| Part | Role |
|---|---|
| **Logic blocks** | Programmable logic (LUTs that mimic AND/OR/XOR…, plus flip-flops) |
| **Programmable interconnect** | Switchable wires connecting the blocks |
| **I/O blocks** | Connect to the pins / outside world |
| **Block RAM** | On-chip memory |
| **DSP units** | Hardwired multipliers / adders for math |
- "Field programmable": it's reconfigured **after manufacturing**, by the user, as many times as you want.

**Using an FPGA**
```
Verilog ──► FPGA CAD tools (Quartus) ──► bitstream ──► download to board (DE1-SoC)
   └──► Simulation (ModelSim) — check behavior first
```

> 💡 **FPGA vs ASIC vs CPU**: an ASIC is custom hardware (fastest, but fixed and very expensive to make); a CPU runs software (flexible, but one instruction at a time); an FPGA sits in between — real parallel hardware that can be rewritten. Used for prototyping ASICs, networking, signal processing, low-latency trading, etc.

## 2. Digital vs Analog
| | Digital | Analog |
|---|---|---|
| Values | Discrete: 0 / 1 (no/yes, off/on, low/high, FALSE/TRUE) | Continuous over a range |
| Noise | A small input error doesn't change the 0/1 meaning | A small input error → large output error |
- Digital: more **accurate and reliable**; self-contained building blocks that are **easy to cascade**; computers are digital inside.
- But **sensors and actuators** (the interface to the real world) are often analog.
- Typical voltages: 0 = 0 V, 1 = 5 V (older) or **3.3 V** (modern).

## 3. Basic Gates
| Gate    | Rule                                 | Notation             |
| ------- | ------------------------------------ | -------------------- |
| **AND** | Out = 1 only if **all** inputs are 1 | `A·B`, `AB`, `A & B` |
| **OR**  | Out = 1 if **any** input is 1        | `A + B`, `A \| B`    |
| **NOT** | Out = opposite of input              | `Ā`, `A'`, `~A`      |
- AND / OR can have more than 2 inputs.

## 4. Combinational vs Sequential
|                   | Combinational           | Sequential                                  |
| ----------------- | ----------------------- | ------------------------------------------- |
| Feedback          | **None**                | Outputs fed back as inputs                  |
| Output depends on | **Current inputs only** | Current inputs **+ stored state** (history) |
| Example           | Adder, multiplexer      | Counter, vending machine                    |

## 5. "Black Box" Design → Truth Table
First decide **what** the circuit must do, treating it as a black box; write every input combination and the output.

**Odd parity**: Out = 1 when an **odd number** of inputs are 1.

| A | B | C | Out |
|---|---|---|---|
| 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 1 |
| 0 | 1 | 0 | 1 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 0 | 1 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 0 |
| 1 | 1 | 1 | 1 |
→ `Out = A'B'C + A'BC' + AB'C' + ABC` = `A ⊕ B ⊕ C` (XOR)

```
A⊕B = A'B + AB'; (A⊕B)' = A'B' + AB (XNOR)

(A⊕B)⊕C = (A⊕B)'·C + (A⊕B)·C'
        = (A'B' + AB)·C + (A'B + AB')·C'
        = A'B'C + ABC + A'BC' + AB'C'
```

## 6. Boolean Algebra Basics
- Values: **0, 1**. Variables: A, B, C… Operations: **NOT, AND, OR**.

| X | Y | X·Y | X+Y | X' |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 1 |
| 0 | 1 | 0 | 1 | 1 |
| 1 | 0 | 0 | 1 | 0 |
| 1 | 1 | 1 | 1 | 0 |

## 7. Truth Table → Equation (Sum of Products)
1. Take every row where the output is **1**.
2. Write one **AND (product) term** per row: variable = 0 → complemented (`A'`), variable = 1 → plain (`A`).
3. **OR** the terms together.

**Half adder** (A + B):

| A | B | Carry | Sum |
|---|---|---|---|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 0 | 1 |
| 1 | 0 | 0 | 1 |
| 1 | 1 | 1 | 0 |
- `Carry = AB`
- `Sum = A'B + AB'` = **A ⊕ B** (XOR)

## 8. Full Adder and Simplification
| A   | B   | C   | Cout | Sum |
| --- | --- | --- | ---- | --- |
| 0   | 0   | 0   | 0    | 0   |
| 0   | 0   | 1   | 0    | 1   |
| 0   | 1   | 0   | 0    | 1   |
| 0   | 1   | 1   | 1    | 0   |
| 1   | 0   | 0   | 0    | 1   |
| 1   | 0   | 1   | 1    | 0   |
| 1   | 1   | 0   | 1    | 0   |
| 1   | 1   | 1   | 1    | 1   |
- `Sum = A'B'C + A'BC' + AB'C' + ABC` = **A ⊕ B ⊕ C**
- `Cout = A'BC + AB'C + ABC' + ABC`
```
simplify Cout:
ABC' + ABC = AB(C' + C) = AB
A'BC + ABC = BC(A' + A) = BC   ← reuse ABC
AB'C + ABC = AC(B' + B) = AC   ← reuse ABC again
Cout = AB + AC + BC
```
- Each simplified term covers **two rows**; a row (here `ABC`) may be covered by **more than one** term (because `X + X = X`).
- Check: put a 1 in every row where some product term is true → must match the original table.

## 9. Representations of One Function
`F = X' + YZ`:

| X | Y | Z | F |
|---|---|---|---|
| 0 | 0 | 0 | 1 |
| 0 | 0 | 1 | 1 |
| 0 | 1 | 0 | 1 |
| 0 | 1 | 1 | 1 |
| 1 | 0 | 0 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 0 |
| 1 | 1 | 1 | 1 |
- Circuit: X → NOT; Y, Z → AND; both into an OR → F.
- Same function, four views: **equation, truth table, circuit, Verilog**.

## 10. Why Minimize Logic?
Goal: fewer **literals** (gate inputs), fewer **gates**, fewer **levels** of gates.

| Benefit  | Why                                                                   |
| -------- | --------------------------------------------------------------------- |
| Faster   | Fewer inputs → faster gates; fewer levels → shorter propagation delay |
| Feasible | Fan-in (inputs per gate) is limited in some technologies              |
| Cheaper  | Fewer gates → lower manufacturing cost                                |
