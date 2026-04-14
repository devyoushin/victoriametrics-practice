# 트러블슈팅 가이드

VictoriaMetrics 운영 중 자주 발생하는 문제와 해결 방법입니다.

---

## 빠른 진단 명령어

```bash
# 1. 전체 파드 상태 한눈에 보기
kubectl get pods -n monitoring -o wide | awk '{print $1, $3, $4, $7}' | column -t

# 2. 재시작이 많은 파드 확인
kubectl get pods -n monitoring \
  --sort-by='.status.containerStatuses[0].restartCount' | tail -10

# 3. 컴포넌트별 헬스 체크
kubectl port-forward svc/vminsert-vm-stack-victoria-metrics-k8s-stack 8480:8480 -n monitoring &
kubectl port-forward svc/vmselect-vm-stack-victoria-metrics-k8s-stack 8481:8481 -n monitoring &
kubectl port-forward svc/vmstorage-vm-stack-victoria-metrics-k8s-stack 8482:8482 -n monitoring &

curl http://localhost:8480/health    # vminsert
curl http://localhost:8481/health    # vmselect
curl http://localhost:8482/health    # vmstorage

# 4. 최근 에러 로그 일괄 수집
for COMP in vminsert vmselect vmstorage vmagent vmalert; do
  echo "===== ${COMP} errors ====="
  kubectl logs -n monitoring \
    -l app=${COMP} --since=30m 2>/dev/null \
    | grep -i "error\|fail\|panic" | tail -5
done
```

---

## 범주별 문제 해결

### 1. 설치 / 시작 문제

#### vmstorage `Pending` 상태

```bash
# PVC 상태 확인
kubectl get pvc -n monitoring

# PVC 이벤트 상세 확인
kubectl describe pvc storage-vmstorage-vm-stack-0 -n monitoring

# 주요 원인 1: EBS CSI Driver 미설치
kubectl get daemonset ebs-csi-node -n kube-system
# 없으면 EBS CSI Driver 설치 필요

# 주요 원인 2: StorageClass gp3 없음
kubectl get storageclass
# gp2를 사용하려면 values.yaml에서 storageClassName: gp2로 변경

# 주요 원인 3: AZ 불일치 (WaitForFirstConsumer)
kubectl describe node <NODE_NAME> | grep topology.kubernetes.io/zone
```

#### vminsert `CrashLoopBackOff`

```bash
# 로그 확인
kubectl logs -n monitoring -l app=vminsert --previous

# 주요 원인: vmstorage 주소 오류
kubectl logs -n monitoring -l app=vminsert | grep -i "storage\|connect\|error"

# vmstorage Service 이름 확인
kubectl get svc -n monitoring | grep vmstorage
```

---

### 2. 쓰기 경로 (Write) 문제

#### `remote_write` 실패 (Prometheus 또는 vmagent)

```bash
# vminsert가 요청을 받고 있는지 확인
curl -s http://localhost:8480/metrics | grep vm_vminsert_api_requests_total

# 테스트 데이터 직접 전송
curl -X POST http://localhost:8480/insert/0/prometheus/api/v1/import/prometheus \
  -d 'test_write_check{source="troubleshoot"} 1'

# 전송 확인
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=test_write_check"
```

#### `429 Too Many Requests` (rate limit)

```bash
# vminsert 로그에서 rate limit 관련 로그 확인
kubectl logs -n monitoring -l app=vminsert | grep -i "limit\|rate\|throttle"

# vminsert 리소스 사용량 확인
kubectl top pods -n monitoring -l app=vminsert
```

```yaml
# 해결: vminsert 리소스 증가
vminsert:
  resources:
    limits:
      cpu: "4"
      memory: 4Gi
  # 또는 replicaCount 증가
  replicaCount: 3
```

#### vmagent WAL 급증

```bash
# WAL 크기 확인
kubectl exec -n monitoring deploy/vmagent-vm-stack-victoria-metrics-k8s-stack \
  -- du -sh /tmp/vmagent-remotewrite-data

# remote_write 지연 확인
kubectl port-forward svc/vmagent-vm-stack-victoria-metrics-k8s-stack 8429:8429 -n monitoring &
curl http://localhost:8429/metrics | grep vm_remotewrite_pending

# vminsert 연결 상태 확인
curl http://localhost:8429/metrics | grep vm_remotewrite_send_duration
```

---

### 3. 읽기 경로 (Query) 문제

#### Grafana에서 `No data` 또는 빈 그래프

```bash
# vmselect API 직접 테스트
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=up" | jq '.data.result | length'
# 0이면 데이터 없음

# 원인 1: accountID 불일치
# remote_write URL의 accountID와 Grafana datasource URL의 accountID가 동일해야 함
# vminsert: /insert/0/prometheus/... → vmselect: /select/0/prometheus/...

# 원인 2: 메트릭 이름 확인
curl "http://localhost:8481/select/0/prometheus/api/v1/label/__name__/values" | \
  jq '.data | length'

# 원인 3: 시간 범위 문제
curl "http://localhost:8481/select/0/prometheus/api/v1/query_range?query=up&start=$(date -d '1h ago' +%s)&end=$(date +%s)&step=60"
```

#### 쿼리 타임아웃 (`context deadline exceeded`)

```bash
# 느린 쿼리 로그 확인
kubectl logs -n monitoring -l app=vmselect --since=10m | grep -i "slow\|timeout"

# 해결책 1: 쿼리 시간 범위 줄이기
# Grafana에서 7d → 1d로 축소

# 해결책 2: Recording Rule로 미리 계산 (recording-rules-guide.md 참조)

# 해결책 3: vmselect 캐시 확인
curl http://localhost:8481/metrics | grep vm_cache

# 해결책 4: vmselect 리소스 증가
```

```yaml
vmselect:
  resources:
    limits:
      cpu: "4"
      memory: 8Gi
  replicaCount: 3
```

---

### 4. vmstorage 문제

#### 디스크 용량 부족

```bash
# 현재 디스크 사용량 확인
kubectl exec -n monitoring vmstorage-vm-stack-0 -- df -h /storage

# 데이터 크기 상세
kubectl exec -n monitoring vmstorage-vm-stack-0 -- du -sh /storage/data/*

# 해결책 1: retention 기간 단축
# values.yaml에서 retentionPeriod: "30d" 로 변경 후 재배포

# 해결책 2: PVC 용량 확장
kubectl patch pvc storage-vmstorage-vm-stack-0 -n monitoring \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

#### vmstorage 응답 느림

```bash
# vmstorage 메트릭 확인
curl http://localhost:8482/metrics | grep -E "vm_parts_count|vm_rows_merged"

# small parts 수가 많으면 병합 중 (일시적 현상)
curl http://localhost:8482/metrics | grep "vm_parts_count{type=\"storage/small\"}"

# 강제 병합 트리거 (주의: CPU 사용량 증가)
curl http://localhost:8482/internal/force_merge?partition_prefix=
```

---

### 5. vmalert / 알림 문제

#### 알림이 발동하지 않음

```bash
# vmalert 포트 포워드
kubectl port-forward svc/vmalert-vm-stack-victoria-metrics-k8s-stack 8880:8880 -n monitoring &

# 규칙 평가 상태 확인
curl http://localhost:8880/api/v1/rules | \
  jq '.data.groups[].rules[] | {name: .name, health: .health, lastError: .lastError}'

# 오류 있는 규칙만
curl http://localhost:8880/api/v1/rules | \
  jq '.data.groups[].rules[] | select(.health != "ok")'

# vmalert 로그 확인
kubectl logs -n monitoring -l app=vmalert | grep -i "error\|eval" | tail -20
```

#### Alertmanager로 알림이 전달되지 않음

```bash
# Alertmanager 현재 알림 확인
kubectl port-forward svc/vmalertmanager-vm-stack 9093:9093 -n monitoring &
curl http://localhost:9093/api/v2/alerts | jq

# Alertmanager 로그 확인
kubectl logs -n monitoring -l app=vmalertmanager | grep -i "error\|webhook\|slack"
```

---

### 6. VMRule / Operator 문제

#### VMRule이 적용되지 않음

```bash
# VMRule 상태 확인
kubectl get vmrule -n monitoring
kubectl describe vmrule alerting-rules -n monitoring

# Operator 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=victoria-metrics-operator --since=30m \
  | grep -i "error\|vmrule"

# vmalert가 vmrule을 로드했는지 확인
kubectl logs -n monitoring -l app=vmalert | grep -i "rule\|load"
```

---

## 로그 수집 명령어 모음

```bash
# 전체 VM 컴포넌트 최근 에러 로그 수집
for COMP in vminsert vmselect vmstorage vmagent vmalert; do
  echo "========== ${COMP} =========="
  kubectl logs -n monitoring \
    -l app=${COMP} \
    --since=1h 2>/dev/null \
    | grep -iE "error|fail|panic|warn" | tail -10
  echo ""
done
```

---

## 유용한 진단 PromQL

```promql
# vminsert 5xx 에러율
rate(vm_vminsert_api_requests_total{status=~"5.."}[5m])
  /
rate(vm_vminsert_api_requests_total[5m])

# vmstorage 디스크 여유율
vm_available_disk_space_bytes
  /
(vm_data_size_bytes{type="storage"} + vm_available_disk_space_bytes)

# vmselect P99 레이턴시
histogram_quantile(0.99,
  rate(vm_request_duration_seconds_bucket{component="vmselect"}[5m])
)

# vmagent 미전송 WAL 크기
vm_remotewrite_pending_data_bytes

# 활성 시계열 수
vm_cache_entries{type="storage/tsid"}

# vmstorage 초당 쓰기 처리량
rate(vm_rows_added_to_storage_total[5m])
```

---

## 긴급 상황 대응

### vmstorage 디스크 Full 위기

```bash
# 1. 즉시 retention 단축 (재시작 필요)
# values.yaml에서 retentionPeriod: "7d" 로 임시 단축 후:
helm upgrade vm-stack vm/victoria-metrics-k8s-stack -n monitoring \
  --values my-values.yaml --reuse-values

# 2. 강제 병합 (작은 파트 통합으로 약간의 공간 확보)
kubectl exec -n monitoring vmstorage-vm-stack-0 -- \
  wget -qO- "http://localhost:8482/internal/force_merge?partition_prefix="

# 3. PVC 긴급 확장
kubectl patch pvc storage-vmstorage-vm-stack-0 -n monitoring \
  -p '{"spec":{"resources":{"requests":{"storage":"300Gi"}}}}'
```

### vminsert가 vmstorage 연결 불가 시

```bash
# vmstorage 파드 재시작
kubectl rollout restart statefulset/vmstorage-vm-stack -n monitoring

# vminsert 재시작 (재연결 강제)
kubectl rollout restart deployment/vminsert-vm-stack-victoria-metrics-k8s-stack -n monitoring

# 복구 확인
kubectl get pods -n monitoring -w
curl http://localhost:8480/health
```

> **데이터 손실 방지**: vminsert 장애 시 vmagent의 WAL이 데이터를 보존합니다.
> vminsert 복구 후 vmagent가 WAL에서 자동으로 재전송합니다.
