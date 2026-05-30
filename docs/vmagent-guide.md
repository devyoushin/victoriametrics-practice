# vmagent — 메트릭 수집 가이드

vmagent는 Prometheus와 호환되는 경량 스크래핑 에이전트입니다.
Prometheus 대비 메모리 사용량이 훨씬 적으며, WAL로 원격 저장소 장애를 안전하게 처리합니다.

---

## vmagent vs Prometheus 비교

| 항목 | vmagent | Prometheus |
|---|---|---|
| 메모리 사용량 | 낮음 (WAL 기반) | 높음 (인메모리 TSDB) |
| 로컬 쿼리 | 불가 (에이전트 전용) | 가능 |
| 원격 저장소 장애 처리 | WAL로 자동 재전송 | 제한적 |
| 다중 원격 저장소 | 지원 (fan-out) | 지원 |
| Kubernetes CRD | VMServiceMonitor 등 지원 | 별도 Operator 필요 |
| 권장 사용 사례 | 대규모 scraping, 에지 수집 | 소규모, 로컬 쿼리 필요 시 |

---

## 전체 흐름

```
Kubernetes Pods/Services/Nodes
    │ scrape (HTTP GET /metrics)
    ▼
vmagent
    ├── WAL (로컬 디스크) ← 원격 저장소 장애 시 버퍼
    │
    └── remote_write (Snappy 압축 protobuf)
            │
            ▼
        vminsert:8480/insert/0/prometheus/api/v1/write
```

---

## 1. vmagent Helm values 설정

```yaml
# ../ops/config/helm/values.yaml
victoria-metrics-k8s-stack:
  vmagent:
    enabled: true
    spec:
      # 수집 대상: 모든 ServiceMonitor/PodMonitor 자동 선택
      selectAllByDefault: true

      # scrape 간격
      scrapeInterval: 30s

      # WAL 설정 (원격 저장소 장애 시 로컬 버퍼)
      replicaCount: 1

      # vmagent 리소스
      resources:
        requests:
          cpu: 200m
          memory: 256Mi
        limits:
          cpu: 1
          memory: 1Gi

      # WAL 용 PVC (원격 저장소 장애 시 데이터 보존)
      volumeMounts:
        - name: vmagentdata
          mountPath: /tmp/vmagent-remotewrite-data

      volumes:
        - name: vmagentdata
          persistentVolumeClaim:
            claimName: vmagent-data

      # remote_write 대상 (vminsert)
      remoteWrite:
        - url: http://vminsert-vm-stack-victoria-metrics-k8s-stack:8480/insert/0/prometheus/api/v1/write
```

---

## 2. ServiceMonitor로 수집 대상 등록

VM Operator는 Prometheus Operator의 ServiceMonitor와 호환되는 `VMServiceMonitor`를 제공합니다.

### 기본 ServiceMonitor 예시

```yaml
# ../ops/config/kubernetes/vmagent.yaml (ServiceMonitor 섹션)
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceMonitor
metadata:
  name: my-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app           # 수집할 Service의 레이블
  namespaceSelector:
    matchNames:
      - default
      - production
  endpoints:
    - port: metrics          # Service의 포트 이름
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

### 인증이 필요한 엔드포인트

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceMonitor
metadata:
  name: secure-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: secure-app
  endpoints:
    - port: metrics
      path: /metrics
      scheme: https
      tlsConfig:
        insecureSkipVerify: true    # 내부 서비스의 경우
      bearerTokenSecret:
        name: app-metrics-token
        key: token
```

### 노드 메트릭 수집 (node-exporter)

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMNodeScrape
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  path: /metrics
  port: "9100"
  interval: 30s
  relabelings:
    - sourceLabels: [__meta_kubernetes_node_name]
      targetLabel: node
```

---

## 3. Prometheus 호환 scrape_configs 사용

기존 Prometheus `scrape_configs`를 그대로 사용할 수 있습니다:

```yaml
# ../ops/config/helm/values.yaml
victoria-metrics-k8s-stack:
  vmagent:
    spec:
      additionalScrapeConfigs:
        name: additional-scrape-configs
        key: prometheus-additional.yaml

---
# ConfigMap (별도 생성)
apiVersion: v1
kind: ConfigMap
metadata:
  name: additional-scrape-configs
  namespace: monitoring
data:
  prometheus-additional.yaml: |
    - job_name: 'external-service'
      static_configs:
        - targets:
            - external-host:9090
      metrics_path: /metrics
      scheme: http
```

---

## 4. 메트릭 필터링 및 리레이블링

불필요한 메트릭을 제거하거나 레이블을 변환합니다:

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceMonitor
metadata:
  name: my-app-filtered
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
      # 수집 시 메트릭 필터링 (불필요한 메트릭 제거)
      metricRelabelings:
        # 필요한 메트릭만 유지
        - sourceLabels: [__name__]
          regex: "http_requests_total|http_request_duration.*|up"
          action: keep

        # 불필요한 레이블 제거
        - regex: "go_.*|process_.*"
          sourceLabels: [__name__]
          action: drop

        # 레이블 이름 변경
        - sourceLabels: [kubernetes_pod_name]
          targetLabel: pod
          action: replace

      # 타겟 레이블 추가
      relabelings:
        - targetLabel: cluster
          replacement: my-eks-cluster
        - sourceLabels: [__meta_kubernetes_namespace]
          targetLabel: namespace
```

---

## 5. vmagent 상태 확인

```bash
# vmagent 포트 포워드
kubectl port-forward svc/vmagent-vm-stack-victoria-metrics-k8s-stack 8429:8429 -n monitoring &

# vmagent UI (scrape 상태 시각화)
# 브라우저: http://localhost:8429

# scrape 상태 확인 (API)
curl http://localhost:8429/api/v1/targets | jq '.data.activeTargets | length'

# 개별 타겟 상태
curl http://localhost:8429/api/v1/targets | \
  jq '.data.activeTargets[] | {job: .labels.job, health: .health, lastError: .lastError}'

# WAL 상태 (원격 저장소 지연)
curl http://localhost:8429/metrics | grep -E "vm_remotewrite|vm_rows"
```

---

## 6. WAL (Write-Ahead Log) 관리

```bash
# WAL 현재 용량 확인
kubectl exec -n monitoring deploy/vmagent-vm-stack-victoria-metrics-k8s-stack \
  -- du -sh /tmp/vmagent-remotewrite-data

# WAL 관련 메트릭 확인 (원격 저장소 전송 지연)
curl http://localhost:8429/metrics | \
  grep -E "vm_remotewrite_send_duration|vm_remotewrite_packets_sent"

# WAL 지연 시각화 PromQL
# vmagent가 vminsert에 보내지 못하고 WAL에 쌓인 행 수
# vm_remotewrite_pending_data_bytes / vm_remotewrite_bytes_sent_total
```

> **WAL 용 PVC 크기**: remote_write 대상이 장기간 다운될 경우를 대비해
> 최소 1시간치 메트릭을 버퍼링할 수 있도록 설정하세요.
> (활성 시계열 100,000 × 30초 간격 × 1시간 ≈ 약 1GB)

---

## 7. Prometheus에서 vmagent로 마이그레이션

기존 Prometheus를 vmagent로 교체하는 경우:

```yaml
# Prometheus remote_write 유지 + vmagent scraping 전환 단계

# 1단계: vmagent 배포 (vmagent도 vminsert로 remote_write)
# 2단계: Prometheus와 vmagent 병렬 운영 (중복 수집 허용)
# 3단계: Prometheus remote_write 제거
# 4단계: Prometheus 종료

# vmagent에서 기존 Prometheus scrape_configs 재사용
spec:
  additionalScrapeConfigs:
    name: prometheus-scrape-configs
    key: scrape_configs.yaml
```

---

## 8. 다중 원격 저장소 (fan-out)

```yaml
spec:
  remoteWrite:
    # 기본 VictoriaMetrics
    - url: http://vminsert:8480/insert/0/prometheus/api/v1/write
      queueCount: 4
      maxBlockSize: 8388608      # 8MB

    # 장기 보존용 별도 VictoriaMetrics (예: 1년 보존)
    - url: http://vminsert-longterm:8480/insert/0/prometheus/api/v1/write
      urlRelabelConfig:
        name: longterm-filter
        key: relabel.yaml       # 특정 메트릭만 장기 보존소로 전송
```

---

## 트러블슈팅

```bash
# scrape 실패 타겟 확인
curl http://localhost:8429/api/v1/targets | \
  jq '.data.activeTargets[] | select(.health == "down") | {job: .labels.job, error: .lastError}'

# vmagent 로그 확인
kubectl logs -n monitoring deploy/vmagent-vm-stack-victoria-metrics-k8s-stack --since=1h \
  | grep -i "error\|fail"

# vminsert 연결 확인
kubectl exec -n monitoring deploy/vmagent-vm-stack-victoria-metrics-k8s-stack \
  -- wget -qO- http://vminsert-vm-stack-victoria-metrics-k8s-stack:8480/health
```

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| 타겟 `down` 상태 | 포트/경로 설정 오류 | ServiceMonitor port, path 재확인 |
| WAL 용량 급증 | vminsert 장애 | vminsert 상태 확인 |
| `scrape timeout` 빈발 | 엔드포인트 응답 느림 | `scrapeTimeout` 값 증가 |
| 레이블 카디널리티 과다 | 불필요한 레이블 수집 | `metricRelabelings` 로 제거 |
| 메트릭 수집 안됨 | ServiceMonitor 셀렉터 불일치 | Service 레이블과 selector 비교 확인 |
