# Access Modes — Chế Độ Truy Cập Volume

> Giải thích chi tiết bốn Access Mode (chế độ truy cập) trong Kubernetes: ReadWriteOnce (RWO), ReadOnlyMany (ROX), ReadWriteMany (RWX), ReadWriteOncePod (RWOP) — cách chúng hoạt động, hạn chế theo storage backend, và khi nào chọn mode nào.

## Mục Lục

1. [Tổng Quan Access Mode](#tổng-quan-access-mode)
2. [ReadWriteOnce (RWO)](#readwriteonce-rwo)
3. [ReadOnlyMany (ROX)](#readonlymany-rox)
4. [ReadWriteMany (RWX)](#readwritemany-rwx)
5. [ReadWriteOncePod (RWOP)](#readwriteoncepod-rwop)
6. [Hỗ Trợ Theo Storage Backend](#hỗ-trợ-theo-storage-backend)
7. [Bản Đồ Quyết Định](#bản-đồ-quyết-định)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Access Mode

**Access Mode** (chế độ truy cập) xác định *số lượng node* được phép mount volume và *quyền đọc/ghi*. Access Mode được khai báo cả trong PV và PVC — phải **khớp** để binding thành công.

```
PV khai báo:  accessModes: [ReadWriteOnce, ReadOnlyMany]
PVC yêu cầu:  accessModes: [ReadWriteOnce]
→ Binding thành công (PVC yêu cầu là tập con của PV hỗ trợ)
```

| Mode | Viết Tắt | Nhiều Node | Ghi | Dùng Khi Nào |
| ---- | -------- | ---------- | --- | ------------ |
| **ReadWriteOnce** | RWO | ❌ Một node | ✅ | Database, block storage |
| **ReadOnlyMany** | ROX | ✅ Nhiều node | ❌ | Shared static assets |
| **ReadWriteMany** | RWX | ✅ Nhiều node | ✅ | Shared file storage |
| **ReadWriteOncePod** | RWOP | Một Pod | ✅ | Strict single-writer |

> **Quan trọng:** Access Mode kiểm soát quyền ở tầng *storage system* — không phải tầng *filesystem permission* trong container. Hai container trong cùng một Pod với volume RWO đều đọc/ghi được vì cùng chạy trên một node.

---

## ReadWriteOnce (RWO)

**ReadWriteOnce** cho phép volume được mount bởi **chỉ một node** tại một thời điểm — node đó có thể đọc và ghi.

```
Node A  ──► Volume ──► Read ✅  Write ✅
Node B  ──► Volume ──► BLOCKED ❌ (volume đang được dùng bởi Node A)
```

### Hành Vi Quan Trọng

```
Trong cùng Node A:
  Pod-1  → Volume RWO  → ✅ đọc/ghi
  Pod-2  → Volume RWO  → ✅ đọc/ghi (cùng node)
  Pod-3  → Volume RWO  → ✅ đọc/ghi (cùng node)

Trên Node B (khác node):
  Pod-4  → Volume RWO  → ❌ lỗi attach (volume đã mounted trên Node A)
```

RWO không giới hạn số Pod trên *cùng node* — chỉ giới hạn *node*. Nếu muốn giới hạn chặt chẽ hơn xuống mức Pod, dùng RWOP.

### Use Case

```
✅ Phù hợp:
- Database (PostgreSQL, MySQL, MongoDB) — chỉ cần 1 node ghi
- StatefulSet single-replica
- Block storage (EBS, GCE PD, Azure Disk) — đây là giới hạn phần cứng thật sự
- Ứng dụng stateful không hỗ trợ multi-writer

❌ Không phù hợp:
- Deployment nhiều replica ở nhiều node (dùng RWX)
- Shared static content (dùng ROX)
```

### Manifest

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-encrypted
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 1           # RWO chỉ phù hợp single replica (hoặc HA với primary/standby)
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

---

## ReadOnlyMany (ROX)

**ReadOnlyMany** cho phép volume được mount bởi **nhiều node cùng lúc** nhưng chỉ với quyền **đọc**.

```
Node A  ──► Volume ──► Read ✅   Write ❌
Node B  ──► Volume ──► Read ✅   Write ❌
Node C  ──► Volume ──► Read ✅   Write ❌
```

### Luồng Sử Dụng Điển Hình

```
1. Admin hoặc CI/CD pipeline ghi dữ liệu vào volume (static assets, ML model)
2. Volume được chuyển sang trạng thái read-only
3. Nhiều Pod trên nhiều node cùng đọc volume này
```

### Use Case

```
✅ Phù hợp:
- Phân phối static assets (HTML, CSS, JS, images) cho nhiều web server Pod
- Phân phối ML model weights cho nhiều inference Pod
- Shared configuration files không thay đổi thường xuyên
- Seed data (dữ liệu khởi tạo) cho nhiều Pod đọc

❌ Không phù hợp:
- Bất kỳ use case nào cần ghi từ ứng dụng
```

### Manifest

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-assets-pvc
spec:
  accessModes:
    - ReadOnlyMany
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 10     # 10 Pod trên nhiều node, tất cả đọc cùng volume
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: assets
              mountPath: /usr/share/nginx/html
              readOnly: true      # khai báo readOnly trong container cũng nên đặt
      volumes:
        - name: assets
          persistentVolumeClaim:
            claimName: static-assets-pvc
            readOnly: true
```

---

## ReadWriteMany (RWX)

**ReadWriteMany** cho phép volume được mount bởi **nhiều node cùng lúc** với quyền **đọc và ghi**.

```
Node A  ──► Volume ──► Read ✅   Write ✅
Node B  ──► Volume ──► Read ✅   Write ✅
Node C  ──► Volume ──► Read ✅   Write ✅
```

### Thách Thức Của RWX

RWX yêu cầu storage backend hỗ trợ **concurrent multi-writer** — điều mà block storage (EBS, GCE PD) **không thể làm**. Chỉ network file system hoặc distributed storage mới hỗ trợ:

| Storage Backend | Hỗ Trợ RWX | Ghi Chú |
| -------------- | ---------- | ------- |
| AWS EFS (Elastic File System) | ✅ | NFS-based, phù hợp general purpose |
| Azure Files | ✅ | SMB hoặc NFS protocol |
| GCS Filestore | ✅ | NFS-based |
| NFS thủ công | ✅ | Cần NFS server riêng |
| CephFS | ✅ | Distributed filesystem |
| Portworx | ✅ | Enterprise storage |
| AWS EBS | ❌ | Chỉ hỗ trợ RWO |
| GCE Persistent Disk | ❌ | Chỉ hỗ trợ RWO, ROX |
| Azure Managed Disk | ❌ | Chỉ hỗ trợ RWO |

### Vấn Đề File Locking Với RWX

RWX không tự động xử lý **file locking** (khoá file) — đây là trách nhiệm của ứng dụng. Nếu hai Pod cùng ghi vào file giống nhau mà không có locking, dữ liệu bị hỏng:

```
Pod-1 đọc file → sửa → ghi lại
Pod-2 đọc file → sửa → ghi lại   ← overwrite thay đổi của Pod-1
→ Race condition → data corruption (hỏng dữ liệu)
```

### Use Case

```
✅ Phù hợp:
- Shared upload directory cho ứng dụng web legacy (CMS, media server)
- Log aggregation từ nhiều Pod vào cùng thư mục
- Development tools cần shared workspace
- Machine learning distributed training với shared checkpoint

❌ Không phù hợp thay thế bằng:
- Object storage (S3, GCS Bucket) nếu ứng dụng có thể dùng HTTP/S3 API
- Database (dùng StatefulSet + RWO + replication thay vì shared volume)
- Message queue (Kafka, SQS) cho event streaming
```

### Manifest Với AWS EFS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0123456789abcdef
  directoryPerms: "700"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi    # EFS thực tế không giới hạn — con số này chỉ là request
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upload-service
spec:
  replicas: 5     # nhiều replica trên nhiều node, cùng đọc/ghi /uploads
  template:
    spec:
      containers:
        - name: app
          image: my-app:1.0
          volumeMounts:
            - name: uploads
              mountPath: /app/uploads
      volumes:
        - name: uploads
          persistentVolumeClaim:
            claimName: shared-uploads
```

---

## ReadWriteOncePod (RWOP)

**ReadWriteOncePod** (từ Kubernetes 1.22+) cho phép volume được mount bởi **chỉ một Pod duy nhất** trong toàn cluster — chặt chẽ hơn RWO (RWO chỉ giới hạn ở node).

```
Pod-1 (Node A) ──► Volume RWOP ──► ✅ đọc/ghi
Pod-2 (Node A) ──► Volume RWOP ──► ❌ BLOCKED (dù cùng node với Pod-1)
Pod-3 (Node B) ──► Volume RWOP ──► ❌ BLOCKED
```

### Khi Nào Cần RWOP

```
Tình huống: Database primary node cần đảm bảo tuyệt đối chỉ 1 Pod ghi
Problem với RWO: Pod-1 và Pod-2 cùng node A đều có thể mount RWO volume
                → split-brain (cả hai nghĩ mình là primary)
                → data corruption

Giải pháp: RWOP đảm bảo chỉ 1 Pod duy nhất trong cluster được mount
           → không thể có split-brain
```

### Manifest

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-primary-data
spec:
  accessModes:
    - ReadWriteOncePod    # yêu cầu K8s >= 1.22 và CSI Driver hỗ trợ
  storageClassName: gp3-encrypted
  resources:
    requests:
      storage: 50Gi
```

> Không phải mọi CSI Driver đều hỗ trợ RWOP — kiểm tra documentation của driver trước khi dùng.

---

## Hỗ Trợ Theo Storage Backend

| Storage Backend | RWO | ROX | RWX | RWOP |
| -------------- | --- | --- | --- | ---- |
| **AWS EBS** (gp2, gp3, io1) | ✅ | ❌ | ❌ | ✅ |
| **AWS EFS** | ✅ | ✅ | ✅ | ❌ |
| **GCE Persistent Disk** | ✅ | ✅ | ❌ | ✅ |
| **GCE Filestore (NFS)** | ✅ | ✅ | ✅ | ❌ |
| **Azure Managed Disk** | ✅ | ❌ | ❌ | ✅ |
| **Azure Files (SMB/NFS)** | ✅ | ✅ | ✅ | ❌ |
| **NFS** | ✅ | ✅ | ✅ | ❌ |
| **CephFS** | ✅ | ✅ | ✅ | ❌ |
| **Portworx** | ✅ | ✅ | ✅ | ✅ |
| **Local PV** | ✅ | ❌ | ❌ | ✅ |
| **HostPath** | ✅ | ❌ | ❌ | ❌ |

---

## Bản Đồ Quyết Định

```
Chọn Access Mode dựa trên yêu cầu:

Bao nhiêu Pod cần đọc/ghi volume cùng lúc?
│
├── Chỉ 1 Pod (hoặc nhiều Pod cùng node)
│   ├── Cần đảm bảo tuyệt đối 1 Pod duy nhất toàn cluster?
│   │   └── ReadWriteOncePod (RWOP) — K8s 1.22+
│   └── Chấp nhận nhiều Pod cùng node đều mount được?
│       └── ReadWriteOnce (RWO) — phổ biến nhất, EBS, Azure Disk
│
├── Nhiều Pod, nhiều node — chỉ đọc
│   └── ReadOnlyMany (ROX) — static assets, ML model serving
│
└── Nhiều Pod, nhiều node — đọc và ghi
    ├── Ứng dụng có thể dùng object storage (S3/GCS API)?
    │   └── Không cần PVC — dùng SDK gọi thẳng S3 ✅ (rẻ hơn, đơn giản hơn)
    └── Phải dùng shared filesystem (legacy app, POSIX API)?
        └── ReadWriteMany (RWX) — EFS, Azure Files, NFS, CephFS
```

---

## Câu Hỏi Phỏng Vấn

**Deployment 3 replica có thể dùng 1 PVC với RWO không?**

> Phụ thuộc: nếu tất cả 3 replica được schedule trên **cùng một node** → được (RWO cho phép nhiều Pod trên cùng node cùng mount). Nhưng trong thực tế production với nhiều node, Kubernetes scheduler thường phân tán replica ra nhiều node → 2 replica trên node khác sẽ không mount được volume RWO → Pod stuck ở `ContainerCreating`. **Giải pháp đúng:** dùng RWX nếu cần shared storage cho Deployment nhiều node, hoặc thiết kế lại để mỗi replica có PVC riêng (StatefulSet pattern).

**RWX có đảm bảo data consistency (tính nhất quán dữ liệu) không?**

> RWX chỉ đảm bảo nhiều node có thể **mount** volume — không đảm bảo gì về data consistency. Consistency phụ thuộc vào: (1) storage backend (NFS có POSIX locking, EFS có distributed lock), (2) ứng dụng có implement locking/coordination không. Database không nên dùng RWX shared volume — thay vào đó dùng database replication protocol (PostgreSQL streaming replication, MySQL Group Replication) trên các PVC RWO riêng biệt.

**Tại sao block storage (EBS, Azure Disk) không hỗ trợ RWX?**

> Block storage hoạt động theo mô hình exclusive access (truy cập độc quyền) ở tầng hardware — thiết bị block chỉ có thể được attach vào một máy chủ tại một thời điểm (do giao thức SCSI/NVMe). Để hỗ trợ nhiều node đọc/ghi đồng thời cần filesystem layer distributed (NFS, CIFS, CephFS) đứng giữa — đây là lý do EFS (NFS over network) hỗ trợ RWX còn EBS (block device) thì không.

**RWOP giải quyết vấn đề gì mà RWO không giải quyết được?**

> RWO ngăn hai *node* khác nhau cùng mount volume, nhưng không ngăn hai Pod trên cùng node. Trong kịch bản database primary failover, Kubernetes có thể tạm thời có 2 Pod cùng tồn tại trên cùng node (Pod cũ chưa terminate, Pod mới đã khởi động) — với RWO, cả hai Pod có thể mount volume và ghi đồng thời → split-brain. RWOP ràng buộc ở tầng API — chỉ 1 Pod duy nhất trong toàn cluster được mount → loại bỏ hoàn toàn split-brain scenario.
