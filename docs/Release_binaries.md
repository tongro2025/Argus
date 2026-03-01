# Argus 릴리즈 Step 4 (사전 빌드 바이너리)

이 문서는 릴리즈 파이프라인의 Step 4,
즉 GitHub Release에 사전 빌드 바이너리를 첨부하는 절차를 설명합니다.

## 워크플로우 파일

- `.github/workflows/release-binaries.yml`

## 실행 트리거

- 자동: `v*` 패턴 태그 push
- 수동: GitHub Actions `workflow_dispatch` (필요 시 `tag` 입력)

## 워크플로우 동작

1. 대상 태그 기준으로 코드 checkout
2. `go.mod` 기준 Go 버전 설정
3. `go test ./...` 실행
4. 크로스 컴파일:
   - `linux/amd64`
   - `darwin/arm64`
5. 재현성 옵션 사용:
   - `-trimpath`
   - `-ldflags "-s -w"`
6. `CGO_ENABLED=0` 우선 빌드, 실패 시 fallback
7. 릴리즈 자산 파일명 생성:
   - `argus-linux-amd64`
   - `argus-macos-arm64`
8. 워크플로우 아티팩트 업로드
9. GitHub Release 생성/업데이트 + 바이너리 자산 첨부

## v1.0.0-validation 실행 방법

```bash
git tag v1.0.0-validation
git push origin v1.0.0-validation
```

워크플로우에 설정된 릴리즈 제목:

- `Argus Validation Protocol v1.0`
