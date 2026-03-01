# Argus 재현성 가이드

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

## 1. 목적 (Purpose)

이 문서는 Argus 측정 기록의 재현 프로토콜을 정의한다.

이 문서가 증명하는 것:

- 동일 프로토콜에서 실행 관측 기록을 반복 생성할 수 있음
- 측정 산출물을 생성하고 보존할 수 있음
- 기존 기록에서 보고서를 다시 생성할 수 있음

이 문서가 증명하지 않는 것:

- 애플리케이션 속도 개선
- 하드웨어 최적화 효과
- 모델 품질 변화

## 2. 지원 환경 (Supported Environment)

- OS: Linux, macOS
- 아키텍처: x86_64, arm64
- 최소 CPU: 2코어
- 최소 메모리: 4 GB RAM
- GPU: 필요 없음

특별한 데이터셋이나 모델은 필요하지 않다.

## 3. 설치 방법 (프리빌트 바이너리)

이 프로토콜에서는 소스 빌드 대신 프리빌트 바이너리를 사용한다.

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

설치 확인:

```bash
argus doctor
```

## 4. 최소 실행 절차 (5분 이내)

작은 테스트용 설정 파일 생성:

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

단일 명령 실행:

```bash
argus run ./config.yaml
```

Argus는 run directory를 자동 생성하고 경로를 출력한다.

## 5. 생성 결과물 (Expected Artifacts)

검증 포인트는 숫자 우열이 아니라 기록 파일 생성 여부다.

run directory에서 확인할 파일:

- `metrics.json`: 원시 측정 지표
- `report.md`: 사람이 읽는 보고서
- `run_meta.json`: 실행 메타데이터와 환경 문맥
- `resolved_config.yaml` (또는 동등한 config 스냅샷): 최종 해석된 실행 설정

## 6. 보고서 재생성 (Report Regeneration)

재측정 없이 기존 run directory에서 보고서를 다시 만들 수 있다.

```bash
argus report <run_dir>
```

이 단계는 기록 기반 보고 재생성 가능성을 검증한다.

## 7. 공유 가능 패키지 (Sanitized Export)

외부 공유용 패키지 생성:

```bash
argus export <run_dir> --sanitize
```

`--sanitize`는 민감 문맥을 제거/치환한다:

- 사용자 계정명
- 절대 사용자 홈 경로
- hostname
- 환경변수 할당 문자열
- IP 주소

이로써 외부 검증 공유가 가능해진다.

## 8. 안정성 판정 (Stability Check)

Argus는 변동성 기록을 포함해 판정 근거를 제공한다.

안정성 관련 출력 항목:

- Mean
- Std Dev
- Min
- Max

변동성이 높으면 Argus는 노이즈 우세 가능성을 표시한다.

## 9. 해석 경계 (Interpretation Boundary)

이 재현 실행이 의미하는 것:

- 해당 환경에서 프로토콜이 정상 실행됨
- 관측 기록이 생성됨
- 동일 run directory로 보고서 재생성이 가능함

이 재현 실행이 의미하지 않는 것:

- 특정 모델 결과에 대한 주장 아님
- 속도 개선 주장 아님
- 특정 하드웨어 우위 주장 아님

## 10. 실패 시 처리 (Failure Handling)

`argus doctor` 실패 시:

- 바이너리/아키텍처 일치 여부 확인 (`exec format error`면 올바른 바이너리 재다운로드)
- 실행 권한 확인 (`chmod +x argus`)
- PATH 확인 (`command -v argus`)

실행 중 권한 제약으로 실패하면:

- 일반 호스트 셸 환경에서 재실행
- 제한된 컨테이너 프로파일에서는 프로토콜 실행을 피함

메모리 부족으로 실패하면:

- `config.yaml`의 `steps`, `repeat` 값을 낮춤
- 다시 실행해 산출물 생성 여부를 확인
