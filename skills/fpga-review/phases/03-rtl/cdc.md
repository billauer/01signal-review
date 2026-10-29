# Phase 3 — RTL: clock-domain crossings

This topic follows the 01signal.com "Crossing clock domains" series
([01S18], [01S19], [01S20]), plus general CDC practice. Items without a tag come
from general practice.

The series' main point [01S18]: **every path between two clock domains needs either
resynchronization logic or enforced timing constraints, and which one depends
on whether the clocks are related.** Timing can only be guaranteed when the
time between the launching and capturing clock edges is known. Between
unrelated clocks it isn't, so no constraint can make such a path safe. Only
resynchronization logic can.

| Path between domains | Related clocks | Unrelated clocks |
|---|---|---|
| No resynchronization logic | Timing constraints **must** be enforced on the path | **A mistake** (A3) |
| Resynchronization logic | Constraints not needed; prefer relaxing them (A5) | Correct; enforced constraints are meaningless |

Examples below use Verilog and Vivado XDC. Constraints and attributes may
be translated to another tool, but don't assume a translated constraint
behaves the same way: see the "Tool-specific behaviour" section in
[../02-timing/README.md](../02-timing/README.md). Statements about
how constraints behave in section F are about Vivado unless stated otherwise.

## Procedure

1. **Classify every pair of clock domains** as related or unrelated (section
   A), using the clock table from Phase 0. Write down how the tools see each
   pair, too.
2. **Find every crossing.** Start from the preliminary CDC list in Phase 0,
   then find crossings it missed: for every register, check the domain of
   every register feeding its D, enable and reset inputs, including through
   combinational logic that mixes domains. In designs where signals cross
   module boundaries, it is easy to lose track of which clock a signal belongs
   to. Check the clock of each port, not only the module's main clock [01S18].
3. **For each crossing**, identify the scheme (sections B–D) and check it.
4. **Check the constraints** that go with each scheme (section F).
5. **Report** each finding with the signals, the source and destination
   clocks, and a concrete failure scenario.

---

## A. Clock relationships

**A1. Related vs. unrelated.** [01S18] Clocks are *related* (synchronous) if they are
derived from a common reference so that the time between their edges is
predictable, apart from known jitter and inaccuracies, e.g. several outputs of
one PLL through the tool's clocking resources. Otherwise they are *unrelated*
(asynchronous). Watch for clocks that look related but aren't:
- Clocks from the same reference whose skew is uncontrolled or unknown (not
  distributed through clock resources with known delay) are unrelated.
- A clock with the same frequency as its reference but an unknown phase (e.g.
  through an external chip) is unrelated to it.
- The same physical oscillator fed to two FPGA pins gives two clock signals.
  Whether the tools treat them as related depends on the constraints, not on
  the board.

**A2. Related clocks are timed at their worst-case edge separation.** Paths
between related clocks are analyzed against the shortest time between a
launch edge and a capture edge, not either clock's period. This is often far
tighter than either clock. Example: 90 MHz and 100 MHz from a 10 MHz
reference have edges as close as ~1.11 ns apart, equivalent to a 900 MHz
clock. This can explain why related clocks are mistakenly declared as
unrelated in the timing constraints, as timing wasn't met otherwise [01S18]
[01S3].

**A3. Path between clocks declared as unrelated in the timing constraints
(false paths and/or clock groups) without resynchronization (harmful
mistake 1).** Flag any crossing between such clocks that has no
resynchronization logic. It works most of the time and fails occasionally
[01S18]. Flag as Notification.

**A3b. Falsely related clocks.** Paths without synchronization between unrelated
clocks that have approximately the same frequency, or an integer ratio between
their frequencies. Unless the timing constraints define them as unrelated,
Vivado assumes them to be related, and enforces timing as if the clocks were
synchronized. As the frequencies have an integer ratio, it may be easy to meet
timing, even though the timing calculations are meaningless due to possible clock drift.

**A4. Clocks related by design, but not known to the tools as related
(harmful mistake 2).** If the logic relies on two clocks being related (no
resynchronization), but the constraints don't tell the tools so, those paths
are unconstrained and timing is not guaranteed. Typical causes: a blanket
`set_clock_groups -asynchronous` or false path that also covers these paths
[01S18] [01S5]. Flag as Notification.

**A5. Unnecessary constraints on resynchronized paths.** Clocks from the same
PLL are usually treated as related by the tools, even when the designer uses
them as unrelated (e.g. separate functional units on 90 MHz and 100 MHz). If
all paths between them go through resynchronization logic (often a dual-clock
FIFO), the tools still try to meet the worst-case edge separation of A2 on
those paths. This is pointless and can make timing closure fail. Fix: declare
the clocks unrelated (`set_clock_groups -asynchronous`), or use false paths or
max-delay exceptions on the crossing paths [01S18]. Flag as Comment.

**A6. Treating related clocks as unrelated.**
If the timing constraints define two clocks as unrelated even though they
are related in reality, and every crossing has resynchronization logic,
that is a design decision and not a finding.

**A7. Don't fix a CDC by adding a timing constraint.** A constraint between
two clocks only helps if they really are related. Flag constraints
(`set_max_delay`, generated-clock relationships, multicycle paths) that
appear to have been added to make a crossing between unrelated clocks "meet
timing". Tools usually accept and enforce such constraints, and the result
looks safe while it isn't [01S18].

**A8. Tool assumptions differ.** Each tool decides which clocks are related
from its own rules. E.g. Vivado is more inclined than ISE to assume clocks are
related [01S18].

## B. Single-bit signals (levels)

Single-bit resynchronization is the cornerstone of every other CDC scheme.

**B1. Metastability guard (synchronizer) structure.** [01S19]
```verilog
always @(posedge clk2)
  begin
    bar_metaguard <= foo;        // may go metastable
    bar <= bar_metaguard;        // first register used by logic
  end
```
Check that:
- At least one guard register (a 2-flip-flop synchronizer) is in the
  **destination** domain. Two guards (3 flip-flops) are sometimes used for
  high clock frequencies and high MTBF requirements, but having only one
  guard is never a finding.
- **A flip-flop at risk of metastability is never connected to more than one
  logic element.** A metastable output can be read as 0 by one destination and
  1 by another, putting the logic in an illegal state. So there is no fan-out
  from the guard, and no logic between the guard and the next register.
- There is one synchronizer per signal per destination domain, in one place,
  not duplicated in several modules (see E2).

Metastability can't be eliminated. Its resolution time has no upper bound and
depends on process, temperature and noise. The goal is an acceptable MTBF.
Don't flag a correct synchronizer for "not being failsafe".

**B2. Naming convention.** Guard registers with a recognizable common name
(e.g. a `_metaguard` suffix, or the vendor's convention) let one constraint
line cover all of them (F1, F2), and make them easy to find in the netlist.
However, it's not a finding if guard registers have no naming convention
[01S19].

**B3. Tool attributes.** Recommend marking synchronizer flip-flops with the tool's
attribute (e.g. Vivado `ASYNC_REG`). It keeps them from being manipulated
by the synthesizer and encourages
placing them next to each other. It can be set in the HDL or in the
constraints:
```tcl
set_property ASYNC_REG true [get_cells -hier -filter {name=~*_metaguard*}]
```
Flag a missing attribute as a minor Comment. Tools usually handle guards correctly
anyway [01S19].

**B4. Transitions the destination must not miss.** No frequency relation
between the clocks is required for a level to cross. But if every transition
of the source must be seen, the source must hold each value long enough:
slightly longer than one destination clock period, more precisely the
destination period plus `tsu + thold + 2·tj` (jitter counted once per clock).
For a signal that can change on every source clock, the source clock period
must be at least that. If this isn't guaranteed and transitions matter, use
C2 or a FIFO [01S19].

## C. Events (pulses, strobes, commands)

**C2. High-rate events: Gray-coded event counter.** To convey events that may
occur faster than the destination can sample a toggle, the source counts
events in a Gray-coded counter and sends the counter across (as in D2). The
destination compares successive values to find how many events occurred. The
counter must be wide enough that, even with an event on every source clock
cycle, it can't wrap between two destination samples. A 1-bit Gray counter is
a plain toggle [01S20].

**C3. Prefer a FIFO for messages.** Encoding the event as a message word and
writing it into a shallow vendor dual-clock FIFO is less bug-prone and more
readable than custom toggle/handshake logic. A shallow FIFO usually doesn't
need block RAM, so the cost is small. Recommend this when custom event logic
looks fragile [01S19].

## D. Multi-bit values (data, counters, status, configuration)

Synchronizing each bit separately doesn't work. Bits arrive with different
routing delays, so the destination may see a value the source never had
(e.g. a counter going from 0xFF to 0x00 may be seen as almost any value).
First ask: must the destination **always** hold a valid copy (continuous
sampling, D2), or does it only need occasional updates (D3/D4)? [01S20]

**D1. Dual-clock FIFO — preferred.** A vendor-generated dual-clock FIFO
(independent-clock mode) is the safest and most common solution, even if the
surrounding design must be adapted to it. A custom dual-clock FIFO is not
recommended, and must use Gray-coded
pointers (D2) with the constraints of F2. Check the FIFO's reset requirements
(see [flow-control-fifos.md](flow-control-fifos.md)) [01S19].

**D2. Continuous sampling: at most one bit changes per source clock.** [01S20] A word
sampled continuously by the destination is safe only if at most one bit
changes per source clock cycle. Each change is then either seen or missed, and
every value the destination sees was a real source value. Gray coding ensures this,
and is used for FIFO pointers and event counters. Check that:
- The value really changes by at most one bit per source clock, including
  at wrap-around, if there is one.
- The Gray encoding (or similar setup) is **registered in the source domain**. A combinational
  binary-to-Gray conversion before the guard can glitch on several bits.
- Each bit goes through a guard (B1) in the destination.
- If the destination is slower, values may be skipped. That must be acceptable
  to the logic (it is for FIFO pointers and event counters).
- The skew between bits is bounded (F2). Without that, bits could arrive out of
  order and the destination could see a value the source never had.

**D3. Occasional updates: toggle handshake with stable data.** [01S20]
```verilog
// Source domain (clk1)
always @(posedge clk1)
  if (do_update) begin
    foo    <= new_value;
    toggle <= !toggle;
  end

// Destination domain (clk2)
always @(posedge clk2)
  begin
    toggle_metaguard <= toggle;
    toggle_a <= toggle_metaguard;
    toggle_b <= toggle_a;
    if (toggle_a != toggle_b)
      bar <= foo;   // foo crosses without a guard: it is stable by now
  end
```
Only the toggle is resynchronized. The data needs no guard, because it has
been stable for at least two destination clocks when it is sampled. Check
that:
- **The update rate is limited.** `do_update` must not come again before the
  destination has sampled `foo`. A reasonable minimum spacing is 4 destination
  clock cycles, converted to source clock cycles and rounded up. If the source
  clock is at least 4 times slower than the destination, no restriction is
  needed. Otherwise, logic must enforce the spacing. A missing rate limit is
  this scheme's main weakness. Flag it.
- `foo` changes only together with the toggle.
- The toggle registers are reset or initialized consistently on both sides.
  A mismatch costs at most one spurious update, but resetting them is
  recommended.
- A one-cycle `new_bar` strobe in the destination, if present, is generated
  from the same `toggle_a != toggle_b` condition. It is a common way to pass
  commands and status.

**D4. Double-buffered RAM.** Large blocks of data can cross with a dual-port
RAM: the source fills one half, then toggles. The destination reads that
half, and the toggle also tells which half is being written. Prefer a FIFO
when possible [01S20].

**D6. Quasi-static values without protection.** Configuration or status
values copied continuously across domains with no mechanism, on the grounds
that they "rarely change", often work and then fail occasionally. Accept it
only if the value changes exclusively while the consumer doesn't depend on
it (for example in reset), and that is **enforced**. Otherwise flag it and
recommend D3. Logic that depends on a signal from a different clock domain
is at risk of entering an undefined state due to synthesis optimization.

## E. Structural CDC hazards

**E1. Reconvergence.** Two signals synchronized independently and then
combined in the destination (e.g. `valid` and `mode`, or two bits of a status
word) may arrive one cycle apart. Synchronize them together through D3 or a
FIFO, or make the combination tolerant.

**E2. Divergence.** One source signal synchronized in two places in the same
destination domain: the copies can disagree for a cycle.

**E3. Register-bus fan-out.** Control registers from a host/register-bus
domain fanning out into other clock domains. Trace each register output to its
consumers. Status going back to the register-bus domain (counters, flags,
FIFO levels) follows the same rules, even though the impact is usually lower:
an occasional meaningless value when the status is read.

**E5. Asynchronous inputs from pins.** Every input that isn't timed to an FPGA
clock (buttons, interrupts, status lines, UART RX) goes through a guard before
use [01S8]. It's recommended to debounce mechanical inputs (e.g. pushbuttons) after
synchronization, although letting the noise through is often harmless.

**E6. Resets crossing domains.** Each domain has its own reset synchronizer.
See [resets.md](resets.md) (A3, D9).

## F. Constraints and verification

**F1. Metastability recovery time.** Without special constraints, the path
from a guard register to the next register is timed against the full
destination period. Metastability adds to the guard's clock-to-output delay.
So if that path has nearly zero slack, a short metastable episode becomes a
timing violation. Recommend a tight max-delay on all paths **from** guard
registers, to leave the rest of the period for recovery:
```tcl
set_max_delay -from [get_cells -hier -filter {name=~*_metaguard*}] 0.75
```
Choose the value by lowering it until the tools fail, then raising it until
the slack is small (e.g. 0.2 ns). 0.75 ns is just one device's result. At
250 MHz this leaves more than 3 ns for metastability to resolve.

This is rarely an issue, as the tools usually pack the registers close anyway.
This topic should be a recommendation at most [01S19].

**F2. Bounded skew for multi-bit crossings.** For Gray-coded values (D2) and
custom FIFO pointers, the difference between the bits' path delays must stay
below one source clock period. Constrain the paths **to** the guards with a
datapath-only maximum delay:
```tcl
set_max_delay -datapath_only \
  -from [get_pins -hier -filter {name=~*/C}] \
  -to   [get_pins -hier -filter {name=~*_metaguard*/D}] 1.5
```
In Vivado, `-datapath_only` ignores clock skew and jitter, which have no
meaning between unrelated clocks. The value must be less than the source
clock period. F1 constrains paths *from* the guards and F2 paths *to*
them, so they are separate constraints. Flag a missing F2 on a custom Gray
crossing as a recommendation under Comments [01S20].

**F3. Exceptions that cancel each other.** In Vivado, if a false path or
`set_clock_groups -asynchronous` covers the same paths, it takes precedence and
the `set_max_delay` of F2 probably has no effect. (F1's paths start and end
in the destination domain, so clock groups don't cover them, but a false
path from the guards would) [01S20] [01S6].

