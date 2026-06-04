# Storage — Lưu Trữ Kubernetes

> Tổng quan về hệ thống lưu trữ Kubernetes: cách Pod gắn dữ liệu, PersistentVolume (PV — Lưu Trữ Ổn Định) cấp phát ra sao, StorageClass (lớp lưu trữ) tự động hoá quá trình đó, và CSI (Container Storage Interface — Giao Diện Lưu Trữ Container) kết nối với hạ tầng cloud.

## Mục Lục

1. [Mô Hình Lưu Trữ Kubernetes](#mô-hình-lưu-trữ-kubernetes)
2. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
3. [Các Thành Phần Storage](#các-thành-phần-storage)
4. [Luồng Cấp Phát Volume Điển Hình](#luồng-cấp-phát-volume-điển-hình)
5. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
6. [Checklist Thực Chiến](#checklist-thực-chiến)

---

## Mô Hình Lưu Trữ Kubernetes

Container có bản chất **ephemeral** (tạm thời) — khi container chết, toàn bộ dữ liệu bên trong biến mất. Kubernetes giải quyết điều này qua hệ thống lưu trữ phân tầng:

```
Ephemeral Storage (lưu trữ tạm thời)
  ├── emptyDir       — dữ liệu chia sẻ giữa container, mất khi Pod chết
  ├── configMap      — cấu hình từ ConfigMap, mount vào filesystem
  └── secret         — dữ liệu nhạy cảm từ Secret, mount vào filesystem

Persistent Storage (lưu trữ bền vững)
  ├── hostPath       — thư mục trên Node (không nên dùng production)
  ├── PV / PVC       — khai báo và yêu cầu lưu trữ bền vững
  └── CSI Volume     — plugin chuẩn kết nối với EBS, GCS, Azure Disk…
```

### Vòng Đời Dữ Liệu

| Loại Lưu Trữ | Tồn Tại Đến Khi | Dùng Khi Nào |
| ------------ | --------------- | ------------ |
| **emptyDir** | Pod bị xoá | Chia sẻ file giữa container trong Pod |
| **hostPath** | File trên node vẫn còn | Log agent, DaemonSet cần đọc node |
| **PV/PVC** | PV bị xoá thủ công (hoặc theo policy) | Database, file upload, stateful app |
| **CSI Volume** | Phụ thuộc provider | Cloud-native storage (EBS, GCS, Azure Disk) |

---

## Bản Đồ Quyết Định

```
Bạn cần loại lưu trữ nào?
│
├── Dữ liệu chỉ cần trong vòng đời Pod?
│   ├── Chia sẻ giữa nhiều container trong cùng Pod
│   │   └── emptyDir
│   └── Mount cấu hình / secret vào filesystem
│       └── configMap volume / secret volume
│           └── Xem: volumes.md
│
├── Dữ liệu phải tồn tại khi Pod bị restart / chuyển node?
│   ├── Cần cấp phát tự động (dynamic provisioning)?
│   │   └── PVC + StorageClass (tự động tạo PV)
│   │       └── Xem: persistent-volume.md
│   └── Cần kiểm soát thủ công (static provisioning)?
│       └── PV + PVC binding thủ công
│           └── Xem: persistent-volume.md
│
├── Cần biết quyền truy cập (một node / nhiều node)?
│   └── Chọn Access Mode phù hợp
│       └── ReadWriteOnce / ReadOnlyMany / ReadWriteMany
│           └── Xem: access-modes.md
│
├── Đang chạy trên cloud và muốn dùng storage native?
│   └── Cài CSI Driver (EBS, GCS, Azure Disk, NFS…)
│       └── Xem: csi-drivers.md
│
└── Cần backup hoặc snapshot volume cho disaster recovery?
    └── VolumeSnapshot + backup strategy
        └── Xem: volume-backup.md
```

---

## Các Thành Phần Storage

### Volume — Gắn Dữ Liệu Vào Pod

**Volume** trong Kubernetes là một thư mục gắn vào một hoặc nhiều container bên trong Pod. Volume được khai báo trong `spec.volumes` và gắn vào container qua `spec.containers[].volumeMounts`.

```yaml
spec:
  volumes:
    - name: shared-data       # tên volume (tham chiếu từ container)
      emptyDir: {}            # loại volume: emptyDir — rỗng khi Pod khởi động
  containers:
    - name: app
      volumeMounts:
        - name: shared-data
          mountPath: /data    # đường dẫn trong container
```

**Các loại Volume phổ biến:**
- `emptyDir` — thư mục rỗng, chia sẻ giữa container, mất khi Pod chết
- `hostPath` — mount thư mục từ Node vào Pod
- `configMap` — mount ConfigMap dưới dạng file
- `secret` — mount Secret dưới dạng file (được giải mã base64)
- `persistentVolumeClaim` — mount PV thông qua PVC

Tham khảo chi tiết: [volumes.md](./volumes.md)

---

### PersistentVolume (PV) và PersistentVolumeClaim (PVC)

**PV (PersistentVolume — Lưu Trữ Ổn Định)** là tài nguyên cluster-level đại diện cho một đơn vị lưu trữ vật lý (NFS server, EBS volume, GCE Persistent Disk…). Admin tạo PV hoặc StorageClass tự động tạo.

**PVC (PersistentVolumeClaim — Yêu Cầu Lưu Trữ Ổn Định)** là yêu cầu lưu trữ từ phía ứng dụng. PVC khai báo dung lượng, access mode cần thiết — Kubernetes tìm PV phù hợp để **bind** (ràng buộc).

```
Developer khai báo PVC           Kubernetes bind PV ↔ PVC
────────────────────────         ─────────────────────────
PVC: cần 10Gi, ReadWriteOnce  →  PV: 20Gi, ReadWriteOnce, EBS volume
```

**StorageClass (lớp lưu trữ)** tự động hoá việc tạo PV khi PVC yêu cầu — gọi là **Dynamic Provisioning** (cấp phát động). Thay vì Admin phải tạo PV trước, StorageClass sẽ gọi CSI Driver để tạo volume trên cloud.

```
PVC tạo mới  →  StorageClass nhận request  →  CSI Driver tạo EBS Volume  →  PV tạo tự động  →  Bind PVC
```

Tham khảo chi tiết: [persistent-volume.md](./persistent-volume.md)

---

### Access Modes — Quyền Truy Cập Volume

| Mode | Viết Tắt | Ý Nghĩa |
| ---- | -------- | ------- |
| **ReadWriteOnce** | RWO | Chỉ một node được đọc/ghi cùng lúc |
| **ReadOnlyMany** | ROX | Nhiều node đọc cùng lúc, không ai ghi |
| **ReadWriteMany** | RWX | Nhiều node đọc và ghi cùng lúc |
| **ReadWriteOncePod** | RWOP | Chỉ một Pod được đọc/ghi (từ K8s 1.22+) |

> Không phải mọi storage backend đều hỗ trợ tất cả access mode. EBS chỉ hỗ trợ RWO; NFS hỗ trợ cả RWO, ROX, RWX.

Tham khảo chi tiết: [access-modes.md](./access-modes.md)

---

### CSI Driver — Cầu Nối Kubernetes Với Storage Backend

**CSI (Container Storage Interface — Giao Diện Lưu Trữ Container)** là tiêu chuẩn mở cho phép bên thứ ba viết plugin lưu trữ mà không cần thay đổi code Kubernetes core. Mọi cloud provider (AWS, GCP, Azure) và storage vendor (NetApp, Ceph, Portworx) đều viết CSI Driver riêng.

```
Kubernetes core  ──CSI API──►  CSI Driver  ──vendor API──►  EBS / GCS / Azure Disk
```

**CSI Driver phổ biến:**
- `ebs.csi.aws.com` — Amazon EBS (Elastic Block Store — Kho Lưu Trữ Khối Co Giãn)
- `pd.csi.storage.gke.io` — Google Persistent Disk
- `disk.csi.azure.com` — Azure Managed Disk
- `efs.csi.aws.com` — Amazon EFS (Elastic File System — Hệ Thống File Co Giãn) hỗ trợ RWX
- `nfs.csi.k8s.io` — NFS (Network File System — Hệ Thống File Mạng)

Tham khảo chi tiết: [csi-drivers.md](./csi-drivers.md)

---

### Volume Backup — Sao Lưu và Phục Hồi

**VolumeSnapshot (Snapshot Volume)** là tính năng Kubernetes cho phép chụp ảnh trạng thái PV tại một thời điểm, tương tự snapshot trên cloud.

```
PVC  →  VolumeSnapshot  →  lưu trên cloud storage  →  khôi phục PVC mới từ snapshot
```

**Công cụ backup phổ biến:**
- **Velero** — backup toàn bộ cluster (object + volume) ra S3/GCS/Azure Blob
- **Stash** — backup volume vào object storage, tích hợp K8s CRD
- **CSI Snapshot** — snapshot native qua API Kubernetes

Tham khảo chi tiết: [volume-backup.md](./volume-backup.md)

---

## Luồng Cấp Phát Volume Điển Hình

### Dynamic Provisioning (Cấp Phát Động)

```
1. Developer tạo PVC với StorageClass "gp3-encrypted"
2. PVC controller phát hiện PVC chưa bound
3. StorageClass "gp3-encrypted" khai báo provisioner "ebs.csi.aws.com"
4. CSI Driver (EBS) được gọi → tạo EBS volume 20Gi trên AWS
5. Kubernetes tạo PV tương ứng và bind với PVC
6. Pod mount PVC → container thấy filesystem tại /data
```

### Static Provisioning (Cấp Phát Tĩnh)

```
1. Admin tạo PV thủ công (ví dụ: EBS volume id đã có sẵn)
2. Developer tạo PVC với spec phù hợp (dung lượng ≤ PV, access mode khớp)
3. Kubernetes controller tìm PV phù hợp và bind
4. Pod mount PVC → container thấy filesystem tại /data
```

### Pod Sử Dụng PVC

```
1. Pod spec khai báo volumes[].persistentVolumeClaim.claimName
2. Scheduler xem xét node có thể attach volume không (với RWO — chỉ 1 node)
3. kubelet trên node gọi CSI Driver để attach volume vào node
4. kubelet mount volume vào container
5. Container chạy — thấy filesystem bình thường tại mountPath
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu Hỏi Cơ Bản

**PV và PVC khác nhau thế nào?**

> - **PV** là tài nguyên cluster-level, đại diện cho storage vật lý thực sự (EBS, NFS…). Admin hoặc StorageClass tạo PV.
> - **PVC** là yêu cầu lưu trữ từ phía namespace/ứng dụng. Developer tạo PVC và Kubernetes tìm PV phù hợp để bind.
> - Quan hệ: PV là *cung*, PVC là *cầu*. Binding là khớp cung với cầu.

**StorageClass dynamic provisioning hoạt động ra sao?**

> Khi PVC được tạo với `storageClassName`, controller phát hiện PVC chưa bound và gọi provisioner được khai báo trong StorageClass. Provisioner (thường là CSI Driver) tạo volume thật trên hạ tầng (EBS, GCS…), sau đó Kubernetes tự động tạo PV tương ứng và bind với PVC. Developer không cần biết chi tiết hạ tầng.

**emptyDir biến mất khi nào?**

> emptyDir bị xoá khi **Pod bị xoá khỏi node** — tức là khi Pod bị terminated, evicted, hoặc node bị drain. emptyDir **không** mất khi container bên trong Pod bị restart (do crash hoặc liveness probe). Dùng emptyDir cho cache tạm thời, buffer, hoặc chia sẻ file giữa container trong cùng Pod.

### Câu Hỏi Nâng Cao

**Khi nào nên dùng ReadWriteMany (RWX)?**

> RWX cần thiết khi nhiều Pod trên nhiều node cùng đọc/ghi một volume — ví dụ: shared file storage cho ứng dụng web truyền thống, log aggregation, shared assets. Tuy nhiên RWX thường đòi hỏi NFS hoặc distributed storage (EFS, CephFS, Portworx) — đắt hơn và phức tạp hơn EBS (RWO). Nếu ứng dụng có thể dùng object storage (S3), hãy ưu tiên object storage thay vì RWX volume.

**Tại sao StatefulSet cần PVC template thay vì dùng chung một PVC?**

> Mỗi replica trong StatefulSet là một instance riêng biệt có định danh ổn định (`pod-0`, `pod-1`…). Nếu dùng chung PVC, tất cả replica ghi vào cùng một volume — vi phạm nguyên tắc phân vùng dữ liệu của database cluster (MySQL Cluster, PostgreSQL HA, Kafka). `volumeClaimTemplates` tạo PVC riêng cho từng replica (`data-pod-0`, `data-pod-1`…) đảm bảo tách biệt dữ liệu.

**VolumeSnapshot giúp gì trong disaster recovery (phục hồi thảm hoạ)?**

> VolumeSnapshot chụp trạng thái PV tại một thời điểm và lưu trên storage backend (EBS snapshot, GCS snapshot…). Khi có sự cố, bạn tạo PVC mới từ snapshot — Kubernetes tạo PV mới có dữ liệu tại thời điểm chụp. Kết hợp với công cụ như Velero (backup cả object K8s lẫn volume data), đây là nền tảng của disaster recovery strategy cho stateful workload.

---

## Checklist Thực Chiến

### Thiết Lập Storage Cơ Bản

- [ ] Xác định workload có cần persistent storage không (stateless vs stateful)
- [ ] Chọn StorageClass phù hợp với cloud provider đang dùng
- [ ] Đặt `reclaimPolicy: Retain` cho PV production (tránh mất dữ liệu khi PVC bị xoá)
- [ ] Kiểm tra access mode phù hợp với số replica ứng dụng
- [ ] Test mount volume bằng `kubectl exec` vào Pod và ghi thử file

### Thiết Lập StatefulSet Với Storage

- [ ] Dùng `volumeClaimTemplates` thay vì PVC cố định trong StatefulSet
- [ ] Đặt `storageClassName` tường minh — không dùng default ngầm ở production
- [ ] Cấu hình `resources.requests.storage` đủ lớn và có kế hoạch resize
- [ ] Cấu hình backup định kỳ với VolumeSnapshot hoặc Velero
- [ ] Test failover: xoá Pod, xác nhận Pod mới mount lại đúng PVC

### Bảo Mật Storage

- [ ] Dùng StorageClass mã hoá (EBS gp3 encrypted, GCS CMEK)
- [ ] Giới hạn StorageClass được phép dùng qua ResourceQuota
- [ ] Không mount hostPath vào container production (rủi ro bảo mật cao)
- [ ] Đặt `readOnly: true` cho volume chỉ cần đọc (configMap, secret)
- [ ] Cấu hình `fsGroup` trong securityContext để kiểm soát ownership

### Tối Ưu Hiệu Năng

- [ ] Chọn loại disk phù hợp (gp3 cho general, io2 cho IOPS cao, sc1 cho cold storage)
- [ ] Monitor disk usage và IOPS với Prometheus + cAdvisor
- [ ] Cân nhắc local PV cho workload cực kỳ nhạy cảm với latency (ví dụ: etcd, database OLTP)
- [ ] Đặt `VolumeExpansion: true` trong StorageClass để resize online khi cần
- [ ] Tránh dùng NFS cho database write-heavy (latency cao, locking phức tạp)

---

**Tài Liệu Liên Quan:**

| File | Nội Dung |
| ---- | -------- |
| [volumes.md](./1-volumes.md) | emptyDir, hostPath, configMap/secret volume chi tiết |
| [persistent-volume.md](./2-persistent-volume.md) | PV, PVC, StorageClass, dynamic provisioning |
| [access-modes.md](./3-access-modes.md) | ReadWriteOnce, ReadOnlyMany, ReadWriteMany |
| [csi-drivers.md](./4-csi-drivers.md) | CSI Driver, so sánh theo cloud provider |
| [volume-backup.md](./5-volume-backup.md) | VolumeSnapshot, Velero, chiến lược backup |
