# Argus v1 주장과 한계

![Argus Compact Logo](../argus_logo/argus_compact_logo.png)

Argus는 통제된 프로토콜 설정에서 비교 가능한 실행 행태를 기록합니다.

## 관측 오버헤드

Argus는 런타임 관측 오버헤드를 유발한다.

도구는 절대 하드웨어 성능이 아니라, 동일한 관측 조건에서의 비교 행태를 측정한다.

## 실패 처리

실패는 관측 결과(`partial_failure`)로 기록되며 숨기지 않습니다.

Argus는 실패를 개선이나 악화로 해석하지 않는다.

Argus는 운영 행태를 기록한다.
