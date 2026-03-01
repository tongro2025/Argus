# Argus v1 Claims and Limits

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

Argus records comparative execution behavior under controlled protocol settings.

## Observation overhead

Argus introduces runtime observation overhead.

The tool measures comparative behavior under identical observation conditions, not absolute hardware performance.

## Failure handling

Failures are recorded as observations (`partial_failure`) and are not hidden.

Argus does not interpret failure as improvement or degradation.

It records operational behavior.
