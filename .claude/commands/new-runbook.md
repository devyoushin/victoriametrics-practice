새 VictoriaMetrics 운영 런북을 생성합니다.

**사용법**: `/new-runbook <작업명>`  **예시**: `/new-runbook 스토리지 용량 확장`

작업 유형: 스토리지 확장, HA 구성, vmagent 파이프라인 변경, 알람 설정, 업그레이드

런북 포함 내용:
- 사전 체크리스트 (현재 스토리지 사용량, 수집 속도)
- 단계별 kubectl/helm 명령어
- VictoriaMetrics HTTP API 활용
- 롤백 절차
