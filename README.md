# Consumable Trackable Resource Plugin with a Cloud Twist

Slurm's `select/cons_tres` plugin can prefer idle CPUs on powered-down cloud
nodes before it tries oversubscribe CPUs on nodes that are already
running. In a cloud environment, that can resume new instances even when current
instances could still accept more shared jobs.

This repository carries patch series for Slurm 25.05, 25.11, 26.05, and 26.11
that make this behavior configurable.

## Behavior

The patches add a new `SelectTypeParameters` option:

```conf
SelectType=select/cons_tres
SelectTypeParameters=PreferPoweredUpNodes
```

When `PreferPoweredUpNodes` is enabled, `select/cons_tres` tests candidate nodes
in this order:

1. Nodes that are neither powered down, powering down, nor powering up.
2. The same set plus nodes that are powering up.
3. The original candidate bitmap, including powered-down and powering-down
   nodes.

The first pass that can satisfy the job wins. If the option is not configured,
Slurm keeps its upstream scheduling behavior.

The implementation skips the multi-pass wrapper for the QoS preemption
extra-row path because that path mutates simulated preemption state while
testing selected preemptees.

## Patch Layout

Each supported Slurm version has a two-patch series under `patches/<version>`:

1. `0001-Make-powered-on-preference-an-option.patch` adds the
   `PreferPoweredUpNodes` config flag, config parsing, config string output, man
   page text, and data parser exposure.
2. `0002-Implement-scheduling-pref-for-powered-on-powering-up.patch` wraps
   `select/cons_tres` job testing with the three-pass power-state preference.

Design notes are available under [docs/plans](docs/plans). The current design is
summarized in [docs/plans/current-design.md](docs/plans/current-design.md).
