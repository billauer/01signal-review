# Phase 3 — RTL: HDL semantics and synthesis/simulation mismatches

Code that compiles can still mean something different from what the designer
intended, or behave differently in simulation and in hardware.

## Verilog

- **Incomplete sensitivity lists**: for a list such as `always @(a or b)`,
  check that the block depends on nothing but `a` and `b`. Where the list
  could easily be wrong or confusing, suggest `always @*`.
- **Signedness** [01S26]: an
  operation is signed only if **all** its operands are signed. One unsigned
  operand makes the whole expression unsigned, so `signed_a * unsigned_b`,
  `signed_a + unsigned_b` and comparisons with unsigned values lose the sign.
  These are unsigned even when built from signed signals:
  - based literals without `s` (`12'd10`, `8'hFF`);
  - bit-selects and part-selects, including a full-range part-select
    `b[15:0]` of a signed `b`;
  - concatenations (`{a, b}`, `{4{b[7]}}`).
- The **target does not affect signedness**: declaring the left-hand side
  `signed` does not make the right-hand expression signed. Signedness is fixed
  by the expression alone, then the result is extended to the target width.
- **Unsigned → signed conversion needs an extra bit**. `$signed(a)` on an
  unsigned `a` reinterprets its MSB as a sign bit, so values ≥ 2^(N-1) become
  negative. Correct: assign `a` to a signed wire one bit wider (zero-extended,
  since `a` is unsigned), or `$signed({1'b0, a})` inline. Not a finding:
  `{1'b0, a}` assigned to a wider signed wire is redundant but harmless.
- **Delay with #**. Ignore when the delay is shorter than the destination
  clock's period. If longer, flag as Notification, as this delay pushes the
  update beyond the next clock cycle. This is possible in simulation, but
  synthesis ignores #-delays altogether.

## Common to all HDLs

- **Asynchronous logic**: latches as storage, SR flip-flops from gates,
  combinatorial outputs used as clocks or asynchronous resets [01S27]. Flag as
  Notification.
- **Unusual coding style**: constructs that are legal but rarely used
  (unusual sensitivity patterns, storage elements updated other than on a
  clock edge or by an asynchronous reset, creative use of language features)
  risk being misinterpreted by the synthesizer, silently. Prefer the
  patterns that vendor-generated HDL and templates use and those mentioned
  explicitly in the RTL part of this skill set;
  proactively flag rare constructs as Speculation and recommend the common form.
- **Patch-on-patch logic**: special cases added to fix symptoms (extra
  conditions, delays, "if this happens, do that"), where correctness can't
  be explained by construction [01S27]. Flag it and recommend a restructure.
