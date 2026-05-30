# 스토리지 구성

VictoriaMetrics는 로컬 디스크(EBS PVC)에 데이터를 저장합니다.
장기 백업은 S3에 `vmbackup`을 사용합니다.

---

## 스토리지 설계 원칙

VictoriaMetrics는 **로컬 SSD 디스크를 기본 저장소**로 사용합니다:

| 항목 | 권장값 | 설명 |
|---|---|---|
| 디스크 타입 | EBS gp3 | IOPS 3,000, 처리량 125 MB/s (기본) |
| 디스크 크기 | 활성 시계열 수 × 보존 기간에 따라 계산 | 아래 계산식 참고 |
| 파일시스템 | ext4 | xfs도 가능 |
| iops | gp3 기본 3,000 | 쓰기 많을 시 증가 고려 |

### 용량 계산식

```
필요 용량 (GB) = 활성 시계열 수 × 보존 기간(일) × 단위 크기

단위 크기 기준 (15초 간격 scraping):
  - VictoriaMetrics: ~0.5 바이트/샘플
  - Prometheus: ~1.5 바이트/샘플  ← VM이 약 3배 효율적

예시:
  100,000 시계열 × 90일 × 86,400초/일 / 15초 × 0.5바이트
  ≈ 25 GB
```

---

## 1. EBS CSI Driver 설치 (EKS 필수)

```bash
# EBS CSI Driver 설치 여부 확인
kubectl get daemonset ebs-csi-node -n kube-system

# 미설치 시 EKS 애드온으로 설치
aws eks create-addon \
  --cluster-name <CLUSTER_NAME> \
  --addon-name aws-ebs-csi-driver \
  --region ap-northeast-2

# IRSA 설정 (EBS CSI Driver 권한 필요)
eksctl create iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=kube-system \
  --name=ebs-csi-controller-sa \
  --attach-policy-arn=arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

---

## 2. StorageClass 생성 (gp3)

EKS 기본 StorageClass는 `gp2`입니다. 성능이 좋은 `gp3`를 사용합니다:

```yaml
# ../ops/config/kubernetes/storageclass-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer   # 파드가 배포된 AZ에 EBS 생성
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"          # 기본값, 필요시 최대 16,000까지 증가
  throughput: "125"     # MB/s, 필요시 최대 1,000까지 증가
  encrypted: "true"
```

```bash
kubectl apply -f ../ops/config/kubernetes/storageclass-gp3.yaml

# 기본 StorageClass 확인
kubectl get storageclass
```

---

## 3. vmstorage PVC 설정 (Helm values)

```yaml
# ../ops/config/helm/values.yaml
victoria-metrics-k8s-stack:
  vmcluster:
    spec:
      vmstorage:
        replicaCount: 2
        storageDataPath: /storage       # 컨테이너 내 데이터 경로

        # PVC 설정
        storage:
          volumeClaimTemplate:
            spec:
              storageClassName: gp3
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 100Gi      # 환경에 맞게 조정

        # 데이터 보존 기간
        retentionPeriod: "90d"         # 형식: Xd, Xw, Xm (일/주/월)
                                       # 기본값: 1 (1개월)

        # 리소스 설정 (100,000 활성 시계열 기준)
        resources:
          requests:
            cpu: "1"
            memory: 2Gi
          limits:
            cpu: "4"
            memory: 8Gi
```

---

## 4. 데이터 보존 (Retention) 설정

```yaml
# 전역 보존 기간 설정
vmcluster:
  spec:
    vmstorage:
      retentionPeriod: "90d"   # 90일
      # 지원 형식:
      #   숫자만: 월 단위 (예: "3" = 3개월)
      #   d: 일 (예: "90d")
      #   w: 주 (예: "13w")
      #   y: 년 (예: "1y")
```

> **비용 계산 기준** (EBS gp3 기준 ap-northeast-2):
> - gp3 가격: 약 $0.08/GB/월
> - 100,000 시계열, 90일 보존 ≈ 25GB ≈ 월 $2
> - 1,000,000 시계열, 90일 보존 ≈ 250GB ≈ 월 $20

---

## 5. 스토리지 상태 확인

```bash
# PVC 상태 확인
kubectl get pvc -n monitoring

# vmstorage 디스크 사용량 확인
kubectl exec -n monitoring vmstorage-vm-stack-0 -- df -h /storage

# 상세 통계 (API 이용)
kubectl port-forward svc/vmstorage-vm-stack-victoria-metrics-k8s-stack 8482:8482 -n monitoring &
curl http://localhost:8482/metrics | grep -E "vm_data_size|vm_rows"

# 보존된 데이터 시간 범위 확인
curl "http://localhost:8481/select/0/prometheus/api/v1/query?query=vm_data_size_bytes" | jq
```

---

## 6. 백업 (vmbackup) 설정

VictoriaMetrics는 `vmbackup`으로 S3에 증분 백업을 지원합니다.

### S3 버킷 생성

```bash
REGION="ap-northeast-2"
BUCKET="mycompany-vm-backup"

aws s3api create-bucket \
  --bucket ${BUCKET} \
  --region ${REGION} \
  --create-bucket-configuration LocationConstraint=${REGION}

# 퍼블릭 액세스 차단
aws s3api put-public-access-block \
  --bucket ${BUCKET} \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### IAM 정책 (vmbackup용)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VMBackupS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::mycompany-vm-backup",
        "arn:aws:s3:::mycompany-vm-backup/*"
      ]
    }
  ]
}
```

### CronJob으로 자동 백업

```yaml
# ../ops/config/kubernetes/vmbackup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vmbackup
  namespace: monitoring
spec:
  schedule: "0 2 * * *"      # 매일 오전 2시 (UTC)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: vmbackup
              image: victoriametrics/vmbackup:v1.101.0
              args:
                - -storageDataPath=/storage
                - -snapshot.createURL=http://vmstorage-vm-stack-victoria-metrics-k8s-stack:8482/snapshot/create
                - -dst=s3://mycompany-vm-backup/daily
                - -credsFilePath=/etc/aws/credentials
              volumeMounts:
                - name: storage
                  mountPath: /storage
                  readOnly: true
          volumes:
            - name: storage
              persistentVolumeClaim:
                claimName: storage-vmstorage-vm-stack-0
          restartPolicy: OnFailure
```

### 수동 백업 실행

```bash
# 1. 스냅샷 생성
kubectl port-forward svc/vmstorage-vm-stack-victoria-metrics-k8s-stack 8482:8482 -n monitoring &

curl http://localhost:8482/snapshot/create
# 응답: {"status":"ok","snapshot":"20260414120000-XXXXXXXX"}

# 2. 스냅샷 목록 확인
curl http://localhost:8482/snapshot/list

# 3. vmbackup으로 S3에 업로드 (로컬에서 실행)
vmbackup \
  -storageDataPath=/path/to/storage \
  -snapshot.name=20260414120000-XXXXXXXX \
  -dst=s3://mycompany-vm-backup/snapshots/20260414
```

---

## 7. 복구 (vmrestore)

```bash
# S3에서 복구 (vmstorage 중단 후 실행)
vmrestore \
  -src=s3://mycompany-vm-backup/daily \
  -storageDataPath=/storage
```

---

## 8. 디스크 용량 확장

EBS gp3는 다운타임 없이 용량 확장이 가능합니다:

```bash
# PVC 용량 증가 (100Gi → 200Gi)
kubectl patch pvc storage-vmstorage-vm-stack-0 -n monitoring \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# 파일시스템 자동 확장 확인 (잠시 후)
kubectl exec -n monitoring vmstorage-vm-stack-0 -- df -h /storage
```

> `allowVolumeExpansion: true` 가 StorageClass에 설정되어 있어야 합니다.

---

## 트러블슈팅

```bash
# PVC가 Pending 상태인 경우
kubectl describe pvc -n monitoring

# vmstorage 디스크 full 경고 확인
kubectl logs -n monitoring vmstorage-vm-stack-0 | grep -i "disk\|storage\|space"

# 디스크 사용량 실시간 확인
kubectl exec -n monitoring vmstorage-vm-stack-0 -- watch -n5 'df -h /storage'
```

| 오류 메시지 | 원인 | 해결 |
|---|---|---|
| `PVC Pending` | StorageClass 없거나 AZ 불일치 | `kubectl describe pvc` 이벤트 확인 |
| `no space left on device` | 디스크 full | retention 단축 또는 PVC 확장 |
| `AccessDenied` (S3 백업) | IAM 권한 부족 | IRSA 정책 재확인 |
| `Error creating snapshot` | vmstorage 부하 | 백업 시간 변경 (트래픽 적은 시간대) |
