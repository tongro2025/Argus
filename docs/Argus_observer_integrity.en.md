# Argus — Observer Effect & Measurement Integrity

![Argus Icon Logo](../argus_logo/argus_icon_logo.png)

Korean version: [Argus_observer_integrity.md](Argus_observer_integrity.md)

## 1. Problem Statement (Why This Is Needed)

Argus observes system execution, but observation itself can alter execution.
As soon as Argus runs, it becomes part of the measured system.

This introduces risks such as:

- changed scheduling order
- distorted runqueue waiting time
- increased cache pressure
- additional disk I/O
- observing a bottleneck created by instrumentation rather than by workload

So Argus should be treated as instrumentation apparatus, not just a monitoring tool.

## 2. Concrete Sources of Observer Effect

### 2.1 CPU Interference

- eBPF program execution
- user-space collector processing
- TUI rendering

### 2.2 Memory Interference

- ring buffer
- event queue
- mmap flush

### 2.3 Scheduling Interference (Most Important)

When Argus threads enter runqueue:

- target process execution order may shift
- measured runqlat itself can be perturbed

### 2.4 I/O Interference

- writing `events.bin`
- added disk writes can increase I/O wait

## 3. Argus Design Principle

Argus does not claim zero impact.
Perfect transparency is unrealistic in production systems.

Argus therefore records both:

- observed data
- impact introduced by observation itself (Integrity Data)

In other words, Argus does not hide opacity; it quantifies it.
This is defined as Measurement Integrity.

## 4. Measurement Integrity Components

### 4.1 Drop Integrity

- ring buffer drop count
- per-CPU drop
- drop burst windows

Purpose: determine whether observation is complete.

### 4.2 Overhead Integrity

Measure observer impact as resource usage.

Example metrics:

- `observer_cpu_time`
- `observer_sched_ratio`
- `observer_context_switch`
- `observer_io_bytes`

### 4.3 Coverage Integrity

Explicitly record unobservable scope caused by environment constraints:

- insufficient privileges
- unsupported kernel features
- missing BTF

Key point: distinguish “zero because none occurred” from “zero because unobservable.”

## 5. Design to Minimize Observer Impact

### 5.1 Self-Profiling

Argus includes itself as a measurement target.

### 5.2 Isolation Mode

Reduce target-observer competition where possible:

- lower priority (`nice` / process priority class)
- optional CPU affinity separation
- optional batch/idle scheduling policy

### 5.3 Write Decoupling

Avoid instrumentation I/O becoming the bottleneck:

- memory buffering
- async flush
- batched writes

### 5.4 Integrity Annotation

Store integrity metadata per segment:

- whether drop occurred
- observer overhead
- whether isolation was successfully applied

## 6. Practical Meaning

With these principles, Argus is not merely a profiler.
It is an execution observatory with explicit measurement trust metadata.

This enables:

- repeatable measurement
- error-range reasoning
- realistic system validation
- experimental verification of performance hypotheses

## One-Line Summary

Argus does not promise perfectly transparent observation.
Instead, it measures observer impact together with observations and reports confidence context.

## Implementation Mapping (Current Code)

- Drop Integrity: `drop_count`, `drop_bursts`, segment `drop_affected`
- Overhead Integrity: `collector_cpu_ns`, `observer_sched_ratio`, `observer_context_switch(_voluntary/_involuntary)`, `observer_io_bytes`
- Coverage Integrity: `unsupported_capabilities`, `unknown_runqlat_count`
- Isolation Mode: `--isolation` and `summary.isolation.*`
- Integrity Annotation: `segment.integrity_status`, `segment.observer_cpu_ratio`, `segment.isolation_applied`
