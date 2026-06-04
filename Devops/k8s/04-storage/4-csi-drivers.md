# CSI Drivers — Container Storage Interface Trên Kubernetes

> Giải thích CSI (Container Storage Interface — Giao Diện Lưu Trữ Container): kiến trúc, cách hoạt động, so sánh CSI Driver phổ biến trên AWS, GCP, Azure, và hướng dẫn cài đặt cơ bản.

## Mục Lục

1. [CSI Là Gì?](#csi-là-gì)
2. [Kiến Trúc CSI](#kiến-trúc-csi)
3. [CSI Driver Trên AWS EKS](#csi-driver-trên-aws-eks)
4. [CSI Driver Trên Google GKE](#csi-driver-trên-google-gke)
5. [CSI Driver Trên Azure AKS](#csi-driver-trên-azure-aks)
6. [CSI Driver Độc Lập Cloud](#csi-driver-độc-lập-cloud)
7. [So Sánh CSI Driver Theo Tính Năng](#so-sánh-csi-driver-theo-tính-năng)
8. [Cài Đặt và Cấu Hình](#cài-đặt-và-cấu-hình)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CSI Là Gì?

**CSI (Container Storage Interface)** là tiêu chuẩn giao diện mở, được CNCF (Cloud Native Computing Foundation — Tổ Chức Điện Toán Đám Mây Native) định nghĩa, cho phép storage vendor viết plugin lưu trữ mà **không cần thay đổi code Kubernetes core**.

### Trước CSI — In-tree Plugin (Plugin Tích Hợp Sẵn)

```
Trước K8s 1.13: mọi storage được tích hợp trực tiếp vào binary Kubernetes
  ├── awsElasticBlockStore   → code AWS trong core K8s
  ├── gcePersistentDisk      → code GCP trong core K8s
  ├── azureDisk              → code Azure trong core K8s
  └── nfs, ceph, glusterfs...

Vấn đề:
- Lỗi trong plugin ảnh hưởng toàn bộ cluster
- Vendor phải đợi K8s release cycle (~4 tháng) để ship fix
- Kubernetes binary ngày càng lớn
```

### Sau CSI — Out-of-tree Plugin (Plugin Ngoài)

```
Từ K8s 1.13+: CSI tách storage plugin ra ngoài Kubernetes core
  Kubernetes core  ──CSI gRPC API──►  CSI Driver (Pod riêng)  ──vendor SDK──►  Cloud Storage
  
Lợi ích:
- Vendor tự deploy và update driver độc lập với K8s version
- Bug trong driver không ảnh hưởng control plane
- Kubernetes core gọn hơn
- Hỗ trợ storage mới không cần đợi K8s release
```

---

## Kiến Trúc CSI

### Các Thành Phần CSI Driver

```
┌─────────────────────────────────────────────┐
│                  Kubernetes                  │
│  ┌──────────────┐    ┌────────────────────┐  │
│  │ CSI Controller│    │   CSI Node Plugin  │  │
│  │   (DaemonSet) │    │   (DaemonSet)      │  │
│  │               │    │                   │  │
│  │ - Provision   │    │ - NodeStageVolume  │  │
│  │ - Delete      │    │ - NodePublishVol   │  │
│  │ - Snapshot    │    │ - NodeUnpublishVol │  │
│  │ - Resize      │    │ - NodeGetInfo      │  │
│  └──────┬────────┘    └────────┬───────────┘  │
│         │ gRPC                 │ gRPC         │
└─────────┼─────────────────────┼──────────────┘
          │                     │
          ▼                     ▼
    CSI Controller Plugin   CSI Node Plugin
    (chạy 1 Pod / cluster)  (chạy trên mỗi node)
          │                     │
          ▼                     ▼
    Cloud Storage API      Attach/Mount Volume
    (tạo/xoá volume)       (gắn vào container)
```

### Ba Hoạt Động Chính

| Hoạt Động | Thực Hiện Bởi | Mô Tả |
| --------- | ------------- | ----- |
| **Provision** | CSI Controller | Tạo volume trên cloud (ví dụ: tạo EBS volume) |
| **Attach** | CSI Controller / Node | Attach volume vào node (ví dụ: attach EBS vào EC2) |
| **Mount** | CSI Node Plugin | Mount volume vào filesystem container |

### Sidecar Containers Của CSI

Kubernetes cung cấp các **sidecar container** chuẩn, CSI Driver không cần tự implement:

| Sidecar | Vai Trò |
| ------- | ------- |
| `external-provisioner` | Theo dõi PVC, gọi CSI `CreateVolume` |
| `external-attacher` | Theo dõi VolumeAttachment, gọi CSI `ControllerPublishVolume` |
| `external-resizer` | Theo dõi PVC resize request, gọi CSI `ControllerExpandVolume` |
| `external-snapshotter` | Theo dõi VolumeSnapshot, gọi CSI `CreateSnapshot` |
| `node-driver-registrar` | Đăng ký CSI Node Plugin với kubelet |

---

## CSI Driver Trên AWS EKS

### AWS EBS CSI Driver

**EBS (Elastic Block Store — Kho Lưu Trữ Khối Co Giãn)** là block storage của AWS, phù hợp cho database và workload cần IOPS cao. Chỉ hỗ trợ **RWO** (ReadWriteOnce).

```yaml
# StorageClass với EBS CSI Driver
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3                 # gp3: General Purpose SSD thế hệ mới
  encrypted: "true"         # mã hoá với AWS KMS
  # iops: "3000"            # IOPS baseline (gp3 mặc định 3000)
  # throughput: "125"       # MB/s (gp3 mặc định 125 MB/s)
volumeBindingMode: WaitForFirstConsumer   # quan trọng: đợi Pod schedule để chọn AZ
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Các loại EBS volume:**

| Loại | IOPS | Throughput | Use Case |
| ---- | ---- | ---------- | -------- |
| `gp3` | 3,000–16,000 | 125–1,000 MB/s | General purpose — khuyến nghị |
| `io2` | Đến 64,000 | Đến 1,000 MB/s | Database OLTP cần IOPS cực cao |
| `st1` | Thấp | 500 MB/s | Data warehouse, log processing |
| `sc1` | Rất thấp | 250 MB/s | Cold data, backup |

**Cài đặt EBS CSI Driver trên EKS:**

```bash
# Cách 1: Dùng EKS Add-on (khuyến nghị)
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole

# Cách 2: Dùng Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole
```

> EBS CSI Driver cần **IAM Role** (quyền tạo/xoá EBS volume). Trên EKS, dùng **IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho Service Account)** để gắn IAM Role cho service account của driver.

### AWS EFS CSI Driver

**EFS (Elastic File System — Hệ Thống File Co Giãn)** là managed NFS của AWS. Hỗ trợ **RWX** — nhiều Pod trên nhiều node cùng đọc/ghi.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap   # efs-ap: tạo Access Point riêng cho mỗi PVC
  fileSystemId: fs-0123456789abcdef
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
```

> EFS tính phí theo dung lượng thực tế sử dụng — phù hợp cho shared storage lớn, không lãng phí. EBS tính phí theo dung lượng đã cấp phát dù dùng hay không.

---

## CSI Driver Trên Google GKE

### GCE Persistent Disk CSI Driver

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-rwo
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd              # pd-standard: HDD | pd-ssd: SSD | pd-balanced: Balanced SSD
  replication-type: none    # none | regional-pd (multi-zone replica)
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Regional Persistent Disk** — replicate volume sang 2 zone, đảm bảo HA:

```yaml
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowedTopologies:
  - matchLabelExpressions:
    - key: topology.gke.io/zone
      values:
        - us-central1-a
        - us-central1-b
```

### GCS Filestore (NFS — Hỗ Trợ RWX)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-rwx
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: standard     # standard | premium | enterprise
  network: default
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## CSI Driver Trên Azure AKS

### Azure Disk CSI Driver

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS        # Standard_LRS | StandardSSD_LRS | Premium_LRS | UltraSSD_LRS
  kind: managed               # managed: Azure Managed Disk
  diskEncryptionSetID: /subscriptions/.../diskEncryptionSets/myDES
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
```

**Các SKU Azure Disk:**

| SKU | Loại | IOPS | Use Case |
| --- | ---- | ---- | -------- |
| `Standard_LRS` | HDD | Thấp | Dev/test |
| `StandardSSD_LRS` | SSD | Trung bình | General purpose |
| `Premium_LRS` | SSD Premium | Cao | Database production |
| `UltraSSD_LRS` | Ultra SSD | Cực cao | Workload OLTP cực kỳ nhạy latency |

### Azure Files CSI Driver (Hỗ Trợ RWX)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-files-rwx
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS       # Premium_LRS hỗ trợ NFS protocol
  protocol: nfs              # nfs (khuyến nghị) hoặc smb
allowVolumeExpansion: true
```

> Azure Files với NFS protocol yêu cầu Premium tier. SMB protocol hỗ trợ mọi tier nhưng không tương thích với Linux filesystem permissions đầy đủ.

---

## CSI Driver Độc Lập Cloud

### NFS CSI Driver

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.internal     # địa chỉ NFS server
  share: /exports/k8s             # đường dẫn share trên NFS server
  mountPermissions: "0755"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

### Longhorn — Distributed Block Storage

**Longhorn** là distributed block storage được CNCF incubate, chạy trực tiếp trên các node Kubernetes — không cần external storage system:

```bash
# Cài đặt Longhorn
helm repo add longhorn https://charts.longhorn.io
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace

# StorageClass tự động được tạo: longhorn (default)
```

Tính năng nổi bật:
- **Replicated volumes** — dữ liệu được replicate sang nhiều node
- **Snapshots và backup** tích hợp sẵn (ra S3, NFS)
- **Live migration** — di chuyển volume sang node khác không downtime
- **Volume expansion** trực tuyến

### Rook Ceph — Distributed Storage

**Rook** là Kubernetes Operator (bộ điều hành Kubernetes) để quản lý Ceph cluster:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
```

Rook-Ceph hỗ trợ đầy đủ RWO (RBD — RADOS Block Device), ROX và RWX (CephFS).

---

## So Sánh CSI Driver Theo Tính Năng

| Driver | RWO | RWX | Snapshot | Resize | Multi-AZ | Chi Phí |
| ------ | --- | --- | -------- | ------ | -------- | ------- |
| **AWS EBS** | ✅ | ❌ | ✅ | ✅ | ❌ | Trung bình |
| **AWS EFS** | ✅ | ✅ | ❌ | N/A | ✅ | Cao |
| **GCE PD** | ✅ | ❌ | ✅ | ✅ | ✅ (Regional) | Trung bình |
| **GCS Filestore** | ✅ | ✅ | ✅ | ✅ | ❌ | Cao |
| **Azure Disk** | ✅ | ❌ | ✅ | ✅ | ❌ | Trung bình |
| **Azure Files** | ✅ | ✅ | ✅ | ✅ | ✅ | Cao |
| **NFS CSI** | ✅ | ✅ | ❌ | ✅ | Phụ thuộc NFS server | Thấp (tự host) |
| **Longhorn** | ✅ | ❌ | ✅ | ✅ | ✅ (replica) | Thấp (tự host) |
| **Rook Ceph** | ✅ | ✅ | ✅ | ✅ | ✅ | Thấp (tự host) |

---

## Cài Đặt và Cấu Hình

### Kiểm Tra CSI Driver Đang Cài

```bash
# Xem danh sách CSI Driver trong cluster
kubectl get csidrivers

# Xem chi tiết một driver
kubectl describe csidriver ebs.csi.aws.com

# Xem CSI Node — driver đang chạy trên node nào
kubectl get csinodes
```

### Kiểm Tra StorageClass

```bash
# Danh sách StorageClass
kubectl get storageclass

# Xem default StorageClass (có annotation is-default-class: "true")
kubectl get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}'
```

### Kiểm Tra Provisioning Có Hoạt Động Không

```bash
# Tạo PVC test
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests:
      storage: 1Gi
EOF

# Xem trạng thái PVC
kubectl get pvc test-pvc
# Nếu STATUS = Bound → driver hoạt động
# Nếu STATUS = Pending → xem events: kubectl describe pvc test-pvc

# Dọn dẹp
kubectl delete pvc test-pvc
```

### Debug CSI Driver

```bash
# Xem log CSI Controller (Pod chạy trong kube-system)
kubectl logs -n kube-system -l app=ebs-csi-controller -c ebs-plugin --tail=50

# Xem log CSI Node DaemonSet
kubectl logs -n kube-system -l app=ebs-csi-node -c ebs-plugin --tail=50

# Xem events liên quan đến PVC/PV
kubectl get events --field-selector reason=ProvisioningSucceeded,reason=ProvisioningFailed
```

---

## Câu Hỏi Phỏng Vấn

**CSI Driver khác gì so với in-tree storage plugin cũ?**

> **In-tree plugin** được compile thẳng vào binary Kubernetes — mọi bug fix hay tính năng mới phải đợi K8s release cycle (~4 tháng). Một lỗi trong plugin có thể crash toàn bộ kubelet. **CSI Driver** là plugin độc lập, chạy trong Pod riêng, được vendor deploy và update độc lập hoàn toàn với K8s. Kubernetes core chỉ giao tiếp với CSI qua gRPC API chuẩn. Kể từ K8s 1.21, tất cả in-tree plugin cloud (EBS, GCE PD, Azure Disk) đã bị deprecated và chuyển sang CSI.

**Tại sao EBS CSI Driver cần IAM Role?**

> EBS CSI Driver cần gọi AWS API để tạo/xoá/attach EBS volume — những API này yêu cầu xác thực IAM. Trên EKS, driver chạy dưới dạng Pod với ServiceAccount. **IRSA (IAM Roles for Service Accounts)** cho phép gắn IAM Role vào ServiceAccount thông qua annotation — driver sẽ tự động nhận temporary credentials từ AWS STS mà không cần hard-code access key. Đây là best practice bảo mật: credentials ngắn hạn, phạm vi hẹp, không lộ trong code.

**Khi nào chọn EBS vs EFS trên AWS?**

> **EBS:** block storage, latency thấp (~1ms), IOPS cao — chọn khi cần performance (database, OLTP). Chỉ hỗ trợ RWO (một node). Chi phí theo dung lượng cấp phát.
> **EFS:** managed NFS, latency cao hơn (~1–10ms), hỗ trợ RWX — chọn khi nhiều Pod cần chia sẻ file (shared uploads, config distribution, ML model serving). Chi phí theo dung lượng thực dùng.
> Nguyên tắc: database → EBS; shared file → EFS; object storage có thể → S3 (không cần CSI).

**volumeBindingMode WaitForFirstConsumer quan trọng thế nào với cloud disk?**

> Rất quan trọng trong cluster multi-AZ. Cloud disk (EBS, GCE PD, Azure Disk) phải nằm cùng AZ với node chạy Pod — chúng không thể attach cross-AZ. Nếu dùng `Immediate`, PVC trigger provisioning ngay lập tức trước khi biết Pod sẽ chạy trên node AZ nào → risk cao disk được tạo ở AZ khác → Pod không mount được → stuck ở `ContainerCreating`. `WaitForFirstConsumer` đợi Scheduler chọn node (và AZ) rồi mới tạo disk trong đúng AZ đó.
