# Argus Validation Protocol v1.0

![Argus Full Logo](argus_logo/argus_full_logo.png)
![status](https://img.shields.io/badge/status-v1.0.0--validation-blue)
![platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS-lightgrey)
![license](https://img.shields.io/badge/license-Apache%202.0-green)

Argus is an execution observatory protocol for reproducible validation.

It does not claim absolute performance.
It records whether structural behavior changes are repeatedly observable under identical conditions.

## What Argus Outputs

Argus output is a **Reproducible Observation Record**:

- `metrics.json`
- `report.md`
- `run_meta.json`

Optional submission package:

- `argus_result.zip`
- `argus_result.sanitized.zip` (`--sanitize`)

## Try it in 10 seconds

Linux (amd64):

```bash
curl -L https://github.com/tongro2025/Argus/releases/latest/download/argus-linux-amd64 -o argus
chmod +x argus
./argus --help
```

macOS (Apple Silicon):

```bash
curl -L https://github.com/tongro2025/Argus/releases/latest/download/argus-macos-arm64 -o argus
chmod +x argus
./argus --help
```

## Core CLI (v1)

```bash
argus doctor
argus run <config.yaml>
argus report <run_dir>
argus export <run_dir>
argus export <run_dir> --sanitize
```

## Example config.yaml

```yaml
seed: 42
steps: 500
repeat: 5
warmup_steps: 20
ricci: [off, on]
workload:
  nodes: 8
  requests_per_step: 128
```

Warmup steps are executed but excluded from final statistics.

## What gets measured

- P95 latency, P99 latency, throughput
- migration / retry / invalidation count
- error_rate, oom_count, deadlock_count, crash_count
- peak_memory_usage, avg_memory_usage
- per-metric `Mean`, `Std Dev`, `Min`, `Max`

## Environment reliability checks

`argus run` records runtime noise context and warns when needed:

- `cpu_load_percent`
- `memory_usage_percent`
- `gpu_temperature`
- `gpu_utilization`
- `thermal_throttle_detected`

If noise is high, Argus shows:

```text
[Argus Warning]
Environment noise detected.
Results may be unreliable.
Continue? (y/n)
```

## Sanitized export

Use `argus export <run_dir> --sanitize` when sharing results.

Sanitization removes/redacts:

- username
- absolute user-home path
- hostname
- env assignments
- IP address

## Documentation

English:

- [Validation Protocol](docs/VALIDATION_PROTOCOL.en.md)
- [Experiment Rules](docs/EXPERIMENT.en.md)
- [Claims and Limits](docs/CLAIMS.en.md)
- [Release Binary Workflow](docs/Release_binaries.en.md)

한국어:

- [검증 프로토콜](docs/VALIDATION_PROTOCOL.md)
- [실험 규칙](docs/EXPERIMENT.md)
- [주장과 한계](docs/CLAIMS.md)
- [릴리즈 바이너리 워크플로우](docs/Release_binaries.md)

## License

Licensed under Apache License 2.0.

- [LICENSE](LICENSE)
- [NOTICE](NOTICE)
