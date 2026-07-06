# Cons_tres Powered-Off Deferral and Mode Dispatch Design

## Goal
Prefer already powered-on nodes before considering powered-off nodes in the `cons_tres` selection path. A powered-off node is considered only if the job cannot be satisfied by powered-on nodes, including by overcommit where allowed.

Centralize this behavior in `job_test()` mode dispatch so `SELECT_MODE_WILL_RUN`, `SELECT_MODE_TEST_ONLY`, and `SELECT_MODE_RUN_NOW` use the same powered-off deferral policy while preserving each mode's existing scheduling semantics.

"Powered-on" includes `POWERING_UP` nodes and excludes `POWERED_DOWN` and `POWERING_DOWN` nodes.

## Non-Goals
- No change to node eligibility rules beyond filtering powered-off nodes in the first pass.
- No change to topology plugins or node scheduler logic.
- No change to preemption candidate mutation/reset behavior.
- No change to `RUN_NOW` behavior when preemption candidates are present; that path must run once with the original node bitmap because preemption handling mutates candidate state.

## Architecture
Use two levels of dispatch:

1. `_will_run_test()` owns the `SELECT_MODE_WILL_RUN` two-pass evaluation.
2. `_run_mode_with_deferral()` owns top-level mode dispatch from `job_test()` and applies powered-off deferral to non-`WILL_RUN` modes where it is safe to do so.

The helper `_filter_powered_on_nodes()` builds a filtered copy of the input bitmap by clearing `POWERED_DOWN` and `POWERING_DOWN` nodes. The original bitmap remains available for fallback.

For `SELECT_MODE_WILL_RUN`:
1. Run will-run using the filtered powered-on bitmap.
2. If Pass 1 succeeds, copy the selected filtered result back to the caller's bitmap and return success.
3. If Pass 1 fails, run will-run using the original bitmap.

For `SELECT_MODE_TEST_ONLY`:
1. Run test-only using the filtered powered-on bitmap.
2. If Pass 1 succeeds, copy the selected filtered result back to the caller's bitmap and return success.
3. If Pass 1 fails, run test-only using the original bitmap.

For `SELECT_MODE_RUN_NOW` without preemption candidates:
1. Run run-now using the filtered powered-on bitmap.
2. If Pass 1 succeeds, copy the selected filtered result back to the caller's bitmap and return success.
3. If Pass 1 fails, run run-now using the original bitmap.

For `SELECT_MODE_RUN_NOW` with preemption candidates:
1. Skip the filtered first pass.
2. Run run-now once using the original bitmap.

Skipping the filtered pass in the preemption case preserves existing preemption behavior because `_run_now()` can reorder `preemptee_candidates` and update per-job `usable_nodes` state while searching for a valid preemption set.

## Components
- `src/plugins/select/cons_tres/job_test.c`: add `_filter_powered_on_nodes()`, split will-run into `_will_run_test()` and `_will_run_test_with_bitmap()`, add mode dispatch helpers, and update `job_test()` to call `_run_mode_with_deferral()`.
- `testsuite/slurm_unit/backfill/backfill_test.c`: add unit tests for will-run powered-off deferral where available.

## Data Flow
`job_test()` -> `_run_mode_with_deferral()`:

- `SELECT_MODE_WILL_RUN` -> `_will_run_test()` -> filtered `_will_run_test_with_bitmap()` -> fallback `_will_run_test_with_bitmap()`.
- `SELECT_MODE_TEST_ONLY` -> filtered `_run_mode_with_bitmap()` -> fallback `_run_mode_with_bitmap()`.
- `SELECT_MODE_RUN_NOW` without preemption candidates -> filtered `_run_mode_with_bitmap()` -> fallback `_run_mode_with_bitmap()`.
- `SELECT_MODE_RUN_NOW` with preemption candidates -> original bitmap `_run_mode_with_bitmap()`.

## Error Handling
- No new error cases.
- Pass 1 failure falls back to the original bitmap.
- Original `node_bitmap` must remain available for fallback and must only be overwritten with filtered results after a successful filtered pass.
- `EINVAL` for unknown modes remains unchanged.

## Testing
- Powered-on node can satisfy a `WILL_RUN` request with overcommit while a powered-down node is also available. Expect will-run to choose the powered-on solution.
- Only powered-down nodes can satisfy a `WILL_RUN` request. Expect will-run success using powered-down nodes.
- `RUN_NOW` with non-empty preemption candidates should call `_run_now()` once with the original bitmap, preserving candidate ordering and `usable_nodes` side effects.

## Open Questions
None.
