# Phase 0 — Inventory

Do this first, do not skip. Build a map of the project before judging anything.
Record it in your working notes; every later phase uses these tables.

## 1. Identify the toolchain and project files

Find out which tool(s) build the design and how the build is driven. Typical
markers:

- AMD Vivado: `*.xpr`, non-project Tcl scripts (`read_verilog`, `synth_design`),
  `*.xci`/`*.xcix`, block designs `*.bd`, constraints `*.xdc`.
- Altera/Intel Quartus: `*.qpf`, `*.qsf`, Platform Designer `*.qsys`/`*.ip`,
  constraints `*.sdc`.
- Simulation: testbenches, simulation scripts and file lists.

Note the tool **versions** if recorded (project files, scripts, IP metadata,
README).

## 2. Classify every file

- **User RTL** (`*.v`, `*.sv`, `*.svh`, `*.vh`, `*.vhd`, `*.vhdl`).
  **Primary review target.**
- **Constraints** (timing and physical, all formats). **Primary review target.**
- **IP configuration** (`*.xci`, `*.xco`, `*.ip`, `*.qsys`, block designs, Tcl that
  generates IP). Review parameters and wrappers.
- **Generated IP HDL** (FIFO/RAM/PLL generator output, simulation models,
  `*_stub.*`, `*_funcsim.*`, `synth/`, `sim/` subdirectories). **Do not
  review** except to check it is consistent with its configuration.
- **Black boxes**: netlists and encrypted sources (`*.edf`, `*.edn`,
  `*.dcp`, `*.qxp`, `*.vqm`, `` `pragma protect `` blocks). Read
  their documentation, templates and README files to learn the port contract.
- **Third-party / vendor cores** delivered as source: treat like IP — check the
  contract, not the internals, unless asked.
- **Testbenches** and simulation scripts. Note which modules have them.
- **Build scripts** and hooks.
- **Documentation**: specs, block diagrams, board schematics
  or pin lists. Note what exists; it is the reference for many checks.

## 3. Identify the target device

Family, part number, package, speed grade, temperature grade. Typical
sources: project file, `set_part`/`set_global_assignment -name DEVICE`, IP
configuration files, Makefile arguments. **Check they agree** across project, IP and scripts; an IP
generated for another part or speed grade is a minor finding to be mentioned
very briefly in the review report.

If it's impossible to determine which device is targeted, say so in the report,
and don't make any assumptions about it.

## 4. Find the top module and walk the hierarchy

- Which top-level entity/module is built (project setting, script argument)?
  Is there more than one candidate?
- List modules instantiated but whose source is missing (cannot review —
  report it).
- List modules/IP present but never instantiated (dead code — flag as Comment).
- List duplicated modules (byte-identical or near-identical files, or the same
  module name defined in two files — which one is used depends on compile
  order). Make a list, flag as Comment.
- Note `generate` blocks and parameter-dependent structure: which branches are
  active for the parameters actually used?

If you can't establish beyond any doubt **exactly** which files are part of
the FPGA project, **STOP AND ASK THE USER**.

## 5. Clock table

Every clock source: top-level ports, differential input buffers, PLL / MMCM
outputs, transceiver recovered and reference clocks, internal
oscillators, configuration clocks, clocks from hard IP (PCIe, Ethernet,
memory controllers, processor subsystems), fabric-generated clocks (counters,
logic). For each, record:

| Clock | Source | Intended frequency | Buffer / routing | Constraint (`create_clock` / generated / IP-supplied) | Domains it clocks |
|---|---|---|---|---|---|

## 6. Reset table

Every reset source (pins, PLL lock, IP status outputs, software, watchdog,
power-on logic), polarity, synchronous/asynchronous, which domains it reaches,
how it is synchronized and released.

## 7. Clock-domain crossing list (preliminary)

From the clock table: every signal, bus, FIFO and handshake that moves between
domains, and how it is synchronized. Completed in Phase 3 (CDC topic).

## 8. External interfaces

Group the top-level ports by external device or connector: protocol,
direction, I/O standard, clocking (system-synchronous, source-synchronous,
asynchronous, embedded clock), data rate. Used in Phase 2 (I/O timing).

## 9. IP inventory

For each IP instance: vendor, name, version, configuration file, key
parameters, and whether the IP ships its own constraints.

## 10. Build and simulation artefacts

Which reports, logs and checkpoints exist, from which tool version, and how
old they are compared to the sources. Decide whether later phases can rely on
them.

## Useful searches

Adapt globs to the tree; search the whole tree, not only a `src/` directory.

```sh
# Clocked processes
grep -rnE '\balways(_ff|_comb|_latch)?\b' --include='*.v' --include='*.sv' .
grep -rniE '\bprocess\b|rising_edge|falling_edge' --include='*.vhd' --include='*.vhdl' .

# Clocking, I/O and special primitives (extend with the vendor's names)
grep -rnE 'MMCM|PLL|BUFG|BUFH|BUFR|BUFIO|CLKCTRL|IBUFDS|ODDR|IDDR|SERDES|DELAY|GT[XHYP]|OSC|STARTUP|ICAP|XADC|SYSMON' .

# Timing and physical constraints, any format
grep -rnE 'create_clock|create_generated_clock|derive_|set_clock_groups|set_false_path|set_max_delay|set_min_delay|set_multicycle_path|set_input_delay|set_output_delay|set_max_skew|set_net_delay' \
     --include='*.xdc' --include='*.sdc' --include='*.tcl' .
grep -rnE 'PACKAGE_PIN|IOSTANDARD|LOCATION|IO_STANDARD|IOBUF|set_location_assignment|set_instance_assignment' .

# Synthesis attributes and pragmas
grep -rnE '\(\*|attribute |synthesis|pragma|translate_(on|off)|ASYNC_REG|DONT_TOUCH|KEEP|preserve' .
```
