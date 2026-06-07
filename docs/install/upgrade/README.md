# VictoriaMetrics 업그레이드 가이드

VictoriaMetrics 업그레이드는 VictoriaMetrics Operator, vmcluster, vmagent, vmalert, Grafana datasource에 영향을 줄 수 있습니다. 저장소, retention, replication 설정을 먼저 확인합니다.

## 1. 사전 점검

```bash
export NAMESPACE="monitoring"
export RELEASE="vm-stack"
export CHART_VERSION="0.26.0"
export VALUES_FILE="my-values.yaml"

helm status ${RELEASE} -n ${NAMESPACE}
helm history ${RELEASE} -n ${NAMESPACE}
helm get values ${RELEASE} -n ${NAMESPACE} > values-before-upgrade.yaml
kubectl get pods,pvc -n ${NAMESPACE}
kubectl get vmcluster,vmagent,vmalert -A
```

## 2. Helm 업그레이드

```bash
helm repo update vm
helm upgrade ${RELEASE} vm/victoria-metrics-k8s-stack \
  --namespace ${NAMESPACE} \
  --values ${VALUES_FILE} \
  --version ${CHART_VERSION} \
  --timeout 15m \
  --wait
```

## 3. 확인

```bash
kubectl get pods -n ${NAMESPACE}
kubectl get vmcluster,vmagent,vmalert -A
kubectl port-forward svc/vminsert-${RELEASE}-victoria-metrics-k8s-stack 8480:8480 -n ${NAMESPACE}
curl http://localhost:8480/health
```

vmagent 수집, remote_write, Grafana query, vmalert rule 평가를 확인합니다.

## 4. 롤백

```bash
helm history ${RELEASE} -n ${NAMESPACE}
helm rollback ${RELEASE} <REVISION> -n ${NAMESPACE} --wait
```

Operator CRD 변경 후에는 이전 controller가 새 CR spec을 처리하지 못할 수 있습니다. major 업그레이드는 CR YAML을 백업하고 staging에서 먼저 검증합니다.

## 5. systemd / Docker Compose

systemd 설치는 `victoria-metrics`, `vmagent`, `vmalert` 바이너리와 설정 파일을 백업한 뒤 서비스별로 교체합니다. Docker Compose 설치는 image tag를 변경하고 `docker compose pull && docker compose up -d`를 실행합니다. 데이터 디렉터리와 retention 설정을 먼저 확인합니다.

