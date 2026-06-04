# ⚡ AWS Compute Services — Lộ Trình Học Toàn Diện

> Hướng dẫn đầy đủ về AWS Compute Services — từ EC2 (Elastic Compute Cloud — Máy Chủ Ảo Đám Mây) và Auto Scaling (Tự Động Co Giãn) đến Serverless với Lambda (Hàm Không Máy Chủ), Container với ECS/EKS (Dịch Vụ Container/Kubernetes), và các chiến lược tối ưu chi phí vận hành.

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Dịch Vụ](#tổng-quan-các-dịch-vụ)
4. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
5. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
6. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng — Fundamentals (Tuần 1-2)**

- [ ] EC2 (Elastic Compute Cloud — Máy Chủ Ảo) — Instance Types, AMI, Key Pairs
- [ ] Security Groups (Nhóm Bảo Mật) & IAM Instance Profiles (Hồ Sơ Quyền Truy Cập Máy Chủ)
- [ ] EBS (Elastic Block Store — Lưu Trữ Khối) & Storage Options (Tùy Chọn Lưu Trữ)
- [ ] User Data (Dữ Liệu Khởi Tạo) & Instance Metadata (Siêu Dữ Liệu Máy Chủ)
- [ ] Pricing Models — On-Demand, Reserved, Spot (Mô Hình Định Giá)

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi — Core Skills (Tuần 3-6)**

- [ ] Auto Scaling Groups — ASG (Nhóm Tự Động Co Giãn) & Launch Templates (Mẫu Khởi Chạy)
- [ ] Lambda (Serverless — Không Máy Chủ) — Functions, Triggers, Concurrency (Đồng Thời)
- [ ] ECS (Elastic Container Service — Dịch Vụ Container) & Fargate (Serverless Container)
- [ ] Elastic Load Balancer Integration (Tích Hợp Cân Bằng Tải)

### **Giai Đoạn 3: Nâng Cao — Advanced (Tuần 7-10)**

- [ ] EKS (Elastic Kubernetes Service — Dịch Vụ Kubernetes) & Node Groups (Nhóm Node)
- [ ] Spot Instances Strategy (Chiến Lược Sử Dụng Spot) & Savings Plans (Kế Hoạch Tiết Kiệm)
- [ ] High Availability (Tính Sẵn Sàng Cao) — Multi-AZ, Multi-Region
- [ ] Monitoring & Observability — CloudWatch, X-Ray (Giám Sát & Quan Sát)

### **Giai Đoạn 4: Chuyên Sâu — Specialization (Tuần 11+)**

- [ ] Kiến trúc Serverless (Không Máy Chủ) nâng cao với Lambda & API Gateway
- [ ] Container Orchestration (Điều Phối Container) với ECS/EKS
- [ ] Cost Optimization (Tối Ưu Chi Phí) & Rightsizing (Chọn Đúng Kích Cỡ)
- [ ] Infrastructure as Code (Hạ Tầng Dưới Dạng Mã) với Terraform/CDK

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                              | Ưu Tiên | Thời Gian | Trạng Thái |
| ------------------------------------- | ------- | --------- | ---------- |
| **EC2 — Instance Management**         | ⭐⭐⭐  | 2 tuần    | -          |
| **Auto Scaling — ASG & Policies**     | ⭐⭐⭐  | 1 tuần    | -          |
| **Lambda — Serverless Functions**     | ⭐⭐⭐  | 2 tuần    | -          |
| **ECS — Container Service**           | ⭐⭐⭐  | 2 tuần    | -          |
| **IAM Roles & Instance Profiles**     | ⭐⭐⭐  | 1 tuần    | -          |
| **High Availability & Fault Tolerance** | ⭐⭐⭐ | 1 tuần   | -          |
| **EKS — Kubernetes Service**          | ⭐⭐    | 2 tuần    | -          |
| **Cost Optimization & Pricing**       | ⭐⭐    | 1 tuần    | -          |
| **CloudWatch Monitoring**             | ⭐⭐    | 1 tuần    | -          |
| **Spot Instances & Savings Plans**    | ⭐⭐    | 1 tuần    | -          |

---

## 🗂️ Tổng Quan Các Dịch Vụ

### So Sánh Nhanh — Chọn Dịch Vụ Phù Hợp

| Dịch Vụ              | Mô Hình            | Tốt Nhất Cho                          | Quản Lý Server |
| -------------------- | ------------------ | ------------------------------------- | -------------- |
| **EC2**              | IaaS (Hạ Tầng)     | Toàn quyền kiểm soát, workload cố định | Bạn quản lý   |
| **Lambda**           | FaaS (Hàm)         | Event-driven, burst traffic, API       | AWS quản lý    |
| **ECS + EC2**        | CaaS (Container)   | Container workload, kiểm soát server  | Bạn quản lý    |
| **ECS + Fargate**    | Serverless CaaS    | Container không muốn quản lý server   | AWS quản lý    |
| **EKS**              | Kubernetes thuần   | Kubernetes workload phức tạp          | Chia sẻ        |
| **Elastic Beanstalk**| PaaS (Nền Tảng)    | Deploy nhanh, ít cấu hình             | AWS quản lý    |
| **Lightsail**        | Simplified VPS     | Blog, app nhỏ, học tập                | Đơn giản hóa   |
| **Batch**            | Managed Batch      | Xử lý hàng loạt, HPC                  | AWS quản lý    |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. EC2 Fundamentals — Nền Tảng EC2** (`01-ec2-fundamentals/`)

- Instance Types (Loại Máy Chủ) — General Purpose, Compute/Memory/Storage/GPU Optimized
- AMI (Amazon Machine Image — Ảnh Máy Ảo) & Snapshot (Ảnh Chụp)
- EBS (Elastic Block Store) — gp3, io2, st1, sc1 Volume Types
- Security Groups (Nhóm Bảo Mật) & Key Pairs (Cặp Khóa SSH)
- User Data (Dữ Liệu Khởi Tạo) & Instance Metadata Service — IMDS

### 📁 **2. EC2 Nâng Cao & Auto Scaling** (`02-auto-scaling/`)

- **Auto Scaling Groups — ASG** — Tự Động Co Giãn Theo Nhóm
- Launch Templates (Mẫu Khởi Chạy) vs Launch Configurations (Cấu Hình Khởi Chạy)
- Scaling Policies — Target Tracking, Step, Scheduled (Chính Sách Co Giãn)
- Spot Instances (Máy Chủ Tạm Thời) & Mixed Instance Policy (Chính Sách Hỗn Hợp)
- Placement Groups — Cluster, Spread, Partition (Nhóm Vị Trí Vật Lý)

### 📁 **3. Serverless — Lambda** (`03-serverless-lambda/`)

- Lambda Functions (Hàm Lambda) — Runtime, Handler, Execution Role
- Event Sources (Nguồn Sự Kiện) — API Gateway, SQS, SNS, S3, DynamoDB Streams
- Lambda Layers (Lớp Lambda) & Extensions (Mở Rộng)
- Concurrency (Đồng Thời) — Reserved, Provisioned Concurrency
- Cold Start (Khởi Động Lạnh) Optimization & SnapStart

### 📁 **4. Containers — ECS** (`04-containers-ecs/`)

- ECS (Elastic Container Service) Architecture — Cluster, Service, Task
- Task Definitions (Định Nghĩa Task) — Container specs, CPU/Memory limits
- ECS Service — Deployment strategies, Auto Scaling
- **Fargate** — Serverless Container Engine (Công Cụ Container Không Máy Chủ)
- ECS Networking — awsvpc mode, Service Connect, Service Discovery

### 📁 **5. Containers — EKS** (`05-containers-eks/`)

- EKS (Elastic Kubernetes Service) Architecture — Control Plane, Data Plane
- Node Groups (Nhóm Node) — Managed, Self-managed, Fargate Profiles
- EKS Networking — VPC CNI (Container Network Interface), CoreDNS, kube-proxy
- EKS Storage — EBS CSI Driver, EFS CSI Driver
- EKS Security — RBAC (Role-Based Access Control), IRSA (IAM Roles for Service Accounts)

### 📁 **6. High Availability — Tính Sẵn Sàng Cao** (`06-high-availability/`)

- Multi-AZ (Đa Vùng Khả Dụng) Design & Multi-Region (Đa Vùng) Strategy
- Auto Scaling Policies — Reactive, Predictive, Scheduled (Chính Sách Phản Ứng/Dự Báo/Lịch)
- Health Checks (Kiểm Tra Sức Khỏe) & Automatic Recovery (Tự Động Phục Hồi)
- Blue/Green Deployment (Triển Khai Xanh/Lam) & Rolling Updates (Cập Nhật Cuộn)
- Capacity Planning (Lập Kế Hoạch Năng Lực) & Load Testing

### 📁 **7. Cost Optimization — Tối Ưu Chi Phí** (`07-cost-optimization/`)

- Pricing Models (Mô Hình Định Giá) — On-Demand, Reserved, Spot, Savings Plans
- Spot Instance Strategy (Chiến Lược Spot) — Interruption Handling, Diversification
- Rightsizing (Chọn Đúng Kích Cỡ) với AWS Compute Optimizer
- Savings Plans (Kế Hoạch Tiết Kiệm) — Compute, EC2 Instance, SageMaker
- Cost Monitoring với AWS Cost Explorer, Budgets, Trusted Advisor

### 📁 **8. Security — Bảo Mật** (`08-security/`)

- IAM Roles (Vai Trò) & Instance Profiles (Hồ Sơ Máy Chủ) — Least Privilege
- AWS Systems Manager — SSM Session Manager (Quản Lý Phiên), Patch Manager
- Secrets Management (Quản Lý Bí Mật) — Secrets Manager, Parameter Store
- Encryption at Rest (Mã Hóa Khi Lưu) — EBS, S3, EFS encryption
- Compliance (Tuân Thủ) & Patching (Vá Lỗi) — Inspector, Security Hub

### 📁 **9. Monitoring & Observability — Giám Sát** (`09-monitoring/`)

- CloudWatch Metrics (Chỉ Số) — EC2, Lambda, ECS/EKS metrics
- CloudWatch Logs (Nhật Ký) — Log Groups, Log Insights, Metric Filters
- CloudWatch Alarms (Cảnh Báo) & Auto Scaling triggers
- AWS X-Ray (Theo Dõi Phân Tán) — Distributed Tracing, Service Map
- Observability Checklist (Checklist Quan Sát) — SLI, SLO, SLA

### 📁 **10. Troubleshooting — Xử Lý Sự Cố** (`10-troubleshooting/`)

- EC2 Connectivity Issues (Vấn Đề Kết Nối) — SSH/RDP, Security Groups, Status Checks
- Auto Scaling Issues — Launch failures, Scaling imbalance
- Lambda Debugging (Gỡ Lỗi Lambda) — Timeouts, Memory, Permissions, Cold starts
- Container Issues (Vấn Đề Container) — Task failures, OOM, Networking
- Production Checklist (Checklist Sản Xuất) — Trước khi go-live

### 📁 **11. Interview Prep — Chuẩn Bị Phỏng Vấn** (`11-interview-prep/`)

- Top 20 AWS Compute Interview Questions (20 Câu Hỏi Phỏng Vấn Hàng Đầu)
- System Design Scenarios (Kịch Bản Thiết Kế Hệ Thống) — Thực tế
- STAR Stories (Câu Chuyện Theo Phương Pháp STAR) — Incident Templates
- Hands-on Exercises (Bài Tập Thực Hành) — Step-by-step Labs
- Kế Hoạch Học 90 Ngày có lộ trình chi tiết

---

## 🎓 Theo Nền Tảng Compute

### **EC2 — Elastic Compute Cloud**

```
Điểm Mạnh: Toàn quyền kiểm soát, linh hoạt, hỗ trợ mọi workload
Phù Hợp Cho: Workload cố định, ứng dụng legacy, database server, gaming
Học Ở: 01-ec2-fundamentals/, 02-auto-scaling/, 08-security/
```

### **Lambda — Serverless**

```
Điểm Mạnh: Không quản lý server, pay-per-use, tự động co giãn
Phù Hợp Cho: Event-driven, API backend, cron jobs, data processing
Học Ở: 03-serverless-lambda/, 06-high-availability/, 09-monitoring/
```

### **ECS + Fargate — Container Serverless**

```
Điểm Mạnh: Container thuần, không quản lý server, tích hợp AWS tốt
Phù Hợp Cho: Microservices, containerized apps, CI/CD pipelines
Học Ở: 04-containers-ecs/, 06-high-availability/, 08-security/
```

### **EKS — Kubernetes**

```
Điểm Mạnh: Kubernetes chuẩn, di động, hệ sinh thái phong phú
Phù Hợp Cho: Kubernetes workload phức tạp, multi-cloud portability
Học Ở: 05-containers-eks/, 06-high-availability/, 08-security/
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề                        | Thư Mục                                                                               | Ưu Tiên          |
| ----------------------------- | ------------------------------------------------------------------------------------- | ---------------- |
| Bắt đầu từ đây                | [README.md](./README.md)                                                              | Start here       |
| EC2 cơ bản                    | [01-ec2-fundamentals/](./01-ec2-fundamentals/)                                        | Nền tảng         |
| Auto Scaling                  | [02-auto-scaling/](./02-auto-scaling/)                                                | Quan trọng       |
| Serverless Lambda             | [03-serverless-lambda/](./03-serverless-lambda/)                                      | Phổ biến         |
| ECS & Fargate                 | [04-containers-ecs/](./04-containers-ecs/)                                            | Phổ biến         |
| EKS Kubernetes                | [05-containers-eks/](./05-containers-eks/)                                            | Nâng cao         |
| Câu hỏi phỏng vấn             | [11-interview-prep/1-INTERVIEW_GUIDE.md](./11-interview-prep/1-INTERVIEW_GUIDE.md)   | Trước phỏng vấn  |

---

## 📊 Ma Trận Kỹ Năng

### Người Mới — Beginner (0-1 năm kinh nghiệm)

- [ ] Khởi chạy EC2 instance từ AWS Console
- [ ] Cấu hình Security Groups cho EC2
- [ ] Tạo và gán IAM Role cho EC2
- [ ] Deploy ứng dụng đơn giản lên EC2
- [ ] Hiểu khái niệm On-Demand vs Reserved vs Spot

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Thiết kế Auto Scaling Group với Target Tracking Policy
- [ ] Viết và deploy Lambda Function với event triggers
- [ ] Tạo ECS Service với Fargate launch type
- [ ] Cấu hình CloudWatch Alarms & Log Groups
- [ ] Thiết lập Blue/Green Deployment với CodeDeploy

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Kiến trúc EKS cluster với managed node groups
- [ ] Tối ưu chi phí với Spot + On-Demand mixed fleet
- [ ] Thiết kế Serverless architecture phức tạp
- [ ] Triển khai multi-region active-active compute
- [ ] Implement FinOps (Quản Lý Tài Chính Đám Mây) practices

---

## 🚀 Bắt Đầu

### Bước 1: Đặt Mục Tiêu Học Tập

```
Chọn hướng đi phù hợp:
- Generalist: Nắm vững EC2 + Lambda + ECS
- Specialist: Chuyên sâu EC2/Auto Scaling hoặc Serverless hoặc Kubernetes
- Cloud-focused: AWS Solutions Architect Associate/Professional
```

### Bước 2: Chuẩn Bị Môi Trường Lab

```bash
# Cài đặt AWS CLI (Command Line Interface — Giao Diện Dòng Lệnh)
aws configure

# Kiểm tra kết nối
aws sts get-caller-identity

# Tạo key pair cho EC2
aws ec2 create-key-pair --key-name my-lab-key --output text > my-lab-key.pem
chmod 400 my-lab-key.pem
```

### Bước 3: Học + Thực Hành

```
1. Đọc module lý thuyết (30 phút)
2. Thực hành trên AWS Console (30-45 phút)
3. Thử dùng AWS CLI cho cùng task (15 phút)
4. Xem lại checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện STAR:
- Situation (Tình Huống): Context của vấn đề
- Task (Nhiệm Vụ): Bạn cần làm gì
- Action (Hành Động): Bạn đã làm gì cụ thể
- Result (Kết Quả): Kết quả đạt được & bài học
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Nên Đọc

- **"AWS Certified Solutions Architect Study Guide"** — Toàn diện về AWS
- **"Cloud Native Patterns"** by Cornelia Davis — Kiến trúc cloud native
- **"Serverless Architectures on AWS"** — Chuyên sâu Lambda
- **"Kubernetes in Action"** — Nếu học EKS

### Tài Liệu Chính Thức AWS

- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [ECS Developer Guide](https://docs.aws.amazon.com/ecs/)
- [EKS User Guide](https://docs.aws.amazon.com/eks/)

### Blog & Nguồn Học Thêm

- AWS Architecture Blog
- Last Week in AWS (newsletter)
- AWS re:Invent talks (YouTube)
- A Cloud Guru / Udemy AWS courses

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Hàng Đầu Theo Danh Mục

#### EC2 & Auto Scaling

- [ ] Giải thích sự khác biệt giữa các loại EC2 instance (instance types)
- [ ] Khi nào dùng Reserved Instance vs Spot vs On-Demand?
- [ ] Mô tả cách thiết kế Auto Scaling Group production-ready
- [ ] Placement Group (Nhóm Vị Trí) — Cluster vs Spread vs Partition khi nào dùng?

#### Serverless & Lambda

- [ ] Cold start là gì? Cách giảm thiểu cold start?
- [ ] Lambda concurrency (đồng thời) hoạt động thế nào? Reserved vs Provisioned?
- [ ] Thiết kế event-driven architecture với Lambda
- [ ] Lambda vs ECS Fargate — khi nào chọn cái nào?

#### Containers — ECS/EKS

- [ ] ECS vs EKS — trade-offs và khi nào chọn cái nào?
- [ ] Fargate vs EC2 launch type trong ECS?
- [ ] Giải thích EKS control plane và data plane
- [ ] IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho Service Accounts) hoạt động thế nào?

#### High Availability & Cost

- [ ] Thiết kế HA architecture cho web application
- [ ] Chiến lược Spot Instance (Máy Chủ Tạm Thời) để tránh interruption?
- [ ] Rightsizing (Chọn Đúng Kích Cỡ) EC2 — quy trình như thế nào?
- [ ] Savings Plans vs Reserved Instances — khác nhau ở đâu?

Xem `11-interview-prep/` để có hướng dẫn đầy đủ Q&A.

---

## ✅ Checklist Tự Đánh Giá

Trước khi phỏng vấn hoặc nhận vai trò mới, xác nhận:

- [ ] Có thể giải thích EC2 instance lifecycle (vòng đời máy chủ) từ đầu đến cuối
- [ ] Có thể thiết kế Auto Scaling với các chính sách phù hợp
- [ ] Có thể viết và deploy Lambda function
- [ ] Có thể tạo ECS service với Fargate từ đầu
- [ ] Có thể phân tích CloudWatch metrics để chẩn đoán vấn đề
- [ ] Có thể giải thích trade-offs giữa EC2 / Lambda / ECS / EKS
- [ ] Có thể tính toán và tối ưu chi phí Compute
- [ ] Có thể thiết kế HA architecture với Auto Scaling + Load Balancer
- [ ] Có thể xử lý sự cố EC2 / Lambda / Container phổ biến
- [ ] Có thể giải thích security best practices (IAM, SSM, Secrets Manager)

---

## 📞 Hỗ Trợ & Tài Nguyên

### Học Tập

- [AWS Free Tier](https://aws.amazon.com/free/) — Thực hành miễn phí
- [AWS Skill Builder](https://skillbuilder.aws/) — Khóa học chính thức
- [AWS Well-Architected Labs](https://wellarchitectedlabs.com/) — Labs thực hành

### Công Cụ

- **AWS Console** — Giao diện web quản lý
- **AWS CLI** — Dòng lệnh tự động hóa
- **AWS CDK** — Infrastructure as Code bằng ngôn ngữ lập trình
- **Terraform** — Infrastructure as Code đa cloud
- **AWS Compute Optimizer** — Gợi ý rightsizing

### Cộng Đồng

- r/aws (Reddit)
- AWS Developer Forums
- AWS re:Post
- AWS User Groups Vietnam

---

## 📋 Cách Sử Dụng Hướng Dẫn Này

### Cho Mục Đích Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Thực hành hands-on sau mỗi module
4. Xây dựng portfolio project thực tế

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào [11-interview-prep/](./11-interview-prep/)
2. Nắm sâu EC2 + Auto Scaling + Lambda (luôn được hỏi)
3. Chuẩn bị incident stories theo phương pháp STAR
4. Luyện giải thích trade-offs giữa các dịch vụ

### Cho Công Việc Thực Tế

1. Tham khảo [10-troubleshooting/](./10-troubleshooting/) khi có sự cố
2. Dùng [07-cost-optimization/](./07-cost-optimization/) để tiết kiệm chi phí
3. Áp dụng [08-security/](./08-security/) cho bảo mật
4. Kiểm tra [10-troubleshooting/production-checklist.md](./10-troubleshooting/5-production-checklist.md) trước go-live

---

## 🗺️ Các Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-ec2-fundamentals/
├─ 4️⃣  Tạo AWS Free Tier account và thực hành
├─ 5️⃣  Hoàn thành exercises cho từng chủ đề
├─ 6️⃣  Xây dựng portfolio project (multi-tier app trên AWS)
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
**Maintainer:** Backend Interview Prep
