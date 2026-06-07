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

## RPM 설치

VictoriaMetrics 계열 바이너리는 내부 RPM으로 묶어 배포하는 경우가 많다. RPM 산출물이 있으면 각 컴포넌트를 고정 버전으로 설치한다.

```bash
sudo dnf install -y ./victoria-metrics-<VERSION>-1.x86_64.rpm
sudo dnf install -y ./vmagent-<VERSION>-1.x86_64.rpm
sudo dnf install -y ./vmalert-<VERSION>-1.x86_64.rpm
rpm -qa | grep -E 'victoria|vmagent|vmalert'
```

패키지가 unit 파일을 포함하는지 확인한다.

```bash
rpm -ql victoria-metrics | grep systemd
systemctl cat victoria-metrics
```

## tarball 설치

공식 릴리스 tarball을 내려받아 필요한 바이너리를 `/usr/local/bin`에 배치한다.

```bash
VM_VERSION="<VERSION>"
curl -LO "https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v${VM_VERSION}/victoria-metrics-linux-amd64-v${VM_VERSION}.tar.gz"
tar -xzf "victoria-metrics-linux-amd64-v${VM_VERSION}.tar.gz"
sudo install -m 0755 victoria-metrics-prod /usr/local/bin/victoria-metrics
sudo install -m 0755 vmagent-prod /usr/local/bin/vmagent
sudo install -m 0755 vmalert-prod /usr/local/bin/vmalert
sudo mkdir -p /etc/victoriametrics /var/lib/victoria-metrics
```

## 절차

1. RPM 또는 tarball 방식 중 하나로 바이너리를 설치한다.
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
