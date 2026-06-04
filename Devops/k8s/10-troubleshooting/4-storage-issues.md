# Storage Issues — Xử Lý Sự Cố Lưu Trữ Kubernetes

> Hướng dẫn chẩn đoán và khắc phục các sự cố lưu trữ: PVC Pending, lỗi mount volume, disk đầy, và các vấn đề với StorageClass.

## Mục Lục

1. [Kiến Trúc Lưu Trữ Kubernetes — Tổng Quan](#kiến-trúc-lưu-trữ-kubernetes)
2. [PVC Pending (PVC Đang Chờ)](#pvc-pending)
3. [Volume Mount Error (Lỗi Gắn Volume)](#volume-mount-error)
4. [Disk Full (Đĩa Đầy)](#disk-full)
5. [ReadWriteMany (RWX) Issues](#readwritemany-issues)
6. [StatefulSet Storage Issues](#statefulset-storage-issues)
7. [Debug Storage Nâng Cao](#debug-storage-nâng-cao)

---

## Kiến Trúc Lưu Trữ Kubernetes

### Các Khái Niệm Cốt Lõi

```
PVC (PersistentVolumeClaim — Yêu Cầu Lưu Trữ Bền Vững)
  ↕ bind (gắn kết)
PV (PersistentVolume — Volume Lưu Trữ Bền Vững)
  ↕ provisioned by
StorageClass (Lớp Lưu Trữ) — định nghĩa loại storage và cách tạo
  ↕ uses
CSI Driver (Container Storage Interface Driver — Driver Giao Diện Lưu Trữ Container)
  ↕ connects to
Backend Storage (EBS, GCS, NFS, Ceph, local disk...)
```

### Vòng Đời PVC — PV

```
PVC được tạo
  → Kubernetes tìm PV phù hợp (access mode, size, storageClass)
  → Nếu tìm thấy: PVC bound với PV
  → Nếu không tìm thấy PV sẵn: Provisioner tạo PV mới (dynamic provisioning)
  → Pod dùng PVC để mount volume

PVC bị xóa
  → Phụ thuộc reclaimPolicy:
    - Retain: PV và dữ liệu được giữ lại (cần xóa thủ công)
    - Delete: PV và backend storage bị xóa tự động
    - Recycle: Dữ liệu bị xóa, PV có thể tái sử dụng (deprecated)
```

### Xem Trạng Thái Storage

```bash
# Xem PVC
kubectl get pvc -n <namespace>
# NAME       STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS
# data-pvc   Bound    pvc-abc123       10Gi       RWO            gp2
# log-pvc    Pending                                             fast-ssd   ← Có vấn đề

# Xem PV
kubectl get pv
# NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
# pvc-abc123   10Gi       RWO            Delete           Bound    default/data-pvc

# Xem StorageClass
kubectl get storageclass
```

---

## PVC Pending

### Định Nghĩa

PVC ở trạng thái `Pending` nghĩa là chưa được bind với PV nào — Pod dùng PVC này cũng sẽ ở trạng thái `Pending`.

### Nguyên Nhân và Cách Xử Lý

#### 1. StorageClass Không Tồn Tại

```bash
# Kiểm tra storageClass trong PVC
kubectl get pvc <name> -n <ns> -o jsonpath='{.spec.storageClassName}'

# Kiểm tra storageClass có tồn tại không
kubectl get storageclass

# Nếu không có → tạo StorageClass hoặc sửa tên trong PVC
```

**Lỗi thường thấy:** `no persistent volumes available for this claim and no storage class is set`

#### 2. Provisioner (Bộ Cấp Phát) Lỗi Hoặc Không Chạy

```bash
# Kiểm tra provisioner pod (tùy thuộc vào CSI driver đang dùng)
kubectl get pods -n kube-system | grep -E "ebs|gce|azure|nfs|ceph"

# Xem log provisioner
kubectl logs -n kube-system <provisioner-pod> | tail -30

# Xem events của PVC
kubectl describe pvc <name> -n <ns>
# Thường có: "waiting for a volume to be created, either by external provisioner..."
```

#### 3. Access Mode Không Được StorageClass Hỗ Trợ

| StorageClass   | RWO | ROX | RWX |
| -------------- | --- | --- | --- |
| AWS EBS (gp2)  | ✅   | ❌   | ❌   |
| GCE PD         | ✅   | ❌   | ❌   |
| Azure Disk     | ✅   | ❌   | ❌   |
| NFS            | ✅   | ✅   | ✅   |
| EFS (AWS)      | ✅   | ✅   | ✅   |
| Ceph RBD       | ✅   | ❌   | ❌   |
| CephFS         | ✅   | ✅   | ✅   |

```
RWO = ReadWriteOnce   — Chỉ 1 node có thể mount read/write
ROX = ReadOnlyMany    — Nhiều node có thể mount read-only
RWX = ReadWriteMany   — Nhiều node có thể mount read/write
```

#### 4. Storage Quota Hết

```bash
# Kiểm tra ResourceQuota trong namespace
kubectl describe resourcequota -n <namespace>
# Xem: persistentvolumeclaims, requests.storage

# Xem tổng storage đang dùng
kubectl get pvc -n <ns> -o custom-columns='NAME:.metadata.name,SIZE:.spec.resources.requests.storage'
```

#### 5. Không Có PV Phù Hợp (Static Provisioning)

Nếu dùng static provisioning (tạo PV thủ công), PV phải có:
- `storageClassName` khớp với PVC
- `accessModes` bao gồm mode PVC yêu cầu
- `capacity.storage` >= size PVC yêu cầu
- `status: Available` (chưa bị bind)

```bash
# Kiểm tra PV sẵn có
kubectl get pv | grep Available

# Xem chi tiết PV
kubectl describe pv <pv-name>
```

#### 6. Node Affinity Không Phù Hợp (Local Storage)

Với local StorageClass (dùng local disk trên node), PV được bind với node cụ thể — Pod phải được schedule trên node đó.

```bash
kubectl describe pv <pv-name> | grep -A10 "nodeAffinity"
kubectl describe pod <pod-name> | grep "Node:"  # Pod đang ở node nào
```

### Quy Trình Debug PVC Pending

```bash
# Bước 1: Xem Events của PVC
kubectl describe pvc <name> -n <ns>

# Bước 2: Kiểm tra StorageClass
kubectl get storageclass <name>
kubectl describe storageclass <name>

# Bước 3: Kiểm tra provisioner
kubectl get pods -n kube-system | grep provisioner
kubectl logs -n kube-system <provisioner-pod> | tail -20

# Bước 4: Kiểm tra quota
kubectl describe resourcequota -n <ns>

# Bước 5: Xem PV có sẵn (static provisioning)
kubectl get pv -o wide | grep Available
```

---

## Volume Mount Error

### Triệu Chứng

```bash
# Pod bị lỗi vì không mount được volume
kubectl describe pod <name> -n <ns>
# Events:
#   Warning  FailedMount  4m  kubelet  Unable to attach or mount volumes:
#             unmounted volumes=[data], ... error syncing pod
```

### Nguyên Nhân và Cách Xử Lý

#### 1. Volume Đang Được Mount Bởi Node Khác (Multi-Attach Error)

```
Multi-Attach error for volume "pvc-abc123"
Volume is already exclusively attached to one node and can't be attached to another
```

**Nguyên nhân:** EBS/Azure Disk chỉ hỗ trợ `ReadWriteOnce` — chỉ 1 node được attach cùng lúc. Nếu Pod cũ chưa unmount mà Pod mới đã cố mount, xảy ra conflict.

**Cách xử lý:**
```bash
# Tìm Pod cũ vẫn đang giữ volume
kubectl get pods -A -o wide | grep <pvc-name>

# Nếu Pod cũ bị stuck, force delete nó
kubectl delete pod <old-pod> -n <ns> --grace-period=0 --force

# Đợi node detach volume (có thể mất 6–7 phút)
# Theo dõi events của Pod mới
kubectl get events -n <ns> --watch | grep <pod-name>
```

#### 2. Secret hoặc ConfigMap Mount Không Tồn Tại

```bash
kubectl describe pod <name> | grep -A5 "FailedMount"
# Error: secret "my-secret" not found

kubectl get secret <name> -n <ns>
kubectl get configmap <name> -n <ns>
```

#### 3. NFS Mount Timeout

```bash
# Kiểm tra NFS server có accessible không
kubectl run nfs-test --rm -it --image=busybox -- \
  timeout 5 sh -c "ls /mnt/nfs" 2>&1

# Kiểm tra NFS server endpoint
kubectl describe pv <pv-name> | grep "NFS\|server"

# Kiểm tra network từ node đến NFS server
ssh <node-ip>
showmount -e <nfs-server-ip>
mount -t nfs <nfs-server>:/<path> /mnt/test
```

#### 4. Permission Denied Khi Mount

```bash
# Xem security context của Pod
kubectl get pod <name> -o jsonpath='{.spec.securityContext}'

# fsGroup — group owner khi volume được mount
# Pod có thể cần:
spec:
  securityContext:
    fsGroup: 1000    # GID sở hữu volume mount point
    runAsUser: 1000
```

#### 5. CSI Driver Không Chạy

```bash
# Kiểm tra CSI driver pods
kubectl get pods -n kube-system | grep csi

# Xem log của CSI node plugin
kubectl logs -n kube-system <csi-node-pod> -c csi-driver | tail -30

# Xem log của CSI controller
kubectl logs -n kube-system <csi-controller-pod> -c csi-driver | tail -30
```

---

## Disk Full

### Triệu Chứng

```bash
# Pod ghi vào volume thất bại
# Error: no space left on device

# Node báo DiskPressure
kubectl get nodes
# NAME    STATUS                     ROLES    AGE
# node-1  Ready,DiskPressure         <none>   30d
```

### Nguồn Chiếm Đĩa Phổ Biến

#### 1. Container Logs Tích Lũy

```bash
# Trên node — xem log container chiếm bao nhiêu
du -sh /var/log/pods/*

# Cấu hình log rotation cho containerd
# /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd]
  max_container_log_line_size = 16384

# Hoặc cấu hình trong kubelet
# --container-log-max-size=50Mi --container-log-max-files=5
```

#### 2. Image Layers Cũ

```bash
# Kiểm tra trên node
df -h /var/lib/containerd

# Xóa image không dùng
crictl rmi --prune

# Xóa container đã dừng
crictl rm $(crictl ps -a -q --state exited)
```

#### 3. emptyDir Volume Quá Lớn

emptyDir bị giới hạn bởi `sizeLimit` trong Pod spec hoặc `limits.ephemeral-storage` trong resource limits.

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 1Gi   # Giới hạn size

containers:
  - resources:
      limits:
        ephemeral-storage: "2Gi"    # Tổng ephemeral storage cho container
```

#### 4. Ứng Dụng Ghi File Không Kiểm Soát

```bash
# Xem file lớn trong PVC
kubectl exec -it <pod> -n <ns> -- df -h /data
kubectl exec -it <pod> -n <ns> -- du -sh /data/* | sort -rh | head -20

# Tìm file lớn
kubectl exec -it <pod> -n <ns> -- find /data -size +100M -type f
```

### Xử Lý Disk Full Khẩn Cấp

```bash
# Bước 1: Giải phóng đĩa ngay lập tức
# Trên node:
docker system prune -f      # Nếu dùng Docker
crictl rmi --prune           # Nếu dùng containerd
journalctl --vacuum-size=500M  # Giới hạn systemd journal

# Bước 2: Mở rộng PVC (nếu StorageClass hỗ trợ expansion)
kubectl edit pvc <name> -n <ns>
# Tăng spec.resources.requests.storage

# Bước 3: Xem StorageClass có hỗ trợ resize không
kubectl get storageclass <name> -o jsonpath='{.allowVolumeExpansion}'
# Phải là "true"
```

---

## ReadWriteMany Issues

### Khi Nào Cần RWX

- Nhiều Pod cùng đọc/ghi vào một volume
- StatefulSet với nhiều replica truy cập chung storage
- Shared cache, shared upload directory

### Lựa Chọn RWX Storage

```
NFS (Network File System)     — Phổ biến, đơn giản, nhưng có latency
AWS EFS (Elastic File System) — Managed NFS trên AWS
Azure Files                   — Managed SMB/NFS trên Azure
CephFS                        — Phân tán, hiệu năng cao, phức tạp
GlusterFS                     — Tương tự CephFS
```

### Debug RWX Issues

```bash
# Kiểm tra PVC có đúng access mode không
kubectl get pvc <name> -n <ns> -o jsonpath='{.spec.accessModes}'

# Test nhiều Pod cùng mount
kubectl get pods -A -o wide | grep <pvc-name>

# Kiểm tra NFS mount trên các node
ssh <node-1>
mount | grep nfs

ssh <node-2>
mount | grep nfs
```

---

## StatefulSet Storage Issues

### Đặc Điểm StatefulSet Storage

- Mỗi Pod trong StatefulSet có PVC riêng (từ `volumeClaimTemplates`)
- PVC không bị xóa khi StatefulSet bị scale down
- PVC có tên dạng `<pvc-name>-<pod-name>` (ví dụ: `data-postgres-0`)

### Vấn Đề Phổ Biến

#### Pod StatefulSet Không Khởi Động Được — PVC Không Có

```bash
# Kiểm tra PVC theo tên pod
kubectl get pvc -n <ns> | grep <statefulset-name>

# PVC mất → Pod Pending
# Nguyên nhân: PVC bị xóa thủ công hoặc StorageClass thay đổi

# Cách xử lý: Tạo lại PVC với cùng tên
kubectl apply -f pvc-restore.yaml
```

#### Scale Down StatefulSet — PVC Vẫn Còn

```bash
# Xem PVC của StatefulSet sau khi scale down
kubectl get pvc -n <ns> | grep postgres
# postgres-data-postgres-0   Bound   (vẫn còn dù Pod đã xóa)
# postgres-data-postgres-1   Bound   (vẫn còn)

# Xóa thủ công nếu không cần
kubectl delete pvc postgres-data-postgres-1 -n <ns>
```

#### Thay Đổi Storage Size Của StatefulSet

```bash
# Không thể edit volumeClaimTemplates trực tiếp
# Cách 1: Edit từng PVC thủ công (nếu StorageClass hỗ trợ expansion)
kubectl edit pvc data-postgres-0 -n <ns>

# Cách 2: Recreate StatefulSet với size mới
kubectl delete statefulset <name> --cascade=orphan  # Xóa StatefulSet nhưng giữ Pod
kubectl apply -f statefulset-new-size.yaml           # Tạo lại với config mới
```

---

## Debug Storage Nâng Cao

### Xem Chi Tiết PV Binding

```bash
# Xem chi tiết binding giữa PVC và PV
kubectl get pvc <name> -n <ns> -o yaml | grep -A5 "volumeName\|storageClass"
kubectl get pv <pv-name> -o yaml | grep -A5 "claimRef\|storageClassName"
```

### Giải Phóng PV Bị Stuck Released

```bash
# PV ở trạng thái Released (PVC đã xóa nhưng PV chưa xóa)
kubectl get pv | grep Released

# Nếu reclaimPolicy là Retain, cần xóa thủ công
# Trước khi xóa PV, xóa claimRef để PV về Available
kubectl edit pv <pv-name>
# Xóa phần spec.claimRef

# Xóa PV
kubectl delete pv <pv-name>
```

### Kiểm Tra Volume Mount Trong Container

```bash
# Xem volume mount points
kubectl exec -it <pod> -n <ns> -- mount | grep /data
kubectl exec -it <pod> -n <ns> -- df -h /data

# Xem permissions
kubectl exec -it <pod> -n <ns> -- ls -la /data

# Kiểm tra read/write
kubectl exec -it <pod> -n <ns> -- sh -c "echo test > /data/test.txt && cat /data/test.txt"
```

### Volume Snapshot (Tạo Bản Chụp Volume)

```bash
# Kiểm tra VolumeSnapshot CRD có tồn tại không
kubectl get crd | grep volumesnapshot

# Tạo snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snapshot-20260510
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: data-pvc

# Kiểm tra snapshot
kubectl get volumesnapshot -n <ns>
kubectl describe volumesnapshot <name> -n <ns>
```

---

## Tóm Tắt Nhanh — Storage Issues

| Triệu Chứng                  | Lệnh Debug Đầu Tiên                            | Cách Xử Lý Thường                          |
| ---------------------------- | ----------------------------------------------- | ------------------------------------------ |
| PVC Pending                  | `kubectl describe pvc <name>`                   | Kiểm tra StorageClass, provisioner quota   |
| Multi-Attach Error           | `kubectl get pods -A \| grep <pvc>`             | Force delete Pod cũ, đợi detach            |
| Volume Mount Failed          | `kubectl describe pod <name>` → Events          | Kiểm tra Secret/CM tồn tại, CSI driver     |
| Disk Full trên Node          | `df -h` trên node                               | Dọn image cũ, log, tăng đĩa               |
| Disk Full trong Container    | `kubectl exec -- df -h /data`                   | Mở rộng PVC hoặc dọn dữ liệu              |
| PV stuck Released            | `kubectl get pv \| grep Released`               | Xóa claimRef trong PV spec                 |
| StatefulSet Pod Pending      | `kubectl get pvc -n <ns>`                       | Kiểm tra PVC còn tồn tại và bound          |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
