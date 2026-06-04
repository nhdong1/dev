# EKS — Elastic Kubernetes Service: Tổng Quan & Kiến Trúc

> **EKS (Elastic Kubernetes Service — Dịch Vụ Kubernetes Được Quản Lý)** là dịch vụ AWS giúp chạy Kubernetes mà không cần tự quản lý control plane (mặt phẳng điều khiển). AWS xử lý tính sẵn sàng, bảo mật và nâng cấp của Kubernetes control plane; bạn chỉ tập trung vào workload (tải công việc).

---

## 📚 Mục Lục

1. [EKS là gì & Khi Nào Dùng](#eks-là-gì--khi-nào-dùng)
2. [Kiến Trúc Tổng Quan](#kiến-trúc-tổng-quan)
3. [Các Thành Phần Chính](#các-thành-phần-chính)
4. [Các File Trong Module Này](#các-file-trong-module-này)
5. [ECS vs EKS — Khi Nào Chọn Cái Nào](#ecs-vs-eks--khi-nào-chọn-cái-nào)
6. [Lộ Trình Học EKS](#lộ-trình-học-eks)
7. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)

---

## EKS là gì & Khi Nào Dùng

### Định Nghĩa

**EKS** là managed Kubernetes service của AWS. "Managed" có nghĩa là:

- AWS vận hành và duy trì **Kubernetes control plane** (API Server, etcd, Scheduler, Controller Manager)
- Bạn chịu trách nhiệm quản lý **data plane** (worker nodes — node làm việc) hoặc dùng Fargate để AWS lo luôn
- Tự động upgrade, HA (High Availability — Tính Sẵn Sàng Cao), patching cho control plane

### Khi Nào Dùng EKS

| Tình Huống | Lý Do Chọn EKS |
|---|---|
| Đang dùng Kubernetes on-premises và muốn migrate lên cloud | Không phải viết lại ứng dụng |
| Cần tính di động multi-cloud (GKE, AKS, EKS) | Kubernetes manifest dùng được ở mọi nơi |
| Team đã thành thạo Kubernetes | Tận dụng hệ sinh thái Helm, Kustomize, ArgoCD |
| Workload phức tạp cần custom scheduling | Kubernetes scheduler linh hoạt hơn ECS |
| Cần advanced networking policy (Chính Sách Mạng Nâng Cao) | NetworkPolicy API của Kubernetes |
| Microservices (Kiến Trúc Vi Dịch Vụ) quy mô lớn | Quản lý hàng trăm services hiệu quả |

### Khi Không Nên Dùng EKS

- Team nhỏ, không ai biết Kubernetes → Dùng ECS Fargate đơn giản hơn
- Ứng dụng đơn giản, không cần orchestration phức tạp → Lambda hoặc ECS
- Muốn tối thiểu overhead quản lý → ECS Fargate tích hợp AWS tốt hơn
- Budget eo hẹp → EKS control plane tốn $0.10/giờ (~$72/tháng) dù không có node nào

---

## Kiến Trúc Tổng Quan

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Account của bạn                         │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │           EKS Control Plane (AWS Managed)                    │   │
│  │                                                              │   │
│  │   ┌──────────────┐  ┌──────────┐  ┌──────────────────────┐  │   │
│  │   │  API Server  │  │  etcd    │  │ Controller Manager   │  │   │
│  │   │ (Máy Chủ API)│  │(Lưu Trữ)│  │ Scheduler            │  │   │
│  │   └──────────────┘  └──────────┘  └──────────────────────┘  │   │
│  │                                                              │   │
│  │   Multi-AZ, Auto-Healing, Automatically Patched by AWS       │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │ kubectl / API calls                  │
│  ┌───────────────────────────▼──────────────────────────────────┐   │
│  │                    Data Plane (Bạn Quản Lý)                  │   │
│  │                                                              │   │
│  │  ┌──────────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │  Managed Node Group  │  │   Self-managed Node Group    │  │   │
│  │  │ (EC2 + AWS quản lý   │  │   (EC2 bạn tự quản lý)      │  │   │
│  │  │  kubelet, AMI update)│  │                              │  │   │
│  │  └──────────────────────┘  └──────────────────────────────┘  │   │
│  │                                                              │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │        Fargate Profile (AWS quản lý node)             │   │   │
│  │  │    Pod → AWS tạo micro-VM riêng, bạn không thấy node  │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Các Thành Phần Chính

### Control Plane (Mặt Phẳng Điều Khiển) — AWS Quản Lý

| Thành Phần | Vai Trò |
|---|---|
| **API Server** | Nhận mọi lệnh kubectl, REST API endpoint |
| **etcd** | Key-value store lưu toàn bộ trạng thái cluster |
| **Scheduler** | Quyết định Pod (đơn vị chạy container) chạy trên node nào |
| **Controller Manager** | Đảm bảo desired state (trạng thái mong muốn) = actual state |
| **Cloud Controller Manager** | Tích hợp với AWS: ELB, EBS, VPC |

### Data Plane (Mặt Phẳng Dữ Liệu) — Bạn Quản Lý

| Thành Phần | Vai Trò |
|---|---|
| **kubelet** | Agent chạy trên mỗi node, thực thi lệnh từ API Server |
| **kube-proxy** | Quản lý iptables/ipvs cho Service networking |
| **Container Runtime** | containerd (mặc định), chạy container thực sự |
| **VPC CNI Plugin** | Plugin mạng của AWS, gán IP VPC thực cho Pod |

### Add-ons (Tiện Ích Mở Rộng) Quan Trọng

| Add-on | Chức Năng |
|---|---|
| **CoreDNS** | DNS nội bộ cho cluster, giải quyết Service name |
| **kube-proxy** | Network proxy, routing Service traffic |
| **Amazon VPC CNI** | Mạng cho Pod dùng ENI (Elastic Network Interface) |
| **AWS Load Balancer Controller** | Tạo ALB/NLB từ Kubernetes Ingress/Service |
| **EBS CSI Driver** | Mount EBS volume cho Pod |
| **EFS CSI Driver** | Mount EFS filesystem cho Pod |
| **Cluster Autoscaler** | Tự động thêm/xóa node theo nhu cầu Pod |
| **KEDA** | Event-driven autoscaling cho Pod |

---

## Các File Trong Module Này

| File | Nội Dung |
|---|---|
| `README.md` | File này — Tổng quan & lộ trình |
| `1-eks-architecture.md` | Control plane, data plane, add-ons chi tiết |
| `2-node-groups.md` | Managed Nodes, Self-managed Nodes, Fargate Profiles |
| `3-eks-networking.md` | VPC CNI, CoreDNS, kube-proxy, Ingress, NetworkPolicy |
| `4-eks-storage.md` | EBS CSI, EFS CSI, StatefulSets, StorageClass |
| `5-eks-security.md` | RBAC, IRSA, Pod Security Standards, Secrets encryption |

---

## ECS vs EKS — Khi Nào Chọn Cái Nào

| Tiêu Chí | ECS | EKS |
|---|---|---|
| **Độ phức tạp** | Thấp — AWS concepts thuần | Cao — cần học Kubernetes |
| **Tích hợp AWS** | Tốt nhất — native | Tốt — qua add-ons |
| **Tính di động** | Khóa vào AWS | Portable sang GKE, AKS |
| **Hệ sinh thái** | Giới hạn AWS | Phong phú — Helm, ArgoCD, Istio |
| **Chi phí control plane** | Miễn phí | $0.10/giờ (~$72/tháng) |
| **Fargate support** | Tốt hơn | Hỗ trợ nhưng nhiều giới hạn |
| **Custom scheduling** | Không | Có |
| **Multi-tenancy** | Hạn chế | Mạnh — Namespace, RBAC |
| **Team phù hợp** | DevOps không biết K8s | DevOps thành thạo Kubernetes |

### Quy Tắc Quyết Định Nhanh

```
Có dùng Kubernetes hiện tại không?
├── Có → EKS (migrate dễ hơn)
└── Không → Team có muốn học K8s không?
    ├── Có + workload phức tạp → EKS
    └── Không → ECS Fargate (đơn giản hơn, tích hợp AWS tốt hơn)
```

---

## Lộ Trình Học EKS

### Tuần 1-2: Nền Tảng Kubernetes

```
1. Hiểu các khái niệm cốt lõi: Pod, Deployment, Service, Namespace
2. Cài kubectl, kết nối vào EKS cluster
3. Deploy ứng dụng đơn giản lên EKS
4. Tạo Managed Node Group đầu tiên
```

### Tuần 3-4: Networking & Storage

```
1. Tìm hiểu VPC CNI và cách Pod nhận IP VPC
2. Cấu hình Ingress với AWS Load Balancer Controller
3. Mount EBS volume vào Pod (StatefulSet)
4. Cấu hình CoreDNS cho service discovery
```

### Tuần 5-6: Security & Operations

```
1. Thiết lập RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Theo Vai Trò)
2. Cấu hình IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho Service Accounts)
3. Enable Secrets encryption với KMS
4. Cấu hình Cluster Autoscaler và HPA (Horizontal Pod Autoscaler — Tự Động Mở Rộng Pod Theo Chiều Ngang)
```

### Tuần 7+: Nâng Cao

```
1. GitOps với ArgoCD hoặc Flux
2. Service Mesh với AWS App Mesh hoặc Istio
3. Observability: Prometheus, Grafana, AWS Container Insights
4. Multi-cluster management
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: EKS khác ECS ở đâu? Khi nào chọn cái nào?

**Trả lời tóm tắt:**
> ECS là dịch vụ container thuần AWS — đơn giản, tích hợp native, không cần biết Kubernetes. EKS chạy Kubernetes trên AWS — portable, hệ sinh thái phong phú nhưng phức tạp hơn. Chọn ECS khi team không biết K8s hoặc workload đơn giản. Chọn EKS khi đang dùng K8s, cần multi-cloud, hoặc workload phức tạp cần K8s ecosystem.

### Câu 2: EKS control plane được quản lý thế nào bởi AWS?

**Trả lời tóm tắt:**
> AWS chạy control plane (API Server, etcd, Scheduler, Controller Manager) trên multi-AZ infrastructure riêng biệt, không nằm trong VPC của bạn. AWS tự động vá lỗi bảo mật, upgrade, và đảm bảo 99.95% SLA. Bạn chỉ tương tác qua endpoint của API Server.

### Câu 3: IRSA là gì và tại sao nó quan trọng hơn instance profile?

**Trả lời tóm tắt:**
> IRSA (IAM Roles for Service Accounts) cho phép mỗi Pod có IAM role riêng thay vì chia sẻ role của node. Điều này áp dụng nguyên tắc least privilege (đặc quyền tối thiểu) ở cấp Pod — Pod A chỉ đọc S3, Pod B chỉ ghi DynamoDB. Instance profile gán cùng role cho mọi Pod trên node, vi phạm least privilege.

### Câu 4: Cluster Autoscaler (Tự Động Co Giãn Cluster) hoạt động thế nào?

**Trả lời tóm tắt:**
> Cluster Autoscaler theo dõi các Pod ở trạng thái Pending (chờ) do không đủ tài nguyên. Khi phát hiện Pod bị Pending, nó tăng số node trong Node Group (scale out). Khi node idle quá 10 phút (mặc định), nó giảm node (scale in). Phối hợp với HPA: HPA tăng số Pod → Pod Pending → Cluster Autoscaler tăng node.

---

## 📌 Tóm Tắt Nhanh

| Điểm Chính | Giá Trị |
|---|---|
| Control plane cost | $0.10/giờ (~$72/tháng) |
| SLA | 99.95% |
| Kubernetes versions hỗ trợ | N-2 minor versions |
| Upgrade cadence | ~3 versions/năm, support 14 tháng mỗi version |
| Max nodes per cluster | 450 nodes (managed node group) |
| Max Pods per node | Phụ thuộc instance type + ENI limits |

---

**Tiếp theo:** [1-eks-architecture.md](./1-eks-architecture.md) — Tìm hiểu chi tiết control plane, data plane và các add-ons quan trọng.
