VictoriaMetrics 트러블슈팅 케이스를 추가합니다.

**사용법**: `/add-troubleshooting <증상 설명>`  **예시**: `/add-troubleshooting 높은 카디널리티로 메모리 급증`

형식:
```markdown
### <증상>
**원인**: <근본 원인>
**확인**:
\`\`\`bash
curl http://victoriametrics:8428/api/v1/status/tsdb?topN=20
curl http://victoriametrics:8428/metrics | grep vm_
\`\`\`
**해결**: <해결 방법>
```
`troubleshooting-guide.md`에 추가하세요.
