# Argus v1 Experiment Rules

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

## Core fields (`config.yaml`)

- `seed`
- `steps`
- `repeat`
- `warmup_steps`
- `ricci: [off, on]`
- `workload.nodes`
- `workload.requests_per_step`

## Warmup Policy

Warmup steps are executed but excluded from all final statistics.

This removes startup effects such as:

- runtime initialization
- cache stabilization
- allocator/JIT warmup

## Statistical output requirements

For each metric, Argus records:

- Mean
- Std Dev
- Min
- Max

This is used to determine whether differences are repeatedly observable or noise-dominated.
