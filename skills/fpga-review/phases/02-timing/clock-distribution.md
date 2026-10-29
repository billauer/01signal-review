# Phase 2 — Timing: buffers, routing and clock usage in logic

## Clock buffers and networks

- Clock-enable buffers (e.g. `BUFGCE`) are used
  for clock gating — not LUT logic.
- Clock muxes use the dedicated primitive.

## Clock usage in logic

- **Clock used as data** (a clock net feeding a LUT, a register D input, or
  compared in logic): draw the user's attention to this only if there is an
  obviously better solution. For example, if the purpose is to detect that
  the clock is running, a toggle register in the clock's own domain,
  synchronized into the other domain, does the job.
- **Data used as clock** (any register clocked by something not in the clock
  table): flag as fabric-generated clock (see clock-generation).
- Both clock edges used in one domain (`posedge` and `negedge` registers on the
  same clock): legal, but halves the time budget for paths between edges and
  depends on duty cycle. Check that it is intended.
- Inverted clocks created in logic (e.g. `~clk`) rather than by using the
  opposite clock edge (`negedge`): flag as Comment. If the inverted clock
  drives a primitive, urge the user to check whether the tools implemented
  it as an aligned clock with inverted polarity.

## Clock forwarding off chip

- Clocks forwarded to output pins go through an output DDR register (D1=1,
  D2=0 or the vendor's equivalent), not by routing the clock net to an output buffer.
  This matters in particular when I/O is synchronous to the forwarded clock
  [01S17].
  If the device receiving this clock is unaffected by an arbitrary
  delay on it, make only a brief Comment.

## Related and unrelated clocks

The rules for classifying clock pairs are in
[../03-rtl/cdc.md](../03-rtl/cdc.md), section A.

