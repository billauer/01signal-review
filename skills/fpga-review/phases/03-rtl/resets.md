# Phase 3 — RTL: resets and power-up

This topic checks that resets and initialization follow the guidelines of the
01signal.com "Resets on FPGA" series ([01S21], [01S22], [01S23]).

The series' main point [01S22]: **a reset only matters if the logic behaves
predictably after the reset is deactivated.** Bringing registers to a known
value is not enough. Releasing the reset asynchronously to the clock, or
running logic on a clock that is not yet stable, causes rare and random
failures. These failures can make an FPGA seem "haunted", and they are very
hard to debug.

## Core principles (what "correct" means here)

- **Decide for each register.** Every register, and every primitive with a
  reset input, needs its own decision [01S22]. There are four options:
  1. Unknown initial value (no reset, no init). Suits data paths, delay lines
     and shift registers whose content is ignored until it is valid.
  2. FPGA initial value from the bitstream (Verilog `initial`, VHDL signal
     default `:= value`, or a primitive's `INIT` parameter). No explicit reset.
  3. Synchronous reset. This is the preferred choice when a reset is needed.
  4. Asynchronous reset. Use where a primitive or IP core explicitly
     requires it, and even then prefer to drive it with the synchronous reset
     signal. Also acceptable if rules in section A are met.
- Some vendor documents recommend asynchronous resets as the default style.
  That is acceptable when each one meets the release rules in section A; a
  design that mixes synchronous and asynchronous resets is fine too if each reset is correct.
- **Always reset state machines explicitly and safely.** The synthesizer may
  re-encode the state variable, for example as one-hot. An illegal encoding may
  then never converge to a legal state, even if the behavioral code looks
  self-recovering [01S22] (see also [fsm.md](fsm.md)).
- **Always reset IP cores and primitives that have reset inputs**. Flag
  as Speculation otherwise [01S22].
- **The release of the reset must be timed.** It must be synchronous to the
  clock of the logic it drives, with timing constraints enforced on that path.
  Otherwise, it must happen while the clock is guaranteed to be stopped, and
  the clock must start glitch-free later [01S21].

## Procedure

1. **Inventory.** Start from the clock and reset tables of Phase 0. Add every
   clocked process and the reset style it uses (none, `initial` or default
   value, sync, async).
2. **Trace each reset to its source.** Follow each reset signal back to its
   origin (pin, PLL lock, IP core output, synchronizer, reset controller,
   software). Determine whether it is synchronous to the clock of every
   flip-flop that uses it, and whether it crosses clock domains.
3. **Run the checklist below** on each process, each reset path and the
   top-level reset and clock module.
4. **Report** findings in the format of SKILL.md, with the additions under
   "Reporting" at the end of this file.

Read the full code paths before reporting. A reset that looks asynchronous may
come from a proper synchronizer in another file. A register without a reset may
be qualified by a valid flag elsewhere.

---

## Checklist

### A. Asynchronous reset hazards

**A1. Raw asynchronous reset with an uncontrolled release.** [01S21] [01S22]
Look for: `always @(posedge clk or negedge rst_n)` / `or posedge rst`, or VHDL
`process(clk, rst)` with `if rst = '1' then ... elsif rising_edge(clk)`.
Then check whether the reset is (a) produced by a reset synchronizer in the same
clock domain and deasserted synchronously, or (b)
released only while that domain's clocks are stopped or gated.
If neither applies, the reset is released asynchronously to `clk`. Some
flip-flops then see the first clock edge and others miss it (recovery/removal
violation). **Flag it**: as Notification if the logic can't tolerate a
missed first edge (see A2), otherwise as Comment.

For other tools, it's unclear whether this path is timed. Mention that.

Regarding (a): since the asynchronous reset's release is timed, it effectively
acts as a synchronous reset. Unless the register must be reset even without
a clock, replacing it with a synchronous reset is better. It's clearer, and
it gives the synthesizer the freedom to use the register's reset input
for any kind of logic, or to merge the synchronous reset into the
regular logic and use only the D input.

**A2. Logic that cannot tolerate a missed first clock after an async release.** [01S21]
Combined with A1, flag these as Notification:
- A state machine that leaves its reset state **unconditionally on the first
  clock**. Under one-hot encoding, a partial transition gives two hot bits or
  none, and the FSM may lock up. The synthesizer can usually choose one-hot
  encoding for any FSM.
- A register or vector whose reset value is followed by a next value in which
  **many bits change at once**, for example a down-counter reset to 0 that
  becomes 0xFFFF. A partial update gives a random value.
- **Free-running toggles and clock enables** such as `en <= ~en` with high
  fan-out. The synthesizer may duplicate the register. Each copy feeds back on
  itself, so the copies can stay out of phase forever. The fix is a single
  `pre_en` source register (with a don't-touch attribute) that feeds all
  copies of `en`. Its reset value must be the "inactive" value.
Flag as Comment only: an up-counter reset to 0 (only bit 0 changes on the
first clock), or an FSM that waits in its reset state for several cycles.

Make a list in the report of all (or up to 10) A1+A2 cases flagged as Notification,
showing the HDL code involved, and explain how this can cause a failure. Prioritize
issues that can have visible results; the reader takes this more seriously if
the description of the consequences matches a problem already experienced.
However, the possible visible results are speculations, so describe them
accordingly: with questions and a tentative tone.

**A3. One asynchronous reset shared across clock domains.** [01S23] If a reset is
synchronized to one clock but used asynchronously in another domain, its
release is untimed or forms an unhandled CDC. Each clock, or group of related
clocks, needs **its own** synchronized reset. Flag every async-reset process
whose clock differs from the clock of the reset's synchronizer as an
unsynchronized CDC.

**A4. Timing on reset-release paths.** [01S21] [01S1] Suppose a synchronized asynchronous reset
drives flip-flops' async clear/preset pins. Then the paths from the synchronizer
output to those pins must be timed (recovery/removal checks). Some
tools ignore paths that end at async pins by default. If constraint files are
available, check for `set_false_path` or similar that removes or relaxes
these paths (see
[../02-timing/timing-exceptions.md](../02-timing/timing-exceptions.md)).

It's a common mistake to relax the timing on the reset because
it's a high-fan-out signal with a large delay, thinking it doesn't matter
because the reset pulse is long enough. The mistake is not realizing that
the reset's release timing is what actually matters.

Without constraint files or timing reports, add a note that this must be
verified in the timing report, except for Vivado, which times such paths
by default.

Conversely, a design that resets asynchronously **with the clocks gated off**
should not time these paths.

**A5. Reset synchronizer structure.** [01S22] A correct synchronized asynchronous reset
looks like this:
```verilog
always @(posedge clk or negedge ext_rst_n)
  if (!ext_rst_n) {rst_n, pre2, pre1} <= 3'b000;
  else            {rst_n, pre2, pre1} <= {pre2, pre1, 1'b1};
```
Check that:
- There are at least 2 flip-flop stages, to resolve metastability on release.
- All stages are cleared asynchronously by the external reset.
- The constant `1` (inactive) shifts in from the first stage.
- The synchronizer is clocked by the **same clock** as the logic it resets.
- The external reset pulse is guaranteed long and glitch-free (driven by
  logic or a reset IC, **not** a raw pushbutton or noisy pin). If this cannot
  be guaranteed, the external reset must be debounced first, and then used
  as `ext_rst_n` above.

**A6. Synchronized-async reset used as a plain synchronous reset.** [01S22] Code that
uses the output of an async-assert synchronizer (as in A5) in a
`posedge clk`-only process is fine. However, its assertion is still
asynchronous and untimed, so logic may misbehave just before the reset takes
effect. Recommend a fully synchronous reset synchronizer for that use.
If clk is guaranteed to be free-running, it's better to generate the reset
with a plain CDC from the external reset signal (after debouncing, if required).
This way, one reset is used for all logic in that clock domain.
Flag as Comment.

**A7. Asynchronous reset combined with a non-standard pattern.** Look for
`always @(posedge clk or negedge rst_n)` where some registers are assigned
outside the reset if/else. For example, `reg2 <= ...` sits at the top level of
the block and only `reg1` is reset inside `if (!rst_n)`. This code asks for a
flip-flop with two clocks, which cannot be built. Synthesizers may silently
build something different from the code. **Flag as Notification.**
Registers that should not be reset must go in a separate `always` block
when asynchronous resets are used.

**A8. Resets block optimization.** [01S22] Asynchronous resets (and synchronous ones
too) on inferred shift registers, SRLs, or RAM arrays prevent the synthesizer
from using those primitives. The same applies to inferred DSP/RAM output
registers, which often accept only a synchronous reset. Flag resets on delay
lines or memory arrays that don't need them as Comment.

### B. Synchronous reset issues

**B1. Accidental freeze (reset acting as clock enable).** [01S22] Look for a process
that uses the standard `if (reset) ... else ...` pattern, sync or async, where
a register assigned in the `else` branch is **missing from the reset branch**.
That register then holds its value while reset is active, which in effect makes
the reset a clock enable for it. This adds logic, and it is usually a bug or an
oversight. VHDL equivalent: a signal assigned in the `elsif rising_edge(clk)` /
`else` branch but not in the `if rst = '1'` branch.
For each such always/process, report as Comment and show the relevant code
snippet (up to 20). This is very rarely intentional.

Acceptable fix for synchronous resets: put the reset **last** so that it
overrides the earlier assignments.
```verilog
always @(posedge clk) begin
  reg1 <= ...; reg2 <= ...; reg3 <= ...;
  if (reset) begin reg1 <= 0; reg2 <= 0; end   // reg3 intentionally not reset
end
```
This pattern is valid and recommended by the source for synchronous resets
only. **Do not flag it as wrong.** Don't use it with asynchronous resets
(see A7).

**B2. Reset fan-out.** [01S22] A single global synchronous reset (or synchronized
asynchronous reset) that drives a very large number of flip-flops across the
design makes timing closure hard. The recommended fix is a **local reset
register** per significant module (delayed by one clock), with roughly 50–100
loads per copy. It must be protected from merging:
```verilog
(* dont_touch = "true" *) reg local_reset;   // Vivado; Quartus: dont_merge / preserve
always @(posedge clk) local_reset <= reset;
```
- Check that local reset copies have a don't-touch or similar attribute.
  Without it, the synthesizer merges identical registers across modules.
- Relying only on a synthesizer max-fanout attribute is less preferred, because
  the duplicates get scattered and routing gets long.
- Each extra copying stage delays the reset by one clock. Check that the reset
  pulse is long enough to cover the deepest copy chain (see D3).
- Preferably, the reset to all logic in the design has the same delay,
  so it comes out of reset at the same time. If a large part of the logic
  is reset with a certain delay, and a minor part has a different delay,
  flag that as Notification and point out the parts of the logic whose
  reset is released out of step.
- If this local reset register trick isn't used, suggest it as a Comment
  unless the reset's fanout is small.

**B3. Expectation that the sync-reset pattern maps to the flip-flop's SR pin.** [01S22]
Synthesizers treat a synchronous reset as just another input to the logic
function. Don't flag code for "not using the dedicated reset pin", and don't
accept comments that rely on that mapping. Tools have attributes for explicit
control if it is needed (e.g. Vivado `DIRECT_RESET` and `EXTRACT_RESET`).

### C. Initial values and unreset registers

**C1. Register without reset or initial value.** [01S22] This is acceptable **only if**
its value is provably ignored until a valid value has been written, or if it
is guaranteed to remain in its initial value after the first clock cycle
by virtue of its assigned next-clock value.
Examples: a data-path pipeline stage qualified by a valid flag that is itself reset, a
delay line, or a register that is always written before it is read. Flag
any other unreset register as Notification, in particular: FSM state,
valid/busy flags, counters that control sequencing, handshakes, and clock enables.

**C2. Reliance on initial values on PLL-generated or unstable clocks.** [01S23]
Rule C1 is correct only when the clock is stable when the FPGA powers up.
Assume an external oscillator is, per "Stable at startup" in
[../02-timing/clock-generation.md](../02-timing/clock-generation.md).
If the clock is unstable immediately after powerup, an unreset register
is acceptable **only when one of these** holds:
- The register is held idle by a clock enable until the clock is stable.
- The register has no reason to change value until the clock is stable.
- Its value is provably ignored until a valid value has been written.
Otherwise, flag it as Notification.

**C3. Prefer `initial` / VHDL default values over vendor primitives with
`INIT`** [01S22] for setting power-up values, unless the synthesizer doesn't support
them. This is a style and portability note only: flag as minor Comment.

**C4. Synthesizer-dependent initial values.** Code such as
`always @(posedge clk) val <= 1;` may be synthesized as a constant or as a
register that starts at 0. Treat its first-cycle value as unknown. Flag it as Comment
if other logic depends on that first cycle.

**C7. Registers whose power-up value matters outside the FPGA.** Check every
register that drives something whose state matters from the moment
configuration ends: resets held to external chips, output enables, chip
selects, power enables, lock/valid flags read by other logic. Each must have
the correct value at power-up (initial value where supported, or reset that is
active from configuration on, per D8).

### D. Reset controller / power-up sequencing (top-level `clkrst`-type module)

**D1. Centralized reset and clock module.** [01S23] Resets and clocks are preferably
generated in one place, for example a `clkrst` module, and not improvised per
module. Several reset sources (PCIe reset, IP core resets, pushbutton,
software, watchdog) may need a defined system-wide response. Suggest this as
a Comment if the reset logic is scattered across the entire design.

**D2. Hold logic in reset until PLLs lock.** [01S23] Logic clocked by a PLL output must
be held in reset, or clock-disabled, until that PLL's `locked` output is
asserted. Flag:
- Any PLL/MMCM output clock domain whose reset doesn't depend on `locked`.
- A `locked` signal used without synchronizing it to the controller's clock (it
  is asynchronous).
- Designs where a later loss of lock leaves logic running on a bad clock: the
  logic should be held in reset again when `locked` drops.

**D3. Reset stretching and debounce.** [01S23] Each reset request, even one lasting a
single cycle, should produce a well-defined, **long** master reset pulse.
Typically a counter is cleared by any reset source and the master reset stays
active until the counter saturates. Counting 10–100 ms is preferred on resets
coming from a pushbutton. For resets that are not expected to wobble, a few
clock cycles are enough.
Flag:
- Reset sources ORed directly into the master reset, with no stretching.
- A pulse too short to cover the multi-stage local-reset distribution (B2).
- A short counter (for example 31 cycles) combined with a pushbutton input that
  has no debouncer. With a counter of 10 ms or more, a 2-flip-flop synchronizer
  on the button is enough.

**D4. Reset-source inputs are synchronized.** [01S23] Every asynchronous reset source
must pass through at least a 2-flip-flop synchronizer into the controller's
clock domain before it is used. This includes the button pin, PLL `locked`,
and resets from other clock domains.

**D5. Wake-up clock choice.** [01S23] The reset controller's clock should be one of:
- An external reference clock known to be stable when the FPGA wakes up. This
  is preferred. Note that the board's power-up sequence has to guarantee it
  (oscillator start-up time vs. configuration time). Assume it does, and
  mention the assumption as described under "Stable at startup" in
  [../02-timing/clock-generation.md](../02-timing/clock-generation.md).
- A PLL output, but then with the safeguards in D6.
- The configuration or internal oscillator (for example AMD `STARTUPE2`
  `CFGMCLK`, the Intel internal oscillator IP on families that have one).
  This is a last resort, and it should be constrained to a higher frequency
  than its nominal one.

**D6. Controller robustness when clocked by an unlocked PLL.** [01S23] If the reset
controller's clock comes from a PLL, the controller must work even while
that clock is above its nominal frequency: the reset controller should be held in reset
while the PLL is unlocked, and emit an asserted reset signal during this time.

Initial values along the chain from `!locked` through the synchronizer to
resetting the controller must all be set so that the controller is held
in reset and emits an asserted reset from the beginning.

Flag a controller otherwise as Notification.

**D8. Guaranteed power-up reset.** [01S23] Some reset source must be active right after
configuration, even in a design without PLLs or with PLLs that are already
locked at wake-up. The recommended source is a wake-up shift register with
`initial` value 0 that shifts in `1`s, and whose MSB drives
`rst_src_shreg = !wakeup_shift[MSB]`, with `initial rst_src_shreg = 1`. Flag
controllers that depend only on an external reset or a PLL lock to produce
the first reset.

**D9. Per-domain reset distribution.** [01S23] The master reset, if one exists,
must cross into each application clock domain through its own synchronizer
of at least 2 stages before it is used as that domain's synchronous reset.
Flag any direct use of the master reset in a different clock domain as a
CDC without synchronizers.

**D10. Local recurring resets** [01S23] are legitimate, for example resetting
per-frame video logic at each frame start. Don't flag these as a misuse of the
system reset. Just check that they are synchronous and that they cover every
register that needs one.

---

## Detection hints

| Pattern | Verilog | VHDL |
|---|---|---|
| Async reset process | `always @(posedge clk or (neg\|pos)edge rst)` | `process(clk, rst)` + `if rst ... elsif rising_edge(clk)` |
| Sync reset process | `always @(posedge clk)` + `if (rst)` | `process(clk)` + `if rising_edge(clk) then if rst = '1'` |
| Initial value | `initial x = v;`, `reg x = v;` | `signal x : t := v;` |
| PLL lock | `locked`, `LOCKED`, `pll_locked`, `mmcm_locked` | same |
| Don't-touch attribute | `(* dont_touch = "true" *)`, `(* dont_merge *)`, `(* keep *)`, `(* preserve *)` | `attribute dont_touch of x : signal is "true";` (attribute name per tool: `preserve`, ...) |
| Timing exceptions | `set_false_path`, `set_max_delay -datapath_only`, `set_clock_groups` on reset nets | — |

Useful searches:
- `grep -rnE "or (neg|pos)edge" --include='*.v' --include='*.sv' .`
- `grep -rniE "process *\(.*,.*\)" --include='*.vhd' --include='*.vhdl' .`
- `grep -rniE "locked|rst|reset" --include='*.xdc' --include='*.sdc' --include='*.qsf' .`

## Reporting

Use the finding format and levels of SKILL.md. In addition, for each reset
finding:

- Name the signal(s) and the clock domain.
- Give a **concrete failure scenario**, e.g. "reset released 0.2 ns before clk
  edge → `hot_state_0` misses the edge, `hot_state_1` doesn't → both 1, FSM
  stuck".

In the architecture summary (in the report's details section), describe the
reset architecture as found: reset sources, controller, the domains covered
and how each is released. Note anything that needs checking outside the code
(timing reports, board power-up sequence, IP documentation). The reset risks
themselves go with the other findings, not here.
