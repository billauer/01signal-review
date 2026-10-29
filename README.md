# FPGA design review skill set

An agent skill set for reviewing FPGA projects in Verilog or VHDL, with a
focus on AMD (Vivado). It covers build-breakers, clocking and timing
constraints, clock domain crossings, resets and RTL correctness, and
produces an HTML review report.

## Introduction

This skill set has two main purposes:

- Producing a useful review of an FPGA project, by spelling out what should be checked and what is considered correct, rather than relying on the AI model's own embedded knowledge.
- Allowing the freedom of choosing an AI model, in particular using locally running LLMs.

Code reviews performed by AI models tend to be a mixture of truth and nonsense. In order to tackle that issue, this skill set lists all conditions that the AI model should report. The AI model can still report issues out of its own embedded knowledge; however, these issues appear in a separate section, named "AI speculations".

The review procedure's last stage requires a separate AI agent to check everything in the review report, and verify that it's covered by this skill set. Anything not explicitly covered is moved to "AI speculations", and any factual claim not based upon this skill set is expressed as a doubt (possibly a question) rather than a fact stated with all confidence.

Large parts of this skill set are based upon the [01signal website](https://www.01signal.com/). The review's remarks often link to specific pages there, which explain the topic that the remark is about. For mistakes made due to a lack of knowledge, these links help fill the gaps.

## Using this skill set

The skill set focuses on AMD FPGAs, and was developed with Claude Code, using various LLMs. It works fine with code for other FPGA vendors; however, this increases the reliance on the LLM's own knowledge.

There is no need to install anything in order to run this skill set, even though it's recommended to do so if you use it repeatedly. Note that Claude Code's CLI tool can be used with a different model.

A typical prompt (without installation, with this repository cloned into `01signal-review/`) could be:

```
> Using 01signal-review/skills/fpga-review/SKILL.md, perform a
  review on the FPGA project. Write out the HTML files.
```

Or, if you're really lazy, don't even clone this repository: give your agent
the link to the repo's GitHub location instead.

Two output files are generated in the current directory:

- `review.html`: the final review report. Read this one.
- `speculative-review.html`: the report as it was before a separate AI agent went through it and verified that anything claimed to be a fact is either written explicitly in the skill set or derived directly from the code. It may be worth comparing it with the final report. The differences aren't always significant, and some AI rubbish sometimes slips through regardless.

And remember that LLMs have an intrinsic randomness. Running the review on the same design twice might give slightly different results.

## Installation

If you'd like to install this skill set permanently, ask your LLM how to do it.

## Why is this necessary

Ask an LLM to write a review, and it will make you one. What it doesn't know it makes up. Besides, the LLM's built-in knowledge has been collected from various sources, not all of which are reliable. Even when the sources are reliable, LLMs often extrapolate unrelated information, ending up with completely false claims, often expressed with convincing language.

Another issue is that LLMs tend to babble. The output isn't always accurate, and the default review usually consists of a lot of tangled wording.

This skill set tackles this in four ways:

- Expressing what to look for, what is correct, and what is an error.
- Requesting that the remarks in the main part of the review consist only of findings that are solidly based upon facts included in the skill set. Plus requesting that the issues be explained through code snippets rather than talk, when possible.
- In the explanations for each finding, softer wording is requested for parts of the logical reasoning not based upon the skill set or derived directly from the code. Rhetorical questions and "I couldn't find proof for..." should be used instead of the LLM's well-known self-confident tone.
- Requesting that all output be reviewed and polished twice: once before creating `speculative-review.html`, and then again by a separate AI agent that goes through this report and creates `review.html`.

The LLM can still produce remarks partly or fully based upon its own built-in knowledge, but these remarks appear as "AI speculations" in the report.

## Improving this skill set

It's quite easy to change the behavior of the review by modifying this skill set. You definitely don't need to know how it works in detail to do that. Rather, use the AI prompt itself, and ask it to modify the skill set. It's of course possible to ask the AI tool to read some documentation (preferably a specific part of it), and enrich the skill set accordingly. But remember to check every single word with a diff tool (see below).

For example (this is already done in the skill set):

```
> In Tcl, a backslash followed by a space at the end of a line doesn't work as
  a line continuation, because the backslash escapes the space instead.
  However, Vivado treats such a backslash as a line continuation in XDC files.
  Update the skill set to include this information. If a backslash followed by
  a space is encountered in a constraint file, no more than a short and minor
  recommendation to remove the trailing space(s) should be made, as a matter
  of better code hygiene. Commit your change.
```

After such a request, check the commit that was made, and make sure it says exactly what you intended. In particular, make sure that the AI tool hasn't added any insights of its own that you don't agree with.

If necessary (it usually is), edit the AI tool's update, possibly in a crude manner: don't make a fuss about spelling and grammar. Then ask the AI tool to clean up your edits with something like "I've edited file X. Clean up this edit to fit the style of a skill file."

But:

- Read every single line added or changed. Take responsibility for every single word. Manual editing is usually required, and often a whole lot of it.
- Actively look for claims that sound correct, but are too general or maybe completely wrong. LLMs are masters at tricking us this way. One of the important points of a skill set is to keep this rubbish out.
- Don't let expressions like "check with the documentation" and such sneak in. This gives the LLM the excuse to rely on some random source that is either incorrect or correct in a different context (e.g. another FPGA family or toolset) and apply its content as a solid base for a finding.
- Be careful with extra features you didn't ask for. You wanted to cover FPGA family X, and got wider skills including Y and Z too. That's OK if you can and are ready to take the time to confirm everything is rigorously correct. Otherwise, you're most likely letting rubbish in.
