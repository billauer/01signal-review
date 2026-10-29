# Phase 2 — Timing: selecting objects in constraints

A constraint whose object query matches the wrong set of objects is the
quietest kind of error. A query that matches nothing usually produces only a
warning. One that matches too much relaxes paths that need timing.
Read every `get_ports` / `get_cells` / `get_pins` / `get_nets` /
`get_clocks` in the constraints as a logical expression, and ask whether it
will stay correct as the design changes. Does the expression look like
a hack that breaks if the synthesizer renames the signals, or does
it express the reason the constraint is needed? [01S10]

## Names that change

[01S10]

- Synthesis renames registers (e.g. `bar_reg` becomes `bar_reg__0` when another
  register collides), replicates them for fan-out, merges identical ones, and
  flattens hierarchy. Constraints on such names silently stop matching, or
  miss the replicas.
- Suffix conventions (e.g. `_reg`) differ between synthesizers.
- Names generated **inside IP** (internal buffers, internal pins of a clocking
  wizard) change between IP versions. Prefer pins on the IP's instance
  boundary when possible.
- **Stable anchors**: top-level port names; instance names and pin names of
  primitives and black-box IP; long, distinctive register names (e.g. a
  `_metaguard` suffix) with a trailing wildcard (slightly less safe).

## Preferred selection methods

[01S10]

Which query options exist, and how they match, depends on the tool. Quartus,
for example, has its own collection commands (`get_registers`,
`get_keepers`, `get_fanouts`, ...), its own options, and `|` as the hierarchy
separator. The methods below are described for Vivado; use the tool's
equivalent where there is one.

- **`-of_objects`** to select by relationship (pins of a net, the clock on a
  pin). Note that `get_clocks -of_objects` on a **PLL cell** can return several
  clocks; use a specific pin.
- **`-filter`** for property- or hierarchy-based selection (Vivado:
  `get_cells -hier -filter {name =~ */foo_metaguard*}`).
- Patterns that look written by trial and error, or copied from a GUI wizard
  without adaptation, need a logical justification.

## Wildcards and `-hierarchical`

[01S11]

The rules below describe Vivado. 01signal calls them the Synopsys
interpretation. Other tools may or may not follow it. For another tool, say
that the pattern **may** not match the desired elements.

The hierarchy separator is `/` in Vivado and `|` in Quartus. A constraint
ported between tools with the wrong separator may silently match nothing.

### Plain wildcards

- `*` matches any number of characters. `?` matches exactly one character.
- Neither wildcard matches the hierarchy separator. This is true with and
  without `-hierarchical`. There are two exceptions: the last `/` of a pin name
  under `get_pins -hierarchical`, and a pattern of only `*` with `-hierarchical`,
  which matches everything, e.g. `get_cells -hierarchical *` (see below).
- So each `/` in a pattern stands for exactly one hierarchy level. `*/clk_*`
  matches objects one level below the top. `*/*/clk_*` matches objects two
  levels below the top. Neither matches objects at any other level.

### Without `-hierarchical`

- The pattern is a full path from the top level. Every level must be written
  out.
- `get_nets clk_i*` finds only nets at the top level. It does not find
  `pll_i/clk_in1`.

### With `-hierarchical`

- Flag as Comment when used without `-filter`: the behavior is counterintuitive
  and confusing. Suggest using `-filter` instead.
- The pattern is tried at every level of the hierarchy. One command searches
  the whole design.
- **`get_cells -hierarchical` and `get_nets -hierarchical`**: the pattern is
  compared with the object's own name only, not its path. So a pattern that
  contains `/` never matches.
- **`get_pins -hierarchical`**: the pattern is compared with the cell's own
  name, plus `/`, plus the pin name (e.g. `clkout1_buf/O`). So the pattern
  must cover exactly one `/`, either written out or matched by a wildcard.

### `-filter` is different

- In a `-filter` expression (`name =~ ...`), `/` has no special meaning. `*`
  matches any characters, including `/`.
- So `get_pins -hierarchical -filter {name =~ pll_i/*out1}` matches both
  `pll_i/clk_out1` and `pll_i/inst/clk_out1`. Check that every match is
  intended.

### Review checks

- A pattern that must match something but matches nothing is a bug, even if
  the tool only warns [01S10].
- Generic patterns (`*metaguard*`, `*sync*`) can match unrelated logic,
  including logic inside IP or logic added later. Prefer distinctive names.
- With Vivado, prefer `-filter` or `-of_objects` over plain wildcards.

