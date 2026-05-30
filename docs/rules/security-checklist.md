# 보안 체크리스트 — victoriametrics-practice

## VictoriaMetrics 보안 설정

### 인증/인가
- [ ] VictoriaMetrics API 인증 설정 (vmauth 프록시 또는 Basic Auth)
- [ ] vmauth로 테넌트별 접근 제어
- [ ] `/api/v1/admin/*` 관리 API 접근 제한 (내부망 전용)
- [ ] 데이터 삭제 API(`delete_series`) 접근 제한

### 스토리지 보안
- [ ] 스토리지 경로 파일 시스템 권한 설정 (victoriametrics 사용자만 접근)
- [ ] S3 백업 버킷 암호화 및 접근 제한 (vmbackup 사용 시)
- [ ] 스냅샷 파일 접근 제한

### 네트워크 보안
- [ ] VictoriaMetrics 포트(8428/8480/8481/8482) 외부 노출 금지
- [ ] vmagent → VictoriaMetrics TLS 설정
- [ ] NetworkPolicy로 접근 제한

### vmagent 보안
- [ ] scrape 자격증명 K8s Secret으로 관리
- [ ] Bearer token 파일 권한 설정 (600)
- [ ] 민감 레이블 metric_relabel_configs로 제거

## 정기 보안 점검 (월별)
- [ ] 관리 API 접근 로그 검토
- [ ] scrape 자격증명 순환
- [ ] VictoriaMetrics 버전 업데이트 확인
- [ ] 스토리지 백업 무결성 검증
