# 🔐 AWS Security, Identity & Compliance — Lộ Trình Học Tập

> Hướng dẫn toàn diện về các dịch vụ Bảo Mật, Định Danh và Tuân Thủ của AWS — từ nền tảng cơ bản đến vận hành nâng cao trong môi trường production.

## 📚 Mục Lục

1. [Lộ Trình Học Tập](#lộ-trình-học-tập)
2. [Năng Lực Cốt Lõi](#năng-lực-cốt-lõi)
3. [Tổng Quan Dịch Vụ](#tổng-quan-dịch-vụ)
4. [Chủ Đề Chi Tiết](#chủ-đề-chi-tiết)

---

## 🎯 Lộ Trình Học Tập

### **Giai Đoạn 1: Nền Tảng (Tuần 1–2)**

- [ ] IAM (Identity and Access Management — Quản Lý Định Danh và Truy Cập) cơ bản: Users, Groups, Roles, Policies
- [ ] Authentication (Xác Thực) vs Authorization (Phân Quyền)
- [ ] Least Privilege Principle (Nguyên Tắc Đặc Quyền Tối Thiểu)
- [ ] MFA (Multi-Factor Authentication — Xác Thực Đa Yếu Tố)
- [ ] AWS Shared Responsibility Model (Mô Hình Trách Nhiệm Chia Sẻ)

### **Giai Đoạn 2: Dịch Vụ Định Danh Nâng Cao (Tuần 3–5)**

- [ ] IAM Roles & Cross-Account Access (Truy Cập Liên Tài Khoản)
- [ ] STS (Security Token Service — Dịch Vụ Token Bảo Mật) & Temporary Credentials
- [ ] AWS Organizations & SCPs (Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ)
- [ ] AWS SSO / IAM Identity Center (Trung Tâm Định Danh)
- [ ] Cognito (Quản Lý Định Danh Người Dùng Ứng Dụng)

### **Giai Đoạn 3: Bảo Mật Hạ Tầng (Tuần 6–8)**

- [ ] KMS (Key Management Service — Dịch Vụ Quản Lý Khóa Mã Hóa)
- [ ] Secrets Manager & Parameter Store (Quản Lý Bí Mật)
- [ ] ACM (AWS Certificate Manager — Quản Lý Chứng Chỉ SSL/TLS)
- [ ] WAF (Web Application Firewall — Tường Lửa Ứng Dụng Web) & Shield
- [ ] VPC Security (Security Groups, NACLs, VPC Endpoints)

### **Giai Đoạn 4: Giám Sát & Tuân Thủ (Tuần 9–11)**

- [ ] CloudTrail (Nhật Ký Kiểm Toán API)
- [ ] AWS Config (Kiểm Soát Tuân Thủ Cấu Hình)
- [ ] GuardDuty (Phát Hiện Mối Đe Dọa Tự Động)
- [ ] Security Hub (Trung Tâm Bảo Mật Tổng Hợp)
- [ ] Inspector (Quét Lỗ Hổng Bảo Mật)

### **Giai Đoạn 5: Chuyên Sâu (Tuần 12+)**

- [ ] Macie (Phát Hiện Dữ Liệu Nhạy Cảm)
- [ ] Detective (Phân Tích Bảo Mật Nâng Cao)
- [ ] Network Firewall & Firewall Manager
- [ ] Compliance Frameworks (SOC2, PCI-DSS, HIPAA, GDPR)
- [ ] Well-Architected Security Pillar (Trụ Cột Bảo Mật Kiến Trúc Tốt)

---

## 🏢 Năng Lực Cốt Lõi

| Năng Lực | Mức Độ Ưu Tiên | Thời Gian | Trạng Thái |
|---|---|---|---|
| **IAM Fundamentals** (Nền Tảng IAM) | ⭐⭐⭐ | 2 tuần | - |
| **Encryption & KMS** (Mã Hóa & Quản Lý Khóa) | ⭐⭐⭐ | 1 tuần | - |
| **Secrets Management** (Quản Lý Bí Mật) | ⭐⭐⭐ | 1 tuần | - |
| **Monitoring & Auditing** (Giám Sát & Kiểm Toán) | ⭐⭐⭐ | 2 tuần | - |
| **Network Security** (Bảo Mật Mạng) | ⭐⭐⭐ | 1 tuần | - |
| **Threat Detection** (Phát Hiện Mối Đe Dọa) | ⭐⭐⭐ | 1 tuần | - |
| **Compliance & Governance** (Tuân Thủ & Quản Trị) | ⭐⭐⭐ | 2 tuần | - |
| **Identity Federation** (Liên Kết Định Danh) | ⭐⭐ | 1 tuần | - |
| **Incident Response** (Phản Hồi Sự Cố) | ⭐⭐ | 1 tuần | - |
| **Cost & Security Optimization** (Tối Ưu Bảo Mật & Chi Phí) | ⭐⭐ | 1 tuần | - |

---

## 🗂️ Tổng Quan Dịch Vụ

### 📁 **1. IAM Fundamentals** (`01-iam-fundamentals/`)

- **Users** (Người Dùng) — đại diện cho người hoặc ứng dụng cụ thể
- **Groups** (Nhóm) — tập hợp users để quản lý policy tập trung
- **Roles** (Vai Trò) — định danh tạm thời, không gắn với người dùng cố định
- **Policies** (Chính Sách) — tài liệu JSON định nghĩa quyền hạn
- **Permission Boundaries** (Ranh Giới Quyền Hạn) — giới hạn tối đa quyền của role/user
- **IAM Conditions** (Điều Kiện) — kiểm soát chi tiết dựa trên context

### 📁 **2. Identity Federation & SSO** (`02-identity-federation/`)

- **AWS IAM Identity Center** (cũ: SSO) — đăng nhập một lần cho nhiều tài khoản
- **SAML 2.0** (Security Assertion Markup Language) — liên kết với IdP doanh nghiệp
- **OIDC** (OpenID Connect) — liên kết với Google, GitHub, Okta...
- **Cognito User Pools** (Nhóm Người Dùng) — quản lý auth cho ứng dụng web/mobile
- **Cognito Identity Pools** (Nhóm Định Danh) — cấp quyền AWS tạm thời cho người dùng
- **Cross-Account Roles** (Role Liên Tài Khoản) — truy cập an toàn giữa các AWS account

### 📁 **3. AWS Organizations & Multi-Account** (`03-organizations/`)

- **AWS Organizations** — quản lý tập trung nhiều tài khoản AWS
- **SCPs** (Service Control Policies — Chính Sách Kiểm Soát Dịch Vụ) — giới hạn quyền ở cấp OU/account
- **OUs** (Organizational Units — Đơn Vị Tổ Chức) — phân cấp tài khoản theo môi trường/dự án
- **AWS Control Tower** — thiết lập landing zone đa tài khoản tự động
- **Delegated Administration** (Quản Trị Ủy Quyền) — phân chia quyền quản lý dịch vụ

### 📁 **4. Encryption & KMS** (`04-encryption-kms/`)

- **KMS** (Key Management Service) — tạo và quản lý CMKs (Customer Master Keys)
- **CMKs** (Customer Managed Keys — Khóa Do Khách Hàng Quản Lý) vs **AWS Managed Keys**
- **Envelope Encryption** (Mã Hóa Phong Bì) — DEKs được mã hóa bởi CMK
- **Key Rotation** (Xoay Vòng Khóa) — tự động thay đổi khóa định kỳ
- **CloudHSM** (Hardware Security Module — Module Bảo Mật Phần Cứng) — FIPS 140-2 Level 3
- **S3 SSE** (Server-Side Encryption) — mã hóa phía máy chủ với các tùy chọn khóa

### 📁 **5. Secrets & Certificate Management** (`05-secrets-certificates/`)

- **Secrets Manager** — lưu trữ, xoay vòng tự động credentials, API keys, DB passwords
- **Parameter Store** (SSM) — lưu cấu hình và bí mật theo cấp độ (Standard/Advanced)
- **ACM** (AWS Certificate Manager) — cấp phát và gia hạn tự động chứng chỉ SSL/TLS
- **ACM Private CA** (Private Certificate Authority) — CA nội bộ cho mạng nội bộ
- **Rotation Strategies** (Chiến Lược Xoay Vòng) — single vs multi-user rotation

### 📁 **6. Network Security** (`06-network-security/`)

- **Security Groups** (Nhóm Bảo Mật) — tường lửa stateful ở cấp instance
- **NACLs** (Network Access Control Lists) — tường lửa stateless ở cấp subnet
- **VPC Endpoints** (Điểm Cuối VPC) — kết nối dịch vụ AWS không qua internet
- **PrivateLink** — truy cập dịch vụ qua mạng nội bộ AWS
- **WAF** (Web Application Firewall) — bảo vệ ứng dụng khỏi OWASP Top 10
- **Shield** (Standard & Advanced) — bảo vệ chống DDoS (Distributed Denial of Service)
- **Network Firewall** — tường lửa stateful có thể lập trình theo luật

### 📁 **7. Monitoring, Auditing & Logging** (`07-monitoring-auditing/`)

- **CloudTrail** — ghi nhật ký mọi lời gọi API trong tài khoản AWS
- **CloudTrail Lake** — lưu trữ và truy vấn sự kiện dài hạn bằng SQL
- **AWS Config** — theo dõi thay đổi cấu hình tài nguyên và đánh giá tuân thủ
- **Config Rules** (Quy Tắc Config) — kiểm tra tự động theo tiêu chuẩn bảo mật
- **CloudWatch Logs Insights** — phân tích log bảo mật theo thời gian thực
- **Access Analyzer** (Bộ Phân Tích Truy Cập) — phát hiện quyền truy cập ngoài ý muốn

### 📁 **8. Threat Detection & Incident Response** (`08-threat-detection/`)

- **GuardDuty** — phát hiện mối đe dọa dựa trên ML từ CloudTrail, VPC Flow Logs, DNS
- **Security Hub** — tổng hợp và chuẩn hóa findings từ nhiều dịch vụ bảo mật
- **Inspector** — quét lỗ hổng CVE tự động trên EC2, Lambda, ECR images
- **Macie** (Macie — Phát Hiện Dữ Liệu Nhạy Cảm) — phát hiện PII/PHI trong S3
- **Detective** — điều tra nguyên nhân gốc rễ sự cố bảo mật bằng đồ thị quan hệ
- **Incident Response Playbooks** (Kịch Bản Phản Hồi Sự Cố) — quy trình xử lý chuẩn

### 📁 **9. Compliance & Governance** (`09-compliance-governance/`)

- **AWS Artifact** — tải báo cáo tuân thủ và thỏa thuận AWS
- **Audit Manager** — tự động thu thập bằng chứng tuân thủ (SOC2, PCI-DSS, HIPAA)
- **Config Conformance Packs** (Gói Tuân Thủ Config) — bộ quy tắc đóng gói sẵn
- **Security Hub Standards** — CIS Benchmarks, AWS Foundational Security Best Practices
- **Firewall Manager** — quản lý WAF, Security Groups, Shield tập trung đa tài khoản
- **GDPR/PCI-DSS/HIPAA** trên AWS — kiến trúc và dịch vụ liên quan

### 📁 **10. Advanced Topics** (`10-advanced/`)

- **Attribute-Based Access Control** — ABAC (Kiểm Soát Truy Cập Dựa Trên Thuộc Tính)
- **Zero Trust Architecture** (Kiến Trúc Không Tin Tưởng Mặc Định)
- **Privileged Access Management** — PAM (Quản Lý Truy Cập Đặc Quyền)
- **Data Perimeter** (Vành Đai Dữ Liệu) — kiểm soát ai/từ đâu truy cập data
- **Security Automation** (Tự Động Hóa Bảo Mật) — EventBridge + Lambda auto-remediation
- **Supply Chain Security** (Bảo Mật Chuỗi Cung Ứng) — Sigstore, SBOM, CodeArtifact

### 📁 **11. Interview Preparation** (`11-interview-prep/`)

- Top 25 câu hỏi phỏng vấn AWS Security
- System Design Scenarios (Kịch Bản Thiết Kế Hệ Thống Bảo Mật)
- Câu chuyện sự cố theo phương pháp STAR
- IAM Policy Troubleshooting (Gỡ Lỗi Chính Sách IAM)
- Security Architecture Patterns (Mẫu Kiến Trúc Bảo Mật)

---

## 🗺️ Sơ Đồ Dịch Vụ Theo Nhóm

### **Định Danh & Truy Cập (Identity & Access)**

```
IAM (Users/Groups/Roles/Policies)
    └── STS (Security Token Service) — phát hành temporary credentials
    └── IAM Identity Center (SSO) — đăng nhập một lần đa tài khoản
    └── Cognito — định danh người dùng ứng dụng
    └── Organizations + SCPs — quản trị đa tài khoản
```

### **Mã Hóa & Quản Lý Bí Mật (Encryption & Secrets)**

```
KMS (Key Management Service)
    └── CloudHSM — hardware-level key protection
    └── Secrets Manager — auto-rotation credentials
    └── Parameter Store — configuration & secrets hierarchy
    └── ACM — SSL/TLS certificate lifecycle
```

### **Phát Hiện & Phản Hồi (Detection & Response)**

```
GuardDuty — threat detection (ML-based)
    └── Security Hub — findings aggregation
    └── Inspector — vulnerability scanning
    └── Macie — sensitive data discovery
    └── Detective — security investigation graph
```

### **Giám Sát & Kiểm Toán (Monitoring & Audit)**

```
CloudTrail — API call logging
    └── CloudTrail Lake — long-term event store
    └── AWS Config — resource compliance tracking
    └── Access Analyzer — unintended access detection
    └── CloudWatch Logs — real-time log analysis
```

### **Bảo Mật Mạng (Network Security)**

```
Security Groups + NACLs — instance & subnet firewall
    └── WAF — web application protection
    └── Shield — DDoS protection
    └── Network Firewall — programmable stateful firewall
    └── Firewall Manager — centralized multi-account management
```

---

## 📊 Ma Trận Kỹ Năng (Skill Matrix)

### Người Mới Bắt Đầu (0–1 năm)

- [ ] Tạo được IAM user, group, role với policy phù hợp
- [ ] Hiểu sự khác biệt giữa authentication và authorization
- [ ] Bật MFA cho root account và IAM users
- [ ] Biết đọc và viết IAM policy cơ bản (JSON)
- [ ] Hiểu AWS Shared Responsibility Model

### Trung Cấp (1–3 năm)

- [ ] Thiết kế IAM role hierarchy cho ứng dụng đa tầng
- [ ] Cài đặt KMS encryption cho S3, RDS, EBS
- [ ] Cấu hình CloudTrail và phân tích log kiểm toán
- [ ] Triển khai Secrets Manager với auto-rotation
- [ ] Thiết lập GuardDuty và xử lý findings
- [ ] Thiết kế multi-account structure với Organizations

### Nâng Cao (3–5+ năm)

- [ ] Triển khai Zero Trust architecture trên AWS
- [ ] Thiết kế data perimeter và resource-based policies
- [ ] Xây dựng security automation pipeline
- [ ] Điều tra sự cố bảo mật với Detective & Security Hub
- [ ] Thiết kế kiến trúc tuân thủ PCI-DSS/HIPAA
- [ ] Thực hiện Well-Architected Security Review

---

## 🚀 Bắt Đầu Nhanh

### Bước 1: Xác Định Mục Tiêu

```
Chọn hướng đi:
- Security Engineer  — tập trung threat detection, incident response
- Cloud Architect    — tập trung IAM design, multi-account, compliance
- DevSecOps Engineer — tập trung automation, CI/CD security, secrets management
- Compliance Analyst — tập trung audit, governance, evidence collection
```

### Bước 2: Thiết Lập Môi Trường Lab

```bash
# Tạo AWS account riêng cho lab (không dùng production)
# Bật AWS Organizations để luyện tập multi-account
# Bật CloudTrail ngay từ đầu để quan sát mọi hành động

aws cloudtrail create-trail \
  --name lab-audit-trail \
  --s3-bucket-name my-audit-logs-bucket \
  --is-multi-region-trail

aws guardduty create-detector --enable
```

### Bước 3: Học Và Thực Hành

```
1. Đọc module lý thuyết (30 phút)
2. Thực hành trên AWS Console hoặc CLI (45 phút)
3. Viết IaC bằng Terraform/CDK để tự động hóa (30 phút)
4. Review checklist bảo mật (10 phút)
```

### Bước 4: Chuẩn Bị Câu Chuyện Phỏng Vấn

```
Với mỗi chủ đề, chuẩn bị theo phương pháp STAR:
- Situation (Tình Huống): Bối cảnh dự án/công ty
- Task (Nhiệm Vụ): Yêu cầu bảo mật cần giải quyết
- Action (Hành Động): Dịch vụ AWS và kiến trúc đã triển khai
- Result (Kết Quả): Cải thiện bảo mật, giảm rủi ro, đạt chứng chỉ
```

---

## 📖 Tài Liệu Tham Khảo

### Sách Thiết Yếu

- **"AWS Security Best Practices"** — AWS Whitepaper (miễn phí)
- **"AWS Well-Architected Framework — Security Pillar"** — AWS Whitepaper (miễn phí)
- **"Hacking the Cloud"** — blog thực tế về tấn công/phòng thủ trên AWS
- **"Cloud Security Alliance Guidance"** — tiêu chuẩn bảo mật cloud

### Tài Liệu Chính Thức

- [IAM User Guide](https://docs.aws.amazon.com/iam/)
- [KMS Developer Guide](https://docs.aws.amazon.com/kms/)
- [GuardDuty User Guide](https://docs.aws.amazon.com/guardduty/)
- [Security Hub User Guide](https://docs.aws.amazon.com/securityhub/)
- [AWS Security Blog](https://aws.amazon.com/blogs/security/)

### Công Cụ Thực Hành

- **IAM Policy Simulator** — kiểm thử policy trước khi áp dụng
- **Prowler** — công cụ audit bảo mật AWS mã nguồn mở
- **ScoutSuite** — multi-cloud security auditing tool
- **CloudMapper** — trực quan hóa mạng và quyền truy cập AWS
- **Pacu** — AWS exploitation framework (dùng cho pentesting lab)

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Câu Hỏi Theo Chủ Đề

#### IAM & Identity

- [ ] Phân biệt IAM Role và IAM User — khi nào dùng cái nào?
- [ ] Giải thích Trust Policy vs Permission Policy
- [ ] Cross-account access hoạt động như thế nào?
- [ ] Cách debug "Access Denied" trong IAM
- [ ] SCPs và Permission Boundaries khác nhau như thế nào?

#### Encryption & Key Management

- [ ] Giải thích Envelope Encryption (Mã Hóa Phong Bì)
- [ ] Khi nào dùng CloudHSM thay vì KMS?
- [ ] Secrets Manager vs Parameter Store — chọn cái nào?
- [ ] Cách rotate database credentials với Secrets Manager
- [ ] CMK key policy hoạt động như thế nào?

#### Threat Detection & Monitoring

- [ ] GuardDuty phát hiện mối đe dọa bằng cách nào?
- [ ] Quy trình xử lý khi GuardDuty phát hiện EC2 bị compromised?
- [ ] CloudTrail vs AWS Config — mục đích khác nhau ra sao?
- [ ] Thiết kế SIEM (Security Information and Event Management) trên AWS
- [ ] Làm thế nào điều tra security incident với Detective?

#### Architecture & Compliance

- [ ] Thiết kế kiến trúc multi-account an toàn cho doanh nghiệp
- [ ] Làm thế nào đạt tuân thủ PCI-DSS cho ứng dụng payment trên AWS?
- [ ] Giải thích Data Perimeter và cách triển khai
- [ ] Zero Trust trên AWS triển khai bằng các dịch vụ nào?
- [ ] Well-Architected Security Pillar — 7 design principles là gì?

Xem `11-interview-prep/` để có hướng dẫn Q&A đầy đủ.

---

## ✅ Checklist Tự Đánh Giá

Trước phỏng vấn hoặc nhận vai trò mới, kiểm tra:

- [ ] Giải thích được Least Privilege Principle và áp dụng thực tế
- [ ] Viết được IAM policy phức tạp với conditions từ đầu
- [ ] Thiết kế được kiến trúc multi-account với Organizations
- [ ] Cấu hình được KMS encryption cho các dịch vụ lưu trữ
- [ ] Biết cách phân tích CloudTrail log để điều tra sự cố
- [ ] Thiết lập và xử lý GuardDuty findings
- [ ] Triển khai Secrets Manager với auto-rotation
- [ ] Hiểu và giải thích được AWS Shared Responsibility Model
- [ ] Biết quy trình phản hồi khi tài khoản bị xâm phạm
- [ ] Nắm được ít nhất một compliance framework (SOC2/PCI-DSS/HIPAA)

---

## 📞 Hỗ Trợ & Cộng Đồng

### Học Tập

- [AWS Security Learning Path](https://aws.amazon.com/training/learn-about/security/)
- [A Cloud Guru — AWS Security](https://acloudguru.com/)
- [CloudSecDocs](https://cloudsecdocs.com/) — wiki bảo mật cloud thực tế

### Chứng Chỉ Liên Quan

- **AWS Certified Security — Specialty** — chứng chỉ chuyên sâu về bảo mật AWS
- **AWS Certified Solutions Architect** — nền tảng kiến trúc (khuyến nghị học trước)
- **CISSP, CCSP** — chứng chỉ bảo mật cloud tổng quát

### Cộng Đồng

- AWS Security Community Slack
- r/aws (Reddit) — tag `security`
- Cloud Security Alliance (CSA)
- OWASP Cloud Security Project

---

## 📋 Cách Dùng Tài Liệu Này

### Tự Học

1. Bắt đầu với [Lộ Trình Học Tập](#lộ-trình-học-tập)
2. Đi qua từng giai đoạn theo thứ tự
3. Làm các bài tập thực hành trên AWS
4. Xây dựng dự án portfolio bảo mật

### Chuẩn Bị Phỏng Vấn

1. Tập trung vào `11-interview-prep/`
2. Ôn kỹ IAM và Encryption (luôn được hỏi)
3. Chuẩn bị câu chuyện sự cố theo STAR
4. Luyện giải thích khái niệm không dùng tài liệu

### Làm Việc Thực Tế

1. Tham khảo `06-network-security/` cho thiết kế mạng
2. Dùng `08-threat-detection/` khi xử lý sự cố
3. Kiểm tra `09-compliance-governance/` cho các yêu cầu audit
4. Xác nhận với Security Checklist trước mỗi deployment

---

## 🗺️ Bước Tiếp Theo

```
├─ 1️⃣  Đọc README này đầy đủ
├─ 2️⃣  Chọn lộ trình phù hợp (Security Engineer / Architect / DevSecOps)
├─ 3️⃣  Bắt đầu với 01-iam-fundamentals/
├─ 4️⃣  Thiết lập lab AWS (account riêng, bật CloudTrail + GuardDuty)
├─ 5️⃣  Thực hành từng chủ đề với AWS Console và CLI
├─ 6️⃣  Xây dựng dự án portfolio (ví dụ: hệ thống multi-account an toàn)
└─ 7️⃣  Chuẩn bị phỏng vấn với 11-interview-prep/
```

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
**Người Duy Trì:** Backend Interview Prep
