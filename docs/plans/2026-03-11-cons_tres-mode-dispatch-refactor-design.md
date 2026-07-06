# Superseded Mode Dispatch Refactor Design

This design has been superseded by the implemented patch series described in
[current-design.md](current-design.md).

## Original Direction

The earlier design centralized powered-off deferral around mode dispatch in
`job_test()`. It treated `SELECT_MODE_WILL_RUN`, `SELECT_MODE_TEST_ONLY`, and
`SELECT_MODE_RUN_NOW` separately and proposed helper functions around those
modes.

## Implemented Direction

The current patches avoid mode-specific dispatch refactoring. Instead, they keep
the existing selection behavior intact inside `_job_test_internal()` and place
the power-state preference in an outer `_job_test()` wrapper.

When `SelectTypeParameters=PreferPoweredUpNodes` is enabled, the wrapper tries:

1. Powered-up nodes only.
2. Powered-up plus powering-up nodes.
3. The original candidate bitmap.

When the option is disabled, or when `qos_preemptees` is present, `_job_test()`
delegates directly to `_job_test_internal()`.

This approach changes power-state preference order without restructuring
`SELECT_MODE_*` handling.
