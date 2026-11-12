## Problem 1
**Q:** Describe when the dome/interior light of the car should be on.
- DriverDoorOpen = true if lefthand door open
- PassDoorOpen = true if righthand door open
- LightSwitch = true if light should be on regardless of door state, false if light should be on when a door is open

**Ans:** `Light = DriverDoorOpen + PassDoorOpen + LightSwitch` → OFF only when both doors are closed and the switch is off.

| DriverDoorOpen | PassDoorOpen | LightSwitch | Light |
| -------------- | ------------ | ----------- | ----- |
| 0              | 0            | 0           | **0** |
| 1              | 0            | 0           | 1     |
| 0              | 1            | 0           | 1     |
| 1              | 1            | 0           | 1     |
| 0              | 0            | 1           | 1     |
| 1              | 0            | 1           | 1     |
| 0              | 1            | 1           | 1     |
| 1              | 1            | 1           | 1     |

## Problem 2
**Q:** What does the following circuit do?
![[Course/UW EE P 535 A/HW/Picture/EEP535_ICTE1_P2_circuit_q.png|450]]

**Ans:** `F = A'B + AB' = A ⊕ B` → This circuit implements an XOR function. The output F is 1 when A and B are different, and 0 when they are the same.

## Problem 3
**Q:** To buy beer, someone must have ID. Which of these sentences tells us exactly when Ann and Bob can get beer?
1. It is not true that both Ann and Bob don't have ID.
2. Ann has ID or Bob has ID or Both have ID.
3. Ann has ID or Bob has ID
4. Ann has ID and Bob does not, or Bob has ID and Ann does not.
5. Ann has ID, or Bob has ID and Ann does not.

**Ans:** Target = `A + B` → **1, 2, 3, 5**
1. `(A'B')' = A + B` ✔ DeMorgan
2. `A + B + AB = A + B` ✔ absorption
3. `A + B` ✔
4. `AB' + A'B = A ⊕ B` ✘ fails when **both** have ID
5. `A + A'B = A + B` ✔

## Problem 4
**Q:** Create the truth table for an equality circuit, which is true whenever the two inputs are the same.

**Ans:** `Out = AB + A'B' = (A ⊕ B)'` → **XNOR**

| A | B | Out |
| - | - | --- |
| 0 | 0 | 1 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

## Problem 5
**Q:** Write the Boolean equation for the 2nd seat belt light: Seat Belt Light (driver belt in, passenger belt in, passenger present).

**Ans:** `SeatBeltLight = PassengerPresent · PassengerBelt'`
- Driver belt does not matter for the passenger light; light only if someone sits there **and** the belt is not in.

## Problem 6
**Q:** Does the following Boolean equation implement the function given in the truth table?
`MyCout' = (A·B) + (A·Cin) + (A·B·Cin)`

| A B Cin | 000 | 001 | 010 | 011 | 100 | 101 | 110 | 111 |
| ------- | --- | --- | --- | --- | --- | --- | --- | --- |
| Cout    | 0   | 0   | 0   | 1   | 0   | 1   | 1   | 1   |

**Ans:** **No.**
- When A = 0, B = 0, and Cin = 0, the equation outputs 0, but the truth table Cout = 0 (Cout' = 1, should be Cout = 1).

## Problem 7
**Q:** Simplify the following Boolean equation: `AB + AC + A'B`

**Ans:** `B + AC`
```
AB + A'B + AC = (A + A')B + AC = B + AC
```

## Problem 8
**Q:** Simplify the following Boolean equation, starting with DeMorgan's Law: `F' = AB' + AC`, `F = ?`

**Ans:** `F = A' + BC'`
```
F = (AB' + AC)'
  = (AB')' · (AC)'            ← DeMorgan
  = (A' + B)(A' + C')
  = A' + A'C' + A'B + BC'
  = A' + BC'                  ← A' absorbs A'C' and A'B
```
