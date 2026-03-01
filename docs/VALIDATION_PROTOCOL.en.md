# Argus v1 Validation Protocol

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

Argus v1 is a reproducible observation protocol.
Its output is a **Reproducible Observation Record**, not a direct performance claim.

## CLI

```bash
argus doctor
argus run <config.yaml>
argus report <run_dir>
argus export <run_dir>
argus export <run_dir> --sanitize
```

## Submission Protocol

Third-party validators run the experiment `repeat` times and submit:

- `metrics.json`
- `report.md`
- `run_meta.json`

Recommended package:

- `argus_result.zip` (or `argus_result.sanitized.zip`)

## Sanitization

`argus export <run_dir> --sanitize` redacts:

- username
- hostname
- absolute user-home paths
- environment-variable assignments
- IPv4 addresses

## Privacy statement

Argus does not collect local file paths, user account names, or environment-variable values as primary experiment metrics.
