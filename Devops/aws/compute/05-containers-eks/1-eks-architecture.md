# EKS Architecture — Control Plane, Data Plane & Add-ons

> Hiểu rõ kiến trúc EKS là nền tảng để vận hành, debug và tối ưu cluster Kubernetes trên AWS. File này đi sâu vào từng thành phần, cách chúng tương tác, và các lựa chọn quan trọng khi thiết kế cluster.

---

## 📚 Mục Lục

1. [Control Plane — AWS Managed](#control-plane--aws-managed)
2. [Data Plane — Bạn Quản Lý](#data-plane--bạn-quản-lý)
3. [Communication Flow — Luồng Giao Tiếp](#communication-flow--luồng-giao-tiếp)
4. [EKS Add-ons](#eks-add-ons)
5. [Cluster Endpoint Access Modes](#cluster-endpoint-access-modes)
6. [EKS Versioning & Upgrade Strategy](#eks-versioning--upgrade-strategy)
7. [Thiết Kế Cluster Production-Ready](#thiết-kế-cluster-production-ready)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Control Plane — AWS Managed

### Tổng Quan

**Control plane (mặt phẳng điều khiển)** là "bộ não" của Kubernetes cluster. Với EKS, AWS chạy và quản lý toàn bộ control plane:

- Chạy trong **VPC riêng của AWS**, không phải VPC của bạn
- Trải rộng trên **ít nhất 2 Availability Zones (Vùng Khả Dụng)** để đảm bảo HA
- AWS tự động **patch, backup etcd, và upgrade minor versions bảo mật**
- SLA (Service Level Agreement — Cam Kết Chất Lượng Dịch Vụ): **99.95% uptime**

### Các Thành Phần Control Plane

#### 1. API Server (Máy Chủ API)

```
Vai trò: Gateway duy nhất cho mọi tương tác với cluster
- Nhận lệnh kubectl từ developer
- Nhận API calls từ worker nodes (kubelet)
- Xác thực và ủy quyền mọi request
- Ghi trạng thái vào etcd

Endpoint: https://<cluster-id>.gr7.us-east-1.eks.amazonaws.com
```

#### 2. etcd (Kho Lưu Trữ Trạng Thái)

```
Vai trò: Database distributed key-value, lưu TOÀN BỘ trạng thái cluster
- Mọi object Kubernetes (Pod, Service, ConfigMap...) đều lưu ở đây
- Strongly consistent: mọi node đọc cùng dữ liệu tại cùng thời điểm
- AWS backup etcd tự động và mã hóa bằng KMS

Lưu ý: Bạn không trực tiếp truy cập etcd trong EKS
```

#### 3. Scheduler (Bộ Lập Lịch)

```
Vai trò: Quyết định Pod chạy trên node nào
Thuật toán:
  1. Filtering: Loại bỏ nodes không đủ điều kiện (CPU, memory, taints)
  2. Scoring: Chấm điểm nodes còn lại theo nhiều tiêu chí
  3. Binding: Gán Pod vào node điểm cao nhất

Tiêu chí scoring phổ biến:
  - LeastRequestedPriority: Node ít bị sử dụng nhất
  - NodeAffinityPriority: Ưu tiên node phù hợp với affinity rules
  - InterPodAffinityPriority: Ưu tiên/tránh node đã có Pod liên quan
```

#### 4. Controller Manager (Bộ Quản Lý Controller)

```
Vai trò: Chạy các control loops (vòng lặp điều khiển) đảm bảo desired state
Các controller quan trọng:
  - Deployment Controller: Đảm bảo số replica đúng
  - ReplicaSet Controller: Tạo/xóa Pod khi cần
  - Node Controller: Phát hiện và xử lý node failure
  - Service Account Controller: Tạo default service accounts
  - Job Controller: Theo dõi Job completion
```

#### 5. Cloud Controller Manager (Bộ Quản Lý Cloud Controller)

```
Vai trò: Tích hợp Kubernetes với AWS infrastructure
- Node Controller: Đánh dấu node unreachable khi EC2 instance terminated
- Route Controller: Cấu hình VPC routes
- Service Controller: Tạo/xóa ELB khi có Service type LoadBalancer
```

---

## Data Plane — Bạn Quản Lý

### Tổng Quan

**Data plane (mặt phẳng dữ liệu)** là nơi workload thực sự chạy. Bạn có toàn quyền kiểm soát nhưng cũng chịu trách nhiệm quản lý.

### Các Thành Phần Trên Mỗi Worker Node

#### 1. kubelet

```
Vai trò: Agent chính trên mỗi node, "tay chân" của API Server
- Liên tục watch API Server để nhận Pod assignments
- Giao tiếp với container runtime để start/stop containers
- Báo cáo node status và resource usage lên API Server
- Chạy health checks (liveness/readiness probes) cho containers

kubelet chạy như systemd service, không phải container
```

#### 2. kube-proxy

```
Vai trò: Network proxy, implement Service networking
- Theo dõi Service và Endpoints objects trong cluster
- Cấu hình iptables/ipvs rules để route traffic đến đúng Pod
- Mỗi Service IP (ClusterIP) được map đến Pod IP qua iptables

Mode mặc định: iptables
Mode hiệu năng cao: ipvs (IPVS — IP Virtual Server)
```

#### 3. Container Runtime (Môi Trường Chạy Container)

```
Mặc định trong EKS: containerd (từ K8s 1.24+)
- Là CRI (Container Runtime Interface — Giao Diện Môi Trường Container)
- Pull image, create/start/stop containers
- Cách Docker: containerd là core của Docker, EKS dùng thẳng containerd

Lịch sử: EKS < 1.24 dùng Docker, >= 1.24 dùng containerd thuần
```

#### 4. VPC CNI Plugin (Plugin Mạng)

```
Vai trò: Gán IP address cho Pod trực tiếp từ VPC subnet
- Mỗi Pod nhận 1 IP từ VPC subnet (không phải overlay network)
- Dùng ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi) để mở rộng IP pool
- Pod communicate trực tiếp với AWS services không qua NAT

Lợi ích: Performance tốt hơn, security group có thể áp dụng cho Pod
Chi tiết: Xem 3-eks-networking.md
```

---

## Communication Flow — Luồng Giao Tiếp

### kubectl → API Server → etcd

```
Developer                   AWS VPC (Control Plane)          Worker Node VPC
    │                              │                               │
    │  kubectl apply -f pod.yaml   │                               │
    │─────────────────────────────>│                               │
    │                              │ 1. Auth: Bearer token/cert    │
    │                              │ 2. Validate manifest          │
    │                              │ 3. Write to etcd              │
    │                              │ 4. Scheduler chọn node        │
    │                              │ 5. Write binding to etcd      │
    │                              │──────────────────────────────>│
    │                              │       kubelet watches         │
    │                              │       API Server via HTTPS    │
    │                              │       → start container       │
    │  Pod Running                 │                               │
    │<─────────────────────────────│<──────────────────────────────│
```

### Node → API Server Connection

```
Worker nodes kết nối đến API Server qua:
- HTTPS (443) ra ngoài nếu public endpoint enabled
- PrivateLink (AWS PrivateLink — Kết Nối Riêng Tư) nếu private endpoint enabled

Lưu ý: kubelet PULL lệnh từ API Server (không phải API Server push vào node)
```

---

## EKS Add-ons

### Add-ons Bắt Buộc (Managed bởi AWS hoặc tự deploy)

#### CoreDNS

```yaml
# Chức năng: DNS server cho cluster
# Mỗi Service có DNS name: <service>.<namespace>.svc.cluster.local
# Mỗi Pod có DNS: <pod-ip>.<namespace>.pod.cluster.local

# Cấu hình scaling cho CoreDNS (khuyến nghị production):
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: coredns
  namespace: kube-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coredns
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

#### kube-proxy

```
Chức năng: Implement Service VIP (Virtual IP — IP Ảo) routing
- ClusterIP Services → iptables DNAT rules
- NodePort Services → iptables DNAT + port forwarding
- ExternalTrafficPolicy → điều khiển traffic routing

Upgrade: Khi upgrade EKS cluster, kube-proxy add-on phải upgrade theo
```

#### Amazon VPC CNI

```
Chức năng: Assign VPC IP cho Pod (xem chi tiết trong 3-eks-networking.md)
- Cấu hình quan trọng: ENABLE_PREFIX_DELEGATION=true
  → Tăng số Pod per node bằng cách dùng /28 prefix thay vì individual IPs
```

### Add-ons Khuyến Nghị

| Add-on | Chức Năng | Khi Nào Cần |
|---|---|---|
| **AWS Load Balancer Controller** | Tạo ALB/NLB từ Ingress/Service | Luôn cần cho production |
| **EBS CSI Driver** | Mount EBS volume | Khi cần persistent storage cho Pod |
| **EFS CSI Driver** | Mount EFS filesystem | Khi nhiều Pod cần share storage |
| **Cluster Autoscaler** | Scale node groups | Khi cần auto-scale nodes |
| **Metrics Server** | Thu thập resource metrics | Cần cho HPA |
| **AWS Distro for OpenTelemetry** | Traces và metrics | Observability |
| **Amazon GuardDuty Agent** | Runtime threat detection | Security |

### Managed Add-ons vs Self-managed

```
Managed Add-ons (qua AWS Console/CLI/EKS API):
  ✅ AWS tự động patch và upgrade
  ✅ Tích hợp với AWS Config và CloudFormation
  ✅ Kiểm soát version cụ thể
  ⚠️  Ít tùy chỉnh cấu hình hơn

Self-managed (deploy bằng Helm/kubectl):
  ✅ Toàn quyền tùy chỉnh
  ✅ Thử nghiệm version mới trước
  ⚠️  Bạn chịu trách nhiệm upgrade và patching
```

---

## Cluster Endpoint Access Modes

### 3 Chế Độ Truy Cập API Server

#### 1. Public Only (Mặc Định)

```
API Server endpoint công khai, worker nodes giao tiếp qua internet
Bảo mật: Giới hạn bằng CIDR whitelist

Dùng khi: Development/test clusters
Rủi ro: API Server expose ra internet (dù có auth)
```

#### 2. Public + Private

```
API Server có cả endpoint công khai và endpoint riêng trong VPC
- Developer ngoài office → qua public endpoint
- Worker nodes trong VPC → qua private endpoint (traffic không ra internet)

Dùng khi: Production với team cần truy cập từ nhiều network
Khuyến nghị: Bật CIDR restriction cho public endpoint
```

#### 3. Private Only (Khuyến Nghị Cho Production)

```
API Server chỉ có endpoint riêng trong VPC
- Developer phải vào VPN hoặc bastion host để kubectl
- Worker nodes giao tiếp hoàn toàn trong VPC

Dùng khi: Production clusters với yêu cầu bảo mật cao
Cần thêm: VPN, AWS Client VPN, hoặc AWS Direct Connect
```

### Cấu Hình Endpoint Access

```bash
# Bật private endpoint và giới hạn public endpoint
aws eks update-cluster-config \
  --name my-cluster \
  --resources-vpc-config \
    endpointPublicAccess=true,\
    endpointPrivateAccess=true,\
    publicAccessCidrs=["203.0.113.0/24","198.51.100.0/24"]
```

---

## EKS Versioning & Upgrade Strategy

### Kubernetes Version Lifecycle Trên EKS

```
Hỗ trợ: N-2 minor versions (ví dụ: 1.30, 1.29, 1.28 khi 1.30 là latest)
Mỗi version: ~14 tháng support
Release mới: ~3 versions/năm
End of support: AWS buộc upgrade sau khi version hết hỗ trợ (extended support có phí)
```

### Quy Trình Upgrade An Toàn

```
Bước 1: Đọc release notes và breaking changes
Bước 2: Test trên non-production cluster trước
Bước 3: Upgrade control plane (qua Console hoặc CLI)
         aws eks update-cluster-version --name <cluster> --kubernetes-version <version>
Bước 4: Upgrade add-ons (CoreDNS, kube-proxy, VPC CNI) theo thứ tự
Bước 5: Upgrade managed node groups (rolling update)
         aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <ng>
Bước 6: Verify workloads healthy sau upgrade

QUAN TRỌNG: Luôn upgrade control plane trước, sau đó mới upgrade nodes
KHÔNG downgrade: Kubernetes không hỗ trợ downgrade version
```

---

## Thiết Kế Cluster Production-Ready

### Checklist Kiến Trúc

```
Control Plane:
  ✅ Private endpoint hoặc Public+Private với CIDR restriction
  ✅ Secrets encryption bằng KMS
  ✅ Control plane logs gửi đến CloudWatch (API Server, Audit, Authenticator)
  ✅ EKS version không quá 1 version cũ hơn latest

Data Plane:
  ✅ Worker nodes phân bổ đều trên ít nhất 2 AZ (khuyến nghị 3)
  ✅ Node groups dùng private subnets
  ✅ Security groups chỉ mở cổng cần thiết
  ✅ Managed node groups để AWS quản lý AMI updates

Networking:
  ✅ VPC đủ IP space cho Pod expansion
  ✅ ENABLE_PREFIX_DELEGATION cho VPC CNI (tăng Pod density)
  ✅ Ingress qua AWS Load Balancer Controller

Security:
  ✅ IRSA thay vì instance profile cho Pod permissions
  ✅ RBAC setup chặt chẽ — least privilege
  ✅ Pod Security Standards enforced
  ✅ Network Policies cho micro-segmentation

Observability:
  ✅ Container Insights bật (CloudWatch Metrics + Logs)
  ✅ Metrics Server cho HPA
  ✅ Cluster Autoscaler cho node scaling
```

### Ví Dụ: Enable Control Plane Logging

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --logging \
  '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Log groups tự động tạo trong CloudWatch:
# /aws/eks/<cluster-name>/cluster
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Mô tả điều gì xảy ra khi bạn chạy `kubectl apply -f deployment.yaml`

**Trả lời đầy đủ:**

```
1. kubectl gửi HTTP PUT/POST request đến API Server endpoint
2. API Server xác thực danh tính (bearer token hoặc client certificate)
3. API Server kiểm tra quyền qua RBAC (bạn có quyền create Deployment không?)
4. Admission controllers (webhook) validate và mutate object nếu cần
5. API Server lưu Deployment object vào etcd
6. Deployment Controller (trong Controller Manager) phát hiện Deployment mới
7. Deployment Controller tạo ReplicaSet object, lưu vào etcd
8. ReplicaSet Controller tạo Pod objects (số lượng = replicas), lưu vào etcd
9. Scheduler phát hiện Pod chưa được assign (nodeName = "")
10. Scheduler chọn node phù hợp (filter + score), cập nhật Pod.spec.nodeName
11. kubelet trên node đó watch API Server, thấy Pod mới assigned
12. kubelet giao tiếp với containerd: pull image, create container, start
13. kubelet cập nhật Pod status (Running) lên API Server → lưu etcd
14. kubectl thấy Deployment READY
```

### Câu 2: EKS control plane SLA là bao nhiêu? Điều gì xảy ra khi control plane down?

**Trả lời:**
> EKS cam kết 99.95% monthly uptime SLA. Khi control plane temporarily unavailable, **workloads đang chạy vẫn tiếp tục chạy bình thường** — kubelet trên nodes đã có Pod specs cần thiết. Tuy nhiên bạn không thể deploy changes mới, auto-scaling sẽ không hoạt động, và health checks không được update. Kubernetes được thiết kế để data plane tiếp tục hoạt động độc lập trong thời gian ngắn.

### Câu 3: Tại sao EKS tính phí control plane dù không có node?

**Trả lời:**
> Vì AWS vẫn đang chạy infrastructure cho control plane của bạn (API Server, etcd, Scheduler trên multi-AZ). Dù cluster rỗng, họ vẫn duy trì HA infrastructure. Đây là sự khác biệt so với ECS — ECS không charge phí cluster management.

### Câu 4: Khi nào nên bật private endpoint cho EKS?

**Trả lời:**
> Luôn bật private endpoint cho production. Với private endpoint, worker node traffic không ra internet — giảm latency, giảm egress cost, và tăng bảo mật. Public endpoint có thể giữ (với CIDR whitelist) để admin kubectl từ xa, hoặc tắt hoàn toàn nếu dùng VPN/bastion. Private-only endpoint là best practice cho regulated industries.

---

**Tiếp theo:** [2-node-groups.md](./2-node-groups.md) — Managed Node Groups, Self-managed Nodes, và Fargate Profiles.
