# Worker Node — Nút Xử Lý Kubernetes

> Worker Node (nút xử lý) là các máy chủ (vật lý hoặc ảo) thực sự chạy các ứng dụng của bạn dưới dạng container bên trong Pod.

## Mục Lục

1. [Tổng Quan Worker Node](#tổng-quan-worker-node)
2. [kubelet — Agent Của Node](#kubelet--agent-của-node)
3. [kube-proxy — Quản Lý Mạng Node](#kube-proxy--quản-lý-mạng-node)
4. [Container Runtime — Thời Gian Chạy Container](#container-runtime--thời-gian-chạy-container)
5. [CRI — Container Runtime Interface](#cri--container-runtime-interface)
6. [CNI — Container Network Interface](#cni--container-network-interface)
7. [CSI — Container Storage Interface](#csi--container-storage-interface)
8. [Node Lifecycle — Vòng Đời Node](#node-lifecycle--vòng-đời-node)
9. [Resource Management — Quản Lý Tài Nguyên](#resource-management--quản-lý-tài-nguyên)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Worker Node

```
┌──────────────────────────────────────────────────────────────────┐
│                         WORKER NODE                              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                       kubelet                             │   │
│  │   (Giao tiếp với API Server, quản lý Pod trên node này)  │   │
│  └────────────────────────────┬─────────────────────────────┘   │
│                               │ CRI (gRPC)                      │
│  ┌────────────────────────────▼─────────────────────────────┐   │
│  │              Container Runtime (containerd)               │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │   Pod A                     Pod B                   │ │   │
│  │  │  ┌─────────────────┐       ┌─────────────────────┐  │ │   │
│  │  │  │ container nginx │       │ container app       │  │ │   │
│  │  │  │ container proxy │       │ container sidecar   │  │ │   │
│  │  │  └────────┬────────┘       └──────────┬──────────┘  │ │   │
│  │  └───────────┼────────────────────────────┼────────────┘ │   │
│  └──────────────┼────────────────────────────┼──────────────┘   │
│                 │ Network (CNI Plugin)        │                  │
│  ┌──────────────▼────────────────────────────▼──────────────┐   │
│  │                     kube-proxy                            │   │
│  │         (iptables / IPVS rules cho Service routing)       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  OS: Linux (kernel 4.19+), Windows (thử nghiệm)                 │
│  Resources: CPU, Memory, Storage, Network                        │
└──────────────────────────────────────────────────────────────────┘
```

### Thành Phần Cốt Lõi

| Thành Phần           | Tiến Trình             | Vai Trò                                        |
| -------------------- | ---------------------- | ---------------------------------------------- |
| **kubelet**          | `kubelet`              | Quản lý Pod, báo cáo trạng thái, health check  |
| **kube-proxy**       | `kube-proxy`           | Routing Service, load balancing                |
| **Container Runtime** | `containerd` / `CRI-O` | Kéo image, tạo và chạy container              |
| **CNI Plugin**       | `calico` / `flannel`   | Cấp IP cho Pod, routing giữa Pod              |

---

## kubelet — Agent Của Node

### Vai Trò

kubelet là agent chạy trên **mỗi node** (bao gồm cả Control Plane node nếu cấu hình như vậy), chịu trách nhiệm:

- Nhận PodSpec (đặc tả Pod) từ API Server và đảm bảo container chạy theo đúng spec
- Giao tiếp với Container Runtime qua CRI để tạo/xoá container
- Thực hiện Health Probe (kiểm tra sức khoẻ) — Liveness, Readiness, Startup
- Báo cáo trạng thái node và Pod về API Server định kỳ (heartbeat)
- Quản lý volume — mount/unmount storage cho Pod
- Cấp phát và thu hồi tài nguyên CPU/memory theo Resource Request và Limit

### Cách kubelet Nhận PodSpec

kubelet nhận thông tin Pod cần chạy từ nhiều nguồn:

```
1. API Server (chính) — watch API Server để nhận Pod được Scheduler gán vào node
2. Static Pod — đọc file YAML từ /etc/kubernetes/manifests/ (dùng cho Control Plane component)
3. HTTP endpoint (ít dùng)
```

**Static Pod** là cách các thành phần Control Plane tự bootstrap:
```bash
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml
# etcd.yaml
```

### Vòng Đời Pod Trên kubelet

```
API Server gán Pod vào node
       │
       ▼
kubelet nhận PodSpec qua watch
       │
       ▼
Kiểm tra và chuẩn bị:
  - Pull image (kéo image) nếu chưa có
  - Cấp phát resource (CPU, memory)
  - Tạo volumes (mount ConfigMap, Secret, PVC)
  - Cấu hình network (gọi CNI plugin)
       │
       ▼
Tạo container qua Container Runtime (CRI):
  1. Tạo sandbox (pause container — container khởi tạo namespace)
  2. Chạy init container theo thứ tự
  3. Chạy các container chính song song
       │
       ▼
Bắt đầu Health Probe:
  - Startup Probe (nếu có): chờ ứng dụng khởi động
  - Liveness Probe: khởi động lại container nếu fail
  - Readiness Probe: điều chỉnh endpoint của Service
       │
       ▼
Báo cáo trạng thái về API Server (heartbeat mỗi 10 giây)
```

### Cấu Hình kubelet Quan Trọng

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Tần suất đồng bộ trạng thái
syncFrequency: 1m
fileCheckFrequency: 20s
httpCheckFrequency: 20s

# Quản lý tài nguyên
evictionHard:
  memory.available: "200Mi"        # Evict Pod khi RAM còn dưới 200Mi
  nodefs.available: "10%"          # Evict khi disk còn dưới 10%
  nodefs.inodesFree: "5%"

# Reserved resources (tài nguyên dự trữ cho hệ thống)
systemReserved:
  cpu: "500m"
  memory: "512Mi"
kubeReserved:
  cpu: "500m"
  memory: "256Mi"

# Image garbage collection (thu gom image cũ)
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
```

### Node Condition (Trạng Thái Node)

kubelet cập nhật các Condition của node:

| Condition            | True Khi                              | False / Unknown Khi          |
| -------------------- | ------------------------------------- | ----------------------------- |
| **Ready**            | Node healthy, kubelet hoạt động bình thường | kubelet crash, network fail |
| **MemoryPressure**   | RAM còn ít (dưới ngưỡng eviction)     | RAM đủ                        |
| **DiskPressure**     | Disk còn ít                           | Disk đủ                       |
| **PIDPressure**      | Số tiến trình gần đến giới hạn hệ thống | Bình thường                  |
| **NetworkUnavailable** | CNI chưa cấu hình xong               | Network OK                   |

---

## kube-proxy — Quản Lý Mạng Node

### Vai Trò

kube-proxy **không** routing traffic trực tiếp giữa Pod với Pod (đó là việc của CNI plugin). Thay vào đó, kube-proxy:

- Duy trì bảng quy tắc mạng (iptables hoặc IPVS) cho **Service**
- Khi Pod gửi traffic đến ClusterIP của một Service, kube-proxy đảm bảo traffic được forward đến Pod backend đúng
- Thực hiện load balancing giữa các Pod của một Service

### Chế Độ Hoạt Động

#### Chế Độ iptables (Mặc Định)

```
Client Pod gửi packet đến ClusterIP:Port
       │
       ▼
iptables PREROUTING chain
       │
       ▼  (DNAT — Destination NAT)
kube-proxy đã cài sẵn rule: ClusterIP:Port → random chọn một Pod IP:Port
       │
       ▼
Packet đến Pod backend
```

iptables dùng **random selection** — không phải round-robin thật sự. Có thể không đều khi số Pod ít.

```bash
# Xem iptables rules do kube-proxy tạo
iptables -t nat -L KUBE-SERVICES -n --line-numbers
iptables -t nat -L KUBE-SVC-<hash> -n   # Chain cho từng Service
```

#### Chế Độ IPVS (IP Virtual Server — Máy Chủ IP Ảo)

IPVS dùng kernel module riêng, hiệu năng tốt hơn iptables với nhiều Service:

- Hỗ trợ nhiều thuật toán load balancing: rr (round-robin), lc (least connection), sh (source hash)
- O(1) lookup thay vì O(n) của iptables
- Khuyến nghị cho cluster > 1000 Service

```bash
# Bật IPVS mode
kubectl edit configmap kube-proxy -n kube-system
# Thay mode: "" thành mode: "ipvs"

# Xem IPVS rules
ipvsadm -ln
```

#### So Sánh iptables vs IPVS

| Tiêu Chí          | iptables         | IPVS                          |
| ----------------- | ---------------- | ----------------------------- |
| **Hiệu năng**     | O(n) — chậm khi nhiều rule | O(1) — ổn định với mọi quy mô |
| **Load Balancing** | Random selection | Round-robin, least connection, source hash |
| **Độ phức tạp**   | Đơn giản, mặc định | Cần kernel module ipvs        |
| **Khuyến nghị**   | Cluster nhỏ      | Production cluster lớn        |

### Cập Nhật Endpoint

Khi Pod mới Ready hoặc Pod bị xoá:
```
Endpoints Controller cập nhật Endpoints object trong etcd
   → API Server thông báo cho kube-proxy trên mọi node
   → kube-proxy cập nhật iptables/IPVS rules
   → Traffic được route đến Pod mới / không còn route đến Pod bị xoá
```

Độ trễ cập nhật thường là 1–3 giây, đây là lý do cần **Graceful Shutdown** và **preStop hook**.

---

## Container Runtime — Thời Gian Chạy Container

### Tổng Quan

Container Runtime là phần mềm thực sự tạo và chạy container. Kubernetes định nghĩa CRI (Container Runtime Interface) để tách biệt với runtime cụ thể.

### containerd

**containerd** là container runtime mặc định của Kubernetes kể từ v1.24:

```
kubelet
  │ CRI (gRPC)
  ▼
containerd                   ← High-level runtime: quản lý image, snapshot
  │ OCI (Open Container Initiative)
  ▼
runc                         ← Low-level runtime: thực sự tạo container (namespace, cgroup)
  │
  ▼
Linux Kernel (namespaces + cgroups)
```

**Kiến trúc containerd:**
```
containerd
├── snapshotter      ← Quản lý layer của container image (overlayfs)
├── content store    ← Lưu trữ blob của image
├── metadata store   ← Mapping tên image, container ID
├── task service     ← Quản lý lifecycle container (start, stop, kill)
└── shim             ← Tách biệt containerd và runc, cho phép hot-reload
```

```bash
# Debug containerd trực tiếp (bypass kubelet)
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
crictl images
crictl logs <container-id>
crictl exec -it <container-id> sh
```

### CRI-O

CRI-O là runtime được thiết kế **chỉ** cho Kubernetes (dùng trong OpenShift):

- Nhẹ hơn containerd, không có CLI riêng
- Tuân thủ OCI spec 100%
- Tự động garbage collect image cũ

### So Sánh Runtime

| Tiêu Chí         | containerd          | CRI-O               |
| ---------------- | ------------------- | ------------------- |
| **Nguồn gốc**    | Docker Engine (tách ra) | Được tạo riêng cho K8s |
| **Độ phổ biến**  | Rất cao (EKS, GKE)  | Phổ biến trong OpenShift |
| **CLI debug**    | `crictl`, `nerdctl` | `crictl`, `podman`  |
| **Tính năng**    | Phong phú, CRI + standalone | Tối giản, chỉ K8s  |
| **Khuyến nghị**  | Default cho hầu hết cluster | OpenShift / RHEL    |

---

## CRI — Container Runtime Interface

### Giao Tiếp kubelet–Runtime

CRI là gRPC API định nghĩa contract giữa kubelet và container runtime:

```protobuf
// Hai service chính trong CRI

service RuntimeService {
  // Pod sandbox management
  rpc RunPodSandbox(RunPodSandboxRequest)     → RunPodSandboxResponse
  rpc StopPodSandbox(StopPodSandboxRequest)   → StopPodSandboxResponse
  rpc RemovePodSandbox(RemovePodSandboxRequest)

  // Container lifecycle
  rpc CreateContainer(CreateContainerRequest) → CreateContainerResponse
  rpc StartContainer(StartContainerRequest)
  rpc StopContainer(StopContainerRequest)
  rpc RemoveContainer(RemoveContainerRequest)

  // Exec / logs
  rpc Exec(ExecRequest)                       → ExecResponse
  rpc Logs(...)
}

service ImageService {
  rpc PullImage(PullImageRequest)             → PullImageResponse
  rpc RemoveImage(RemoveImageRequest)
  rpc ListImages(ListImagesRequest)           → ListImagesResponse
}
```

### Pause Container (Infra Container)

Mỗi Pod có một **pause container** (còn gọi là sandbox container) được kubelet tạo trước:

```bash
# Xem pause container (thường ẩn)
docker ps | grep pause
# gcr.io/google_containers/pause:3.9

# Pause container chia sẻ với các container trong Pod:
# - Network namespace (cùng IP, cùng port space)
# - IPC namespace (shared memory)
# - PID namespace (tuỳ cấu hình)
```

Pause container **không làm gì** — chỉ giữ namespace sống, cho phép các container trong Pod khởi động lại mà không mất network namespace.

---

## CNI — Container Network Interface

### Cách CNI Hoạt Động

Khi kubelet tạo Pod, nó gọi CNI plugin để cấp phát network:

```
kubelet tạo Pod sandbox
  │
  ▼
Gọi CNI plugin: /opt/cni/bin/<plugin>
  │
  ▼
CNI plugin:
  1. Tạo veth pair (virtual ethernet pair — cặp card mạng ảo)
  2. Một đầu đặt trong network namespace của Pod → eth0
  3. Đầu kia đặt trên host → vethXXXXXXX
  4. Cấp phát IP cho Pod từ subnet của node
  5. Cập nhật routing table để traffic reach Pod
  6. Trả IP về kubelet → kubelet lưu vào PodSpec
```

### So Sánh CNI Plugin Phổ Biến

| CNI Plugin  | Mô Hình Mạng            | Network Policy | Hiệu Năng  | Use Case                      |
| ----------- | ----------------------- | -------------- | ---------- | ----------------------------- |
| **Calico**  | BGP routing hoặc overlay | Có (full)     | Cao        | Production, cần NetworkPolicy |
| **Flannel** | VXLAN overlay           | Không          | Trung bình | Lab, môi trường đơn giản      |
| **Cilium**  | eBPF kernel              | Có (L7 aware) | Rất cao    | Cloud-native, service mesh    |
| **Weave**   | Overlay                 | Có             | Trung bình | Multi-platform                |

**Lưu ý:** Flannel không hỗ trợ NetworkPolicy — cần dùng Calico hoặc Cilium nếu cần kiểm soát network.

---

## CSI — Container Storage Interface

### Cách CSI Hoạt Động

Tương tự CRI cho storage — CSI định nghĩa interface giữa Kubernetes và storage provider:

```
Kubernetes (PV Controller + kubelet)
  │ CSI (gRPC)
  ▼
CSI Driver (do vendor cung cấp: AWS EBS CSI, GCE PD CSI, Ceph CSI...)
  │
  ▼
Storage Backend (EBS, GCE PD, NFS, Ceph, NVMe...)
```

**Hai loại operation:**
1. **Controller Plugin:** Tạo/xoá volume, attach/detach vào node (chạy trên Control Plane)
2. **Node Plugin:** Mount/unmount volume vào container (chạy trên mỗi node)

---

## Node Lifecycle — Vòng Đời Node

### Trạng Thái Node

```
Node gia nhập cluster (join)
       │
       ▼
       Ready (sẵn sàng nhận Pod)
       │
       ├─── Drain (kubectl drain)
       │         └─── Evict Pod → SchedulingDisabled
       │
       ├─── Cordon (kubectl cordon)
       │         └─── SchedulingDisabled (Pod đang chạy không bị ảnh hưởng)
       │
       ├─── kubelet crash / mất kết nối
       │         └─── Unknown (sau 40s)
       │                  └─── NotReady (sau thêm 20s)
       │                           └─── Pod bị evict (sau pod-eviction-timeout: 5 phút)
       │
       └─── Node bị xoá khỏi cluster
```

### Taint và Toleration (Vết Bẩn và Dung Sai)

**Taint** đặt lên node để ngăn Pod không muốn chạy trên đó:

```bash
# Thêm taint vào node
kubectl taint nodes node1 dedicated=gpu:NoSchedule
# Effect: NoSchedule | PreferNoSchedule | NoExecute

# Xoá taint
kubectl taint nodes node1 dedicated=gpu:NoSchedule-
```

**Toleration** đặt trong Pod spec để chấp nhận taint:

```yaml
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
    - key: "node.kubernetes.io/not-ready"   # Hệ thống tự tạo khi node NotReady
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300                # Tối đa 5 phút rồi evict Pod
```

**Taint hệ thống mặc định:**

| Taint                                          | Khi Nào Được Tạo                     |
| ---------------------------------------------- | ------------------------------------ |
| `node.kubernetes.io/not-ready`                 | Node condition Ready = False         |
| `node.kubernetes.io/unreachable`               | Node condition Ready = Unknown       |
| `node.kubernetes.io/memory-pressure`           | Node có MemoryPressure               |
| `node.kubernetes.io/disk-pressure`             | Node có DiskPressure                 |
| `node.kubernetes.io/unschedulable`             | Node bị cordon                       |
| `node.kubernetes.io/network-unavailable`       | CNI chưa ready                       |

### Drain vs Cordon

```bash
# Cordon: ngăn Pod mới được schedule, Pod cũ KHÔNG bị ảnh hưởng
kubectl cordon node1

# Drain: cordon + evict tất cả Pod (trừ DaemonSet và static Pod)
kubectl drain node1 \
  --ignore-daemonsets \      # DaemonSet Pod không evict được (chạy lại ngay)
  --delete-emptydir-data \   # Xoá Pod dùng emptyDir volume
  --grace-period=60          # Cho ứng dụng 60s để shutdown gracefully

# Sau khi maintenance xong, uncordon để node nhận Pod trở lại
kubectl uncordon node1
```

---

## Resource Management — Quản Lý Tài Nguyên

### Request vs Limit

```yaml
spec:
  containers:
    - name: app
      resources:
        requests:           # Tài nguyên tối thiểu — Scheduler dùng để chọn node
          cpu: "250m"       # 250 millicores = 0.25 CPU core
          memory: "128Mi"   # 128 Mebibytes
        limits:             # Tài nguyên tối đa — enforcement tại runtime
          cpu: "1000m"      # 1 CPU core
          memory: "512Mi"   # 512 Mi
```

**CPU vs Memory — hành vi khác nhau khi vượt limit:**
- **CPU vượt limit:** CPU bị **throttle** (làm chậm) — container vẫn chạy nhưng chậm hơn
- **Memory vượt limit:** Container bị **OOMKilled** (Out of Memory Killed — Bị giết do hết bộ nhớ) và restart

### QoS Class (Lớp Chất Lượng Dịch Vụ)

Kubernetes phân Pod vào 3 lớp QoS dựa trên Request/Limit:

| QoS Class      | Điều Kiện                                        | Ưu Tiên Khi Thiếu Tài Nguyên |
| -------------- | ------------------------------------------------ | -----------------------------|
| **Guaranteed** | Request = Limit cho mọi container                | Cuối cùng bị evict           |
| **Burstable**  | Request < Limit, hoặc chỉ có Request             | Evict sau BestEffort         |
| **BestEffort** | Không có Request lẫn Limit                       | Bị evict đầu tiên            |

```bash
# Xem QoS class của Pod
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
```

**Khuyến nghị production:** Luôn đặt Resource Request cho mọi container. Không đặt quá cao hoặc quá thấp.

### Node Allocatable (Tài Nguyên Có Thể Phân Bổ)

```
Node Capacity (tổng tài nguyên node)
  - kube-reserved  (dành cho kubelet, container runtime)
  - system-reserved (dành cho OS processes)
  - eviction-threshold (ngưỡng khởi động eviction)
  ═══════════════════════════════
  = Node Allocatable (có thể dùng cho Pod)
```

```bash
# Xem tài nguyên allocatable
kubectl describe node <node-name> | grep -A 5 "Allocatable:"
```

---

## Câu Hỏi Phỏng Vấn

### Q1: kubelet vs kube-proxy — hai thành phần này khác nhau thế nào?

**Trả lời:** kubelet quản lý **Pod lifecycle** trên node — nhận PodSpec, tạo container, thực hiện health probe, báo cáo trạng thái. kube-proxy quản lý **Service networking** — duy trì iptables/IPVS rule để route traffic đến Pod backend. kubelet không biết về Service; kube-proxy không biết về Pod lifecycle.

### Q2: Container Runtime là gì? Sự khác biệt giữa containerd và Docker?

**Trả lời:** Container Runtime là phần mềm chạy container. Docker là platform đầy đủ (CLI + daemon + runtime), containerd là runtime được tách ra từ Docker và trở thành runtime mặc định của K8s từ v1.24. Kubernetes không dùng Docker daemon trực tiếp kể từ v1.24 — nhưng image Docker vẫn chạy được vì đều tuân chuẩn OCI.

### Q3: Static Pod là gì và tại sao Control Plane dùng nó?

**Trả lời:** Static Pod là Pod do kubelet quản lý trực tiếp, không qua API Server — định nghĩa bằng file YAML trong `/etc/kubernetes/manifests/`. Control Plane (API Server, etcd, Scheduler, Controller Manager) dùng Static Pod để tự bootstrap: kubelet khởi động trước API Server, đọc file manifest và tạo Control Plane container. Nếu API Server crash, kubelet vẫn tự khởi động lại nó mà không cần contact API Server.

### Q4: OOMKilled là gì và làm thế nào để phòng tránh?

**Trả lời:** OOMKilled (Out of Memory Killed) xảy ra khi container vượt memory limit — kernel Linux kills process. Phòng tránh: (1) Đặt memory limit hợp lý bằng cách profile ứng dụng trước; (2) Đặt limit = request cho workload quan trọng (QoS Guaranteed); (3) Bật JVM heap sizing đúng cho Java; (4) Cấu hình Liveness Probe thay vì để OOMKilled restart; (5) Dùng VPA để tự động điều chỉnh memory request.

### Q5: Drain node khác Cordon như thế nào? Khi nào dùng cái nào?

**Trả lời:** Cordon chỉ đánh dấu node `SchedulingDisabled` — Pod cũ không bị ảnh hưởng, Pod mới không được schedule lên. Drain = Cordon + evict tất cả Pod (trừ DaemonSet). Dùng Cordon khi cần dừng nhận Pod mới mà không muốn gián đoạn ngay (ví dụ: kiểm tra node). Dùng Drain khi cần bảo trì node thực sự — cần `PodDisruptionBudget` (PDB — Ngân Sách Gián Đoạn Pod) để đảm bảo không evict quá nhiều Pod một lúc.

---

**Xem tiếp:** [kubernetes-objects.md](./kubernetes-objects.md) để hiểu các tài nguyên cốt lõi của Kubernetes.
