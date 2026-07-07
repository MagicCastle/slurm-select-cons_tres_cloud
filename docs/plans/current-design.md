# PreferPoweredUpNodes Current Design

## Goal

Make Slurm `select/cons_tres` prefer capacity on already powered-up nodes before
resuming powered-down cloud nodes, while preserving upstream behavior unless the
administrator opts in.

The public configuration knob is:

```conf
SelectType=select/cons_tres
SelectTypeParameters=PreferPoweredUpNodes
```

## Implemented Patch Series

The repository carries two-patch series for Slurm 25.05, 25.11, 26.05, and
26.11:

1. `0001-Make-powered-on-preference-an-option.patch`
   - Documents `PreferPoweredUpNodes` in `slurm.conf(5)`.
   - Adds the select type parameter bit.
   - Parses `SelectTypeParameters=PreferPoweredUpNodes`.
   - Emits the option from `select_type_param_string()`.
   - Exposes the option through the versioned data parsers present in each
     Slurm branch.
2. `0002-Implement-scheduling-pref-for-powered-on-powering-up.patch`
   - Splits the existing `select/cons_tres` `_job_test()` body into
     `_job_test_internal()`.
   - Adds `_filter_node_bitmap_by_power()`.
   - Adds an outer `_job_test()` wrapper that applies the power-state preference
     when the config option is enabled.

The internal flag name is `SELECT_PREFER_POWERED_UP_NODES`. The config name
remains `PreferPoweredUpNodes` in every patch series.

## Scheduling Behavior

When `PreferPoweredUpNodes` is disabled, `_job_test()` delegates directly to the
original scheduling logic and behavior is unchanged.

When it is enabled, `_job_test()` evaluates the original candidate bitmap in up
to three passes:

1. **Powered-up only:** exclude `POWERED_DOWN`, `POWERING_DOWN`, and
   `POWERING_UP` nodes.
2. **Powered-up plus powering-up:** exclude only `POWERED_DOWN` and
   `POWERING_DOWN` nodes.
3. **All candidates:** use the original bitmap, including powered-down and
   powering-down nodes.

The first pass that returns `SLURM_SUCCESS` wins. If a filtered pass fails, the
wrapper restores the original node bitmap and mutable job fields before trying
the next pass:

- `job_ptr->job_resrcs`
- `job_ptr->details->min_cpus`
- `job_ptr->total_cpus`
- `job_ptr->start_protocol_ver`
- `job_ptr->best_switch`

This preserves the original allocation logic inside each pass and changes only
the order in which power-state groups are considered.

## Preemption Exception

If `qos_preemptees` is present, `_job_test()` skips the multi-pass wrapper and
runs `_job_test_internal()` once with the original node bitmap.

That path mutates simulated row state while testing selected preemptees, so
replaying it across filtered bitmaps would risk changing existing preemption
semantics.

## Non-Goals

- No behavior change unless `PreferPoweredUpNodes` is configured.
- No change to topology plugins.
- No change to node eligibility beyond temporary bitmap filtering in the
  preference wrapper.
- No change to QoS preemption extra-row retry behavior.
