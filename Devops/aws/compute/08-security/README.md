# 08 — Security — Bảo Mật AWS Compute

> Tổng quan toàn diện về bảo mật AWS Compute — từ IAM (Identity and Access Management — Quản Lý Danh Tính và Quyền Truy Cập) và Instance Profiles (Hồ Sơ Máy Chủ) đến Systems Manager (Trình Quản Lý Hệ Thống), Secrets Management (Quản Lý Bí Mật), Encryption (Mã Hóa) và Compliance (Tuân Thủ).

---

## 📚 Mục Lục Section Này

| File | Chủ Đề | Mức Độ |
|------|--------|--------|
| [1-iam-instance-profiles.md](./1-iam-instance-profiles.md) | IAM Roles & Instance Profiles, Least Privilege | Cơ bản → Nâng cao |
| [2-systems-manager.md](./2-systems-manager.md) | SSM Session Manager, Run Command, Patch Manager | Trung cấp |
| [3-secrets-management.md](./3-secrets-management.md) | Secrets Manager, Parameter Store, Rotation | Trung cấp |
| [4-encryption.md](./4-encryption.md) | EBS Encryption, KMS, Data-in-Transit | Trung cấp → Nâng cao |
| [5-compliance-patching.md](./5-compliance-patching.md) | Inspector, Security Hub, Patch Baselines | Nâng cao |

---

## 🎯 Tại Sao Bảo Mật AWS Compute Quan Trọng?

### Chi Phí Của Breach (Vi Phạm Bảo Mật)

```
Trung bình chi phí một data breach năm 2024: $4.88 triệu USD
→ IAM misconfiguration (cấu hình sai IAM): nguyên nhân #1
→ Exposed credentials (lộ thông tin xác thực): nguyên nhân #2
→ Unpatched vulnerabilities (lỗ hổng chưa vá): nguyên nhân #3
```

### Trách Nhiệm Chia Sẻ — Shared Responsibility Model

```
┌─────────────────────────────────────────────────────────────┐
│                    CLOUD SECURITY MODEL                      │
├──────────────────────────┬──────────────────────────────────┤
│     AWS Chịu Trách Nhiệm │     Bạn Chịu Trách Nhiệm         │
│     (Security OF Cloud)  │     (Security IN Cloud)           │
├──────────────────────────┼──────────────────────────────────┤
│ • Physical hardware      │ • IAM users, roles, policies      │
│ • Data center security   │ • OS patching trên EC2            │
│ • Network infrastructure │ • Application security            │
│ • Hypervisor security    │ • Data encryption (lựa chọn)      │
│ • Managed service patches│ • Security Group rules            │
│   (RDS, Lambda, ECS…)   │ • Network ACL configuration       │
│                          │ • Secrets & credentials           │
└──────────────────────────┴──────────────────────────────────┘
```

---

## 🏗️ Kiến Trúc Bảo Mật Tổng Thể

### Defense in Depth — Bảo Mật Theo Chiều Sâu

```
                    ┌─────────────────────────────────┐
                    │       INTERNET / PUBLIC          │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │   WAF (Web Application Firewall  │  ← Lớp 1: Lọc traffic ác ý
                    │   — Tường Lửa Ứng Dụng Web)      │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
                    │   CloudFront / ALB               │  ← Lớp 2: Cân bằng tải có SSL
                    │   (TLS Termination)              │
                    └──────────────┬──────────────────┘
                                   │
          ┌────────────────────────▼────────────────────────┐
          │              VPC (Virtual Private Cloud)         │  ← Lớp 3: Cách ly mạng
          │                                                  │
          │  ┌─────────────────────────────────────────┐    │
          │  │           Public Subnet                  │    │
          │  │  ┌──────────┐      ┌──────────────┐     │    │
          │  │  │ Bastion  │      │ NAT Gateway  │     │    │
          │  │  │ (không   │      │ (chuyển tiếp │     │    │
          │  │  │ khuyến   │      │ outbound)    │     │    │
          │  │  │ dùng nữa)│      └──────────────┘     │    │
          │  │  └──────────┘                            │    │
          │  └─────────────────────────────────────────┘    │
          │                                                  │
          │  ┌─────────────────────────────────────────┐    │
          │  │           Private Subnet                 │    │
          │  │                                          │    │
          │  │  ┌──────────┐  Security Group ← Lớp 4   │    │
          │  │  │ EC2 /    │  (Tường lửa stateful)      │    │
          │  │  │ ECS Task │                            │    │
          │  │  │ IAM Role │ ← Lớp 5: Quyền truy cập  │    │
          │  │  └────┬─────┘                            │    │
          │  │       │ Encrypted EBS ← Lớp 6: Mã hóa  │    │
          │  └───────┼─────────────────────────────────┘    │
          └──────────┼──────────────────────────────────────┘
                     │
          ┌──────────▼──────────────────────────────────────┐
          │  AWS Services (IAM, KMS, SSM, Secrets Manager)  │  ← Lớp 7: Dịch vụ bảo mật
          └──────────────────────────────────────────────────┘
```

---

## 🔑 Các Trụ Cột Bảo Mật AWS Compute

### 1. Identity & Access — Danh Tính và Quyền Truy Cập

```
IAM (Identity and Access Management — Quản Lý Danh Tính và Quyền Truy Cập)
├── Users (Người Dùng) — chỉ cho con người, không cho ứng dụng
├── Groups (Nhóm) — tập hợp users với cùng permissions
├── Roles (Vai Trò) — cho AWS services, applications, cross-account
│   ├── EC2 Instance Profile — Role gắn vào EC2
│   ├── ECS Task Role — Role cho từng container task
│   ├── Lambda Execution Role — Role cho Lambda function
│   └── EKS IRSA (IAM Roles for Service Accounts — Vai Trò IAM Cho Service Accounts)
└── Policies (Chính Sách) — JSON document định nghĩa permissions
    ├── AWS Managed Policies — AWS tạo và quản lý
    ├── Customer Managed Policies — bạn tạo và quản lý
    └── Inline Policies — gắn trực tiếp vào entity
```

### 2. Network Security — Bảo Mật Mạng

```
Security Groups (Nhóm Bảo Mật) — Stateful firewall (tường lửa stateful)
├── Inbound Rules: chỉ cho phép những gì cần thiết
├── Outbound Rules: thường restrict để tránh data exfiltration
└── Reference other SGs thay vì CIDR khi có thể

Network ACLs (Access Control Lists — Danh Sách Kiểm Soát Truy Cập)
├── Stateless: phải define cả inbound lẫn outbound
├── Apply ở subnet level (cấp độ subnet)
└── Bổ sung thêm lớp bảo vệ cùng Security Groups

VPC Flow Logs — ghi lại toàn bộ traffic trong VPC
└── Phân tích với CloudWatch Logs Insights hoặc Athena
```

### 3. Secrets & Credentials — Bí Mật và Thông Tin Xác Thực

```
❌ KHÔNG BAO GIỜ làm:
   - Hard-code credentials trong source code
   - Lưu access keys trong EC2 user data
   - Commit secrets lên Git
   - Dùng IAM User với long-term access keys cho EC2/Lambda

✅ LUÔN làm:
   - Dùng IAM Roles cho EC2, ECS, Lambda
   - Lưu secrets trong Secrets Manager hoặc Parameter Store
   - Rotate (xoay vòng) credentials định kỳ
   - Dùng IMDS v2 (Instance Metadata Service version 2) trên EC2
```

### 4. Encryption — Mã Hóa

```
Encryption at Rest (Mã Hóa Khi Lưu Trữ):
├── EBS Volumes — KMS encryption (bật mặc định trong account settings)
├── S3 — SSE-S3, SSE-KMS, SSE-C
├── RDS — encryption bật lúc tạo (không thể bật sau)
└── EFS (Elastic File System) — KMS encryption

Encryption in Transit (Mã Hóa Khi Truyền):
├── TLS/HTTPS cho mọi API calls và web traffic
├── AWS Certificate Manager — SSL/TLS certificates miễn phí
└── VPC endpoints — traffic không ra internet
```

### 5. Visibility & Compliance — Khả Năng Quan Sát và Tuân Thủ

```
CloudTrail — ghi lại mọi API call vào AWS
CloudWatch Logs — ghi lại application & OS logs
AWS Config — theo dõi thay đổi cấu hình
Amazon Inspector — quét lỗ hổng bảo mật tự động
AWS Security Hub — tổng hợp security findings từ nhiều nguồn
AWS GuardDuty — phát hiện mối đe dọa bằng ML
```

---

## 🛡️ AWS Security Services — Dịch Vụ Bảo Mật AWS

### Dịch Vụ Phát Hiện Mối Đe Dọa

| Dịch Vụ | Chức Năng | Output |
|---------|-----------|--------|
| **GuardDuty** | ML-based threat detection (phát hiện mối đe dọa bằng ML) | Findings (phát hiện) |
| **Inspector** | Vulnerability scanning (quét lỗ hổng) cho EC2, Lambda, ECR | CVE reports |
| **Detective** | Điều tra root cause của security incidents | Investigation graphs |
| **Macie** | Phát hiện sensitive data (dữ liệu nhạy cảm) trong S3 | Data findings |

### Dịch Vụ Bảo Vệ

| Dịch Vụ | Chức Năng | Phù Hợp Cho |
|---------|-----------|-------------|
| **WAF** (Web Application Firewall) | Lọc HTTP requests ác ý | ALB, CloudFront, API Gateway |
| **Shield Standard** | DDoS protection tự động | Tất cả AWS services (miễn phí) |
| **Shield Advanced** | DDoS protection nâng cao + 24/7 support | Enterprise workloads |
| **Network Firewall** | Stateful firewall ở VPC level | Kiểm soát traffic phức tạp |

### Dịch Vụ Quản Lý

| Dịch Vụ | Chức Năng | Ghi Chú |
|---------|-----------|---------|
| **IAM** | Identity & access control | Nền tảng mọi thứ |
| **KMS** (Key Management Service — Dịch Vụ Quản Lý Khóa) | Tạo và quản lý encryption keys | Tích hợp với hầu hết services |
| **Secrets Manager** | Lưu trữ và rotate secrets | Tính phí per secret |
| **Systems Manager** | Quản lý EC2 fleet không cần SSH | SSM Agent cần cài |
| **Security Hub** | Tổng hợp findings từ nhiều services | Central dashboard |

---

## 📋 Security Checklist Nhanh — Danh Sách Kiểm Tra Bảo Mật

### Trước Khi Deploy EC2

```
□ IAM Role (không dùng access keys) đã gắn vào EC2?
□ Security Group chỉ mở đúng ports cần thiết?
□ SSH (port 22) KHÔNG mở từ 0.0.0.0/0?
□ EBS volumes được encrypt?
□ IMDS v2 được enforce (IMDSv2)?
□ Instance trong Private Subnet (nếu không cần public IP)?
□ CloudWatch agent cài và cấu hình để gửi logs?
□ SSM Agent cài để dùng Session Manager (không cần SSH)?
```

### Trước Khi Deploy Lambda

```
□ Execution Role chỉ có permissions cần thiết?
□ Environment variables sensitive KHÔNG hard-code?
□ Secrets lấy từ Secrets Manager hoặc Parameter Store?
□ VPC configuration nếu cần access private resources?
□ Function URL có authentication nếu expose ra ngoài?
□ Reserved concurrency đặt để tránh quota exhaustion?
```

### Trước Khi Deploy Container (ECS/EKS)

```
□ Task Role (ECS) hoặc IRSA (EKS) đúng permissions?
□ Container images được scan vulnerabilities trong ECR?
□ Không chạy container với root user?
□ Secrets không trong environment variables dạng plaintext?
□ Network mode awsvpc (ECS) để isolation tốt hơn?
□ Pod Security Standards (EKS) được áp dụng?
```

---

## 🎓 Lộ Trình Học Bảo Mật AWS Compute

### Beginner (Người Mới) — Tuần 1-2

```
1. Hiểu IAM Users vs Roles vs Groups vs Policies
2. Tạo IAM Role và gán vào EC2 instance
3. Cấu hình Security Groups đúng cách (Least Privilege)
4. Hiểu sự khác biệt Security Group vs NACL
5. Bật CloudTrail để audit mọi API calls
```

### Intermediate (Trung Cấp) — Tuần 3-4

```
1. Dùng SSM Session Manager thay SSH
2. Lưu secrets trong Secrets Manager
3. Bật EBS encryption mặc định cho account
4. Cấu hình VPC endpoints để traffic không ra internet
5. Thiết lập AWS Config rules cho compliance
```

### Advanced (Nâng Cao) — Tuần 5-6

```
1. Implement SCPs (Service Control Policies) trong AWS Organizations
2. Thiết lập GuardDuty và Security Hub cho threat detection
3. Cấu hình Inspector cho automated vulnerability scanning
4. Implement secrets rotation tự động với Secrets Manager
5. Thiết kế Zero Trust Architecture trên AWS
```

---

## 🔗 Điều Hướng

| Chủ Đề | File |
|--------|------|
| ← Cost Optimization | [../07-cost-optimization/README.md](../07-cost-optimization/README.md) |
| → Monitoring | [../09-monitoring/README.md](../09-monitoring/README.md) |
| IAM & Instance Profiles | [1-iam-instance-profiles.md](./1-iam-instance-profiles.md) |
| Systems Manager | [2-systems-manager.md](./2-systems-manager.md) |
| Secrets Management | [3-secrets-management.md](./3-secrets-management.md) |
| Encryption & KMS | [4-encryption.md](./4-encryption.md) |
| Compliance & Patching | [5-compliance-patching.md](./5-compliance-patching.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
