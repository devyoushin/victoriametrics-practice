# VictoriaMetrics 컨벤션 규칙

## 배포 모드 선택 기준
| 모드 | 시계열 수 | 특징 |
|------|---------|------|
| 단일 노드 | ~10M | 단순, 저비용 |
| 클러스터 | 10M+ | 수평 확장, 고가용성 |

## 스토리지 경로 규칙
```bash
# 단일 노드
-storageDataPath=/var/lib/victoria-metrics-data

# 클러스터 vmstorage
-storageDataPath=/var/lib/vmstorage-data
```
- 스토리지 경로는 전용 볼륨 마운트 (시스템 디스크와 분리)
- SSD 권장 (HDD 사용 시 성능 50% 이상 저하)

## vmagent 스크레이핑 컨벤션
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```
- `scrape_interval`: 기본 30s
- `remote_write` 재시도: 지수 백오프 설정
- 큐 설정: `maxDiskUsagePerURL` 1GB 이상

## vmalert 알림 규칙 컨벤션
```yaml
groups:
  - name: vm-alerts
    rules:
      - alert: AlertName
        expr: |
          # MetricsQL 표현식
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }}: 요약"
          runbook_url: "https://..."
```

## 보존 정책
- 기본: `-retentionPeriod=12` (12개월)
- 다운샘플링: VictoriaMetrics Enterprise 기능
- 데이터 삭제: `http://victoriametrics:8428/api/v1/admin/tsdb/delete_series` (주의!)

## vmui 활용
- 쿼리 디버깅: `http://victoriametrics:8428/vmui`
- Cardinality 분석: `http://victoriametrics:8428/vmui/#/cardinality`
- 트레이스 분석: `http://victoriametrics:8428/vmui/#/trace`
