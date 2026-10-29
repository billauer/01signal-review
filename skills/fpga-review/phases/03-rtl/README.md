# Phase 3 — RTL correctness

Read every user RTL file in full. For each `always`/`process` block, know its
clock domain (from the clock table built in Phase 0) and its reset style.

Go through each topic below. Read a topic's file when you start on it.

| Topic | File |
|---|---|
| HDL semantics and synthesis/simulation mismatches | [coding-semantics.md](coding-semantics.md) |
| Clock-domain crossings (go through every signal between domains) | [cdc.md](cdc.md) |
| Resets and power-up | [resets.md](resets.md) |
| State machines | [fsm.md](fsm.md) |
| Memories | [memories.md](memories.md) |
| FIFOs | [flow-control-fifos.md](flow-control-fifos.md) |
