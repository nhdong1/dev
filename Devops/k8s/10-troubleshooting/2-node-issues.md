# Node Issues — Xử Lý Sự Cố Node Kubernetes

> Hướng dẫn chẩn đoán và khắc phục các sự cố phổ biến trên Worker Node: NotReady, MemoryPressure, DiskPressure, taint/toleration, drain và cordon.

## Mục Lục

1. [Kiến Trúc Node Cơ Bản](#kiến-trúc-node-cơ-bản)
2. [Node NotReady](#node-notready)
3. [Node Pressure Conditions (Điều Kiện Áp Lực Node)](#node-pressure-conditions)
4. [Taint và Toleration](#taint-và-toleration)
5. [Cordon — Cách Ly Node Khỏi Lịch Phân Công Pod](#cordon)
6. [Drain — Chuyển Tất Cả Pod Khỏi Node](#drain)
7. [Node Maintenance (Bảo Trì Node)](#node-maintenance)
8. [Debug Node Nâng Cao](#debug-node-nâng-cao)

---

## Kiến Trúc Node Cơ Bản

### Các Thành Phần Trên Worker Node

```
Worker Node
├── kubelet          — Agent chạy trên mỗi node, nhận lệnh từ API Server
├── kube-proxy       — Quản lý iptables/ipvs rules cho Service networking
├── Container Runtime — containerd hoặc CRI-O để chạy container
└── Các Pod          — Workload thực sự chạy trên node
```

### Trạng Thái Node

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# node-1     Ready    control-plane   30d   v1.28.0
# node-2     Ready    <none>          30d   v1.28.0
# node-3     NotReady <none>          30d   v1.28.0   ← Có vấn đề
```

### Xem Chi Tiết Trạng Thái Node

```bash
kubectl describe node <node-name>

# Phần quan trọng cần chú ý:
# Conditions:    → Trạng thái sức khoẻ (Ready, MemoryPressure, DiskPressure...)
# Capacity:      → Tổng tài nguyên node có
# Allocatable:   → Tài nguyên có thể dùng cho Pod
# Allocated:     → Tài nguyên đã được Pod đặt chỗ (request)
# Events:        → Sự kiện gần đây trên node
```

---

## Node NotReady

### Định Nghĩa

Node ở trạng thái `NotReady` nghĩa là kubelet không giao tiếp được với API Server, hoặc node đang gặp vấn đề nghiêm trọng.

### Hậu Quả

- Pod mới không được lên lịch trên node này
- Sau khoảng 5 phút (mặc định), Pod trên node NotReady sẽ bị **evict** (di chuyển) sang node khác
- StatefulSet Pod không bị evict tự động (cần xử lý thủ công)

### Nguyên Nhân và Cách Xử Lý

#### 1. kubelet Không Chạy

```bash
# SSH vào node bị ảnh hưởng
ssh <node-ip>

# Kiểm tra trạng thái kubelet
systemctl status kubelet

# Xem log kubelet
journalctl -u kubelet -n 100 --no-pager

# Khởi động lại nếu bị dừng
systemctl restart kubelet
```

**Dấu hiệu log phổ biến:**
- `failed to run Kubelet: failed to create kubelet: misconfiguration`
- `Unable to connect to the server: x509 certificate has expired`
- `connection refused` khi kết nối API Server

#### 2. Hết Đĩa (DiskPressure)

```bash
# Trên node
df -h                              # Kiểm tra disk usage
du -sh /var/lib/docker/*           # Tìm container chiếm nhiều đĩa nhất
du -sh /var/log/*                  # Log có thể rất lớn

# Giải phóng đĩa
docker system prune -a             # Xóa container/image không dùng
crictl rmi --prune                 # Nếu dùng containerd
journalctl --vacuum-size=1G        # Giới hạn systemd log
```

#### 3. Hết Bộ Nhớ (MemoryPressure)

```bash
# Trên node
free -h                            # Xem memory usage
top                                # Xem process dùng nhiều RAM

# Trong K8s
kubectl top pods -A --sort-by=memory | head -20   # Pod dùng nhiều RAM nhất
```

#### 4. Network Node Bị Cô Lập

```bash
# Kiểm tra kết nối từ node đến API Server
curl -k https://<api-server-ip>:6443/healthz

# Kiểm tra DNS
nslookup kubernetes.default.svc.cluster.local

# Kiểm tra iptables/route
iptables -L -n | grep DROP
ip route show
```

#### 5. Certificate Hết Hạn

```bash
# Kiểm tra certificate kubelet
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout | grep "Not After"

# Nếu cert hết hạn, rotate:
kubeadm certs renew all   # Nếu dùng kubeadm
```

#### 6. Tài Nguyên Kernel Cạn Kiệt

```bash
# Kiểm tra số lượng file descriptor đang mở
cat /proc/sys/fs/file-nr

# Kiểm tra pid limit
cat /proc/sys/kernel/pid_max

# Kiểm tra inotify watches
cat /proc/sys/fs/inotify/max_user_watches
```

### Quy Trình Debug Node NotReady

```bash
# Bước 1: Xem conditions của node
kubectl describe node <name> | grep -A20 "Conditions:"

# Bước 2: Xem events của node
kubectl get events -A | grep <node-name>

# Bước 3: SSH vào node và kiểm tra kubelet
ssh <node-ip>
systemctl status kubelet
journalctl -u kubelet -f

# Bước 4: Kiểm tra resource
free -h && df -h

# Bước 5: Kiểm tra network
ping <api-server-ip>
curl -k https://<api-server-ip>:6443/healthz
```

---

## Node Pressure Conditions

### Các Loại Pressure (Áp Lực)

```bash
kubectl describe node <name> | grep -A5 "Conditions:"
# Conditions:
#   Type             Status  ...  Message
#   ----             ------       -------
#   MemoryPressure   False        kubelet has sufficient memory available
#   DiskPressure     False        kubelet has no disk pressure
#   PIDPressure      False        kubelet has sufficient PID available
#   Ready            True         kubelet is posting ready status
```

### MemoryPressure (Áp Lực Bộ Nhớ)

Xảy ra khi memory khả dụng < `evictionHard.memory.available` (mặc định 100Mi).

**Hậu quả:** kubelet sẽ evict Pod (ưu tiên BestEffort trước, rồi Burstable, cuối cùng Guaranteed).

```bash
# Xem QoS class của Pod
kubectl get pod <name> -o jsonpath='{.status.qosClass}'

# QoS classes:
# Guaranteed — requests == limits cho CPU và memory
# Burstable  — có request nhưng request < limit, hoặc chỉ set một phần
# BestEffort — không có request lẫn limit (dễ bị evict nhất)
```

### DiskPressure (Áp Lực Ổ Đĩa)

Xảy ra khi disk khả dụng < `evictionHard.nodefs.available` (mặc định 10% hoặc 5%).

**Nguyên nhân phổ biến:**
- Container log quá lớn
- Image không được dọn dẹp
- emptyDir volume quá lớn
- `/var/log` systemd log tích tụ

```bash
# Trên node — tìm nguồn chiếm đĩa
du -sh /var/lib/containerd/      # containerd image layers
du -sh /var/log/pods/            # container logs
du -sh /tmp/                     # temp files
```

### PIDPressure (Áp Lực PID)

Xảy ra khi số lượng process/thread > ngưỡng cho phép.

```bash
cat /proc/sys/kernel/pid_max
ps aux | wc -l
```

---

## Taint và Toleration

### Định Nghĩa

- **Taint** (Dấu Bẩn): Nhãn đặt trên Node để **ngăn** Pod được lên lịch trên đó, trừ khi Pod có Toleration phù hợp.
- **Toleration** (Sự Khoan Dung): Khai báo trong Pod để nói "Pod này chấp nhận Taint đó và vẫn có thể chạy trên node bị đánh dấu".

### Cú Pháp Taint

```bash
# Thêm taint
kubectl taint nodes <node-name> key=value:effect

# Effect có 3 loại:
# NoSchedule    — Pod mới không được lên lịch (Pod đang chạy không bị ảnh hưởng)
# PreferNoSchedule — Cố gắng không lên lịch (không đảm bảo)
# NoExecute     — Pod không được lên lịch VÀ Pod đang chạy bị evict

# Xóa taint (thêm dấu - ở cuối)
kubectl taint nodes <node-name> key=value:effect-

# Ví dụ thực tế
kubectl taint nodes node-1 dedicated=gpu:NoSchedule
kubectl taint nodes node-2 node-role=control-plane:NoSchedule  # K8s tự thêm cho control-plane
```

### Cú Pháp Toleration

```yaml
spec:
  tolerations:
    # Toleration chính xác
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"

    # Toleration wildcard — chấp nhận mọi taint có key này
    - key: "dedicated"
      operator: "Exists"
      effect: "NoSchedule"

    # Toleration tất cả (không giới hạn)
    - operator: "Exists"
```

### Use Cases (Trường Hợp Sử Dụng) Phổ Biến

```bash
# 1. Dành node GPU riêng cho AI workload
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule
# Pod AI phải có toleration:
#   - key: gpu, operator: Equal, value: "true", effect: NoSchedule

# 2. Node bảo trì — không nhận Pod mới
kubectl taint nodes node-1 maintenance=true:NoSchedule

# 3. Spot/Preemptible node — Pod có thể chấp nhận bị interrupt
kubectl taint nodes spot-node-1 cloud.google.com/gke-spot=true:NoSchedule
```

### Debug Taint/Toleration

```bash
# Xem taint trên tất cả node
kubectl describe nodes | grep Taints

# Xem taint của một node
kubectl get node <name> -o jsonpath='{.spec.taints}'

# Kiểm tra tại sao Pod Pending
kubectl describe pod <name> | grep -A5 "Events:"
# Thường thấy: "0/3 nodes are available: 3 node(s) had untolerated taint {key: value}"
```

---

## Cordon

### Định Nghĩa

`kubectl cordon` đánh dấu node là **Unschedulable** (không nhận Pod mới), nhưng Pod đang chạy **không bị ảnh hưởng**.

### Khi Nào Dùng

- Chuẩn bị bảo trì node nhưng muốn Pod hiện tại tiếp tục chạy
- Ngăn Cluster Autoscaler thêm Pod vào node sắp được xóa

### Cú Pháp

```bash
# Cordon — không nhận Pod mới
kubectl cordon <node-name>

# Uncordon — cho phép nhận Pod trở lại
kubectl uncordon <node-name>

# Kiểm tra trạng thái
kubectl get nodes
# NAME     STATUS                     ROLES   AGE
# node-1   Ready,SchedulingDisabled   <none>  30d   ← Đã cordon
```

---

## Drain

### Định Nghĩa

`kubectl drain` thực hiện 2 việc:
1. **Cordon** node (không nhận Pod mới)
2. **Evict** (di chuyển) tất cả Pod trên node sang node khác

### Khi Nào Dùng

- Trước khi bảo trì node (update kernel, hardware maintenance)
- Trước khi xóa node khỏi cluster
- Trước khi upgrade kubelet

### Cú Pháp

```bash
# Drain cơ bản
kubectl drain <node-name> --ignore-daemonsets

# Drain với các flag phổ biến
kubectl drain <node-name> \
  --ignore-daemonsets \          # Bỏ qua DaemonSet Pod (không thể evict)
  --delete-emptydir-data \       # Xóa Pod dùng emptyDir volume
  --grace-period=300 \           # Cho Pod 5 phút để graceful shutdown
  --timeout=600s                 # Timeout tổng cộng

# Drain pod cụ thể (nếu drain bị block)
kubectl delete pod <pod-name> -n <namespace> --grace-period=0
```

### Các Lý Do Drain Có Thể Fail

```bash
# 1. DaemonSet Pod — cần --ignore-daemonsets
# ERROR: cannot delete DaemonSet-managed Pods

# 2. Pod dùng emptyDir — cần --delete-emptydir-data
# ERROR: cannot delete Pods with local storage

# 3. Pod không có controller — cần --force
kubectl drain <node> --force --ignore-daemonsets
# CẢNH BÁO: Pod sẽ bị xóa vĩnh viễn, không được tạo lại trên node khác

# 4. PodDisruptionBudget (PDB — Ngân Sách Gián Đoạn Pod) không cho phép
kubectl get pdb -A
kubectl describe pdb <pdb-name> -n <ns>
# Có thể cần tăng maxUnavailable tạm thời
```

### PodDisruptionBudget (PDB)

PDB giới hạn số Pod của một ứng dụng có thể bị gián đoạn cùng lúc, đảm bảo tính sẵn sàng.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2          # Luôn có ít nhất 2 Pod running
  # hoặc
  maxUnavailable: 1        # Tối đa 1 Pod có thể gián đoạn
  selector:
    matchLabels:
      app: my-app
```

### Quy Trình Bảo Trì Node An Toàn

```bash
# Bước 1: Cordon node (không nhận Pod mới)
kubectl cordon <node-name>

# Bước 2: Drain node (di chuyển Pod hiện tại)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Bước 3: Thực hiện bảo trì (update, restart, etc.)
ssh <node-ip>
# ... bảo trì ...

# Bước 4: Xác nhận node healthy sau bảo trì
kubectl get node <node-name>

# Bước 5: Uncordon — cho node nhận Pod trở lại
kubectl uncordon <node-name>
```

---

## Node Maintenance

### Quản Lý Lifecycle Node Trên Cloud

#### AWS EKS — Node Group Lifecycle

```bash
# Terminate instance an toàn (Cluster Autoscaler sẽ drain trước)
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id i-1234567890abcdef0 \
  --should-decrement-desired-capacity

# Hoặc dùng managed node group update
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup
```

#### GKE — Node Pool Lifecycle

```bash
# Upgrade node pool
gcloud container clusters upgrade my-cluster \
  --node-pool my-pool \
  --cluster-version 1.28.0-gke.100

# Resize node pool
gcloud container clusters resize my-cluster \
  --node-pool my-pool \
  --num-nodes 5
```

### Debug Node Nâng Cao

```bash
# Xem tất cả resource được allocate trên node
kubectl describe node <name> | grep -A20 "Non-terminated Pods:"

# Xem node conditions theo thời gian
kubectl get node <name> -o jsonpath='{.status.conditions}' | python3 -m json.tool

# Xem log systemd trên node
journalctl -u kubelet --since "1 hour ago" --no-pager | grep -i "error\|warn"

# Xem container runtime status
systemctl status containerd
crictl info

# Xem số lượng Pod trên mỗi node
kubectl get pods -A -o wide | awk '{print $8}' | sort | uniq -c | sort -rn
```

---

## Tóm Tắt Nhanh — Node Issues

| Triệu Chứng              | Lệnh Kiểm Tra Đầu Tiên                               | Cách Xử Lý Thường                        |
| ------------------------ | ---------------------------------------------------- | ----------------------------------------- |
| Node NotReady            | `kubectl describe node` → Conditions & Events        | Restart kubelet, kiểm tra disk/memory     |
| MemoryPressure           | `kubectl top nodes`, `free -h` trên node             | Tăng memory hoặc evict Pod               |
| DiskPressure             | `df -h`, `du -sh /var/lib/containerd`               | Xóa image/log cũ, tăng đĩa               |
| Pod không được schedule  | `kubectl describe pod` → Events → taint/toleration  | Sửa toleration hoặc bỏ taint             |
| Drain bị block           | `kubectl get pdb -A`, `kubectl drain` → error msg   | Sửa PDB, dùng `--ignore-daemonsets`      |
| Node không uncordon      | `kubectl get node` → SchedulingDisabled              | `kubectl uncordon <node>`                 |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
