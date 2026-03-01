# Argus v1 실험 규칙

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

## 핵심 필드 (`config.yaml`)

- `seed`
- `steps`
- `repeat`
- `warmup_steps`
- `ricci: [off, on]`
- `workload.nodes`
- `workload.requests_per_step`

## Warmup 정책

warmup 구간은 실행하지만 최종 통계에서 제외합니다.

초기 지연은 시스템의 특성이 아닌 런타임 초기화 비용으로 간주하며 관측 대상에서 제외한다.

예:

- 런타임 초기화
- 캐시 안정화
- allocator/JIT warmup

## 통계 출력 요구사항

각 지표에 대해 Argus는 아래를 기록합니다.

- Mean
- Std Dev
- Min
- Max

목적은 차이가 실제 반복 관측 현상인지, 노이즈인지 구분하는 것입니다.
