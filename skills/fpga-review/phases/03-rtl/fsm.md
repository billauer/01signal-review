# Phase 3 — RTL: state machines

## Structure and encoding

- The state register must be reset explicitly to a defined state (see
  [resets.md](resets.md), A1–A2: with an asynchronous reset whose release
  isn't controlled, FSMs must not leave the reset state on the first clock
  after the release) [01S21].
- The synthesis tool may re-encode the state (one-hot, Gray, binary) and
  remove "unreachable" states. Behavioural code that looks self-recovering
  (`default: state <= IDLE`) is **not** recovery logic after re-encoding
  [01S22].
- Hand-coded one-hot FSMs can reach invalid encodings (zero or several hot
  bits). Check whether some logic detects this. If not, suggest simple logic
  that counts the active bits and resets the FSM if the count isn't 1. With
  many state bits, pipeline the count; it doesn't matter if this reset
  arrives a few clock cycles late.

## Behaviour

- Inputs from other domains that affect the state registers must be synchronized
  before they reach the FSM (one unsynchronized input can send different state
  bits in different directions). Flag as Notification.
- Unless all possible state variable values are explicitly covered, flag
  missing `default` in `case` (or `when others` in VHDL) in the next-state
  logic as Notification.
