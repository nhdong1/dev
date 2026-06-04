# Control Plane — Mặt Điều Khiển Kubernetes

> Control Plane là tập hợp các thành phần chịu trách nhiệm đưa ra quyết định toàn cục về cluster (lên lịch, phát hiện và phản hồi với các sự kiện) cũng như duy trì trạng thái mong muốn.

## Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [API Server](#api-server)
3. [etcd](#etcd)
4. [Scheduler](#scheduler)
5. [Controller Manager](#controller-manager)
6. [Cloud Controller Manager](#cloud-controller-manager)
7. [High Availability](#high-availability-tính-sẵn-sàng-cao)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

```
┌──────────────────────────────────────────────────────────────────┐
│                  CONTROL PLANE                                   │
│                                                                  │
│  kubectl / Helm / ArgoCD / CI-CD Pipeline                        │
│          │                                                       │
│          ▼                                                       │
│  ┌───────────────┐    ┌─────────────────────────────────────┐   │
│  │  API Server   │◄──►│                etcd                 │   │
│  │ (kube-apiserver)│   │  (Cluster State — Trạng Thái Cụm)  │   │
│  └───────┬───────┘    └─────────────────────────────────────┘   │
│          │                                                       │
│   ┌──────┴──────┐                                               │
│   │             │                                               │
│   ▼             ▼                                               │
│  ┌──────────┐  ┌────────────────────┐                          │
│  │Scheduler │  │ Controller Manager │                          │
│  │(kube-    │  │ (kube-controller-  │                          │
│  │scheduler)│  │  manager)          │                          │
│  └──────────┘  └────────────────────┘                          │
└──────────────────────────────────────────────────────────────────┘
```

**Nguyên tắc quan trọng:** Mọi thành phần trong Control Plane (trừ etcd) chỉ giao tiếp với nhau **thông qua API Server** — không thành phần nào gọi trực tiếp etcd ngoài API Server.

---

## API Server

### Vai Trò

`kube-apiserver` là thành phần trung tâm của Control Plane:

- **Cổng vào duy nhất** cho mọi thao tác: kubectl, Helm, operator, CI/CD
- **Xác thực và phân quyền** mọi request trước khi xử lý
- **Cơ chế watch** (theo dõi thay đổi): cho phép các component khác subscribe sự kiện thay đổi
- **Khả năng mở rộng** qua Admission Webhook và Aggregated API Server

### Pipeline Xử Lý Request

```
Request đến API Server
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  1. AUTHENTICATION (Xác Thực)                         │
│     - X.509 Client Certificate (chứng chỉ)           │
│     - Bearer Token (token xác thực)                   │
│     - OIDC Token (OpenID Connect)                     │
│     - Webhook Token Authentication                    │
└───────────────────────┬───────────────────────────────┘
                        │
        ▼
┌───────────────────────────────────────────────────────┐
│  2. AUTHORIZATION (Uỷ Quyền)                          │
│     - RBAC (Role-Based Access Control)                │
│     - ABAC (Attribute-Based Access Control)           │
│     - Node Authorization                              │
│     - Webhook Authorization                           │
└───────────────────────┬───────────────────────────────┘
                        │
        ▼
┌───────────────────────────────────────────────────────┐
│  3. ADMISSION CONTROL (Kiểm Soát Đầu Vào)             │
│     Bước Mutating (Biến Đổi):                         │
│       - DefaultStorageClass                           │
│       - MutatingAdmissionWebhook (tuỳ chỉnh)         │
│     Bước Validating (Kiểm Tra):                       │
│       - NamespaceLifecycle                            │
│       - ResourceQuota                                 │
│       - ValidatingAdmissionWebhook (tuỳ chỉnh)       │
└───────────────────────┬───────────────────────────────┘
                        │
        ▼
┌───────────────────────────────────────────────────────┐
│  4. PERSIST TO etcd (Lưu Vào etcd)                    │
│     Serialize object → lưu vào etcd                   │
└───────────────────────┬───────────────────────────────┘
                        │
        ▼
        Trả Response cho Client (201 Created / 200 OK)
```

### API Groups và Versioning (Nhóm API và Phiên Bản)

Kubernetes tổ chức API theo group và version:

```
/api/v1                          → Core group: Pod, Service, ConfigMap, Secret
/apis/apps/v1                    → apps group: Deployment, StatefulSet, DaemonSet
/apis/batch/v1                   → batch group: Job, CronJob
/apis/networking.k8s.io/v1       → networking group: Ingress, NetworkPolicy
/apis/rbac.authorization.k8s.io/v1 → RBAC group: Role, ClusterRole
```

Quy ước phiên bản:
- `v1alpha1` → đang thử nghiệm, có thể thay đổi bất kỳ lúc nào
- `v1beta1` → tương đối ổn định, sắp ra chính thức
- `v1` → ổn định, được đảm bảo tương thích ngược

### Cơ Chế Watch (Theo Dõi Thay Đổi)

Đây là cơ chế cho phép các thành phần khác phản ứng ngay khi có thay đổi trạng thái:

```bash
# Theo dõi Pod thay đổi real-time
kubectl get pods --watch

# Dưới hood: API Server duy trì kết nối HTTP long-polling hoặc WebSocket
# và đẩy event khi có thay đổi trong etcd
```

Event types (Loại sự kiện): `ADDED`, `MODIFIED`, `DELETED`

### Admission Controller Quan Trọng

| Admission Controller        | Chức Năng                                                            |
| --------------------------- | -------------------------------------------------------------------- |
| **NamespaceLifecycle**      | Ngăn tạo resource trong namespace đang bị xoá                       |
| **ResourceQuota**           | Kiểm tra quota tài nguyên của namespace                              |
| **LimitRanger**             | Áp dụng giá trị mặc định cho resource limit/request                 |
| **ServiceAccount**          | Tự động gắn ServiceAccount mặc định                                  |
| **DefaultStorageClass**     | Gắn StorageClass mặc định cho PVC không chỉ định StorageClass        |
| **MutatingAdmissionWebhook** | Webhook tuỳ chỉnh để biến đổi object trước khi lưu                |
| **ValidatingAdmissionWebhook** | Webhook tuỳ chỉnh để từ chối object không hợp lệ               |
| **PodSecurity**             | Kiểm tra Pod theo chính sách bảo mật (Privileged/Baseline/Restricted) |

---

## etcd

### Đặc Điểm

`etcd` là cơ sở dữ liệu phân tán key-value:

- Thuật toán đồng thuận **Raft**: đảm bảo tính nhất quán mạnh (strong consistency)
- **Chỉ ghi khi quorum** (đa số node đồng ý) — với 3 node thì cần 2 đồng ý, với 5 node cần 3
- Hỗ trợ **MVCC (Multi-Version Concurrency Control — Kiểm Soát Đồng Thời Đa Phiên Bản)**: mỗi thay đổi tạo ra một revision mới
- Mọi thay đổi trong etcd đều tạo ra **event** mà API Server có thể watch

### Cấu Trúc Dữ Liệu

```
/registry/
├── pods/
│   ├── default/
│   │   ├── nginx-pod-abc123    ← Dữ liệu Pod, serialize bằng protobuf
│   │   └── web-xyz456
│   └── kube-system/
│       └── coredns-def789
├── deployments/
│   └── default/
│       └── nginx-deployment
├── configmaps/
├── secrets/                    ← Mặc định lưu dạng base64, không mã hoá
└── services/
```

### Quorum và Fault Tolerance (Khả Năng Chịu Lỗi)

| Số Node etcd | Quorum Cần | Số Node Có Thể Mất |
| ------------ | ---------- | ------------------- |
| 1            | 1          | 0                   |
| 3            | 2          | 1                   |
| 5            | 3          | 2                   |
| 7            | 4          | 3                   |

**Khuyến nghị production:** Chạy **3 node etcd** (cân bằng giữa fault tolerance và hiệu năng).

### Backup và Phục Hồi

```bash
# Tạo snapshot etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Kiểm tra snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# Khôi phục từ snapshot (thực hiện trên mọi node etcd, dừng cluster trước)
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored
```

**Best practice:** Backup etcd mỗi giờ trên production, lưu ít nhất 24 giờ gần nhất.

### Encryption at Rest (Mã Hoá Dữ Liệu Lưu Trữ)

Mặc định Secret lưu trong etcd **không mã hoá**. Để bật mã hoá:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}   # fallback: đọc được các secret cũ chưa mã hoá
```

---

## Scheduler

### Mô Hình Quyết Định

`kube-scheduler` chọn node cho Pod qua hai giai đoạn:

#### Giai Đoạn 1: Filtering (Lọc) — Predicates

Loại bỏ node **không đủ điều kiện**:

```
Filter: NodeResourcesFit
  → Kiểm tra CPU, memory, storage đủ không?
  → Ví dụ: Pod yêu cầu 500m CPU, node chỉ còn 200m → loại

Filter: NodeSelector
  → Pod có nodeSelector: disktype=ssd → chỉ giữ node có label này

Filter: NodeAffinity
  → Kiểm tra requiredDuringSchedulingIgnoredDuringExecution

Filter: TaintToleration
  → Node có taint NoSchedule mà Pod không có toleration → loại

Filter: VolumeBinding
  → Pod yêu cầu PVC ở vùng us-east-1a → chỉ giữ node ở vùng đó

Filter: PodTopologySpread
  → Kiểm tra MaxSkew — phân tán Pod đều giữa các zone/node
```

#### Giai Đoạn 2: Scoring (Tính Điểm) — Priorities

Chấm điểm node còn lại (thang 0–100):

```
Score: LeastAllocated
  → Node ít tải hơn → điểm cao hơn → phân tán tải đều

Score: NodeAffinityPriority
  → preferredDuringSchedulingIgnoredDuringExecution → cộng điểm

Score: PodTopologySpreadPriority
  → Ưu tiên phân tán Pod ra nhiều zone

Score: ImageLocality
  → Node đã có sẵn image Docker → điểm cộng nhỏ
```

**Node được chọn:** Node có tổng điểm cao nhất. Nếu nhiều node bằng điểm → chọn ngẫu nhiên.

### Ví Dụ YAML — NodeAffinity và Taint/Toleration

```yaml
# NodeAffinity — yêu cầu Pod chạy trên node có SSD
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd

# Toleration — cho phép Pod chạy trên node có taint
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

### PodTopologySpreadConstraints — Phân Tán Topology

```yaml
# Đảm bảo Pod phân tán đều giữa các Availability Zone
spec:
  topologySpreadConstraints:
    - maxSkew: 1               # Chênh lệch tối đa giữa các zone
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: web
```

### Custom Scheduler (Scheduler Tuỳ Chỉnh)

Kubernetes cho phép chạy nhiều scheduler song song. Thường dùng khi cần logic đặc biệt:

```yaml
spec:
  schedulerName: my-custom-scheduler  # Dùng scheduler tuỳ chỉnh thay vì mặc định
```

---

## Controller Manager

### Kiến Trúc

`kube-controller-manager` là một tiến trình duy nhất chạy nhiều **controller** độc lập. Mỗi controller:

1. **Watch** một loại resource qua API Server
2. So sánh Desired State với Current State
3. Thực hiện hành động để đưa về Desired State

```
kube-controller-manager
├── ReplicaSet Controller
├── Deployment Controller
├── StatefulSet Controller
├── DaemonSet Controller
├── Job Controller
├── CronJob Controller
├── Node Controller
├── Service Account Controller
├── Endpoints Controller
├── EndpointSlice Controller
├── Namespace Controller
├── PersistentVolume Controller
└── ... (tổng cộng ~30+ controller)
```

### Các Controller Quan Trọng

#### ReplicaSet Controller

```
Sự kiện: Pod bị xoá
  → ReplicaSet Controller phát hiện: currentReplicas < desiredReplicas
  → Tạo Pod mới để bù vào
  → Cập nhật status.availableReplicas
```

#### Deployment Controller

```
Sự kiện: Deployment được update (image mới)
  → Deployment Controller tạo ReplicaSet mới (với image mới)
  → Tăng dần replica của ReplicaSet mới (maxSurge)
  → Giảm dần replica của ReplicaSet cũ (maxUnavailable)
  → Lặp cho đến khi tất cả Pod chạy image mới
  → Giữ lại ReplicaSet cũ để rollback
```

#### Node Controller

```
Trường hợp 1: Node mất heartbeat
  → Node Controller đánh dấu node Condition Unknown
  → Sau 40 giây: đánh dấu NotReady
  → Sau 5 phút (pod-eviction-timeout): evict Pod khỏi node

Trường hợp 2: Node recover
  → Node Controller nhận heartbeat trở lại
  → Đánh dấu node Ready
  → Scheduler có thể gán Pod lên node lại
```

#### Endpoints Controller

```
Sự kiện: Pod trở thành Ready
  → Endpoints Controller thêm IP của Pod vào Endpoints của Service
  → kube-proxy trên mỗi node cập nhật iptables rules
  → Traffic có thể được route đến Pod mới
```

### Leader Election (Bầu Chọn Leader)

Trong môi trường HA (High Availability), chỉ một instance Controller Manager hoạt động tại một thời điểm:

```bash
# Xem leader hiện tại
kubectl get endpoints kube-controller-manager -n kube-system -o yaml
# Annotation leaderelection.k8s.io/leader chứa tên pod đang là leader
```

---

## Cloud Controller Manager

`cloud-controller-manager` tách biệt logic cloud-specific ra khỏi Controller Manager chính:

```
cloud-controller-manager
├── Node Controller (cloud)      → Xoá node khỏi cloud khi instance bị terminate
├── Route Controller             → Cấu hình route trong cloud VPC
├── Service Controller           → Tạo/cập nhật Cloud Load Balancer khi Service type=LoadBalancer
└── Volume Controller            → Tích hợp với cloud storage (EBS, GCE PD, Azure Disk)
```

Mỗi cloud provider (AWS, GCP, Azure) có Cloud Controller Manager riêng.

---

## High Availability (Tính Sẵn Sàng Cao)

### Cấu Hình HA Control Plane Phổ Biến

```
┌────────────────────────────────────────────────────────┐
│                  Load Balancer (VIP)                   │
│              (HAProxy / AWS NLB / GCP LB)              │
└──────────────────┬─────────────────┬───────────────────┘
                   │                 │
        ┌──────────▼──────┐  ┌───────▼──────────┐
        │ Control Plane 1 │  │ Control Plane 2  │   ← Thêm node 3 cho quorum
        │  API Server     │  │  API Server      │
        │  Scheduler      │  │  Scheduler (standby)
        │  Ctrl Manager   │  │  Ctrl Manager (standby)
        └─────────────────┘  └──────────────────┘
                   │                 │
        ┌──────────▼─────────────────▼───────────┐
        │          etcd Cluster (3 node)          │
        └─────────────────────────────────────────┘
```

**Điểm quan trọng:**
- API Server: **active-active** (cả hai xử lý request qua load balancer)
- Scheduler và Controller Manager: **active-passive** (chỉ leader xử lý, dùng leader election)
- etcd: **active-active** nhưng chỉ leader mới ghi, follower đọc

### Checklist HA Control Plane

```
[ ] 3 Control Plane node trên 3 Availability Zone khác nhau
[ ] 3 etcd node (thường cùng với Control Plane node)
[ ] Load Balancer trước API Server (Internal LB cho internal traffic)
[ ] Backup etcd mỗi giờ, lưu offsite
[ ] Cert rotation (xoay chứng chỉ) được tự động hoá
[ ] Monitor etcd: latency < 10ms, disk I/O, snapshot size
[ ] Test recover từ etcd backup ít nhất mỗi quý
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Điều gì xảy ra khi API Server bị chết?

**Trả lời:** Cluster vẫn tiếp tục chạy — Pod đang chạy không bị ảnh hưởng vì kubelet chạy độc lập. Tuy nhiên: không thể deploy mới, Scheduler không thể lên lịch Pod mới, Controller Manager không thể phát hiện và sửa lỗi. Đây là lý do cần HA Control Plane trên production.

### Q2: etcd có phải là nút thắt cổ chai (bottleneck) không?

**Trả lời:** Có thể, nhất là với cluster lớn (>5000 Pod). etcd nhạy cảm với disk I/O latency — nên dùng SSD NVMe, đặt etcd trên dedicated node. Ngoài ra: không nên store dữ liệu ứng dụng trong etcd (dùng database riêng), giới hạn kích thước object lưu trong ConfigMap/Secret.

### Q3: Tại sao Kubernetes dùng kiến trúc Declarative (khai báo) thay vì Imperative (mệnh lệnh)?

**Trả lời:** Declarative (khai báo trạng thái mong muốn) giúp: (1) Self-healing — system tự đưa về trạng thái đúng; (2) Idempotent — apply nhiều lần không gây hại; (3) GitOps — lưu YAML trong Git và sync tự động; (4) Audit trail — có thể xem lịch sử thay đổi.

### Q4: Admission Webhook dùng để làm gì trong thực tế?

**Trả lời:** Rất nhiều use case: (1) Inject sidecar container tự động (Istio envoy proxy); (2) Validate Pod không dùng latest tag; (3) Enforce resource limit bắt buộc; (4) Inject secret từ Vault; (5) OPA Gatekeeper để enforce policy phức tạp.

### Q5: Sự khác nhau giữa MutatingAdmissionWebhook và ValidatingAdmissionWebhook?

**Trả lời:** Mutating chạy trước và **có thể thay đổi** object (ví dụ: thêm label, set default value, inject container). Validating chạy sau và chỉ **chấp nhận hoặc từ chối** — không sửa object. Mutating chạy song song nhau, nhưng tất cả Mutating phải hoàn thành trước khi Validating bắt đầu.

---

**Xem tiếp:** [worker-node.md](./worker-node.md) để hiểu các thành phần trên Worker Node.
