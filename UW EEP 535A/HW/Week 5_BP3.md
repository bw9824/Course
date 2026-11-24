
> **Context.** Weeks 4–5 covered FSM design: draw the state diagram → state table → state encoding → next-state / output equations. In a **Mealy** machine the output depends on the state **and** the current input, so it is written on the arrow as `input / output`. In a **Moore** machine the output depends only on the state and is written inside the circle.

## Question

> Design an **FSM** system that takes in a string of inputs and **removes the second 1 from every consecutive string of 1's**.

"Removes" means the output is 0 at the position of the second 1 in each run. The output stream has the same length as the input stream.

Example: In = 0 1 1 1 0 1 1 0 → Out = 0 1 0 1 0 1 0 0
(Run "111": the second 1 becomes 0. Run "11": the second 1 becomes 0.)

## Answer

**Idea:** the machine only needs to remember how many 1s are in the current run: none (last input was 0), exactly one, or two or more.

| State | Meaning | In = 0 | In = 1 |
|---|---|---|---|
| S0 (00) | Last input was 0 | S0 / 0 | S1 / **1** (first 1 → pass) |
| S1 (10) | Exactly one 1 so far | S0 / 0 | S2 / **0** (second 1 → drop) |
| S2 (11) | Two or more 1s | S0 / 0 | S2 / **1** (third+ 1 → pass) |

Notation: `next state / output`.

![[EEP535_BP3_fsm.png|650]]

(Red arrow = the only transition where an input 1 gives output 0.)

**Why Mealy:** the output must react to the *current* input in the same clock. A Moore version would need extra states (one per output value) and would delay the output by one clock.

**Trace:** In = 0 1 1 1 0 1 1 0
States: S0 → S0 → S1 → S2 → S2 → S0 → S1 → S2 → S0
Out = 0 1 0 1 0 1 0 0 ✓
