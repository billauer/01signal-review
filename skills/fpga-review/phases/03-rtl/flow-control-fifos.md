# Phase 3 — RTL: FIFOs

## FIFOs

- **Every FIFO**: `full` honoured by the writer, `empty` honoured by the
  reader (`wr_en` low whenever `full` is high, `rd_en` low whenever `empty`
  is high). FIFO IP often ignores such requests, but not honoring `empty`
  and `full` can be an indication of oversight [01S24]. Flag as Comment.
- **Read latency** matches the reader logic [01S24] [01S25]:
  - Standard FIFO: `dout` is valid on the clock **after** `rd_en`. Consumers
    must qualify `dout` with a delayed `rd_en` (`rd_en_d`), not `rd_en`.
  - First-word-fall-through (FWFT): `empty` low means **`dout` is valid**, not
    that more words are stored. A read (`rd_en`) consumes the word on the
    data output port.
  - Unless the IP's configuration file shows definitively whether the FIFO
    is standard or FWFT, don't check or comment on read latency.
- Single-clock FIFOs are preferred when both sides are on the same clock
  domain. Suggest a single-clock FIFO as a minor Comment in this case.
  It is **not** a mistake to use dual-clock FIFOs where no CDC exists:
  it merely adds a little logic [01S25].
- A single-clock FIFO's reset must be synchronous to its clock [01S25].
- **Flags lag behind**: in dual-clock FIFOs, `empty` reacts to writes and
  `full` to reads only after several clocks of synchronization, varying with
  clock alignment. Logic that assumes an immediate update is a finding [01S24].
- **Fill counters** on dual-clock FIFOs
  are synchronous to one side and usually pessimistic. A
  read-side counter (which may briefly show fewer words than stored) used to
  control writing can cause overflow. Check that no unsynchronized CDC
  occurs on a fill counter [01S25].
- **Almost-full / almost-empty / programmable thresholds** account for the
  pipeline latency of the logic that reacts to them: count how many words
  can still arrive after the flag asserts. For a registered
  `rd_en` driven by `!almost_empty`: check that the **last word isn't stuck**:
  the safe form is
  `rd_en <= want && (!almost_empty || (!empty && !rd_en));` [01S25].
- A registered `wr_en` that uses `full` (rather than `almost_full`) must skip
  a cycle after each write, halving throughput; check that's acceptable or
  suggest using `almost_full` instead [01S25].
- **Asymmetric widths**: with a wider read side,
  a partial word isn't visible until complete (it may be stuck at end of
  stream) [01S25].
- **FIFO reset**: if the FIFO is dual-clock, its reset comes from a register,
  not from combinatorial logic: this reset is asynchronous, so a glitch resets
  the FIFO [01S24].
- Whether `full` is asserted during reset is configurable. Logic that interprets
  `full` as an overflow must ignore it during and right after reset, unless the
  FIFO is guaranteed to hold `full` low during reset [01S24].
