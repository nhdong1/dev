# EKS Storage — EBS CSI, EFS CSI & StatefulSets

> Containers (container) theo bản chất là stateless (không trạng thái) — khi container khởi động lại, dữ liệu trong filesystem bị mất. Để lưu trữ persistent data (dữ liệu bền vững), EKS sử dụng CSI Drivers (Container Storage Interface Driver — Trình Điều Khiển Giao Diện Lưu Trữ Container) để kết nối với EBS và EFS.

---

## 📚 Mục Lục

1. [Kubernetes Storage Concepts — Khái Niệm Lưu Trữ](#kubernetes-storage-concepts)
2. [EBS CSI Driver — Lưu Trữ Block](#ebs-csi-driver)
3. [EFS CSI Driver — Lưu Trữ Tệp Được Chia Sẻ](#efs-csi-driver)
4. [StorageClass & Dynamic Provisioning — Cấp Phát Động](#storageclass--dynamic-provisioning)
5. [StatefulSets — Ứng Dụng Có Trạng Thái](#statefulsets)
6. [EBS vs EFS — Khi Nào Dùng Cái Nào](#ebs-vs-efs)
7. [Backup & Snapshot — Sao Lưu](#backup--snapshot)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Kubernetes Storage Concepts

### Các Object Lưu Trữ Chính

```
PV (PersistentVolume — Volume Bền Vững)
  - Tài nguyên storage thực trong cluster
  - Tạo thủ công (static) hoặc tự động qua StorageClass (dynamic)
  - Lifecycle độc lập với Pod

PVC (PersistentVolumeClaim — Yêu Cầu Volume Bền Vững)
  - Pod "yêu cầu" storage: "Tôi cần 10GB ReadWriteOnce"
  - Kubernetes tìm PV phù hợp và bind (gắn kết)
  - Khi PVC được xóa → PV reclaimed theo reclaim policy

StorageClass (Lớp Lưu Trữ)
  - Template để tự động tạo PV khi có PVC
  - Định nghĩa: loại storage (gp3, io2), IOPS, encryption...

CSI Driver (Trình Điều Khiển Giao Diện Lưu Trữ Container)
  - Plugin kết nối Kubernetes với storage system thực (EBS, EFS)
  - EBS CSI Driver, EFS CSI Driver cho AWS
```

### Vòng Đời Của Volume

```
1. Admin tạo StorageClass "ebs-gp3"
2. Developer tạo PVC: "Cần 20GB, StorageClass: ebs-gp3"
3. Kubernetes → EBS CSI Driver → tạo EBS volume 20GB gp3
4. Kubernetes tạo PV, bind với PVC
5. Pod mount PVC vào /data
6. Pod chạy, ghi dữ liệu vào /data → thực ra ghi vào EBS
7. Pod crash → EBS vẫn còn dữ liệu
8. Pod mới mount cùng PVC → đọc lại dữ liệu từ EBS
```

### Access Modes — Chế Độ Truy Cập

| Mode | Viết Tắt | Ý Nghĩa | Hỗ Trợ |
|---|---|---|---|
| ReadWriteOnce | RWO | 1 node đọc+ghi | EBS, EFS |
| ReadOnlyMany | ROX | Nhiều node đọc | EFS |
| ReadWriteMany | RWX | Nhiều node đọc+ghi | EFS (không phải EBS) |
| ReadWriteOncePod | RWOP | 1 Pod đọc+ghi (K8s 1.22+) | EBS |

---

## EBS CSI Driver

### EBS CSI Driver Là Gì?

**EBS CSI Driver (Amazon EBS Container Storage Interface Driver)** cho phép EKS Pods mount EBS volumes như filesystem. AWS duy trì driver này như managed add-on.

### Cài Đặt EBS CSI Driver

```bash
# Cài qua managed add-on (khuyến nghị)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789:role/EBSCSIDriverRole
```

### IAM Policy Cho EBS CSI Driver

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

### StorageClass Cho EBS gp3

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # Đặt làm mặc định
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"          # IOPS tùy chỉnh (gp3 cho phép điều chỉnh độc lập)
  throughput: "125"     # MB/s throughput
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123:key/abc   # KMS key để mã hóa
volumeBindingMode: WaitForFirstConsumer    # Tạo EBS ở cùng AZ với Pod
reclaimPolicy: Retain   # Giữ EBS khi PVC bị xóa (tránh mất data)
allowVolumeExpansion: true    # Cho phép tăng size volume
```

> **Quan trọng:** `WaitForFirstConsumer` nghĩa là EBS volume được tạo cùng AZ (Availability Zone — Vùng Khả Dụng) với Pod. EBS chỉ attach được vào instance cùng AZ. Nếu dùng `Immediate`, EBS có thể tạo ở AZ khác với Pod → Pod Pending mãi.

### PVC và Pod Sử Dụng EBS

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
  - ReadWriteOnce       # EBS chỉ mount được 1 node cùng lúc
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 50Gi

---
# Pod mount PVC
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: postgres
    image: postgres:15
    env:
    - name: PGDATA
      value: /var/lib/postgresql/data/pgdata
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-data    # Tham chiếu đến PVC ở trên
```

### Volume Snapshot — Chụp Ảnh Volume

```yaml
# Tạo VolumeSnapshot (backup theo yêu cầu)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-backup-2026-05-15
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: postgres-data

---
# Restore từ snapshot: tạo PVC mới từ snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: postgres-backup-2026-05-15
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## EFS CSI Driver

### EFS CSI Driver Là Gì?

**EFS CSI Driver (Amazon EFS Container Storage Interface Driver)** cho phép Pods mount **Amazon EFS (Elastic File System — Hệ Thống Tệp Đàn Hồi)** như NFS (Network File System — Hệ Thống Tệp Mạng). EFS hỗ trợ **ReadWriteMany** — nhiều Pods trên nhiều nodes đọc/ghi cùng lúc.

### Khác Biệt Cốt Lõi: EBS vs EFS

```
EBS:
  - Block storage (lưu trữ khối)
  - Gắn với 1 EC2 instance trong 1 AZ
  - Như ổ cứng cắm trực tiếp vào máy
  - Hiệu năng cao, low latency

EFS:
  - Shared filesystem (hệ thống tệp chia sẻ)
  - Truy cập từ nhiều instances, nhiều AZs
  - Như shared network drive
  - Tự động scale không giới hạn, pay-per-GB-used
```

### Cài Đặt EFS CSI Driver

```bash
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-efs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789:role/EFSCSIDriverRole
```

### StorageClass Cho EFS

```yaml
# Static Provisioning (PV tạo thủ công trỏ đến EFS filesystem có sẵn)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi            # Con số symbolic — EFS không có limit thực sự
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany           # Điểm mạnh của EFS
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0123456789abcdef0    # EFS Filesystem ID

---
# Dynamic Provisioning với EFS StorageClass (dùng Access Points)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap    # Mỗi PVC tạo 1 EFS Access Point riêng
  fileSystemId: fs-0123456789abcdef0
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic-provisioning"
```

### Ví Dụ: Web Servers Chia Sẻ Static Files

```yaml
# PVC dùng EFS — nhiều Pods có thể mount cùng lúc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-assets
spec:
  accessModes:
  - ReadWriteMany    # Nhiều Pods đọc/ghi cùng lúc
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi

---
# Deployment với nhiều replicas cùng mount 1 PVC EFS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 10    # 10 Pods cùng mount shared-assets
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: assets
          mountPath: /usr/share/nginx/html/assets
      volumes:
      - name: assets
        persistentVolumeClaim:
          claimName: shared-assets
```

---

## StorageClass & Dynamic Provisioning

### Dynamic vs Static Provisioning

```
Static Provisioning (Cấp Phát Tĩnh):
  Admin → tạo EBS/EFS thủ công → tạo PV manually → Developer tạo PVC

Dynamic Provisioning (Cấp Phát Động):
  Developer tạo PVC → Kubernetes + CSI Driver → tự động tạo EBS/EFS → PV tự động tạo

Dynamic provisioning khuyến nghị cho production (ít manual work hơn)
```

### Cấu Hình Nhiều StorageClasses

```yaml
# gp3 cho general workloads (mặc định)
---
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer

# io2 cho databases cần IOPS cao
---
kind: StorageClass
metadata:
  name: io2-high-iops
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "50000"    # io2 tối đa 64,000 IOPS
volumeBindingMode: WaitForFirstConsumer

# EFS cho shared storage
---
kind: StorageClass
metadata:
  name: efs-shared
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-abc123
```

---

## StatefulSets

### StatefulSet là gì?

**StatefulSet** là Kubernetes workload object cho **stateful applications (ứng dụng có trạng thái)** — ứng dụng cần:
- Identity ổn định (stable network identity — tên Pod không đổi)
- Stable storage (EBS gắn với cùng 1 Pod identity)
- Ordered deployment và scaling

### Deployment vs StatefulSet

| Tính Năng | Deployment | StatefulSet |
|---|---|---|
| **Pod names** | Random: pod-xyz123 | Có thứ tự: mysql-0, mysql-1 |
| **Scaling** | Song song, thứ tự ngẫu nhiên | Theo thứ tự: 0, 1, 2... |
| **Storage** | PVC chia sẻ hoặc không | Mỗi Pod có PVC riêng |
| **DNS** | Không ổn định | mysql-0.mysql.svc.cluster.local |
| **Delete Pod** | Pod mới tên khác | Pod mới tên giống (mysql-0) |
| **Dùng cho** | Stateless apps | Databases, Kafka, Elasticsearch |

### StatefulSet Ví Dụ: MySQL Cluster

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless     # Headless Service — không có ClusterIP
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:       # Mỗi Pod tạo PVC riêng
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: gp3
      resources:
        requests:
          storage: 50Gi

---
# Headless Service cho StatefulSet DNS
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None              # Headless — không có ClusterIP, DNS trả về Pod IP
  selector:
    app: mysql
  ports:
  - port: 3306
```

### StatefulSet Pod DNS

```
Headless Service "mysql-headless" + StatefulSet "mysql":
  mysql-0.mysql-headless.default.svc.cluster.local → 10.0.1.11
  mysql-1.mysql-headless.default.svc.cluster.local → 10.0.1.12
  mysql-2.mysql-headless.default.svc.cluster.local → 10.0.1.13

Ứng dụng có thể kết nối trực tiếp đến từng MySQL instance theo tên ổn định.
Khi mysql-0 crash và restart, nó vẫn là "mysql-0" với cùng PVC và DNS name.
```

---

## EBS vs EFS

| Tiêu Chí | EBS (gp3/io2) | EFS Standard |
|---|---|---|
| **Loại storage** | Block (khối) | File (tệp) / NFS |
| **Access mode** | ReadWriteOnce | ReadWriteMany |
| **Multi-AZ** | Không (1 AZ) | Có (multi-AZ) |
| **Performance** | Cao — microsecond latency | Thấp hơn — millisecond |
| **Pricing** | ~$0.08/GB/tháng (gp3) | ~$0.30/GB/tháng |
| **Max size** | 64 TB per volume | Không giới hạn |
| **Fargate** | Không hỗ trợ | ✅ Hỗ trợ |
| **Dùng cho** | Database, transactional | Shared config, CMS, ML training data |

### Quyết Định Nhanh

```
Dùng EBS khi:
  ✅ Database (PostgreSQL, MySQL, MongoDB) — cần IOPS cao
  ✅ Single Pod, không cần share
  ✅ StatefulSet với mỗi Pod có storage riêng
  ✅ Cost-sensitive workloads

Dùng EFS khi:
  ✅ Nhiều Pods cần đọc/ghi cùng 1 filesystem
  ✅ Web server cluster chia sẻ static assets
  ✅ Machine learning — nhiều training jobs đọc cùng dataset
  ✅ Fargate Pods cần persistent storage
  ✅ CMS (Content Management System) như WordPress với nhiều replicas
```

---

## Backup & Snapshot

### AWS Backup Cho EKS Volumes

```bash
# Enable AWS Backup cho EKS (backup EBS volumes tự động)
aws backup create-backup-plan --backup-plan '{
  "BackupPlanName": "eks-daily-backup",
  "Rules": [{
    "RuleName": "DailyBackup",
    "TargetBackupVaultName": "eks-vault",
    "ScheduleExpression": "cron(0 2 * * ? *)",
    "StartWindowMinutes": 60,
    "CompletionWindowMinutes": 180,
    "Lifecycle": {
      "DeleteAfterDays": 30
    }
  }]
}'
```

### Velero — Backup Tool Phổ Biến Cho Kubernetes

```bash
# Velero backup toàn bộ namespace (bao gồm PVCs)
velero backup create production-backup \
  --include-namespaces production \
  --storage-location aws-backup \
  --snapshot-volumes=true

# Restore vào cluster khác (disaster recovery)
velero restore create --from-backup production-backup \
  --namespace-mappings production:production-restored
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Khi nào dùng EBS, khi nào dùng EFS trong EKS?

**Trả lời:**
> EBS cho stateful single-Pod applications cần performance cao — databases như PostgreSQL, MongoDB. EBS là block storage, chỉ mount được 1 node cùng lúc, phù hợp StatefulSet. EFS cho shared file storage — khi nhiều Pods cần đọc/ghi cùng filesystem, như web servers chia sẻ static assets, ML training data dùng bởi nhiều jobs. EFS đắt hơn (~4x) nhưng hỗ trợ ReadWriteMany và multi-AZ. EFS còn là lựa chọn duy nhất cho Fargate Pods cần persistent storage.

### Câu 2: Tại sao StatefulSet không dùng Deployment cho database?

**Trả lời:**
> Deployment không đảm bảo identity ổn định — Pod names thay đổi mỗi lần restart. Database cluster như MySQL replication cần stable DNS name để các replica biết master là ai. StatefulSet đảm bảo: (1) Pod có tên ổn định (mysql-0, mysql-1); (2) Mỗi Pod có PVC riêng không bị xóa khi Pod restart; (3) Khi mysql-0 restart, nó gắn lại đúng EBS volume cũ của mình; (4) Ordered startup — mysql-0 phải running trước khi mysql-1 start.

### Câu 3: `volumeBindingMode: WaitForFirstConsumer` quan trọng thế nào?

**Trả lời:**
> Với EBS, volume chỉ có thể attach cho EC2 instance trong cùng AZ. Nếu dùng `Immediate`, EBS volume tạo ngay khi PVC được tạo, có thể ở AZ khác với node Pod được schedule. Kết quả: Pod bị Pending mãi vì không thể mount EBS từ AZ khác. `WaitForFirstConsumer` trì hoãn tạo EBS cho đến khi biết Pod được schedule ở AZ nào, sau đó tạo EBS ở đúng AZ đó. Đây là cấu hình bắt buộc cho StorageClass EBS trong multi-AZ cluster.

### Câu 4: Mô tả cách implement disaster recovery cho database trên EKS?

**Trả lời:**
> Approach đa lớp: (1) EBS snapshots tự động hàng ngày qua AWS Backup hoặc VolumeSnapshot; (2) Dùng Velero để backup cả Kubernetes objects (StatefulSet, PVC spec) lẫn data snapshots; (3) Velero lưu backup vào S3 bucket cross-region; (4) Khi disaster: restore Velero backup vào cluster mới, Velero recreate PVC từ EBS snapshot. Test disaster recovery ít nhất mỗi quý để đảm bảo RTO/RPO (Recovery Time Objective/Recovery Point Objective — Mục Tiêu Thời Gian/Điểm Phục Hồi) đúng yêu cầu.

---

**Tiếp theo:** [5-eks-security.md](./5-eks-security.md) — RBAC, IRSA, Pod Security Standards, và Secrets encryption.
