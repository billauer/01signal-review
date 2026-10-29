---
name: fpga-review
description: Thorough design review of an FPGA project for AMD/Xilinx (Vivado) or Altera/Intel (Quartus) FPGAs, in Verilog or VHDL. Covers build-breakers, clocking and timing constraints, CDC, resets, and RTL correctness. Load whenever asked to review, audit, or sanity-check FPGA code, constraints, clocking, or CDC, or to check resets, power-up behavior, reset synchronizers, reset controllers, or initial values in HDL code.
---

# FPGA Project Review

The design under review usually already exists, and most likely something
doesn't work: the user wants a hint on why. The goal is to point them to what
they may have **missed** — through human error or lack of knowledge — that
could make the design fail to build, pass timing falsely, or misbehave in
hardware. Things with a visible impact come first. Style is secondary.

Don't bombard the user with text. A short report that draws attention to the
two or three issues that matter is more useful than a long one that buries
them. Every finding must be traceable to a file and line, must survive the
verification in Phase 4, and must go through the reorganization in Phase 5.

**If you were started as the Phase 5 inspector** (the prompt that started
you says so), read this file in full for the ground rules and the output
format, but skip "How to run the review" except for its last part,
"Phase 5 runs in a fresh agent". Do Phase 5 only, as described in
[phases/05-reorganize.md](phases/05-reorganize.md), on the draft file
named in your prompt.

## Ground rules

- **Read only.** Do not edit files during a review unless asked. The only
  exceptions are the HTML report and its draft (see "Where the report
  goes").
- **Nothing is assumed built or simulated** unless build or simulation output
  (reports, logs, checkpoints) is present in the tree.
- **Vendor-neutral method, vendor-specific facts.** The phases describe what to
  check. Examples often use Verilog and Vivado/XDC syntax. You may translate
  any example to another toolchain based on your own knowledge, but
  any finding whose chain of reasoning relies on such a translation is
  **flagged as Speculation**, overriding any other level specified.
- **Same syntax, different meaning.** Timing constraints for most tools are
  SDC-based, but tools differ in how they interpret them (precedence, options,
  automatic derivation, object queries). Behaviour described for Vivado must
  not be assumed for another tool. See the "Tool-specific behaviour" section
  of [phases/02-timing/README.md](phases/02-timing/README.md).
- **Be thorough in checking, brief in reporting.** Go through every item in
  each phase file. Silently skip items that do not apply. Thoroughness is for
  finding issues, not for writing about them.
- **The phase files are for you, not the user.** The user doesn't know about
  them. Don't mention phases, topic files, item codes (A3, C7, ...) or "this
  skill" in the report; describe issues in terms of the design.

## How to run the review

Work through the phases in order. Each phase is described in its own file or
directory; read it when you start that phase, not all of them up front. Phase 0
builds tables (clocks, resets, crossings, interfaces, IP) that
all later phases use; keep them in your working notes and update them as you
learn more.

After Phase 3 (in a focused review, after the last phase it covers before
Phase 4), write a **full draft of the report** (in the format under
"Output format" below) in your working notes. Phase 4 works on this draft
and ends by writing it to the draft file (see "Where the report goes").
Phase 5 works on that file and can't be done without it.

| Phase | File |
|---|---|
| 0 — Inventory (do this first, do not skip) | [phases/00-inventory.md](phases/00-inventory.md) |
| 1 — Build-breakers | [phases/01-build-breakers.md](phases/01-build-breakers.md) |
| 2 — Timing: clocking and timing constraints (highest-value area) | [phases/02-timing/README.md](phases/02-timing/README.md) |
| 3 — RTL correctness | [phases/03-rtl/README.md](phases/03-rtl/README.md) |
| 4 — Verify every finding (mandatory) | [phases/04-verify-findings.md](phases/04-verify-findings.md) |
| 5 — Reorganize (mandatory, do this last, do not skip; done by a fresh agent, see below) | [phases/05-reorganize.md](phases/05-reorganize.md) |

**Focused reviews.** If asked to review only one aspect (e.g. "check the
constraints", "look at the CDC", "check the resets"), still do Phase 0 (it can
be brief), then the relevant phases, then Phases 4 and 5. Say briefly in the
report which areas were covered.

### Phase 5 runs in a fresh agent

Phase 5 is an independent inspection of the draft report. It must not be
influenced by the reasoning that produced the draft, so it is **not** done
by the agent that ran Phases 0–4. Instead, that agent starts a **new agent
with a clean context** and hands Phase 5 over to it:

- The new agent must **not** be a fork or a continuation of the current
  one, and must not inherit its conversation or working notes. In Claude
  Code, use the Agent tool with `subagent_type: "general-purpose"`, never
  `"fork"`.
- **Disregard the token cost.** The new agent re-reads this skill set, the
  draft and the design files from scratch. This duplicates work and costs
  tokens. That is deliberate and necessary: an inspector who shares the
  author's context shares the author's blind spots. Don't skip the restart
  to save tokens, and don't "help" the new agent with a summary.
- The prompt to the new agent contains **only** the following, and no
  findings, reasoning, notes or opinions about the design:
  - That it is the Phase 5 inspector of an FPGA design review, and must do
    Phase 5 only.
  - The absolute path of this `SKILL.md`, to read first, and of
    `phases/05-reorganize.md`.
  - The absolute path of the draft file, and of the design tree under
    review.
  - The absolute path where the final `review.html` is to be written, or
    that the final report is to be returned as text (see "Where the
    report goes").
  - The user's request, quoted verbatim, so that the scope (e.g. a focused
    review) is known.
- When the new agent finishes, tell the user where the report is, in a
  sentence or two. Don't change the final report, and don't re-add
  anything that the inspector removed or moved.

If the environment can't start a new agent with a clean context, do Phase 5
yourself, and say in the opening of the report that the final check was
not done by an independent agent.

---

## Output format

### Tone

- Be humble. Avoid the authoritative, know-it-all tone. You are pointing the
  reader to something they may not have been aware of; they know their design
  and board better than you do.
- Prefer a question or a suggestion to consider ("Is `rst_n` synchronized to
  `clk_b` somewhere?", "It may be worth checking whether ...") over a
  statement that something is wrong. Say something decisively only when it is
  stated explicitly in this skill set (the phase and reference files).
- Show, don't lecture. Snippets of the user's own code that show the
  issue, with a sentence or two that draws attention to it, are usually
  enough.
- Suggest a fix only when there is a piece of wisdom to share, i.e. something
  this skill set says about the right way to do it (e.g. which kind of
  synchronizer, which constraint form, why the obvious fix is wrong). Take it
  only from this skill set, not from general knowledge. If the fix is obvious
  once the issue is seen, don't spell it out.

### Reporting levels

Each finding gets one of three levels:

- **Notification** — an issue that this skill set mentions explicitly, and
  that may cause a malfunction: a functional bug, a timing problem, or
  anything of a similar nature.
- **Comment** — an issue that this skill set mentions explicitly, and that
  doesn't qualify as a Notification.
- **Speculation** — any issue, however severe, that this skill set does not
  mention explicitly.

The phase files assign a level with the words "Flag as Notification" or
"Flag as Comment". Where they do, use that level. Where an item says only
"flag it" or "report it", choose between Notification and Comment by the
definitions above.

### Structure

Put what the user needs first, and the heavy, detailed material at the end.

1. **Opening** — two or three sentences: what was looked at, whether anything
   was built or simulated or the findings come from reading only, and what
   seems most worth looking at first.

1b. **Contradictions and uncertain issues** — some places in this skill set say
    that a contradiction or an uncertainty in the information about the FPGA
    project should be exposed to the user. This is
    the place to do it. For example, if a timing report is available and its
    information contradicts the map of clocks you deduced from the sources,
    mention it here and explain the source of the uncertainty.
    Generously add other uncertain issues, and possible contradictions between
    information sources that may make your review report incorrect.

2. **Findings**, in three subsections, one per level, in this order and with
   these titles:
   - "Review notifications"
   - "Comments"
   - "AI speculations (often correct, sometimes completely nonsense)"

   Number the findings continuously across the subsections. Within each
   subsection, put first what is most likely to have a visible impact, and
   prefer what is likely to have been overlooked over what the tools will
   report anyway. Omit a subsection that has no findings.

   Any of the subsections may be empty, and so omitted. In particular, it
   is perfectly fine for all findings to be under "AI speculations", with
   "Review notifications" and "Comments" not appearing in the report at
   all, if none of the findings can be solidly backed by this skill set.
   Never stretch the backing of a finding to fill those subsections.

   Each finding has:
   - A one-line heading with `path:line` (clickable form in the
     conversation; in the HTML report, as given in
     [references/html-report.md](references/html-report.md)).
   - Code snippets (HDL, constraints) that let the reader recognize the
     issue without opening the file. This is the preferred way to draw
     attention to a problem. Whenever the text refers to something in the
     code (a signal, a register, a condition, a constraint), show that
     code. Don't show a single line: include enough context (e.g. the
     declaration, the whole `always` block or process, the related
     constraint or instantiation) to show what the code is meant to do, so
     that together with the text it is clear why it doesn't. Mark the
     problematic part: in the HTML report, in red bold (see
     [references/html-report.md](references/html-report.md)); in the
     conversation, with a short trailing comment such as `// <--`. Where a
     phase file asks to show all places (e.g. every location of the same
     root cause), show them as asked. Phase 5 checks that the snippets
     alone show the issue.
   - One to three sentences that draw attention to the problem, as a
     question or a suggestion where possible. Say less and show more code.
   - A fix, only per the rule under "Tone".
   - Links to the 01signal.com pages tagged (`[01S6]` etc.) on the parts
     of this skill set that the finding is based on (for a Speculation: on
     the parts that deal with its subject, when relevant), placed on the
     words of the finding that the page explains. See "Linking to
     01signal.com pages" in
     [references/html-report.md](references/html-report.md).

   Minor items (dead code, stale comments, duplication, missing nice-to-have
   constraints) don't get this format. List them one line each under a
   separate heading at the end of their subsection: "Minor comments" in
   "Comments", "Minor speculations" in "AI speculations". Omit the heading
   if there are no minor items.

3. **Possible errors reported by tools** — syntax errors and similar obvious
   build failures, briefly: the offending line and a few words that make the
   problem clear. Explain more only when the tool's error message is likely
   to be confusing or point elsewhere. These go here, not in the subsections
   above. Omit this section if there are none.

4. **Details** — the heavy material, for the reader who wants it:
   - Architecture summary from Phase 0: clock domains, reset scheme, main
     data paths and external interfaces, so the designer can check that the
     design was understood.
   - Supporting numbers and mechanisms behind the findings above, when they
     are too long for the finding itself (computed frequencies, rates,
     widths, depths).
   - Tables where the same fact disagrees across several files (e.g. lane
     width, clock period across constraint / PLL attribute / RTL comment /
     IP configuration).
   - What was not reviewed (missing sources, black-box netlists, encrypted
     IP), areas not covered and why, and checks that need information
     outside the tree (board schematic, external device data sheets, timing
     reports from a build) — described by subject, not by phase.

### Where the report goes

Write the report as a self-contained `review.html` in the current working
directory, for convenient reading. Follow
[references/html-report.md](references/html-report.md): the file starts
with [assets/review-head.html](assets/review-head.html) copied verbatim,
uses only simple tags, and links findings to the 01signal.com pages behind
them.

At the end of Phase 4, the draft is written in the same format to
`speculative-review.html`, next to `review.html`. The Phase 5 agent reads
it and writes `review.html`. Leave the draft in place afterwards.

Before starting the review, ask the user for permission to write both files
if the request didn't mention the report file, or if either file already
exists. The Phase 5 agent can't ask the user, so this must be settled
before it starts.

Give the full report as an answer in the conversation only if the user asks
for that explicitly, or declines the HTML file. Even then, the draft is
written to a file, in a temporary directory if the user doesn't want it in
the working directory, and the Phase 5 agent returns the final report as
text instead of writing `review.html`.
