# **Argus v1.0 — 실행 관측기 아키텍처**

![Argus Full Logo](../argus_logo/argus_full_logo.png)

**(재현 가능한 관측 프로토콜 / 설정 기반 실행 / 보고서 재생성 / Sanitized 공유 / 무결성 기반 해석 경계)**

English version: [Argus v1.0.en.md](Argus%20v1.0.en.md)

---

## **0. 한 줄 정의**

**Argus v1.0은 동일 조건에서 재현 가능한 관측 산출물을 기록하고, 무결성 경계를 포함한 보고서를 재생성 및 공유할 수 있도록 설계된 실행 관측 프로토콜이다.**

v1.0은 성능 최적화 도구를 표방하지 않는다.
재현 가능한 측정/보고 프로토콜을 제공한다.

---

## **1. 왜 v1.0으로 바뀌었는가**

Argus v0.2는 대기 중심 관측 아키텍처를 확립했다.
Argus v1.0은 그 철학을 유지하면서 제품 계약을 다음처럼 바꿨다:

- PID 중심 운영 추적에서 **프로토콜 중심 재현 실행**으로 전환
- 단발성 결과 확인에서 **산출물 계약 + 보고 재생성**으로 전환
- 로컬 결과 보관에서 **sanitize 가능한 외부 공유 패키지**로 확장

---

## **2. v0.2 → v1.0 아키텍처 변경점**

| 영역 | v0.2 | v1.0 |
|---|---|---|
| 기본 관점 | PID 중심 캡처 아키텍처 | 재현 가능한 검증 프로토콜 |
| 진입 명령 | PID + duration 기반 `trace/view/export/report/run` | `config.yaml` + run directory 기반 `doctor/run/report/export` |
| 핵심 산출물 계약 | 이벤트 타임라인 + 무결성 맥락 | 재현 기록 세트 (`metrics.json`, `report.md`, `run_meta.json`, config snapshot) |
| 결과 재사용 | 세션 분석 중심 | 재측정 없이 보고 재생성 명시 |
| 공유 모델 | 로컬 파일 중심 | `export --sanitize` 기반 외부 검증 공유 |
| 신뢰도 언어 | 관찰자 효과/무결성 | 무결성 + 안정성 판정 + 해석 경계 |
| 주장 경계 | 실행 대기 가시화 | 동일 조건에서 관측/보고 재현성 |

---

## **3. v1.0 시스템 아키텍처**

```text
[argus doctor]
  -> 실행 환경 적합성 검사
     (플랫폼/권한/런타임 준비 상태)

[argus run <config.yaml>]
  -> config 해석
  -> warmup 구간 실행 (최종 통계에서 제외)
  -> 반복 관측 실행
  -> 지표 집계 + 무결성 계산
  -> run directory 기록

[Run Directory]
  - metrics.json
  - report.md
  - run_meta.json
  - resolved_config.yaml (또는 동등 스냅샷)

[argus report <run_dir>]
  -> 기존 기록으로 보고서 재생성
     (새 측정 없음)

[argus export <run_dir> --sanitize]
  -> 공유용 패키지 생성
  -> 개인정보/환경 식별 문맥 제거
```

---

## **4. v1.0 CLI 계약**

```bash
argus doctor
argus run <config.yaml>
argus report <run_dir>
argus export <run_dir>
argus export <run_dir> --sanitize
```

명령 의미:

- `doctor`: 재현 프로토콜 실행 전 환경 적합성 점검
- `run`: 프로토콜 정의에 따라 관측 실행 후 산출물 생성
- `report`: 기존 산출물에서 사람이 읽는 보고서 재생성
- `export`: 산출물 패키징
- `export --sanitize`: 민감 문맥 제거 후 패키징

---

## **5. 프로토콜 설정 모델**

대표 필드:

- `seed`
- `steps`
- `repeat`
- `warmup_steps`
- `workload.*`

설계 의도:

- warmup은 실행 연속성을 위해 수행하되 최종 비교 통계에서 제외
- 반복 실행(repeat)을 1급 개념으로 두어 변동성 관측 가능

---

## **6. 산출물 계약 (v1.0)**

필수 기록 파일:

- `metrics.json`: 원시 지표 및 파생 요약
- `report.md`: 안정성 판정 문구를 포함한 사람이 읽는 보고서
- `run_meta.json`: 환경/런타임 메타데이터와 프로토콜 문맥
- `resolved_config.yaml` (또는 동등 파일): 해석 완료된 실행 설정

검증 포인트:

- 성공 기준은 재현 가능한 산출물 생성과 보고서 재생성
- 숫자 우열은 검증 목표가 아님

---

## **7. 무결성과 안정성 모델**

v1.0은 무결성 기반 해석을 유지하면서 확장한다:

- **Drop/Coverage/Overhead integrity** 유지
- **Stability check**를 분산 정보(Mean/Std Dev/Min/Max)로 명시
- 변동성이 높으면 노이즈 우세 가능성을 표시해 과해석을 방지

---

## **8. Sanitized Export의 1급 기능화**

`argus export <run_dir> --sanitize`는 다음 정보를 제거/치환해 검증 공유를 쉽게 만든다:

- 사용자 계정명
- 절대 사용자 홈 경로
- hostname
- 환경변수 할당 문자열
- IP 주소

이는 제3자 재현 검증 시 공유 장벽을 낮춘다.

---

## **9. 해석 경계 (v1.0)**

v1.0 프로토콜 실행이 의미하는 것:

- 선언된 실행 프로토콜 하에서 관측 기록이 생성됨
- 동일 기록에서 보고서를 재생성할 수 있음
- 변동성과 무결성 문맥이 보존됨

의미하지 않는 것:

- 성능 향상 주장 아님
- 모델 수준 우월성 주장 아님
- 특정 하드웨어 우위 주장 아님

---

## **10. v0.2 사용자 전환 메모**

v0.2 아키텍처 문서에서 넘어오는 경우:

- 대기/무결성 중심 사고는 그대로 유지
- 운영 관점은 PID 세션 분석에서 실행 프로토콜 재현으로 이동
- 기본 경로는 `doctor -> run -> report -> export --sanitize`

v1.0은 철학 교체가 아니라 계약 고도화다.
