# 모니터링 규칙 — victoriametrics-practice

## VictoriaMetrics 자체 모니터링

### 핵심 메트릭

```metricsql
# 활성 시계열 수
vm_new_timeseries_created_total

# 수집 속도
rate(vm_rows_inserted_total[5m])

# 디스크 읽기/쓰기 속도
rate(vm_data_size_bytes{type="storage/big"}[5m])

# 쿼리 지연 (p99)
histogram_quantile(0.99, rate(vm_request_duration_seconds_bucket[5m]))

# 캐시 히트율
vm_cache_requests_total{type="indexdb/tagFilters"} / vm_cache_misses_total{type="indexdb/tagFilters"}
```

### 알림 규칙 (권장)

| 알림 | 조건 | 심각도 |
|------|------|--------|
| VMDown | `up{job="victoriametrics"} == 0` | critical |
| VMStorageHigh | 디스크 90% 초과 | critical |
| VMSlowQuery | 쿼리 p99 > 30s | warning |
| VMHighCardinality | 시계열 수 급증 | warning |

## SLO 정의 (VictoriaMetrics 서비스)

| SLI | 목표 | 측정 방법 |
|-----|------|---------|
| 가용성 | 99.9% | `up{job="victoriametrics"}` |
| 수집 성공률 | 99.5% | 수집 오류율 |
| 쿼리 p99 지연 | < 30초 | `vm_request_duration_seconds` |

## 대시보드 구성 (Grafana)
- **Overview**: 시계열 수, 수집 속도, 쿼리 수
- **Storage**: 디스크 사용량, 파티션 상태
- **Queries**: 쿼리 성능, 캐시 히트율
- **vmagent**: 스크레이핑 현황, 큐 상태
- **vmalert**: 알림 평가 현황
