# Consumable Trackable Resource Plugin with a Cloud Twist

Slurm's resource selection algorithm in the cons_tres plugin operates in a sequence of prioritized "steps."

1. Step 1: Seek Idle Resources: Slurm first attempts to find nodes that have enough idle (unallocated) cores and memory to satisfy
    the job's request.
    * Powered-on nodes that are already running jobs may have few or no idle cores left.
    * Powered-off nodes are treated as having all of their cores idle and available for allocation (once powered on).
2. Steps 2-4: Seek Shared/Overcommitted Resources: Only if Step 1 fails (meaning there aren't enough idle cores anywhere in the
    cluster, including powered-off nodes) will Slurm proceed to look for resources that are already in use but could be shared via
    OverSubscribe (oversubscription).

Because Step 1 succeeds by finding idle cores on powered-off nodes, Slurm stops there and selects those nodes. 
It prefers the "cost" of powering on a node over the "cost" of CPU contention and reduced performance that 
comes with overcommitting/sharing cores on a powered-on node.

This results in Slurm powering nodes instead of overcommiting CPUs, which in a cloud
environment is not always ideal. In our specific case, we want Slurm to overcommit
every powered-on nodes up to the oversubscription limit before powering on more nodes.

At the moment, Slurm provide a configuration mechanism to enable this behavior.

## Our solution

With the help of gpt-5.2-codex and [superpowers](https://github.com/obra/superpowers),
we tracked down the node selection logic to a single file `job_test.c` and proceeded
to implement a two pass filter when selecting nodes. First pass is done with only
powered on nodes, and if no nodes can satisfy the job requirements, a second pass is
accomplished with every nodes.

The superpowers plan used to produced the modified `job_test.c` for the supported
Slurm versions are available under [docs/plans](docs/plans).