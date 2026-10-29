# Phase 2 — Timing: clock sources, PLLs and derived clocks

## External clock sources

- Each external clock enters on a **clock-capable pin** (dedicated clock input)
  suitable for the buffer/PLL it drives. A clock on a general-purpose pin
  either fails placement or is routed on general fabric with large skew.
  This matters only if the clock needs a phase relationship with inputs or
  outputs, or if it drives the PLL of an MGT, in which case it must be one
  of the clock inputs dedicated to the MGT.
  If no input or output is synchronous to a non-MGT clock, routing it on
  general fabric is acceptable: flag as Comment at most.
- Clocks from a processor, a peripheral chip, or another board (rather than
  a dedicated oscillator): software or a reset can stop, glitch or
  reprogram them. Logic on such a clock needs the same protection as a
  source-synchronous input clock (PLL with reset on unlock, or 01-signal
  sampling; see io-timing). Prefer a dedicated oscillator [01S27].
- **Stable at startup.** A dedicated oscillator that feeds a PLL or drives
  logic directly is almost always stable long before the FPGA becomes
  active. Assume it is, unless there is an indication otherwise: in
  particular, the FPGA programs or enables the oscillator, or the clock
  comes from another chip as above. So a PLL without a reset, or logic
  that relies on the clock from the first cycle, is not wrong in itself.
  If the logic works only when the clock is stable at startup, make one
  Comment (for all such clocks together) that draws attention to this
  assumption: the clock must be stable when the FPGA becomes active.

## PLLs and MMCMs

- **Frequencies computed from the attributes** (not from comments or IP GUI
  names) match the clock table, the constraints and every RTL assumption.
  Unless there is an explicit description **in this skill set**
  on how to calculate frequencies
  from the clock resources' attributes, perform these calculations as well
  as you can, but mark the resulting frequencies as speculation. **All findings
  based upon a speculated clock frequency are flagged as Speculation**.
  Everything you say based upon a speculated clock frequency is expressed
  with reservation by using a question (e.g. "is the frequency X MHz?") or by
  careful language (e.g. "it appears that the frequency is X MHz").
- PLL and MMCM IPs of AMD have attributes called `CLKIN1_PERIOD` and
  `CLKIN2_PERIOD`, stating the clock period expected by this IP in ns.
  The real input clock must have the same frequency, at least approximately
  (ignore differences below 5%). Flag as Notification otherwise.
  Flag as Speculation if you apply a similar rule to another IP based upon your
  own prior knowledge.
- Related clocks from one PLL preferably have frequencies that are integer
  multiples of each other [01S18]. Non-integer ratios give a very short worst-case
  edge separation (see [../03-rtl/cdc.md](../03-rtl/cdc.md), A2). If that
  separation is too short, treat the clocks as unrelated, with CDC logic and
  matching constraints.
- **Lock**: the `locked` output is used to hold downstream logic in reset or
  clock-gated, and loss of lock is handled (see
  [../03-rtl/resets.md](../03-rtl/resets.md), D2). `locked` is
  asynchronous, so it is synchronized before use [01S23].
- **PLL reset**: driven (not tied off) unless the input clock is stable when
  the FPGA powers up (assume it is, per "Stable at startup" above). It must not be driven from logic clocked by the PLL's
  own output (deadlock) [01S23].
- Cascaded PLLs: jitter accumulates, and the second PLL must stay in reset
  until the first locks. Flag cascaded PLLs as Comment.

## Unaligned related clocks

[01S13]

- Logic running directly on a reference clock alongside PLL outputs derived
  from it (or on two clocks from the same reference but using different PLLs):
  Vivado treats them as related by default, but their edges aren't
  aligned. That shows up as a large clock path skew in the reports, and as
  needlessly tight paths. Taking both clocks from the same PLL is preferred
  (adding a PLL output at the reference frequency if needed); otherwise they
  should be treated as unrelated, with CDC logic and constraints. Using the reference clock
  directly is justified only for logic that must run before the PLL locks.

## Dynamically reconfigured clocks

- Logic downstream must be held in reset during reconfiguration and released only
  after lock. Alternatively, downstream logic can be reset only after reconfiguration
  if its messy behavior during reconfiguration can be proven harmless.

## Clocks from transceivers and hard IP

- Recovered (RX) clocks are only valid after CDR lock, and logic on them is
  held in reset until then.
  Alternatively, downstream logic can be reset only after lock
  if its messy behavior before lock can be proven harmless.
- Hard IP clocks (PCIe user clock, memory controller user clock, processor
  fabric clocks): logic on them follows the IP's reset outputs.

## Clocks generated or gated by logic

- Clocks produced by counters, flip-flops or logic and used as clocks:
  **flag them as Comment**. Prefer a clock enable in the original domain, or a PLL or
  clock-divider buffer. If kept, such a clock needs its own timing
  constraint, and every crossing to the parent clock is treated as CDC.
- Gated clocks built from logic (`assign gclk = clk & en`) cause glitches and
  skew. Flag as Notification. Use the vendor's clock-enable buffer or a clock
  enable on the registers.
- Clocks derived from data (e.g. a strobe from an external device used as a
  clock): using such a clock directly on logic is usually a catastrophic choice
  unless it is guaranteed clean; see the strategies in io-timing, "Clocking
  source-synchronous inputs" [01S15].

## Frequency assumptions in RTL

- Every counter, divider or timeout that assumes a frequency (baud rates,
  I2C/SPI/MDIO clock dividers, delays, refresh timers, watchdogs) is
  computed from the **actual** clock it runs on. Recompute each one, and
  report it as Comment if comments or other documentation suggest a different figure.
- Parameters that hold the clock frequency are set consistently with the
  clock table at every instance. Flag as Comment otherwise.
