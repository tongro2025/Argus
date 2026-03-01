# Argus v1 검증 프로토콜

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

Argus v1은 재현 가능한 관측 프로토콜입니다.
산출물은 성능 주장 자체가 아니라 **재현 가능한 관측 기록(Reproducible Observation Record)** 입니다.

## CLI

```bash
argus doctor
argus run <config.yaml>
argus report <run_dir>
argus export <run_dir>
argus export <run_dir> --sanitize
```

## 제출 프로토콜

제3자 검증자는 `repeat` 횟수만큼 실험을 반복하고 아래를 제출합니다.

- `metrics.json`
- `report.md`
- `run_meta.json`

권장 패키지:

- `argus_result.zip` (또는 `argus_result.sanitized.zip`)

## Sanitization

`argus export <run_dir> --sanitize`는 다음을 제거/치환합니다.

- 사용자 계정명
- hostname
- 절대 사용자 홈 경로
- 환경변수 할당 문자열
- IPv4 주소

## 프라이버시 문구

Argus는 로컬 파일 경로, 사용자 계정명, 환경변수 값을 주요 실험 지표로 수집하지 않는다.
