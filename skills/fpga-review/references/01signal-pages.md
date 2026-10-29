# 01signal.com pages referenced by this skill set

The phase files tag paragraphs and list items with codes such as `[01S6]`.
Each code stands for one page below. The prefix names the source (`01S` =
01signal.com), so that other sources can get their own prefix and list. The
tag marks the page that a finding based on that text should link to in
the report. How the links appear in the report is described in
[html-report.md](html-report.md).

Scope of a tag:
- A tag inside a paragraph or list item applies to that paragraph or item.
- A tag right after an item's bold title (e.g. `**B1. Metastability
  guard.** [01S19]`) applies to the whole item, up to the next item.
- A tag on its own line right below a heading applies to the whole section
  under that heading, including its subsections.

To add a page, give it the next free number. Don't renumber existing
entries; the phase files refer to them.

## Timing and timing constraints

| Code | Page |
|---|---|
| 01S1 | [The fundamentals of timing in logic design](https://www.01signal.com/constraints/timing/basic-theory/) |
| 01S2 | [The clock period constraint and its timing analysis](https://www.01signal.com/constraints/timing/period-constraint/) |
| 01S3 | [More about the clock period constraint](https://www.01signal.com/constraints/timing/period-constraint-more/) |
| 01S4 | [The clock period constraint and clock objects](https://www.01signal.com/constraints/timing/period-constraint-other/) |
| 01S5 | [Timing constraints and clock domain crossing](https://www.01signal.com/constraints/timing/clock-domain-crossing/) |
| 01S6 | [Timing exceptions](https://www.01signal.com/constraints/timing/timing-exceptions/) |
| 01S7 | [Timing constraints for multi-cycle paths](https://www.01signal.com/constraints/timing/timing-exceptions-advanced/) |
| 01S8 | [Choosing the strategy for I/O timing](https://www.01signal.com/constraints/timing/fpga-io-clocking-methods/) |
| 01S9 | [I/O timing constraints in SDC syntax](https://www.01signal.com/constraints/timing/sdc-input-output-delay/) |
| 01S10 | [Using Tcl commands to select logic elements](https://www.01signal.com/constraints/timing/tcl-filters-paths/) |
| 01S11 | [Using wildcards and -hierarchical in SDC timing constraints](https://www.01signal.com/constraints/timing/logic-elements-wildcards/) |
| 01S12 | [Validating that the timing constraints are correct](https://www.01signal.com/constraints/timing/validation-check-report/) |
| 01S13 | [Strategies for timing closure](https://www.01signal.com/constraints/timing/timing-strategy/) |

## The FPGA as an electronic component (I/O)

| Code | Page |
|---|---|
| 01S14 | [Using registers inside the I/O block](https://www.01signal.com/electronics/iob-registers/) |
| 01S15 | [Source-synchronous inputs](https://www.01signal.com/electronics/source-synchronous-inputs/) |
| 01S16 | [Using 01-signal sampling with source-synchronous inputs](https://www.01signal.com/electronics/01-signal-sampling/) |
| 01S17 | [Source-synchronous outputs](https://www.01signal.com/electronics/parallel-outputs/) |

## Crossing clock domains

| Code | Page |
|---|---|
| 01S18 | [Clock domains, related clocks and unrelated clocks](https://www.01signal.com/verilog-design/clockdomains/introduction/) |
| 01S19 | [Metastability and the basics of clock domain crossing](https://www.01signal.com/verilog-design/clockdomains/crossing-basics/) |
| 01S20 | [Clock domain crossing with data](https://www.01signal.com/verilog-design/clockdomains/crossing-vector/) |

## Resets

| Code | Page |
|---|---|
| 01S21 | [Asynchronous resets on FPGA: Not as easy as many believe](https://www.01signal.com/verilog-design/reset/asynchronous-pitfalls/) |
| 01S22 | [Resets on FPGA: Synchronous, asynchronous or not at all?](https://www.01signal.com/verilog-design/reset/synchronous-vs-asynchronous/) |
| 01S23 | [The logic for starting off and resetting an FPGA properly](https://www.01signal.com/verilog-design/reset/powerup-state-machine/) |

## FIFOs

| Code | Page |
|---|---|
| 01S24 | [Introduction to FPGA FIFOs](https://www.01signal.com/using-ip/fpga-fifo/introduction/) |
| 01S25 | [FPGA FIFOs: Different features and variants](https://www.01signal.com/using-ip/fpga-fifo/variants/) |

## Other topics

| Code | Page |
|---|---|
| 01S26 | [Signed arithmetic in Verilog: The only rule one needs to know](https://www.01signal.com/verilog-design/arithmetic/signed-wire-reg/) |
| 01S27 | [Electronic Exorcism: Why FPGAs sometimes behave as if they are possessed](https://www.01signal.com/intro/fpga-black-magic/) |

## Vendor-specific

| Code | Page |
|---|---|
| 01S28 | [Quartus: The importance of derive_pll_clocks in the SDC file](https://www.01signal.com/vendor-specific/intel/derive-pll-clocks/) |
| 01S29 | [Quartus: Packing registers into I/O cells](https://www.01signal.com/vendor-specific/intel/io-ff-packing/) |
| 01S30 | [Quartus' timing analysis on set_input_delay and set_output_delay constraints](https://www.01signal.com/vendor-specific/intel/quartus-timing-report-io/) |
| 01S31 | [Vivado's timing analysis on set_input_delay and set_output_delay constraints](https://www.01signal.com/vendor-specific/xilinx/vivado-timing-report-io/) |
| 01S32 | [Vivado's timing analysis on set_max_delay and set_min_delay](https://www.01signal.com/vendor-specific/xilinx/vivado-timing-report-io-delay/) |
