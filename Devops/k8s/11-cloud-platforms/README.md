# Cloud Platforms — Nền Tảng Kubernetes Được Quản Lý

> So sánh toàn diện Amazon EKS, Google GKE, Azure AKS và Self-Managed Kubernetes — giúp bạn chọn đúng nền tảng, hiểu điểm mạnh/yếu từng loại, và chuẩn bị câu hỏi phỏng vấn về cloud-native Kubernetes.

## Mục Lục

1. [Tổng Quan Về Managed vs Self-Managed](#tổng-quan-về-managed-vs-self-managed)
2. [So Sánh Các Nền Tảng](#so-sánh-các-nền-tảng)
3. [Amazon EKS](#amazon-eks)
4. [Google GKE](#google-gke)
5. [Azure AKS](#azure-aks)
6. [Self-Managed Kubernetes](#self-managed-kubernetes)
7. [Ma Trận Lựa Chọn](#ma-trận-lựa-chọn)
8. [Tích Hợp IAM với Cloud Provider](#tích-hợp-iam-với-cloud-provider)
9. [Tối Ưu Chi Phí Trên Cloud](#tối-ưu-chi-phí-trên-cloud)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan Về Managed vs Self-Managed

### Managed Kubernetes (Kubernetes Được Quản Lý)

Cloud provider chịu trách nhiệm vận hành **Control Plane** (mặt điều khiển) — bao gồm API Server, etcd, Scheduler, Controller Manager. Bạn chỉ quản lý Worker Node và workload chạy trên đó.

```
┌─────────────────────────────────────────────────┐
│           MANAGED CONTROL PLANE                 │
│  (Cloud Provider quản lý — SLA được đảm bảo)   │
│                                                 │
│  API Server  │  etcd  │  Scheduler  │  CM       │
└─────────────────────────────────────────────────┘
         ↕ kết nối an toàn
┌─────────────────────────────────────────────────┐
│           WORKER NODES (Bạn quản lý)            │
│                                                 │
│  Node 1      │  Node 2      │  Node 3           │
│  kubelet     │  kubelet     │  kubelet           │
│  kube-proxy  │  kube-proxy  │  kube-proxy        │
│  containerd  │  containerd  │  containerd        │
└─────────────────────────────────────────────────┘
```

**Ưu điểm:**
- Không cần quản lý etcd backup, Control Plane upgrade
- SLA (Service Level Agreement — Thỏa Thuận Cấp Độ Dịch Vụ) thường đạt 99.9%–99.99%
- Tích hợp sâu với dịch vụ cloud (IAM, Load Balancer, Storage)
- Tự động vá lỗi bảo mật cho Control Plane

**Nhược điểm:**
- Chi phí cao hơn (phí quản lý Control Plane)
- Bị khoá vào một cloud provider (vendor lock-in)
- Phiên bản K8s có thể bị trễ so với upstream

### Self-Managed Kubernetes (Kubernetes Tự Quản Lý)

Bạn tự cài đặt và vận hành toàn bộ cluster — cả Control Plane lẫn Worker Node.

**Ưu điểm:**
- Toàn quyền kiểm soát cấu hình
- Tiết kiệm chi phí (không có phí quản lý)
- Không bị khoá vào cloud provider
- Phù hợp on-premise, bare-metal, air-gapped

**Nhược điểm:**
- Trách nhiệm vận hành hoàn toàn thuộc về bạn
- Phải tự xử lý backup etcd, upgrade, HA (High Availability — Tính Sẵn Sàng Cao)
- Đòi hỏi đội ngũ có kiến thức chuyên sâu

---

## So Sánh Các Nền Tảng

| Tiêu Chí                          | EKS (AWS)           | GKE (GCP)            | AKS (Azure)          | Self-Managed        |
| --------------------------------- | ------------------- | -------------------- | -------------------- | ------------------- |
| **Control Plane**                 | Managed             | Managed              | Managed              | Tự quản lý          |
| **Phí Control Plane**             | $0.10/giờ/cluster   | Miễn phí (Standard)  | Miễn phí             | Không (tự vận hành) |
| **IAM tích hợp**                  | IRSA / Pod Identity | Workload Identity    | Workload Identity    | Tự cấu hình         |
| **Serverless Node**               | Fargate             | Autopilot            | Virtual Nodes (ACI)  | Không có            |
| **Upgrade tự động**               | Có (managed)        | Có (auto-upgrade)    | Có (auto-upgrade)    | Tự thực hiện        |
| **GPU Node**                      | Có                  | Có                   | Có                   | Phụ thuộc phần cứng |
| **Windows Container**             | Có                  | Có (hạn chế)         | Hỗ trợ tốt nhất      | Có (thủ công)       |
| **Multi-cluster**                 | EKS Connector       | Fleet / Anthos       | Azure Arc            | Federation/Admiralty|
| **Service Mesh tích hợp**         | App Mesh / Istio    | Anthos Service Mesh  | Open Service Mesh    | Tự cài              |
| **Phù hợp nhất**                  | AWS ecosystem       | Đơn giản nhất        | Enterprise + .NET    | On-premise / Edge   |

---

## Amazon EKS

**EKS (Elastic Kubernetes Service — Dịch Vụ Kubernetes Đàn Hồi)** là managed Kubernetes trên AWS.

### Điểm Nổi Bật

- **IRSA (IAM Roles for Service Accounts — Vai Trò IAM cho ServiceAccount):** Cho phép Pod truy cập AWS service (S3, DynamoDB, SQS...) mà không cần access key
- **Fargate Profile:** Chạy Pod serverless — không cần quản lý node, AWS tự động cấp phát tài nguyên
- **Managed Node Group (Nhóm Node Được Quản Lý):** AWS tự động cập nhật và vá lỗi node
- **EKS Add-on (Tiện Ích EKS):** Quản lý CoreDNS, kube-proxy, VPC CNI, EBS CSI qua AWS Console
- **AWS VPC CNI (Container Network Interface — Giao Diện Mạng Container):** Mỗi Pod nhận IP trực tiếp từ VPC — giao tiếp mạng native

### Khi Nào Chọn EKS

```
✅ Hệ thống đang dùng nhiều dịch vụ AWS (RDS, S3, ALB, SQS)
✅ Cần tích hợp IAM chặt chẽ với workload K8s
✅ Team đã quen với AWS Console và CLI
✅ Cần Fargate để giảm overhead quản lý node
```

Chi tiết xem: [eks.md](./eks.md)

---

## Google GKE

**GKE (Google Kubernetes Engine — Google Kubernetes Engine)** là managed Kubernetes trên GCP — được coi là mature nhất vì Google là người tạo ra Kubernetes.

### Điểm Nổi Bật

- **Autopilot Mode (Chế Độ Tự Động):** GKE tự động quản lý node, scaling và bảo mật — bạn chỉ deploy workload
- **Workload Identity:** Liên kết Kubernetes ServiceAccount với Google Service Account — truy cập GCP API không cần key
- **Release Channel (Kênh Phát Hành):** Rapid, Regular, Stable — kiểm soát tốc độ nhận bản cập nhật K8s
- **GKE Autopilot:** Tính phí theo Pod resource sử dụng thực tế, không theo node

### Khi Nào Chọn GKE

```
✅ Muốn cluster đơn giản nhất để vận hành (Autopilot)
✅ Sử dụng GCP ecosystem (BigQuery, Cloud SQL, GCS, Artifact Registry)
✅ Cần tính năng K8s mới nhất (GKE thường cập nhật nhanh nhất)
✅ Môi trường đa khu vực (multi-region) phức tạp
```

Chi tiết xem: [gke.md](./gke.md)

---

## Azure AKS

**AKS (Azure Kubernetes Service — Dịch Vụ Azure Kubernetes)** là managed Kubernetes trên Azure — mạnh về tích hợp với hệ sinh thái Microsoft.

### Điểm Nổi Bật

- **Azure AD Integration (Tích Hợp Azure Active Directory):** RBAC K8s kết hợp với Azure AD user/group
- **Workload Identity (Định Danh Workload):** Thay thế AAD Pod Identity — liên kết ServiceAccount với managed identity Azure
- **Node Pool (Nhóm Node):** Hỗ trợ nhiều pool với OS khác nhau (Linux và Windows) trong cùng cluster
- **Azure DevOps Integration:** Pipeline tích hợp sẵn với AKS deployment
- **Windows Container:** Hỗ trợ tốt nhất trong ba managed provider

### Khi Nào Chọn AKS

```
✅ Hệ thống enterprise đang dùng Azure và Office 365
✅ Cần chạy Windows container (.NET Framework, SQL Server Agent)
✅ Dùng Azure DevOps để CI/CD
✅ Cần tích hợp Active Directory sâu cho RBAC
```

Chi tiết xem: [aks.md](./aks.md)

---

## Self-Managed Kubernetes

Cài đặt và vận hành K8s thủ công bằng các công cụ như **kubeadm**, **k3s**, hoặc **RKE2**.

### Điểm Nổi Bật

- **kubeadm:** Công cụ chính thức của K8s upstream — phù hợp cluster production on-premise
- **k3s:** Phân phối K8s nhẹ của Rancher — phù hợp edge, IoT, lab
- **RKE2 (Rancher Kubernetes Engine 2):** Tập trung bảo mật, phù hợp môi trường có yêu cầu compliance
- **Kubespray:** Cài đặt K8s qua Ansible — dễ tự động hoá

### Khi Nào Chọn Self-Managed

```
✅ On-premise hoặc bare-metal không dùng cloud
✅ Môi trường air-gapped (không có kết nối Internet)
✅ Yêu cầu compliance nghiêm ngặt (cần kiểm soát toàn bộ stack)
✅ Muốn tiết kiệm chi phí — không trả phí managed service
✅ Edge computing và IoT với tài nguyên hạn chế
```

Chi tiết xem: [self-managed.md](./self-managed.md)

---

## Ma Trận Lựa Chọn

### Theo Mục Tiêu

```
Cần đơn giản nhất?
  → GKE Autopilot

Đang dùng AWS nặng?
  → EKS

Đang dùng Azure / cần Windows container?
  → AKS

On-premise hoặc air-gapped?
  → kubeadm / RKE2

Cluster nhỏ, edge, IoT?
  → k3s

Muốn học K8s thuần?
  → kubeadm (học nhiều nhất từ việc cài tay)
```

### Theo Quy Mô Đội Nhóm

| Quy Mô Đội           | Khuyến Nghị                                                        |
| -------------------- | ------------------------------------------------------------------ |
| 1–3 kỹ sư            | GKE Autopilot hoặc EKS Fargate (giảm overhead vận hành)           |
| 3–10 kỹ sư           | EKS / GKE / AKS standard với Managed Node Group                   |
| 10+ kỹ sư (platform) | Multi-cluster với Anthos / Azure Arc hoặc tự quản lý              |
| Enterprise           | AKS với Azure AD, hoặc RKE2 với bảo mật nâng cao                  |

### Theo Yêu Cầu Compliance

| Tiêu Chuẩn            | Giải Pháp Khuyến Nghị                        |
| --------------------- | -------------------------------------------- |
| SOC 2 / ISO 27001     | EKS, GKE, AKS đều được chứng nhận           |
| FedRAMP (Chính phủ Mỹ)| EKS Gov Cloud, AKS Gov                      |
| GDPR (Châu Âu)        | Cần lưu dữ liệu trong EU region cụ thể      |
| Air-gapped            | Self-managed với kubeadm hoặc RKE2           |

---

## Tích Hợp IAM với Cloud Provider

### Vấn Đề Cần Giải Quyết

Pod trong Kubernetes cần truy cập cloud service (S3, Cloud Storage, Blob Storage...) một cách an toàn — không dùng static credential (thông tin đăng nhập tĩnh) vì:

- Static credential có thể bị lộ qua log, environment variable
- Khó rotate (xoay vòng) thường xuyên
- Không có audit trail (dấu vết kiểm tra) chi tiết theo từng Pod

### Giải Pháp Theo Platform

```
┌──────────────────┬──────────────────────────────────────────────────────┐
│ Platform         │ Giải Pháp IAM                                        │
├──────────────────┼──────────────────────────────────────────────────────┤
│ EKS (AWS)        │ IRSA — gán IAM Role cho ServiceAccount qua OIDC     │
│                  │ EKS Pod Identity — thế hệ mới, đơn giản hơn IRSA    │
├──────────────────┼──────────────────────────────────────────────────────┤
│ GKE (GCP)        │ Workload Identity — liên kết KSA với GSA qua OIDC   │
│                  │ (KSA = K8s ServiceAccount, GSA = Google SA)          │
├──────────────────┼──────────────────────────────────────────────────────┤
│ AKS (Azure)      │ Workload Identity — liên kết KSA với Azure Managed  │
│                  │ Identity qua federated credentials                   │
├──────────────────┼──────────────────────────────────────────────────────┤
│ Self-Managed     │ Tự cài OIDC provider, hoặc dùng Vault Agent Injector │
└──────────────────┴──────────────────────────────────────────────────────┘
```

### Nguyên Lý Chung (OIDC Federation)

```
Pod → ServiceAccount → OIDC Token → Cloud IAM → Temporary Credentials
                                                  (Thông Tin Xác Thực Tạm Thời)
```

1. Pod được gắn ServiceAccount có annotation chứa cloud role ARN/ID
2. Cloud provider xác thực OIDC token do K8s API Server phát hành
3. Cloud provider trả về temporary credential ngắn hạn (thường 1 giờ)
4. Pod dùng credential đó để gọi cloud API — không cần secret nào trong cluster

---

## Tối Ưu Chi Phí Trên Cloud

### Spot / Preemptible Node

- **AWS Spot Instance (Instance Spot):** Tiết kiệm 70–90% so với On-Demand — nhưng có thể bị thu hồi với 2 phút cảnh báo
- **GCP Preemptible / Spot VM:** Tiết kiệm 60–91% — bị thu hồi sau tối đa 24 giờ
- **Azure Spot VM:** Tiết kiệm tương tự — giá thay đổi theo thị trường

**Nguyên tắc sử dụng Spot Node:**

```
✅ Workload có thể bị gián đoạn: batch job, CI/CD runner, data processing
✅ Stateless service với nhiều replica (tolerate node loss)
✅ Kết hợp với Cluster Autoscaler để tự động thay thế khi Spot bị thu hồi

❌ Stateful workload (database, message queue) không phù hợp Spot
❌ Single-replica service không nên chạy hoàn toàn trên Spot
```

### Các Chiến Lược Tiết Kiệm Chi Phí

```
1. Right-sizing (Định Kỡ Đúng):
   - Dùng VPA để tìm resource request phù hợp thực tế
   - Không over-provision CPU/memory

2. Node Auto-provisioning (Cấp Phát Node Tự Động):
   - Cluster Autoscaler thu nhỏ cluster khi ít tải
   - Karpenter (AWS) — nhanh hơn Cluster Autoscaler chuẩn

3. Scheduling tối ưu:
   - Dùng nodeSelector / affinity để Pod phù hợp đúng node type
   - Pack Pod lên ít node hơn để tắt node dư

4. Reserved/Committed Use Discount:
   - Committed Use (GCP): tiết kiệm 57% với 1-3 năm commit
   - Reserved Instance (AWS): tiết kiệm 40–75%
   - Reserved Capacity (Azure): tiết kiệm 40–72%
```

---

## Câu Hỏi Phỏng Vấn

### Câu Hỏi Về Lựa Chọn Platform

**Q: Khi nào bạn chọn EKS thay vì GKE hay AKS?**

> Tôi chọn EKS khi hệ thống đã đang dùng nhiều dịch vụ AWS như RDS, S3, SQS — vì IRSA và VPC CNI giúp tích hợp native, giảm độ phức tạp cấu hình. Nếu bắt đầu mới và muốn vận hành đơn giản nhất, tôi sẽ chọn GKE Autopilot.

**Q: Sự khác biệt giữa Managed Node Group và Fargate trên EKS?**

> Managed Node Group là EC2 instance bình thường — bạn vẫn phải quản lý OS, nhưng AWS tự động vá lỗi và upgrade. Fargate là serverless — không có node để quản lý, AWS tự cấp phát tài nguyên theo từng Pod; phù hợp khi muốn giảm hoàn toàn overhead node management, nhưng có giới hạn (không hỗ trợ DaemonSet, không mount hostPath).

**Q: IRSA hoạt động thế nào? Tại sao dùng thay vì IAM User?**

> IRSA dùng OIDC federation: khi Pod khởi động, API Server phát hành JWT token cho ServiceAccount. AWS IAM xác thực token đó qua OIDC endpoint của EKS cluster, rồi cấp temporary credential ngắn hạn theo role đã cấu hình. Lợi thế: không có static secret, credential tự expire, audit trail chi tiết per-Pod qua CloudTrail.

### Câu Hỏi Về Vận Hành

**Q: Làm thế nào để upgrade EKS/GKE/AKS cluster an toàn?**

> Quy trình chung: (1) Test upgrade trên staging cluster trước. (2) Upgrade Control Plane trước — các bước này managed provider tự xử lý. (3) Upgrade node theo từng nhóm — drain từng node, upgrade AMI/image, uncordon. (4) Verify workload hoạt động bình thường sau mỗi bước. Với EKS, có thể dùng eksctl upgrade cluster hoặc tạo node group mới rồi migrate workload sang.

**Q: Bạn xử lý Spot Node bị thu hồi đột ngột thế nào?**

> Cần đảm bảo workload tolerant với disruption: (1) Chạy ít nhất 2 replica cho mỗi service quan trọng. (2) Cấu hình PodDisruptionBudget — Ngân Sách Gián Đoạn Pod — để đảm bảo luôn có đủ replica khi node bị thu hồi. (3) Dùng node affinity để phân tán replica qua nhiều AZ — Availability Zone — Vùng Khả Dụng. (4) Với Spot trên AWS, bật Cluster Autoscaler với mixed instance policy để fallback sang On-Demand khi Spot không sẵn.

**Q: So sánh Workload Identity trên GKE và IRSA trên EKS?**

> Cả hai đều dùng OIDC federation để cấp quyền cloud API cho Pod không cần secret. Khác biệt: IRSA cần annotate ServiceAccount với IAM Role ARN và cấu hình OIDC provider trên AWS IAM — phức tạp hơn. Workload Identity trên GKE hoặc AKS có quy trình tương tự nhưng tool hỗ trợ tốt hơn và ít bước hơn. EKS Pod Identity (thế hệ mới) đơn giản hoá IRSA đáng kể — không cần cấu hình OIDC provider thủ công.

### Câu Hỏi Thiết Kế

**Q: Thiết kế cluster production EKS với yêu cầu HA và bảo mật cao?**

```
Đề xuất kiến trúc:
- Multi-AZ node group: phân bổ node qua 3 AZ
- Managed Node Group cho node production, Fargate cho batch job
- VPC riêng với private subnet cho node; public subnet chỉ cho Load Balancer
- IRSA cho mọi ServiceAccount cần truy cập AWS API
- Pod Security Admission Restricted profile cho namespace production
- Network Policy phân lập namespace
- AWS ALB Ingress Controller thay vì NGINX (tích hợp native với AWS)
- Cluster Autoscaler với mixed On-Demand + Spot theo tỷ lệ 30/70
- Secrets Manager + External Secrets Operator thay vì K8s Secret thuần
- OIDC + RBAC kết hợp với AWS SSO cho quyền developer
```

---

## Danh Sách File Trong Module

| File                                    | Nội Dung                                                    |
| --------------------------------------- | ----------------------------------------------------------- |
| [README.md](./README.md)                | Tổng quan và so sánh các nền tảng (file này)               |
| [eks.md](./1-eks.md)                      | Amazon EKS — IRSA, Fargate, add-on, node group             |
| [gke.md](./2-gke.md)                      | Google GKE — Autopilot, Workload Identity, GCS              |
| [aks.md](./3-aks.md)                      | Azure AKS — Azure AD, node pool, Windows container         |
| [self-managed.md](./4-self-managed.md)    | kubeadm, k3s, RKE2 — cài đặt và vận hành thủ công         |

---

**Cập Nhật Lần Cuối:** 2026-05-10
**Phiên Bản:** 1.0
