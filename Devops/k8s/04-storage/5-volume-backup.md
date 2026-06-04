# Volume Backup — Snapshot, Sao Lưu và Khôi Phục Dữ Liệu

> Hướng dẫn chiến lược backup và disaster recovery (phục hồi thảm hoạ) cho dữ liệu Kubernetes: VolumeSnapshot (Snapshot Volume), Velero, Stash, và quy trình khôi phục dữ liệu từ backup.

## Mục Lục

1. [Tại Sao Cần Backup Volume?](#tại-sao-cần-backup-volume)
2. [VolumeSnapshot — Snapshot Kubernetes Native](#volumesnapshot--snapshot-kubernetes-native)
3. [Velero — Backup Toàn Bộ Cluster](#velero--backup-toàn-bộ-cluster)
4. [Stash — Backup Tập Trung Vào Volume](#stash--backup-tập-trung-vào-volume)
5. [Chiến Lược Backup Theo Loại Workload](#chiến-lược-backup-theo-loại-workload)
6. [Quy Trình Khôi Phục (Recovery)](#quy-trình-khôi-phục-recovery)
7. [Disaster Recovery — Phục Hồi Thảm Hoạ](#disaster-recovery--phục-hồi-thảm-hoạ)
8. [Kiểm Tra Backup (Backup Testing)](#kiểm-tra-backup-backup-testing)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tại Sao Cần Backup Volume?

Kubernetes không tự động backup dữ liệu. Các rủi ro thực tế:

```
Rủi ro dữ liệu trong K8s:
├── Xoá nhầm PVC (kubectl delete pvc) với reclaimPolicy: Delete
│   → EBS volume bị xoá ngay lập tức → mất dữ liệu vĩnh viễn
│
├── Lỗi ứng dụng ghi dữ liệu hỏng
│   → Database corruption → cần rollback về trước sự cố
│
├── Ransomware hoặc tấn công xoá dữ liệu
│   → Cần recover từ backup ngoài cluster
│
├── Node failure kéo theo dữ liệu
│   → Local PV mất khi node chết
│
└── Human error (lỗi con người)
    → DROP TABLE, DELETE FROM không có WHERE
    → Cần point-in-time recovery
```

### RPO và RTO — Hai Chỉ Số Quan Trọng

| Chỉ Số | Ý Nghĩa | Ví Dụ |
| ------ | ------- | ----- |
| **RPO (Recovery Point Objective — Mục Tiêu Điểm Phục Hồi)** | Mất tối đa bao nhiêu dữ liệu? | RPO = 1 giờ: chấp nhận mất dữ liệu tối đa 1 giờ trước sự cố |
| **RTO (Recovery Time Objective — Mục Tiêu Thời Gian Phục Hồi)** | Mất tối đa bao nhiêu thời gian khôi phục? | RTO = 4 giờ: hệ thống phải hoạt động trở lại trong 4 giờ |

---

## VolumeSnapshot — Snapshot Kubernetes Native

**VolumeSnapshot** là tính năng Kubernetes (stable từ v1.20) cho phép chụp trạng thái PV tại một thời điểm, tận dụng snapshot API của storage backend (EBS Snapshot, GCS Snapshot…).

### Các Tài Nguyên Snapshot

```
VolumeSnapshotClass  → định nghĩa driver và tham số snapshot (như StorageClass)
VolumeSnapshot       → yêu cầu tạo snapshot từ PVC
VolumeSnapshotContent → đại diện snapshot thật trên storage backend (như PV)
```

### Cài Đặt Snapshot CRD và Controller

```bash
# Cài đặt snapshot CRDs (Custom Resource Definitions)
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/main/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/main/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/main/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Cài đặt snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/main/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/main/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

### VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-vsc
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com           # phải khớp với CSI Driver đang dùng
deletionPolicy: Delete            # Delete: xoá snapshot khi VolumeSnapshot bị xoá
                                  # Retain: giữ snapshot khi VolumeSnapshot bị xoá
parameters:
  tagSpecification_1: "backup=daily"
```

### Tạo VolumeSnapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-20260510
  namespace: production
spec:
  volumeSnapshotClassName: ebs-vsc
  source:
    persistentVolumeClaimName: postgres-pvc   # PVC muốn snapshot
```

```bash
# Áp dụng
kubectl apply -f postgres-snapshot.yaml

# Xem trạng thái snapshot
kubectl get volumesnapshot -n production
# NAME                          READYTOUSE   SOURCEPVC      RESTORESIZE   AGE
# postgres-snapshot-20260510    true         postgres-pvc   20Gi          2m
```

### Khôi Phục PVC Từ Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc-restored
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-encrypted
  resources:
    requests:
      storage: 20Gi
  dataSource:
    name: postgres-snapshot-20260510    # tên VolumeSnapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Tự Động Snapshot Định Kỳ

VolumeSnapshot không có scheduler tích hợp. Dùng CronJob để tự động:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-snapshot
  namespace: production
spec:
  schedule: "0 2 * * *"      # 2 giờ sáng mỗi ngày
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-sa
          restartPolicy: OnFailure
          containers:
            - name: snapshot
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  DATE=$(date +%Y%m%d%H%M)
                  kubectl apply -f - <<EOF
                  apiVersion: snapshot.storage.k8s.io/v1
                  kind: VolumeSnapshot
                  metadata:
                    name: postgres-snap-$DATE
                    namespace: production
                  spec:
                    volumeSnapshotClassName: ebs-vsc
                    source:
                      persistentVolumeClaimName: postgres-pvc
                  EOF
```

---

## Velero — Backup Toàn Bộ Cluster

**Velero** là công cụ backup Kubernetes phổ biến nhất, được VMware donate cho CNCF. Velero backup cả **Kubernetes objects** (Deployment, Service, ConfigMap…) lẫn **PV data** (qua CSI Snapshot hoặc restic/kopia).

```
Velero backup:
├── Kubernetes Objects → lưu manifest YAML vào object storage (S3, GCS, Azure Blob)
└── PV Data → snapshot qua CSI Snapshot hoặc upload data qua restic/kopia
```

### Cài Đặt Velero (Trên AWS)

```bash
# Tạo S3 bucket để lưu backup
aws s3api create-bucket \
  --bucket my-k8s-backup \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Cài đặt Velero CLI
brew install velero   # macOS
# hoặc download binary từ GitHub releases

# Cài đặt Velero vào cluster với AWS plugin
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-k8s-backup \
  --backup-location-config region=ap-southeast-1 \
  --snapshot-location-config region=ap-southeast-1 \
  --secret-file ./credentials-velero \
  --use-node-agent \             # enable kopia/restic cho PV backup
  --features=EnableCSI           # enable CSI Snapshot integration
```

### Tạo Backup Thủ Công

```bash
# Backup toàn bộ cluster
velero backup create full-cluster-backup-20260510

# Backup theo namespace
velero backup create prod-backup \
  --include-namespaces production \
  --ttl 720h         # giữ backup 30 ngày

# Backup chỉ Kubernetes objects (không backup PV data)
velero backup create config-backup \
  --include-namespaces production \
  --exclude-resources persistentvolumeclaims,persistentvolumes

# Backup với label selector
velero backup create app-backup \
  --selector app=postgres \
  --include-namespaces production
```

### Lên Lịch Backup Tự Động

```bash
# Backup hàng ngày lúc 1 giờ sáng, giữ 30 ngày
velero schedule create daily-backup \
  --schedule="0 1 * * *" \
  --ttl 720h \
  --include-namespaces production,staging

# Backup hàng tuần với TTL dài hơn
velero schedule create weekly-backup \
  --schedule="0 0 * * 0" \
  --ttl 2160h   # 90 ngày

# Xem danh sách schedule
velero schedule get
```

### Xem Trạng Thái Backup

```bash
# Xem danh sách backup
velero backup get

# NAME                          STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES
# full-cluster-backup-20260510  Completed   0        0          2026-05-10 01:00:00 +0000 UTC   29d

# Chi tiết backup
velero backup describe full-cluster-backup-20260510

# Xem logs backup
velero backup logs full-cluster-backup-20260510
```

---

## Stash — Backup Tập Trung Vào Volume

**Stash** (by AppsCode) là Kubernetes Operator chuyên backup volume data, hỗ trợ application-aware backup (backup consistent với database):

```bash
# Cài đặt Stash
helm repo add appscode https://charts.appscode.com/stable
helm install stash appscode/stash \
  --namespace kube-system \
  --set features.enterprise=false
```

Stash hỗ trợ database-specific backup hooks (pre/post backup script cho PostgreSQL, MySQL, MongoDB).

---

## Chiến Lược Backup Theo Loại Workload

### Stateless Application (Ứng Dụng Không Có Trạng Thái)

```
Stateless app (chỉ có code, không có persistent data):
→ Không cần backup PV
→ Backup Deployment manifest bằng GitOps (ArgoCD/Flux)
→ Disaster recovery = redeploy từ Git repository
```

### Database Workload

```
Database (PostgreSQL, MySQL, MongoDB):
→ Dùng database-native backup (pg_dump, mysqldump, mongodump)
  kết hợp với CronJob + upload lên S3
→ Kết hợp VolumeSnapshot (crash-consistent) cho point-in-time recovery nhanh
→ WAL (Write-Ahead Log) archiving nếu cần RPO < 1 phút

Ví dụ PostgreSQL:
1. CronJob chạy pg_basebackup → upload S3 (daily)
2. WAL archiving liên tục → upload S3 (every 1–5 phút)
3. VolumeSnapshot mỗi 6 giờ (nhanh hơn, không cần dump)
```

### File Storage Workload

```
File storage (NFS, EFS):
→ VolumeSnapshot định kỳ (nếu backend hỗ trợ)
→ Velero với restic/kopia để sync file lên S3
→ Tránh backup file lớn quá thường xuyên (bandwidth, cost)
```

### Configuration và Secret

```
ConfigMap / Secret:
→ Không backup trong PV — backup bằng Velero (object backup)
→ Tốt hơn: lưu manifest trong Git (GitOps)
→ Secret: dùng External Secrets Operator, Vault — secret lưu ngoài cluster
```

---

## Quy Trình Khôi Phục (Recovery)

### Khôi Phục Từ VolumeSnapshot

```bash
# 1. Xem danh sách snapshot
kubectl get volumesnapshot -n production

# 2. Tạo PVC mới từ snapshot
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc-recovered
  namespace: production
spec:
  dataSource:
    name: postgres-snapshot-20260510
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 20Gi
  storageClassName: gp3-encrypted
EOF

# 3. Scale down StatefulSet (quan trọng!)
kubectl scale statefulset postgres --replicas=0 -n production

# 4. Cập nhật StatefulSet dùng PVC mới
# (hoặc xoá PVC cũ, rename PVC mới trùng tên cũ)

# 5. Scale up lại
kubectl scale statefulset postgres --replicas=1 -n production

# 6. Kiểm tra dữ liệu
kubectl exec -it postgres-0 -- psql -U postgres -c "\dt"
```

### Khôi Phục Từ Velero Backup

```bash
# 1. Xem danh sách backup
velero backup get

# 2. Restore backup cụ thể
velero restore create \
  --from-backup prod-backup-20260510 \
  --include-namespaces production \
  --wait

# 3. Xem trạng thái restore
velero restore get

# 4. Chi tiết restore
velero restore describe prod-backup-20260510-20260511000000

# Restore với namespace mapping (khôi phục vào namespace khác)
velero restore create \
  --from-backup prod-backup-20260510 \
  --namespace-mappings production:production-restored
```

### Khôi Phục Cluster Sau Thảm Hoạ

```bash
# Kịch bản: cluster bị mất hoàn toàn, cần rebuild

# 1. Tạo cluster mới
eksctl create cluster --name new-cluster --region ap-southeast-1

# 2. Cài đặt Velero trỏ vào cùng S3 bucket backup cũ
velero install \
  --provider aws \
  --bucket my-k8s-backup \
  ...

# 3. Velero tự động sync backup metadata từ S3
velero backup get   # thấy backup cũ từ cluster cũ

# 4. Restore
velero restore create --from-backup latest-full-backup

# 5. Kiểm tra workload hoạt động
kubectl get pods --all-namespaces
kubectl get pvc --all-namespaces
```

---

## Disaster Recovery — Phục Hồi Thảm Hoạ

### 3-2-1 Backup Rule

```
Quy tắc 3-2-1:
├── 3 bản sao dữ liệu (1 production + 2 backup)
├── 2 loại storage khác nhau (ví dụ: EBS + S3)
└── 1 bản ở vị trí địa lý khác (cross-region backup)
```

### Cross-Region Backup Với Velero

```bash
# Cấu hình BackupStorageLocation thứ hai ở region khác
velero backup-location create dr-region \
  --provider aws \
  --bucket my-k8s-backup-dr \
  --config region=us-east-1

# Backup gửi đến cả hai location
velero backup create full-backup \
  --storage-location dr-region
```

### RDS Snapshot + EBS Snapshot Kết Hợp

```
Chiến lược Production điển hình (AWS):

Database:
├── RDS với automated backup (daily) + Multi-AZ
├── EBS Snapshot của PVC mỗi 6 giờ
└── pg_dump hàng ngày → S3 với versioning

Kubernetes Objects:
├── Velero daily backup → S3 (primary region)
└── S3 Cross-Region Replication → S3 (DR region)

RPO mục tiêu: 1 giờ
RTO mục tiêu: 4 giờ
```

---

## Kiểm Tra Backup (Backup Testing)

**Backup không được kiểm tra là backup không đáng tin cậy.** Định kỳ kiểm tra khôi phục:

```bash
# Hàng tháng: test restore vào staging environment
velero restore create test-restore-$(date +%Y%m) \
  --from-backup prod-backup-latest \
  --namespace-mappings production:staging-restore \
  --wait

# Kiểm tra dữ liệu trong namespace staging-restore
kubectl exec -n staging-restore postgres-0 -- \
  psql -U postgres -c "SELECT COUNT(*) FROM orders;"

# So sánh với production
kubectl exec -n production postgres-0 -- \
  psql -U postgres -c "SELECT COUNT(*) FROM orders;"

# Dọn dẹp sau test
kubectl delete namespace staging-restore
```

### Checklist Kiểm Tra Backup

```
Định kỳ kiểm tra:
□ Backup job chạy đúng lịch (kiểm tra CronJob logs)
□ Backup thành công trong 24 giờ qua (alert nếu fail)
□ Dung lượng backup hợp lý (không tăng bất thường)
□ Restore test thành công vào môi trường staging (hàng tháng)
□ RTO thực tế (đo thời gian restore test) so với mục tiêu
□ RPO thực tế (dữ liệu mất tối đa bao nhiêu phút)
□ Alert khi backup fail được gửi đến đúng kênh
□ Cross-region backup hoạt động
□ Backup retention policy đúng (không giữ quá ít hoặc quá nhiều)
```

---

## Câu Hỏi Phỏng Vấn

**VolumeSnapshot khác database backup (pg_dump) thế nào?**

> **VolumeSnapshot** là *crash-consistent snapshot* — chụp trạng thái block-level của storage tại một thời điểm, rất nhanh (thường dưới 1 giây) nhưng không đảm bảo application-consistent (database có thể có transaction đang dở). Phù hợp cho disaster recovery nhanh; dữ liệu có thể cần repair sau khi restore.
> **pg_dump / database dump** là *application-consistent backup* — database biết và phối hợp để dump trạng thái nhất quán, có thể dùng cho point-in-time recovery. Chậm hơn (phụ thuộc dung lượng) nhưng dữ liệu sạch hơn.
> **Best practice:** kết hợp cả hai — snapshot để RTO nhanh, dump để backup sạch hơn.

**Velero backup PV data bằng cách nào nếu CSI không hỗ trợ snapshot?**

> Velero dùng **restic hoặc kopia** (file-level backup agent) chạy như sidecar container bên cạnh Pod, đọc dữ liệu trực tiếp từ volume mount path và upload lên object storage (S3). Không cần CSI snapshot support. Nhược điểm: chậm hơn snapshot (đọc từng byte data), tốn băng thông hơn, ứng dụng phải ở trạng thái consistent (hoặc Velero hỗ trợ hooks để quiesce ứng dụng trước khi backup).

**Tại sao cần test restore định kỳ?**

> Backup là hứa hẹn; restore là kiểm chứng. Trong thực tế: backup job chạy thành công nhưng (1) file backup bị corrupt hoặc thiếu, (2) version database thay đổi làm restore format cũ không tương thích, (3) permission hoặc IAM thay đổi làm Velero không đọc được S3, (4) cluster mới thiếu CRD cần thiết để restore resource. Phát hiện những vấn đề này trong bài test hàng tháng tốt hơn nhiều so với phát hiện lúc đang có sự cố production thật sự.

**reclaimPolicy Retain vs Delete — nên chọn gì cho production?**

> **Production: luôn dùng Retain.** Với `Delete`, một lệnh `kubectl delete pvc` (dù vô tình) sẽ cascade xoá EBS volume ngay lập tức — không thể phục hồi nếu không có backup. Với `Retain`, PV và storage vật lý vẫn còn sau khi PVC bị xoá — Admin có cơ hội kiểm tra, backup, rồi mới xoá thủ công. Chi phí là phải có quy trình thủ công để dọn dẹp orphaned PV. Đây là trade-off an toàn > tiện lợi cho production data.
