# **Argus v0.2 — Execution Observatory Architecture**

![Argus Full Logo](../argus_logo/argus_full_logo.png)

**(PC / Linux / PID-centric / TUI + Perfetto / CO-RE Single Binary / Loss-aware Ground Truth)**

Korean version: [Argus v0.2.md](Argus%20v0.2.md)

---

## **0. One-line Definition**

**Argus is an execution observatory that runs as a single eBPF CO-RE binary, captures a target process with microsecond-level wait-event and integrity metadata, reconstructs segment-based timelines, and presents results in TUI and Perfetto.**

Core point:

**Zero-loss is not a promise. It is an explicit, measurable Integrity Contract.**

---

## **1. Goals and Non-goals**

### **1.1 Goals**

1. **Loss-aware Ground Truth (Integrity Contract)**
   - record event loss with amount and time windows
   - mark `Complete` when `DropCount == 0`, `Partial` when `DropCount > 0`
2. **CO-RE single binary (zero-dependency runtime)**
   - run without compiler/kernel-header installation on target hosts
   - check BTF/privileges/lockdown capability before capture
3. **PID-centric lifecycle tracing**
   - reconstruct timeline for target PID and all threads (TIDs)
   - options: `--follow-children`, `--follow-exec`
   - v0.2 scope: PID-centric (name/cmdline targeting deferred)
4. **Segment-based observation**
   - manual markers + event-rule auto segments
   - per-segment waiting composition quantification (`Q/L/I/M`)
5. **Machine Snapshot before capture**
   - persist machine/environment state before trace start
   - support reproducibility and cross-host comparison
6. **Clean dynamic TUI + Perfetto export**
   - keep TUI focused and low-noise
   - delegate deep GUI analysis to Perfetto export

### **1.2 Non-goals**

- distributed multi-node tracing
- integrated GPU tracing in v0.2
- auto optimization recommendation/AI diagnosis
- heavy custom web dashboard in v0.2
- cloud collector infrastructure in v0.2

---

## **2. Core Principles**

1. Argus stores **events** as primary source, not only aggregated metrics.
2. First-class targets are **waiting and transitions**, not top-level utilization.
3. **Identify the machine before identifying the process.**
4. Minimize observer effect and record **overhead/loss/coverage** together.
5. Fix representation and mapping rules before interpretation.

---

## **3. Architecture Overview**

```text
(Target Process)
   ↓ kernel events
[Argus Agent (CO-RE eBPF)]
   ↓ ring buffer (per-CPU) + drop counters
[Argus Collector]
   - time sync (ktime ↔ monotonic ↔ realtime)
   - state reconstruction (thread timeline)
   - high-speed flush (binary)
   - integrity accounting
   ↓
[Local Store]
   - events.bin
   - argus_meta.sqlite
   - machine.json + fingerprint
   ↓
[TUI Viewer] or [argus export --format perfetto]
```

---

## **4. Observation Scope (v0.2)**

### **4.1 Wait/Execution Event Axes (5)**

1. scheduler wait: runnable wait / runqueue latency (`RUNQLAT`)
2. context switches: `sched_switch`
3. lock/sync wait: `futex_wait/wake` (within available range)
4. I/O wait: block I/O start/done (within available range)
5. memory wait: major page fault (within available range)

### **4.2 Integrity Axes (3)**

A) **Drop Integrity**
- global/per-CPU ringbuffer drop count
- drop burst marker segments

B) **Overhead Integrity**
- collector/agent CPU usage and scheduler pressure
- optional A/B trace-on/off overhead comparison (future)

C) **Coverage Integrity**
- explicitly declare unsupported/unobservable event classes
- distinguish “zero observed” vs “not observable”

---

## **5. Machine Snapshot (Machine First)**

Capture and persist machine profile at trace start.

### **5.1 Required Fields**

- CPU model/topology/governor/frequency/thermal hints
- memory+swap totals and status summary
- storage device/mount/fs scheduler info (when available)
- OS/kernel/cgroup/boot cmdline info
- environment constraints (BTF, privilege state, lockdown hints)

### **5.2 Optional Fields**

- NUMA/hugepage
- cache sizes (L1/L2/L3)
- GPU presence/driver (record-only)
- NIC baseline info (record-only)

### **5.3 Artifacts**

- `machine.json`
- `machine_fingerprint = sha256(canonical(machine.json))`

---

## **6. Time Synchronization (Enhanced)**

Argus preserves synchronization quality instead of storing a single static offset.

### **6.1 Time Sources**

- `ktime_ns` (kernel monotonic from eBPF)
- `monotonic_ns` (user-space monotonic)
- `realtime_ns` (wall-clock for external correlation)

### **6.2 Recommended Method**

- collect N=32 samples at session start
- estimate `offset = realtime - ktime` via trimmed median
- repeat at end and persist drift
- schema is extensible for piecewise offset segments

### **6.3 Persistence**

`time_sync` table stores:
- `start_offset_estimate`, `end_offset_estimate`
- `drift_estimate`, `method`, `sample_n`

---

## **7. Data Model (Implementation-centered)**

## **7.1 Session**

- `session_id`, start/end timestamps
- target PID and follow options
- `machine_fingerprint`
- integrity summary (drop/overhead/coverage)
- time-sync metadata

## **7.2 Event File Format (`events.bin`)**

### **7.2.1 File Header (fixed, recommended 4KB)**

- magic (`ARGUS`)
- format version (`0.2`)
- endianness
- record struct size
- timebase and sync linkage
- build id / git hash
- capability bitmap
- reserved fields

### **7.2.2 Event Record (fixed struct in v0.2)**

Common fields:
- `ts_ktime_ns`, `cpu`, `pid`, `tid`
- `record_type`, `payload_len`
- `duration_ns` (if applicable)
- metadata fields

Recommended types:
- `SCHED_SWITCH`
- `SCHED_WAKEUP`
- `RUNQLAT` (derived)
- `FUTEX_WAIT`, `FUTEX_WAKE`
- `IO_START`, `IO_DONE`
- `PAGEFAULT_MAJOR`
- `EVENT_DROPPED`
- `CAPABILITY_NOTICE` (optional)

## **7.3 RUNQLAT Definition**

Base formula:
- `wakeup_ts[tid] = ts(sched_wakeup(tid))`
- `run_ts = ts(sched_switch(next=tid))`
- `runqlat = run_ts - wakeup_ts[tid]`

Required edge-case rules:
- duplicate wakeups: use policy-defined most recent (or min) value
- CPU migration: track by TID, not by CPU-local key
- TID reuse/exit: clean up with process lifecycle events
- no wakeup edge: mark as unknown and count in coverage integrity

## **7.4 Thread State Reconstruction**

Viewer/export states:
- `RUNNING`
- `RUNNABLE_WAIT_CPU`
- `BLOCKED_LOCK`
- `BLOCKED_IO`
- `BLOCKED_MEM`
- optional sleep/timer state (only when confidence is clear)

## **7.5 Lock/Futex Attribution Levels**

- `L0`: lock wait presence only
- `L1` (v0.2 target): grouping by futex key/address
- `L2+` (v0.3): stack hash + symbolized root cause

Attribution level is persisted into session capabilities.

---

## **8. Segment System**

### **8.1 Segment Schema**

- `segment_id`
- `t_start`, `t_end`
- `label`
- `trigger` = `manual` | `event:<rule>`
- `pad_before`, `pad_after`
- `integrity_status` (`Complete`/`Partial`)

### **8.2 Segment Creation**

1. **Manual marking (TUI)**
   - `m` start / `M` end / `l` label
2. **Event-triggered auto segments**
   - recommended baseline: runqlat spike + `pad=2s`
   - examples: `majorfault:any:pad=3s`, `io_wait>threshold:window=1s`

v0.2 focus is robust observation slicing, not automated interpretation.

---

## **9. TUI (Terminal Viewer) Implementation Rules**

### **9.1 Three-panel layout**

A) **Dynamic Timeline**
- fixed bin set by zoom level: `{1ms, 5ms, 10ms, 50ms, 100ms, 500ms}`
- dominant-state rendering per bin
- optional light secondary indicator
- symbols: `R`, `Q`, `L`, `I`, `M`
- drop windows marked with `D` and segment integrity `Partial`

B) **Segment Breakdown**
- per-segment totals/ratios for `Q/L/I/M`
- show top off-CPU TID
- optionally show top futex key/block device summary

C) **Event Burst List**
- timestamps for runqlat/majorfault/io/drop bursts

### **9.2 Key bindings**

- `←/→`: move time
- `+/-`: zoom in/out
- `↑/↓`: select thread
- `m/M`: segment start/end
- `l`: label
- `g`: jump to time
- `s`: save
- `q`: quit

---

## **10. Perfetto Export (Fixed Mapping Rules)**

`argus export --format perfetto` is the standard external GUI path.

### **10.1 Mapping**

- Track: one thread track per TID
- Slice:
  - state slices: `RUNNING/Q/L/I/M`
  - event slices: majorfault/io/drop with annotations
- Metadata:
  - `machine_fingerprint`
  - `integrity_summary`
  - capabilities/coverage
- Drop:
  - drop burst slices with `dropped N events` annotation

---

## **11. CLI Specification**

```bash
# 1) Trace
argus trace --pid <pid> [--duration 30s] \
            [--follow-children] [--follow-exec] \
            [--segment-on "<rule>"]... \
            [--out <dir>]

# 2) View
argus view --run <session_id|latest>

# 3) Export
argus export --run <session_id|latest> --format perfetto --out trace.json
```

Fixed trace sequence:

1. machine snapshot
2. capability/permission/BTF checks + coverage metadata
3. attach + optional warmup
4. event capture + integrity accounting
5. flush + persistence

---

## **12. Storage Layout**

```text
argus_runs/
  <session_id>/
    machine.json
    machine.sha256
    argus_meta.sqlite
    events.bin
    summary.json
    export/
      perfetto_trace.json   (optional)
```

Recommended SQLite(meta) tables:
- `sessions`
- `segments`
- `time_sync`
- `integrity` (drop/overhead/coverage details)
- `capabilities`

---

## **13. Implementation Stack (Practical Targets)**

**Linux-first / CO-RE single-binary goal**

Option A (Rust)
- eBPF: `libbpf-rs` or `Aya` (validate CO-RE/runtime maturity)
- TUI: `ratatui`
- store: `rusqlite` + mmap

Option B (Go)
- eBPF: `cilium/ebpf` (clear CO-RE path and packaging)
- TUI: `bubbletea`
- store: SQLite + mmap

v0.2 priority is integrity and stability:
- keep agent/collector unified in one language
- keep binary format mmap-friendly

---

## **14. Runtime Privilege/Compatibility Declaration**

Before execution, Argus records:

- required privilege level (`root` or capabilities)
- BTF existence
- lockdown mode hint (when observable)
- supported kernel range

If not runnable:
- fail with explicit reason in coverage metadata (no silent fallback)

---

## **15. Success Criteria (v0.2)**

1. single-binary execution on target machine
2. PID session capture with Drop/Overhead/Coverage persisted together
3. stable zoomable TUI timeline
4. manual/auto segment persistence with quantified `Q/L/I/M`
5. Perfetto export with clear drop-window representation
6. `events.bin` header/versioning compatible with future extension

---

## **16. v0.2 → v0.3 Extension Points**

- follow by process name/cmdline
- futex attribution L2 (stack hash + symbolization)
- phase-change hints (waiting composition change detection)

---

## **17. Observer Effect & Measurement Integrity (Reinforcement)**

Argus does not assume observer effect is zero; it quantifies it per session.

- Drop Integrity: drop count / drop bursts / segment partial marks
- Overhead Integrity: observer CPU/scheduler/context-switch/I/O bytes
- Coverage Integrity: explicit unsupported/unobservable items
- Isolation Mode: lower observer priority where possible and persist outcome
- Segment Annotation: include drop impact, observer CPU ratio, isolation status

See detailed document:

- [Argus_observer_integrity.md](Argus_observer_integrity.md)
- [Argus_observer_integrity.en.md](Argus_observer_integrity.en.md)
