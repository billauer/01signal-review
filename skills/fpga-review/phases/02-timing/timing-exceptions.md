# Phase 2 — Timing: timing exceptions

Timing exceptions tell the tool to analyze a path differently or not at all.
Each one is a claim about the design. Check that the logic makes the claim
true. An exception that is too broad hides real violations, which cause
mysterious malfunctions in hardware, and nothing in the reports shows it
[01S6] [01S12].

## General

- Internal paths should need only clock period constraints, and clock groups
  or false paths where a CDC exists in the logic.
  Every other exception needs a specific reason; flag unexplained ones [01S12].
- **Every `-from` / `-to` / `-through` must match something.** A `get_*` that
  matches nothing silently yields an exception covering no paths, and the
  tool may only warn [01S6]. See [constraint-objects.md](constraint-objects.md).
- Multiple arguments are ANDed: each added argument narrows the match [01S6].
- **Precedence (Vivado)**, highest first: `set_clock_groups`,
  `set_false_path`, `set_max_delay`/`set_min_delay`, `set_multicycle_path`,
  then clock period. Within the same priority, the more specific constraint
  wins, otherwise the last one does. Other tools have their own rules; don't
  assume this order [01S6]. Consequences:
  - Flag paths matched by more than one exception, in any tool. With Vivado,
    check that the result of the overlap makes sense in relation to the logic,
    taking the precedence rule mentioned above into consideration.
    With other tools, flag overlaps as Comment, advising the reader to verify
    that the tool interprets the overlap consistently with the logic.
  - In Vivado, a false path or clock group that accidentally matches a path
    silently cancels any max-delay or multicycle on it. For other tools,
    flag conflicts as Comment, advising the reader to verify
    that the tool resolves the conflict consistently with the logic.
- `-reset_path` on `set_max_delay` (Vivado) overrides earlier exceptions on
  its paths [01S6]. Check that it's used when necessary, and not used otherwise.
- Exceptions with `-from`/`-to` on non-sequential objects (pins, nets,
  combinational cells) cause path segmentation in Vivado, with a critical
  warning [01S6]. Flag them. Other tools treat such start/end points in their own
  way; flag as Comment, saying only that this should be looked into.

## Clock groups

- Clock group constraints must match the logic. A CDC synchronizer implies
  unrelated clocks; no synchronizer implies related clocks (see
  [../03-rtl/cdc.md](../03-rtl/cdc.md), section A). Tools can't infer this
  [01S5].
- **Preferred**: one `set_clock_groups -asynchronous` that divides all
  clocks into groups, placed after all `create_clock` commands. Flag several
  overlapping `set_clock_groups` commands as Comment, suggesting that they be
  merged into one, but without example code [01S5].
- **Mixed relations**: the same clock pair treated as related by some logic
  and unrelated by other logic can't be expressed with clock groups. Question
  the logic before accepting per-path exceptions. They are acceptable only on
  paths that go through a synchronizer, even though one isn't strictly needed there.
  For example, it's correct to have false paths on the synchronizer inside a
  dual-clock FIFO between related clocks to avoid unnecessary constraining
  [01S5].
- Groups reference clocks that actually exist. A group built from
  `get_clocks -of_objects [get_pins ...]` on a renamed pin is **silently
  empty**, and then nothing is cut [01S10].
- Clocks grouped as asynchronous that the logic treats as related
  (usually from the same PLL): Flag as Notification. Give all examples from the
  code where this happens, or up to 10 of the most serious ones [01S5].
- In Vivado, clock groups override the max-delay constraints that bound
  multi-bit CDC skew (see [../03-rtl/cdc.md](../03-rtl/cdc.md), F2–F3).
  The metastability recovery constraint (F1) is on paths within the
  destination clock, so clock groups don't affect it; a false path that
  starts at the guards would. Where the skew bound matters, use `-reset_path`
  with Vivado [01S6] [01S20].

## False paths

- A false path is correct only when the path has **no timing requirement at
  all**: crossings into a synchronizer, static or quasi-static signals,
  meaningless I/O (LEDs, DIP switches). Each is justified by a synchronizer, a
  static signal, or a documented quasi-static configuration [01S6].
- **Per-pair false paths between clocks** (`set_false_path -from [get_clocks
  a] -to [get_clocks b]`) need one command per direction for every pair and
  are easy to get wrong. Flag them; recommend clock groups [01S5].
- **Overbroad patterns**:
  - `set_false_path -to [get_clocks clk]` (no `-from`) also cuts paths that
    start and end on `clk` [01S5].
- For crossings into a synchronizer, a false path **to the first guard
  register** (`set_false_path -to [get_cells *_metaguard_reg]`) is simple and
  correct [01S5]. Make a minor Comment suggesting a check that the cell name matches
  the netlist (synthesis may add suffixes such as `_reg__0`).
- False paths on asynchronous reset nets leave the **release** of a reset that is
  used asynchronously without recovery/removal analysis (see
  [../03-rtl/resets.md](../03-rtl/resets.md), A4). That is correct only if the
  reset is released while clocks are stopped, or is re-synchronized at the
  destination [01S21].

## Max/min delay

- In Vivado, `set_max_delay`/`set_min_delay` without `-datapath_only` adjust
  the requirement between clock edges, including clock path delays. They are
  valid between synchronous elements of related clocks, and for tighter
  requirements (e.g. paths from metastability guards). A `set_max_delay` that
  just replaces what a clock period or multicycle constraint should express
  suggests that the wrong command was used [01S6].
- **`-datapath_only`** (Vivado) ignores clock paths, and assumes that the
  clock delay is zero on both sides. It is the right choice
  between unrelated clocks, but it **enforces no hold requirement**, and
  `set_min_delay` on the same paths is ignored. Flag `-datapath_only` with
  `set_min_delay` as a Notification, and check that hold violations are covered by the
  design (e.g. a synchronizer) [01S6].
- A `set_max_delay` without `-datapath_only` between unrelated clocks
  includes meaningless clock skew (Vivado) [01S6]. Flag as Notification.

## Multicycle paths

[01S7]

- **Generally discouraged.** For logic that runs at a fraction of a clock,
  prefer a separate clock from the same PLL: the tool derives correct
  constraints automatically. Accept multicycle exceptions when clocking
  resources are scarce.
- **Eligibility**: a path may be relaxed only if the clock enable controls
  **both** endpoints, so that neither changes while the enable is low. Paths
  from enabled registers to ungated ones are ordinary paths: relaxing them is
  wrong, flag as Notification and make a list of all such paths, or up to 10.
- **Setup N, hold N-1** (standard SDC behaviour, and so in Vivado and
  Quartus; check other tools): `-setup` also moves the hold requirement, so a
  `-hold` of N-1 is required on the same paths. Flag a setup multicycle without
  the matching hold. N here is the division ratio created by the clock enable.
- **Selecting the enabled registers**:
  - Select the endpoints in the enable's fan-out (e.g. Vivado
    `all_fanout -endpoints_only -only_cells -flat [get_nets en]`, Quartus
    `get_fanouts en`), or better, the clock's synchronous elements filtered by
    the module's hierarchy.
  - `get_cells -of_objects [get_nets en]` is wrong: it includes the enable
    register itself and misses logic where the enable was absorbed into a LUT.
  - **Extra cells in the list are the dangerous error**: they relax paths
    that need full timing, and nothing reports it.
  - Name-based lists fail silently (empty list) when synthesis renames the
    enable net, and miss tool-made replicas.
- **RTL conditions** for the exception to be valid:
  - The enable doesn't feed its own next state (`en <= !en`). Use a separate
    `pre_en` register, protected from removal.
  - The enable is used only as a clock enable. If it also drives general
    logic, registers that change every cycle end up in the list. Use a
    separate, merge-protected replica for that logic.
  - Enable replicas are protected from merging, and after an asynchronous
    reset all replicas load from the same source, so they can't diverge (see
    [../03-rtl/resets.md](../03-rtl/resets.md), A2).
  - A synchronous reset on the enabled registers is also qualified by the
    enable (`if (en && reset)`). Holding the reset for several cycles
    isn't enough on its own, as the clock enable may be asserted less than N-1
    clocks after the reset is deasserted.

