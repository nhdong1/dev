# PersistentVolume — PV, PVC, StorageClass và Dynamic Provisioning

> Giải thích chi tiết hệ thống lưu trữ bền vững Kubernetes: PersistentVolume (PV — Lưu Trữ Ổn Định), PersistentVolumeClaim (PVC — Yêu Cầu Lưu Trữ), StorageClass (Lớp Lưu Trữ), và Dynamic Provisioning (Cấp Phát Động).

## Mục Lục

1. [Tổng Quan PV / PVC / StorageClass](#tổng-quan-pv--pvc--storageclass)
2. [PersistentVolume (PV)](#persistentvolume-pv)
3. [PersistentVolumeClaim (PVC)](#persistentvolumeclaim-pvc)
4. [Quá Trình Binding PV ↔ PVC](#quá-trình-binding-pv--pvc)
5. [StorageClass và Dynamic Provisioning](#storageclass-và-dynamic-provisioning)
6. [Reclaim Policy — Chính Sách Thu Hồi](#reclaim-policy--chính-sách-thu-hồi)
7. [Volume Expansion — Mở Rộng Dung Lượng](#volume-expansion--mở-rộng-dung-lượng)
8. [StatefulSet và volumeClaimTemplates](#statefulset-và-volumeclaimtemplates)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan PV / PVC / StorageClass

Ba tài nguyên này tạo thành hệ thống **abstraction** (trừu tượng hoá) tách biệt việc *cung cấp* lưu trữ (Admin / cloud) khỏi việc *sử dụng* lưu trữ (Developer):

```
ADMIN / CLOUD PROVIDER              DEVELOPER / APP
──────────────────────              ──────────────────
StorageClass                        PVC (yêu cầu 10Gi)
    │ provisioner                       │
    ▼                                   ▼
PersistentVolume ◄──── Kubernetes ────► BOUND
(đại diện EBS 20Gi)     binding         │
                                        ▼
                                    Pod mount PVC
                                    container thấy /data
```

| Tài Nguyên | Ai Tạo | Phạm Vi | Vai Trò |
| ---------- | ------ | ------- | ------- |
| **PV** | Admin hoặc StorageClass tự tạo | Cluster-level | Đại diện storage vật lý |
| **PVC** | Developer | Namespace-level | Yêu cầu storage |
| **StorageClass** | Admin | Cluster-level | Định nghĩa cách tạo PV tự động |

---

## PersistentVolume (PV)

**PV** là tài nguyên cluster-level biểu diễn một đơn vị lưu trữ vật lý đã được cấp phát và sẵn sàng sử dụng.

### Manifest PV (Static Provisioning — Cấp Phát Tĩnh)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgres-data
  labels:
    type: ssd
    env: production
spec:
  capacity:
    storage: 50Gi

  accessModes:
    - ReadWriteOnce            # RWO — chỉ 1 node đọc/ghi

  persistentVolumeReclaimPolicy: Retain   # giữ lại data sau khi PVC bị xoá

  storageClassName: manual     # PVC phải khai báo đúng storageClassName này

  # Backend: NFS (Network File System — Hệ Thống File Mạng)
  nfs:
    server: nfs-server.internal
    path: /exports/postgres

  # Hoặc backend: AWS EBS (Elastic Block Store — Kho Lưu Trữ Khối Co Giãn)
  # csi:
  #   driver: ebs.csi.aws.com
  #   volumeHandle: vol-0a1b2c3d4e5f67890   # EBS volume ID
  #   fsType: ext4
```

### Các Trạng Thái PV (Phase)

| Phase | Ý Nghĩa |
| ----- | ------- |
| `Available` | Sẵn sàng, chưa được bind với PVC nào |
| `Bound` | Đã bind với một PVC |
| `Released` | PVC đã xoá nhưng PV chưa được admin thu hồi |
| `Failed` | Cố gắng thu hồi tự động thất bại |

### Xem Trạng Thái PV

```bash
kubectl get pv
# NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM
# pv-postgres-data    50Gi       RWO            Retain           Available   

kubectl describe pv pv-postgres-data
```

---

## PersistentVolumeClaim (PVC)

**PVC** là yêu cầu lưu trữ từ phía ứng dụng. Developer khai báo dung lượng cần, access mode, và optional StorageClass — Kubernetes tự động tìm PV phù hợp.

### Manifest PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 20Gi           # yêu cầu tối thiểu 20Gi
                              # Kubernetes sẽ chọn PV có dung lượng ≥ 20Gi

  storageClassName: gp3-encrypted   # dùng StorageClass này để dynamic provisioning
  # storageClassName: ""            # binding với PV không có StorageClass
  # storageClassName: manual        # binding với PV có storageClassName: manual

  # Selector để lọc PV (tuỳ chọn — chỉ dùng với static provisioning)
  selector:
    matchLabels:
      type: ssd
      env: production
```

### Dùng PVC Trong Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  containers:
    - name: postgres
      image: postgres:15
      env:
        - name: POSTGRES_DB
          value: myapp
      volumeMounts:
        - name: pg-data
          mountPath: /var/lib/postgresql/data

  volumes:
    - name: pg-data
      persistentVolumeClaim:
        claimName: postgres-pvc    # tham chiếu tên PVC
        readOnly: false
```

### Trạng Thái PVC

```bash
kubectl get pvc -n production
# NAME           STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS    AGE
# postgres-pvc   Bound    pvc-abc123-xyz789    20Gi       RWO            gp3-encrypted   5m

# PVC Pending = chưa tìm được PV phù hợp
# PVC Bound   = đã được bind với PV
# PVC Lost    = PV bị xoá khi PVC vẫn còn tồn tại
```

---

## Quá Trình Binding PV ↔ PVC

Kubernetes sử dụng **PV Controller** để matching PVC với PV phù hợp:

```
PVC yêu cầu:                    PV được chọn khi:
─────────────                   ─────────────────
storage: 20Gi           →       capacity.storage ≥ 20Gi
accessModes: RWO        →       accessModes chứa RWO
storageClassName: X     →       storageClassName == X
selector matchLabels    →       PV labels khớp selector
```

**Thuật toán binding:**
1. Loại bỏ PV không phù hợp storageClass, access mode, dung lượng
2. Trong số còn lại, chọn PV có dung lượng **nhỏ nhất đủ dùng** (best-fit) → tránh lãng phí
3. Nếu không có PV nào → PVC ở trạng thái `Pending` cho đến khi có PV mới

> Với dynamic provisioning (StorageClass có provisioner), Kubernetes bỏ qua bước matching PV có sẵn và gọi provisioner để tạo PV mới phù hợp với yêu cầu PVC.

---

## StorageClass và Dynamic Provisioning

**StorageClass** là template để tạo PV tự động. Khi PVC khai báo `storageClassName`, Kubernetes gọi provisioner tương ứng để tạo PV (và storage vật lý).

### Manifest StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # đặt làm default StorageClass

provisioner: ebs.csi.aws.com      # CSI Driver sẽ xử lý provisioning

parameters:
  type: gp3                        # loại EBS volume
  encrypted: "true"                # bật mã hoá AES-256
  kmsKeyId: "arn:aws:kms:..."      # KMS key để mã hoá (tuỳ chọn)
  throughput: "125"                # MB/s throughput (chỉ gp3)
  iopsPerGB: "3000"                # IOPS baseline

reclaimPolicy: Delete              # xoá PV và EBS khi PVC bị xoá
allowVolumeExpansion: true         # cho phép resize volume trực tuyến
volumeBindingMode: WaitForFirstConsumer   # đợi Pod được schedule rồi mới tạo volume
                                          # (đảm bảo volume ở cùng AZ với node)

mountOptions:
  - noatime                        # tùy chọn mount để tăng performance
```

### volumeBindingMode — Thời Điểm Tạo Volume

| Mode | Khi Nào Tạo PV | Khi Nào Dùng |
| ---- | --------------- | ------------ |
| `Immediate` | Ngay khi PVC được tạo | Storage không phụ thuộc AZ (NFS, Ceph) |
| `WaitForFirstConsumer` | Khi Pod đầu tiên dùng PVC được schedule | Cloud disk (EBS, GCE PD, Azure Disk) phải cùng AZ với node |

> Với EBS trên AWS, **bắt buộc dùng `WaitForFirstConsumer`** nếu cluster multi-AZ (nhiều vùng khả dụng). Nếu dùng `Immediate`, EBS volume có thể được tạo ở AZ khác với node, gây lỗi mount.

### StorageClass Phổ Biến Theo Cloud

```yaml
# AWS EKS — General Purpose SSD
provisioner: ebs.csi.aws.com
parameters:
  type: gp3

# AWS EKS — EFS (Elastic File System) hỗ trợ RWX
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap

# Google GKE — Standard SSD
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd

# Azure AKS — Premium SSD
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_LRS

# NFS (bất kỳ cloud nào có NFS server)
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.internal
  share: /exports
```

### Xem StorageClass Trong Cluster

```bash
kubectl get storageclass
# NAME                PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
# gp3-encrypted (default)  ebs.csi.aws.com   Delete          WaitForFirstConsumer   true
# gp2             ebs.csi.aws.com             Delete          WaitForFirstConsumer   false
```

---

## Reclaim Policy — Chính Sách Thu Hồi

Khi PVC bị xoá, PV chuyển sang `Released`. **Reclaim Policy** quyết định điều gì xảy ra tiếp theo:

| Policy | Hành Động | Dùng Khi Nào |
| ------ | --------- | ------------ |
| **Retain** | Giữ PV và dữ liệu — Admin phải xử lý thủ công | Production — không bao giờ muốn mất dữ liệu |
| **Delete** | Xoá PV và storage vật lý tự động | Dev/staging — muốn tự dọn dẹp |
| **Recycle** | ⚠️ Deprecated (đã bị loại bỏ) — dùng dynamic provisioning thay thế | Không dùng |

### Quy Trình Với Retain Policy (Production)

```
1. PVC bị xoá (kubectl delete pvc, hoặc app uninstalled)
2. PV chuyển sang trạng thái "Released"
3. PV không thể bind với PVC mới (dù spec khớp)
4. Admin kiểm tra dữ liệu:
   - Backup nếu cần
   - Hoặc xoá PV thủ công: kubectl delete pv <pv-name>
5. Storage vật lý (EBS, GCE PD) vẫn còn → Admin xoá thủ công trên cloud console
```

### Đổi Reclaim Policy Sau Khi Tạo

```bash
kubectl patch pv pv-postgres-data \
  -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

---

## Volume Expansion — Mở Rộng Dung Lượng

Với StorageClass có `allowVolumeExpansion: true`, có thể tăng dung lượng PVC mà **không cần downtime**:

```bash
# 1. Sửa PVC — tăng request storage từ 20Gi lên 50Gi
kubectl patch pvc postgres-pvc \
  -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# 2. Kubernetes gọi CSI Driver để resize volume vật lý (EBS)
# 3. Sau khi volume vật lý được resize, cần Pod restart để filesystem expand
kubectl rollout restart deployment/postgres

# 4. Kiểm tra
kubectl get pvc postgres-pvc
# CAPACITY sẽ hiển thị 50Gi sau khi hoàn tất
```

**Lưu ý quan trọng:**
- Chỉ tăng dung lượng được — **không thể giảm** (downsize)
- Một số storage backend cần Pod unmount volume trước khi resize (offline expansion)
- EBS gp3, GCE PD, Azure Managed Disk hỗ trợ online expansion (không cần downtime)

---

## StatefulSet và volumeClaimTemplates

StatefulSet dùng `volumeClaimTemplates` để tạo PVC riêng cho từng replica, đảm bảo mỗi Pod có storage độc lập:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
spec:
  serviceName: postgres-headless
  replicas: 3

  template:
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:           # template PVC cho mỗi replica
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 20Gi
```

Kết quả — 3 PVC được tạo tự động:
```
data-postgres-cluster-0   → PV riêng cho pod-0
data-postgres-cluster-1   → PV riêng cho pod-1
data-postgres-cluster-2   → PV riêng cho pod-2
```

**Quan trọng:** Khi StatefulSet bị xoá, PVC **không** bị xoá — phải xoá thủ công. Đây là hành vi có chủ đích để bảo vệ dữ liệu.

```bash
# Xoá StatefulSet (PVC vẫn còn)
kubectl delete statefulset postgres-cluster

# Xoá thủ công PVC nếu chắc chắn muốn xoá
kubectl delete pvc -l app=postgres-cluster
```

---

## Câu Hỏi Phỏng Vấn

**PV và PVC có quan hệ 1-1 không? Một PV có thể bind nhiều PVC?**

> Quan hệ là **1-1** — một PV chỉ bind với một PVC tại một thời điểm. Khi PVC bị xoá, PV chuyển sang Released và không thể bind với PVC mới ngay (trừ khi Admin xoá và tạo lại PV hoặc PV có policy Recycle). Nếu cần nhiều Pod/PVC chia sẻ một storage, dùng NFS hoặc CephFS với access mode ReadWriteMany.

**Khi nào dùng `storageClassName: ""` (chuỗi rỗng)?**

> `storageClassName: ""` yêu cầu PVC binding với PV **không có StorageClass** (PV được tạo thủ công với `storageClassName: ""`). Điều này tắt dynamic provisioning hoàn toàn và buộc Kubernetes phải tìm PV đã có sẵn. Khác với không khai báo `storageClassName` — trường hợp đó Kubernetes dùng default StorageClass.

**Tại sao volumeClaimTemplates trong StatefulSet không bị xoá cùng StatefulSet?**

> Kubernetes cố ý bảo vệ dữ liệu bằng cách không cascade-delete PVC khi StatefulSet bị xoá. Database data thường có giá trị cao hơn StatefulSet object — mất PVC vô tình có thể gây thảm hoạ không thể phục hồi. Developer phải chủ động xoá PVC nếu thực sự muốn. Đây là nguyên tắc **safe by default** (an toàn theo mặc định) trong thiết kế Kubernetes.

**WaitForFirstConsumer giải quyết vấn đề gì?**

> Với cloud disk (EBS, GCE PD), volume phải nằm trong **cùng Availability Zone** (AZ — Vùng Khả Dụng) với node chạy Pod. Nếu dùng `Immediate`, StorageClass tạo PV (và EBS) ngay khi PVC được tạo — trước khi biết Pod sẽ chạy trên node nào, ở AZ nào. Kết quả: EBS ở AZ-a nhưng node ở AZ-b → mount fail. `WaitForFirstConsumer` đợi Pod được schedule, biết node/AZ, rồi mới tạo EBS ở đúng AZ. Đây là lý do AWS khuyến nghị bắt buộc dùng `WaitForFirstConsumer` cho EBS trong cluster multi-AZ.
