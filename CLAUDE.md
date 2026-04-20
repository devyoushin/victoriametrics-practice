# victoriametrics-practice — 프로젝트 가이드

## 프로젝트 설정
- 환경: EKS / 로컬 Docker
- VictoriaMetrics 버전: 1.x
- 네임스페이스: monitoring
- 배포 모드: 단일 노드 (학습) / 클러스터 (운영)
- 수집기: vmagent
- 알림: vmalert + Alertmanager

---

## 디렉토리 구조

```
victoriametrics-practice/
├── CLAUDE.md                  # 이 파일 (자동 로드)
├── .claude/
│   ├── settings.json
│   └── commands/              # /new-doc, /new-runbook, /review-doc, /add-troubleshooting, /search-kb
├── agents/                    # doc-writer, storage-designer, alerting-advisor, troubleshooter
├── templates/                 # service-doc, runbook, incident-report
├── rules/                     # doc-writing, victoriametrics-conventions, security-checklist, monitoring
├── helm/                      # Helm values 파일
├── kubernetes/                # K8s 매니페스트
├── prometheus/                # Prometheus 비교 설정
└── *-guide.md                 # 주제별 가이드 문서
```

---

## 커스텀 슬래시 명령어

| 명령어 | 설명 | 사용 예시 |
|--------|------|---------|
| `/new-doc` | 새 가이드 문서 생성 | `/new-doc vmbackup-restore` |
| `/new-runbook` | 새 런북 생성 | `/new-runbook vmstorage 디스크 포화 대응` |
| `/review-doc` | 문서 검토 | `/review-doc vmagent-guide.md` |
| `/add-troubleshooting` | 트러블슈팅 케이스 추가 | `/add-troubleshooting 쿼리 타임아웃` |
| `/search-kb` | 지식베이스 검색 | `/search-kb MetricsQL vs PromQL` |

---

## 가이드 문서 목록

| 문서 | 주제 |
|------|------|
| `install.md` | VictoriaMetrics 설치 |
| `architecture-guide.md` | 아키텍처 (단일/클러스터) |
| `vmagent-guide.md` | vmagent 스크레이핑 설정 |
| `recording-rules-guide.md` | Recording Rule 최적화 |
| `alerting-guide.md` | vmalert 알림 설정 |
| `storage-guide.md` | 스토리지 구성 및 보존 정책 |
| `ha-guide.md` | 고가용성 클러스터 구성 |
| `grafana-datasource-guide.md` | Grafana 데이터소스 설정 |
| `monitoring-guide.md` | VM 자체 모니터링 |
| `troubleshooting-guide.md` | 트러블슈팅 |

---

## 핵심 명령어

```bash
# VictoriaMetrics 상태 확인
kubectl get pods -n monitoring -l app=victoriametrics

# vmui 접속 (포트포워딩)
kubectl port-forward -n monitoring svc/victoriametrics 8428:8428

# 카디널리티 분석
curl http://localhost:8428/api/v1/status/tsdb

# vmagent 타겟 상태
curl http://vmagent:8429/targets

# MetricsQL 쿼리 테스트
curl 'http://localhost:8428/api/v1/query?query=up'
```
