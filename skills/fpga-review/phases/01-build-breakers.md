# Phase 1 — Build-breakers

Things that stop synthesis, implementation, or bitstream generation — and the
related class of problems that do **not** stop the build but silently remove
or change logic. The second class is often worse.

**Levels.** Errors that the tools report themselves (syntax errors, illegal
constructs) go under "Possible errors reported by tools" in the report. Flag every
other issue here as Comment, unless the tools may silently interpret the code
or constraints differently than intended; then flag it as Notification.

**Reporting.** Keep plain errors brief. The tools will reject a syntax error
anyway, so show the offending line with a few words that make clear what is
wrong, and move on. Say more only when the tool's error message is likely to
confuse the user (e.g. it points to a different line, or the real cause is a
missing file or wrong compile mode). The issues that happen **silently** — lost
logic, implicit nets, truncated widths — are the ones worth a proper finding.

## HDL: Verilog

- Syntax: trailing commas in parameter/port lists, missing `endmodule` /
  `end`, misplaced semicolons after `begin`/`end` labels.
- `wire` driven from `always`, `reg` driven by `assign`.
- Multiple drivers: the same net assigned from two `always` blocks, two
  `assign` statements, or an `assign` plus a port output.
- Implicit nets: a typo in a port connection or `assign` creates a new 1-bit
  wire silently. Check for `` `default_nettype none `` and, if absent, check
  each identifier is declared.
- Width mismatches on port connections and assignments are silently truncated
  or zero-extended. Check every vector port connection against the port
  declaration, especially when parameters change widths. Meaningful data
  lost due to a truncation is a finding flagged as Notification. Other mismatches,
  which are practically harmless, are grouped and briefly mentioned as Comments.
- In instantiations, check for unconnected input ports,
  positional port connections that are out of order, `.*` connections that
  pick up the wrong same-named signal. Ignore if the affected input ports are
  unused by the instantiated module or output ports unused by the instantiating
  module. Flag as Comment otherwise.
- Macros that depend on file compile order; `` `define `` reused with
  different values.
- Non-synthesizable constructs in synthesis code:
  `fork`/`join`, `wait`, `$random`, `real` arithmetic, `force`/`release`,
  `initial` blocks doing more than setting initial values or loading memories.


## Constraints (Tcl)

- A backslash followed by one or more spaces at the end of a line: in Tcl,
  this is not a line continuation, because the backslash escapes the space.
  In an XDC file, Vivado nevertheless treats such a backslash as a line
  continuation. Don't report it as an error. At most, recommend removing
  the trailing space(s), as a matter of code hygiene: flag as Comment,
  as a one-line minor item.
