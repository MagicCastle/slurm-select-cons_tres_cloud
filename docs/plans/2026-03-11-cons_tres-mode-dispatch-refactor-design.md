# Cons_tres Mode Dispatch Refactor Design

## Goal
Simplify `job_test()` mode handling by centralizing two-pass powered-off deferral logic into a single helper, while preserving all existing behavior (including WILL_RUN’s internal two-pass logic) and leaving preemption list mutation behavior unchanged.

## Non-Goals
- No behavior changes in scheduling decisions.
- No changes to preemption candidate mutation/reset behavior.
- No new tests.

## Architecture
Introduce a single helper that performs the correct dispatch for each mode:
- For `SELECT_MODE_WILL_RUN`: delegate directly to `_will_run_test()` (which already performs two-pass powered-off deferral internally).
- For `SELECT_MODE_TEST_ONLY` and `SELECT_MODE_RUN_NOW`: perform the existing two-pass powered-off deferral using `_filter_powered_on_nodes()` and `_run_mode_with_bitmap()`.

Then `job_test()` calls this helper and removes the per-mode branching. This consolidates control flow without altering semantics.

## Components
- `src/plugins/select/cons_tres/job_test.c`: add a new helper (or refactor existing ones) and update `job_test()` to call it.

## Data Flow
`job_test()` → `run_mode_with_deferral()` → (WILL_RUN: `_will_run_test()`), (others: pass1 filtered `_run_mode_with_bitmap()` → fallback `_run_mode_with_bitmap()`).

## Error Handling
No new error cases. `EINVAL` for unknown modes remains unchanged.

## Testing
Skipped per request.

## Open Questions
None.
