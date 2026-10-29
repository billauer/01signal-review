# Phase 2 — Timing: clock constraints, timing coverage and timing reports

## Primary clocks

- Every external clock driving a PLL or used directly to clock logic
  has a `create_clock` or equivalent. Flag unconstrained clocks as Notification.
- **Defined on the port**, with `get_ports`, `get_pins` or equivalent,
  and named after the port or pin,
  e.g. `create_clock -period 4 -name clk [get_ports clk]` [01S4].
- **Referencing a pin**: When a clock is defined on a global buffer output or another internal pin
  (`[get_pins my_bufg/O]`), Vivado uses that pin as the time
  reference and ignores the skew to everything else. Such a clock
  should therefore be treated as unrelated to all other clocks, both in the
  user-defined timing constraints and in the logic.
  Flag deviations from this rule as Notification [01S4].

  Apply this rule to other SDC-based tools too, but consider it
  speculation and flag any findings from it as Speculation.
- **Period agreement**: each `-period` matches any RTL comment or counter that
  assumes a frequency. A clock constrained slower than reality hides
  violations on every derived domain.
- **No overconstraining**: a period shorter than the real one "for extra
  safety": flag as Comment. It makes the tools work harder for nothing. If
  it seemed to fix a problem, the problem is elsewhere [01S2].
- Constraints for clocks that no longer exist (stale, flag as Comment), or
  two `create_clock` commands on the same net: flag as Notification.

## Generated clocks

- **Don't duplicate what the tool derives.** In Vivado, PLL/MMCM outputs
  and feedback clocks are derived automatically. A manual `create_clock`
  there breaks the relationship to the input clock, so the tool computes the
  timing between the clocks wrongly, and needing one is a red flag (Vivado)
  [01S3].
  Other tools differ: some require explicit generated clocks or a derive
  command. Check which applies before flagging either a missing or an extra
  PLL clock constraint.
- Tool-specific derivation:
  - Vivado derives PLL/MMCM outputs automatically from the input clock.
  - Quartus Standard/Lite, and Pro for Arria 10 / Cyclone 10 GX: the SDC
    must contain `derive_pll_clocks` (or explicit `create_generated_clock`
    constraints for the PLL outputs). Without it, paths
    clocked by PLL outputs are timed with the **input clock's period**: they
    don't appear as unconstrained, no critical warning is issued, and the
    reports show comfortable slack [01S28].
    Check the latch edge times of PLL-clocked paths in a report. Quartus
    also needs `derive_clock_uncertainty`.
  - Quartus Pro for Stratix 10, Agilex and later: `derive_pll_clocks` is
    not supported. The SDC files generated with the PLL IP create the
    generated clocks.
- Clock names: later constraints refer to generated clocks by names that
  exist (auto-derived names change when the hierarchy changes; prefer
  `get_clocks -of_objects [get_pins ...]` on a specific pin, see
  [constraint-objects.md](constraint-objects.md)).
- In reports, the clock name comes from the clock object (`-name`, or the
  tool's auto-generated name such as `clk_out1_clk_wiz_1`), not from a net in
  the design. Don't assume a net of that name exists [01S4].

## Uncertainty and jitter

- Input jitter (`set_input_jitter`, the IP's input jitter setting) is
  specified. Without it, Vivado assumes zero jitter on the input clock.
  Zero input jitter is common practice
  but still optimistic: flag as Comment and report very briefly. This is
  very rarely a source of problems when a proper oscillator is used [01S2].

## Check the clocks the tool actually created

From reports, if such are available:
- **Clock summary**: every intended clock exists, with the intended period
  and waveform. Each PLL output and feedback clock has the expected frequency.
  No unexpected clocks. Equal edges at 0 ns in the summary don't mean the
  clocks are aligned on the chip [01S3].
- **"Registers with no clock"** (or "unclocked") messages usually mean a
  missing or wrong clock constraint, not a logic problem [01S4].

## Timing coverage

- **Unconstrained paths should be zero** for every endpoint that is a
  flip-flop or other synchronous element. For this purpose, an unconstrained
  path is any path on which the tools don't enforce timing, and that ends on a
  pin of a synchronous element other than its clock or asynchronous set/reset.
  Clock-buffer-to-clock-resource paths
  are often listed as unconstrained and can be ignored. So can outputs to
  LEDs or other very slow devices, and inputs that are synchronized in RTL
  and much slower than the sampling clock (e.g. a properly sampled I2C
  input). It's recommended to declare false paths on I/O that is deliberately
  unconstrained for this reason, or else it clutters the list of unconstrained paths.
  Make a minor Comment on this if found.
  The default timing report may not include this section [01S12].

  Check every unconstrained path ending inside the FPGA, and verify that some sort of
  CDC exists [01S12]. List all endpoints violating this rule (or up to 10 examples)
  and flag as Notification.

- **Clock relationships as the tool sees them**: there may be a clock-interaction
  report available. Don't use the information in this report directly; rather,
  compare it with your own expectations. If there is a difference, rely on
  your own interpretation, and add a visible remark that your view of the
  clock relationship is uncertain. Expose the uncertainty, don't hide it.
  [01S12]

- **Ignored constraints**: flag every ignored constraint as a Notification.

## Timing results

If timing reports are present: don't use the information in these reports directly;
rather, use it to check against your own working notes.
If there is a difference, rely on
your own interpretation, and add a visible remark that your view on this
is uncertain. Expose the uncertainty, don't hide it.


## Build gates

- Most tools **write a bitstream even when timing fails** (Vivado, Quartus).
  Check whether the flow stops on timing failure. If not, make a friendly
  suggestion to add such a mechanism (for Vivado, e.g. the `showstopper.tcl`
  script included in the Xillybus demo bundles).

