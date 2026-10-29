# Phase 5 — Reorganize (mandatory)

This phase is done by a new agent, started with a clean context, that acts
as an **independent inspector** of the draft report. If you are that agent,
you know nothing about how the draft came about, and that is the point:
judge the draft only by this skill set and the design files, not by the
reasoning of whoever wrote it. Do this phase only; don't redo Phases 0–4
or look for new issues.

Re-reading the skill set, the draft and the design files from scratch costs
tokens. That is deliberate; don't cut corners to save them. Re-open every
file you need to check a finding.

The draft is in the file named in the prompt that started you (normally
`speculative-review.html` in the user's working directory), checked in Phase 4.
If there is no draft file, stop and report that, without writing a report
of your own.

Read through the entire draft. For each finding in the "Review
notifications" and "Comments" subsections:

1. **Check that this skill set backs the finding.** Keep it in its
   subsection only if at least one of these holds:
   - The finding matches a guideline **in this skill set** for what is
     correct, and the design contradicts that guideline.
   - This skill set **requires** reporting this kind of issue, and the
     design meets the stated conditions.

   Check the match strictly: the guideline or requirement must cover this
   case as it stands, not a similar one. If neither holds, move the finding
   to "AI speculations", however sure you are about it.

2. **Check every sentence in the finding.** This covers factual statements
   as well as any claim about what the issue will or won't cause: that it
   is harmless, has no effect, works anyway, is unlikely to matter, will
   fail or is dangerous. Keep a sentence only if it is backed in one of
   these ways:
   - A statement in this skill set backs it explicitly and directly.
   - It follows directly from the strict syntax of the HDL code, the timing
     constraints or any other part of the reviewed material. What merely
     seems obvious doesn't count if the syntax itself doesn't establish
     it. For example, setting a PLL's CLKIN1_PERIOD attribute to 10.0
     surely means a 10 ns clock period, but that is not good enough to
     meet this condition. It is backed only because this skill set states
     it explicitly, which meets the first condition.

   Remove every other sentence, with these exceptions:
   - If removing it would leave nothing of the finding, move the whole
     finding, unchanged, to "AI speculations" instead.
   - If the finding can't be understood without the sentence, rephrase it
     as a question or as an expression of uncertainty. For example, if the
     finding says "Vivado will ignore this timing constraint", nothing in
     this skill set backs that, and the point can't be made without it,
     write "Are you sure Vivado won't ignore this timing constraint?" or
     "I couldn't determine whether Vivado ignores this timing constraint."

3. **Check the chain of claims leading to each finding, and every cause
   and effect in it.** If a finding can't be logically established without
   a claim that isn't directly supported by this skill set or by a strict
   interpretation of HDL syntax, move the finding to "AI speculations".

   Then check every statement of cause and effect in the finding. A wrong
   reason is worse than no reason: a reader who sees that the stated reason
   is wrong dismisses the whole finding, even when the finding itself is
   right. So be rigorous here.

   **Find every causal statement**, explicit or implied:
   - Explicit wording: "because", "since", "as", "due to", "caused by",
     "therefore", "so", "thus", "hence", "as a result", "consequently",
     "which means", "leads to", "causes", "results in", "makes",
     "prevents", "allows", "breaks", "so that", "in order to", "otherwise",
     "if ... then", "without ... ".
   - Implied cause: two facts placed side by side so that one reads as the
     explanation of the other ("The reset is asynchronous. The FSM may lock
     up."); a consequence attached to an observation ("..., which can
     corrupt the data"); a heading that names an effect next to code that
     shows a cause; a fix or a link phrased so that it implies what the
     cause is.

   **For each one, check all of the following.** Re-open the phase or
   reference file that the finding is based on, and re-read the paragraphs
   about this subject; don't rely on your memory of them. Also look for other
   places in this skill set that deal with the same subject: they may give
   a different or more fundamental reason.
   - **The cause is present.** It is actually in the reviewed code,
     constraints or reports, as shown in the snippets, and not inferred
     from names, comments or the usual practice.
   - **The cause produces the effect.** The mechanism is stated in this
     skill set, or follows strictly from the semantics of the HDL or the
     constraints. "It is plausible" is not enough.
   - **It is the main reason.** If this skill set gives a more fundamental
     reason for the same effect, give that one. For example, the problem
     with an asynchronous reset is usually not that it is asynchronous, but
     that its release isn't timed relative to the clock. A secondary reason
     may stay only after the main one, and worded as secondary.
   - **The reason is complete.** If the effect needs several conditions
     together, name all of them (e.g. an untimed reset release **and**
     logic that can't tolerate some flip-flops missing the first clock
     edge). Don't present one condition as if it were enough on its own.
   - **The direction is right.** The cause and the effect are not
     reversed, and two things that merely occur together are not presented
     as cause and effect.
   - **The effect is the right one, at the right strength.** The stated
     consequence is the one that this cause produces, not a more dramatic
     one. "May" or "can" where the skill set doesn't say "will".
   - **The reason applies to this case.** It holds for this design, this
     tool and these conditions, not for a similar case that this skill set
     discusses (e.g. Vivado behaviour applied to another tool, or a rule for
     unrelated clocks applied to related ones).
   - **The link fits.** Where a link to a page is placed on the words
     giving the reason, the page explains that reason.

   **If a check fails:** replace the reason with the correct main reason,
   as this skill set gives it. If the correct reason can't be established,
   remove the causal claim, or turn it into a question or an expression of
   uncertainty ("Could this be why ...?"). If the finding depends on the
   claim, move it to "AI speculations". Never keep a reason only because it
   sounds right.

Findings moved to "AI speculations" keep their order of importance there.
They keep their links to 01signal.com pages where the pages are still
relevant to them, as background rather than as backing.

Run the cause-and-effect check of step 3 on every finding in "AI
speculations" too, including those that were there from the start. A
speculation may give a reason that this skill set doesn't back, but only
worded tentatively, and only if it is the most likely main reason. Where
this skill set covers the subject, the reason must agree with it.

Then, for every finding in all three subsections:

4. **Check that the code snippets tell the story.** Imagine a reader who
   has the full theoretical background but reads only the finding's code
   snippets, with the problematic parts marked (red bold, the `punch`
   class), and not the text. Could that reader see what the code is meant
   to do and why it doesn't? If not, expand the snippets until they can:
   add the declarations, the rest of the `always` block or process, the
   code that drives or uses the marked signal, the related constraint or
   instantiation, and mark the problematic parts. The only exception is an
   issue that can't be shown with code at all (e.g. a missing file or a
   missing constraint).

5. **Cut the text that the snippets made unnecessary.** Remove sentences
   that describe what the snippets now show (what drives a signal, which
   clock a block runs on, what a constraint covers). Keep what the code
   can't show: the question to the reader, the consequence, and the links.

It is perfectly fine if, at the end, every finding is in "AI speculations"
and the "Review notifications" and "Comments" subsections are gone. Don't
keep a finding in them for the sake of having something there.

Then renumber all findings continuously, and write the final report to the
location named in your prompt (normally `review.html`, next to the draft),
following [../references/html-report.md](../references/html-report.md), or
return it as text if the prompt says so. Don't modify the draft file.
Finish with a short reply saying where the report is; don't repeat its
content.
