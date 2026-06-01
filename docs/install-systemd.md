# VictoriaMetrics systemd 설치

단일 VM에서 `victoria-metrics`, `vmagent`, `vmalert`를 서비스로 관리할 때 사용한다.

## 대상

- `victoria-metrics.service`
- `vmagent.service`
- `vmalert.service`

## 준비물

- 바이너리
- 설정 파일: `/etc/victoriametrics/`
- 데이터 디렉터리: `/var/lib/victoria-metrics`

## 절차

1. 바이너리를 설치한다.
2. 설정 파일과 rule 파일을 배치한다.
3. unit 파일을 `/etc/systemd/system/`에 둔다.
4. `systemctl enable --now`로 시작한다.
5. `journalctl -u <service> -f`로 로그를 본다.

## 확인 명령

```bash
systemctl status victoria-metrics
systemctl status vmagent
systemctl status vmalert
curl http://localhost:8428/health
```

## 운영 포인트

- vmagent는 remote_write endpoint를 명확히 둔다.
- vmalert는 rule path와 alertmanager 주소를 함께 관리한다.
- 데이터 디렉터리는 삭제하지 않는다.
