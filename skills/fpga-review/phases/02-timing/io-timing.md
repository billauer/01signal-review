# Phase 2 — Timing: I/O timing

Go through the external interfaces list from Phase 0, one interface at a time.

## First, the clocking strategy

[01S8]

Constraints written without a clocking strategy for the interface may be
worthless. For each interface, identify the strategy, then check that the
logic and the constraints fit it.

- **System-synchronous** (FPGA and device share a board clock): delays derived
  from the external device's tCO / tSU / tH plus board trace delays and clock
  skew. For Vivado, verify that set_input_delay (for input pins) or set_output_delay
  (for output pins) is defined. In either case, verify that two set_X_delay
  constraints are defined for each pin, one with -max and one with -min
  [01S9]. Flag as Notification otherwise.
- **Source-synchronous** (clock travels with data). Identify the case:
  - FPGA sends clock and data (outputs only): clock forwarded through an
    output DDR register.
  - Device sends clock and data (inputs): input relative to a clock
    defined on the incoming clock pin, where the PCB should be designed
    for a minimal skew between clock and data.
  - FPGA sends the clock, data is input or bidirectional: the input timing
    must account for the round trip of the clock through the PCB.
  - Device sends the clock, data is bidirectional: inputs are
    source-synchronous, but **outputs are constrained as system-synchronous**
    to that clock [01S17].
  Also check:
  - source-synchronous **inputs**: see "Clocking source-synchronous inputs"
    below; source-synchronous **outputs**: see "Source-synchronous outputs".
- **Asynchronous inputs** (buttons, I2C, UART RX, status lines from other
  chips): a guard register plus a second register for every input; a single
  flip-flop is a common mistake (see [../03-rtl/cdc.md](../03-rtl/cdc.md), B1
  and E5).
- **Embedded clock / transceivers and hard-wired interfaces** (gigabit
  transceivers, pins tied to a hard processor): constrained by the IP; no
  user I/O constraints expected.

## Clocking source-synchronous inputs

[01S15]

An external clock may stop, glitch, pick up noise through a loose connector,
or have an irregular period or high jitter (typical for clocks from cables,
cameras, and microprocessor I/O peripherals). Logic clocked by such a clock
can reach states that look impossible from the HDL, and only a reset
recovers it. Identify which strategy is used and check it:

- **01-signal sampling** (most robust; only for relatively low data rates):
  the external clock is treated as data and sampled by a stable internal
  clock. See the next section.
- **PLL**: the external clock feeds a PLL, and the logic runs on the PLL
  output. Check that the logic is **held in reset while the PLL is not
  locked**.
- **Phase shifting / calibration** (data rates near the I/O maximum, e.g.
  DDR memory): static timing can't guarantee capture here, so a state
  machine searches for the best sampling delay at run time, using a known
  pattern. Check that the calibration exists and is rerun when needed;
  the internal clock must be guaranteed to have the same frequency as the
  data rate. Constraints may still serve to keep I/O registers in place.
- **External clock used directly** on internal logic: usually a poor
  choice. Acceptable only if the clock is guaranteed clean and stable;
  otherwise flag as Notification, and in any
  case check that a reset covers start-up while the clock is unstable.
  Clock-pin-to-flip-flop delay may need dedicated clock pins and regional
  clock resources.

The PLL and direct-clock strategies are constrained like system-synchronous
clocks, relative to a clock defined on the input pin.

## 01-signal sampling

[01S16]

- Structure: data and the external clock both pass through a guard register
  and a sampling register on the internal clock; a rising edge is detected as
  `clk_samp && !clk_samp_d` (falling edge, `1 0`, if the data changes on the
  rising edge), and that strobe captures the sampled data.
- Registers in the I/O cells keep the skew small. set_max_delay constraints
  (or similar with other tools) with tight timing requirements leave the tools
  little choice but to use I/O registers, and no freedom to insert extra
  delays in the I/O cells. Synthesis directives (e.g. IOB = "TRUE") are also
  recommended, even though they aren't always necessary.


## Source-synchronous outputs

[01S17]

- **Forwarded clock through an output DDR primitive**, instantiated (e.g.
  AMD `ODDR`/`ODDRE1`).
  Behavioural "flip-flop on both edges" code rarely synthesizes into one.
  A clock toggled by a plain register runs at half the frequency and halves
  the data rate.
- Data: through I/O-cell registers, or through output DDR primitives with
  the same value on both inputs (strongest alignment). If the clock goes
  through an ODDR and the data through plain I/O registers, the alignment is
  not obvious.
- Output pins physically close together (pins far apart add internal clock
  skew) and electrically identical: same I/O standard, slew rate and drive
  strength.
- Without I/O-cell registers, constraints are essential: they must keep the
  register-to-pin delay small and equal on all outputs.
- **Bidirectional data**: incoming data is aligned to the clock at the
  PCB, which differs from the internal clock by the clock-to-output delay
  (several ns). Look for a **clock loopback** (forwarded clock routed back to
  an input pin, used as a system-synchronous clock), or treat the inputs as
  source-synchronous with phase-shift calibration (the usual way for DDR
  SDRAM). Without either, the input timing is likely unjustified: flag as
  Notification, since it is usually only fixable on the PCB.

## Every I/O is either timed or explicitly excepted

- Inputs and outputs without `set_input_delay` / `set_output_delay` **and**
  without an explicit exception are a finding. Keep the unconstrained I/O list
  empty, so a missing constraint stands out [01S12].
- **False paths only when the port's timing is completely meaningless**
  (LEDs, DIP switches, truly asynchronous inputs that are synchronized in
  RTL): `set_false_path -to [get_ports led_0]`. Flag a false path on a port
  with real timing requirements. A blanket false path on all I/O ports
  (used to silence unconstrained-path warnings) is a finding [01S8] [01S12].

## `set_input_delay` / `set_output_delay`

[01S9]

- The reference clock is the one defined on the **port** (`get_ports`), not on
  an internal pin (see timing-constraints, primary clocks).
- **Both `-max` and `-min` are given** for every constraint. Without them,
  the tool fills in defaults that are probably not wanted. E.g. in Vivado,
  `set_output_delay -clock clk 8 [get_ports x]` sets both min and max to 8,
  which implies a hold requirement of -8 ns. Other tools' defaults may
  differ; the recommendation to give both applies everywhere.
- Meaning of the values:
  - input `-max` = external tCO(max) + maximum trace delay;
  - input `-min` = external tCO(min) (zero if the data sheet gives none) +
    minimum trace delay;
  - output `-max` = external tSU + maximum trace delay;
  - output `-min` = **−tH** of the receiver + minimum trace delay. It is
    usually negative; a positive output `-min` larger than the trace delay
    usually means the sign was missed. For how the tools apply these
    values, see [01S31] (Vivado) and [01S30] (Quartus).
- **Input hold without clock compensation**: when the capturing clock comes
  from the clock pin without a PLL compensating the clock network delay, the
  clock reaches the register nanoseconds after the data. Input hold can then
  fail even with a positive external tCO(min), unless constrained properly
  with a set_input_delay -min (or similar with other tools).
- **Output hold**: a PLL that shifts the internal clock earlier (to meet output
  setup) can make outputs change before the external clock edge, violating
  the receiver's hold time. Only a correct `set_output_delay -min` catches this.
- **Trial and error** (flipping clock edges or phase shifts until the
  interface works in the lab) is not a timing strategy. The result can change
  with temperature and process. Flag interfaces that look tuned that way
  without constraints that justify the setting.

## `set_max_delay` / `set_min_delay` on I/O

[01S9]

- In Vivado, on outputs they bound the FPGA's clock-to-output (port treated
  as an ideal flip-flop). Other tools' interpretation of max/min delay on
  ports may differ. They don't scale with the clock period, while
  `set_input_delay`/`set_output_delay` do. Use them only deliberately (e.g. to
  force minimal I/O delay), not for system-synchronous interfaces.
- On inputs, `set_max_delay` without `-datapath_only` is analyzed by Vivado,
  but its requirement (measured against the capturing register's clock
  arrival) has no intuitive meaning. Use `set_input_delay`.
- `-datapath_only` (Vivado) on I/O for "as short as possible" paths; the
  output form needs `-from [all_registers]`. It drops the clock network
  delay, so the result can be more or less pessimistic than the full
  analysis; it bounds a delay, not the interface's timing relationship
  [01S32].
- A forwarded clock generated by an output DDR register is a path from the
  clock, not from a register: in Quartus, e.g.
  `set_max_delay -from [get_clocks main_clk] -to [get_ports sram_clk]`
  (`set_output_delay` covers it automatically) [01S29].

## Making I/O timing repeatable

[01S14]

- Input and output registers packed into the I/O cells where the interface
  needs consistent timing (Vivado `IOB` property; Quartus
  `FAST_INPUT_REGISTER` / `FAST_OUTPUT_REGISTER` /
  `FAST_OUTPUT_ENABLE_REGISTER`; vendor I/O register primitives). Otherwise timing changes from build to
  build, and a design may appear "fixed" or "broken" by an unrelated change.
  Treat I/O-cell
  registers as the default for synchronous I/O; exceptions should be
  deliberate.
- A register whose output also feeds fabric logic can't be packed as is
  (an I/O-cell register drives only the pin), so it must be **replicated**.
  A constraint-file property alone may not achieve this; a synthesis
  attribute is often needed (e.g. Vivado `(* IOB = "TRUE" *)` on the
  register) [01S14].
- Packing can be lost silently after a logic change; tools rarely warn.
  A register that also drives other logic (e.g. an output enable that is
  also used internally) is a common cause (see replication above).
- Ideally the I/O constraints are tight enough to be met **only** with the
  register in the I/O cell, so that losing it fails timing. Tightening them
  further should make timing fail; if not, the constraint isn't doing what's
  intended.
- Some tools insert a delay between the pin and the input register to gain
  hold margin when the input constraint leaves room for it (Quartus: visible
  as "Input Pin to Input Register Delay" in the Resource Property Editor).
  This can hide a constraint that is too loose. Where it matters, tighten the
  input constraint until it fails, then loosen it just enough to pass, or set
  the delay explicitly [01S29].
