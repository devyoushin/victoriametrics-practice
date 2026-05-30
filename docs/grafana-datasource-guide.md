# Grafana 연동 (Read Path)

VictoriaMetrics는 Prometheus와 완전히 호환되는 HTTP API를 제공합니다.
Grafana에서 데이터소스를 `Prometheus` 타입으로 추가하면 됩니다.

---

## 전체 흐름

```
Grafana
    │ PromQL 쿼리
    │ HTTP GET /select/0/prometheus/api/v1/...
    ▼
vmselect:8481
    │ fan-out (병렬)
    ├──▶ vmstorage-0
    └──▶ vmstorage-1
결과 병합 + 중복 제거 → Grafana 반환
```

---

## 1. Grafana 접속

```bash
# Grafana 포트 포워드
kubectl port-forward svc/vm-stack-grafana 3000:80 -n monitoring

# 기본 로그인 정보
# ID: admin
# PW: helm values의 grafana.adminPassword (기본: admin)
```

---

## 2. VictoriaMetrics 데이터소스 추가

### Grafana UI 설정

```
Configuration → Data Sources → Add data source → Prometheus

Name: VictoriaMetrics
URL: http://vmselect-vm-stack-victoria-metrics-k8s-stack:8481/select/0/prometheus/
(Grafana가 Kubernetes 내부에 있는 경우 Service DNS 사용)

또는 포트 포워드 경유:
URL: http://localhost:8481/select/0/prometheus/

Access: Server (기본값)
Scrape interval: 30s
Query timeout: 60s
HTTP method: POST   ← 긴 쿼리에서 URL 길이 초과 방지

Save & Test 클릭 → "Data source is working" 확인
```

### Grafana Provisioning (코드로 관리)

```yaml
# ../ops/config/helm/values.yaml (Grafana datasource 자동 생성)
grafana:
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: VictoriaMetrics
          type: prometheus
          url: http://vmselect-vm-stack-victoria-metrics-k8s-stack:8481/select/0/prometheus/
          access: proxy
          isDefault: true
          jsonData:
            timeInterval: 30s
            queryTimeout: "60"
            httpMethod: POST
```

---

## 3. vmselect URL 구조

VictoriaMetrics Cluster는 URL에 테넌트 ID를 포함합니다:

```
http://vmselect:8481/select/{accountID}/prometheus/{endpoint}

accountID 예시:
  0         → 기본 테넌트 (단일 테넌트 환경)
  42        → 테넌트 ID 42
  multitenant → 모든 테넌트 통합 조회 (주의: 권한 관리 필요)
```

---

## 4. PromQL 쿼리 예시

VictoriaMetrics는 Prometheus PromQL을 완전히 지원합니다.
추가로 MetricsQL(VictoriaMetrics 확장 문법)도 지원합니다.

### 기본 PromQL

```promql
# 서비스 가용성 (scrape 성공률)
sum by (job) (up)

# HTTP 요청 처리율 (초당)
sum by (job) (rate(http_requests_total[5m]))

# HTTP 5xx 에러율
sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
  /
sum by (job) (rate(http_requests_total[5m]))

# P99 레이턴시
histogram_quantile(0.99,
  sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
)
```

### MetricsQL 확장 문법 (VictoriaMetrics 전용)

```promql
# default_rollup: 데이터 간격에 자동으로 맞는 롤업
default_rollup(http_requests_total)

# 이상 감지 (zscore 기반)
zscore(rate(http_requests_total[1h]))

# running_max: 누적 최대값
running_max(rate(http_requests_total[5m]))

# quantiles: 여러 분위수를 한 번에
quantiles("quantile", 0.5, 0.9, 0.99)(http_request_duration_seconds_bucket)
```

---

## 5. 권장 Grafana 대시보드

### VictoriaMetrics 공식 대시보드 (Grafana.com)

```bash
# Grafana UI → Dashboards → Import → 아래 ID 입력

# VictoriaMetrics Cluster 개요
ID: 11176

# vmagent 상태
ID: 12683

# Kubernetes 클러스터 모니터링 (VictoriaMetrics 버전)
ID: 14205
```

### 임포트 방법

```
Dashboards → + Import
→ Grafana.com Dashboard ID 입력 → Load
→ VictoriaMetrics 데이터소스 선택 → Import
```

---

## 6. 알림 채널 (Alert Contact Points) 설정

```yaml
# Grafana Provisioning으로 Alertmanager 연동
grafana:
  alerting:
    contactpoints.yaml:
      apiVersion: 1
      contactPoints:
        - name: Alertmanager
          receivers:
            - uid: alertmanager-receiver
              type: alertmanager
              settings:
                url: http://vmalertmanager-vm-stack:9093
                basicAuthUser: ""
                basicAuthPassword: ""
```

---

## 7. 멀티 테넌트 데이터소스 분리

팀별로 데이터소스를 분리할 수 있습니다:

```yaml
grafana:
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        # 팀 A용 데이터소스
        - name: VictoriaMetrics-TeamA
          type: prometheus
          url: http://vmselect:8481/select/1/prometheus/
          orgId: 1

        # 팀 B용 데이터소스
        - name: VictoriaMetrics-TeamB
          type: prometheus
          url: http://vmselect:8481/select/2/prometheus/
          orgId: 2
```

---

## 8. 쿼리 성능 최적화

### vmselect 캐시 활성화

```yaml
# ../ops/config/helm/values.yaml
vmcluster:
  spec:
    vmselect:
      # 쿼리 결과 캐시 (인메모리)
      extraArgs:
        cacheDataPath: /cache
        # 캐시 크기 (메모리의 10% 권장)

      # 캐시용 PVC
      cacheMountPath: /cache
      storage:
        volumeClaimTemplate:
          spec:
            storageClassName: gp3
            resources:
              requests:
                storage: 10Gi
```

### 느린 쿼리 최적화 팁

```promql
# ❌ 느린 쿼리: 긴 시간 범위 + 고카디널리티
sum by (pod) (rate(http_requests_total[5m]))[7d:5m]

# ✅ 빠른 쿼리: Recording Rule로 미리 계산
# (recording-rules-guide.md 참조)
http_requests:rate5m  # Recording Rule 결과 사용

# ✅ 시간 범위 제한
sum by (job) (rate(http_requests_total[5m]))   # 최근 데이터만

# ✅ 레이블 필터링 먼저 적용
http_requests_total{job="my-service", status="200"}
```

---

## 트러블슈팅

```bash
# vmselect 로그 확인
kubectl logs -n monitoring -l app=vmselect --since=10m | grep -i "error\|slow"

# 느린 쿼리 확인
kubectl logs -n monitoring -l app=vmselect | grep "slow query" | tail -10

# vmselect API 직접 테스트
kubectl port-forward svc/vmselect-vm-stack-victoria-metrics-k8s-stack 8481:8481 -n monitoring &
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=up" | jq '.data.result | length'

# 수집된 메트릭 이름 목록 확인
curl "http://localhost:8481/select/0/prometheus/api/v1/label/__name__/values" | jq '.data | length'
```

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| Grafana `No data` | vmselect URL 또는 accountID 오류 | URL 포맷 재확인 (`/select/0/prometheus/`) |
| 쿼리 타임아웃 | 긴 시간 범위 쿼리 | Recording Rule 사용 또는 시간 범위 축소 |
| 데이터 불일치 | 중복 제거 미작동 | `replicationFactor` 설정 확인 |
| `connection refused` | vmselect Service 이름 오류 | `kubectl get svc -n monitoring` 으로 확인 |
