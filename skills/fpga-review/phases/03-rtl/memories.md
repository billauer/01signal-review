# Phase 3 — RTL: memories

## Inference

- Block RAM needs a **synchronous read** (registered address or registered
  output). Asynchronous read infers distributed/LUT RAM or registers — fine
  for small memories, bad for large ones.
- Resets on RAM contents prevent inference.
- Memory depth rounded up to a power of two wastes block RAM. Flag as
  Comment.
