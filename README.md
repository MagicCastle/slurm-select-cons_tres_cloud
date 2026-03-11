# Consumable Trackable Resource Plugin with a Cloud Twist

When scheduling jobs, [cons_tres](https://slurm.schedmd.com/cons_tres.html)
ranks nodes by sched_weight and then by “least‑loaded” (highest available/total CPU ratio).
In an environment where oversubscription and power-saving are activated,
powered‑down nodes are eligible, and because they appear fully idle, they win 
the least‑loaded tie‑break over powered‑on nodes that would require overcommit.
Oversubscription itself is not a scheduling weight penalty.

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