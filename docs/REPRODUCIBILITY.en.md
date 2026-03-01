# Argus Reproducibility Guide

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

## 1. Purpose

This document defines a reproducibility protocol for Argus measurement records.

What this document proves:

- execution observation can be recorded repeatedly under the same protocol
- measurement artifacts can be generated and preserved
- reports can be regenerated from existing records

What this document does not prove:

- application speed improvement
- hardware optimization effects
- model quality change

## 2. Supported Environment

- OS: Linux, macOS
- Architecture: x86_64, arm64
- Minimum CPU: 2 cores
- Minimum Memory: 4 GB RAM
- GPU: not required

No special dataset or model is required.

## 3. Installation (Prebuilt Binary)

Do not build from source for this protocol. Use prebuilt binaries.

Linux (x86_64):

```bash
curl -L https://github.com/tongro2025/Argus/releases/latest/download/argus-linux-amd64 -o argus
chmod +x argus
mkdir -p "$HOME/.local/bin"
mv ./argus "$HOME/.local/bin/argus"
export PATH="$HOME/.local/bin:$PATH"
```

macOS (Apple Silicon / arm64):

```bash
curl -L https://github.com/tongro2025/Argus/releases/latest/download/argus-macos-arm64 -o argus
chmod +x argus
mkdir -p "$HOME/.local/bin"
mv ./argus "$HOME/.local/bin/argus"
export PATH="$HOME/.local/bin:$PATH"
```

Installation check:

```bash
argus doctor
```

## 4. Minimal Execution (Under 5 Minutes)

Create a small test config:

```bash
cat > config.yaml <<'EOF'
seed: 42
steps: 120
repeat: 3
warmup_steps: 10
workload:
  nodes: 4
  requests_per_step: 32
EOF
```

Run with one command:

```bash
argus run ./config.yaml
```

Argus automatically creates a run directory and prints its location.

## 5. Expected Artifacts

Verification point is file generation, not numeric values.

Expected files in a run directory:

- `metrics.json`: raw measured metrics
- `report.md`: human-readable report
- `run_meta.json`: execution metadata and environment context
- `resolved_config.yaml` (or equivalent config snapshot): resolved run configuration

## 6. Report Regeneration

You can regenerate a report from an existing run directory without re-measurement.

```bash
argus report <run_dir>
```

This validates record-to-report reproducibility.

## 7. Shareable Sanitized Export

Create a shareable package:

```bash
argus export <run_dir> --sanitize
```

`--sanitize` removes or redacts sensitive context such as:

- usernames
- absolute user-home paths
- hostnames
- environment-variable assignments
- IP addresses

This enables external validation sharing.

## 8. Stability Check

Argus provides variability-aware judgment through recorded dispersion statistics.

Stability-relevant outputs include:

- Mean
- Std Dev
- Min
- Max

If variability is high, Argus marks the result as potentially noise-dominated.

## 9. Interpretation Boundary

What this reproducibility run means:

- the protocol executed successfully in your environment
- observation records were produced
- the report is reproducible from the same run directory

What this reproducibility run does not mean:

- no claim about model-specific outcomes
- no claim about speed improvement
- no claim tied to a specific hardware advantage

## 10. Failure Handling

If `argus doctor` fails:

- confirm binary and architecture match (for `exec format error`, download the correct binary)
- confirm execute permission (`chmod +x argus`)
- confirm PATH contains the installed location (`command -v argus`)

If runtime fails due to permission constraints:

- run in a normal host shell with standard user permissions
- avoid restricted container profiles for this protocol run

If runtime fails due to low memory:

- reduce `steps` and `repeat` in `config.yaml`
- rerun and confirm artifact generation
