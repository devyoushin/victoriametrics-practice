새 VictoriaMetrics 가이드 문서를 생성합니다.

**사용법**: `/new-doc <주제명>`  **예시**: `/new-doc vmcluster-setup`

주제 분류: architecture, storage, ha, vmagent, alerting, recording-rules, grafana-datasource

`<주제명>-guide.md` 생성 시 포함 내용:
- 환경 설정 반영 (EKS, VictoriaMetrics 버전)
- Helm values 또는 Kubernetes 매니페스트 예시
- MetricsQL 쿼리 예시 (PromQL 호환)
- vmctl/curl 확인 명령어
- 트러블슈팅 섹션
