# Argus — Observer Effect & Measurement Integrity

![Argus Icon Logo](../argus_logo/argus_icon_logo.png)

English version: [Argus_observer_integrity.en.md](Argus_observer_integrity.en.md)

## 1. 문제 정의 (왜 필요한가)

Argus는 시스템 실행을 관찰하는 도구이지만, 관찰 자체가 실행에 영향을 줄 수 있다.
즉 Argus가 실행되는 순간 Argus는 측정 대상 시스템의 일부가 된다.

이로 인해 다음 위험이 생긴다.

- 스케줄링 순서 변화
- runqueue 대기 시간 변형
- 캐시 압박 증가
- 디스크 I/O 증가
- 실제 병목과 다른 병목이 관측될 가능성

따라서 Argus는 단순 모니터링 툴이 아니라 실험 장비(instrumentation apparatus)로 취급해야 한다.

## 2. 관찰자 효과의 구체적 원인

### 2.1 CPU 간섭

- eBPF 프로그램 실행
- 유저 공간 collector 처리
- TUI 렌더링

### 2.2 메모리 간섭

- ring buffer
- event queue
- mmap flush

### 2.3 스케줄링 간섭 (핵심)

Argus thread가 runqueue에 들어오면 타겟 프로세스 실행 순서가 바뀌고,
runqlat 측정값 자체가 변형될 수 있다.

### 2.4 I/O 간섭

- events.bin 기록으로 디스크 write 발생
- 그 자체로 I/O wait를 늘릴 수 있음

## 3. Argus 설계 원칙

Argus의 목표는 "영향 0"이 아니다. 현실적으로 완전한 투명성은 불가능하다.

Argus는 다음을 함께 제공한다.

- 관측 데이터
- 관측이 시스템에 미친 영향량(Integrity Data)

즉 Argus는 불투명함을 숨기지 않고 정량화한다.
이를 Measurement Integrity로 정의한다.

## 4. Measurement Integrity 구성 요소

### 4.1 Drop Integrity

- ring buffer drop count
- per-CPU drop
- drop burst 구간 표시

목적: 관측 완전성 판단

### 4.2 Overhead Integrity

관찰자 효과를 자원량으로 측정한다.

예시 항목:

- observer_cpu_time
- observer_sched_ratio
- observer_context_switch
- observer_io_bytes

### 4.3 Coverage Integrity

환경/권한/커널 기능 제약으로 인한 관측 불가능 범위를 명시한다.

- 권한 부족
- 커널 기능 미지원
- BTF 미존재

핵심: "0이라서 없음" 과 "관측 불가능"을 구분

## 5. 관찰자 영향 최소화 설계

### 5.1 Self-Profiling

Argus 자신도 관측 대상에 포함한다.

### 5.2 Isolation Mode

가능하면 타겟과 관측자의 경쟁을 줄인다.

- priority 낮춤(nice)
- 가능 시 CPU affinity 분리
- 가능 시 batch/idle 스케줄링 정책

### 5.3 Write Decoupling

관측 I/O가 타겟 병목을 만들지 않게 한다.

- 메모리 버퍼링
- 비동기 flush
- batched write

### 5.4 Integrity Annotation

세그먼트마다 다음을 같이 저장한다.

- drop 발생 여부
- 관측 오버헤드
- isolation 성공 여부

## 6. 결과적 의미

이 원칙이 포함되면 Argus는 단순 프로파일러가 아니라 실행 관측 실험 장비가 된다.

가능해지는 것:

- 반복 측정
- 오차 범위 판단
- 실제 시스템 검증
- 이론의 실험적 확인

## 핵심 요약

Argus는 완전 투명 관찰을 약속하지 않는다.
대신 관찰이 시스템에 준 영향을 함께 측정해 결과의 신뢰도를 제공한다.

## 구현 매핑 (현재 코드 기준)

- Drop Integrity: `drop_count`, `drop_bursts`, segment `drop_affected`
- Overhead Integrity: `collector_cpu_ns`, `observer_sched_ratio`,
  `observer_context_switch(_voluntary/_involuntary)`, `observer_io_bytes`
- Coverage Integrity: `unsupported_capabilities`, `unknown_runqlat_count`
- Isolation Mode: `--isolation` 옵션과 `summary.isolation.*` 기록
- Integrity Annotation: `segment.integrity_status`,
  `segment.observer_cpu_ratio`, `segment.isolation_applied`
