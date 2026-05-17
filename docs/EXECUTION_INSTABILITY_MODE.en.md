# Execution Instability Mode

Korean version: [EXECUTION_INSTABILITY_MODE.md](EXECUTION_INSTABILITY_MODE.md)

## Overview

`execution_instability` is the Argus v1 research observation mode for recording execution-level variability and redundant-computation signals.

The mode keeps the standard Argus v1 artifacts and adds a schema-specific artifact, `instability_metrics.json`, for path, reuse, recomputation, and work-inflation observations.

Argus records these values as non-causal observation proxies. It does not claim that path variability causes reuse failure, that reuse failure causes recomputation, or that recomputation directly causes latency changes.

## Usage

CLI override:

```bash
argus run config.yaml --mode execution_instability
```

Config-driven mode:

```yaml
seed: 42
steps: 500
repeat: 5
warmup_steps: 20
mode: execution_instability
ricci: [off, on]
workload:
  nodes: 8
  requests_per_step: 128
```

## Output Contract

An `execution_instability` run writes the standard Argus v1 record:

```text
run_dir/
|-- metrics.json
|-- report.md
|-- run_meta.json
|-- config.resolved.yaml
`-- instability_metrics.json
```

`argus export <run_dir>` includes `instability_metrics.json` when the file exists. `argus export <run_dir> --sanitize` applies the same redaction policy before packaging.

## Schema

Top-level structure:

```json
{
  "schema_version": "0.1.0",
  "mode": "execution_instability",
  "run_info": {},
  "system_info": {},
  "workload_info": {},
  "scenario_info": {},
  "measurement_window": {},
  "raw_metrics": {},
  "derived_metrics": {},
  "artifacts": {},
  "notes": {}
}
```

Raw metrics include:

- `timing.iteration_latency_ms`
- `timing.total_runtime_ms`
- `work_observation.total_operations_observed`
- `work_observation.repeated_operations_observed`
- `work_observation.reference_operations`
- `reuse_observation.reuse_opportunities_observed`
- `reuse_observation.reuse_failures_observed`
- `reuse_observation.intermediate_rebuild_count`
- `path_observation.distinct_path_count`
- `path_observation.ordering_change_count`
- `memory_observation.cache_miss_proxy`
- `memory_observation.reload_events_observed`

Derived metrics include:

- latency summary: mean, p95, p99, standard deviation
- `execution_path_variability_score`
- `reuse_failure_rate`
- `recompute_ratio`
- `work_inflation_ratio`

## Interpretation Boundary

The mode is designed for reproducible observation, not optimization or causal attribution.

The schema is intentionally proxy-based because complete execution paths and complete intermediate-state reuse are not always directly observable across CPU, GPU, and runtime environments.

## Relationship to Execution-Instability Benchmarks

Argus is the observation and reporting layer. It runs the validation protocol, records counters and environment context, writes schema-compliant artifacts, regenerates reports, and packages results.

External execution-instability benchmarks can provide controlled workloads and scenario definitions as long as their observations can be represented through the same `instability_metrics.json` schema.
