# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Critical Implementation Notes

Hidden tests primarily evaluate:

- Correct valid/busy/out_valid handshake behavior
- Exact operation latency
- IEEE-754 correctness for normal FP32 values
- Round-to-nearest-even behavior
- Synthesizable RTL

The design must compile and run under Icarus Verilog.

Avoid:
- Mixing blocking and non-blocking assignments to the same register
- Declaring temporary variables inside procedural case branches
- Variable-width part-select expressions that are not synthesizable
- Simulation-only constructs

---
## Hidden Test Priorities

The majority of hidden tests evaluate:

1. Correct FP32 multiplication for normal finite numbers
2. Correct hidden-bit insertion
3. Correct exponent computation
4. Correct normalization
5. Correct round-to-nearest-even (RNE)
6. Correct 7-cycle latency
7. Correct valid/busy/out_valid handshake

Hidden tests primarily use normal finite FP32 operands:
- exponent ∈ [1..254]
- no NaN inputs
- no Infinity inputs

Special values (NaN, Infinity, subnormal inputs) are a secondary concern.


## Scoring Guidance

Hidden tests heavily favor:
- correct arithmetic
- correct latency
- correct handshake

A simple implementation that passes all normal finite-number tests will score higher than a complex implementation with partial NaN/Infinity/subnormal support.
---


## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (posedge) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

### Handshake contract
- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy==1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

### Handshake Example

Cycle 10:
valid=1, busy=0 → operation starts

Cycle 12:
valid=1, busy=1 → request ignored

Cycle 17:
out_valid=1 → result available

While busy=1, new valid requests must not overwrite the current operation.

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

A safe expectation for system-level timing is:
- **`out_valid` occurs 7 clock cycles after the start edge** (the clock edge where `valid` was sampled when idle).

### Latency Example

If valid is sampled high while busy=0 at cycle N:

Cycle N      : request accepted
Cycle N+1    : stage 1
Cycle N+2    : stage 2
Cycle N+3    : stage 3
Cycle N+4    : stage 4
Cycle N+5    : stage 5
Cycle N+6    : stage 6
Cycle N+7    : stage 7 and out_valid asserted

out_valid must pulse for exactly one cycle.

### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

### Latency Self-Test

If valid is accepted at cycle 10:

cycle 10 -> accept request
cycle 11 -> stage1
cycle 12 -> stage2
cycle 13 -> stage3
cycle 14 -> stage4
cycle 15 -> stage5
cycle 16 -> stage6
cycle 17 -> stage7 + out_valid

If out_valid appears at cycle 16 or cycle 18, the implementation is incorrect.

---
## Non-Negotiable Behavioral Requirements
The following requirements must always hold:
- valid is ignored while busy=1
- a_r and b_r must not be overwritten while busy=1
- out_valid must pulse for exactly one cycle
- z must only be updated when the operation completes
- operation latency must remain fixed at 7 cycles


## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals:
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE

---
## Worked Example
Inputs:

2.0 = 0x40000000
2.0 = 0x40000000

Expected output:
4.0 = 0x40800000

Reasoning:
sign:
0 XOR 0 = 0

unbiased exponent:
1 + 1 = 2

biased exponent:
2 + 127 = 129

fraction:
0

packed result:
0x40800000

---
## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- For normal operation:
  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 1`
- Mantissa product: `product = a_m * b_m * 4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
This stage performs:
1. **Underflow alignment** toward exponent -126:
   - Computes shift amount `sh = (-126 - z_e)` when `z_e < -126`.
   - Shifts mantissa right and accumulates shifted-out bits into sticky.
2. **Normalize** if MSB missing:
   - Left-shifts mantissa while adjusting exponent, carrying guard into LSB.
3. **RNE rounding**:
   - If `G == 1` and `(R || S || LSB)` then increment mantissa.
   - Handles carry-out from rounding:
     - If rounding overflows mantissa, set mantissa to 0x800000 and increment exponent.

### Stage 7 — Pack
- For normal path:
  - Pack sign, biased exponent, fraction.
  - If exponent indicates overflow -> output INF.
  - If exponent indicates exact denorm boundary -> force exponent field to 0 (denormal/zero representation).
- Asserts `out_valid` for one cycle and clears `busy`.

---

## Supported Input Space

Hidden tests primarily use normal finite FP32 values:

- exponent ∈ [1..254]
- no NaN inputs
- no Infinity inputs

Correctness on normal finite numbers is mandatory.
Special-value support must not break correctness on normal finite numbers.
---

## Required Self-Checks

Before considering the implementation complete, verify:

Arithmetic:
- 1.0 × 1.0
- 2.0 × 2.0
- 3.0 × 5.0
- 0.5 × 0.5
- negative × positive
- overflow case

Normalization:
- a case requiring normalization
- a case not requiring normalization

Rounding:
- a case where guard=1 but no rounding occurs
- a case where guard=1 and rounding occurs

Handshake:
- valid asserted while busy=1
- exact 7-cycle latency

Use directed tests before relying on random testing.


## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

### Verification Guidance
Before considering the implementation complete, verify:

- Exact latency from valid to out_valid
- Positive and negative operands
- Overflow cases
- Rounding behavior near mantissa boundaries
- Handshake behavior when valid is asserted while busy=1

Passing a few arithmetic examples is not sufficient.

---

## Implementation Priority
Implement in the following order:
Priority 1:
- normal finite FP32 multiplication
- hidden-bit insertion
- exponent arithmetic
- normalization
- RNE rounding
- handshake behavior

Priority 2:
- zero handling

Priority 3:
- Infinity and NaN handling

A solution that is correct for normal finite numbers is preferred over a partially-correct implementation that attempts full IEEE-754 coverage.


## Implementation Strategy Guidance
Before writing RTL:
1. Implement and verify normal finite-number multiplication first.
2. Treat normalized inputs as having an implicit leading 1 in the mantissa.
3. Normalize the mantissa product before final exponent computation.
4. Adjust the exponent after normalization when required.
5. Verify latency and handshake behavior.
6. Only then add special-case support.

Prioritize arithmetic correctness before introducing complexity.


## Mandatory Verification
The solution is NOT complete until a testbench has been written and executed.
The testbench must verify:

1. 1.0 × 1.0
2. 2.0 × 2.0
3. 3.0 × 5.0
4. negative × positive
5. overflow case
6. handshake rejection while busy
7. exact 7-cycle latency

The testbench must:
- pulse valid for exactly one cycle
- wait for out_valid
- compare z against expected values
- report PASS/FAIL for every test

Compile and run the testbench under Icarus Verilog before considering the implementation complete.


## Icarus Verilog Compatibility
Do NOT:
- Declare registers inside case branches
- Declare temporary variables inside procedural blocks
- Use variable-width part selects
- Use SystemVerilog features unsupported by Icarus
- Mix blocking and non-blocking assignments on the same register

Declare all temporary registers at module scope.

The implementation must compile cleanly under Icarus Verilog before submission.


## Common Sources of Failure
1. Missing hidden-bit insertion.
2. Incorrect exponent bias handling.
3. Incorrect exponent correction after normalization.
4. Incorrect mantissa extraction.
5. Incorrect RNE implementation.
6. Handshake accepting inputs while busy.
7. Incorrect out_valid pulse width.
8. Off-by-one latency errors.
9. Updating z before the final stage.
10. Forgetting to register inputs when valid is accepted.