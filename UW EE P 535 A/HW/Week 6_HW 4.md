### Problem 1
**Q:** Perform the following computations on the given unsigned binary numbers.
1. `101100 + 111010`
2. `1100101 – 0101110`
3. `10011 * 10110` (Note: give the full 9-bit result)

**Ans:**
```
i.    101100 (44)       ii.   1100101 (101)      iii.        10011 (19)
    + 111010 (58)           − 0101110  (46)            ×     10110 (22)
    --------                ---------                  ------------
     1100110 (102)            0110111  (55)                 10011
                                                           10011
                                                         10011 
                                                       ------------
                                                         110100010 (418)
```

### Problem 2
**Q:** Perform the specified operation assuming the signed numbers are (a) sign magnitude and (b) twos complement. For the case of subtraction, perform the addition of the negation.
1. `(10111) – (11100)`
2. `(10101) + (01011)`
3. `(011011) + (000111)`
4. `(001110) – (011011)`

**Ans:**

| #   | (a) Sign-magnitude               | (b) Two's complement                                              |
| --- | -------------------------------- | ----------------------------------------------------------------- |
| i   | −7 − (−12) = −7 + 12 → 00101     | −9 − (−4) = 10111 + 00100 = **11011 (−5)**                        |
| ii  | −5 + 11 → 00110                  | −11 + 11 = 10101 + 01011 = 1 00000 → **00000 (0)**, carry dropped |
| iii | 27 + 7 = 34 > 31 → **overflow**  | 011011 + 000111 = 100010 → **overflow**                           |
| iv  | 14 − (+27) = 14 + (−27) → 101101 | 001110 + 100101 = **110011 (−13)**                                |
- Sign-magnitude negation: flip the sign bit only. Mixed signs → subtract the smaller magnitude from the larger, take the larger one's sign.
- Two's complement negation: invert all bits, +1 (e.g. `11100 (−16 + 8 + 4 = -4) → 00011 + 1 = 00100`).
- Overflow (2C): both operands have the same sign and the result's sign is different.

### Problem 3
**Q:** Design a circuit that can tell whether a 4-bit 2's complement number (A3A2A1A0) is greater than 0.
**Ans:** `F = A3'·(A2 + A1 + A0)`
- Positive = sign bit 0 **and** not zero (0000 is not > 0).
- 2-gate circuit: `F = NOR(A3, NOR(A2, A1, A0))`
![[EEP535_HW4_P3_circuit.png|450]]

### Problem 4
**Q:** Design a circuit that can tell whether a 4-bit sign-magnitude number (A3A2A1A0) is greater than or equal to 0. Note: if you think this is just a wire connected to A3 you're missing something!
**Ans:** `F = A3' + A2'A1'A0'`
- Sign-magnitude has **two zeros**: `0000` (+0) and `1000` (−0). −0 has A3 = 1 but is still ≥ 0, so `A3'` alone misses it.
- 2-gate circuit: `F = NAND(A3, OR(A2, A1, A0))` (DeMorgan: `(A3·(A2+A1+A0))' = A3' + A2'A1'A0'`)
![[EEP535_HW4_P4_circuit.png|450]]
