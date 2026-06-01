# victoriametrics-practice

EKS + VictoriaMetrics Cluster 기준으로 메트릭 저장소, vmagent, vminsert/vmstorage/vmselect, Grafana 연동, 알림, 고가용성을 정리한 개인 학습 문서입니다.

## 빠른 시작

- 처음 볼 문서: `docs/install.md`
- 설치 방식: Helm / systemd / Docker Compose
- 전체 흐름: 설치 -> 아키텍처/스토리지 -> vmagent/remote write -> Grafana -> 규칙/알림 -> 운영
- AI 작업 지침: `CLAUDE.md`

## 구조

```text
victoriametrics-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── README.md
│   ├── agents/
│   ├── rules/
│   ├── templates/
│   └── *.md
└── ops/
    ├── README.md
    └── config/     # Helm values, VM CRD, remote_write 예시
```

## 학습 경로

| 단계 | 문서 |
|------|------|
| 설치 | `docs/install.md` |
| 핵심 개념 | `docs/architecture-guide.md`, `docs/storage-guide.md` |
| 데이터 흐름 | `docs/vmagent-guide.md`, `docs/grafana-datasource-guide.md` |
| 규칙/알림 | `docs/recording-rules-guide.md`, `docs/alerting-guide.md` |
| 운영 | `docs/ha-guide.md`, `docs/monitoring-guide.md`, `docs/troubleshooting-guide.md` |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Chart | `vm/victoria-metrics-k8s-stack` |
| Namespace | `monitoring` |
| Storage | EBS gp3 |
| Region | `ap-northeast-2` |
