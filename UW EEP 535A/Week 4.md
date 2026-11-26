# Combinational Design, Latches, Flip-Flops, FSMs

## 1. Combinational Design Process
1. **Understand the problem**: what should it do? List inputs (data, control) and outputs; draw a block diagram.
2. **Formulate**: truth table, Boolean algebra, Verilog…
3. **Choose the implementation target** (gates, FPGA…).
4. **Implement**: K-maps, Boolean algebra, Quartus synthesis.

## 2. Example: Process Line Control
**Q (problem statement):**
- Rods of varying length (±10%) travel on a conveyor belt.
- Arm 1 pushes rods **within spec (±5%)** to one side; arm 2 pushes rods **too long** to the other side; rods **too short** stay on the belt.
- 3 light barriers (light source + photocell) as sensors A, B, C; a sensor reads 1 when tripped.
- Design combinational logic to activate the arms. Inputs: A, B, C. Outputs: `Accept` (arm 1), `Long` (arm 2).

**Sensor placement** (the slide's "draw a picture" step):
- A detects the rod's **leading edge**.
- A to B distance = spec − 5%; A to C distance = spec + 5%.
- When A trips, a rod that reaches B but not C is in spec; one that also reaches C is too long; one that doesn't reach B is too short.

**Truth table** (professor's handwritten version):

| A | B | C | Meaning | Accept | Long |
|---|---|---|---|---|---|
| 0 | 0 | 0 | No rod | 0 | 0 |
| 0 | 0 | 1 | Rod coming | 0 | 0 |
| 0 | 1 | 0 | Too short | 0 | 0 |
| 0 | 1 | 1 | Rod coming | 0 | 0 |
| 1 | 0 | 0 | Too short | 0 | 0 |
| 1 | 0 | 1 | ?? | 0 | 0 |
| 1 | 1 | 0 | **In spec** | **1** | 0 |
| 1 | 1 | 1 | **Too long** | 0 | **1** |

**K-maps** (rows A = 0 / 1, columns BC = 00 01 11 10):
```
Accept   BC→ 00 01 11 10        Long   BC→ 00 01 11 10
A=0           0  0  0  0        A=0         0  0  0  0
A=1           0  0  0  1        A=1         0  0  1  0
```
Each map has a single 1 with no neighbor to group with.

**Ans:** `Accept = ABC'`, `Long = ABC`

## 3. Example: Logical Function Unit
Compute AND / OR / XOR of A and B, chosen by control lines C1 C0 (a mini Arithmetic Logic Unit (ALU), like the core of a CPU).

| C1C0 \ AB | 00  | 01  | 11  | 10  | F          |
| --------- | --- | --- | --- | --- | ---------- |
| 00        | 0   | 0   | 1   | 0   | A AND B    |
| 01        | 0   | 1   | 1   | 1   | A OR B     |
| 11        | X   | X   | X   | X   | unused (X) |
| 10        | 0   | 1   | 0   | 1   | A XOR B    |

- Using the don't cares: **`F = C0(A + B) + C1'AB + C1(A ⊕ B)`**
- Alternative structure: compute all three in parallel and pick with a **4:1 mux** driven by C1 C0 (easy in Verilog with `case`).

## 4. S-R Latch (NOR, Active High)
Two cross-coupled NOR gates.

| S   | R   | Q             |                                                                                     |
| --- | --- | ------------- | ----------------------------------------------------------------------------------- |
| 0   | 0   | Q (unchanged) | (0 + Q')' = Q, (Q + 0)' = Q' → **hold** (memory)                                    |
| 0   | 1   | 0             | (1 + Q')' = Q, (Q + 0)' = Q' → **reset**                                            |
| 1   | 0   | 1             | (0 + Q')' = Q, (Q + 1)' = Q' → **set**                                              |
| 1   | 1   | 0 (Q' also 0) | (1 + Q')' = Q, (Q + 1)' = Q' → ⚠️ **forbidden** — unpredictable when both drop to 0 |

## 5. S'-R' Latch (NAND, Active Low)
Two cross-coupled NAND gates; inputs act when **0**.

| S'  | R'  | Q             |                                             |
| --- | --- | ------------- | ------------------------------------------- |
| 0   | 0   | 1 (Q' also 1) | (0 & Q')' = Q, (Q & 0)' = Q' → ⚠️ forbidden |
| 0   | 1   | 1             | (0 & Q')' = Q, (Q & 1)' = Q' → **set**      |
| 1   | 0   | 0             | (1 & Q')' = Q, (Q & 0)' = Q' → **reset**    |
| 1   | 1   | Q (unchanged) | (1 & Q')' = Q, (Q & 1)' = Q' → **hold**     |

## 6. Controlled (Enabled) S-R Latch
- Gate S and R with the clock C: `S' = NAND(S, C)`, `R' = NAND(R, C)` → into a NAND S'-R' latch.

| C   | S   | R   | S'  | R'  | Q            |
| --- | --- | --- | --- | --- | ------------ |
| 0   | X   | X   | 1   | 1   | Q (hold)     |
| 1   | 0   | 0   | 1   | 1   | Q (hold)     |
| 1   | 0   | 1   | 1   | 0   | 0 (reset)    |
| 1   | 1   | 0   | 0   | 1   | 1 (set)      |
| 1   | 1   | 1   | 0   | 0   | ⚠️ forbidden |

## 7. D Latch
- Removes the forbidden case: S = D, R = D' (they can never both be 1).

| C   | D   | S'  | R'  | Q         |
| --- | --- | --- | --- | --------- |
| 0   | X   | 1   | 1   | Q (hold)  |
| 1   | 0   | 1   | 0   | 0 (reset) |
| 1   | 1   | 0   | 1   | 1 (set)   |
- While C = 1, **Q follows D** ("transparent"); when C = 0, Q keeps the last value.

## 8. Latch vs Flip-Flop
![[EEP535_W4_latch_filpflops.png]]

|              | Latch                                                                               | Flip-flop                            |
| ------------ | ----------------------------------------------------------------------------------- | ------------------------------------ |
| Sensitive to | Clock **level** (samples continuously while clock level is high)                    | Clock **edge** (posedge or negedge)  |
| Problem      | Glitches / unsynchronized input changes during the high phase pass straight through | Input only matters right at the edge |
- **Use flip-flops.** In this class, every storage element = positive-edge D flip-flop.
- Waveform intuition: if D changes several times while Clk is high, Q_latch follows every change; Q_ff changes once, at the edge.

> 💡 A flip-flop is built from two latches (master–slave) enabled on opposite clock phases. Real FFs also need D stable a little before (**setup time**) and after (**hold time**) the edge.

## 9. D Flip-Flop in Verilog
```systemverilog
module basic_D_FF (q, d, clk);
	output logic q;
	input  logic d, clk;
	always_ff @(posedge clk) begin
	q <= d;              // ALWAYS use <= for clocked elements
	end
endmodule

// With synchronous reset (reset checked only at the clock edge)
module D_FF (q, d, reset, clk);
	output logic q;
	input  logic d, reset, clk;
	always_ff @(posedge clk) begin
		if (reset) q <= 0;
		else       q <= d;
	end
endmodule
```

**Testbench**
```systemverilog
module stimulus;
  logic clk, reset, d, q;
  parameter ClockDelay = 100;
  D_FF dut (.q, .d, .reset, .clk);
  initial begin                         // clock: period 100
    clk <= 0;
    forever #(ClockDelay/2) clk <= ~clk;
  end
  initial begin                         // one line per clock cycle
    d <= 0; reset <= 1; @(posedge clk);
            reset <= 0; @(posedge clk);
    d <= 1;             @(posedge clk);
    d <= 0;             @(posedge clk);
                        @(posedge clk);
    $stop();
  end
endmodule
```
Rising edges at 50, 150, 250, 350, 450:

| Edge | reset | d | q after edge |
|---|---|---|---|
| 50 | 1 | 0 | 0 (reset) |
| 150 | 0 | 0 | 0 |
| 250 | 0 | 1 | **1** |
| 350 | 0 | 0 | 0 |
| 450 | 0 | 0 | 0 → `$stop` |
- q changes **only at rising edges**, one cycle after d changes.

## 10. FSM Example: Odd Parity Checker
Output 1 if an **odd** number of 1s has been seen **before** the current input.
- States: Even (0), Odd (1). Output = present state.

| Present state | Input | Output | Next state |
|---|---|---|---|
| 0 (Even) | 0 | 0 | 0 |
| 0 (Even) | 1 | 0 | 1 |
| 1 (Odd) | 0 | 1 | 1 |
| 1 (Odd) | 1 | 1 | 0 |
- **NS = PS ⊕ Input; OUT = PS** → one XOR gate + one D flip-flop (with reset).

Trace (start in Even):

| Input | 1 | 0 | 0 | 1 | 1 | 0 | 1 | 0 | 1 | 1 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| PS / Out | 0 | 1 | 1 | 1 | 0 | 1 | 1 | 0 | 0 | 1 | 0 | 1 |

![[EEP535_W4_odd_parity.png|550]]
- Professor's note: label every arrow **input / output**; this is a **Mealy** machine.

## 11. State Diagrams
- **States** = circles; **transitions** = arrows; arrow label = `input / output`.
- This is a **Mealy** machine (outputs on transitions, depend on state + input). **Moore** machines put outputs on states (depend on state only).
- Finite number of states → **FSM**.
```
        Reset
          ↓        1/0
   0/0 ⟲ Even ─────────► Odd ⟲ 0/1
          ◄─────────
               1/1
```

## 12. Vending Machine
- Gum after **≥ 10¢**; one coin slot (N = nickel 5¢, D = dime 10¢); **no change**.
- Inputs N, D (at most one per cycle), Reset. Output Open.
- Moore states: **0¢, 5¢, 10¢** (10¢ = "open"; covers 10¢ and 15¢ since no change).

| Present | N | D | Next | Open |
|---|---|---|---|---|
| 0¢ | 0 | 0 | 0¢ | 0 |
| 0¢ | 1 | 0 | 5¢ | 0 |
| 0¢ | 0 | 1 | 10¢ | 0 |
| 5¢ | 0 | 0 | 5¢ | 0 |
| 5¢ | 1 | 0 | 10¢ | 0 |
| 5¢ | 0 | 1 | 10¢ | 0 |
| 10¢ | X | X | 0¢ | **1** |
```systemverilog
enum {S0, S5, S10} ps, ns;
always_comb begin
case (ps)
S0:  ns = D ? S10 : (N ? S5  : S0);
//S5:  ns = (D | N) ? S10 : S5;
S10: ns = S0;                 // dispense, start over
endcase
end
assign open = (ps == S10);
always_ff @(posedge clk)
if (reset) ps <= S0; else ps <= ns;
```

![[EEP535_W4_vending.png|600]]
- Handwritten implementation (2 DFFs is overkill; the professor used a compact encoding with one bit `PS` plus the inputs):
  - `g` (open) `= D + PS·N` → dime always opens, or a nickel when 5¢ is already in.
  - `NS = PS'·N + PS·D'·N'` → go to "5¢" on the first nickel, stay there while nothing is inserted.

## 13. FSMs in Verilog — Template
FSM `simple`: out = 1 after `w` has been 1 for **two or more** cycles in a row.
```systemverilog
module simple (clk, reset, w, out);
  input  logic clk, reset, w;
  output logic out;
  enum {A, B, C} ps, ns;            // present state, next state

  // Next-state logic (combinational)
  always_comb begin
    case (ps)
      A: if (w) ns = B; else ns = A;
      B: if (w) ns = C; else ns = A;
      C: if (w) ns = C; else ns = A;
    endcase
  end

  // Output logic
  assign out = (ps == C);

  // State register (DFFs)
  always_ff @(posedge clk) begin
    if (reset) ps <= A;
    else       ps <= ns;
  end
endmodule
```
- `enum` lets the tools choose the state encoding.
- Three parts: **next-state logic**, **output logic**, **state flip-flops**.

**Testbench** — one line per clock cycle ("ONLY use this form"):
```systemverilog
initial begin
                        @(posedge clk);
  reset <= 1;           @(posedge clk);
  reset <= 0; w <= 0;   @(posedge clk);
                        @(posedge clk);
                        @(posedge clk);
                        @(posedge clk);
              w <= 1;   @(posedge clk);
              w <= 0;   @(posedge clk);
              w <= 1;   @(posedge clk);
                        @(posedge clk);
                        @(posedge clk);
                        @(posedge clk);
              w <= 0;   @(posedge clk);
                        @(posedge clk);
  $stop;
end
```
Result: ps stays **A** during reset and while w = 0; a single-cycle `w = 1` goes A → B → back to A (out stays 0); then w = 1 for 4 cycles: A → B → **C, C, C** (out = 1 while in C); w = 0 → back to A.

## 14. String Recognizer: "101"
Output 1 when the last three inputs are `1 0 1` (overlaps allowed). Mealy machine:

| State | Meaning | In 0 | In 1 |
|---|---|---|---|
| S0 | nothing useful | S0 / 0 | S1 / 0 |
| S1 | seen "1" | S2 / 0 | S1 / 0 |
| S2 | seen "10" | S0 / 0 | S1 / **1** |

| Input | 1 | 0 | 0 | 1 | 0 | 1 | 0 | 1 | 1 | 0 | 0 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Output | 0 | 0 | 0 | 0 | 0 | **1** | 0 | **1** | 0 | 0 | 0 | 0 | 0 |
- After a match, go to S1 (the final "1" can start the next "101") → overlapping matches work.
- Implementation: 3 states → 2 DFFs; or Verilog with `enum {S0, S1, S2}` and `assign out = (ps == S2) & in;`.

![[EEP535_W4_string101.png|600]]
- Professor's handwritten gate-level solution, encoding **S0 = 00, S1 = 10 (PS1 = 1), S2 = 01 (PS0 = 1)**:

| PS1 PS0 | in = 0 → NS1 NS0 / out | in = 1 → NS1 NS0 / out |
|---|---|---|
| 00 (S0) | 00 / 0 | 10 / 0 |
| 10 (S1) | 01 / 0 | 10 / 0 |
| 01 (S2) | 00 / 0 | 10 / 1 |
- From the K-maps: **NS1 = in**, **NS0 = PS1·in'**, **out = PS0·in** → 2 DFFs + 1 AND + 1 inverter.

## 15. Traffic Light Controller
Highway crossed by a rarely used farm road. Sensor **C** = car waiting on farm road. Interval timer: **TS** (short, yellow) and **TL** (long, green) become true after a **START** and stay true until the next START.

| Input | Meaning | Output | Meaning |
|---|---|---|---|
| C | Car on farm road | HG, HY, HR | Highway green/yellow/red |
| TS | Short interval expired | FG, FY, FR | Farm green/yellow/red |
| TL | Long interval expired | START | Restart the timer |

States and transitions:

| State | Lights | Stay while | Go to (and assert START) |
|---|---|---|---|
| **HG** | HG, FR | `TL' + C'` | **HY** when `TL·C` (highway had its minimum green AND a car waits) |
| **HY** | HY, FR | `TS'` | **FG** when `TS` |
| **FG** | HR, FG | `TL'·C` | **FY** when `TL + C'` (time's up OR no more cars) |
| **FY** | HR, FY | `TS'` | **HG** when `TS` |

![[EEP535_W5_traffic_light.png|650]]
- Professor's handwritten note: solving this state diagram in class gave **+4/+5 points on the midterm** — know it.

## 16. `=` vs `<=`
- `=` **blocking**: assigns immediately, in order (like C).
- `<=` **non-blocking**: first evaluate **all** right-hand sides, then update everything **simultaneously** at the end of the step (like real flip-flops).

```systemverilog
// swap1: blocking                // swap2: non-blocking
val0 = val1;                      val0 <= val1;
val1 = val0;                      val1 <= val0;
out  = val1;                      out  <= val1;
```

| | swap1 (`=`) | swap2 (`<=`) |
|---|---|---|
| val0 | old val1 | old val1 |
| val1 | **old val1** (already overwritten) → no swap ❌ | old val0 → **real swap** ✔ |
| out | old val1 | old val1 |

**Rules in practice**
- `=` in combinational logic (`always_comb`, `assign`)
- `<=` in sequential logic (`always_ff @(posedge clk)`)
- **Never mix** them in one always block
- Each variable is written in **only one** always block
