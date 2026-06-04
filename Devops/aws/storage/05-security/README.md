# 🔐 Bảo Mật Lưu Trữ AWS — Tổng Quan

> Hướng dẫn toàn diện về bảo mật cho S3, EBS và EFS — từ kiểm soát truy cập đến mã hóa dữ liệu.

## 📚 Mục Lục Chủ Đề

| File | Chủ Đề | Mức Độ |
|------|--------|--------|
| [1-iam-policies-for-storage.md](./1-iam-policies-for-storage.md) | IAM Policies cho S3, EBS, EFS | Trung cấp |
| [2-s3-bucket-policies.md](./2-s3-bucket-policies.md) | Bucket Policies, Condition Keys, Cross-Account | Trung cấp |
| [3-block-public-access.md](./3-block-public-access.md) | Block Public Access — Chặn Truy Cập Công Khai | Cơ bản |
| [4-encryption-at-rest.md](./4-encryption-at-rest.md) | SSE-S3, SSE-KMS, SSE-C, CSE — Mã Hóa Khi Lưu | Nâng cao |
| [5-encryption-in-transit.md](./5-encryption-in-transit.md) | TLS/HTTPS, VPC Endpoints — Mã Hóa Khi Truyền | Trung cấp |
| [6-vpc-endpoints.md](./6-vpc-endpoints.md) | Gateway vs Interface Endpoint cho S3/EFS | Nâng cao |
| [7-s3-access-analyzer.md](./7-s3-access-analyzer.md) | S3 Access Analyzer — Phân Tích Quyền Truy Cập | Trung cấp |

---

## 🎯 Tại Sao Bảo Mật Lưu Trữ Quan Trọng

### Ba Lớp Bảo Mật Cốt Lõi

```
┌─────────────────────────────────────────────────────────┐
│                  Lớp 1: Kiểm Soát Truy Cập              │
│   IAM Policies + Bucket Policies + Block Public Access   │
├─────────────────────────────────────────────────────────┤
│                  Lớp 2: Bảo Vệ Dữ Liệu                  │
│         Mã hóa at-rest (SSE-S3/KMS/C) + in-transit      │
├─────────────────────────────────────────────────────────┤
│                  Lớp 3: Giám Sát & Kiểm Toán             │
│        S3 Access Analyzer + CloudTrail + Logging         │
└─────────────────────────────────────────────────────────┘
```

### Rủi Ro Thường Gặp

| Rủi Ro | Hậu Quả | Biện Pháp |
|--------|---------|-----------|
| S3 bucket public ngẫu nhiên | Lộ dữ liệu nhạy cảm | Block Public Access + Access Analyzer |
| Dữ liệu không được mã hóa | Vi phạm compliance | SSE-KMS bắt buộc qua policy |
| IAM policy quá rộng | Privilege escalation | Principle of Least Privilege |
| Không có audit trail | Không điều tra được incident | CloudTrail + S3 Access Logging |
| Truyền tải không mã hóa | MITM attack | Enforce TLS qua bucket policy |

---

## 🗺️ Mô Hình Bảo Mật AWS Storage

### Shared Responsibility Model — Mô Hình Trách Nhiệm Chia Sẻ

```
AWS chịu trách nhiệm:               Bạn chịu trách nhiệm:
┌────────────────────┐               ┌────────────────────┐
│ Bảo mật cơ sở hạ  │               │ Cấu hình IAM       │
│ tầng vật lý        │               │ Bucket policies     │
│ Encryption keys    │               │ Bật mã hóa         │
│ cho SSE-S3         │               │ Quản lý KMS keys   │
│ Network security   │               │ Block Public Access │
│ hardware layer     │               │ Monitoring & alert  │
└────────────────────┘               └────────────────────┘
```

### Nguyên Tắc Bảo Mật Cốt Lõi

1. **Principle of Least Privilege** — Nguyên tắc quyền tối thiểu: chỉ cấp quyền thực sự cần thiết
2. **Defense in Depth** — Phòng thủ nhiều lớp: không dựa vào một cơ chế duy nhất
3. **Encrypt Everything** — Mã hóa mọi thứ: at-rest và in-transit
4. **Audit Continuously** — Kiểm toán liên tục: log mọi API call và truy cập

---

## 🔑 Kiểm Soát Truy Cập S3 — Tổng Quan

### Thứ Tự Đánh Giá Quyền S3

```
Yêu cầu API đến S3
        │
        ▼
┌───────────────────┐    DENY   ┌──────────────┐
│ Block Public      │──────────▶│ TỪ CHỐI      │
│ Access settings   │           └──────────────┘
└───────────────────┘
        │ Không bị chặn
        ▼
┌───────────────────┐    DENY   ┌──────────────┐
│ SCPs (Service     │──────────▶│ TỪ CHỐI      │
│ Control Policies) │           └──────────────┘
└───────────────────┘
        │ Không bị chặn
        ▼
┌───────────────────┐    DENY   ┌──────────────┐
│ IAM identity      │──────────▶│ TỪ CHỐI      │
│ policies          │           └──────────────┘
└───────────────────┘
        │ Không bị chặn
        ▼
┌───────────────────┐    DENY   ┌──────────────┐
│ Bucket policies   │──────────▶│ TỪ CHỐI      │
│ & ACLs            │           └──────────────┘
└───────────────────┘
        │ ALLOW rõ ràng
        ▼
┌───────────────────┐
│ CHO PHÉP TRUY CẬP │
└───────────────────┘
```

**Nguyên tắc:** Một DENY rõ ràng ở bất kỳ tầng nào sẽ ghi đè mọi ALLOW.

---

## 🔐 Tổng Quan Mã Hóa

### Mã Hóa At-Rest (Khi Lưu) — S3

| Loại | Quản Lý Key | Overhead | Dùng Khi |
|------|------------|---------|---------|
| **SSE-S3** | AWS quản lý hoàn toàn | Không có | Mã hóa cơ bản, không cần audit |
| **SSE-KMS** | AWS KMS, bạn kiểm soát | Chi phí KMS | Cần audit trail, compliance |
| **SSE-C** | Bạn quản lý key | Phức tạp | Yêu cầu giữ key phía client |
| **CSE** | Bạn mã hóa trước | Cao nhất | Zero-trust, tuân thủ nghiêm ngặt |

### Mã Hóa In-Transit (Khi Truyền)

- **TLS 1.2+** bắt buộc qua bucket policy `aws:SecureTransport`
- **VPC Endpoints** — lưu lượng không qua internet công cộng

---

## 📋 Security Checklist — Danh Sách Kiểm Tra Bảo Mật

### Trước Khi Đưa Vào Production

```
S3 Bucket Security:
□ Block Public Access bật ở tất cả 4 cấu hình
□ Bucket policy có điều khoản deny HTTP (enforce TLS)
□ Mã hóa mặc định bật (ít nhất SSE-S3)
□ Versioning bật cho dữ liệu quan trọng
□ Server access logging bật
□ S3 Access Analyzer đang chạy

IAM & Access Control:
□ Không có IAM user với quyền s3:* không hạn chế
□ Cross-account access được kiểm soát bằng STS
□ Access keys được rotate định kỳ
□ MFA bật cho tài khoản root

Monitoring:
□ CloudTrail bật cho S3 data events
□ CloudWatch alarms cho GetObject bất thường
□ AWS Config rules cho S3 security checks
```

---

## 🔗 Điều Hướng

| Chủ Đề Tiếp Theo | Link |
|-----------------|------|
| IAM Policies chi tiết | [1-iam-policies-for-storage.md](./1-iam-policies-for-storage.md) |
| Bucket Policies nâng cao | [2-s3-bucket-policies.md](./2-s3-bucket-policies.md) |
| Tối ưu chi phí | [../06-cost-optimization/README.md](../06-cost-optimization/README.md) |
| EFS Security | [../04-efs/1-mount-targets-and-access-points.md](../04-efs/1-mount-targets-and-access-points.md) |

---

**Phiên Bản:** 1.0
**Cập Nhật:** 2026-05-16
