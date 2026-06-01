# VictoriaMetrics Docker Compose 설치

로컬 단일 노드에서 VictoriaMetrics 스택을 빠르게 검증할 때 사용한다.

## 대상

- `victoria-metrics`
- `vmagent`
- `vmalert`

## 절차

1. `compose.yaml`에 이미지, 포트, volume을 정의한다.
2. 설정 파일을 mount 한다.
3. `docker compose up -d`로 올린다.
4. `docker compose logs -f`와 `docker compose ps`를 확인한다.

## 확인 명령

```bash
docker compose ps
curl http://localhost:8428/health
```

## 운영 포인트

- 로컬 실험은 단일 binary mode로 시작하는 편이 단순하다.
- 운영용 replication과 storage policy는 Helm 문서를 따른다.
