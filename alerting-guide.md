# Alerting Rules & Alertmanager

vmalert가 Alerting Rules를 평가하고 Alertmanager로 알림을 전송합니다.
Prometheus AlertManager와 완전히 호환됩니다.

---

## 전체 흐름

```
vmalert (규칙 평가, 기본 1분 주기)
    │ 조건 만족 + for 시간 경과
    ▼
Alertmanager
    │ 중복 제거 → 그루핑 → 라우팅
    ▼
Slack / PagerDuty / Email / Webhook
```

### Alert 상태 주기

```
INACTIVE ──(조건 만족)──▶ PENDING ──(for 시간 경과)──▶ FIRING
    ▲                                                      │
    └──────────────────(조건 해소)─────────────────────────┘
```

- **INACTIVE**: 조건 불만족
- **PENDING**: 조건 만족하였으나 `for` 시간 미충족 (일시적 스파이크 무시)
- **FIRING**: `for` 시간 초과 → Alertmanager에 전송

---

## 1. Alerting Rules 작성 (VMRule CRD)

```yaml
# kubernetes/vmalert.yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: alerting-rules
  namespace: monitoring
spec:
  groups:
    # ─────────────────────────────────────────────
    # 서비스 가용성
    # ─────────────────────────────────────────────
    - name: service.availability
      rules:
        - alert: ServiceDown
          expr: up == 0
          for: 2m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "서비스 {{ $labels.job }} 응답 없음"
            description: |
              {{ $labels.job }}/{{ $labels.instance }} 가 2분 이상 응답하지 않습니다.
            runbook: "https://wiki.company.com/runbooks/service-down"

        - alert: HighErrorRate
          expr: |
            (
              sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
              /
              sum by (job) (rate(http_requests_total[5m]))
            ) > 0.05
          for: 5m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "{{ $labels.job }} 에러율 {{ $value | humanizePercentage }} 초과"
            description: |
              {{ $labels.job }}의 5분 에러율이 {{ $value | humanizePercentage }} 입니다.
              (임계값: 5%)

        - alert: HighLatency
          expr: |
            histogram_quantile(0.99,
              sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
            ) > 1.0
          for: 5m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "{{ $labels.job }} P99 레이턴시 {{ $value | humanizeDuration }} 초과"
            description: "P99 레이턴시가 1초를 초과했습니다."

    # ─────────────────────────────────────────────
    # Kubernetes 리소스
    # ─────────────────────────────────────────────
    - name: kubernetes.resources
      rules:
        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "파드 {{ $labels.namespace }}/{{ $labels.pod }} 재시작 반복"
            description: "15분 내 {{ $value | printf \"%.0f\" }}번 재시작되었습니다."

        - alert: HighMemoryUsage
          expr: |
            (
              container_memory_working_set_bytes{container!=""}
              /
              container_spec_memory_limit_bytes{container!=""}
            ) > 0.9
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "{{ $labels.namespace }}/{{ $labels.pod }} 메모리 사용량 90% 초과"
            description: "현재 메모리 사용률: {{ $value | humanizePercentage }}"

        - alert: PersistentVolumeAlmostFull
          expr: |
            (
              kubelet_volume_stats_used_bytes
              /
              kubelet_volume_stats_capacity_bytes
            ) > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} 용량 85% 초과"
            description: "사용률: {{ $value | humanizePercentage }}"

    # ─────────────────────────────────────────────
    # VictoriaMetrics 자체 모니터링
    # ─────────────────────────────────────────────
    - name: victoriametrics.health
      rules:
        - alert: VMStorageDiskSpaceLow
          expr: |
            vm_available_disk_space_bytes / vm_data_size_bytes < 0.2
          for: 5m
          labels:
            severity: warning
            component: vmstorage
          annotations:
            summary: "vmstorage 디스크 여유 공간 20% 미만"
            description: |
              {{ $labels.instance }}: 디스크 여유 공간이 부족합니다.
              현재 사용률: {{ $value | humanizePercentage }}

        - alert: VMInsertDroppedSamples
          expr: |
            rate(vm_vminsert_api_requests_total{status=~"5.."}[5m]) > 0
          for: 5m
          labels:
            severity: critical
            component: vminsert
          annotations:
            summary: "vminsert 샘플 유실 발생"
            description: |
              vminsert에서 5xx 오류 발생으로 샘플이 유실되고 있습니다.
              원인: vmstorage 장애 또는 디스크 full

        - alert: VMSelectSlowQueries
          expr: |
            rate(vm_request_duration_seconds_bucket{le="10"}[5m]) /
            rate(vm_request_duration_seconds_count[5m]) < 0.95
          for: 10m
          labels:
            severity: warning
            component: vmselect
          annotations:
            summary: "vmselect 쿼리 95%가 10초 이내 완료되지 않음"
            description: "Recording Rule 추가 또는 vmselect 리소스 증가를 검토하세요."

        - alert: VMAgentRemoteWriteBehind
          expr: |
            vm_remotewrite_pending_data_bytes > 1073741824   # 1GB
          for: 15m
          labels:
            severity: warning
            component: vmagent
          annotations:
            summary: "vmagent WAL 미전송 데이터 1GB 초과"
            description: |
              vmagent가 vminsert로 데이터를 전송하지 못하고 있습니다.
              vminsert 또는 네트워크 상태를 확인하세요.
```

---

## 2. Alertmanager 설정

```yaml
# helm/values.yaml
alertmanager:
  enabled: true
  config:
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/XXXXX/YYYYY/ZZZZZ'

    route:
      receiver: slack-default
      group_by: ['alertname', 'namespace', 'job']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h

      routes:
        # Critical → PagerDuty + Slack 동시 알림
        - match:
            severity: critical
          receiver: pagerduty-critical
          continue: true

        - match:
            severity: critical
          receiver: slack-critical
          repeat_interval: 1h

        # Warning → Slack
        - match:
            severity: warning
          receiver: slack-warning

        # VictoriaMetrics 내부 알림 → 인프라팀 채널
        - match:
            component: vmstorage
          receiver: slack-infra
        - match:
            component: vminsert
          receiver: slack-infra
        - match:
            component: vmselect
          receiver: slack-infra
        - match:
            component: vmagent
          receiver: slack-infra

    receivers:
      - name: slack-default
        slack_configs:
          - channel: '#alerts'
            title: '[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}'
            text: |
              {{ range .Alerts }}
              *요약:* {{ .Annotations.summary }}
              *상세:* {{ .Annotations.description }}
              {{ if .Annotations.runbook }}*런북:* {{ .Annotations.runbook }}{{ end }}
              {{ end }}
            send_resolved: true

      - name: slack-critical
        slack_configs:
          - channel: '#alerts-critical'
            color: 'danger'
            title: '[CRITICAL] {{ .GroupLabels.alertname }}'
            text: |
              {{ range .Alerts.Firing }}
              *요약:* {{ .Annotations.summary }}
              *상세:* {{ .Annotations.description }}
              *시작:* {{ .StartsAt | since | humanizeDuration }} 전
              {{ if .Annotations.runbook }}*런북:* <{{ .Annotations.runbook }}|바로가기>{{ end }}
              {{ end }}
            send_resolved: true

      - name: slack-warning
        slack_configs:
          - channel: '#alerts-warning'
            color: 'warning'
            title: '[WARNING] {{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

      - name: slack-infra
        slack_configs:
          - channel: '#infra-alerts'
            text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

      - name: pagerduty-critical
        pagerduty_configs:
          - routing_key: '<PD_INTEGRATION_KEY>'
            description: '{{ .GroupLabels.alertname }}: {{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
            severity: critical

    # Inhibition: Critical 발생 시 동일 job의 Warning 억제
    inhibit_rules:
      - source_match:
          severity: critical
        target_match:
          severity: warning
        equal: ['alertname', 'namespace', 'job']
```

---

## 3. 알림 상태 확인

```bash
# vmalert 포트 포워드
kubectl port-forward svc/vmalert-vm-stack-victoria-metrics-k8s-stack 8880:8880 -n monitoring &

# 현재 발생 중인 알림 목록
curl http://localhost:8880/api/v1/alerts | \
  jq '.data.alerts[] | {name: .labels.alertname, state: .state, severity: .labels.severity}'

# FIRING 알림만
curl http://localhost:8880/api/v1/alerts | \
  jq '.data.alerts[] | select(.state == "firing")'

# 규칙 평가 상태 (오류 있는 규칙)
curl http://localhost:8880/api/v1/rules | \
  jq '.data.groups[].rules[] | select(.health != "ok") | {name: .name, lastError: .lastError}'

# Alertmanager UI 접근
kubectl port-forward svc/vmalertmanager-vm-stack 9093:9093 -n monitoring &
# 브라우저: http://localhost:9093
```

---

## 4. Silence (음소거)

유지보수 시 알림을 일시적으로 음소거합니다:

```bash
# 특정 알림 음소거 (2시간)
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "alertname", "value": "PodCrashLooping", "isRegex": false}
    ],
    "startsAt": "2026-04-14T00:00:00Z",
    "endsAt": "2026-04-14T02:00:00Z",
    "comment": "배포 작업으로 인한 임시 음소거",
    "createdBy": "operator"
  }'

# 네임스페이스 전체 음소거
curl -X POST http://localhost:9093/api/v2/silences \
  -H "Content-Type: application/json" \
  -d '{
    "matchers": [
      {"name": "namespace", "value": "staging", "isRegex": false}
    ],
    "startsAt": "2026-04-14T00:00:00Z",
    "endsAt": "2026-04-14T04:00:00Z",
    "comment": "스테이징 유지보수",
    "createdBy": "operator"
  }'

# 음소거 목록 확인
curl http://localhost:9093/api/v2/silences | jq '.[].comment'
```

---

## 트러블슈팅

```bash
# vmalert 로그 확인
kubectl logs -n monitoring -l app=vmalert -f | grep -i "error\|fail"

# Alertmanager 로그 (알림 발송 오류)
kubectl logs -n monitoring -l app=vmalertmanager -f | grep -i "error\|slack\|pagerduty"
```

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| 알림이 발생하지 않음 | 규칙 문법 오류 | vmalert 로그에서 `error` 확인 |
| PENDING에서 FIRING 안됨 | `for` 시간 미충족 | 조건이 실제 만족인지 재확인 |
| Slack 전송 안됨 | Webhook URL 만료 | Slack App에서 새 Webhook 발급 |
| 중복 알림 과다 | `repeat_interval` 짧음 | `repeat_interval: 12h` 이상으로 조정 |
| VMRule 적용 안됨 | Operator가 VMRule 미감지 | `kubectl get vmrule -n monitoring` 확인 |
