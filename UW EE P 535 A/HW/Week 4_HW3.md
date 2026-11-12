## Problem 1
**Q:** 2-bit binary numbers represent the values 0..3 (00 = 0, 01 = 1, 10 = 2, 11 = 3). Develop the simplest Sum-of-Products equation possible that, when given two 2-bit binary numbers X[1:0] and Y[1:0], will output whether the value of X is greater than the value of Y.

**Ans:** `F = X1Y1' + X0Y1'Y0' + X1X0Y0'`

| Y1Y0 \ X1X0 | 00 | 01 | 11 | 10 |
| ----------- | -- | -- | -- | -- |
| 00          | 0  | 1  | 1  | 1  |
| 01          | 0  | 0  | 1  | 1  |
| 11          | 0  | 0  | 0  | 0  |
| 10          | 0  | 0  | 1  | 0  |
- Circle 1, X1X0Y1Y0 = 0100,1100 => X0Y1’Y0’
- Circle 2, X1X0Y1Y0 = 1100,1000,1101,1001 => X1Y1’
- Circle 3, X1X0Y1Y0 = 1100,1110 => X1X0Y0’

## Problem 2
**Q:** Assume that we have already implemented the solution to problem #1 as a module "isGreater(F, X, Y)" in Verilog, and cannot change that module at all. Using that module, write the simplest possible Verilog module that will compute whether A[1:0] is greater than or equal to B[1:0].

**Ans:** `A ≥ B = ~(B > A)` → swap the inputs and invert.
```verilog
module GreaterEqual (
	output F, 
	input [1:0] A, B
);
	wire BA; 
	isGreater ig (.F(BA), .X(B), .Y(A)); // BA = (B > A) 
	assign F = ~BA;
endmodule
```

## Problem 3
**Q:** The OnesCount of a multi-bit signal is the total number of true signals there are in that multi-bit signal, where the output count is given in the 2-bit binary encoding listed above. Thus, the OnesCount of "000" is "00" (representing the value zero), and the OnesCount of "101" is "10" (representing the value two). Design the simplest Sum-of-Products equations possible that will return the OnesCount of a 3-bit input Z[2:0].

**Ans:**
- `F1 = Z2Z1 + Z2Z0 + Z1Z0` (at least two 1s = majority)
- `F0 = Z2Z1'Z0' + Z2'Z1Z0' + Z2'Z1'Z0 + Z2Z1Z0` = `Z2 ⊕ Z1 ⊕ Z0` (odd number of 1s; checkerboard K-map → no SOP grouping)

| Z2Z1Z0 | 000 | 001 | 010 | 011 | 100 | 101 | 110 | 111 |
| ------ | --- | --- | --- | --- | --- | --- | --- | --- |
| F1F0   | 00  | 01  | 01  | 10  | 01  | 10  | 10  | 11  |

## Problem 4
**Q:** We have a circuit that hooks a positive-edge triggered DFF (from class) and a negative-edge triggered DFF to the same D input. The negative-edge triggered DFF responds only to the falling (negative) edge of the clock, the instant when the clock goes from true to false. Fill in the following timing diagram for this case. Assume that all gate delays are very tiny compared to the length of the clock period.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P4_timing_q.png|550]]
Note: All machines must be Mealy machines, like we taught in class. That is, all outputs are given on the transitions (edges between states), not on the states.

**Ans:**
- posedge DFF: copies D **only at rising edges**, holds in between; negedge DFF: copies D **only at falling edges**.
- Glitches of D between edges are ignored.
- Q is **unknown (X)** until the first active edge of that DFF.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P4_ans.png|650]]

## Problem 5
**Q:** Design a string recognizer whose output will be true if the last four input values seen are "1100" or "1010", including the current one. For this circuit (a) draw the state diagram and (b) implement it with D flip-flops.

**Ans:** States remember the useful suffix: start, `1`, `11`, `110`, `10`, `101`. Output 1 on input 0 from `110` or `101`.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P5_state.png|700]]

| Q2Q1Q0 (state) | X     | D2D1D0 (Next) | Z         |
| -------------- | ----- | ------------- | --------- |
| 000 (start)    | 0 / 1 | 000 / 001     | 0 / 0     |
| 001            | 0 / 1 | 010 / 011     | 0 / 0     |
| 011            | 0 / 1 | 110 / 011     | 0 / 0     |
| 110            | 0 / 1 | 000 / 101     | **1** / 0 |
| 010            | 0 / 1 | 000 / 101     | 0 / 0     |
| 101            | 0 / 1 | 010 / 011     | **1** / 0 |
```
Z  = Q2·X'
D0 = X
D1 = Q0
D2 = Q1Q0X' + Q1Q0'X = Q1(Q0 ⊕ X)
```
- Encoding trick: Q0 = last input, Q1 = input before → D0 and D1 become plain wires (shift register).
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P5_circuit.png|650]]

## Problem 6
**Q:** Create an implementation of the following finite state machine. You can use any basic gates, and D flip-flops.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P6_fsm_q.png|450]]
**Ans:**

| S1S0 | In = 0, D1D0/Out | In = 1, D1D0/Out |
| ---- | ---------------- | ---------------- |
| 00   | 00 / 0           | 01 / 0           |
| 01   | 00 / 0           | 11 / 0           |
| 11   | 01 / 1           | 11 / 1           |
```
D1  = S0·In
D0  = S1 + In
Out = S1          (10 is unused → don't care)
```
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P6_circuit.png|400]]

## Problem 7
**Q:** We wish to design a soda vending machine which charges $.30 cents per soda (very economical). There is one coin slot, which will accept only quarters, nickels, and dimes. The controller has two outputs, Give_Soda and Return_Last_Coin. Draw the corresponding state diagram assuming the vending machine only accepts the correct change. You do NOT need to provide the state table or implementation. Notes: If the user provides more than $.30, you should return the last coin put in. If you get the correct amount, you should dispense a soda and be ready for the next user.
**Ans:** States = money so far: 0¢, 5¢, 10¢, 15¢, 20¢, 25¢. Coin N = nickel, D = dime, Q = quarter. Label = coin / Give_Soda, Return_Last_Coin.
- Total = 30¢ → `/10`, go back to 0¢.
- Total > 30¢ → `/01`, stay in the same state (coin returned).
- Otherwise → `/00`, add the coin.
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P7_soda.png|700]]

## Problem 8
**Q:** In many games, players have to win by two points. Design an FSM to enforce this rule. Specifically, you have an input In. Make the output TrueWins true only when you get a total of two more TRUEs than FALSEs, and start a new game. Make the output FalseWins true only when you get a total of two more FALSEs than TRUEs, and start a new game. Draw the state diagram for this design.
**Ans:** Only the **difference** matters → 3 states: `Tied`, `True+1`, `False+1`. Label = In / TrueWins, FalseWins.
- Tied: 1 → True+1, 0 → False+1 (`/00`)
- True+1: 1 → Tied `/10` (True wins, new game); 0 → Tied `/00`
- False+1: 0 → Tied `/01` (False wins, new game); 1 → Tied `/00`
![[personal_note/美國留學 & 生活/UW/課程/EE P 535 A/Picture/EEP535_HW3_P8_win_by_two.png|400]]
