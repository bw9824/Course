# ICTE 3 — Flip-Flops & FSMs

## Problem 1
**Q:** The following two flip-flops are subtly different, but both useful. The difference in code is shown in bold. What is the difference in their behavior?
```systemverilog
module D_FF1 (q, d, reset, clk); module D_FF2 (q, d, reset, clk);
  output logic q;                  output logic q;
  input  logic d, reset, clk;      input  logic d, reset, clk; 
  always_ff @(posedge clk) begin   always_ff @(posedge clk or posedge reset) begin
    if (reset) q <= 0;               if (reset) q <= 0;
    else       q <= d;               else       q <= d;
  end                              end
endmodule                        endmodule
```
**Ans:**
- **D_FF1 = synchronous reset**: q clears to 0 only at the **next rising clock edge** while reset is high.
- **D_FF2 = asynchronous reset**: q clears to 0 **immediately** when reset rises, no clock needed.

## Problem 2
**Q:** Given the light display shown, build the FSM for a "move left" arrow traffic sign. It should animate an arrow moving left. Hint: Can any of the bulbs be connected to the same signal?
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_ICTE3_P2_lights_q.png|350]]
**Ans:** 3 states, no input, cycle S0 → S1 → S2 → S0 (arrow moves right → middle → left).
- S0: right arrow `{C, H, N, U, Y}`
- S1: middle arrow `{B, F, L, S, X}`
- S2: left arrow `{A, D, J, Q, W}`
- Each arrow's 5 bulbs share one signal → only **3 output signals**.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_ICTE3_P2_arrow.png|650]]

## Problem 3
**Q:** What is the series of inputs that will produce the most TRUEs on the output?
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_ICTE3_P3_fsm_q.png|400]]
**Ans:** Best loop: `B →(0/1) D →(1/0) C →(1/1) B` = **2 TRUEs per 3 inputs**
- From reset (A): input `0` to reach B, then repeat `0 1 1` → `0 011 011 011 …`
- Other loops are worse (e.g. C→A→B→D→C gives 2 per 4).

## Problem 4
**Q:** An ambulance company wants a flashing yellow light that, when a button is held, will instead hold a solid red. Design this machine.
**Ans:** Input: Button. Outputs: (Yellow, Red). Extra state **RED** holds solid red. Mealy: the lights on an arrow are the lights of the **next** state.

| State | Button = 0 | Button = 1 |
| ----- | ---------- | ---------- |
| OFF   | → ON / 10  | → RED / 01 |
| ON    | → OFF / 00 | → RED / 01 |
| RED   | → OFF / 00 | → RED / 01 |
- No button: OFF ↔ ON every clock → yellow flashes. Button held: stay in RED (solid red). Release → OFF, flashing restarts.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_ICTE3_P4_ambulance.png|500]]

## Problem 5
**Q:** What does this FSM do?
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_ICTE3_P5_fsm_q.png|300]]
**Ans:** Output alternates **0, 1, 0, 1, …** regardless of input → **divide-by-2 toggle** (zero/one only remember the input, output ignores it).

## Problem 6
**Q:** Draw the state diagram of a machine that continuously outputs a true once at least two 0's and at least two 1's (in any order, not necessarily consecutively) have been seen, not including current input.
**Ans:** Count zeros N0 and ones N1, each **saturating at 2** → 3 × 3 = **9 states**.
- 0 → N0+1 (stay at 2), 1 → N1+1 (stay at 2).
- Output 1 only on edges **leaving (N0=2, N1=2)** ("not including current input"); every other edge /0.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_ICTE3_P6_two0s_two1s.png|550]]
