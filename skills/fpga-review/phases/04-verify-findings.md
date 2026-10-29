# Phase 4 — Verify every finding (mandatory)

Work on the draft report in your working notes. For each finding, re-open
the cited lines and try to **disprove** it:

- Is the "unused" signal actually used through a hierarchical reference,
  generate block, `.*` connection, or a Tcl/constraint property?
- Is the "missing constraint" supplied by an IP's own constraints, by a
  tool's automatic derivation, or by a file not in this tree? If you can't
  tell, phrase the finding as a question and say what would settle it.
- Is the "CDC bug" for real? Are the two clocks really unrelated?
- Is the "reset bug" handled in a way you missed?
- Did you compute PLL frequencies, counter ranges, throughput and FIFO depths
  from the actual parameters, not comments or names?
- Is the behaviour intentional and documented in a comment, spec or commit
  message? If so, either drop the finding or reduce it to a one-line note on
  the documented risk.
- Is the code active for the parameters actually used (generate branches,
  `ifdef`s, unused modules)?
- Does the finding depend on simulation-only code (`translate_off`,
  `ifdef SIMULATION`)?
- Is the level right? Re-read the definitions under "Reporting levels" in
  SKILL.md, and any "Flag as" in the phase file the finding came from. Don't
  turn style issues into Notifications; don't make timing-constraint holes
  Comments.
- Are several findings the same root cause? Merge them and list all
  locations.
- Is the finding likely to matter to the user — a visible effect, and
  something they may have overlooked? If it is an error the tools will
  report anyway (see "Possible errors reported by tools" in SKILL.md), move it
  to that section and keep it short.
- Is anything stated decisively that this skill set does not state
  explicitly? Rephrase it as a question or a point to consider.
- Does a suggested fix carry wisdom from this skill set? If it is only the
  obvious correction, or comes from general knowledge, drop it.
- Does the text refer to phases, topic files or item codes? Rephrase in
  terms of the design. This includes the `01S` codes: the report links to the pages
  instead.
- Are the links right? For a Notification or Comment, each linked page
  comes from a tag on the text the finding is based on, and no such page is
  missing. For a Speculation, each linked page is relevant to its subject.
  See "Linking to 01signal.com pages" in
  [../references/html-report.md](../references/html-report.md).

Drop findings that do not survive. Do not pad the report.

## Write the draft file

Write the draft, as it stands after this phase, to `speculative-review.html`
(see "Where the report goes" in [../SKILL.md](../SKILL.md)), in the same
format as the final report, following
[../references/html-report.md](../references/html-report.md). It must be
complete on its own: the Phase 5 agent sees only this file, the design tree
and this skill set, not your working notes. Don't add anything to it that
isn't part of the report (no notes to the inspector, no reasoning behind
the findings).

Then start the Phase 5 agent as described under "Phase 5 runs in a fresh
agent" in [../SKILL.md](../SKILL.md). Don't do Phase 5 yourself.
