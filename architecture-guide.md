# VictoriaMetrics 아키텍처

---

## 전체 구조

```
                ┌────────────────────────────────────────────────────────────┐
                │               VictoriaMetrics Cluster                      │
                │                                                            │
vmagent ───────▶│  vminsert ──▶ vmstorage-0 (EBS, AZ-a)                    │
Prometheus      │     │                                                      │
remote_write    │  consistent   vmstorage-1 (EBS, AZ-b)                    │◀── Grafana
                │   hashing                │                                 │   (PromQL)
                │              vmselect (fan-out + 중복 제거)                │
                │                                                            │
                │  vmalert ──▶ Recording/Alerting Rules                     │
                │  Alertmanager ──▶ Slack / PagerDuty / Email               │
                └────────────────────────────────────────────────────────────┘
```

---

## 컴포넌트 상세

### vminsert (쓰기 경로 입구)

- Prometheus `remote_write` 요청을 수신 (포트 8480)
- InfluxDB line protocol, OpenTSDB 등 다양한 형식도 지원
- 일관된 해싱(consistent hashing)으로 vmstorage에 분산
- `replicationFactor`에 따라 여러 vmstorage에 동시 복제
- **Stateless** — 자유롭게 수평 확장 가능

```
요청 수신 → 해시 계산(metric + labels) → replicationFactor만큼 vmstorage에 동시 전송
```

**Write 내구성**: `replicationFactor=2` 이면 vmstorage 2개에 동일 데이터 기록
→ vmstorage 1개 장애 시에도 데이터 유실 없음

---

### vmstorage (데이터 저장소)

- 수신한 샘플을 **EBS 디스크**에 VictoriaMetrics 자체 포맷으로 저장
- **자체 압축 알고리즘**: Prometheus TSDB 대비 최대 7배 높은 압축률
- 데이터를 시간 파티션(part)으로 관리하며 백그라운드에서 자동 병합
- WAL 없이 직접 디스크 기록 (메모리 사용량이 낮음)
- **StatefulSet** — PVC에 데이터 영속 보존

```
vmstorage 디렉토리 구조 (/storage):
├── data/
│   ├── big/         # 병합 완료된 큰 파트
│   │   └── YYYY_MM/ # 월별 파티션
│   └── small/       # 최신 수신 파트 (병합 전)
├── snapshots/       # vmbackup 스냅샷
└── tmp/             # 병합 작업 임시 파일
```

**vmstorage 주요 특징**:
- 읽기/쓰기 분리: vminsert가 쓰고, vmselect가 읽는 구조
- 자동 백그라운드 병합으로 I/O 최적화
- `retention` 설정으로 오래된 데이터 자동 삭제

---

### vmselect (읽기 경로)

- Grafana / PromQL 클라이언트의 쿼리 수신 (포트 8481)
- **Fan-out 방식**: 모든 vmstorage에 병렬로 쿼리 전송
- 결과를 수집하고 **중복 제거** (replicationFactor > 1인 경우)
- 결과 캐싱 지원 (인메모리 또는 디스크)
- **Stateless** — 자유롭게 수평 확장 가능

```
vmselect 쿼리 흐름:
Grafana ──▶ vmselect
                ├──▶ vmstorage-0 (병렬)
                └──▶ vmstorage-1 (병렬)
            결과 병합 + 중복 제거 ──▶ Grafana 반환
```

---

### vmagent (메트릭 수집 에이전트)

- Prometheus와 호환되는 경량 scraping 에이전트
- Prometheus보다 **메모리 효율이 10배 이상** 높음
- 수집한 메트릭을 vminsert로 remote_write
- **WAL (Write-Ahead Log)**: 원격 저장소 장애 시 로컬 WAL에 보관 후 재전송
- ServiceMonitor, PodMonitor CRD 지원 (VM Operator 사용 시)
- 여러 원격 저장소에 동시 전송 가능 (fan-out)

```
vmagent 흐름:
Kubernetes Pods/Services
    │ scrape (pull)
    ▼
vmagent (WAL + 필터링 + 리레이블링)
    │ remote_write (push)
    ▼
vminsert (포트 8480)
```

---

### vmalert (규칙 엔진)

- **Recording Rules**: 복잡한 쿼리를 주기적으로 실행하여 새 메트릭 생성
- **Alerting Rules**: 조건 만족 시 Alertmanager에 알림 전송
- vmselect에서 데이터 조회, 결과를 vminsert로 재기록 (recording)
- Prometheus AlertManager API 호환

---

### vmauth (인증 및 라우팅 프록시)

- HTTP 프록시로 vminsert / vmselect 앞에 배치
- 기본 인증(Basic Auth) 또는 Bearer Token 지원
- 멀티 테넌트 라우팅: 테넌트별 다른 vmcluster로 전달

---

## 쓰기 경로 (Write Path)

```
Prometheus / vmagent
    │ HTTP POST (Snappy 압축 protobuf)
    │ 포트 8480
    ▼
vminsert
    │ consistent hash(metric + labels)
    │ replicationFactor=2
    ├──▶ vmstorage-0 (AZ-a, EBS)  ──┐
    └──▶ vmstorage-1 (AZ-b, EBS)  ──┘ 둘 다 성공해야 200 OK

         vmstorage 내부 저장:
         small parts → 백그라운드 병합 → big parts
```

---

## 읽기 경로 (Read Path)

```
Grafana / PromQL 클라이언트
    │ HTTP GET /select/0/prometheus/...
    │ 포트 8481
    ▼
vmselect
    │ fan-out (모든 vmstorage에 병렬 전송)
    ├──▶ vmstorage-0 → 결과 반환
    └──▶ vmstorage-1 → 결과 반환

결과 병합 + 중복 제거 → Grafana 반환
```

> **중복 제거**: replicationFactor > 1로 동일 데이터가 여러 vmstorage에 있을 때
> vmselect가 타임스탬프 기준으로 중복을 자동 제거합니다.

---

## 데이터 생명주기

```
[0s]    vmagent scrape → remote_write → vminsert → vmstorage (small part)
[1~2h]  vmstorage: small parts 자동 병합 → big part
[1d]    vmstorage: 일별 big parts 병합
[1mo]   vmstorage: 월별 파티션으로 정리
[1y]    retention 초과 → 자동 삭제
```

---

## Single-node vs Cluster 비교

| 항목 | Single-node | Cluster |
|---|---|---|
| 구성 | 단일 바이너리 (`victoria-metrics`) | 3개 컴포넌트 분리 |
| 스케일 | 수직 확장만 가능 | 수평 확장 가능 |
| 고가용성 | 지원 안 함 | 지원 (replication) |
| 권장 규모 | < 1M 활성 시계열 | > 1M 활성 시계열 |
| 운영 복잡도 | 매우 낮음 | 보통 |
| 사용 사례 | 개발/소규모 | 프로덕션 |

---

## URL 경로 구조

VictoriaMetrics Cluster는 URL에 테넌트 ID를 포함합니다:

```
# 쓰기: /insert/{accountID}/prometheus/...
curl -X POST http://vminsert:8480/insert/0/prometheus/api/v1/write

# 읽기: /select/{accountID}/prometheus/...
curl http://vmselect:8481/select/0/prometheus/api/v1/query?query=up

# accountID=0: 기본 테넌트 (단일 테넌트 환경에서 사용)
# accountID=multitenant: 모든 테넌트 데이터 통합 조회
```

---

## 참고 링크

- [VictoriaMetrics Cluster 공식 문서](https://docs.victoriametrics.com/cluster-victoriametrics/)
- [VictoriaMetrics vs Prometheus 비교](https://docs.victoriametrics.com/#prominent-features)
- [vmagent 공식 문서](https://docs.victoriametrics.com/vmagent/)
