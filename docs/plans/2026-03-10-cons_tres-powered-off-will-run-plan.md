# Superseded Will-Run Deferral Plan

This implementation plan has been superseded by the patch series in
`patches/<version>`.

The original plan proposed changing only `SELECT_MODE_WILL_RUN` with an
always-on two-pass powered-off deferral. The implemented patches instead add an
administrator-controlled option:

```conf
SelectTypeParameters=PreferPoweredUpNodes
```

## Current Patch Tasks

The implemented series is split into two patches per supported Slurm branch.

### Patch 1: Make Powered-On Preference an Option

Files touched vary slightly by Slurm version, but the patch consistently:

- Adds `PreferPoweredUpNodes` to the `slurm.conf(5)` `SelectTypeParameters`
  documentation.
- Adds a select type parameter flag.
- Parses `PreferPoweredUpNodes` from `slurm.conf`.
- Includes `PreferPoweredUpNodes` in `select_type_param_string()` output.
- Exposes the flag in the versioned data parser tables present in that branch.

Version notes:

- The internal macro is `SELECT_PREFER_POWERED_UP_NODES`.
- Newer branches expose the flag through newer parser tables such as
  `SELECT_TYPE_PARAM` with a description.

### Patch 2: Implement Powered-Up Preference

The implementation in `src/plugins/select/cons_tres/job_test.c`:

- Renames the original `_job_test()` body to `_job_test_internal()`.
- Adds `_filter_node_bitmap_by_power(node_bitmap, include_powering_up,
  include_powered_down)`.
- Adds a new `_job_test()` wrapper that runs the original scheduling logic in
  power-state preference order when the option is enabled.

The pass order is:

1. Exclude powered-down, powering-down, and powering-up nodes.
2. Exclude powered-down and powering-down nodes.
3. Use the original node bitmap.

Between failed filtered passes, the wrapper restores the original node bitmap and
mutable job fields before retrying.

The wrapper delegates directly to `_job_test_internal()` when either condition is
true:

- `PreferPoweredUpNodes` is not configured.
- `qos_preemptees` is non-null.

## Testing Notes

No unit tests are currently carried in the patch series. Manual review should
focus on:

- Config parsing and string output for `PreferPoweredUpNodes`.
- Data parser exposure for the relevant branch.
- Correct pass ordering in `_job_test()`.
- Restoration of job state before fallback passes.
- Preservation of the QoS preemption extra-row path.
