# Phase 2 — Timing: clocking and timing constraints

This is the highest-value area of the review. A constraint error is worse
than an RTL bug: the tool reports "timing met" while the hardware fails, and
the failure is intermittent and temperature/voltage/part dependent.

Check each item against the clock table from Phase 0. Read a topic's file
when you start on it.

| Topic | File |
|---|---|
| Clock sources, PLLs and derived clocks | [clock-generation.md](clock-generation.md) |
| Clock buffers, clocks used in logic, clock forwarding and clock relationships | [clock-distribution.md](clock-distribution.md) |
| Clock constraints, timing coverage and timing reports | [timing-constraints.md](timing-constraints.md) |
| Timing exceptions (clock groups, false paths, max delay, multicycle) | [timing-exceptions.md](timing-exceptions.md) |
| I/O timing | [io-timing.md](io-timing.md) |
| Selecting objects in constraints (names, wildcards, `-hierarchical`) | [constraint-objects.md](constraint-objects.md) |

## Background reading for findings

Much of this phase follows the 01signal.com tutorial on timing constraints.
The topic files tag their paragraphs with the pages they draw on (`[01S6]`
etc., listed in
[../../references/01signal-pages.md](../../references/01signal-pages.md)).
Keep findings short. When the reasoning behind a finding is explained on a
tagged page, link to that page instead of repeating the explanation.

## Tool-specific behaviour

The 01signal pages, and most constraint examples in these files, are written
for Vivado. Other tools (e.g. Quartus) share much of the SDC syntax but **not necessarily its
meaning**. These things differ between tools:

- exception precedence, and how exceptions interact (e.g. whether clock
  groups cancel a max delay);
- options and their semantics (`-datapath_only` is Vivado's; Quartus
  `set_max_delay` / `set_net_delay` / `set_max_skew` work differently);
- which clocks the tool derives automatically, and what it assumes when
  `-min`/`-max` or jitter are not given;
- how object queries match (`-hierarchical`, wildcards, `-filter`, hierarchy
  separators);
- what the reports show and call things.

Statements marked **(Vivado)** describe Vivado only. Don't translate
one tool's command into another tool's syntax and assume it
means the same.

If the meaning of a command isn't explicitly defined in this skill set, but
a finding based on your own judgment still seems significant, report it
as Speculation.

The checks about **design intent** (is each clock constrained with its real
period, is each crossing treated the way the logic requires, does each
exception match only the paths it should) apply to every tool.

## Evidence

Many checks here can only be confirmed in the implemented design: clock
summaries, clock-interaction reports, unconstrained-path reports, and the
objects each constraint actually matches. If such reports are in the tree,
read them and compare with your own working notes. A mismatch is reported as
a high-priority Notification, to disclose your own mistake and the resulting
uncertainty of the review report.
