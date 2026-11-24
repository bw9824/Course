# SystemVerilog Line-by-Line Walkthrough

### SystemVerilog main code

| Syntax                          | Meaning                                                                                                            |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `` `timescale 1 ps / 1 ps ``    | Simulation time unit 1 ps, precision 1 ps. Affects simulation only, not synthesis                                  |
| `logic`                         | SystemVerilog's general signal type; replaces Verilog's wire/reg                                                   |
| `assign x = …;`                 | Continuous assignment: combinational logic, the left side follows the right side immediately                       |
| `always_comb`                   | Combinational block, uses `=` (blocking). Every output must get a value on every path, or synthesis infers a latch |
| `always_ff @(posedge clk)`      | Rising-edge registers, uses `<=` (non-blocking). All registers update together on the same clock edge              |
| `#(parameter X = …)`            | Module parameter; override it at instantiation with `#(.X(…))`, e.g. the testbench makes SCRUB\_DIV small          |
| `.clk`                          | Implicit named port connection, same as `.clk(clk)`. Only works when the signal and port have the same name        |
| `^{a, b, c}`                    | Reduction XOR: XORs all bits together; the 1-bit result is the parity                                              |
| `{a, b}`                        | Concatenation; the leftmost part is the most significant                                                           |
| `// synthesis translate_off/on` | Code between them (the testbench) is skipped by Quartus synthesis but still compiled by ModelSim                   |

### Testbenches

| Syntax                       | Meaning                                                                                                                                                            |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `module xxx_testbench();`    | A testbench is a module too, but it has no ports because it is the outermost level                                                                                 |
| `dut`                        | Device Under Test: the instance name of the module being tested                                                                                                    |
| `initial begin … end`        | Starts at simulation time 0 and runs once. A testbench can have several `initial` blocks; they all start at the same time and run in parallel                      |
| `forever`                    | Repeat forever; used to generate the clock                                                                                                                         |
| `#50`                        | Wait 50 time units (ps here)                                                                                                                                       |
| `@(posedge clk)`             | Stop and wait for the next rising clock edge                                                                                                                       |
| `repeat (n)`                 | Repeat n times, typically `repeat (4) @(posedge clk);` to wait 4 edges                                                                                             |
| `#1`                         | Wait a tiny bit after an edge so the `<=` updates have finished before checking results                                                                            |
| `integer`                    | 32-bit signed integer, for loop variables and error counts                                                                                                         |
| `task … endtask`             | Packages a repeated sequence of actions, like a function, except it can wait for clock edges                                                                       |
| `!==` / `===`                | 4-state comparison that also compares X and Z. With `!=`, an X makes the result X, which `if` treats as false and can hide an error; `!==` treats X as "not equal" |
| `$display("… %h %b %0d", …)` | Prints to the ModelSim Transcript. `%h` hex, `%b` binary, `%0d` decimal without padding                                                                            |
| `$urandom`                   | A random 32-bit unsigned number                                                                                                                                    |
| `$stop`                      | Pause the simulation so you can inspect the waveform                                                                                                               |
| `dut.ctrl.pending`           | Hierarchical name: reads a signal inside the `ctrl` instance inside `dut`. Simulation only                                                                         |
## 1. synchronizer.sv

Passes the asynchronous KEY/SW inputs through two flip-flops before the rest of the circuit uses them, so metastability does not propagate.

```systemverilog
  1 `timescale 1 ps / 1 ps
  2
  3 module synchronizer #(parameter WIDTH = 1) (clk, in, out);
  4     input  logic             clk;
  5     input  logic [WIDTH-1:0] in;
  6     output logic [WIDTH-1:0] out;
  7
  8     logic [WIDTH-1:0] q1;
  9
 10     always_ff @(posedge clk) begin
 11         q1  <= in;
 12         out <= q1;
 13     end
 14 endmodule
```

| Line  | How it is written / logic                                                                                                                                                                                                                                                        |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 3     | Module declaration with parameter `WIDTH`, default 1. DE1_SoC uses `#(.WIDTH(4))` for the 4 KEYs and `#(.WIDTH(10))` for the 10 SWs, so one module handles any width                                                                                                             |
| 4–6   | Ports: clock, asynchronous input `in`, synchronized output `out`. `[WIDTH-1:0]` before the name makes each one a WIDTH-bit vector (a bus), not an array                                                                                                                          |
| 8     | `q1` is the output of the first flip-flop stage                                                                                                                                                                                                                                  |
| 10–13 | On every rising edge, `in` goes into stage 1 (`q1`) and `q1` goes into stage 2 (`out`). With `<=`, all right-hand sides are sampled first and then all left-hand sides update together, so `out` gets the **old** `q1`. A change on `in` therefore takes 2 clocks to reach `out` |

**Why two stages**: the buttons and switches have no relation to the 50 MHz clock, so sooner or later one changes inside the setup/hold window and stage 1 can go metastable. Stage 2 gives it a full clock (20 ns) to settle, so stage 2 almost always captures a clean 0 or 1. It also guarantees that every circuit using the signal sees the same value in the same clock. The cost is 40 ns of delay, far below anything a person can notice.

### Testbench (lines 16–44)

```systemverilog
 16 module synchronizer_testbench();
 17     logic clk, in, out;
 18
 19     parameter CLOCK_PERIOD = 100;
 20
 21     synchronizer dut (.clk(clk), .in(in), .out(out));
 22
 23     initial begin
 24         clk <= 0;
 25         forever #(CLOCK_PERIOD / 2) clk <= ~clk;
 26     end
 27
 28     initial begin
 29         in <= 0;
 30         repeat (3) @(posedge clk);
 31
 32         in <= 1;
 33         repeat (4) @(posedge clk);
 34         in <= 0;
 35         repeat (4) @(posedge clk);
 36
 37         in <= 1;
 38         @(posedge clk);
 39         in <= 0;
 40         repeat (4) @(posedge clk);
 41
 42         $stop;
 43     end
 44 endmodule
```

| Line  | How it is written / logic                                                                                    |
| ----- | ------------------------------------------------------------------------------------------------------------ |
| 16    | Testbench module, no ports                                                                                   |
| 17    | The three wires to the synchronizer. `WIDTH` is left at its default of 1, so they are 1 bit each             |
| 19    | Clock period: 100 ps                                                                                         |
| 21    | One synchronizer instance, named `dut`                                                                       |
| 23–26 | Clock generator: start at 0, invert every 50 ps, forever                                                     |
| 29–30 | `in` = 0 for 3 edges, so `q1` and `out` are flushed from their initial X to 0                                |
| 32–35 | **Long pulse**: `in` = 1 for 4 clocks, then back to 0 for 4 clocks                                           |
| 37–40 | **Short pulse**: `in` = 1 for only 1 clock, to check that the shortest possible signal passes through intact |
| 42    | Pause the simulation and look at the waveform                                                                |
## 2. userInput.sv

A 2-state FSM that outputs 1 only in the first clock of a button press, no matter how long the button is held (rising-edge detection). Without it, holding KEY0 for half a second is 25 million clocks, which would repeat a write or a bit flip millions of times.

```systemverilog
  3 module userInput (clk, reset, in, out);
  4     input  logic clk, reset;
  5     input  logic in;
  6     output logic out;
  7
  8     enum logic { RELEASED, HELD } ps, ns;
  9
 10     always_comb begin
 11         case (ps)
 12             RELEASED: if (in) ns = HELD;
 13                       else    ns = RELEASED;
 14             HELD:     if (in) ns = HELD;
 15                       else    ns = RELEASED;
 16             default:          ns = RELEASED;
 17         endcase
 18     end
 19
 20     // Pulse on the first cycle of a press.
 21     assign out = (ps == RELEASED) & in;
 22
 23     always_ff @(posedge clk) begin
 24         if (reset)
 25             ps <= RELEASED;
 26         else
 27             ps <= ns;
 28     end
 29 endmodule
```

| Line  | How it is written / logic                                                                                                                                                                       |
| ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 3–6   | Ports: clock, reset, synchronized button `in` (1 = pressed), output pulse `out`                                                                                                                 |
| 8     | `enum` declares the two states RELEASED and HELD; `logic` means 1 bit is enough (RELEASED = 0，HELD = 1). `ps` = present state, `ns` = next state: the standard Lab 3 style                      |
| 10–18 | Next-state logic. From either state: pressed → HELD, released → RELEASED. `default` makes sure `ns` gets a value on every path, so no latch is inferred                                         |
| 21    | Output logic (Mealy): 1 only in the clock where the state is still RELEASED but `in` is already 1. Next clock the state is HELD and `out` drops back to 0, so the pulse is exactly 1 clock wide |
| 23–28 | State register: RELEASED on reset, otherwise `ps <= ns` on every rising edge                                                                                                                    |

**Use in this project**: DE1_SoC uses three of them: KEY0 → `go` (do one operation), KEY1 → `addr_ld` (select a word), KEY2 → `scrub_toggle` (turn the scrubber on/off). KEY3 is reset, which should stay active while held, so it does not go through userInput.

### Testbench (lines 31–68)

```systemverilog
 31 module userInput_testbench();
 32     logic clk, reset, in, out;
 33
 34     parameter CLOCK_PERIOD = 100;
 35
 36     userInput dut (.clk(clk), .reset(reset), .in(in), .out(out));
 37
 38     initial begin
 39         clk <= 0;
 40         forever #(CLOCK_PERIOD / 2) clk <= ~clk;
 41     end
 42
 43     initial begin
 44         reset <= 1;
 45         in    <= 0;
 46         @(posedge clk);
 47
 48         reset <= 0;
 49         repeat (2) @(posedge clk);
 50
 51         in <= 1;
 52         repeat (6) @(posedge clk);
 53         in <= 0;
 54         repeat (2) @(posedge clk);
 55
 56         in <= 1;
 57         @(posedge clk);
 58         in <= 0;
 59         repeat (2) @(posedge clk);
 60
 61         in <= 1;
 62         repeat (3) @(posedge clk);
 63         in <= 0;
 64         repeat (2) @(posedge clk);
 65
 66         $stop;
 67     end
 68 endmodule
```

| Line  | How it is written / logic                                                                                                          |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------- |
| 44–46 | Hold `reset` = 1 for one edge so `ps` starts in RELEASED instead of X; `in` = 0                                                    |
| 48–49 | Release reset and wait 2 clocks with the button up: `out` must stay 0                                                              |
| 51–54 | **Long press**: `in` = 1 for 6 clocks. `out` must be 1 for only the first clock, then 0 for the other 5 while the FSM sits in HELD |
| 56–59 | **Shortest press**: `in` = 1 for 1 clock. Still exactly one 1-clock pulse, so even a very short press is not lost                  |
| 61–64 | **Medium press**: 3 clocks, again exactly one pulse                                                                                |
| 66    | Pause and inspect the waveform                                                                                                     |
## 3. enable\_gen.sv

Produces a 1-clock enable pulse every DIV clocks. The scrubber uses it to set its pace, while the whole design still runs on the single CLOCK_50.

```systemverilog
  6 module enable_gen #(parameter DIV = 2) (clk, reset, en);
  7     input  logic clk, reset;
  8     output logic en;    // 1 for one clock every DIV clocks (always 1 if DIV = 1)
  9
 10     logic [31:0] count;
 11
 12     always_ff @(posedge clk) begin
 13         if (reset || count == DIV - 1) count <= 0;
 14         else                           count <= count + 1;
 15     end
 16
 17     assign en = (count == DIV - 1);
 18 endmodule
```

| Line  | How it is written / logic                                                                                                                                                      |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 6     | Parameter `DIV` = number of clocks per pulse. On the board DE1_SoC passes 12,500,000 (50 MHz ÷ 12.5M = 4 pulses per second); the testbench passes 20 so simulation stays short |
| 10    | 32-bit counter (up to about 4.3 billion), plenty for 12.5M. Quartus removes the unused upper bits; 12.5M needs only 24 bits                                                    |
| 12–15 | Reset to 0 on reset or when the count reaches DIV−1, otherwise +1. The counter cycles 0 … DIV−1, one round = DIV clocks                                                        |
| 17    | `en` = 1 in the clock where count equals DIV−1. That value occurs once per round, so the pulse is exactly 1 clock wide                                                         |

**Why not a divided slow clock** : using a counter bit as the clock for other logic would create a second clock domain. That breaks the Lab 4 single-50 MHz-clock rule and brings clock skew and clock-domain-crossing problems. With a clock enable, every register still runs on CLOCK_50; `en` only decides whether it does something in that clock.

### Testbench (lines 20–44)

```systemverilog
 20 // Testbench: with DIV = 5 there must be exactly 4 pulses in 20 clocks.
 21 // synthesis translate_off
 22 module enable_gen_testbench();
 23     logic clk, reset, en;
 24     integer pulses;
 25
 26     enable_gen #(.DIV(5)) dut (.clk(clk), .reset(reset), .en(en));
 27
 28     parameter CLOCK_PERIOD = 100;
 29     initial begin
 30         clk <= 0;
 31         forever #(CLOCK_PERIOD/2) clk <= ~clk;
 32     end
 33
 34     always @(posedge clk) if (reset) pulses <= 0; else if (en) pulses <= pulses + 1;
 35
 36     initial begin
 37         reset <= 1; @(posedge clk);
 38         reset <= 0; repeat (20) @(posedge clk); #1;
 39         if (pulses == 4) $display("enable_gen_testbench: PASS");
 40         else             $display("enable_gen_testbench: FAIL pulses=%0d", pulses);
 41         $stop;
 42     end
 43 endmodule
 44 // synthesis translate_on
```

| Line | How it is written / logic |
| --- | --- |
| 20 | Comment stating the pass condition |
| 21, 44 | translate\_off/on: Quartus skips the testbench |
| 23–24 | Wires to the dut, plus an `integer` that counts pulses |
| 26 | Instance with `#(.DIV(5))`, overriding the default DIV = 2. A small DIV keeps the simulation short |
| 28–32 | 100 ps clock |
| 34 | **The checker**: a small counter written in the testbench itself. On every edge where `en` = 1 it adds 1 to `pulses`. It uses plain `always`, which is allowed in a testbench; the design files use `always_ff` |
| 37 | Reset for one edge: `count` and `pulses` both start at 0 |
| 38 | Run 20 clocks, then `#1` so the last `pulses <= pulses + 1` has finished before it is read |
| 39–40 | 20 clocks ÷ 5 = **4 pulses** expected. Print PASS, or FAIL with the actual count |
| 41 | Pause |
## 4. seg7.sv

Converts a 4-bit value (0–F) to the 7 segments of a seven-segment display.

```systemverilog
  3 module seg7 (hex, leds);
  4     input  logic [3:0] hex;
  5     output logic [6:0] leds;
  6
  7     // Active-low segments in gfedcba order.
  8     always_comb begin
  9         case (hex)
 10             4'h0: leds = 7'b1000000;
 11             4'h1: leds = 7'b1111001;
                …(2–E written the same way)
 25             4'hF: leds = 7'b0001110;
 26             default: leds = 7'bX;
 27         endcase
 28     end
 29 endmodule
```

| Line | How it is written / logic                                                                                                                                                                    |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 3–5  | Input: a 4-bit digit. Output: 7 segments                                                                                                                                                     |
| 7    | The comment states two facts: segments are **active-low** (0 = on), and the bit order from high to low is g f e d c b a                                                                      |
| 8–28 | A lookup `case`. For example 0 = `1000000`: only g (the top bit, middle bar) is 1 = off, the other 6 are on, forming a "0". 1 = `1111001`: only b and c are on, the right-hand vertical line |
| 26   | `default` gives X. All 16 values of a 4-bit input are listed, so it is never reached; it only makes the `always_comb` complete                                                               |

**Use in this project**: DE1_SoC places six of them with `generate`, one per HEX0–HEX5. Turning a display off is not done here; DE1_SoC replaces the output with `7'b1111111` (all segments off). B and D are drawn as lowercase b and d so they do not look like 8 and 0.

### Testbench (lines 31–44)

```systemverilog
 31 module seg7_testbench();
 32     logic [3:0] hex;
 33     logic [6:0] leds;
 34     integer i;
 35
 36     seg7 dut (.hex, .leds);
 37
 38     initial begin
 39         for (i = 0; i < 16; i++) begin
 40             hex = i;
 41             #10;
 42         end
 43     end
 44 endmodule
```

| Line  | How it is written / logic                                                                                                                                                                  |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 31–34 | Testbench module: 4-bit input, 7-bit output, and a loop variable                                                                                                                           |
| 36    | The seg7 instance, with implicit connections because the names match                                                                                                                       |
| 38–43 | Try every value 0–F. seg7 is purely combinational, so there is **no clock**: set `hex`, wait `#10` for the output to settle, move on. Blocking `=` is fine here because nothing is clocked |
## 5. ecc\_encoder.sv

Pure combinational logic: adds 5 check bits to 8 data bits to form a 13-bit codeword. The code is SECDED (Single Error Correct, Double Error Detect), a shortened extended Hamming code (13,8).

```systemverilog
  6 //   Codeword bit k = Hamming position k (k = 1..12); bit 0 = overall parity.
  7 //     position : 12 11 10  9  8  7  6  5  4  3  2  1  0
  8 //     contents : d7 d6 d5 d4 p8 d3 d2 d1 p4 d0 p2 p1 p0
  …
 12 module ecc_encoder (data, cw);
 13     input  logic [7:0]  data;
 14     output logic [12:0] cw;
 15
 16     logic p1, p2, p4, p8, p0;
 17
 18     assign p1 = data[0] ^ data[1] ^ data[3] ^ data[4] ^ data[6];   // positions 3 5 7 9 11
 19     assign p2 = data[0] ^ data[2] ^ data[3] ^ data[5] ^ data[6];   // positions 3 6 7 10 11
 20     assign p4 = data[1] ^ data[2] ^ data[3] ^ data[7];             // positions 5 6 7 12
 21     assign p8 = data[4] ^ data[5] ^ data[6] ^ data[7];             // positions 9 10 11 12
 22     assign p0 = ^{data, p1, p2, p4, p8};                           // overall parity
 23
 24     assign cw = {data[7:4], p8, data[3:1], p4, data[0], p2, p1, p0};
 25 endmodule
```

| Line  | How it is written / logic                                                                                                                                                                                        |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 6–8   | **The bit-layout table**, the basis of the whole ECC. Positions 1, 2, 4, 8 (powers of 2) hold check bits p1, p2, p4, p8; positions 3, 5, 6, 7, 9–12 hold d0–d7 in order; position 0 holds the overall parity p0  |
| 12–14 | Input: 8-bit `data`. Output: 13-bit `cw`                                                                                                                                                                         |
| 16    | Intermediate signals for the 5 check bits                                                                                                                                                                        |
| 18    | **p1 covers every position whose number has bit 0 set (binary xxx1)**: position 3 (data: d0), 5 (data: d1), 7(data: d3), 9(data: d4), 11(data: d6). XOR them so that p1 plus those bits has an even number of 1s |
| 19    | p2 covers positions with bit 1 set (xx1x) : 3(d0), 6(d2), 7(d3), 10(d5), 11(d6)                                                                                                                                  |
| 20    | p4 covers positions with bit 2 set (x1xx) : 5(d1), 6(d2), 7(d3), 12(d7)                                                                                                                                          |
| 21    | p8 covers positions with bit 3 set (1xxx) : 9–12 (d4–d7)                                                                                                                                                         |
| 22    | p0 reduction-XORs the 8 data bits and 4 check bits, so the whole 13-bit word has an even number of 1s                                                                                                            |
| 24    | Concatenation places each bit according to the table on line 8, bit 12 on the left. `data[3:1]` puts d3, d2, d1 into positions 7–5 in one go                                                                     |

**Why this layout works**: if position k is wrong, only the check bits that cover k fail, and those are exactly the bits that are 1 in the binary number k. So the decoder can line up the 4 check results and read k directly: the position of the error.

**Worked example, A5**: A5 = 1010 0101, so d0=1, d1=0, d2=1, d3=0, d4=0, d5=1, d6=0, d7=1.

- p1 = 1^0^0^0^0 = 1; p2 = 1^1^0^1^0 = 1; p4 = 0^1^0^1 = 0; p8 = 0^1^0^1 = 0
- Bits 12–1 = 1 0 1 0 | 0 0 1 0 | 0 1 1 1: six 1s, so p0 = 0
- cw = 1 0100 0100 1110 = **0x144E**, the value shown on HEX3–0 in mode 01 on the board

### Testbench (lines 27–58)

```systemverilog
 27 // Testbench: for all 256 data values the codeword is compared with a reference built the
 28 //   textbook way (place the data bits, then compute each check bit from the positions).
 29 // synthesis translate_off
 30 module ecc_encoder_testbench();
 31     logic [7:0]  data;
 32     logic [12:0] cw, ref_cw;
 33     integer      errors, n, k, b, j;
 34
 35     ecc_encoder dut (.data, .cw);
 36
 37     initial begin
 38         errors = 0;
 39         for (n = 0; n < 256; n++) begin
 40             data = n; #10;
 41             // reference: data bits at the positions that are not powers of two
 42             ref_cw = 13'd0; j = 0;
 43             for (k = 1; k <= 12; k++)
 44                 if (k != 1 && k != 2 && k != 4 && k != 8) begin ref_cw[k] = data[j]; j++; end
 45             for (b = 1; b <= 8; b = b * 2)
 46                 for (k = 1; k <= 12; k++)
 47                     if ((k & b) && k != b) ref_cw[b] = ref_cw[b] ^ ref_cw[k];
 48             ref_cw[0] = ^ref_cw[12:1];
 49             if (cw !== ref_cw) begin
 50                 $display("ERROR data %h: cw = %b, expected %b", data, cw, ref_cw); errors++;
 51             end
 52         end
 53         if (errors == 0) $display("ecc_encoder_testbench: PASS (all 256 data values)");
 54         else             $display("ecc_encoder_testbench: FAIL (%0d errors)", errors);
 55         $stop;
 56     end
 57 endmodule
 58 // synthesis translate_on
```

| Line  | How it is written / logic                                                                                                                                                                                                     |
| ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 27–28 | Comment: what is tested and how the expected value is built                                                                                                                                                                   |
| 31–33 | dut wires, the reference codeword `ref_cw`, the error count and loop variables                                                                                                                                                |
| 35    | Encoder instance, implicit connections                                                                                                                                                                                        |
| 38    | No errors yet                                                                                                                                                                                                                 |
| 39–40 | Loop over every 8-bit value 0–255. The encoder is combinational, so no clock: set `data`, wait `#10` wait cw to update                                                                                                        |
| 42    | Start the reference from all zeros; `j` walks through d0–d7                                                                                                                                                                   |
| 43–44 | **Place the data bits**: go through positions 1–12 and put the next data bit into every position that is not 1, 2, 4 or 8. This produces d0@3, d1@5, d2@6, d3@7, d4@9 … d7@12 without typing the table by hand                |
| 45–47 | **Compute each check bit the textbook way**: for b = 1, 2, 4, 8, XOR every position k whose number has bit b set (`k & b`), except b itself, into position b. This is the definition of a Hamming check bit, written as loops |
| 48    | Overall parity of bits 12–1 into bit 0                                                                                                                                                                                        |
| 49–51 | Compare the dut with the reference using `!==`, so an X in `cw` also counts as an error. Print the value in binary on a mismatch                                                                                              |
| 53–55 | PASS if all 256 match, then pause                                                                                                                                                                                             |

## 6. ecc\_decoder.sv

Pure combinational logic: takes the 13-bit codeword read from memory, works out which bit is wrong and whether it is a CE (**C**orrectable **E**rror) or a UE, and outputs the corrected codeword and the 8-bit data.

```systemverilog
 12 module ecc_decoder (cw, data, syndrome, err_pos, ce, ue, fixed_cw);
 13     input  logic [12:0] cw;          // codeword read from memory
 14     output logic [7:0]  data;        // corrected data (valid when ue = 0)
 15     output logic [3:0]  syndrome;
 16     output logic [3:0]  err_pos;     // bit that was flipped (valid when ce = 1)
 17     output logic        ce;          // correctable (single-bit) error
 18     output logic        ue;          // uncorrectable (double-bit) error
 19     output logic [12:0] fixed_cw;    // codeword with the single error flipped back
 20
 21     logic overall;
 22
 23     assign syndrome[0] = cw[1] ^ cw[3] ^ cw[5] ^ cw[7] ^ cw[9]  ^ cw[11];
 24     assign syndrome[1] = cw[2] ^ cw[3] ^ cw[6] ^ cw[7] ^ cw[10] ^ cw[11];
 25     assign syndrome[2] = cw[4] ^ cw[5] ^ cw[6] ^ cw[7] ^ cw[12];
 26     assign syndrome[3] = cw[8] ^ cw[9] ^ cw[10] ^ cw[11] ^ cw[12];
 27     assign overall     = ^cw;
 28
 29     assign ce      = overall & (syndrome <= 4'd12);
 30     assign ue      = (~overall & (syndrome != 4'd0)) | (overall & (syndrome > 4'd12));
 31     assign err_pos = syndrome;                         // 0 = the overall parity bit p0
 32
 33     assign fixed_cw = ce ? (cw ^ (13'd1 << syndrome)) : cw;
 34     assign data     = {fixed_cw[12:9], fixed_cw[7:5], fixed_cw[3]};
 35 endmodule
```

| Line  | How it is written / logic                                                                                                                                                                                                                                                                                          |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 12–19 | Ports: input `cw`; outputs corrected data, syndrome, error position, CE, UE and the corrected codeword. The comments say when each output is valid                                                                                                                                                                 |
| 23    | Syndrome bit 0 = **re-check the p1 group**: p1 itself (position 1) plus the positions it covers, 3, 5, 7, 9, 11. With no error the group has an even number of 1s and the XOR is 0; a 1 means the error position has bit 0 set                                                                                     |
| 24–26 | The same re-check for the p2, p4 and p8 groups gives syndrome bits 1–3. **The four results side by side are the error position in binary.** Equivalently: syndrome = XOR of the position numbers of every bit that is 1                                                                                            |
| 27    | `overall` = parity of all 13 bits. An odd number of flipped bits → 1; an even number (including none) → 0                                                                                                                                                                                                          |
| 29    | **CE condition**: overall = 1 means one bit flipped. A syndrome of 1–12 is that position; syndrome 0 means p0 itself flipped (it is in no group). Both are correctable                                                                                                                                             |
| 30    | **UE condition**, two cases. (1) overall = 0 but syndrome ≠ 0: overall parity looks fine but some group fails, which only happens with 2 flipped bits. (2) overall = 1 but syndrome 13–15: a single error cannot point to a position that does not exist, so there are 3 or more errors; treat it as UE to be safe |
| 31    | The error position is the syndrome; a separate name just makes the display side easier to read                                                                                                                                                                                                                     |
| 33    | Correction: on a CE, flip bit `syndrome` back with `1 << syndrome` (e.g. syndrome 5 → XOR `0000000100000`); otherwise pass `cw` through unchanged. On a UE nothing is flipped, because it is not known which two bits to flip                                                                                      |
| 34    | Pick the data bits out of the corrected codeword: positions 12–9 are d7–d4, 7–5 are d3–d1, 3 is d0. The reverse of encoder line 24                                                                                                                                                                                 |

**All cases at a glance**

| Syndrome | Overall | Meaning | Result |
| --- | --- | --- | --- |
| 0 | 0 | No error | Data as stored |
| 0 | 1 | p0 itself flipped | CE, flip bit 0; data not affected |
| 1–12 | 1 | One bit flipped | CE, flip bit `syndrome` |
| 13–15 | 1 | Cannot be a single bit | UE |
| ≠ 0 | 0 | Two bits flipped | UE |

### Testbench (lines 37–81, exhaustive)

```systemverilog
 37 // Testbench (exhaustive): for all 256 data values
 38 //   - no error              : ce = ue = 0, data correct                         (256 cases)
 39 //   - every single-bit flip : ce = 1, err_pos = flipped bit, data corrected      (256 x 13)
 40 //   - every double-bit flip : ue = 1, ce = 0                                     (256 x 78)
 41 // synthesis translate_off
 42 module ecc_decoder_testbench();
 43     logic [7:0]  din, data;
 44     logic [12:0] good, cw, fixed_cw;
 45     logic [3:0]  syndrome, err_pos;
 46     logic        ce, ue;
 47     integer      errors, n, i, j, n_clean, n_single, n_double;
 48
 49     ecc_encoder enc (.data(din), .cw(good));
 50     ecc_decoder dut (.cw, .data, .syndrome, .err_pos, .ce, .ue, .fixed_cw);
 51
 52     initial begin
 53         errors = 0; n_clean = 0; n_single = 0; n_double = 0;
 54         for (n = 0; n < 256; n++) begin
 55             din = n; #1;
 56             cw = good; #1;
 57             if (ce || ue || data !== din || syndrome != 0) begin $display("ERROR clean %h", din); errors++; end
 58             n_clean++;
 59             for (i = 0; i < 13; i++) begin
 60                 cw = good ^ (13'd1 << i); #1;
 61                 if (!ce || ue || data !== din || err_pos != i || fixed_cw !== good) begin
 62                     $display("ERROR data %h, bit %0d: ce=%b ue=%b data=%h pos=%0d", din, i, ce, ue, data, err_pos);
 63                     errors++;
 64                 end
 65                 n_single++;
 66                 for (j = i + 1; j < 13; j++) begin
 67                     cw = good ^ (13'd1 << i) ^ (13'd1 << j); #1;
 68                     if (!ue || ce) begin
 69                         $display("ERROR data %h, bits %0d+%0d: ce=%b ue=%b", din, i, j, ce, ue); errors++;
 70                     end
 71                     n_double++;
 72                 end
 73             end
 74         end
 75         $display("  checked %0d clean, %0d single-bit and %0d double-bit cases", n_clean, n_single, n_double);
 76         if (errors == 0) $display("ecc_decoder_testbench: PASS");
 77         else             $display("ecc_decoder_testbench: FAIL (%0d errors)", errors);
 78         $stop;
 79     end
 80 endmodule
 81 // synthesis translate_on
```

| Line  | How it is written / logic                                                                                                                                                                                 |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 37–40 | Comment: the three groups of cases and how many of each                                                                                                                                                   |
| 43–47 | `din` = original data, `good` = its correct codeword, `cw` = what the decoder gets (possibly with errors). Three counters prove how many cases actually ran                                               |
| 49    | **The encoder is used as the golden model**: it is already verified by its own testbench, so it produces the correct codeword for any `din`                                                               |
| 50    | The decoder under test                                                                                                                                                                                    |
| 53    | Clear the error count and the case counters                                                                                                                                                               |
| 54–56 | For every data value: set `din`, wait for the encoder, give the clean `good` codeword to the decoder, wait again. Both modules are combinational, so `#1` is enough                                       |
| 57–58 | **Clean case**: no CE, no UE, data unchanged, syndrome 0                                                                                                                                                  |
| 59–60 | **Single-bit case**: flip bit i (0–12) of the good codeword                                                                                                                                               |
| 61–64 | It must be a CE, not a UE; the data must be corrected back to `din`; `err_pos` must name the flipped bit; and `fixed_cw` must equal the original codeword exactly. On a failure, print every useful value |
| 66–67 | **Double-bit case**: for every j > i, flip bits i and j. Starting j at i + 1 tries each pair once and never flips the same bit twice (that would undo it)                                                 |
| 68–71 | It must be a UE and must not be a CE. A decoder that "corrected" a double error would silently return wrong data, the worst possible bug                                                                  |
| 75    | Print how many cases ran: expected 256, 3,328 and 19,968                                                                                                                                                  |
| 76–78 | PASS if no errors, then pause                                                                                                                                                                             |
## 7. fault\_injector.sv

Pure combinational logic: flips one chosen bit of a codeword, to imitate a memory bit hit by radiation (a soft error).

```systemverilog
  6 module fault_injector (cw_in, pos, cw_out, valid);
  7     input  logic [12:0] cw_in;
  8     input  logic [3:0]  pos;         // SW[3:0]
  9     output logic [12:0] cw_out;
 10     output logic        valid;
 11
 12     assign valid  = (pos <= 4'd12);
 13     assign cw_out = valid ? (cw_in ^ (13'd1 << pos)) : cw_in;
 14 endmodule
```

| Line | How it is written / logic                                                                                                                                                                                 |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 6–10 | Inputs: the original codeword and the position `pos` to flip (from SW\[3:0\]). Outputs: the flipped codeword and `valid`                                                                                  |
| 12   | A 4-bit pos can be 0–15, but the codeword only has bits 0–12. Only 0–12 is valid                                                                                                                          |
| 13   | `13'd1 (0 0000 0000 0001) << pos` builds a mask with only bit `pos` set; XOR with the codeword flips that bit (0→1 or 1→0) and leaves the rest alone. If not valid, the codeword passes through unchanged |

**What `valid` is for**: the datapath write enable is `do_inject & inj_valid`, so pressing KEY0 with the switches at 13–15 leaves memory completely unchanged instead of doing a pointless write.

**How a double error is made**: this module flips one bit at a time. Inject twice into the same word at two different positions to get a double error. Injecting the same position twice flips it back, which is the same as no error.

### Testbench (lines 16–45)

```systemverilog
 16 // Testbench: every position 0..15 on two different codewords.
 17 // synthesis translate_off
 18 module fault_injector_testbench();
 19     logic [12:0] cw_in, cw_out;
 20     logic [3:0]  pos;
 21     logic        valid;
 22     integer      errors, p, w;
 23
 24     fault_injector dut (.cw_in, .pos, .cw_out, .valid);
 25
 26     initial begin
 27         errors = 0;
 28         for (w = 0; w < 2; w++) begin
 29             cw_in = w ? 13'h1A5C : 13'h0000;
 30             for (p = 0; p < 16; p++) begin
 31                 pos = p; #10;
 32                 if (p < 13 && (cw_out !== (cw_in ^ (13'd1 << p)) || !valid)) begin
 33                     $display("ERROR pos %0d", p); errors++;
 34                 end
 35                 if (p >= 13 && (cw_out !== cw_in || valid)) begin
 36                     $display("ERROR pos %0d should do nothing", p); errors++;
 37                 end
 38             end
 39         end
 40         if (errors == 0) $display("fault_injector_testbench: PASS");
 41         else             $display("fault_injector_testbench: FAIL (%0d errors)", errors);
 42         $stop;
 43     end
 44 endmodule
 45 // synthesis translate_on
```

| Line  | How it is written / logic                                                                                                                                     |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 16    | Comment: what is tested                                                                                                                                       |
| 19–22 | dut wires, error count, loop variables `p` (position) and `w` (which codeword)                                                                                |
| 24    | Injector instance, implicit connections                                                                                                                       |
| 28–29 | Two input codewords: all zeros, and 1A5C (a mix of 0s and 1s). With all zeros, every flip is 0→1; with 1A5C some flips are 1→0, so both directions are tested |
| 30–31 | Try every 4-bit position 0–15; combinational, so `#10` and check                                                                                              |
| 32–34 | **Valid positions 0–12**: the output must be the input with exactly bit p flipped, and `valid` must be 1                                                      |
| 35–37 | **Invalid positions 13–15**: the output must be unchanged and `valid` must be 0, so the datapath will not write                                               |
| 40–42 | PASS if all 2 × 16 = 32 cases match, then pause                                                                                                               |
## 8. ecc\_mem.sv

A 16-word × 13-bit memory built from registers: 208 flip-flops (Lab 4 asks for at least 16 storage elements). One read/write port for the FSM and one read-only port for the display.

```systemverilog
  7 module ecc_mem (clk, reset, we, addr, wdata, rdata, view_addr, view_data);
  8     input  logic        clk, reset;
  9     input  logic        we;
 10     input  logic [3:0]  addr;
 11     input  logic [12:0] wdata;
 12     output logic [12:0] rdata;
 13     input  logic [3:0]  view_addr;
 14     output logic [12:0] view_data;
 15
 16     logic [12:0] mem [0:15];
 17
 18     always_ff @(posedge clk) begin
 19         if (reset) begin
 20             for (int i = 0; i < 16; i++) mem[i] <= 13'd0;
 21         end else if (we) begin
 22             mem[addr] <= wdata;
 23         end
 24     end
 25
 26     assign rdata     = mem[addr];
 27     assign view_data = mem[view_addr];
 28 endmodule
```

| Line  | How it is written / logic                                                                                                                                                                                                                                 |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 7–14  | Ports: `we` write enable, `addr` read/write address, `wdata` write data, `rdata` read data; `view_addr`/`view_data` is a second, read-only port                                                                                                           |
| 16    | A 2-D array: `[12:0]` before the name is the width of each word (packed); `[0:15]` after the name means 16 words (unpacked)                                                                                                                               |
| 18–20 | On reset, a `for` loop clears all 16 words. In hardware this is not a loop that runs 16 times; it synthesizes into 16 groups of registers that all reset at once. The codeword for data 0 is all zeros, so after reset every word is valid and error-free |
| 21–22 | On a rising edge with `we` = 1, only word `addr` is written; the other 15 keep their values                                                                                                                                                               |
| 26    | Combinational read: when the address changes, `rdata` changes in the same clock. It synthesizes into a 16-to-1 multiplexer                                                                                                                                |
| 27    | A second 16-to-1 multiplexer that always looks at the word the user selected, so HEX/LEDR show its contents at any time without waiting for the FSM                                                                                                       |

**Why two read ports**: while the scrubber runs, the FSM address switches to `scrub_addr`. If the display used the same port, HEX would keep jumping to whatever word the scrubber is checking. The second port is just one more multiplexer, so it costs very little.

**Why not M10K block RAM**: the DE1-SoC M10K blocks cannot be cleared all at once by a reset, and their read data arrives one clock later. For 16 words, registers are the simplest choice and every word is easy to see in ModelSim.

### Testbench (lines 30–69)

```systemverilog
 30 // Testbench: write a different 13-bit pattern to every word, read them back on both
 31 //   ports, check that reset clears the memory.
 32 // synthesis translate_off
 33 module ecc_mem_testbench();
 34     logic        clk, reset, we;
 35     logic [3:0]  addr, view_addr;
 36     logic [12:0] wdata, rdata, view_data;
 37     integer      errors, i;
 38
 39     ecc_mem dut (.clk, .reset, .we, .addr, .wdata, .rdata, .view_addr, .view_data);
 40
 41     parameter CLOCK_PERIOD = 100;
 42     initial begin
 43         clk <= 0;
 44         forever #(CLOCK_PERIOD / 2) clk <= ~clk;
 45     end
 46
 47     initial begin
 48         errors = 0;
 49         reset <= 1; we <= 0; addr <= 0; wdata <= 0; view_addr <= 0;
 50         @(posedge clk); reset <= 0;
 51         for (i = 0; i < 16; i++) begin
 52             we <= 1; addr <= i; wdata <= 13'h1000 + 37 * i; @(posedge clk);
 53         end
 54         we <= 0;
 55         for (i = 0; i < 16; i++) begin
 56             addr <= i; view_addr <= 15 - i; #10;
 57             if (rdata !== 13'h1000 + 37 * i || view_data !== 13'h1000 + 37 * (15 - i)) begin
 58                 $display("ERROR word %0d", i); errors++;
 59             end
 60             @(posedge clk);
 61         end
 62         reset <= 1; @(posedge clk); reset <= 0; addr <= 4'd7; #10;
 63         if (rdata !== 13'd0) begin $display("ERROR reset"); errors++; end
 64         if (errors == 0) $display("ecc_mem_testbench: PASS");
 65         else             $display("ecc_mem_testbench: FAIL (%0d errors)", errors);
 66         $stop;
 67     end
 68 endmodule
 69 // synthesis translate_on
```

| Line  | How it is written / logic                                                                                                                                                                                                                |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 30–31 | Comment: the three things tested                                                                                                                                                                                                         |
| 34–37 | Wires for both ports, error count, loop variable                                                                                                                                                                                         |
| 39    | Memory instance, implicit connections                                                                                                                                                                                                    |
| 41–45 | 100 ps clock                                                                                                                                                                                                                             |
| 49–50 | Reset for one edge with every input at a known value, then release reset                                                                                                                                                                 |
| 51–53 | **Write all 16 words**, one per clock: `we` = 1, address i, data `1000 + 37 × i`. The formula gives every word a different 13-bit pattern, so a wrong address (two words swapped, or one word written twice) would show up as a mismatch |
| 54    | Stop writing                                                                                                                                                                                                                             |
| 55–56 | **Read both ports at once, in opposite orders**: the read/write port goes 0, 1, … 15 while the display port goes 15, 14, … 0. The two ports always look at different words, which proves they are independent                            |
| 57–59 | Both read values must equal the pattern that was written to their word. Reads are combinational, so `#10` after setting the address is enough                                                                                            |
| 60    | Next clock, next word                                                                                                                                                                                                                    |
| 62–63 | **Reset test**: pulse reset, then read word 7; it must be 0 (any word would do, since reset clears all of them)                                                                                                                          |
| 64–66 | PASS if no errors, then pause                                                                                                                                                                                                            |
## 9. ecc\_control.sv

The control path: only the FSM and a few flags, no data. It decides what happens in each clock and tells the datapath through control signals. The memory has one read/write port shared by two users: you at the board (KEY0) and the background scrubber.

### 9.1 Comments and ports (lines 3–30)

```systemverilog
 17 module ecc_control (clk, reset, go, mode, scrub_toggle, scrub_tick, ce_now,
 18                     do_write, do_inject, ld_cw, check, do_fix, by_scrub, clr_count,
 19                     scrub_on, busy);
 20     input  logic       clk, reset;
 21     input  logic       go;            // KEY0 pulse
 22     input  logic [1:0] mode;          // SW[9:8]
 23     input  logic       scrub_toggle;  // KEY2 pulse
 24     input  logic       scrub_tick;    // time to scrub the next word
 25     input  logic       ce_now;        // from datapath: the latched codeword has a CE
 26     output logic       do_write, do_inject, ld_cw, check, do_fix;
 27     output logic       by_scrub;      // the current access belongs to the scrubber
 28     output logic       clr_count;     // clear the error counters
 29     output logic       scrub_on;      // scrubbing enabled
 30     output logic       busy;
```

| Line  | How it is written / logic                                                                                                                                                                                                            |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 3–16  | Header comment: the two users, the arbitration rule (the person wins), and what each of the 6 states does. It can be used directly as the FSM description in the report                                                              |
| 21–25 | Inputs: `go` = KEY0 pulse; `mode` = SW[9:8]; `scrub_toggle` = KEY2 pulse; `scrub_tick` = timer pulse from enable_gen; `ce_now` = datapath status "the latched codeword has a single-bit error", the only data status control can see |
| 26    | Five action signals, one per state: write, inject, latch codeword, check, fix                                                                                                                                                        |
| 27–30 | `by_scrub`: this access belongs to the scrubber (the datapath uses it to pick the address); `clr_count`: clear the counters; `scrub_on`: scrubber on/off; `busy`: FSM is not in IDLE                                                 |

### 9.2 States and arbitration (lines 32–40)

```systemverilog
 32     enum logic [2:0] { IDLE, WRITE, INJECT, READ_MEM, CHECK, FIX } ps, ns;
 33
 34     logic pending;                    // a KEY0 press is waiting
 35     logic scrub_due;                  // a scrub tick is waiting
 36     logic user_req, serve_user, start_scrub;
 37
 38     assign user_req    = pending | go;
 39     assign serve_user  = (ps == IDLE) & user_req;
 40     assign start_scrub = (ps == IDLE) & ~user_req & scrub_on & scrub_due;
```

| Line | How it is written / logic                                                                                                                                                                             |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 32   | 6 states need 3 bits (`logic [2:0]`). Lab 4 requires an FSM with at least 3 states                                                                                                                    |
| 34   | `pending`: remembers a KEY0 press that arrived while the FSM was busy. `go` lasts only 1 clock and would otherwise be gone when the FSM is free                                                       |
| 35   | `scrub_due`: the timer says scrub the next word, but it has not started yet. Same reason: `scrub_tick` is also only 1 clock                                                                           |
| 38   | `user_req`: the user wants something, either just pressed (`go`) or pressed earlier and still waiting (`pending`)                                                                                     |
| 39   | `serve_user`: in IDLE and the user wants something → start serving the user this clock                                                                                                                |
| 40   | `start_scrub`: in IDLE, **no user request**, scrubber on, and a tick waiting; all four must hold. `~user_req` is the "person first" arbitration rule: if both want the same clock, the person gets it |

### 9.3 Next-state logic (lines 42–63)

```systemverilog
 43     always_comb begin
 44         case (ps)
 45             IDLE:     if (user_req) begin
 46                           case (mode)
 47                               2'b00:   ns = WRITE;
 48                               2'b01:   ns = INJECT;
 49                               2'b10:   ns = READ_MEM;
 50                               default: ns = IDLE;    // 11: clear counters
 51                           endcase
 52                       end
 53                       else if (start_scrub)  ns = READ_MEM;
 54                       else                   ns = IDLE;
 55             WRITE:                           ns = IDLE;
 56             INJECT:                          ns = IDLE;
 57             READ_MEM:                        ns = CHECK;
 58             CHECK:    if (by_scrub & ce_now) ns = FIX;
 59                       else                   ns = IDLE;
 60             FIX:                             ns = IDLE;
 61             default:                         ns = IDLE;
 62         endcase
 63     end
```

| Current state | Condition                                          | Next state                  | What the datapath does in this state                     |
| ------------- | -------------------------------------------------- | --------------------------- | -------------------------------------------------------- |
| IDLE          | User request, mode 00 / 01 / 10                    | WRITE / INJECT / READ_MEM   | Waits                                                    |
| IDLE          | User request, mode 11                              | IDLE                        | `clr_count` is pulsed in the same clock                  |
| IDLE          | No user request, start_scrub                       | READ_MEM (for the scrubber) |                                                          |
| WRITE         | —                                                  | IDLE                        | Writes the encoded SW[7:0]                               |
| INJECT        | —                                                  | IDLE                        | Writes the codeword with one bit flipped                 |
| READ_MEM      | —                                                  | CHECK                       | Latches the memory codeword into `cw_reg`                |
| CHECK         | Scrubber access and CE                             | FIX                         | Decodes; for a user read, saves the result and counts it |
| CHECK         | Otherwise (user read, or scrubber sees clean / UE) | IDLE                        |                                                          |
| FIX           | —                                                  | IDLE                        | Writes the corrected codeword back                       |

Key lines:

- **Lines 45–52**: `mode` is read in the clock where service starts. If KEY0 was pressed during a scrub and went into pending, the SW value from a few clocks later is used, but nobody can move a switch in tens of nanoseconds, so it makes no difference.
- **Line 53**: `else if`, so the user is checked first and the scrubber second. Same rule as `~user_req` on line 40.
- **Line 57**: a read takes two states. READ_MEM only latches, CHECK decodes. The decoder input then comes from a register, which splits the long path memory mux → decoder → result registers into two parts and helps timing.
- **Line 58**: Only the **scrubber** goes to FIX on a CE. A user read reports the corrected data but does not write it back. This is on purpose: in the demo you can see the error still sitting in memory until the scrubber repairs it.
- **Line 61**: 3 bits encode 8 values but only 6 are used. An undefined encoding returns to IDLE.

### 9.4 Output logic (lines 65–72)

```systemverilog
 66     assign do_write  = (ps == WRITE);
 67     assign do_inject = (ps == INJECT);
 68     assign ld_cw     = (ps == READ_MEM);
 69     assign check     = (ps == CHECK);
 70     assign do_fix    = (ps == FIX);
 71     assign clr_count = serve_user & (mode == 2'b11);
 72     assign busy      = (ps != IDLE);
```

| Line  | How it is written / logic                                                                                                            |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------ |
| 66–70 | Moore outputs: each action signal depends only on the current state. Every state lasts 1 clock, so each action happens exactly once  |
| 71    | Clearing the counters needs no state of its own: when IDLE serves a mode-11 request, a 1-clock pulse goes out directly (Mealy style) |
| 72    | `busy`: the FSM is not in IDLE. Testbenches use it to wait for an operation to finish                                                |

### 9.5 State register and flags (lines 74–90)

```systemverilog
 75     always_ff @(posedge clk) begin
 76         if (reset) begin
 77             ps        <= IDLE;
 78             pending   <= 1'b0;
 79             scrub_due <= 1'b0;
 80             scrub_on  <= 1'b0;
 81             by_scrub  <= 1'b0;
 82         end else begin
 83             ps        <= ns;
 84             pending   <= user_req & ~serve_user;
 85             scrub_due <= scrub_on & (scrub_tick | (scrub_due & ~start_scrub));
 86             if (scrub_toggle) scrub_on <= ~scrub_on;
 87             if (start_scrub)      by_scrub <= 1'b1;
 88             else if (serve_user)  by_scrub <= 1'b0;
 89         end
 90     end
```

| Line  | How it is written / logic                                                                                                                                                                                                                                                             |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 76–81 | Reset: back to IDLE, all flags cleared, scrubber off by default, so the demo has time to inject errors first                                                                                                                                                                          |
| 83    | State advances                                                                                                                                                                                                                                                                        |
| 84    | `pending` = a request that was not served this clock. A `go` that arrives while busy stays here; in the clock the FSM is back in IDLE, `serve_user` = 1 and pending clears on the next clock                                                                                          |
| 85    | `scrub_due`: always 0 while the scrubber is off. While on, a tick sets it, and it stays 1 if it was 1 and the scrub has not started this clock. If the user keeps the FSM busy, the tick waits here instead of being lost                                                             |
| 86    | Each KEY2 press toggles scrub_on                                                                                                                                                                                                                                                      |
| 87–88 | `by_scrub` = 1 when a scrub starts and 0 when serving the user starts. It is a register, so it stays constant through READ_MEM, CHECK and FIX and the datapath knows which address the round uses. Leaving the old value in IDLE is harmless, since the datapath does nothing in IDLE |

### 9.6 Testbench (lines 93–179)

The datapath is not instantiated. The testbench drives `ce_now` by hand and counts how often each control signal fires, which tests the FSM and the arbitration on their own.

```systemverilog
101 module ecc_control_testbench();
102     logic       clk, reset, go, scrub_toggle, scrub_tick, ce_now;
103     logic [1:0] mode;
104     logic       do_write, do_inject, ld_cw, check, do_fix, by_scrub, clr_count, scrub_on, busy;
105     integer     errors, writes, injects, reads, user_checks, scrub_checks, fixes, clears;
106
107     ecc_control dut (.clk, .reset, .go, .mode, .scrub_toggle, .scrub_tick, .ce_now,
108                      .do_write, .do_inject, .ld_cw, .check, .do_fix, .by_scrub, .clr_count,
109                      .scrub_on, .busy);
110
111     parameter CLOCK_PERIOD = 100;
112     initial begin
113         clk <= 0;
114         forever #(CLOCK_PERIOD / 2) clk <= ~clk;
115     end
116
117     always_ff @(posedge clk) begin
118         if (reset) begin
119             writes <= 0; injects <= 0; reads <= 0; user_checks <= 0; scrub_checks <= 0; fixes <= 0; clears <= 0;
120         end else begin
121             if (do_write)  writes  <= writes + 1;
122             if (do_inject) injects <= injects + 1;
123             if (ld_cw)     reads   <= reads + 1;
124             if (check && !by_scrub) user_checks  <= user_checks + 1;
125             if (check &&  by_scrub) scrub_checks <= scrub_checks + 1;
126             if (do_fix)    fixes   <= fixes + 1;
127             if (clr_count) clears  <= clears + 1;
128         end
129     end
130
131     task press (input logic [1:0] m);
132         begin
133             mode <= m; go <= 1; @(posedge clk); go <= 0;
134             repeat (5) @(posedge clk);
135         end
136     endtask
137
138     initial begin
139         errors = 0;
140         go <= 0; mode <= 0; scrub_toggle <= 0; scrub_tick <= 0; ce_now <= 0;
141         reset <= 1; @(posedge clk); reset <= 0; @(posedge clk);
142
143         // 1. the four user operations
144         press(2'b00); press(2'b01); press(2'b10); press(2'b11);
145         #1;
146         if (writes != 1 || injects != 1 || reads != 1 || user_checks != 1 || clears != 1 || fixes != 0) begin
147             $display("ERROR user operations"); errors++;
148         end
149
150         // 2. scrubbing
151         scrub_tick <= 1; @(posedge clk); scrub_tick <= 0; repeat (5) @(posedge clk); #1;
152         if (scrub_checks != 0) begin $display("ERROR scrubbed while off"); errors++; end
153         scrub_toggle <= 1; @(posedge clk); scrub_toggle <= 0;
154         scrub_tick <= 1; @(posedge clk); scrub_tick <= 0; repeat (5) @(posedge clk); #1;
155         if (scrub_checks != 1 || fixes != 0) begin $display("ERROR clean scrub"); errors++; end
156         ce_now <= 1;
157         scrub_tick <= 1; @(posedge clk); scrub_tick <= 0; repeat (5) @(posedge clk); #1;
158         if (scrub_checks != 2 || fixes != 1) begin $display("ERROR scrub with CE did not FIX"); errors++; end
159         ce_now <= 0;
160
161         // 3a. go while the scrubber is busy
162         scrub_tick <= 1; @(posedge clk); scrub_tick <= 0;
163         wait (ld_cw && by_scrub);
164         mode <= 2'b10; go <= 1; @(posedge clk); go <= 0;
165         repeat (8) @(posedge clk); #1;
166         if (user_checks != 2 || scrub_checks != 3) begin $display("ERROR pending read was lost"); errors++; end
167
168         // 3b. go and a scrub tick at the same time: the person first
169         mode <= 2'b10; go <= 1; scrub_tick <= 1; @(posedge clk); go <= 0; scrub_tick <= 0;
170         #1; if (by_scrub) begin $display("ERROR scrubber went first"); errors++; end
171         repeat (10) @(posedge clk); #1;
172         if (user_checks != 3 || scrub_checks != 4) begin $display("ERROR both requests must be served"); errors++; end
173
174         if (errors == 0) $display("ecc_control_testbench: PASS");
175         else             $display("ecc_control_testbench: FAIL (%0d errors)", errors);
176         $stop;
177     end
178 endmodule
```

| Line | How it is written / logic |
| --- | --- |
| 93–100 | Comment listing the three test groups, then translate\_off |
| 102–105 | All dut inputs and outputs, plus 7 event counters and the error count |
| 107–109 | The control instance, implicit connections |
| 111–115 | 100 ps clock |
| 117–129 | **The monitor**: on every edge, count each control pulse. `check` is split by `by_scrub` into user checks and scrub checks. The test then only compares counters, instead of checking every signal on every clock |
| 131–136 | `press` task: set the mode, hold `go` for exactly one clock (like userInput), then wait 5 clocks, enough for the longest user operation (READ\_MEM → CHECK) to finish |
| 140–141 | All inputs to 0, reset for one edge, one more idle edge |
| 144–148 | **Test 1, user operations**: modes 00, 01, 10, 11 once each. Expect exactly one WRITE, one INJECT, one READ\_MEM, one user CHECK, one clear, and no FIX (a user read never fixes) |
| 151–152 | **Test 2a**: a tick while the scrubber is still off must do nothing |
| 153–155 | **Test 2b**: turn the scrubber on, send a tick: one scrub CHECK, no FIX because `ce_now` = 0 (a clean word) |
| 156–159 | **Test 2c**: same with `ce_now` = 1: the scrub must go on to FIX. Then `ce_now` back to 0 |
| 162–166 | **Test 3a, KEY0 during a scrub**: start a scrub, use `wait` to stop exactly when it is in READ\_MEM (`ld_cw && by_scrub`), and press `go` then. The FSM is busy, so the press must go into `pending` and be served after the scrub: user checks 1 → 2 and scrub checks 2 → 3. If `pending` did not work, the user read would be lost |
| 169–170 | **Test 3b, both in the same clock**: `go` and `scrub_tick` together. Right after that edge `by_scrub` must be 0, meaning the user was chosen |
| 171–172 | Wait 10 clocks: **both** must have been served (user 2 → 3, scrub 3 → 4), so the tick waited in `scrub_due` instead of being dropped |
| 174–176 | PASS if no errors, then pause |

## 10. ecc\_datapath.sv

The datapath: every data register, the memory, encoder, decoder and fault injector live here. It makes no decisions of its own; it acts on the control signals and reports `ce_now` back to control.

### 10.1 Ports (lines 10–23)

```systemverilog
 10 module ecc_datapath (clk, reset, do_write, do_inject, ld_cw, check, do_fix, by_scrub, clr_count,
 11                      addr_ld, sw, addr, scrub_addr, raw_cw, ce_now,
 12                      rd_data, rd_syn, rd_pos, rd_ce, rd_ue, ce_count, ue_count, fix_count);
 13     input  logic        clk, reset;
 14     input  logic        do_write, do_inject, ld_cw, check, do_fix, by_scrub, clr_count;  // from control
 15     input  logic        addr_ld;         // KEY1 pulse: addr <- SW[3:0]
 16     input  logic [7:0]  sw;              // SW[7:0]: data to write, or SW[3:0] = bit to flip
 17     output logic [3:0]  addr, scrub_addr;
 18     output logic [12:0] raw_cw;          // codeword stored at addr (display)
 19     output logic        ce_now;          // to control: cw_reg has a single-bit error
 20     output logic [7:0]  rd_data;
 21     output logic [3:0]  rd_syn, rd_pos;
 22     output logic        rd_ce, rd_ue;
 23     output logic [7:0]  ce_count, ue_count, fix_count;
```

| Line | How it is written / logic |
| --- | --- |
| 14 | The 7 control signals from control, one-to-one with control's outputs |
| 15 | KEY1 loads the address directly, without the FSM. Selecting a word does not touch memory, so it needs no arbitration |
| 16 | SW\[7:0\] means two things: the data in write mode, and the bit to flip (SW\[3:0\]) in inject mode |
| 17–23 | Outputs for the display: the current address, scrubber address, the raw codeword of the selected word, the last read result and the three counters. `ce_now` is the only status that goes back to control |

### 10.2 Memory and its address (lines 25–34)

```systemverilog
 25     logic [3:0]  mem_addr;
 26     logic [12:0] rdata, wdata, enc_cw, inj_cw, cw_reg, fixed_cw;
 27     logic        we, inj_valid, ue_now;
 28     logic [7:0]  dec_data;
 29     logic [3:0]  dec_syn, dec_pos;
 30
 31     // ---------------- memory and its address ----------------
 32     assign mem_addr = by_scrub ? scrub_addr : addr;
 33
 34     ecc_mem mem (.clk, .reset, .we, .addr(mem_addr), .wdata, .rdata, .view_addr(addr), .view_data(raw_cw));
```

| Line | How it is written / logic |
| --- | --- |
| 25–29 | Internal signals: memory address, read/write data, encoder output, injector output, latched codeword, and the decoder outputs |
| 32 | **Address multiplexer**: `scrub_addr` for a scrubber round, `addr` for a user round. This is what "two users share one port" looks like in hardware |
| 34 | The memory instance. The read/write port uses `mem_addr`; the display port is tied to the user's `addr` and drives `raw_cw`. `.clk` is an implicit connection; `.addr(mem_addr)` must be explicit because the names differ |

### 10.3 Encoder, injector, decoder, write mux (lines 36–50)

```systemverilog
 37     ecc_encoder enc (.data(sw), .cw(enc_cw));
 38
 39     fault_injector inj (.cw_in(rdata), .pos(sw[3:0]), .cw_out(inj_cw), .valid(inj_valid));
 40
 41     ecc_decoder dec (.cw(cw_reg), .data(dec_data), .syndrome(dec_syn), .err_pos(dec_pos),
 42                      .ce(ce_now), .ue(ue_now), .fixed_cw);
 43
 44     // what is written to memory
 45     always_comb begin
 46         if (do_write)       wdata = enc_cw;
 47         else if (do_inject) wdata = inj_cw;
 48         else                wdata = fixed_cw;            // do_fix
 49     end
 50     assign we = do_write | (do_inject & inj_valid) | do_fix;
```

| Line | How it is written / logic |
| --- | --- |
| 37 | The encoder input is SW\[7:0\] directly, so `enc_cw` is always the encoded version of the current switches; the WRITE state just stores it |
| 39 | The injector takes the memory read data `rdata`. In the INJECT clock the address is the user's `addr`: read, flip one bit, write back to the same address on the rising edge, a read-modify-write in one clock |
| 41–42 | The decoder input is the latched `cw_reg`, not `rdata`. The decoder's `ce` goes straight to the `ce_now` output for control; `.fixed_cw` has the same name and uses an implicit connection |
| 45–49 | **Write data multiplexer**: WRITE stores the encoded new data, INJECT the flipped codeword, anything else (in practice FIX) the corrected codeword. The final `else` means no latch |
| 50 | **Write enable**: only WRITE, a valid INJECT, or FIX write. At any other time `wdata` has a value but nothing is written |

**Why FIX writes to the right word**: during FIX `by_scrub` = 1 so `mem_addr` = `scrub_addr`, and `scrub_addr` only increments on the FIX rising edge. READ\_MEM, CHECK and FIX all use the same word.

### 10.4 Registers (lines 52–77)

```systemverilog
 53     always_ff @(posedge clk) begin
 54         if (reset) begin
 55             addr <= 4'd0; scrub_addr <= 4'd0; cw_reg <= 13'd0;
 56             rd_data <= 8'd0; rd_syn <= 4'd0; rd_pos <= 4'd0; rd_ce <= 1'b0; rd_ue <= 1'b0;
 57             ce_count <= 8'd0; ue_count <= 8'd0; fix_count <= 8'd0;
 58         end else begin
 59             if (addr_ld) addr <= sw[3:0];
 60             if (ld_cw)   cw_reg <= rdata;
 61
 62             if (check && !by_scrub) begin                // result of a user read
 63                 rd_data <= dec_data; rd_syn <= dec_syn; rd_pos <= dec_pos;
 64                 rd_ce <= ce_now; rd_ue <= ue_now;
 65                 if (ce_now && ce_count != 8'hFF) ce_count <= ce_count + 8'd1;
 66                 if (ue_now && ue_count != 8'hFF) ue_count <= ue_count + 8'd1;
 67             end
 68
 69             // the scrubber moves on after a clean / UE check, or after its FIX
 70             if ((check && by_scrub && !ce_now) || do_fix) scrub_addr <= scrub_addr + 4'd1;
 71             if (do_fix && fix_count != 8'hFF) fix_count <= fix_count + 8'd1;
 72
 73             if (clr_count) begin
 74                 ce_count <= 8'd0; ue_count <= 8'd0; fix_count <= 8'd0;
 75             end
 76         end
 77     end
```

| Line | How it is written / logic |
| --- | --- |
| 54–57 | Reset clears every register |
| 59 | On KEY1, load SW\[3:0\] into the address |
| 60 | In READ\_MEM, latch the memory codeword into `cw_reg`. In the next clock (CHECK) the decoder outputs are the result for this word |
| 62–64 | A **user read** saves its result in CHECK: corrected data, syndrome, error position, CE, UE. These are what mode 10 shows on HEX and LEDR9/8. A scrubber CHECK does not update them, so background activity never overwrites the display |
| 65–66 | CE / UE counts +1. `!= 8'hFF` makes them **saturate**: they stop at FF instead of wrapping to 00 and looking like no errors |
| 70 | **Scrubber advance rule**: after a check with no CE (clean or UE), move on; on a CE, stay and move on only after FIX writes back. `+ 4'd1` wraps from 15 to 0, so the scrubber sweeps memory in a loop |
| 71 | Each repaired word adds 1 to the fix count, also saturating |
| 73–75 | Clear the counters. This is the last statement in the always\_ff, so if it happens in the same clock as a +1 above, **the later assignment wins** and the result is 0 |

### 10.5 Testbench (lines 80–171)

Control is not instantiated. The testbench plays the role of control and sends the control signals itself, so each datapath operation can be checked in isolation.

```systemverilog
 86 module ecc_datapath_testbench();
 87     logic        clk, reset, do_write, do_inject, ld_cw, check, do_fix, by_scrub, clr_count, addr_ld;
 88     logic [7:0]  sw, rd_data, ce_count, ue_count, fix_count;
 89     logic [3:0]  addr, scrub_addr, rd_syn, rd_pos;
 90     logic [12:0] raw_cw;
 91     logic        ce_now, rd_ce, rd_ue;
 92     logic [12:0] good;
 93     integer      errors;
 94
 95     ecc_datapath dut (.clk, .reset, .do_write, .do_inject, .ld_cw, .check, .do_fix, .by_scrub, .clr_count,
 96                       .addr_ld, .sw, .addr, .scrub_addr, .raw_cw, .ce_now,
 97                       .rd_data, .rd_syn, .rd_pos, .rd_ce, .rd_ue, .ce_count, .ue_count, .fix_count);
 98
 99     parameter CLOCK_PERIOD = 100;
100     initial begin
101         clk <= 0;
102         forever #(CLOCK_PERIOD / 2) clk <= ~clk;
103     end
104
105     task pulse (input integer which);   // 0 addr_ld 1 write 2 inject 3 user read 4 clear
106         begin
107             case (which)
108                 0: begin addr_ld <= 1; @(posedge clk); addr_ld <= 0; end
109                 1: begin do_write <= 1; @(posedge clk); do_write <= 0; end
110                 2: begin do_inject <= 1; @(posedge clk); do_inject <= 0; end
111                 3: begin ld_cw <= 1; @(posedge clk); ld_cw <= 0; check <= 1; @(posedge clk); check <= 0; end
112                 4: begin clr_count <= 1; @(posedge clk); clr_count <= 0; end
113             endcase
114             #1;
115         end
116     endtask
117
118     initial begin
119         errors = 0;
120         do_write <= 0; do_inject <= 0; ld_cw <= 0; check <= 0; do_fix <= 0; by_scrub <= 0;
121         clr_count <= 0; addr_ld <= 0; sw <= 0;
122         reset <= 1; @(posedge clk); reset <= 0;
123
124         sw <= 8'd3; pulse(0);                              // addr = 3
125         sw <= 8'h5A; pulse(1);                             // write 5A
126         good = raw_cw;
127         pulse(3);
128         if (rd_ce || rd_ue || rd_data !== 8'h5A) begin $display("ERROR clean read"); errors++; end
129
130         sw <= 8'd6; pulse(2);                              // flip bit 6
131         if (raw_cw !== (good ^ 13'h0040)) begin $display("ERROR inject bit 6"); errors++; end
132         pulse(3);
133         if (!rd_ce || rd_ue || rd_data !== 8'h5A || rd_pos != 6) begin
134             $display("ERROR single error: ce=%b ue=%b data=%h pos=%0d", rd_ce, rd_ue, rd_data, rd_pos); errors++;
135         end
136         if (raw_cw !== (good ^ 13'h0040)) begin $display("ERROR a user read must not repair memory"); errors++; end
137
138         sw <= 8'd11; pulse(2);                             // second flip: double error
139         pulse(3);
140         if (!rd_ue || rd_ce) begin $display("ERROR double error not detected"); errors++; end
141         if (ce_count != 1 || ue_count != 1) begin $display("ERROR counters %0d %0d", ce_count, ue_count); errors++; end
142
143         sw <= 8'd9; pulse(0);                              // addr = 9
144         sw <= 8'hA5; pulse(1); good = raw_cw;
145         sw <= 8'd0; pulse(2);                              // flip p0
146         pulse(3);
147         if (!rd_ce || rd_pos != 0 || rd_data !== 8'hA5) begin $display("ERROR p0 error"); errors++; end
148
149         // scrubber visits word 9: set scrub_addr by stepping it through clean words 0..8
150         by_scrub <= 1;
151         while (scrub_addr != 4'd9) begin
152             ld_cw <= 1; @(posedge clk); ld_cw <= 0; check <= 1; @(posedge clk); check <= 0;
153         end
154         ld_cw <= 1; @(posedge clk); ld_cw <= 0; #1;
155         if (!ce_now) begin $display("ERROR scrubber did not see the CE"); errors++; end
156         check <= 1; @(posedge clk); check <= 0;
157         do_fix <= 1; @(posedge clk); do_fix <= 0; by_scrub <= 0; #1;
158         if (raw_cw !== good || fix_count != 1 || scrub_addr != 10) begin
159             $display("ERROR scrub repair: raw=%b fix=%0d", raw_cw, fix_count); errors++;
160         end
161
162         pulse(4);
163         if (ce_count || ue_count || fix_count) begin $display("ERROR clear counters"); errors++; end
164
165         if (errors == 0) $display("ecc_datapath_testbench: PASS");
166         else             $display("ecc_datapath_testbench: FAIL (%0d errors)", errors);
167         $stop;
168     end
169 endmodule
```

| Line | How it is written / logic |
| --- | --- |
| 87–93 | Every datapath input and output, `good` (a saved correct codeword) and the error count |
| 95–97 | The datapath instance, implicit connections |
| 99–103 | 100 ps clock |
| 105–116 | `pulse` task = **a stand-in for control**. Each case produces the same signal sequence the FSM would: 0 KEY1, 1 WRITE, 2 INJECT, 3 a user read (READ\_MEM then CHECK, two clocks), 4 clear. The final `#1` lets the `<=` updates finish so the caller can check results right away |
| 120–122 | All control signals to 0, reset for one edge |
| 124–126 | Select word 3 and write 5A. Save the stored codeword as `good` for later comparisons |
| 127–128 | **Clean read**: no CE, no UE, data 5A |
| 130–131 | Flip bit 6 (a data bit, d2). The stored codeword must be `good` XOR `0x0040`, bit 6 only |
| 132–135 | **Single-error read**: CE, not UE, data still corrected to 5A, error position 6 |
| 136 | **A user read must not repair memory**: the stored word must still contain the flipped bit |
| 138–141 | Flip bit 11 too → **double error**: UE, not CE. Counters now CE 1, UE 1 |
| 143–147 | Word 9: write A5, flip bit 0 (the overall parity p0, a check bit). Read must be CE at position **0** with data A5, the edge case where the syndrome is 0 |
| 150–153 | **Act as the scrubber**: `by_scrub` = 1, then run READ\_MEM + CHECK on words 0–8. None of them has a CE (word 3 holds the double error, and a UE also moves on), so each check moves `scrub_addr` forward, until it reaches 9 |
| 154–155 | READ\_MEM on word 9: after latching, `ce_now` must be 1, which is what tells control to go to FIX |
| 156–157 | CHECK, then FIX, then back to user mode |
| 158–160 | **Repair check**: word 9 must equal the original `good` again, `fix_count` = 1, and `scrub_addr` must have moved on to 10 |
| 162–163 | Clear: all three counters must be 0 |
| 165–167 | PASS if no errors, then pause |

## 11. ecc\_core.sv

Connects control and datapath into the complete ECC memory system, with the same structure as binarySearch in Lab 3. It contains no logic, only wiring.

```systemverilog
  5 module ecc_core (clk, reset, go, mode, addr_ld, scrub_toggle, scrub_tick, sw,
  6                  addr, scrub_addr, raw_cw, rd_data, rd_syn, rd_pos, rd_ce, rd_ue,
  7                  ce_count, ue_count, fix_count, scrub_on, busy);
  8     input  logic        clk, reset;
  9     input  logic        go, addr_ld, scrub_toggle;   // KEY0, KEY1, KEY2 pulses
 10     input  logic [1:0]  mode;                        // SW[9:8]
 11     input  logic        scrub_tick;
 12     input  logic [7:0]  sw;                          // SW[7:0]
 13     output logic [3:0]  addr, scrub_addr, rd_syn, rd_pos;
 14     output logic [12:0] raw_cw;
 15     output logic [7:0]  rd_data, ce_count, ue_count, fix_count;
 16     output logic        rd_ce, rd_ue, scrub_on, busy;
 17
 18     logic do_write, do_inject, ld_cw, check, do_fix, by_scrub, clr_count, ce_now;
 19
 20     ecc_control ctrl (.clk, .reset, .go, .mode, .scrub_toggle, .scrub_tick, .ce_now,
 21                       .do_write, .do_inject, .ld_cw, .check, .do_fix, .by_scrub, .clr_count,
 22                       .scrub_on, .busy);
 23
 24     ecc_datapath dp (.clk, .reset, .do_write, .do_inject, .ld_cw, .check, .do_fix, .by_scrub, .clr_count,
 25                      .addr_ld, .sw, .addr, .scrub_addr, .raw_cw, .ce_now,
 26                      .rd_data, .rd_syn, .rd_pos, .rd_ce, .rd_ue, .ce_count, .ue_count, .fix_count);
 27 endmodule
```

| Line | How it is written / logic |
| --- | --- |
| 5–16 | External ports. Inputs are already-cleaned button pulses, mode, SW and the timer pulse; outputs are every value to display. It knows nothing about HEX or LEDR; display is the top level's job |
| 18 | Internal wires between control and datapath: 7 control signals going down, 1 status (`ce_now`) going up |
| 20–22 | Control instance. Every port has the same name as its signal, so all connections are implicit |
| 24–26 | Datapath instance, also all implicit |

**Why this extra level**: (1) Lab 4 requires separate control and datapath, and this level makes the boundary obvious. (2) ecc\_core has nothing board-specific, so a system-level testbench can drive it directly without simulating buttons and seven-segment displays.

### Testbench (lines 29–135, randomized end to end)

The first test of control and datapath working together, with the real FSM and real arbitration.

```systemverilog
 39 module ecc_core_testbench();
 40     logic        clk, reset, go, addr_ld, scrub_toggle, scrub_tick;
 41     logic [1:0]  mode;
 42     logic [7:0]  sw, rd_data, ce_count, ue_count, fix_count;
 43     logic [3:0]  addr, scrub_addr, rd_syn, rd_pos;
 44     logic [12:0] raw_cw;
 45     logic        rd_ce, rd_ue, scrub_on, busy;
 46     logic [7:0]  value [0:15];
 47     integer      kind  [0:15];        // 0 clean, 1 single, 2 double
 48     integer      errors, i, b1, b2, t;
 49
 50     ecc_core dut (.clk, .reset, .go, .mode, .addr_ld, .scrub_toggle, .scrub_tick, .sw,
 51                   .addr, .scrub_addr, .raw_cw, .rd_data, .rd_syn, .rd_pos, .rd_ce, .rd_ue,
 52                   .ce_count, .ue_count, .fix_count, .scrub_on, .busy);
 53
 54     parameter CLOCK_PERIOD = 100;
 55     initial begin
 56         clk <= 0;
 57         forever #(CLOCK_PERIOD / 2) clk <= ~clk;
 58     end
 59
 60     // scrub tick every 7 clocks
 61     always_ff @(posedge clk) begin
 62         if (reset) t <= 0; else t <= (t == 6) ? 0 : t + 1;
 63     end
 64     assign scrub_tick = (t == 6);
 65
 66     task op (input logic [1:0] m, input logic [7:0] s);   // one KEY0 press
 67         begin
 68             mode <= m; sw <= s; go <= 1; @(posedge clk); go <= 0; #1;
 69             while (dut.ctrl.pending || busy) begin @(posedge clk); #1; end   // served and finished
 70             @(posedge clk); #1;
 71         end
 72     endtask
 73
 74     task select (input integer a);                        // KEY1: addr = a
 75         begin
 76             sw <= a; addr_ld <= 1; @(posedge clk); addr_ld <= 0; #1;
 77         end
 78     endtask
 79
 80     task read_check (input integer a, input integer expect_kind);
 81         begin
 82             select(a); op(2'b10, 8'd0);
 83             if (expect_kind == 0 && (rd_ce || rd_ue || rd_data !== value[a])) begin
 84                 $display("ERROR word %0d: expected clean %h, got %h ce=%b ue=%b", a, value[a], rd_data, rd_ce, rd_ue); errors++;
 85             end
 86             if (expect_kind == 1 && (!rd_ce || rd_ue || rd_data !== value[a])) begin
 87                 $display("ERROR word %0d: expected CE with %h, got %h ce=%b ue=%b", a, value[a], rd_data, rd_ce, rd_ue); errors++;
 88             end
 89             if (expect_kind == 2 && (!rd_ue || rd_ce)) begin
 90                 $display("ERROR word %0d: expected UE", a); errors++;
 91             end
 92         end
 93     endtask
 94
 95     initial begin
 96         errors = 0;
 97         go <= 0; addr_ld <= 0; scrub_toggle <= 0; mode <= 0; sw <= 0;
 98         reset <= 1; @(posedge clk); reset <= 0; @(posedge clk);
 99
100         // 1. write random data
101         for (i = 0; i < 16; i++) begin
102             value[i] = $urandom; select(i); op(2'b00, value[i]);
103         end
104         // 2. inject faults
105         for (i = 0; i < 16; i++) begin
106             kind[i] = (i % 8 < 3) ? 1 : ((i % 8 < 5) ? 2 : 0);   // 6 single, 4 double, 6 clean
107             select(i);
108             if (kind[i] >= 1) begin b1 = $urandom % 13; op(2'b01, b1); end
109             if (kind[i] == 2) begin b2 = (b1 + 1 + $urandom % 12) % 13; op(2'b01, b2); end
110         end
111         // 3. read everything
112         for (i = 0; i < 16; i++) read_check(i, kind[i]);
113         $display("  before scrubbing: CE count = %0d, UE count = %0d", ce_count, ue_count);
114         if (ce_count != 6 || ue_count != 4) begin $display("ERROR counters"); errors++; end
115         for (i = 0; i < 16; i++) read_check(i, kind[i]);           // reading does not repair
116
117         // 4. scrub, with user reads in between (5.)
118         op(2'b11, 8'd0);                                          // clear counters
119         scrub_toggle <= 1; @(posedge clk); scrub_toggle <= 0;
120         for (i = 0; i < 16; i++) read_check(i, -1);              // reads during scrubbing
121         repeat (2 * 16 * 7 + 50) @(posedge clk);
122         scrub_toggle <= 1; @(posedge clk); scrub_toggle <= 0;
123         repeat (5) @(posedge clk);
124         $display("  scrubber repaired %0d words", fix_count);
125         if (fix_count != 6) begin $display("ERROR fix_count = %0d", fix_count); errors++; end
126         op(2'b11, 8'd0);
127         for (i = 0; i < 16; i++) read_check(i, kind[i] == 1 ? 0 : kind[i]);
128         if (ce_count != 0 || ue_count != 4) begin $display("ERROR after scrub: CE %0d UE %0d", ce_count, ue_count); errors++; end
129
130         if (errors == 0) $display("ecc_core_testbench: PASS");
131         else             $display("ecc_core_testbench: FAIL (%0d errors)", errors);
132         $stop;
133     end
134 endmodule
```

| Line | How it is written / logic |
| --- | --- |
| 29–37 | Comment listing the five test steps, then translate\_off |
| 40–45 | Every ecc\_core input and output |
| 46–47 | **The scoreboard**: `value[i]` remembers the data written to word i, and `kind[i]` remembers what error it was given (0 clean, 1 single, 2 double). Every read is judged against these |
| 48 | Error count, loop variable, the two flipped bit positions, and the tick timer |
| 50–52 | The ecc\_core instance |
| 54–58 | 100 ps clock |
| 60–64 | A stand-in for enable\_gen: `scrub_tick` = 1 every 7 clocks. Much faster than the board, so a full sweep fits in a short simulation |
| 66–72 | `op` task = **one KEY0 press**: set mode and SW, `go` for one clock, then wait while the request is still queued (`dut.ctrl.pending`) or the FSM is busy. Checking the internal `pending` with a hierarchical name matters: if the scrubber is busy, the press sits in pending while `busy` already looks done for a moment |
| 74–78 | `select` task = **one KEY1 press**: `addr_ld` for one clock with the word number on SW |
| 80–93 | `read_check` task: select word `a`, read it, and compare with the scoreboard. Clean → no CE/UE and the original data. Single → CE, no UE, and the **original** data (proof of correction). Double → UE, no CE. `expect_kind = -1` checks nothing, used when only the read itself matters |
| 97–98 | All inputs to 0, reset, one idle edge |
| 101–103 | **Step 1**: write a random byte to each of the 16 words and remember it |
| 105–106 | **Step 2**: assign error types by word number: `i % 8` of 0–2 → single, 3–4 → double, 5–7 → clean. That is 6 single, 4 double and 6 clean words |
| 108 | Single and double words: flip a random bit `b1` (0–12) |
| 109 | Double words: flip a second bit `b2`. `b1 + 1 + (0–11)` mod 13 is always a different position from `b1`, so the two flips never cancel |
| 112–114 | **Step 3**: read every word and check it. The counters must be exactly CE 6, UE 4 |
| 115 | Read everything a second time with the same expectations: if a user read had repaired memory, the 6 single errors would now read clean and fail |
| 118–119 | **Step 4**: clear the counters (mode 11), turn the scrubber on |
| 120 | **Arbitration test**: read all 16 words while the scrubber is running. Results are not checked (`-1`), because the scrubber may or may not have reached a word yet; what matters is that `op` returns every time, meaning no user request was lost |
| 121–123 | Let the scrubber run for two full sweeps (2 × 16 words × 7 clocks, plus margin), then turn it off and let the last operation finish |
| 124–125 | Exactly the **6** single-error words must have been repaired; the 4 double errors must not be "repaired" |
| 126–128 | **Step 5**: clear the counters and read everything again. Former single-error words must now be **clean** (`kind 1 → expect 0`), double errors are still UE. Counters must be CE 0, UE 4 |
| 130–132 | PASS if no errors, then pause |

## 12. DE1\_SoC.sv

The board top level: turns KEY/SW into clean signals for ecc\_core, then decides what HEX0–5 and LEDR show in each operating mode.

### 12.1 Comments, parameter and ports (lines 3–44)

```systemverilog
 26 module DE1_SoC #(parameter SCRUB_DIV = 12_500_000)     // 50 MHz / 12.5 M = 4 words per second
 27                 (CLOCK_50, HEX0, HEX1, HEX2, HEX3, HEX4, HEX5, KEY, LEDR, SW);
 28     input  logic       CLOCK_50;
 29     output logic [6:0] HEX0, HEX1, HEX2, HEX3, HEX4, HEX5;
 30     input  logic [3:0] KEY;
 31     output logic [9:0] LEDR;
 32     input  logic [9:0] SW;
 33
 34     logic        reset, scrub_tick;
 35     logic [3:0]  key_s;              // synchronized keys, 1 = pressed
 36     logic [9:0]  sw_s;               // synchronized switches
 37     logic        go, addr_ld, scrub_toggle;
 38     logic [3:0]  addr, scrub_addr, rd_syn, rd_pos;
 39     logic [12:0] raw_cw;
 40     logic [7:0]  raw_data, rd_data, ce_count, ue_count, fix_count;
 41     logic        rd_ce, rd_ue, scrub_on, busy;
 42     logic [3:0]  digit [0:5];        // value shown on HEX0..HEX5
 43     logic [5:0]  blank;              // 1 = that HEX display is off
 44     logic [6:0]  seg [0:5];
```

| Line | How it is written / logic |
| --- | --- |
| 3–25 | The header comment is the user manual: what each KEY and SW does and what HEX/LEDR show in each mode |
| 26 | `SCRUB_DIV` is a parameter: 12,500,000 on the board (4 words per second, one sweep in 4 seconds), 20 in the testbench. The underscores in `12_500_000` are only for readability |
| 27–32 | Port names must match the pin assignments in the qsf, or Quartus cannot connect them to the board |
| 35–36 | `key_s`, `sw_s`: the synchronized versions. All later logic uses only these, never KEY/SW directly |
| 42–44 | Display arrays: `digit[i]` = the value for HEX i, `blank[i]` = turn HEX i off, `seg[i]` = the seg7 output |

### 12.2 Input handling (lines 46–56)

```systemverilog
 47     synchronizer #(.WIDTH(4))  sync_key (.clk(CLOCK_50), .in(~KEY), .out(key_s));
 48     synchronizer #(.WIDTH(10)) sync_sw  (.clk(CLOCK_50), .in(SW),   .out(sw_s));
 49
 50     assign reset = key_s[3];
 51
 52     userInput press0 (.clk(CLOCK_50), .reset, .in(key_s[0]), .out(go));
 53     userInput press1 (.clk(CLOCK_50), .reset, .in(key_s[1]), .out(addr_ld));
 54     userInput press2 (.clk(CLOCK_50), .reset, .in(key_s[2]), .out(scrub_toggle));
 55
 56     enable_gen #(.DIV(SCRUB_DIV)) scrub_timer (.clk(CLOCK_50), .reset, .en(scrub_tick));
```

| Line | How it is written / logic |
| --- | --- |
| 47 | DE1-SoC KEYs are **active-low** (pressed = 0), so `~KEY` inverts them to pressed = 1 before synchronizing |
| 48 | All 10 SWs synchronized together |
| 50 | reset = KEY3 after the synchronizer, so it is a synchronous reset, active while held |
| 52–54 | Three userInputs turn KEY0/1/2 into 1-clock pulses: execute, select word, scrubber on/off |
| 56 | The scrubber timer: one `scrub_tick` every `SCRUB_DIV` clocks |

### 12.3 ecc\_core instance (lines 58–63)

```systemverilog
 59     ecc_core core (.clk(CLOCK_50), .reset, .go, .mode(sw_s[9:8]), .addr_ld, .scrub_toggle, .scrub_tick,
 60                    .sw(sw_s[7:0]), .addr, .scrub_addr, .raw_cw, .rd_data, .rd_syn, .rd_pos, .rd_ce, .rd_ue,
 61                    .ce_count, .ue_count, .fix_count, .scrub_on, .busy);
 62
 63     assign raw_data = {raw_cw[12:9], raw_cw[7:5], raw_cw[3]};   // data bits as stored
```

| Line | How it is written / logic |
| --- | --- |
| 59–61 | SW\[9:8\] goes in as the mode and SW\[7:0\] as the data; the other same-named signals connect implicitly |
| 63 | Picks the data bits out of the selected word's **raw** codeword (no correction), the same selection as decoder line 34. So LEDR7-0 show what is really stored in memory; after flipping bit 5 you see A7 |

### 12.4 HEX display modes (lines 65–93)

```systemverilog
 66     always_comb begin
 67         blank = 6'b000000;
 68         digit[5] = addr; digit[4] = 4'd5; digit[3] = 4'd0; digit[2] = 4'd0; digit[1] = 4'd0; digit[0] = 4'd0;
 69         blank[4] = ~scrub_on;                                   // "S" while scrubbing
 70         case (sw_s[9:8])
 71             2'b00: begin                                       // WRITE
 72                 digit[3] = raw_data[7:4]; digit[2] = raw_data[3:0];
 73                 digit[1] = sw_s[7:4];     digit[0] = sw_s[3:0];
 74             end
 75             2'b01: begin                                       // INJECT
 76                 digit[4] = sw_s[3:0]; blank[4] = 1'b0;
 77                 digit[3] = {3'b000, raw_cw[12]}; digit[2] = raw_cw[11:8];
 78                 digit[1] = raw_cw[7:4];          digit[0] = raw_cw[3:0];
 79             end
 80             2'b10: begin                                       // READ
 81                 digit[3] = rd_syn; digit[2] = rd_pos;
 82                 digit[1] = rd_data[7:4]; digit[0] = rd_data[3:0];
 83                 blank[2] = ~rd_ce;
 84                 blank[1] = rd_ue; blank[0] = rd_ue;
 85             end
 86             default: begin                                     // STATS
 87                 digit[5] = ce_count[7:4];  digit[4] = ce_count[3:0];
 88                 digit[3] = ue_count[7:4];  digit[2] = ue_count[3:0];
 89                 digit[1] = fix_count[7:4]; digit[0] = fix_count[3:0];
 90                 blank[4] = 1'b0;
 91             end
 92         endcase
 93     end
```

| Line | How it is written / logic |
| --- | --- |
| 67–69 | **Defaults first**: all on, HEX5 = current word, HEX4 = 5, the rest 0; HEX4 off while the scrubber is off. The case below only overrides what changes. Every output has a value on every path, so `always_comb` infers no latch and each branch does not need all 12 assignments |
| 71–74 | **00 WRITE**: HEX3-2 = data currently in memory, HEX1-0 = the new data on the switches. Compare before and after KEY0 to see the write |
| 75–79 | **01 INJECT**: HEX4 shows the bit to flip and is always on. HEX3-0 = the 13-bit codeword in hex; the top digit only has bit 12, padded with 3 zeros to 4 bits, so HEX3 is always 0 or 1 |
| 80–85 | **10 READ**: HEX3 syndrome, HEX2 error position, HEX1-0 corrected data. HEX2 is off when there is no CE (no position to show); HEX1-0 are off on a UE (the data cannot be trusted, so no wrong value is shown) |
| 86–91 | **11 STATS**: the three 8-bit counters, two digits each. HEX5 no longer shows the address and HEX4 is always on (a digit, not S). Using `default` for the last mode makes the case complete |

### 12.5 Seven-segment displays and LEDs (lines 95–112)

```systemverilog
 95     genvar d;
 96     generate
 97         for (d = 0; d < 6; d++) begin : hexes
 98             seg7 h (.hex(digit[d]), .leds(seg[d]));
 99         end
100     endgenerate
101
102     assign HEX0 = blank[0] ? 7'b1111111 : seg[0];
  …
107     assign HEX5 = blank[5] ? 7'b1111111 : seg[5];
108
109     assign LEDR[9]   = rd_ue;
110     assign LEDR[8]   = rd_ce;
111     assign LEDR[7:0] = (sw_s[9:8] == 2'b11) ? {scrub_on, 3'b000, scrub_addr} : raw_data;
112 endmodule
```

| Line | How it is written / logic |
| --- | --- |
| 95–100 | `generate for` places 6 seg7 instances, the same as writing 6 instantiations by hand. `genvar` is a loop variable that exists only at compile time; `hexes` is the block name, so ModelSim shows `hexes[0].h` through `hexes[5].h` |
| 102–107 | If `blank` = 1, output `1111111` (active-low, all off); otherwise the seg7 output |
| 109–110 | LEDR9 = the last read was a UE, LEDR8 = it was a CE. They are datapath registers, so they stay on until the next read |
| 111 | LEDR7-0: in mode 11, scrubber status (LEDR7 = on, LEDR3-0 = next word to check, so you can watch it count); in other modes, the raw data stored in memory |

### 12.6 Testbench (lines 114–219)

The whole board, driven only through KEY and SW and checked only through HEX and LEDR, exactly like a person at the board. `SCRUB_DIV` is overridden to 20 so one scrub sweep takes 320 clocks instead of 4 seconds.

```systemverilog
122 module DE1_SoC_testbench();
123     logic       CLOCK_50;
124     logic [6:0] HEX0, HEX1, HEX2, HEX3, HEX4, HEX5;
125     logic [3:0] KEY;
126     logic [9:0] LEDR, SW;
127     integer     errors;
128     logic [6:0] s [0:15];
129     localparam logic [6:0] OFF = 7'b1111111;
130
131     DE1_SoC #(.SCRUB_DIV(20)) dut (.CLOCK_50, .HEX0, .HEX1, .HEX2, .HEX3, .HEX4, .HEX5, .KEY, .LEDR, .SW);
132
133     parameter CLOCK_PERIOD = 100;
134     initial begin
135         CLOCK_50 <= 0;
136         forever #(CLOCK_PERIOD / 2) CLOCK_50 <= ~CLOCK_50;
137     end
138
139     task press (input integer k);
140         begin
141             KEY[k] <= 0; repeat (4) @(posedge CLOCK_50);
142             KEY[k] <= 1; repeat (6) @(posedge CLOCK_50);
143         end
144     endtask
145
146     task do_op (input logic [1:0] mode, input logic [7:0] value);   // set SW, press KEY0
147         begin
148             SW[9:8] <= mode; SW[7:0] <= value; repeat (3) @(posedge CLOCK_50);
149             press(0);
150         end
151     endtask
152
153     task select (input integer a);
154         begin
155             SW[7:0] <= a; repeat (3) @(posedge CLOCK_50); press(1);
156         end
157     endtask
158
159     task show (input string what, input logic [6:0] h5, h4, h3, h2, h1, h0);
160         begin
161             repeat (3) @(posedge CLOCK_50);
162             if ({HEX5, HEX4, HEX3, HEX2, HEX1, HEX0} !== {h5, h4, h3, h2, h1, h0}) begin
163                 $display("ERROR %s: HEX does not match", what); errors++;
164             end else $display("  %-40s OK", what);
165         end
166     endtask
167
168     initial begin
169         s[0] = 7'b1000000; s[1] = 7'b1111001; s[2] = 7'b0100100; s[3] = 7'b0110000;
170         s[4] = 7'b0011001; s[5] = 7'b0010010; s[6] = 7'b0000010; s[7] = 7'b1111000;
171         s[8] = 7'b0000000; s[9] = 7'b0010000; s[10] = 7'b0001000; s[11] = 7'b0000011;
172         s[12] = 7'b1000110; s[13] = 7'b0100001; s[14] = 7'b0000110; s[15] = 7'b0001110;
173         errors = 0;
174         KEY <= 4'b1111; SW <= 10'd0;
175         repeat (3) @(posedge CLOCK_50);
176         press(3);
177
178         // 1. write A5 to word 3, flip bit 5, read
179         select(3);
180         do_op(2'b00, 8'hA5);
181         show("write A5 to word 3", s[3], OFF, s[10], s[5], s[10], s[5]);
182         SW[9:8] <= 2'b01; SW[7:0] <= 8'd5;
183         show("inject view: codeword 144E", s[3], s[5], s[1], s[4], s[4], s[14]);
184         press(0);                                                // flip bit 5 (data bit d1)
185         if (LEDR[7:0] !== 8'hA7) begin $display("ERROR LEDR should show A7 (bit flipped)"); errors++; end
186         do_op(2'b10, 8'd0);
187         show("read: CE at bit 5, data A5", s[3], OFF, s[5], s[5], s[10], s[5]);
188         if (!LEDR[8] || LEDR[9]) begin $display("ERROR LEDR8/9 after a single error"); errors++; end
189
190         // 2. second flip: UE
191         do_op(2'b01, 8'd10);
192         do_op(2'b10, 8'd0);
193         if (!LEDR[9] || LEDR[8] || HEX1 !== OFF) begin $display("ERROR double error"); errors++; end
194         else $display("  %-40s OK", "read after 2 flips: UE (LEDR9)");
195
196         // 3. word 7 with a check-bit error, then scrub
197         select(7);
198         do_op(2'b00, 8'h3C);
199         do_op(2'b01, 8'd2);                                      // flip p2
200         press(2);                                                // scrubber on
201         repeat (16 * 20 + 100) @(posedge CLOCK_50);
202         press(2);                                                // scrubber off
203         do_op(2'b10, 8'd0);
204         show("word 7 repaired by the scrubber", s[7], OFF, s[0], OFF, s[3], s[12]);
205         select(3); do_op(2'b10, 8'd0);
206         if (!LEDR[9]) begin $display("ERROR word 3 should still be UE"); errors++; end
207         SW[9:8] <= 2'b11;
208         show("stats: CE 01, UE 02, repaired 01", s[0], s[1], s[0], s[2], s[0], s[1]);
209
210         // 4. clear the counters
211         press(0);
212         show("stats cleared", s[0], s[0], s[0], s[0], s[0], s[0]);
213
214         if (errors == 0) $display("DE1_SoC_testbench: PASS");
215         else             $display("DE1_SoC_testbench: FAIL (%0d errors)", errors);
216         $stop;
217     end
218 endmodule
```

| Line | How it is written / logic |
| --- | --- |
| 114–121 | Comment listing the four test steps, then translate\_off |
| 123–126 | The board's pins as testbench signals |
| 127–129 | Error count; `s[0..15]` = the seg7 pattern for each hex digit, so expected HEX values can be written as `s[10]` ("A") instead of raw 7-bit numbers; `OFF` = a blanked display |
| 131 | The top-level instance with `#(.SCRUB_DIV(20))` |
| 133–137 | 100 ps clock on `CLOCK_50` |
| 139–144 | `press` task = **pressing a real button**: KEY is active-low, so 0 = down for 4 clocks, then 1 = up for 6 clocks. 4 clocks is longer than the synchronizer's 2-clock delay, so the press is guaranteed to get through, and the 6 clocks afterwards let the FSM finish |
| 146–151 | `do_op` task: set SW\[9:8\] (mode) and SW\[7:0\], wait 3 clocks for the switch synchronizer, then press KEY0 |
| 153–157 | `select` task: put the word number on SW, wait, press KEY1 |
| 159–166 | `show` task = **reading the displays**: wait 3 clocks, then compare all six HEX outputs at once (concatenated into 42 bits) with the expected patterns. Prints `OK` or `ERROR` with a description; `%-40s` left-aligns the description in 40 characters so the log lines up |
| 169–172 | Fill the `s` table with the same patterns as seg7.sv |
| 174–176 | All buttons up (1), all switches down, then press KEY3: **reset** |
| 179–181 | **Step 1**: select word 3, write A5. Expect HEX `3 – A 5 A 5`: word 3, scrubber off, stored A5, switches A5 |
| 182–183 | Switch to mode 01 with bit 5 on the switches, **without** pressing KEY0: expect `3 5 1 4 4 E`, the codeword 144E |
| 184–185 | Press KEY0 to flip bit 5. LEDR7-0 show the raw stored data, which must now be A7 |
| 186–188 | Mode 10, read: expect `3 – 5 5 A 5` (syndrome 5, position 5, corrected A5), LEDR8 on and LEDR9 off |
| 191–194 | **Step 2**: flip bit 10 as well and read: LEDR9 on, LEDR8 off, HEX1 blanked |
| 197–199 | **Step 3**: select word 7, write 3C, flip bit 2 (p2, a check bit) |
| 200–202 | KEY2 on, wait for one full sweep (16 words × 20 clocks + margin), KEY2 off |
| 203–204 | Read word 7: expect `7 – 0 – 3 C` (syndrome 0, no position, data 3C), proving the scrubber repaired it |
| 205–206 | Read word 3: still UE, because the scrubber must not touch a double error |
| 207–208 | Mode 11: expect stats `01 02 01`. CE 1 (step 1), UE 2 (steps 2 and 3), repaired 1 (word 7) |
| 211–212 | **Step 4**: KEY0 in mode 11 clears everything: `00 00 00` |
| 214–216 | PASS if no errors, then pause |

This testbench covers the whole chain: synchronizer → userInput → FSM → datapath → display logic → seg7. If it passes, the board should behave exactly like the board test table.



## 第一部分：板子上每個元件的功能

### SW（撥桿開關）

|開關|功能|
|---|---|
|**SW9、SW8**|**選模式**，決定按 KEY0 時要做什麼，以及 HEX 顯示什麼|
|**SW7–SW0**|依模式不同有不同意思（見下表）|
|**SW3–SW0**|另外也是**要選的 word 編號**（按 KEY1 時讀進去）|

|SW9 SW8|模式|按 KEY0 會做什麼|SW7–0 的意思|
|---|---|---|---|
|下 下（00）|**WRITE 寫入**|把 SW7–0 編碼後存進目前的 word|要寫入的 8 bit 資料|
|下 上（01）|**INJECT 注入錯誤**|把目前 word 的某一個 bit 翻轉|SW3–0 = 要翻第幾個 bit（0–12）|
|上 下（10）|**READ 讀取**|讀出目前的 word，解碼並更正|不用|
|上 上（11）|**STATS 統計**|把三個計數器清成 0|不用|

撥開關本身**不會改變記憶體**，只會改變顯示。只有按 KEY0 才會真的執行。

### KEY（按鈕）

|按鈕|功能|說明|
|---|---|---|
|**KEY0**|執行|做 SW9–8 選的那個操作。按多久都只做一次（userInput 只送一個脈衝）|
|**KEY1**|選 word|把 SW3–0 的值存成「目前的 word」（0–15），顯示在 HEX5|
|**KEY2**|scrubber 開／關|每按一次切換一次；開著時每秒自動檢查 4 個 word|
|**KEY3**|Reset|16 個 word 全部清成 0、計數器清 0、scrubber 關掉、目前 word 回到 0|

### HEX（七段顯示器）

每個模式顯示的內容不一樣：

|模式|HEX5|HEX4|HEX3|HEX2|HEX1|HEX0|
|---|---|---|---|---|---|---|
|**00 寫入**|目前 word|S（scrubber 開著才亮，顯示 5）|記憶體裡現在的資料（高 4 bit）|記憶體裡現在的資料（低 4 bit）|SW7–4|SW3–0|
|**01 注入**|目前 word|**要翻的 bit**（SW3–0）|codeword bit 12|codeword bit 11–8|codeword bit 7–4|codeword bit 3–0|
|**10 讀取**|目前 word|S|**syndrome**|**錯誤位置**（只有 CE 才亮）|更正後資料（高 4 bit，UE 時不亮）|更正後資料（低 4 bit，UE 時不亮）|
|**11 統計**|CE 次數（高）|CE 次數（低）|UE 次數（高）|UE 次數（低）|修復數（高）|修復數（低）|

幾個名詞：

- **codeword**：13 bit，就是記憶體裡實際存的東西，是 8 bit 資料加上 5 個檢查位元。
- **syndrome**：解碼器算出的錯誤位置編號。0 代表沒錯，或錯的是 p0。
- **錯誤位置**：要翻回來的那個 bit。

### LEDR（紅色 LED）

|LED|模式 00、01、10|模式 11|
|---|---|---|
|**LEDR9**|上一次讀取是 **UE**（2 個 bit 錯，修不了）|同左|
|**LEDR8**|上一次讀取是 **CE**（1 個 bit 錯，已更正）|同左|
|LEDR7|記憶體裡存的資料 bit 7|scrubber 開著|
|LEDR6–4|存的資料 bit 6–4|不亮|
|LEDR3–0|存的資料 bit 3–0|scrubber 下一個要檢查的 word（0–15，會一直數）|

**最重要的觀念：LEDR7–0 和 HEX1–0 的差別**

||LEDR7–0|HEX1–0（讀取模式）|
|---|---|---|
|顯示|記憶體裡**實際存的**資料，**沒更正**|解碼器**更正後**的資料|
|有錯時|看得到錯的值（例如 A7）|顯示正確的值（例如 A5）|
|用途|證明錯誤**真的在記憶體裡**|證明 ECC **修好了**|

LEDR9、LEDR8 是暫存器，會一直亮到**下一次讀取**才更新。所以切換 word 或模式時它們還亮著，這是正常的。