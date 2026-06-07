# VictoriaMetrics Docs

VictoriaMetrics를 처음 보는 사람이 설치부터 운영과 알림까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install/install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `install/upgrade/` | VictoriaMetrics 업그레이드 |
| 3 | `architecture-guide.md` | Cluster 컴포넌트와 데이터 흐름 |
| 4 | `storage-guide.md` | EBS 스토리지와 백업 |
| 5 | `vmagent-guide.md` | scraping과 remote_write 에이전트 |
| 6 | `grafana-datasource-guide.md` | Grafana 데이터소스 연동 |
| 7 | `recording-rules-guide.md`, `alerting-guide.md` | 규칙과 알림 |
| 8 | `ha-guide.md`, `monitoring-guide.md` | 고가용성과 메타 모니터링 |
| 9 | `troubleshooting-guide.md` | 문제 해결 |
| 10 | `rules/README.md` | 문서와 운영 규칙 |
| 11 | `agents/README.md` | AI 작업 지침 |
| 12 | `templates/README.md` | 문서 템플릿 |
| 13 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/기초 | `install/install.md`, `install/upgrade/`, `architecture-guide.md`, `storage-guide.md` |
| 데이터 흐름 | `vmagent-guide.md`, `grafana-datasource-guide.md` |
| 규칙/알림 | `recording-rules-guide.md`, `alerting-guide.md` |
| 운영 | `ha-guide.md`, `monitoring-guide.md`, `troubleshooting-guide.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install/install.md`
2. `install/upgrade/`
3. `architecture-guide.md`
4. `storage-guide.md`
5. `vmagent-guide.md`
6. `grafana-datasource-guide.md`
7. `recording-rules-guide.md`
8. `alerting-guide.md`
9. `ha-guide.md`
10. `monitoring-guide.md`
11. `rules/README.md`
12. `agents/README.md`
13. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Helm values, VM CRD, remote_write 예시
