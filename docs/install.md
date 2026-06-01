# VictoriaMetrics 설치 (Helm)

EKS 환경에서 `victoria-metrics-k8s-stack` Helm Chart로 VictoriaMetrics Cluster를 설치합니다.

---

## 사전 조건 확인

```bash
# kubectl 연결 확인
kubectl get nodes

# Helm 버전 확인 (>= 3.10 권장)
helm version

# AWS CLI 설정 확인
aws sts get-caller-identity

# EBS CSI Driver 설치 여부 확인 (PVC용)
kubectl get daemonset ebs-csi-node -n kube-system
```

---

## 1. 네임스페이스 생성

```bash
kubectl create namespace monitoring
```

---

## 2. Helm 저장소 추가

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts
helm repo update

# 사용 가능한 버전 확인
helm search repo vm/victoria-metrics-k8s-stack --versions | head -10
```

---

## 3. Helm values 파일 준비

```bash
# 기본 values 확인
helm show values vm/victoria-metrics-k8s-stack > default-values.yaml

cp ../ops/config/helm/values.yaml my-values.yaml
```

`../ops/config/helm/values.yaml` 핵심 설정 구조:

```yaml
# VictoriaMetrics Cluster 활성화
victoria-metrics-k8s-stack:
  vmcluster:
    enabled: true
    spec:
      # vmstorage: 데이터 저장 노드
      vmstorage:
        replicaCount: 2
        storage:
          volumeClaimTemplate:
            spec:
              storageClassName: gp3
              resources:
                requests:
                  storage: 100Gi
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi

      # vminsert: 쓰기 경로
      vminsert:
        replicaCount: 2
        extraArgs:
          replicationFactor: "2"   # vmstorage 2개에 복제

      # vmselect: 읽기 경로
      vmselect:
        replicaCount: 2
        cacheMountPath: /select-cache
        storage:
          volumeClaimTemplate:
            spec:
              storageClassName: gp3
              resources:
                requests:
                  storage: 10Gi

  # vmagent: 메트릭 수집
  vmagent:
    enabled: true
    spec:
      selectAllByDefault: true    # 모든 ServiceMonitor 자동 수집

  # vmalert: 알림 규칙 평가
  vmalert:
    enabled: true

  # Grafana 포함 설치
  grafana:
    enabled: true
    adminPassword: admin          # 운영 환경에서는 Secret으로 관리

  # Alertmanager
  alertmanager:
    enabled: true
```

---

## 4. VictoriaMetrics Cluster 설치

```bash
VM_CHART_VERSION="0.25.0"

helm install vm-stack vm/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --values my-values.yaml \
  --version ${VM_CHART_VERSION} \
  --timeout 10m \
  --wait
```

> **팁**: `--wait` 플래그를 사용하면 모든 파드가 Ready 상태가 될 때까지 대기합니다.

---

## 5. 설치 확인

```bash
# 파드 상태 확인
kubectl get pods -n monitoring -w

# CRD 설치 확인
kubectl get crds | grep victoriametrics
```

정상 설치 시 아래 컴포넌트 파드가 모두 `Running` 상태여야 합니다:

| 컴포넌트 | 기본 파드 수 | 유형 |
|---|---|---|
| vmstorage | 2 | StatefulSet (PVC 보유) |
| vminsert | 2 | Deployment |
| vmselect | 2 | Deployment |
| vmagent | 1 | Deployment |
| vmalert | 1 | Deployment |
| alertmanager | 1 | StatefulSet |
| grafana | 1 | Deployment |
| operator | 1 | Deployment |

```bash
# 빠른 상태 요약
kubectl get pods -n monitoring \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount'
```

---

## 6. 헬스 체크 (포트 포워드)

```bash
# vminsert 헬스 체크
kubectl port-forward svc/vminsert-vm-stack-victoria-metrics-k8s-stack 8480:8480 -n monitoring &
curl http://localhost:8480/health

# vmselect 헬스 체크
kubectl port-forward svc/vmselect-vm-stack-victoria-metrics-k8s-stack 8481:8481 -n monitoring &
curl http://localhost:8481/health

# vmstorage 헬스 체크
kubectl port-forward svc/vmstorage-vm-stack-victoria-metrics-k8s-stack 8482:8482 -n monitoring &
curl http://localhost:8482/health
```

모두 `OK` 응답이면 정상입니다.

---

## 7. 데이터 쓰기 테스트

```bash
# vminsert에 테스트 메트릭 전송
curl -X POST http://localhost:8480/insert/0/prometheus/api/v1/import/prometheus \
  -d 'test_metric{env="dev"} 42'

# vmselect로 조회 확인
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=test_metric"
```

---

## 업그레이드

```bash
# 업그레이드 전 현재 values 확인
helm get values vm-stack -n monitoring

# 업그레이드
helm upgrade vm-stack vm/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --values my-values.yaml \
  --version 0.26.0 \
  --timeout 10m

# 히스토리 확인
helm history vm-stack -n monitoring
```

> **vmstorage 업그레이드 주의**: vmstorage는 StatefulSet이므로 롤링 업그레이드 중
> 한 번에 1개씩 재시작됩니다. replication_factor가 2이면 1개 다운 시에도 데이터 접근이 가능합니다.

---

## 롤백

```bash
helm rollback vm-stack 1 -n monitoring
```

---

## 삭제

```bash
helm uninstall vm-stack -n monitoring

# PVC 삭제 (vmstorage 데이터 포함)
kubectl delete pvc -n monitoring -l app.kubernetes.io/instance=vm-stack

# 네임스페이스 삭제
kubectl delete namespace monitoring

# CRD 삭제 (주의: 다른 VM 인스턴스가 없을 때만)
kubectl get crds | grep victoriametrics | awk '{print $1}' | xargs kubectl delete crd
```

> **주의**: PVC를 삭제하면 vmstorage의 모든 데이터가 영구 삭제됩니다.

---

## 설치 시 자주 발생하는 문제

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| vmstorage `Pending` | EBS CSI Driver 미설치 또는 PVC 오류 | `kubectl describe pvc` 로 이벤트 확인 |
| `Error: INSTALLATION FAILED: timed out` | 리소스 부족 | 노드 용량 확인, `--timeout` 연장 |
| vminsert `CrashLoopBackOff` | vmstorage 미준비 | vmstorage 파드 먼저 Running 확인 |
| Operator `Permission denied` | RBAC 설정 오류 | `kubectl logs -n monitoring -l app=vm-operator` 확인 |
| StorageClass `gp3` 없음 | EKS 기본 StorageClass가 `gp2` | `gp2` 로 변경하거나 gp3 StorageClass 생성 |


---

## systemd 설치

Kubernetes 없이 단일 VM이나 베어메탈에서 돌릴 때는 바이너리를 내려받아 systemd로 관리합니다.

1. 바이너리 또는 패키지를 설치합니다.
2. 설정 파일을 /etc/<component>/ 아래에 둡니다.
3. `systemctl enable --now <service>`로 등록합니다.
4. `journalctl -u <service> -f`로 로그를 확인합니다.

## Docker Compose 설치

로컬 개발이나 빠른 검증은 Docker Compose가 가장 간단합니다.

1. `compose.yaml`을 만들고 이미지, 포트, 볼륨을 정의합니다.
2. `docker compose up -d`로 올립니다.
3. `docker compose logs -f`와 `docker compose ps`로 상태를 확인합니다.
4. 개발용은 같은 설정을 유지하되, 운영용은 Helm 또는 systemd를 사용합니다.
