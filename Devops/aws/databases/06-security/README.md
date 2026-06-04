# 06 — Security — Bảo Mật Cơ Sở Dữ Liệu Toàn Diện

> Bảo mật là lớp phòng thủ không thể thiếu cho mọi hệ thống database trên AWS. Module này bao gồm toàn bộ chiến lược bảo mật: từ Network Isolation (Cô Lập Mạng) với VPC, xác thực danh tính với IAM, mã hóa dữ liệu với KMS, quản lý thông tin bí mật với Secrets Manager, đến kiểm toán và tuân thủ với CloudTrail và Database Activity Streams.

---

## 📚 Mục Lục Module

| File | Chủ Đề | Độ Ưu Tiên |
|------|--------|------------|
| [1-vpc-security-groups.md](./1-vpc-security-groups.md) | VPC, Private Subnets, Security Groups | ⭐⭐⭐ Bắt buộc |
| [2-iam-authentication.md](./2-iam-authentication.md) | IAM Roles, Database Authentication | ⭐⭐⭐ Bắt buộc |
| [3-encryption.md](./3-encryption.md) | KMS, Encryption at Rest & in Transit | ⭐⭐⭐ Bắt buộc |
| [4-secrets-management.md](./4-secrets-management.md) | Secrets Manager, Parameter Store, Rotation | ⭐⭐⭐ Bắt buộc |
| [5-audit-compliance.md](./5-audit-compliance.md) | CloudTrail, Activity Streams, PCI/HIPAA/GDPR | ⭐⭐ Nên Có |

---

## 🎯 Tổng Quan Bảo Mật Database Trên AWS

### Mô Hình Bảo Mật Nhiều Lớp (Defense in Depth)

```
┌─────────────────────────────────────────────────────────────────┐
│                     INTERNET / USERS                            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    [Layer 1: Network]
          ┌────────────────▼────────────────┐
          │   VPC — Virtual Private Cloud   │
          │   Private Subnets               │
          │   Security Groups               │
          │   NACLs (Network ACLs)         │
          └────────────────┬────────────────┘
                           │
                  [Layer 2: Identity]
          ┌────────────────▼────────────────┐
          │   IAM — Identity & Access Mgmt  │
          │   Database Authentication       │
          │   Resource-based Policies       │
          └────────────────┬────────────────┘
                           │
                   [Layer 3: Data]
          ┌────────────────▼────────────────┐
          │   KMS Encryption at Rest        │
          │   TLS Encryption in Transit     │
          │   Secrets Manager               │
          └────────────────┬────────────────┘
                           │
                  [Layer 4: Audit]
          ┌────────────────▼────────────────┐
          │   CloudTrail (API Audit)        │
          │   Database Activity Streams     │
          │   CloudWatch Logs               │
          └─────────────────────────────────┘
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### 1. Network Isolation — Cô Lập Mạng

**VPC** (Virtual Private Cloud — Đám Mây Riêng Ảo) là nền tảng bảo mật mạng:

- Database **không bao giờ** được đặt trong public subnet (mạng con công khai)
- Chỉ cho phép kết nối từ application tier (tầng ứng dụng) trong cùng VPC
- Security Groups (Nhóm Bảo Mật) kiểm soát traffic ở cấp độ instance
- NACLs (Network Access Control Lists — Danh Sách Kiểm Soát Truy Cập Mạng) kiểm soát ở cấp subnet

### 2. Identity & Access — Danh Tính & Quyền Truy Cập

**IAM** (Identity and Access Management — Quản Lý Danh Tính và Quyền Truy Cập):

- **Principle of Least Privilege** (Nguyên Tắc Đặc Quyền Tối Thiểu) — chỉ cấp quyền thực sự cần thiết
- IAM Authentication cho RDS — đăng nhập bằng IAM token thay vì password
- IAM Roles cho EC2/Lambda — không hardcode credentials trong code
- Resource-based policies cho DynamoDB

### 3. Data Protection — Bảo Vệ Dữ Liệu

**Encryption** (Mã Hóa) ở hai trạng thái:

- **At Rest** (Khi Lưu Trữ) — dữ liệu trên đĩa được mã hóa bằng KMS
- **In Transit** (Khi Truyền Tải) — kết nối qua TLS/SSL
- **KMS** (Key Management Service — Dịch Vụ Quản Lý Khóa) — quản lý khóa mã hóa tập trung

**Secrets Management** (Quản Lý Thông Tin Bí Mật):

- **Secrets Manager** — lưu và tự động xoay vòng database passwords
- **Parameter Store** — lưu configuration và secrets ít nhạy cảm hơn

### 4. Audit & Compliance — Kiểm Toán & Tuân Thủ

- **CloudTrail** — ghi lại mọi API call tới AWS (WHO did WHAT WHEN)
- **Database Activity Streams** (Luồng Hoạt Động Database) — ghi lại từng câu SQL
- **CloudWatch Logs** — log tập trung, alert theo ngưỡng

---

## 🏗️ Kiến Trúc Bảo Mật Chuẩn

### Kiến Trúc 3-Tier Điển Hình

```
┌──────────────────────────────────────────────────────────────┐
│                        VPC (10.0.0.0/16)                     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Public Subnet (10.0.1.0/24)             │    │
│  │  ┌──────────────────────────────────────────────┐   │    │
│  │  │  Load Balancer (ALB)                          │   │    │
│  │  │  Security Group: inbound 443/80 từ 0.0.0.0/0 │   │    │
│  │  └────────────────────┬─────────────────────────┘   │    │
│  └───────────────────────│───────────────────────────── ┘    │
│                          │                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Private Subnet App (10.0.2.0/24)           │    │
│  │  ┌──────────────────────────────────────────────┐   │    │
│  │  │  EC2 / ECS / Lambda (Application Layer)       │   │    │
│  │  │  Security Group: inbound từ ALB SG only       │   │    │
│  │  └────────────────────┬─────────────────────────┘   │    │
│  └───────────────────────│────────────────────────────── ┘   │
│                          │                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Private Subnet DB (10.0.3.0/24)            │    │
│  │  ┌──────────────────────────────────────────────┐   │    │
│  │  │  RDS / Aurora / ElastiCache                   │   │    │
│  │  │  Security Group: inbound từ App SG only       │   │    │
│  │  │  KMS Encryption at Rest                       │   │    │
│  │  │  TLS in Transit                               │   │    │
│  │  └──────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Điểm Quan Trọng Trong Kiến Trúc Này

| Thành Phần | Nguyên Tắc Bảo Mật |
|------------|---------------------|
| Load Balancer | Public, chỉ nhận HTTPS (443) |
| Application Server | Private subnet, không có public IP |
| Database | Private subnet sâu nhất, chỉ nhận từ App SG |
| Credentials | Lấy từ Secrets Manager qua IAM Role |
| Dữ liệu | Mã hóa AES-256 bằng KMS at rest + TLS in transit |

---

## ⚡ AWS Shared Responsibility Model — Mô Hình Trách Nhiệm Chia Sẻ

Hiểu rõ AWS quản lý gì và bạn phải tự quản lý gì:

| Thành Phần | AWS Quản Lý | Bạn Quản Lý |
|------------|-------------|-------------|
| **Physical datacenter** | ✅ Hoàn toàn | ❌ |
| **Hypervisor / Host OS** | ✅ Với managed services | ❌ |
| **RDS Engine patching** | ✅ Tự động | Lên lịch maintenance window |
| **Database networking** | ✅ Hạ tầng | VPC config, Security Groups |
| **Encryption keys (AWS managed)** | ✅ Rotate tự động | ❌ |
| **Encryption keys (Customer managed)** | Key storage an toàn | Bạn tự rotate và manage |
| **IAM policies** | IAM engine | Bạn tự viết policy đúng |
| **Database user accounts** | ❌ | Bạn tự tạo và quản lý |
| **Application data** | ❌ | Bạn hoàn toàn chịu trách nhiệm |
| **Compliance certification** | AWS cung cấp certifications | Bạn phải cấu hình đúng |

---

## 🔐 Security Best Practices Nhanh

### Network

```
✅ Đặt database trong private subnet — không bao giờ public
✅ Dùng Security Group thay vì IP ranges khi có thể
✅ Không mở port database ra internet (3306, 5432, 6379...)
✅ Enable VPC Flow Logs để monitor traffic bất thường
✅ Dùng AWS PrivateLink hoặc VPC Peering khi cần cross-VPC
```

### Identity

```
✅ Dùng IAM Roles cho EC2/Lambda — không hardcode credentials
✅ Enable IAM Authentication cho RDS khi có thể
✅ Áp dụng Principle of Least Privilege cho mọi IAM policy
✅ Rotate database passwords định kỳ qua Secrets Manager
✅ Enable MFA cho tài khoản AWS admin
```

### Encryption

```
✅ Enable encryption at rest cho mọi RDS, Aurora, ElastiCache
✅ Bật require_secure_transport (enforce SSL) trên RDS
✅ Dùng Customer Managed Keys (CMK) cho dữ liệu nhạy cảm
✅ Enable automatic key rotation cho CMK
✅ Không dùng unencrypted snapshots để chia sẻ
```

### Secrets

```
✅ Lưu mọi database credential trong Secrets Manager
✅ Enable automatic rotation — không để password không đổi
✅ Không hardcode passwords trong code, config files, hoặc env vars không bảo mật
✅ Dùng Parameter Store cho configs không nhạy cảm (save cost)
✅ Tag secrets với môi trường (dev/staging/prod)
```

### Audit

```
✅ Enable CloudTrail ở tất cả regions
✅ Enable Database Activity Streams cho RDS/Aurora quan trọng
✅ Enable Enhanced Monitoring và CloudWatch Logs
✅ Set alert khi có login failure bất thường
✅ Review IAM policies định kỳ với IAM Access Analyzer
```

---

## 🎯 Câu Hỏi Phỏng Vấn Nhanh

**Q: Làm thế nào để bảo mật RDS database trên AWS?**

> Trả lời theo 4 lớp: (1) Network — VPC private subnet, Security Groups restrictive; (2) Identity — IAM Authentication, least privilege; (3) Data — KMS encryption at rest, TLS in transit, Secrets Manager cho credentials; (4) Audit — CloudTrail, Database Activity Streams.

**Q: Khác biệt giữa Secrets Manager và Parameter Store?**

> Secrets Manager: tự động rotate, native integration với RDS, đắt hơn ($0.40/secret/tháng). Parameter Store: rẻ hơn (free tier), không tự rotate, phù hợp cho config thông thường. Dùng Secrets Manager cho database credentials, Parameter Store cho app config.

**Q: Nếu phát hiện database credentials bị lộ, làm gì ngay lập tức?**

> (1) Rotate ngay password qua Secrets Manager; (2) Revoke mọi active session; (3) Kiểm tra CloudTrail xem có truy cập bất thường không; (4) Check Database Activity Streams nếu đã bật; (5) Review IAM policies có liên quan; (6) Post-mortem và cải thiện process.

---

## 📂 Điều Hướng

| ← Trước | Chủ Đề Hiện Tại | Tiếp → |
|---------|-----------------|--------|
| [05-ha-backup/](../05-ha-backup/README.md) | **06-security/** | [07-performance-tuning/](../07-performance-tuning/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15 | **Trạng Thái:** ✅ Hoàn Thành
