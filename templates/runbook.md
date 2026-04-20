# [알림명] — VictoriaMetrics 런북

> 심각도: Critical / Warning / Info
> 담당팀: | 최종 수정: YYYY-MM-DD

## 알림 요약
<!-- 이 알림이 무엇을 감지하는지 한 문장으로 -->

## 알림 조건

```metricsql
# 알림 트리거 MetricsQL
```

## 영향도
<!-- 이 알림이 발생했을 때 사용자/서비스에 미치는 영향 -->

## 즉시 확인 사항

```bash
# 1. VictoriaMetrics 상태 확인
kubectl get pods -n monitoring -l app=victoriametrics

# 2. vmui에서 쿼리 테스트
# http://victoriametrics:8428/vmui

# 3. vmstorage 디스크 확인
kubectl exec -n monitoring victoriametrics-0 -- df -h /victoria-metrics-data
```

## 진단 단계

### 1단계: 컴포넌트 상태 파악
```bash
# 내부 메트릭 확인
curl http://victoriametrics:8428/metrics | grep vm_
```

### 2단계: 원인 분석
<!-- 일반적인 원인 목록 -->

### 3단계: 해결 조치
<!-- 단계별 해결 방법 -->

## 에스컬레이션
- 15분 내 해결 불가 시: 팀 리드 호출
- 메트릭 수집 전면 중단 시: 인시던트 선언

## 참고
- 관련 Grafana 대시보드:
- 관련 문서:
