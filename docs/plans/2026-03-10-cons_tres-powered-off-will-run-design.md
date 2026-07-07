# Superseded Will-Run Deferral Design

This document records the first design direction. It has been superseded by the
implemented patch series described in
[current-design.md](current-design.md).

## Original Direction

The initial design targeted only `SELECT_MODE_WILL_RUN` and proposed an
always-on two-pass evaluation:

1. Try a bitmap that excluded powered-down and powering-down nodes.
2. Fall back to the original node bitmap if the first pass failed.

In that version, powering-up nodes were grouped with powered-on nodes.

## Implemented Behavior

The patches now implement an opt-in `SelectTypeParameters` flag named
`PreferPoweredUpNodes`.

When enabled, `select/cons_tres` evaluates `_job_test()` in up to three passes:

1. Powered-up nodes only, excluding powered-down, powering-down, and powering-up
   nodes.
2. Powered-up plus powering-up nodes, excluding powered-down and powering-down
   nodes.
3. The original candidate bitmap.

The behavior applies at the `_job_test()` wrapper level rather than only in
`SELECT_MODE_WILL_RUN`. The original scheduling logic is kept in
`_job_test_internal()` and the wrapper restores mutable job state between failed
filtered passes.

The wrapper is skipped for the QoS preemption extra-row path because that path
mutates simulated preemption state during scheduling tests.
