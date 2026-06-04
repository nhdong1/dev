# Encryption & KMS — Chiến Lược Mã Hóa Trên AWS

> Module này bao gồm toàn bộ chiến lược bảo vệ dữ liệu thông qua mã hóa trên AWS: từ quản lý khóa với KMS (Key Management Service — Dịch Vụ Quản Lý Khóa), kỹ thuật envelope encryption (Mã Hóa Phong Bì), bảo vệ phần cứng với CloudHSM (Hardware Security Module — Module Bảo Mật Phần Cứng), đến các tùy chọn mã hóa cho S3.

---

## 📚 Mục Lục Module

| File | Nội Dung | Trạng Thái |
|---|---|---|
| [1-kms-key-types.md](1-kms-key-types.md) | Loại khóa KMS: CMK, AWS Managed, Data Keys | ✅ |
| [2-envelope-encryption.md](2-envelope-encryption.md) | Mã hóa phong bì, DEK lifecycle | ✅ |
| [3-key-policies.md](3-key-policies.md) | Key policy vs IAM policy — kiểm soát truy cập khóa | ✅ |
| [4-kms-grants.md](4-kms-grants.md) | Grants — ủy quyền dùng khóa tạm thời | ✅ |
| [5-cloudhsm.md](5-cloudhsm.md) | CloudHSM — FIPS 140-2 Level 3, custom key store | ✅ |
| [6-s3-encryption-options.md](6-s3-encryption-options.md) | SSE-S3 vs SSE-KMS vs SSE-C vs CSE | ✅ |

---

## 🎯 Tại Sao Mã Hóa Quan Trọng

### Shared Responsibility Model (Mô Hình Trách Nhiệm Chia Sẻ)

```
AWS chịu trách nhiệm:
  ├── Mã hóa physical storage (hardware-level)
  ├── Network encryption in-transit trong AWS infrastructure
  └── Bảo vệ KMS hardware (FIPS 140-2 Level 3 HSMs)

Khách hàng chịu trách nhiệm:
  ├── Bật encryption cho từng dịch vụ (S3, RDS, EBS, ...)
  ├── Quản lý CMKs (Customer Managed Keys — Khóa Do Khách Hàng Quản Lý)
  ├── Key rotation policy (Chính Sách Xoay Vòng Khóa)
  ├── Key access policy — ai được phép dùng khóa nào
  └── Xóa key khi không còn cần thiết
```

### Ba Trạng Thái Dữ Liệu Cần Bảo Vệ

| Trạng Thái | Tiếng Anh | Giải Pháp AWS |
|---|---|---|
| Dữ liệu lưu trữ | Data at Rest | KMS + SSE cho S3/RDS/EBS/DynamoDB |
| Dữ liệu truyền đi | Data in Transit | TLS 1.2+, ACM, VPC endpoints |
| Dữ liệu đang xử lý | Data in Use | Nitro Enclaves, CloudHSM |

---

## 🏗️ Kiến Trúc Mã Hóa Tổng Quan

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS KMS                                   │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  AWS Owned   │  │ AWS Managed  │  │ Customer Managed │  │
│  │    Keys      │  │    Keys      │  │     Keys (CMK)   │  │
│  │  (miễn phí)  │  │ (tự động)    │  │  (toàn quyền)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                             │
│              Envelope Encryption                            │
│         CMK → mã hóa → DEK → mã hóa → Data                │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                  CloudHSM (tuỳ chọn)                        │
│  ├── FIPS 140-2 Level 3 hardware                            │
│  ├── Custom Key Store trong KMS                             │
│  └── Yêu cầu compliance nghiêm ngặt (PCI-DSS, HIPAA)       │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔑 Các Khái Niệm Cốt Lõi

### KMS (Key Management Service — Dịch Vụ Quản Lý Khóa)

Dịch vụ quản lý khóa mã hóa của AWS. Mọi thao tác mã hóa/giải mã đều đi qua KMS API — khóa **không bao giờ rời khỏi** hạ tầng KMS.

**Ba loại khóa chính:**

```
AWS Owned Keys      — AWS sở hữu và quản lý hoàn toàn, miễn phí
AWS Managed Keys    — AWS tạo trong account của bạn, tự động rotate
Customer Managed    — Bạn tạo và kiểm soát hoàn toàn ($1/key/tháng)
```

### Envelope Encryption (Mã Hóa Phong Bì)

Kỹ thuật mã hóa hai tầng:

```
1. KMS tạo DEK (Data Encryption Key — Khóa Mã Hóa Dữ Liệu)
2. DEK mã hóa dữ liệu thực tế (AES-256)
3. CMK mã hóa DEK → Encrypted DEK
4. Encrypted DEK lưu cùng dữ liệu đã mã hóa
5. Để giải mã: KMS giải mã Encrypted DEK → DEK plaintext → giải mã data
```

**Lợi ích:** Không cần gửi dữ liệu lớn đến KMS; chỉ gửi DEK (nhỏ) để mã hóa/giải mã.

### Key Policy (Chính Sách Khóa)

**Bắt buộc** phải có với CMK. Key policy là tài liệu JSON gắn liền với key, xác định ai được phép dùng key.

> Khác với IAM policy: IAM policy **không đủ** để truy cập CMK nếu key policy không cho phép.

---

## 🗺️ Luồng Quyết Định — Chọn Giải Pháp Mã Hóa

```
Cần mã hóa dữ liệu?
│
├── Dùng dịch vụ AWS lưu trữ? (S3, RDS, EBS, DynamoDB)
│   ├── Yêu cầu audit key usage?          → CMK + CloudTrail
│   ├── Cần rotate thủ công?              → CMK
│   ├── Chỉ cần mã hóa cơ bản?           → AWS Managed Key hoặc SSE-S3
│   └── Khách hàng tự quản lý key riêng? → SSE-C (S3) hoặc Client-Side
│
├── Yêu cầu FIPS 140-2 Level 3?
│   └── CloudHSM + Custom Key Store
│
└── Cần ủy quyền tạm thời cho dịch vụ/người dùng?
    └── KMS Grants
```

---

## 🔒 Mô Hình Bảo Mật KMS

### Kiểm Soát Truy Cập (Access Control)

```
Để sử dụng CMK, cần ĐỒNG THỜI thỏa mãn:

1. Key Policy (bắt buộc)
   └── Principal phải được liệt kê trong key policy

2. IAM Policy (cần thiết nếu key policy ủy quyền cho IAM)
   └── Entity phải có kms:Encrypt, kms:Decrypt, ... trong IAM policy

3. VPC Endpoint Policy (nếu dùng VPC endpoint)
   └── Cho phép KMS API calls qua private network
```

### Kiểm Toán (Auditing)

Mọi thao tác KMS đều được ghi lại trong **CloudTrail**:

```json
{
  "eventName": "Decrypt",
  "userIdentity": { "arn": "arn:aws:iam::123456789012:role/AppRole" },
  "requestParameters": {
    "keyId": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
  },
  "responseElements": null
}
```

---

## 📋 Bảng So Sánh Nhanh

| Tiêu Chí | AWS Owned | AWS Managed | Customer Managed | CloudHSM |
|---|---|---|---|---|
| Chi phí | Miễn phí | Miễn phí | $1/key/tháng | ~$1.45/HSM/giờ |
| Rotation | AWS tự động | Mỗi 1 năm | Tùy chỉnh | Thủ công |
| Audit | Không | CloudTrail | CloudTrail | CloudTrail |
| Key policy | Không | Không | ✅ | ✅ |
| Cross-region | Không | Không | Multi-region ✅ | Không |
| FIPS Level | Level 2 | Level 2 | Level 2 | **Level 3** |
| Xóa key | Không thể | Không thể | ✅ (7–30 ngày) | ✅ |

---

## 🔗 Điều Hướng

| Chủ Đề | File |
|---|---|
| Loại khóa KMS chi tiết | [1-kms-key-types.md](1-kms-key-types.md) |
| Cơ chế envelope encryption | [2-envelope-encryption.md](2-envelope-encryption.md) |
| Viết key policy và IAM policy | [3-key-policies.md](3-key-policies.md) |
| Grants — ủy quyền tạm thời | [4-kms-grants.md](4-kms-grants.md) |
| CloudHSM — hardware HSM | [5-cloudhsm.md](5-cloudhsm.md) |
| Mã hóa S3 — SSE options | [6-s3-encryption-options.md](6-s3-encryption-options.md) |
| Module trước: Organizations | [../03-organizations/README.md](../03-organizations/README.md) |
| Module tiếp: Secrets & Certificates | [../05-secrets-certificates/README.md](../05-secrets-certificates/README.md) |

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp (Module Này)

1. Giải thích Envelope Encryption — tại sao không mã hóa trực tiếp bằng CMK?
2. Sự khác biệt giữa Key Policy và IAM Policy đối với KMS?
3. Khi nào dùng CloudHSM thay vì KMS?
4. KMS Grant là gì, dùng trong trường hợp nào?
5. Phân biệt SSE-S3, SSE-KMS, SSE-C và CSE?
6. Multi-Region Key (Khóa Đa Vùng) hoạt động như thế nào?
7. Làm thế nào audit ai đã dùng CMK nào, khi nào?

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
