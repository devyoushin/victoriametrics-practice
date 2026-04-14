# VictoriaMetrics on EKS — 실습 저장소

EKS 환경에서 VictoriaMetrics Cluster를 처음부터 운영 수준까지 학습하는 실습 저장소입니다.

---

## 환경 정보

| 항목 | 값 |
|---|---|
| 플랫폼 | AWS EKS |
| VM Helm Chart | `vm/victoria-metrics-k8s-stack` (권장 버전: 0.25.x) |
| 네임스페이스 | `monitoring` |
| 스토리지 | EBS gp3 (vmstorage PVC) |
| 리전 | `ap-northeast-2` |

---

## 사전 요구사항

```bash
# 필요 도구 확인
kubectl version --client      # >= 1.25
helm version                  # >= 3.10
aws --version                 # AWS CLI v2

# EKS 클러스터 접속 확인
kubectl get nodes
```

---

## 빠른 시작 (Quick Start)

```bash
# 1. Helm 저장소 추가
helm repo add vm https://victoriametrics.github.io/helm-charts
helm repo update

# 2. 네임스페이스 생성 후 설치
kubectl create namespace monitoring
helm install vm-stack vm/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --values helm/values.yaml \
  --version 0.25.0

# 3. 헬스 체크
kubectl port-forward svc/vm-stack-victoria-metrics-k8s-stack-vminsert 8480:8480 -n monitoring &
curl http://localhost:8480/health
```

---

## 학습 경로

### 1단계: 설치
- [Helm으로 VictoriaMetrics 설치](./install.md)

### 2단계: 핵심 개념
- [아키텍처 개요](./architecture-guide.md)
- [스토리지 구성 (EBS + 백업)](./storage-guide.md)

### 3단계: 데이터 흐름
- [Write Path — vmagent & Remote Write](./vmagent-guide.md)
- [Read Path — Grafana 연동](./grafana-datasource-guide.md)

### 4단계: 규칙 & 알림
- [Recording Rules](./recording-rules-guide.md)
- [Alerting Rules & Alertmanager](./alerting-guide.md)

### 5단계: 운영 심화
- [고가용성 (Replication & Multi-AZ)](./ha-guide.md)
- [메타 모니터링 (VM으로 VM 관찰)](./monitoring-guide.md)

### 6단계: 문제 해결
- [트러블슈팅 가이드](./troubleshooting-guide.md)

---

## 저장소 구조

```
victoriametrics-practice/
├── README.md
├── install.md                    # Helm 설치 가이드
├── architecture-guide.md         # 아키텍처 설명
├── storage-guide.md              # EBS 스토리지 + 백업
├── vmagent-guide.md              # vmagent 메트릭 수집
├── grafana-datasource-guide.md   # Grafana 연동
├── recording-rules-guide.md      # Recording Rules
├── alerting-guide.md             # Alerting + Alertmanager
├── ha-guide.md                   # 고가용성
├── monitoring-guide.md           # 메타 모니터링
├── troubleshooting-guide.md      # 트러블슈팅
├── helm/
│   ├── values.yaml               # 개발/테스트용 Helm values
│   └── values-ha.yaml            # 운영 HA Helm values
├── kubernetes/
│   ├── vmcluster.yaml            # VMCluster CRD 예시
│   ├── vmagent.yaml              # VMAgent CRD 예시
│   └── vmalert.yaml              # VMAlert CRD 예시
└── prometheus/
    └── remote-write.yaml         # Prometheus remote_write 설정 예시
```

---

## 아키텍처 요약

```
Prometheus / vmagent ──remote_write──▶ vminsert ──▶ vmstorage (EBS)
                                           │               │
                                      consistent        replication
                                        hashing         factor=2
                                                         │
Grafana ◀──── vmselect ◀──────────────────────────────────┘
(PromQL)   (fan-out 쿼리 + 중복 제거)

              vmalert ──▶ Recording/Alerting Rules ──▶ Alertmanager
                  │
              vminsert (recording 결과 재기록)
```

| 컴포넌트 | 역할 |
|---|---|
| **vminsert** | remote_write 수신, 일관된 해싱으로 vmstorage에 분산 |
| **vmstorage** | 디스크(EBS)에 시계열 데이터 저장, 자체 압축 포맷 |
| **vmselect** | PromQL 쿼리 처리, 여러 vmstorage에서 fan-out 후 병합 |
| **vmagent** | Prometheus 호환 scraping + remote_write 에이전트 |
| **vmalert** | Recording/Alerting Rules 평가, Alertmanager 연동 |
| **vmauth** | 인증 프록시, 멀티 테넌트 라우팅 |

---

## Mimir와 비교

| 항목 | VictoriaMetrics | Grafana Mimir |
|---|---|---|
| 스토리지 | 로컬 디스크 (EBS) / S3 백업 | S3 (필수) |
| 아키텍처 | 단순 3-tier (insert/storage/select) | 다수 컴포넌트 (8+) |
| 멀티 테넌시 | vmauth로 구현 | 기본 내장 |
| 운영 복잡도 | 낮음 | 높음 |
| 압축 효율 | 매우 높음 (자체 포맷) | 높음 (Thanos 기반) |
| Helm Chart | victoria-metrics-k8s-stack | mimir-distributed |

---

## 참고 링크

- [VictoriaMetrics 공식 문서](https://docs.victoriametrics.com/)
- [VictoriaMetrics Helm Charts](https://github.com/VictoriaMetrics/helm-charts)
- [VictoriaMetrics Operator](https://docs.victoriametrics.com/operator/)
- [victoria-metrics-k8s-stack Chart](https://github.com/VictoriaMetrics/helm-charts/tree/master/charts/victoria-metrics-k8s-stack)
