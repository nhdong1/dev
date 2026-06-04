# 🌐 AWS Networking & Content Delivery — Lộ Trình Học Toàn Diện

> Hướng dẫn đầy đủ về AWS Networking and Content Delivery — từ nền tảng VPC (Virtual Private Cloud — Đám Mây Riêng Ảo) đến các dịch vụ phân phối nội dung nâng cao như CloudFront (Mạng Phân Phối Nội Dung) và Route 53 (Dịch Vụ DNS).

## 📚 Mục Lục

1. [Lộ Trình Học](#lộ-trình-học)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Các Chủ Đề](#tổng-quan-các-chủ-đề)
4. [Ma Trận Kỹ Năng](#ma-trận-kỹ-năng)
5. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🎯 Lộ Trình Học

### **Giai Đoạn 1: Nền Tảng — Fundamentals (Tuần 1-2)**

- [ ] VPC (Virtual Private Cloud — Đám Mây Riêng Ảo) & kiến trúc mạng cơ bản
- [ ] Subnets (Mạng Con) — Public và Private
- [ ] Route Tables (Bảng Định Tuyến) & Internet Gateway (Cổng Internet)
- [ ] NAT Gateway (Network Address Translation Gateway — Cổng Dịch Địa Chỉ Mạng)
- [ ] Security Groups (Nhóm Bảo Mật) & Network ACLs (Access Control Lists — Danh Sách Kiểm Soát Truy Cập)

### **Giai Đoạn 2: Kỹ Năng Cốt Lõi — Core Skills (Tuần 3-6)**

- [ ] ALB/NLB (Application/Network Load Balancer — Cân Bằng Tải Ứng Dụng/Mạng)
- [ ] Route 53 — DNS (Domain Name System — Hệ Thống Tên Miền) & Routing Policies (Chính Sách Định Tuyến)
- [ ] CloudFront — CDN (Content Delivery Network — Mạng Phân Phối Nội Dung)
- [ ] VPN (Virtual Private Network — Mạng Riêng Ảo) & Direct Connect

### **Giai Đoạn 3: Nâng Cao — Advanced (Tuần 7-10)**

- [ ] Transit Gateway (Cổng Trung Chuyển) & VPC Peering (Kết Nối Ngang Hàng VPC)
- [ ] PrivateLink & VPC Endpoints (Điểm Cuối VPC)
- [ ] AWS WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web) & Shield
- [ ] Global Accelerator (Tăng Tốc Toàn Cầu)

### **Giai Đoạn 4: Chuyên Sâu — Specialization (Tuần 11+)**

- [ ] Kiến trúc Multi-Region (Đa Vùng) & Hybrid Cloud (Đám Mây Lai)
- [ ] Network Performance Optimization (Tối Ưu Hiệu Suất Mạng)
- [ ] Compliance (Tuân Thủ) & Network Security Best Practices
- [ ] Infrastructure as Code (Hạ Tầng Dưới Dạng Mã) với Terraform/CDK

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực                           | Ưu Tiên | Thời Gian | Trạng Thái |
| ---------------------------------- | ------- | --------- | ---------- |
| **VPC Design & Subnetting**        | ⭐⭐⭐  | 2 tuần    | -          |
| **Load Balancing — ALB/NLB**       | ⭐⭐⭐  | 1 tuần    | -          |
| **Route 53 & DNS**                 | ⭐⭐⭐  | 1 tuần    | -          |
| **CloudFront — CDN**               | ⭐⭐⭐  | 1 tuần    | -          |
| **Security Groups & Network ACL**  | ⭐⭐⭐  | 1 tuần    | -          |
| **VPN & Direct Connect**           | ⭐⭐⭐  | 1 tuần    | -          |
| **Transit Gateway**                | ⭐⭐    | 1 tuần    | -          |
| **WAF & Shield**                   | ⭐⭐    | 1 tuần    | -          |
| **PrivateLink & VPC Endpoints**    | ⭐⭐    | 1 tuần    | -          |
| **Global Accelerator**             | ⭐⭐    | 0.5 tuần  | -          |
| **Network Monitoring & Flow Logs** | ⭐⭐    | 1 tuần    | -          |

---

## 🗂️ Tổng Quan Các Chủ Đề

### 📁 **1. VPC Fundamentals — Nền Tảng VPC** (`01-vpc-fundamentals/`)

- VPC (Virtual Private Cloud) — Kiến trúc và thiết kế
- Subnets (Mạng Con) — Public, Private, Isolated
- CIDR (Classless Inter-Domain Routing — Định Tuyến Liên Miền Không Phân Lớp) & IP Planning
- Route Tables (Bảng Định Tuyến) & Internet Gateway (IGW)
- NAT Gateway vs NAT Instance — So sánh & lựa chọn
- VPC Peering (Kết Nối Ngang Hàng) & Resource Sharing

### 📁 **2. Security — Bảo Mật Mạng** (`02-security/`)

- Security Groups (Nhóm Bảo Mật) — Stateful Firewall (Tường Lửa Trạng Thái)
- Network ACLs (NACLs) — Stateless Firewall (Tường Lửa Phi Trạng Thái)
- WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web)
- AWS Shield Standard & Advanced — Chống DDoS (Distributed Denial of Service)
- GuardDuty (Phát Hiện Mối Đe Dọa) cho lưu lượng mạng
- Network Firewall (Tường Lửa Mạng) của AWS

### 📁 **3. Load Balancing — Cân Bằng Tải** (`03-load-balancing/`)

- ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng) — Layer 7
- NLB (Network Load Balancer — Cân Bằng Tải Mạng) — Layer 4
- CLB (Classic Load Balancer — Cân Bằng Tải Cổ Điển) — Legacy
- Target Groups (Nhóm Mục Tiêu) & Health Checks (Kiểm Tra Sức Khỏe)
- Listener Rules (Quy Tắc Lắng Nghe) & Routing Strategies
- SSL/TLS Termination (Kết Thúc SSL/TLS) & Certificate Management (Quản Lý Chứng Chỉ)

### 📁 **4. DNS & Route 53** (`04-dns-route53/`)

- DNS (Domain Name System) — Nguyên lý hoạt động
- Hosted Zones (Vùng Lưu Trữ) — Public & Private
- Record Types (Loại Bản Ghi) — A, AAAA, CNAME, MX, TXT, Alias
- Routing Policies (Chính Sách Định Tuyến):
  - Simple (Đơn Giản)
  - Weighted (Có Trọng Số)
  - Latency-based (Dựa Trên Độ Trễ)
  - Failover (Chuyển Đổi Dự Phòng)
  - Geolocation (Dựa Trên Vị Trí Địa Lý)
  - Geoproximity (Dựa Trên Khoảng Cách Địa Lý)
  - Multi-value Answer (Đa Giá Trị)
- Health Checks & DNS Failover

### 📁 **5. CloudFront — CDN** (`05-cdn-cloudfront/`)

- CloudFront Architecture (Kiến Trúc) — Edge Locations (Điểm Biên)
- Distributions (Phân Phối) — Web & RTMP
- Origins (Nguồn Gốc) — S3, ALB, Custom HTTP
- Cache Behaviors (Hành Vi Cache) & Cache Policies (Chính Sách Cache)
- OAC/OAI (Origin Access Control/Identity — Kiểm Soát Truy Cập Nguồn Gốc)
- Lambda@Edge & CloudFront Functions
- Invalidations (Vô Hiệu Cache) & TTL (Time To Live — Thời Gian Sống)
- Price Classes (Hạng Giá) & Cost Optimization (Tối Ưu Chi Phí)

### 📁 **6. Connectivity — Kết Nối Mạng** (`06-connectivity/`)

- Site-to-Site VPN (VPN Địa Điểm-đến-Địa Điểm)
- Client VPN (VPN Khách Hàng) — Remote Access
- AWS Direct Connect — Kết Nối Chuyên Dụng
- Direct Connect Gateway (Cổng Direct Connect)
- Transit Gateway (Cổng Trung Chuyển) — Hub-and-Spoke Architecture
- VPC Peering vs Transit Gateway — So sánh & lựa chọn

### 📁 **7. Advanced Networking — Mạng Nâng Cao** (`07-advanced-networking/`)

- VPC Endpoints (Điểm Cuối VPC) — Gateway & Interface
- AWS PrivateLink — Kết Nối Dịch Vụ Riêng Tư
- Global Accelerator (Tăng Tốc Toàn Cầu) — Anycast Routing
- Elastic IP (IP Đàn Hồi) & Elastic Network Interface — ENI
- Prefix Lists (Danh Sách Tiền Tố) & Managed Prefix Lists
- IPv6 trong VPC — Dual-stack Architecture

### 📁 **8. Monitoring — Giám Sát Mạng** (`08-monitoring/`)

- VPC Flow Logs (Nhật Ký Luồng VPC) — Phân Tích Lưu Lượng
- CloudWatch Metrics (Số Liệu CloudWatch) cho Networking
- Network Access Analyzer (Bộ Phân Tích Truy Cập Mạng)
- AWS Config Rules cho Network Compliance
- CloudTrail (Nhật Ký API) cho Network Changes
- Reachability Analyzer (Bộ Phân Tích Khả Năng Tiếp Cận)

### 📁 **9. Troubleshooting — Xử Lý Sự Cố** (`09-troubleshooting/`)

- Kết Nối Không Thành Công — Checklist Chẩn Đoán
- Security Group & NACL Debugging (Gỡ Lỗi)
- Route Table Issues (Vấn Đề Bảng Định Tuyến)
- DNS Resolution Problems (Vấn Đề Phân Giải DNS)
- Load Balancer Health Check Failures
- CloudFront Cache Issues
- Network Performance Bottlenecks (Điểm Nghẽn Hiệu Suất)

### 📁 **10. Interview Prep — Chuẩn Bị Phỏng Vấn** (`10-interview-prep/`)

- Top 20 Câu Hỏi Phỏng Vấn AWS Networking
- System Design Scenarios (Kịch Bản Thiết Kế Hệ Thống)
- Incident Stories theo phương pháp STAR
- Thực Hành Thiết Kế Kiến Trúc Mạng
- Kế Hoạch Học 90 Ngày

---

## 🎓 Theo Cấp Độ Dịch Vụ AWS

### **VPC & Networking Core**

```
Dịch vụ chính: VPC, Subnets, Route Tables, IGW, NAT Gateway
Phù hợp cho: Mọi kiến trúc AWS production
Học trong: 01-vpc-fundamentals/, 02-security/
```

### **Load Balancing & High Availability**

```
Dịch vụ chính: ALB, NLB, Target Groups, Auto Scaling
Phù hợp cho: Web applications, microservices, API backends
Học trong: 03-load-balancing/
```

### **DNS & Global Routing**

```
Dịch vụ chính: Route 53, CloudFront, Global Accelerator
Phù hợp cho: Multi-region apps, global user base, DR setup
Học trong: 04-dns-route53/, 05-cdn-cloudfront/, 07-advanced-networking/
```

### **Hybrid & Multi-Cloud Connectivity**

```
Dịch vụ chính: Direct Connect, VPN, Transit Gateway
Phù hợp cho: Enterprise hybrid cloud, on-premises integration
Học trong: 06-connectivity/
```

### **Security & Compliance**

```
Dịch vụ chính: WAF, Shield, Network Firewall, GuardDuty
Phù hợp cho: Production security, compliance frameworks (PCI-DSS, HIPAA)
Học trong: 02-security/
```

---

## 🔗 Liên Kết Nhanh

| Chủ Đề             | Thư Mục                                        | Ưu Tiên         |
| ------------------ | ---------------------------------------------- | --------------- |
| Bắt đầu từ đây     | [README.md](./README.md)                       | Trước tiên      |
| Toàn bộ chỉ mục    | [INDEX.md](./INDEX.md)                         | Điều hướng      |
| VPC cơ bản         | [01-vpc-fundamentals/](./01-vpc-fundamentals/) | Thiết yếu       |
| Load Balancer      | [03-load-balancing/](./03-load-balancing/)     | Quan trọng      |
| CloudFront CDN     | [05-cdn-cloudfront/](./05-cdn-cloudfront/)     | Quan trọng      |
| Chuẩn bị phỏng vấn | [10-interview-prep/](./10-interview-prep/)     | Trước phỏng vấn |

---

## 📊 Ma Trận Kỹ Năng

### Người Mới Bắt Đầu — Beginner (0-1 năm kinh nghiệm)

- [ ] Hiểu khái niệm VPC, Subnet, Route Table
- [ ] Cấu hình Security Groups cơ bản
- [ ] Tạo và quản lý Internet Gateway & NAT Gateway
- [ ] Sử dụng ALB (Application Load Balancer) đơn giản
- [ ] Đăng ký domain và cấu hình DNS trên Route 53

### Trung Cấp — Intermediate (1-3 năm kinh nghiệm)

- [ ] Thiết kế VPC multi-tier (3 lớp: Public/Private/Database)
- [ ] Cấu hình CloudFront với custom origins và cache behaviors
- [ ] Triển khai VPN Site-to-Site hoặc Direct Connect
- [ ] Sử dụng Route 53 với Weighted & Failover routing policies
- [ ] Phân tích VPC Flow Logs để troubleshoot kết nối
- [ ] Cấu hình WAF rules cho ứng dụng web

### Nâng Cao — Advanced (3-5+ năm kinh nghiệm)

- [ ] Thiết kế kiến trúc Transit Gateway hub-and-spoke
- [ ] Tối ưu CloudFront cho performance & cost
- [ ] Xây dựng network security-in-depth với WAF, Shield Advanced, GuardDuty
- [ ] Thiết kế multi-region active-active architecture
- [ ] Tự động hóa network provisioning với Terraform/CDK
- [ ] Phân tích và giải quyết network performance bottlenecks

---

## 🚀 Bắt Đầu Nhanh

### Bước 1: Xác Định Mục Tiêu Học

```
Lựa chọn hướng đi phù hợp:
- Cloud Engineer tổng quát (toàn bộ dịch vụ AWS Networking)
- DevOps/Platform Engineer (VPC, ALB, VPN, Transit Gateway)
- Solutions Architect (thiết kế kiến trúc, multi-region, CDN)
- Security Engineer (WAF, Shield, Network Firewall, GuardDuty)
```

### Bước 2: Thiết Lập Môi Trường Thực Hành

```bash
# Dùng AWS CLI để tạo VPC cơ bản
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Hoặc dùng Terraform để tạo lab environment
terraform init && terraform apply
```

### Bước 3: Học & Thực Hành

```
1. Đọc tài liệu lý thuyết (30 phút)
2. Xem AWS Console và khám phá dịch vụ (20 phút)
3. Thực hành hands-on trên AWS Free Tier (45-60 phút)
4. Vẽ sơ đồ kiến trúc từ bộ nhớ (15 phút)
5. Ôn lại checklist (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị câu chuyện theo STAR:
- Situation (Tình huống): Bối cảnh vấn đề mạng cụ thể
- Task (Nhiệm vụ): Yêu cầu kỹ thuật đặt ra
- Action (Hành động): Giải pháp thiết kế và triển khai
- Result (Kết quả): Số liệu cải thiện đo lường được
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Thiết Yếu

- **"AWS Certified Solutions Architect Study Guide"** — Jon Bonso & Adrian Cantrill
- **"Networking in AWS"** — Tài liệu chính thức AWS
- **"Cloud Architecture Patterns"** — Bill Wilder

### Tài Liệu Chính Thức AWS

- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/)
- [Amazon CloudFront Developer Guide](https://docs.aws.amazon.com/cloudfront/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/route53/)
- [Elastic Load Balancing Documentation](https://docs.aws.amazon.com/elasticloadbalancing/)

### Tài Nguyên Học Tập

- AWS re:Invent sessions về Networking (YouTube)
- A Cloud Guru / Pluralsight AWS Networking courses
- AWS Well-Architected Framework — Reliability Pillar

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Hàng Đầu Theo Danh Mục

#### VPC & Networking Cơ Bản

- [ ] Giải thích sự khác biệt giữa Security Group và Network ACL
- [ ] Thiết kế VPC cho ứng dụng 3-tier (3 lớp)
- [ ] NAT Gateway vs Internet Gateway — khi nào dùng cái nào?
- [ ] Giải thích CIDR notation và cách tính Subnets

#### Load Balancing

- [ ] ALB vs NLB — sự khác biệt và khi nào dùng?
- [ ] Cách cấu hình health check để đảm bảo high availability
- [ ] Giải thích sticky sessions (phiên dính) và ảnh hưởng đến scalability
- [ ] Cross-zone load balancing là gì?

#### DNS & CloudFront

- [ ] Route 53 routing policies — giải thích từng loại với use case
- [ ] Cách CloudFront cải thiện performance và giảm chi phí?
- [ ] Sự khác biệt giữa CNAME và Alias record trong Route 53?
- [ ] Cách bảo vệ S3 bucket chỉ cho phép truy cập qua CloudFront?

#### Kết Nối Hybrid

- [ ] Direct Connect vs VPN Site-to-Site — trade-offs
- [ ] Transit Gateway giải quyết vấn đề gì mà VPC Peering không làm được?
- [ ] PrivateLink vs VPC Endpoints — khi nào dùng cái nào?

#### Xử Lý Sự Cố Thực Tế

- [ ] Kể về một sự cố mạng bạn đã giải quyết (STAR)
- [ ] Cách debug khi EC2 instance không thể kết nối internet
- [ ] Phân tích VPC Flow Logs để tìm nguồn gốc traffic bất thường

Xem `10-interview-prep/` để có hướng dẫn Q&A đầy đủ.

---

## ✅ Danh Sách Tự Đánh Giá

Trước phỏng vấn hoặc khi đảm nhận vai trò mới, kiểm tra:

- [ ] Có thể thiết kế VPC với Public/Private subnets từ đầu
- [ ] Hiểu rõ sự khác biệt stateful/stateless firewall (Security Group vs NACL)
- [ ] Có thể giải thích tất cả Route 53 routing policies với use case
- [ ] Biết cách tối ưu CloudFront cache hit ratio
- [ ] Có thể thiết kế multi-region DR (Disaster Recovery) architecture
- [ ] Hiểu cách Transit Gateway kết nối nhiều VPCs và on-premises
- [ ] Biết khi nào dùng ALB, NLB, và CLB
- [ ] Có thể phân tích VPC Flow Logs để troubleshoot kết nối
- [ ] Hiểu WAF rules và cách chống các cuộc tấn công phổ biến
- [ ] Có thể tính toán CIDR blocks cho Subnets

---

## 📋 Cách Sử Dụng Tài Liệu Này

### Cho Mục Đích Tự Học

1. Bắt đầu với [Lộ Trình Học](#lộ-trình-học)
2. Đi qua từng giai đoạn theo thứ tự
3. Thực hành hands-on sau mỗi chủ đề
4. Xây dựng một dự án lab cuối khóa

### Cho Chuẩn Bị Phỏng Vấn

1. Tập trung vào [10-interview-prep/](./10-interview-prep/)
2. Nắm vững VPC, Load Balancing, Route 53, CloudFront
3. Chuẩn bị 2-3 incident stories theo định dạng STAR
4. Luyện tập vẽ sơ đồ kiến trúc trên giấy

### Cho Công Việc Thực Tế

1. Tham khảo [09-troubleshooting/](./09-troubleshooting/) khi gặp sự cố
2. Sử dụng [02-security/](./02-security/) để kiểm tra security posture
3. Theo dõi [08-monitoring/](./08-monitoring/) để thiết lập giám sát
4. Xem [06-connectivity/](./06-connectivity/) cho thiết kế hybrid network

---

## 🗺️ Các Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình học (Beginner / Intermediate / Advanced)
├─ 3️⃣  Bắt đầu với 01-vpc-fundamentals/
├─ 4️⃣  Thiết lập AWS Free Tier account để thực hành
├─ 5️⃣  Hoàn thành bài tập cho từng chủ đề
├─ 6️⃣  Xây dựng lab project: thiết kế VPC 3-tier production-ready
└─ 7️⃣  Chuẩn bị phỏng vấn với 10-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
