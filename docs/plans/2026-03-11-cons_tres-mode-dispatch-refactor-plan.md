# Superseded Mode Dispatch Refactor Plan

This plan has been superseded by the implemented patch series in
`patches/<version>`.

## What Changed

The plan originally proposed adding mode helpers such as `_run_mode_with_bitmap()`
and `_run_mode_with_deferral()` to centralize `SELECT_MODE_*` dispatch.

The implemented patches do not add those helpers. They instead:

- Add a config option named `PreferPoweredUpNodes`.
- Preserve upstream behavior unless that option is enabled.
- Rename the original `select/cons_tres` `_job_test()` implementation to
  `_job_test_internal()`.
- Add a new outer `_job_test()` that evaluates filtered node bitmaps in
  power-state order.

## Implemented Flow

With `PreferPoweredUpNodes` enabled:

1. Try `_job_test_internal()` after filtering out powered-down, powering-down,
   and powering-up nodes.
2. If that fails, restore mutable job state and try `_job_test_internal()` after
   filtering out powered-down and powering-down nodes.
3. If that fails, restore mutable job state and try `_job_test_internal()` with
   the original node bitmap.

With the option disabled, `_job_test()` calls `_job_test_internal()` once.

With `qos_preemptees` present, `_job_test()` also calls `_job_test_internal()`
once so the QoS preemption extra-row behavior is not replayed across filtered
passes.

## Current Source of Truth

Use [current-design.md](current-design.md) and the patch files under
`patches/<version>` as the current implementation reference.
