# Execution Instability Mode

English version: [EXECUTION_INSTABILITY_MODE.en.md](EXECUTION_INSTABILITY_MODE.en.md)

## 개요

`execution_instability`는 실행 경로 변동성, 재사용 실패 proxy, 재계산 관련 신호, work inflation을 기록하기 위한 Argus v1 연구 관측 모드이다.

이 모드는 표준 Argus v1 산출물을 유지하면서 `instability_metrics.json`을 추가로 생성한다. 해당 파일은 path, reuse, recomputation, work 관측값을 별도 스키마로 기록한다.

Argus는 이 값을 비인과적 관측 proxy로 기록한다. 즉, path variability가 reuse failure를 유발한다거나, reuse failure가 recomputation을 유발한다거나, recomputation이 latency 변화를 직접 유발한다고 해석하지 않는다.

## 실행 방식

CLI override:

```bash
argus run config.yaml --mode execution_instability
```

Config 기반 실행:

```yaml
seed: 42
steps: 500
repeat: 5
warmup_steps: 20
mode: execution_instability
ricci: [off, on]
workload:
  nodes: 8
  requests_per_step: 128
```

## 산출물 계약

`execution_instability` 실행은 표준 Argus v1 기록과 함께 `instability_metrics.json`을 생성한다.

```text
run_dir/
|-- metrics.json
|-- report.md
|-- run_meta.json
|-- config.resolved.yaml
`-- instability_metrics.json
```

`argus export <run_dir>`는 파일이 존재할 경우 `instability_metrics.json`을 함께 포함한다. `argus export <run_dir> --sanitize`는 동일한 redaction 정책을 적용한 뒤 패키징한다.

## 스키마

Top-level 구조:

```json
{
  "schema_version": "0.1.0",
  "mode": "execution_instability",
  "run_info": {},
  "system_info": {},
  "workload_info": {},
  "scenario_info": {},
  "measurement_window": {},
  "raw_metrics": {},
  "derived_metrics": {},
  "artifacts": {},
  "notes": {}
}
```

Raw metrics:

- `timing.iteration_latency_ms`
- `timing.total_runtime_ms`
- `work_observation.total_operations_observed`
- `work_observation.repeated_operations_observed`
- `work_observation.reference_operations`
- `reuse_observation.reuse_opportunities_observed`
- `reuse_observation.reuse_failures_observed`
- `reuse_observation.intermediate_rebuild_count`
- `path_observation.distinct_path_count`
- `path_observation.ordering_change_count`
- `memory_observation.cache_miss_proxy`
- `memory_observation.reload_events_observed`

Derived metrics:

- latency mean, p95, p99, standard deviation
- `execution_path_variability_score`
- `reuse_failure_rate`
- `recompute_ratio`
- `work_inflation_ratio`

## 해석 경계

이 모드는 최적화나 인과 분석이 아니라 재현 가능한 관측을 위한 모드이다.

CPU, GPU, runtime 환경 전반에서 완전한 execution path와 intermediate-state reuse를 직접 관측하기 어려울 수 있으므로, 스키마는 proxy 기반 관측값을 명시적으로 사용한다.

## Execution-Instability Benchmark와의 관계

Argus는 관측 및 리포팅 레이어이다. 검증 프로토콜을 실행하고, counter와 환경 문맥을 기록하며, 스키마 호환 산출물을 생성하고, 보고서 재생성과 결과 패키징을 담당한다.

외부 execution-instability benchmark는 controlled workload와 scenario definition을 제공할 수 있다. 해당 관측값이 동일한 `instability_metrics.json` 스키마로 표현될 수 있으면 Argus의 측정/보고 레이어와 함께 사용할 수 있다.
