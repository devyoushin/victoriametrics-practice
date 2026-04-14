# Recording Rules

vmalert가 Recording Rules를 주기적으로 평가하여 새 메트릭을 생성합니다.
자주 사용하는 무거운 쿼리를 미리 계산해두면 Grafana 대시보드 성능이 크게 향상됩니다.

---

## 전체 흐름

```
vmalert
    │ PromQL 평가 (기본 1분 주기)
    │ vmselect에서 데이터 조회
    ▼
새 메트릭 생성
    │ remote_write
    ▼
vminsert → vmstorage (영속 저장)
    │
    ▼
Grafana (Recording Rule 결과 조회)
```

---

## 1. VMRule CRD로 Recording Rules 작성

VM Operator를 사용하는 경우 `VMRule` CRD로 규칙을 관리합니다:

```yaml
# kubernetes/vmalert.yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: recording-rules
  namespace: monitoring
spec:
  groups:
    # ─────────────────────────────────────────────
    # HTTP 서비스 메트릭 사전 계산
    # ─────────────────────────────────────────────
    - name: http.recording
      interval: 1m
      rules:
        # 초당 요청 수 (job별)
        - record: job:http_requests_total:rate5m
          expr: |
            sum by (job) (
              rate(http_requests_total[5m])
            )

        # 에러율 (job별)
        - record: job:http_requests_errors:rate5m
          expr: |
            sum by (job) (
              rate(http_requests_total{status=~"5.."}[5m])
            )
            /
            sum by (job) (
              rate(http_requests_total[5m])
            )

        # P50 레이턴시 (job별)
        - record: job:http_request_duration_seconds:p50
          expr: |
            histogram_quantile(0.5,
              sum by (job, le) (
                rate(http_request_duration_seconds_bucket[5m])
              )
            )

        # P95 레이턴시 (job별)
        - record: job:http_request_duration_seconds:p95
          expr: |
            histogram_quantile(0.95,
              sum by (job, le) (
                rate(http_request_duration_seconds_bucket[5m])
              )
            )

        # P99 레이턴시 (job별)
        - record: job:http_request_duration_seconds:p99
          expr: |
            histogram_quantile(0.99,
              sum by (job, le) (
                rate(http_request_duration_seconds_bucket[5m])
              )
            )

    # ─────────────────────────────────────────────
    # Kubernetes 리소스 사전 계산
    # ─────────────────────────────────────────────
    - name: kubernetes.recording
      interval: 1m
      rules:
        # 네임스페이스별 CPU 사용량
        - record: namespace:container_cpu_usage_seconds:sum_rate5m
          expr: |
            sum by (namespace) (
              rate(container_cpu_usage_seconds_total{container!=""}[5m])
            )

        # 네임스페이스별 메모리 사용량
        - record: namespace:container_memory_working_set_bytes:sum
          expr: |
            sum by (namespace) (
              container_memory_working_set_bytes{container!=""}
            )

        # 파드별 CPU 사용률 (limit 대비)
        - record: pod:container_cpu_usage:ratio
          expr: |
            sum by (namespace, pod) (
              rate(container_cpu_usage_seconds_total{container!=""}[5m])
            )
            /
            sum by (namespace, pod) (
              container_spec_cpu_quota{container!=""} / 100000
            )

        # 파드별 메모리 사용률 (limit 대비)
        - record: pod:container_memory_usage:ratio
          expr: |
            sum by (namespace, pod) (
              container_memory_working_set_bytes{container!=""}
            )
            /
            sum by (namespace, pod) (
              container_spec_memory_limit_bytes{container!=""}
            )

    # ─────────────────────────────────────────────
    # VictoriaMetrics 자체 메트릭 사전 계산
    # ─────────────────────────────────────────────
    - name: victoriametrics.recording
      interval: 1m
      rules:
        # vminsert 초당 수신 샘플 수
        - record: vm:vminsert_samples:rate5m
          expr: |
            sum(rate(vm_vminsert_api_requests_total[5m]))

        # vmstorage 활성 시계열 수
        - record: vm:vmstorage_active_time_series:sum
          expr: |
            sum(vm_cache_entries{type="storage/tsid"})

        # vmstorage 디스크 사용률
        - record: vm:vmstorage_disk_usage:ratio
          expr: |
            vm_data_size_bytes{type="storage"}
            /
            vm_available_disk_space_bytes
```

---

## 2. 파일 기반 Rules 작성 (VMRule 없이)

VM Operator를 사용하지 않는 경우 파일로 관리합니다:

```yaml
# recording-rules.yaml
groups:
  - name: http.recording
    interval: 1m
    rules:
      - record: job:http_requests_total:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

```bash
# vmalert에 규칙 파일 마운트 (ConfigMap 방식)
kubectl create configmap recording-rules \
  --from-file=recording-rules.yaml \
  -n monitoring
```

---

## 3. Rules 적용 확인

```bash
# vmalert 포트 포워드
kubectl port-forward svc/vmalert-vm-stack-victoria-metrics-k8s-stack 8880:8880 -n monitoring &

# 적용된 규칙 목록 확인
curl http://localhost:8880/api/v1/rules | jq '.data.groups[].rules[] | select(.type=="recording") | .name'

# 특정 Recording Rule 쿼리 테스트
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=job:http_requests_total:rate5m" | \
  jq '.data.result'

# 규칙 평가 오류 확인
curl http://localhost:8880/api/v1/rules | \
  jq '.data.groups[].rules[] | select(.health != "ok") | {name: .name, error: .lastError}'
```

---

## 4. Grafana에서 Recording Rule 활용

```promql
# ❌ 무거운 원본 쿼리 (1주일 범위에서 느림)
histogram_quantile(0.99,
  sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
)

# ✅ Recording Rule 결과 활용 (빠름)
job:http_request_duration_seconds:p99

# 트렌드 분석 (Recording Rule + 범위 쿼리)
job:http_requests_total:rate5m[7d:5m]
```

---

## 5. Recording Rule 네이밍 컨벤션

```
{레이블 집합}:{메트릭 이름}:{연산}

예시:
  job:http_requests_total:rate5m
  └──┘ └─────────────────┘ └────┘
  레이블   원본 메트릭 이름   집계 함수 + 시간 범위

  namespace:container_cpu_usage_seconds:sum_rate5m
  pod:container_memory_usage:ratio
  cluster:node_cpu_utilization:avg
```

---

## 트러블슈팅

```bash
# vmalert 로그 확인
kubectl logs -n monitoring -l app=vmalert --since=30m | grep -i "error\|fail"

# Recording Rule 평가 간격 확인
curl http://localhost:8880/api/v1/rules | \
  jq '.data.groups[] | {name: .name, interval: .interval}'

# 특정 Recording Rule이 생성되지 않는 경우
# 1. 원본 메트릭이 존재하는지 확인
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=http_requests_total" | \
  jq '.data.result | length'

# 2. 규칙 문법 확인 (vmalert로 dry-run)
vmalert -rule recording-rules.yaml -dryRun
```

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| Recording 메트릭 없음 | 원본 메트릭 미존재 | scraping 대상 확인 |
| `evaluation failed` | PromQL 문법 오류 | `vmalert -dryRun`으로 사전 검증 |
| Recording 메트릭 지연 | vmalert ↔ vminsert 지연 | vmalert 리소스 증가 |
| 오래된 데이터가 계속 보임 | `interval` 설정 너무 김 | `interval: 1m` 으로 단축 |
