# 고가용성 (HA) 구성

VictoriaMetrics Cluster는 컴포넌트별로 독립적인 고가용성을 구성합니다.

---

## HA 아키텍처 개요

```
AZ-a                          AZ-b
────────────────────          ────────────────────
vminsert-0                    vminsert-1
    │                             │
    ├──▶ vmstorage-0 (EBS)        ├──▶ vmstorage-0
    └──▶ vmstorage-1 (EBS)        └──▶ vmstorage-1

vmselect-0                    vmselect-1
    ├──▶ vmstorage-0              ├──▶ vmstorage-0
    └──▶ vmstorage-1              └──▶ vmstorage-1

vmagent-0 (primary)           vmagent-1 (standby)
    └──▶ vminsert LB (round-robin)
```

---

## 1. vmstorage HA 설정

### Replication Factor

```yaml
# ../ops/config/helm/values-ha.yaml
victoria-metrics-k8s-stack:
  vmcluster:
    spec:
      vmstorage:
        replicaCount: 2          # 최소 2개 (AZ당 1개 권장)

        # Multi-AZ 배포
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    app: vmstorage
                topologyKey: topology.kubernetes.io/zone

      vminsert:
        replicaCount: 2
        extraArgs:
          replicationFactor: "2"   # 쓰기 시 2개 vmstorage에 복제

      vmselect:
        replicaCount: 2
        extraArgs:
          dedup.minScrapeInterval: "30s"   # 중복 제거 간격 (scrapeInterval과 동일)
```

### replicationFactor 설명

```
replicationFactor=2:
  - vminsert가 각 샘플을 2개의 vmstorage에 기록
  - vmstorage 1개 장애 시에도 데이터 접근 가능
  - 디스크 사용량 2배

  vminsert ──▶ vmstorage-0  ← 같은 데이터
           └──▶ vmstorage-1  ← 같은 데이터

  vmselect: 두 vmstorage 모두에 쿼리 → 중복 제거
```

> **최소 vmstorage 수**: `replicationFactor` 이상이어야 합니다.
> `replicationFactor=2` + `vmstorage replicaCount=2` 가 최소 HA 구성

---

## 2. vminsert HA 설정

vminsert는 Stateless이므로 Deployment + HPA로 수평 확장합니다:

```yaml
vminsert:
  replicaCount: 2

  # 자동 확장
  hpa:
    maxReplicas: 5
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70

  # Anti-affinity (AZ 분산)
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: vminsert
            topologyKey: topology.kubernetes.io/zone

  # PodDisruptionBudget (노드 드레인 시 최소 1개 보장)
  podDisruptionBudget:
    maxUnavailable: 1
```

---

## 3. vmselect HA 설정

```yaml
vmselect:
  replicaCount: 2

  # 중복 제거 설정 (replicationFactor > 1일 때 필수)
  extraArgs:
    dedup.minScrapeInterval: "30s"   # vmagent/Prometheus scrapeInterval과 동일하게

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: vmselect
          topologyKey: topology.kubernetes.io/zone

  podDisruptionBudget:
    maxUnavailable: 1

  # vmselect 캐시 (독립적으로 관리)
  cacheMountPath: /cache
```

---

## 4. vmagent HA 설정

vmagent를 이중화하면 동일 타겟을 2번 스크래핑하게 되어 중복이 발생합니다.
VictoriaMetrics의 중복 제거 기능으로 처리합니다:

```yaml
vmagent:
  replicaCount: 2    # 동일 타겟을 2개 vmagent가 스크래핑

  # 샤딩 설정 (타겟을 분산)
  # shard 0: 짝수 타겟, shard 1: 홀수 타겟
  # → 각 vmagent가 전체 타겟의 절반만 담당
  # (선택사항: 타겟이 매우 많을 때 사용)
  spec:
    shardCount: 2
```

**vmagent 이중화 시 vmselect 중복 제거**:

```yaml
vmselect:
  extraArgs:
    # scrapeInterval과 동일하게 설정 (중복 샘플 제거 기준)
    dedup.minScrapeInterval: "30s"
```

---

## 5. Alertmanager HA 설정

```yaml
alertmanager:
  replicaCount: 3          # 홀수 개 (Raft 합의)

  # Alertmanager 클러스터링
  config:
    global:
      resolve_timeout: 5m

  # StatefulSet으로 배포 (클러스터 상태 보존)
  podDisruptionBudget:
    maxUnavailable: 1
```

---

## 6. 전체 HA values 파일

```yaml
# ../ops/config/helm/values-ha.yaml
victoria-metrics-k8s-stack:
  vmcluster:
    spec:
      vmstorage:
        replicaCount: 2
        retentionPeriod: "90d"
        storage:
          volumeClaimTemplate:
            spec:
              storageClassName: gp3
              resources:
                requests:
                  storage: 200Gi
        resources:
          requests:
            cpu: "2"
            memory: 4Gi
          limits:
            cpu: "8"
            memory: 16Gi
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    app: vmstorage
                topologyKey: topology.kubernetes.io/zone
        podDisruptionBudget:
          maxUnavailable: 1

      vminsert:
        replicaCount: 2
        extraArgs:
          replicationFactor: "2"
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: "2"
            memory: 2Gi
        affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchLabels:
                      app: vminsert
                  topologyKey: topology.kubernetes.io/zone
        podDisruptionBudget:
          maxUnavailable: 1

      vmselect:
        replicaCount: 2
        extraArgs:
          dedup.minScrapeInterval: "30s"
        cacheMountPath: /cache
        storage:
          volumeClaimTemplate:
            spec:
              storageClassName: gp3
              resources:
                requests:
                  storage: 10Gi
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: "2"
            memory: 2Gi
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    app: vmselect
                topologyKey: topology.kubernetes.io/zone
        podDisruptionBudget:
          maxUnavailable: 1

  vmagent:
    spec:
      replicaCount: 2
      scrapeInterval: 30s

  alertmanager:
    replicaCount: 3
```

---

## 7. HA 검증

```bash
# vmstorage 노드 분산 확인
kubectl get pods -n monitoring -l app=vmstorage -o wide | awk '{print $1, $7}'

# vmstorage 1개 다운 시 읽기/쓰기 정상 여부 테스트
kubectl delete pod vmstorage-vm-stack-0 -n monitoring

# 쓰기 테스트
curl -X POST http://localhost:8480/insert/0/prometheus/api/v1/import/prometheus \
  -d 'ha_test_metric{test="ha"} 1'

# 읽기 테스트
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=ha_test_metric"

# 삭제된 파드 자동 복구 확인
kubectl get pods -n monitoring -l app=vmstorage -w
```

---

## 8. 장애 시나리오별 동작

| 장애 상황 | 동작 | 데이터 유실 |
|---|---|---|
| vminsert 1개 다운 | 나머지 vminsert로 계속 처리 | 없음 |
| vmstorage 1개 다운 (replication=2) | vminsert: 나머지 1개에만 기록 | 없음 |
| vmstorage 2개 모두 다운 | 쓰기 실패, vmagent WAL 버퍼링 | 복구 후 WAL 재전송 |
| vmselect 1개 다운 | 나머지 vmselect로 계속 처리 | 없음 |
| vmagent 1개 다운 | 해당 샤드 타겟 일시 중단 | scrape 누락 |
| Alertmanager 1개 다운 | Raft로 나머지 2개가 계속 처리 | 없음 |

---

## 트러블슈팅

```bash
# replication 상태 확인 (vminsert 로그)
kubectl logs -n monitoring -l app=vminsert | grep -i "replication\|storage"

# vmstorage 헬스 확인
for i in 0 1; do
  echo "=== vmstorage-${i} ==="
  kubectl exec -n monitoring vmstorage-vm-stack-${i} -- \
    wget -qO- http://localhost:8482/health
done

# 중복 제거 동작 확인 (vmselect 로그)
kubectl logs -n monitoring -l app=vmselect | grep -i "dedup"
```
