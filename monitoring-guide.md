# 메타 모니터링 (VM으로 VM 관찰)

VictoriaMetrics 자체의 건강 상태를 VictoriaMetrics로 수집하고 모니터링합니다.
각 컴포넌트는 `/metrics` 엔드포인트에서 Prometheus 형식의 메트릭을 노출합니다.

---

## 메타 모니터링 구조

```
vmstorage /metrics
vminsert  /metrics     ──▶  vmagent  ──▶  vminsert  ──▶  vmstorage
vmselect  /metrics                              (자기 자신을 모니터링)
vmalert   /metrics
vmagent   /metrics

Grafana ◀── vmselect (VictoriaMetrics 대시보드 조회)
```

---

## 1. VM 컴포넌트 메트릭 수집 설정

VM Operator를 사용하는 경우 `VMServiceMonitor`로 자동 수집됩니다.
`victoria-metrics-k8s-stack` Chart는 기본적으로 자체 메트릭을 수집합니다.

수동으로 추가해야 하는 경우:

```yaml
# kubernetes/vm-self-monitoring.yaml
---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceMonitor
metadata:
  name: vmstorage-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: vmstorage
  endpoints:
    - port: http
      path: /metrics
      interval: 30s

---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceMonitor
metadata:
  name: vminsert-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: vminsert
  endpoints:
    - port: http
      path: /metrics
      interval: 30s

---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceMonitor
metadata:
  name: vmselect-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: vmselect
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

---

## 2. 핵심 모니터링 메트릭

### vmstorage 핵심 메트릭

```promql
# 디스크 사용량
vm_data_size_bytes{type="storage"}

# 디스크 여유 공간
vm_available_disk_space_bytes

# 디스크 사용률 (85% 이상이면 경고)
vm_data_size_bytes{type="storage"} / vm_available_disk_space_bytes * 100

# 활성 시계열 수 (캐시에 보관 중)
vm_cache_entries{type="storage/tsid"}

# 초당 수신 행 수 (쓰기 처리량)
rate(vm_rows_added_to_storage_total[5m])

# 백그라운드 병합 속도 (정상: > 0)
rate(vm_rows_merged_total[5m])

# 데이터 파트 수 (높을수록 병합 필요)
vm_parts_count{type="storage/big"}
vm_parts_count{type="storage/small"}
```

### vminsert 핵심 메트릭

```promql
# 초당 처리 요청 수
rate(vm_vminsert_api_requests_total[5m])

# 5xx 오류율
rate(vm_vminsert_api_requests_total{status=~"5.."}[5m])
  /
rate(vm_vminsert_api_requests_total[5m])

# 요청 레이턴시 (P99)
histogram_quantile(0.99,
  rate(vm_request_duration_seconds_bucket{component="vminsert"}[5m])
)
```

### vmselect 핵심 메트릭

```promql
# 초당 쿼리 수
rate(vm_vmselect_api_requests_total[5m])

# 쿼리 오류율
rate(vm_vmselect_api_requests_total{status=~"5.."}[5m])
  /
rate(vm_vmselect_api_requests_total[5m])

# 쿼리 레이턴시 P50 / P95 / P99
histogram_quantile(0.50, rate(vm_request_duration_seconds_bucket{component="vmselect"}[5m]))
histogram_quantile(0.95, rate(vm_request_duration_seconds_bucket{component="vmselect"}[5m]))
histogram_quantile(0.99, rate(vm_request_duration_seconds_bucket{component="vmselect"}[5m]))

# 캐시 히트율 (높을수록 좋음)
rate(vm_cache_requests_total{type="vmselect"}[5m])
  - rate(vm_cache_misses_total{type="vmselect"}[5m])
  /
rate(vm_cache_requests_total{type="vmselect"}[5m])
```

### vmagent 핵심 메트릭

```promql
# 스크래핑 성공률
rate(vm_promscrape_scraped_samples_sum[5m])
  /
(rate(vm_promscrape_scraped_samples_sum[5m]) + rate(vm_promscrape_scrape_failed_total[5m]))

# WAL 미전송 데이터 크기 (0에 가까울수록 좋음)
vm_remotewrite_pending_data_bytes

# remote_write 지연 시간 (초)
vm_remotewrite_send_duration_seconds

# 활성 타겟 수
vm_promscrape_active_targets_count
```

---

## 3. Grafana 대시보드

### 공식 대시보드 임포트

```bash
# Grafana UI → Dashboards → Import

# VictoriaMetrics Cluster 상태
ID: 11176

# vmagent 상태
ID: 12683

# VictoriaMetrics Single (단일 노드용)
ID: 10229
```

### 수동 대시보드 패널 예시

```json
{
  "title": "vmstorage 디스크 사용량",
  "type": "gauge",
  "targets": [
    {
      "expr": "vm_data_size_bytes{type=\"storage\"} / vm_available_disk_space_bytes * 100",
      "legendFormat": "{{ instance }}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "thresholds": {
        "steps": [
          {"color": "green", "value": 0},
          {"color": "yellow", "value": 70},
          {"color": "red", "value": 85}
        ]
      },
      "unit": "percent",
      "max": 100
    }
  }
}
```

---

## 4. 핵심 SLI/SLO 정의

| SLI | PromQL | SLO |
|---|---|---|
| 쓰기 성공률 | `1 - rate(vm_vminsert_api_requests_total{status=~"5.."}[5m]) / rate(vm_vminsert_api_requests_total[5m])` | > 99.9% |
| 읽기 성공률 | `1 - rate(vm_vmselect_api_requests_total{status=~"5.."}[5m]) / rate(vm_vmselect_api_requests_total[5m])` | > 99.9% |
| P99 쿼리 레이턴시 | `histogram_quantile(0.99, rate(vm_request_duration_seconds_bucket{component="vmselect"}[5m]))` | < 5초 |
| scrape 성공률 | `rate(vm_promscrape_scraped_samples_sum[5m]) / (rate(vm_promscrape_scraped_samples_sum[5m]) + rate(vm_promscrape_scrape_failed_total[5m]))` | > 99% |
| 디스크 여유 공간 | `vm_available_disk_space_bytes / (vm_data_size_bytes{type="storage"} + vm_available_disk_space_bytes)` | > 20% |

---

## 5. 메트릭 수집 확인

```bash
# 각 컴포넌트 메트릭 엔드포인트 확인
kubectl port-forward svc/vmstorage-vm-stack-victoria-metrics-k8s-stack 8482:8482 -n monitoring &
curl -s http://localhost:8482/metrics | wc -l

kubectl port-forward svc/vminsert-vm-stack-victoria-metrics-k8s-stack 8480:8480 -n monitoring &
curl -s http://localhost:8480/metrics | wc -l

kubectl port-forward svc/vmselect-vm-stack-victoria-metrics-k8s-stack 8481:8481 -n monitoring &
curl -s http://localhost:8481/metrics | wc -l

# vmagent가 VM 자체 메트릭을 수집 중인지 확인
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=vm_data_size_bytes" | \
  jq '.data.result | length'
# 0이면 VM 자체 메트릭 수집 안 됨
```

---

## 6. 용량 계획 (Capacity Planning)

```promql
# 현재 vmstorage 디스크 증가 속도 (bytes/day)
rate(vm_data_size_bytes{type="storage"}[24h]) * 86400

# 현재 증가 속도 기준 디스크 full 예상 시간 (일)
vm_available_disk_space_bytes
  /
rate(vm_data_size_bytes{type="storage"}[24h]) / 86400

# 활성 시계열 수 추이
vm_cache_entries{type="storage/tsid"}

# 초당 샘플 수신 추이
rate(vm_rows_added_to_storage_total[5m])
```

---

## 트러블슈팅

```bash
# VM 컴포넌트 메트릭이 수집 안될 때
kubectl get vmservicemonitor -n monitoring
kubectl describe vmservicemonitor vmstorage-monitor -n monitoring

# vmagent가 VM 타겟을 발견했는지 확인
kubectl port-forward svc/vmagent-vm-stack-victoria-metrics-k8s-stack 8429:8429 -n monitoring &
curl http://localhost:8429/api/v1/targets | \
  jq '.data.activeTargets[] | select(.labels.job | contains("vm"))'
```
