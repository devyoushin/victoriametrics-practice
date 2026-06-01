# VictoriaMetrics 설치 (Helm)

EKS 환경에서 `victoria-metrics-k8s-stack`로 VictoriaMetrics Cluster를 설치한다.

## 사전 준비

```bash
kubectl get nodes
helm version
aws sts get-caller-identity
kubectl get daemonset ebs-csi-node -n kube-system
kubectl create namespace monitoring
helm repo add vm https://victoriametrics.github.io/helm-charts
helm repo update
```

## 설치

```bash
VM_CHART_VERSION="0.25.0"
helm install vm-stack vm/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --values my-values.yaml \
  --version ${VM_CHART_VERSION} \
  --timeout 10m \
  --wait
```

## 확인

```bash
kubectl get pods -n monitoring -w
kubectl get crds | grep victoriametrics
kubectl port-forward svc/vminsert-vm-stack-victoria-metrics-k8s-stack 8480:8480 -n monitoring
curl http://localhost:8480/health
```

## 업그레이드 / 삭제

```bash
helm upgrade vm-stack vm/victoria-metrics-k8s-stack -n monitoring --values my-values.yaml --version 0.26.0 --timeout 10m
helm history vm-stack -n monitoring
helm uninstall vm-stack -n monitoring
```
