# Cons_tres Will-Run Powered-Off Deferral Design

## Goal
When running in `SELECT_MODE_WILL_RUN`, only consider powered-off nodes if there are zero powered-on nodes that can satisfy the job (including with overcommit). "Powered-on" includes `POWERING_UP` but excludes `POWERING_DOWN`.

## Non-Goals
- No change to immediate scheduling behavior outside `SELECT_MODE_WILL_RUN`.
- No change to node eligibility rules beyond filtering powered-off nodes in the first pass.
- No change to topology plugins or node scheduler logic.

## Architecture
In the `SELECT_MODE_WILL_RUN` path, perform a two-pass eligibility evaluation using existing `cons_tres` will-run logic:
1. Pass 1: run will-run using a node bitmap filtered to exclude `POWERED_DOWN` and `POWERING_DOWN` nodes (include `POWERING_UP`).
2. If Pass 1 succeeds, return those results and do not consider powered-off nodes.
3. If Pass 1 fails, run Pass 2 using the original node bitmap (current behavior).

## Components
- `src/plugins/select/cons_tres/job_test.c`: update `_will_run_test()` to perform two-pass evaluation and add a helper to build the powered-on eligible bitmap.
- `testsuite/slurm_unit/backfill/backfill_test.c`: add unit tests for the will-run powered-off deferral behavior.

## Data Flow
1. `_will_run_test()` receives `node_bitmap`.
2. Create `node_bitmap_on` as a filtered copy excluding `POWERED_DOWN` and `POWERING_DOWN` nodes.
3. Execute the existing will-run logic with `node_bitmap_on`.
4. If success: return (powered-on only).
5. If failure: execute the existing will-run logic with the original `node_bitmap`.

## Error Handling
- No new error cases.
- Pass 1 failure falls back to Pass 2.
- Original `node_bitmap` must remain unmodified.

## Testing
- Test 1: powered-on node can satisfy job with overcommit, powered-down node also available. Expect will-run to choose powered-on solution (powered-down not selected).
- Test 2: only powered-down nodes can satisfy job. Expect will-run success using powered-down nodes.

## Open Questions
None.
