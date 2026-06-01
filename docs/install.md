# VictoriaMetrics 설치 안내

설치 방식별 문서를 분리해 둔 입구다.

## 설치 방식

| 방식 | 문서 | 설명 |
|------|------|------|
| Helm | `install-helm.md` | VictoriaMetrics Cluster / k8s-stack 설치 |
| systemd | `install-systemd.md` | RPM 또는 tarball로 설치 후 `vmagent`, `vmalert`, `victoria-metrics` 관리 |
| Docker Compose | `install-docker-compose.md` | 로컬 단일 노드 검증용 |

## 읽는 순서

1. `install-helm.md`
2. `install-systemd.md`
3. `install-docker-compose.md`
