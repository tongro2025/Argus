# **Argus v0.2 — Execution Observatory Architecture**

![Argus Full Logo](../argus_logo/argus_full_logo.png)

**(PC / Linux / PID-centric / TUI + Perfetto / CO-RE Single Binary / Loss-aware Ground Truth)**

English version: [Argus v0.2.en.md](Argus%20v0.2.en.md)

---

## **0. 한 줄 정의**

**Argus는 eBPF CO-RE 기반 단일 바이너리로 동작하며, 특정 프로세스의 실행을 “하드웨어 큐에서의 대기 사건(wait events)”과 “관측 무결성(integrity)”으로 마이크로초 단위로 기록하고, 세그먼트 단위 타임라인으로 재구성해 TUI 및 Perfetto로 표현하는 실행 관측기(Execution Observatory)다.**

> 핵심:
> 
> 
> **Zero-loss를 ‘약속’이 아닌 ‘측정 가능한 무결성 계약(Integrity Contract)’으로 제공**
> 

---

## **1. 목표와 비목표**

### **1.1 목표 (Goals)**

1. **Loss-aware Ground Truth (무결성 계약)**
    - 이벤트 유실 여부/규모/구간을 **동일 해상도로 측정·기록·표기**
    - DropCount==0인 구간은 “완전 관측(Complete)”로, DropCount>0 구간은 “불완전 관측(Partial)”로 명시
2. **CO-RE 단일 바이너리(Zero-dependency Runtime)**
    - 타겟 머신에서 **컴파일러/커널 헤더 설치 없이** 실행
    - BTF/권한/락다운 상태를 사전에 점검하고 “가능/불가”를 명시적으로 출력
3. **PID 중심 관측(프로세스 생애 추적)**
    - 특정 PID + 모든 thread(tid) 타임라인 복원
    - 옵션: --follow-children, --follow-exec
    - v0.2 스코프: PID 중심(이름/커맨드라인 기반은 v0.3)
4. **세그먼트 기반 관측**
    - 수동 마킹(키 입력) + 이벤트 규칙 트리거(자동 세그먼트)
    - 세그먼트별 waiting composition(Q/L/I/M) 정량화
5. **Machine Snapshot (프로세스 전 기계 특정)**
    - trace 시작 전에 하드웨어/OS/power 상태를 고정 저장
    - 결과의 재현성과 비교 실험(다른 PC/서버) 기반 제공
6. **Clean + Dynamic TUI + Perfetto Export**
    - TUI는 “관찰” 중심(노이즈 최소, 동적 해상도 줌)
    - 복잡 분석은 argus export --format perfetto로 외부 위임

### **1.2 비목표 (Non-goals)**

- 분산/멀티노드 트레이싱
- GPU trace 통합(향후)
- 자동 최적화/추천/AI 분석
- 무거운 자체 웹 대시보드 구축
- 클라우드 자동 수집 인프라(에이전트/서버)

---

## **2. 핵심 철학(원칙)**

1. Argus는 **메트릭이 아니라 이벤트(event)** 를 원본으로 저장한다.
2. CPU/GPU 사용률이 아니라 **대기(waiting)와 전이(transition)** 를 1차 관측 대상으로 둔다.
3. **“프로세스를 보기 전에 기계를 특정한다.”** (Machine Snapshot → Trace)
4. 관찰자 효과(observer effect)를 최소화하고, **오버헤드/유실/커버리지**를 무결성으로 함께 기록한다.
5. 해석보다 먼저 **표현(representation)과 매핑 규칙**을 고정한다(특히 Perfetto).

---

## **3. 전체 아키텍처**

```
(Target Process)
   ↓ kernel events
[Argus Agent (libbpf CO-RE eBPF)]
   ↓ ring buffer (per-cpu) + drop counters
[Argus Collector]
   - time sync (ktime ↔ monotonic ↔ realtime)
   - state reconstruction (thread timeline)
   - high-speed flush (mmap binary)
   - integrity accounting
   ↓
[Local Store]
   - events.bin (mmap binary)
   - argus_meta.sqlite (session/segments/integrity/time sync)
   - machine.json + fingerprint
   ↓
[Argus TUI Viewer]  or  [argus export → Perfetto]
```

---

## **4. 관측 범위 (v0.2)**

Argus v0.2는 “지연 전파”의 최소 원인을 잡는 5개 축 + 3종 무결성을 관측한다.

### **4.1 대기/실행 이벤트 축 (5)**

1. **스케줄러 대기**: runnable wait / runqueue latency (RUNQLAT)
2. **컨텍스트 스위치**: sched_switch
3. **락/동기화 대기**: futex_wait/wake (가능 범위)
4. **I/O 대기**: block I/O start/done (가능 범위)
5. **메모리 대기**: major page fault (가능 범위)

### **4.2 무결성(Integrity) 축 (3) — “관측 신뢰도”**

A) **Drop Integrity (유실)**

- ringbuf drop count (global + per-cpu)
- drop burst 구간 세그먼트/마커 생성

B) **Overhead Integrity (관찰자 효과)**

- Argus collector/agent의 CPU 시간(프로세스 단위)
- (선택) trace on/off 구간 비교를 위한 간단한 A/B 모드(후속)

C) **Coverage Integrity (커버리지)**

- 해당 커널/권한/환경에서 **수집 불가 이벤트**를 명시
- “0이라서 없는 것” vs “관측 불가능해서 없는 것”을 구분

---

## **5. Machine Snapshot (프로세스 전 ‘기계’ 특정)**

Trace 시작 시 자동으로 머신 구성을 캡처하여 세션에 고정 저장한다.

### **5.1 필수 항목**

- **CPU**
    - 모델, 소켓/코어/스레드
    - governor / 현재 주파수 / min-max 주파수
    - (가능하면) thermal throttling 관련 상태(thermal zone 요약)
- **메모리**
    - 총 RAM, swap 유무/크기 + 현재 사용 현황 요약
- **디스크/파일시스템**
    - 장치 타입(NVMe/SSD/HDD), 모델/큐 설정(가능하면)
    - 마운트 목록, FS 타입
    - I/O scheduler(가능하면)
- **OS/커널/부팅 파라미터**
    - distro, kernel version, cgroup v1/v2
    - /proc/cmdline (isolcpus, mitigations 등 확인 목적)
- **환경 제약**
    - BTF 존재 여부
    - 권한/캡빌리티 상태
    - 커널 락다운/secure boot 영향(가능하면 식별)

### **5.2 선택 항목**

- NUMA 노드 구성, hugepage
- 캐시 크기(L1/L2/L3)
- GPU 존재/드라이버(있으면 기록만)
- NIC 기본 정보(있으면 기록만)

### **5.3 산출물**

- machine.json
- machine_fingerprint = sha256(machine.json canonical form)

---

## **6. 시간 동기화(Time Synchronization) — 강화 버전**

Argus는 이벤트의 기준 시간을 “단일 offset 값”으로 단순화하지 않고, **동기화 신뢰도를 함께 보존**한다.

### **6.1 시간 원천 3종**

- ktime_ns : eBPF bpf_ktime_get_ns() (단조, 커널 기준)
- monotonic_ns : user-space CLOCK_MONOTONIC
- realtime_ns : user-space CLOCK_REALTIME (외부 로그 결합용)

### **6.2 동기화 방법(권장)**

- 세션 시작 시 N=32 샘플을 짧은 간격으로 수집
- offset = realtime_ns - ktime_ns를 **trimmed median**으로 추정
- 세션 종료 시 동일하게 측정해 drift를 기록
- 스키마는 offset_segments[](piecewise) 확장 가능하도록 열어둔다

### **6.3 저장**

- SQLite meta에 time_sync 테이블로 저장
    - start_offset_estimate, end_offset_estimate, drift_estimate, method, N

---

## **7. 데이터 모델(실제 구현 중심)**

## **7.1 세션(Session)**

- session_id, start_ts, end_ts
- target: pid, follow 옵션, thread 포함 정책
- machine_fingerprint
- integrity_summary (drop/overhead/coverage 요약)
- time_sync (위 6장)

---

## **7.2 이벤트 파일 포맷(events.bin) — “읽을 수 있는 바이너리”로 고정**

### **7.2.1 File Header (고정 크기 권장: 4KB)**

- magic = “ARGUS”
- format_version = 0.2
- endianness
- record_struct_size
- timebase: ktime 기준 정보 + time_sync 참조 키
- build_id / git_hash
- capabilities bitmap (수집 가능한 이벤트 종류)
- reserved

### **7.2.2 Event Record (C struct)**

공통 필드:

- ts_ktime_ns
- cpu, pid, tid
- record_type (event 종류)
- payload_len
- duration_ns (해당 시)
- meta (고정/가변 중 택1, v0.2는 고정 최소가 안전)

권장 event types:

- SCHED_SWITCH
- SCHED_WAKEUP
- RUNQLAT (collector가 계산한 “derived record”로도 가능)
- FUTEX_WAIT, FUTEX_WAKE
- IO_START, IO_DONE
- PAGEFAULT_MAJOR
- EVENT_DROPPED (drop burst marker)
- CAPABILITY_NOTICE (커버리지 무결성 표시용 선택)

---

## **7.3 RUNQLAT 정의(관측 가능한 공식)**

RUNQLAT은 Argus의 중심 개념이므로 “정의”를 설계도에 고정한다.

### **기본 정의**

- wakeup_ts[tid] = ts at sched_wakeup(tid)
- run_ts = ts at sched_switch(next=tid)
- runqlat = run_ts - wakeup_ts[tid] (if exists and valid)

### **필수 Edge Case 처리(명시)**

- 중복 wakeup: 가장 최신 wakeup_ts 사용(또는 최소값 정책을 명시)
- CPU migration: wakeup_ts는 per-tid로 유지(퍼CPU 맵이 아닌 글로벌 맵)
- tid 재사용/종료: sched_process_exit/exec 계열 이벤트로 정리(가능 범위)
- wakeup 없이 schedule-in 되는 케이스는 runqlat=unknown으로 처리하고 coverage에 기록

---

## **7.4 Thread State Reconstruction (Collector 산출)**

TUI/Perfetto 표현용 상태:

- RUNNING
- RUNNABLE_WAIT_CPU (RUNQLAT)
- BLOCKED_LOCK (futex 기반)
- BLOCKED_IO
- BLOCKED_MEM (major fault)
- (선택) SLEEP/TIMER (확실히 식별 가능한 경우만)

---

## **7.5 Lock/Futex Attribution Level (한계 선언)**

v0.2는 “정확한 관측 범위”를 아래로 선언한다.

- **L0**: LOCK_WAIT 존재 여부만
- **L1 (v0.2 목표)**: futex key(주소) 기반 그룹핑
- **L2+ (v0.3)**: user stack hash / symbolization 기반 원인 추적

이 레벨은 세션 메타(capabilities)에 기록한다.

---

## **8. 세그먼트 시스템**

### **8.1 Segment 스키마**

- segment_id
- t_start, t_end (기본: ktime, export 시 realtime으로 변환)
- label
- trigger = manual | event:<rule>
- pad_before, pad_after
- integrity_status (Complete/Partial)

### **8.2 세그먼트 생성**

1. **Manual Marking (TUI)**
    - m 시작 / M 종료 / l 라벨
2. **Event Triggered**
    - 기본 권장: runqlat spike + pad=2s
    - 예: majorfault:any:pad=3s, io_wait>threshold:window=1s

> v0.2의 목적은 “자동 해석”이 아니라
> 
> 
> **관측 구간을 잘라주는 기능**
> 

---

## **9. TUI (Terminal Viewer) — 구현 기준 포함**

### **9.1 3패널 레이아웃**

A) **Dynamic Timeline**

- zoom level별 bin size를 **명시적으로 제한된 집합**으로 사용
    - 예: {1ms, 5ms, 10ms, 50ms, 100ms, 500ms}
- 표시 정책(클린 우선):
    - bin 내 **dominant state 1개**를 기본 표시
    - (선택) secondary indicator를 작은 표식으로만 추가
- 상태 문자:
    - R running, Q runnable wait, L lock, I IO, M mem
- **Drop 구간 표기**
    - D 마커 + 세그먼트 integrity=Partial

B) **Segment Breakdown**

- 세그먼트별 Q/L/I/M 누적 시간 + 비율
- “Off-CPU 대기가 가장 긴 tid” 표시
- (가능하면) top futex key / top block device 등 요약

C) **Event Burst List**

- runqlat spike / major fault / io burst / drop burst 발생 시각 목록

### **9.2 키 바인딩**

- ←/→: 시간 이동
- +/−: 줌 인/아웃
- ↑/↓: 스레드 선택
- m/M: 세그먼트 시작/종료
- l: 라벨
- g: 시간 점프
- s: 저장
- q: 종료

---

## **10. Perfetto Export — “표준 매핑 규칙” 고정**

argus export --format perfetto는 Argus의 외부 GUI 표준이다.

### **10.1 매핑 규칙**

- Track: 각 tid에 대해 thread track 생성
- Slice:
    - 상태 slice: RUNNING/Q/L/I/M
    - 이벤트 slice: majorfault/io/drop (annotation 포함)
- Metadata:
    - machine_fingerprint
    - integrity_summary
    - capabilities(coverage)
- Drop 표기:
    - drop burst 구간에 slice + "dropped N events" annotation

---

## **11. CLI 명세**

```
# 1) Trace
argus trace --pid <pid> [--duration 30s] \
            [--follow-children] [--follow-exec] \
            [--segment-on "<rule>"]... \
            [--out <dir>]

# 2) View (TUI)
argus view --run <session_id|latest>

# 3) Export (Perfetto)
argus export --run <session_id|latest> --format perfetto --out trace.json
```

Trace 실행 순서(고정):

1. Machine Snapshot
2. Capability/권한/BTF 체크 + coverage 기록
3. Attach + (옵션) warmup
4. Event capture + integrity accounting
5. Flush + store

---

## **12. 저장 구조**

```
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

### **SQLite(meta) 권장 테이블**

- sessions
- segments
- time_sync
- integrity (drop/overhead/coverage detail)
- capabilities

---

## **13. 구현 스택 (실제 구현 목적)**

**Linux 전용 / CO-RE 단일 바이너리 목표**

권장 옵션 A (Rust)

- eBPF: libbpf-rs 또는 Aya(단, CO-RE/배포 성숙도 확인)
- TUI: ratatui
- store: rusqlite + mmap

권장 옵션 B (Go)

- eBPF: cilium/ebpf (CO-RE 지원 경로 정리 필요)
- TUI: bubbletea
- store: mattn/go-sqlite3 + mmap

> v0.2는 “성능/무결성”이 중요하니,
> 
> 
> **agent/collector는 한 언어로 통일**
> 
> **mmap 기반 바이너리**
> 

---

## **14. 실행 권한/호환성 명시 (배포 현실)**

Argus는 실행 전 다음을 검사하고 세션 메타에 기록한다.

- 필요한 권한: root 또는 적절한 CAP_BPF/CAP_SYS_ADMIN 등
- BTF 존재 여부
- 커널 락다운 모드(가능하면)
- 지원 커널 버전 범위(최소 범위를 명시)

불가 시:

- “왜 불가인지(coverage)”를 명확히 출력하고 종료(침묵 금지)

---

## **15. 성공 기준 (v0.2)**

1. 타겟 머신에 **단일 바이너리**로 실행 가능(추가 설치 최소)
2. 특정 PID 세션 기록이 남고, **Drop/Overhead/Coverage 무결성**이 함께 저장된다
3. TUI에서 스레드 타임라인이 **줌 인/아웃**으로 안정적으로 보인다
4. 수동/자동 세그먼트 저장이 가능하고, 세그먼트별 **Q/L/I/M**이 정량 출력된다
5. Perfetto export가 표준 매핑 규칙대로 동작하며, drop 구간이 GUI에서도 명확히 표시된다
6. events.bin이 header/versioning을 갖추고, v0.3 확장에도 읽기 호환이 깨지지 않는다

---

## **16. v0.2 → v0.3 확장 포인트(열어둠)**

- follow-by-name/cmdline
- futex attribution L2(스택 해시) + symbolization
- phase change hint(대기 구성 변화점 탐지)

---

## **17. Observer Effect & Measurement Integrity (보강 설계)**

Argus는 관찰자 효과를 “없다”고 가정하지 않고, 세션 결과와 함께 정량화한다.

- Drop Integrity: drop count / drop burst / 세그먼트 partial 표기
- Overhead Integrity: observer cpu/sched/context switch/io bytes 기록
- Coverage Integrity: 관측 불가 항목 명시
- Isolation Mode: 가능한 범위에서 observer 우선순위 하향 및 적용 여부 기록
- Segment Annotation: drop 영향, observer cpu ratio, isolation 적용 여부를 세그먼트에 포함

상세 설계 문서는 아래를 참조:

- `Argus_observer_integrity.md`
- GPU 통합(Nsight/driver trace), cgroup/클라우드
