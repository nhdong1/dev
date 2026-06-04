# Kiến Trúc Kubernetes — Tổng Quan

> Kubernetes (K8s) là hệ thống điều phối container mã nguồn mở, tự động hoá việc triển khai, mở rộng và quản lý ứng dụng container hoá.

## Mục Lục

1. [Tổng Quan Kiến Trúc](#tổng-quan-kiến-trúc)
2. [Control Plane (Mặt Điều Khiển)](#control-plane-mặt-điều-khiển)
3. [Worker Node (Nút Xử Lý)](#worker-node-nút-xử-lý)
4. [Kubernetes Objects (Tài Nguyên)](#kubernetes-objects-tài-nguyên)
5. [Request Flow (Luồng Xử Lý)](#request-flow-luồng-xử-lý)
6. [Các Tài Liệu Chi Tiết](#các-tài-liệu-chi-tiết)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## Tổng Quan Kiến Trúc

Kubernetes theo mô hình **Master–Worker** (Chủ–Tớ), gồm hai thành phần chính:

```
┌─────────────────────────────────────────────────────────────────┐
│                       KUBERNETES CLUSTER                        │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                 CONTROL PLANE (Mặt Điều Khiển)           │   │
│  │                                                          │   │
│  │  ┌─────────────┐  ┌──────┐  ┌───────────┐  ┌────────┐  │   │
│  │  │  API Server │  │ etcd │  │ Scheduler │  │  Ctrl  │  │   │
│  │  │ (Cổng vào)  │  │ (DB) │  │ (Lên lịch)│  │Manager │  │   │
│  │  └──────┬──────┘  └──────┘  └───────────┘  └────────┘  │   │
│  └─────────┼────────────────────────────────────────────────┘   │
│            │ (giao tiếp qua API)                                │
│  ┌─────────┼──────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  WORKER NODE 1  │  │  WORKER NODE 2  │  │  WORKER NODE 3  │  │
│  │  ┌────────────┐ │  │  ┌────────────┐ │  │  ┌────────────┐ │  │
│  │  │   kubelet  │ │  │  │   kubelet  │ │  │  │   kubelet  │ │  │
│  │  │ kube-proxy │ │  │  │ kube-proxy │ │  │  │ kube-proxy │ │  │
│  │  │ [Pod][Pod] │ │  │  │ [Pod][Pod] │ │  │  │ [Pod][Pod] │ │  │
│  │  └────────────┘ │  │  └────────────┘ │  │  └────────────┘ │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Vai Trò Từng Thành Phần

| Thành Phần             | Vai Trò                                                                |
| ---------------------- | ---------------------------------------------------------------------- |
| **API Server**         | Cổng vào duy nhất — nhận mọi request từ kubectl, UI, CI/CD             |
| **etcd**               | Cơ sở dữ liệu phân tán — lưu toàn bộ trạng thái cluster               |
| **Scheduler**          | Quyết định Pod chạy trên node nào                                      |
| **Controller Manager** | Vòng lặp điều khiển — đảm bảo trạng thái thực khớp trạng thái mong muốn |
| **kubelet**            | Agent trên mỗi node — thực thi lệnh từ Control Plane                   |
| **kube-proxy**         | Quản lý networking và load balancing trên node                         |
| **Container Runtime**  | Thực thi container (containerd, CRI-O)                                 |

---

## Control Plane (Mặt Điều Khiển)

Control Plane là "bộ não" của cluster, chịu trách nhiệm đưa ra mọi quyết định toàn cục.

### API Server — Cổng Vào Duy Nhất

- Là thành phần **duy nhất** giao tiếp trực tiếp với etcd
- Xác thực (Authentication — Xác Thực) và phân quyền (Authorization — Uỷ Quyền) mọi request
- Cung cấp RESTful API và hỗ trợ **watch** (theo dõi thay đổi theo thời gian thực)
- Là điểm vào của kubectl, Helm, ArgoCD, và mọi controller

**Luồng xử lý request của API Server:**
```
Client Request
   → Authentication (Xác thực: certificate, token, webhook)
   → Authorization (Uỷ quyền: RBAC, ABAC, webhook)
   → Admission Control (Kiểm soát đầu vào: Mutating → Validating)
   → Ghi vào etcd
   → Trả response
```

### etcd — Nguồn Sự Thật Duy Nhất

- Cơ sở dữ liệu key-value phân tán, dùng thuật toán **Raft** để đồng thuận
- Lưu **toàn bộ trạng thái cluster**: Pod, Deployment, ConfigMap, Secret...
- **Chỉ** API Server được phép đọc/ghi etcd trực tiếp
- Khuyến nghị: chạy **3 hoặc 5 replica** etcd để đảm bảo high availability (tính sẵn sàng cao)

```
etcd lưu dữ liệu tại: /registry/<resource-type>/<namespace>/<name>
Ví dụ: /registry/pods/default/nginx-pod
```

### Scheduler — Bộ Lên Lịch

Scheduler quyết định **Pod chạy trên node nào** qua hai bước:

1. **Filtering (Lọc):** Loại bỏ các node không đủ điều kiện
   - Không đủ CPU/memory
   - Taint không khớp toleration
   - NodeSelector hoặc NodeAffinity không thoả mãn
   - Volume không mount được trên node đó

2. **Scoring (Tính điểm):** Chấm điểm các node còn lại và chọn node cao điểm nhất
   - Ưu tiên node ít tải hơn
   - Ưu tiên phân tán Pod ra nhiều node (Pod Anti-Affinity)
   - Ưu tiên locality với dữ liệu

### Controller Manager — Vòng Lặp Điều Khiển

Chạy nhiều controller con trong một tiến trình duy nhất:

| Controller               | Chức Năng                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| **ReplicaSet Controller** | Đảm bảo số lượng Pod đúng như khai báo trong spec                    |
| **Deployment Controller** | Quản lý rolling update và rollback                                    |
| **Node Controller**       | Theo dõi trạng thái node, đánh dấu NotReady khi node mất kết nối    |
| **Job Controller**        | Theo dõi Job, tạo lại Pod nếu thất bại                               |
| **Endpoints Controller**  | Cập nhật danh sách IP Pod cho Service                                 |
| **Namespace Controller**  | Xử lý việc xoá namespace và tài nguyên bên trong                     |

**Mô hình Reconciliation Loop (Vòng lặp điều hoà):**
```
Lặp liên tục:
  1. Đọc Desired State (Trạng thái mong muốn) từ etcd
  2. Đọc Current State (Trạng thái hiện tại) từ cluster
  3. Nếu Desired ≠ Current → thực hiện hành động để đưa về Desired
  4. Quay lại bước 1
```

---

## Worker Node (Nút Xử Lý)

Mỗi Worker Node chạy ba thành phần chính:

### kubelet — Agent Trên Node

- Nhận PodSpec từ API Server và đảm bảo container chạy theo đúng spec đó
- Giao tiếp với Container Runtime qua **CRI (Container Runtime Interface — Giao Diện Thời Gian Chạy Container)**
- Báo cáo trạng thái node và Pod về API Server
- Thực hiện health check (Liveness / Readiness Probe)

### kube-proxy — Quản Lý Mạng

- Duy trì các quy tắc mạng trên node (thường dùng **iptables** hoặc **IPVS**)
- Thực hiện load balancing cho Service
- Không tham gia vào routing Pod-to-Pod trực tiếp (đó là việc của CNI plugin)

### Container Runtime (Thời Gian Chạy Container)

Phần mềm thực sự chạy container, giao tiếp với kubelet qua CRI:

| Runtime       | Ghi Chú                                                         |
| ------------- | --------------------------------------------------------------- |
| **containerd** | Mặc định của Kubernetes từ v1.24+, nhẹ và hiệu quả            |
| **CRI-O**     | Nhẹ hơn, được thiết kế chuyên dụng cho Kubernetes              |
| **Docker**    | Đã bị bỏ hỗ trợ trực tiếp từ K8s v1.24 (dùng containerd bên trong) |

---

## Kubernetes Objects (Tài Nguyên)

Mọi thứ trong Kubernetes đều là **Object** (tài nguyên) được lưu trong etcd. Mỗi object có:

```yaml
apiVersion: apps/v1          # Phiên bản API group
kind: Deployment              # Loại tài nguyên
metadata:
  name: nginx                 # Tên tài nguyên
  namespace: default          # Namespace chứa tài nguyên
spec:                         # Desired State — trạng thái mong muốn
  replicas: 3
status:                       # Current State — trạng thái thực tế (do K8s cập nhật)
  availableReplicas: 3
```

### Phân Loại Object Cơ Bản

| Loại              | Object                                          | Mục Đích                             |
| ----------------- | ----------------------------------------------- | ------------------------------------ |
| **Workload**      | Pod, Deployment, StatefulSet, DaemonSet, Job    | Chạy ứng dụng                        |
| **Networking**    | Service, Ingress, NetworkPolicy                 | Kết nối và bảo vệ lưu lượng          |
| **Storage**       | PersistentVolume, PersistentVolumeClaim, StorageClass | Quản lý dữ liệu bền vững        |
| **Config**        | ConfigMap, Secret                               | Cấu hình và bí mật                   |
| **Security**      | ServiceAccount, Role, ClusterRole, RoleBinding  | Xác thực và phân quyền               |
| **Namespace**     | Namespace                                       | Phân vùng tài nguyên trong cluster   |

---

## Request Flow (Luồng Xử Lý)

Khi chạy `kubectl apply -f deployment.yaml`:

```
1. kubectl đọc file YAML → gửi HTTP request đến API Server
2. API Server xác thực (TLS, token) và phân quyền (RBAC)
3. Admission Webhook kiểm tra / biến đổi manifest
4. API Server lưu Deployment object vào etcd
5. Deployment Controller phát hiện Deployment mới → tạo ReplicaSet
6. ReplicaSet Controller phát hiện thiếu Pod → tạo Pod object trong etcd
7. Scheduler phát hiện Pod chưa có node → tính toán → gán node
8. kubelet trên node được chọn phát hiện Pod mới được gán → kéo image → khởi động container
9. kubelet cập nhật trạng thái Pod lên API Server
10. Endpoints Controller cập nhật Service endpoints
```

Xem chi tiết tại [request-flow.md](./request-flow.md).

---

## Các Tài Liệu Chi Tiết

| File                                            | Nội Dung                                               |
| ----------------------------------------------- | ------------------------------------------------------ |
| [control-plane.md](./1-control-plane.md)          | API Server, etcd, Scheduler, Controller Manager        |
| [worker-node.md](./2-worker-node.md)              | kubelet, kube-proxy, Container Runtime                 |
| [kubernetes-objects.md](./3-kubernetes-objects.md) | Pod, Deployment, Service, Namespace và các object khác |
| [request-flow.md](./4-request-flow.md)            | Luồng từ `kubectl apply` đến Pod chạy                  |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Mô tả kiến trúc Kubernetes

**Gợi ý trả lời:** Kubernetes gồm Control Plane và Worker Node. Control Plane gồm API Server (cổng vào duy nhất), etcd (lưu trạng thái), Scheduler (lên lịch Pod), Controller Manager (vòng lặp điều khiển). Worker Node gồm kubelet (thực thi Pod), kube-proxy (mạng), và Container Runtime (chạy container).

### Câu 2: etcd là gì và tại sao quan trọng?

**Gợi ý trả lời:** etcd là cơ sở dữ liệu phân tán key-value dùng thuật toán Raft, là nguồn sự thật duy nhất của cluster. Mọi trạng thái cluster (Pod, Deployment, ConfigMap...) đều lưu ở đây. Mất etcd = mất toàn bộ trạng thái cluster. Cần backup etcd định kỳ và chạy ít nhất 3 replica.

### Câu 3: Scheduler quyết định đặt Pod lên node nào?

**Gợi ý trả lời:** Scheduler dùng hai bước: Filtering (lọc node không đủ tài nguyên, không thoả mãn NodeSelector/Affinity/Taint) và Scoring (chấm điểm dựa trên tài nguyên còn trống, Affinity, Topology Spread). Pod được gán lên node có điểm cao nhất.

### Câu 4: Reconciliation Loop là gì?

**Gợi ý trả lời:** Đây là mô hình cốt lõi của Kubernetes. Controller liên tục so sánh trạng thái mong muốn (Desired State trong etcd) với trạng thái thực tế (Current State của cluster) và thực hiện hành động để đưa về trạng thái mong muốn. Đây là nền tảng của tính tự hồi phục (self-healing) của K8s.

### Câu 5: API Server làm gì khi nhận một request?

**Gợi ý trả lời:** API Server xử lý theo thứ tự: (1) Authentication — xác thực danh tính (TLS cert, Bearer token, webhook); (2) Authorization — kiểm tra quyền (RBAC); (3) Admission Control — Mutating Webhook biến đổi object, Validating Webhook kiểm tra hợp lệ; (4) Serialize & lưu vào etcd; (5) Trả response cho client.

---

**Xem tiếp:** [control-plane.md](./control-plane.md) để đi sâu vào từng thành phần Control Plane.
