## Problem 1
**Q:** What does this circuit do?
![[EEP535_ICTE2_P1_circuit_q.png|450]]

**Ans:** **XOR** — `A = NAND(X,Y)`, `B = NOR(X,Y)`, `F = A ⊕ B = X ⊕ Y`

| X Y | A   | B   | F   |
| --- | --- | --- | --- |
| 0 0 | 1   | 1   | 0   |
| 0 1 | 1   | 0   | 1   |
| 1 0 | 1   | 0   | 1   |
| 1 1 | 0   | 0   | 0   |

## Problem 2
**Q:** Convert the truth table (ABCD → F = g … v, in binary order 0000 → 1111) to a K-map. 
- Letters are used as variables to hold the function value
- First value (g) already filled in.
**Ans:**

| CD \ AB | 00 | 01 | 11 | 10 |
| ------- | -- | -- | -- | -- |
| 00      | g  | k  | s  | o  |
| 01      | h  | l  | t  | p  |
| 11      | j  | n  | v  | r  |
| 10      | i  | m  | u  | q  |
Columns AB and rows CD both use **Gray order 00, 01, 11, 10**.

## Problem 3
**Q:** Convert the truth table to a K-map, then solve.

| ABC | 000 | 001 | 010 | 011 | 100 | 101 | 110 | 111 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| F   | 1   | 1   | 0   | 0   | 1   | 1   | 1   | 0   |

**Ans:** `F = B' + AC'`

| C \ AB | 00 | 01 | 11 | 10 |
| ------ | -- | -- | -- | -- |
| 0      | 1  | 0  | 1  | 1  |
| 1      | 1  | 0  | 0  | 1  |
- Circle 0, ABC = 000, 001, 100, 101 => B’
- Circle 1, ABC = 110, 100 => AC’

## Problem 4
**Q:** Solve this K-map.

| CD \ AB | 00 | 01 | 11 | 10 |
| ------- | -- | -- | -- | -- |
| 00      | 0  | 0  | 0  | 0  |
| 01      | 0  | 1  | 1  | 0  |
| 11      | 1  | 1  | 1  | 1  |
| 10      | 1  | 0  | 1  | 1  |

**Ans:** `F = AC + BD + B'C`
- Circle 0, ABCD = 1011,1010,0011,0010 => B’C
- Circle 1, ABCD = 0101,1101,0111,1111 => BD
- Circle 2, ABCD = 1111,1011,1110,1010 => AC

## Problem 5
**Q:** Solve the K-map (X = don't care).

| CD \ AB | 00 | 01 | 11 | 10 |
| ------- | -- | -- | -- | -- |
| 00      | X  | 0  | 0  | 1  |
| 01      | 0  | X  | 0  | 0  |
| 11      | 0  | 0  | 0  | 0  |
| 10      | 1  | X  | 1  | X  |

**Ans:** `F = B'D' + CD' = D'(B' + C)`
- Circle 0, ABCD = 0000,1000,0010,1010 => B’D’
- Circle 1, ABCD = 0010,0110,1110,1010 => CD’
X's are free: treat as 1 only if they make a group bigger.

## Problem 6
**Q:** Solve the K-map.

| CD \ AB | 00 | 01 | 11 | 10 |
| ------- | -- | -- | -- | -- |
| 00      | 1  | 0  | 0  | X  |
| 01      | 1  | X  | X  | 1  |
| 11      | X  | 1  | 0  | 0  |
| 10      | 1  | 1  | 0  | X  |

**Ans:** `F = A'C + B'C'`
- Circle 0, ABCD = 0000, 1000, 0001, 1001 => B’C’
- Circle 1, ABCD = 0011, 0111, 0010, 0110 => A’C

## Problem 7
**Q:** Amy, Bill, Carol, and Dennis each decide independently whether they want to play Chess or Checkers, each a 2-player game. Develop a circuit that can tell if the 4 people can be organized into two simultaneous games, respecting each person's choice.

**Ans:** `F = (A ⊕ B ⊕ C ⊕ D)'` (Chess = 0, Checkers = 1)
- Possible ⇔ an **even** number choose Checkers (0, 2 or 4) → even parity = **XNOR** of all four.
- K-map is a checkerboard → no grouping possible → use XOR.
