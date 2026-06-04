# 05 — Secrets & Certificate Management (Quản Lý Bí Mật & Chứng Chỉ)

> Hướng dẫn toàn diện về vòng đời quản lý bí mật và chứng chỉ TLS trên AWS — từ lưu trữ an toàn, xoay vòng tự động đến PKI (Public Key Infrastructure — Hạ Tầng Khóa Công Khai) nội bộ.

---

## 📚 Mục Lục Module

| File | Nội Dung | Mức Độ |
|---|---|---|
| [1-secrets-manager.md](1-secrets-manager.md) | AWS Secrets Manager — lưu trữ, xoay vòng, truy xuất bí mật | ⭐⭐ |
| [2-parameter-store.md](2-parameter-store.md) | SSM Parameter Store — phân cấp, SecureString, phiên bản | ⭐⭐ |
| [3-secrets-vs-parameter.md](3-secrets-vs-parameter.md) | So sánh Secrets Manager vs Parameter Store | ⭐⭐ |
| [4-acm-certificates.md](4-acm-certificates.md) | ACM — vòng đời chứng chỉ TLS, gia hạn tự động | ⭐⭐ |
| [5-acm-private-ca.md](5-acm-private-ca.md) | ACM Private CA — PKI nội bộ trên AWS | ⭐⭐⭐ |

---

## 🎯 Tại Sao Module Này Quan Trọng

Credentials (thông tin xác thực) bị lộ là một trong những nguyên nhân hàng đầu gây ra các vụ vi phạm bảo mật đám mây:

- **2019 — Capital One breach:** AWS credentials bị lộ qua SSRF (Server-Side Request Forgery — Giả Mạo Yêu Cầu Phía Máy Chủ) dẫn đến 100 triệu hồ sơ khách hàng bị đánh cắp
- **2020 — Twitch breach:** Secrets hardcoded (mã hóa cứng) trong source code bị lộ
- **Ngày nay:** Hàng nghìn secret bị lộ mỗi ngày qua GitHub public repositories

AWS cung cấp hai dịch vụ chính để giải quyết vấn đề này: **Secrets Manager** và **SSM Parameter Store** — mỗi dịch vụ có ưu/nhược điểm riêng.

---

## 🗺️ Bản Đồ Dịch Vụ

```
Secrets & Certificates
├── AWS Secrets Manager
│   ├── Lưu trữ: DB passwords, API keys, OAuth tokens
│   ├── Auto-rotation (Xoay Vòng Tự Động): Lambda + schedule
│   ├── Replication (Nhân Bản): Multi-region
│   └── Integration: RDS, Redshift, DocumentDB native
│
├── SSM Parameter Store
│   ├── Standard: miễn phí, 4KB/param, không rotation
│   ├── Advanced: phí, 8KB/param, có rotation (qua Secrets Manager)
│   ├── SecureString: mã hóa bằng KMS
│   └── Hierarchy: /app/env/key — dễ quản lý config
│
├── ACM (AWS Certificate Manager)
│   ├── Public certificates: miễn phí, auto-renewal
│   ├── Private certificates: phí theo cert
│   ├── Integration: ALB, CloudFront, API Gateway, ELB
│   └── Validation: DNS hoặc Email
│
└── ACM Private CA
    ├── Root CA (Tổ Chức Phát Hành Gốc) hoặc Subordinate CA
    ├── FIPS 140-2 Level 3 (tùy chọn CloudHSM backend)
    ├── CRL (Certificate Revocation List — Danh Sách Chứng Chỉ Thu Hồi)
    └── OCSP (Online Certificate Status Protocol — Giao Thức Kiểm Tra Trạng Thái Chứng Chỉ Trực Tuyến)
```

---

## 🔑 Khái Niệm Cốt Lõi

### Secret Rotation (Xoay Vòng Bí Mật)

**Rotation** là quá trình thay thế secret cũ bằng secret mới theo định kỳ mà không gây gián đoạn dịch vụ.

```
Tại sao rotation quan trọng:
- Giới hạn thời gian tấn công nếu secret bị lộ
- Đáp ứng yêu cầu tuân thủ (PCI-DSS, HIPAA)
- Thực hành tốt nhất theo CIS Benchmarks
```

**Hai chiến lược rotation:**

| Chiến Lược | Mô Tả | Khi Dùng |
|---|---|---|
| **Single-user rotation** | Cập nhật password của một user duy nhất | Ứng dụng chấp nhận brief downtime |
| **Multi-user rotation** | Dùng luân phiên hai user (clone user) | Zero-downtime, production critical |

### Envelope Encryption cho Secrets (Mã Hóa Phong Bì cho Bí Mật)

```
Secret plaintext (văn bản gốc)
    └── Được mã hóa bằng DEK (Data Encryption Key — Khóa Mã Hóa Dữ Liệu)
        └── DEK được mã hóa bằng CMK (Customer Master Key — Khóa Chính Do Khách Hàng Quản Lý) trong KMS
            └── CMK không bao giờ rời khỏi KMS
```

### PKI (Public Key Infrastructure — Hạ Tầng Khóa Công Khai)

```
PKI Hierarchy (Phân Cấp PKI):
Root CA (Tổ Chức Phát Hành Gốc)
└── Intermediate CA / Subordinate CA (Tổ Chức Phát Hành Trung Gian)
    ├── End-entity certificate (chứng chỉ thực thể cuối) — servers
    ├── End-entity certificate — clients
    └── End-entity certificate — code signing
```

---

## 🔄 Vòng Đời Bí Mật (Secret Lifecycle)

```
1. CREATE (Tạo)
   └── Lưu secret trong Secrets Manager hoặc Parameter Store
   └── Gắn resource policy (nếu cần)
   └── Cấu hình KMS key để mã hóa

2. DISTRIBUTE (Phân Phối)
   └── Ứng dụng gọi API để lấy secret (không hardcode)
   └── Cache secret trong bộ nhớ (tránh gọi API quá nhiều)
   └── SDK tự động refresh khi secret thay đổi

3. ROTATE (Xoay Vòng)
   └── Lambda function thực hiện rotation logic
   └── Test kết nối với secret mới trước khi finalize
   └── Rollback tự động nếu test thất bại

4. AUDIT (Kiểm Toán)
   └── CloudTrail ghi lại mọi GetSecretValue call
   └── CloudWatch Metrics theo dõi rotation success/failure
   └── Access Analyzer phát hiện quyền truy cập không mong muốn

5. REVOKE/DELETE (Thu Hồi/Xóa)
   └── Đặt deletion window (7–30 ngày)
   └── Vô hiệu hóa secret trước khi xóa
   └── Đảm bảo không còn ứng dụng nào dùng secret
```

---

## 📊 So Sánh Nhanh

| Tiêu Chí | Secrets Manager | Parameter Store Standard | Parameter Store Advanced |
|---|---|---|---|
| **Chi Phí** | $0.40/secret/tháng + $0.05/10K API calls | Miễn phí | $0.05/param/tháng |
| **Kích Thước Tối Đa** | 65KB | 4KB | 8KB |
| **Auto-Rotation** | ✅ Native | ❌ | ✅ (qua Secrets Manager) |
| **Phân Cấp (Hierarchy)** | ❌ | ✅ | ✅ |
| **Cross-Account** | ✅ | ❌ | ❌ |
| **Multi-Region Replication** | ✅ | ❌ | ❌ |
| **Database Integration** | ✅ Native (RDS, Redshift) | ❌ | ❌ |

---

## 🔒 Mô Hình Bảo Mật

### Defense-in-Depth (Phòng Thủ Theo Chiều Sâu) cho Secrets

```
Layer 1 — Network:
  VPC Endpoint cho Secrets Manager/SSM (không qua internet)

Layer 2 — Identity:
  IAM policy chỉ cho phép ứng dụng cần thiết (least privilege)
  Resource-based policy (policy tài nguyên) trên secret

Layer 3 — Encryption:
  KMS CMK mã hóa secret at rest
  TLS 1.2+ mã hóa secret in transit

Layer 4 — Audit:
  CloudTrail ghi lại mọi truy cập
  CloudWatch Alarm khi truy cập bất thường
  AWS Config rule kiểm tra rotation đang bật
```

### IAM Policy Ví Dụ — Chỉ Đọc Một Secret

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:prod/myapp/db-*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-southeast-1"
        }
      }
    }
  ]
}
```

---

## 🏗️ Kiến Trúc Tham Chiếu

### Pattern 1: Ứng Dụng Web Đơn Giản

```
EC2 / ECS Task
    │
    ├── IAM Role có quyền GetSecretValue
    │
    └── VPC Endpoint → Secrets Manager
                           └── KMS CMK (mã hóa)
                           └── Auto-rotation (hàng 30 ngày)
```

### Pattern 2: Microservices Đa Môi Trường

```
Parameter Store Hierarchy:
/myapp/
├── prod/
│   ├── db/password        (SecureString — KMS)
│   ├── db/host            (String)
│   └── api/key            (SecureString — KMS)
├── staging/
│   ├── db/password
│   └── api/key
└── dev/
    └── db/password
```

### Pattern 3: Multi-Account Secret Sharing (Chia Sẻ Secret Liên Tài Khoản)

```
Account A (Security Account)
└── Secrets Manager secret
    └── Resource Policy cho phép Account B role đọc

Account B (Application Account)
└── Application role assume → gọi GetSecretValue
    └── Cross-account KMS grant nếu dùng CMK Account A
```

---

## ⚠️ Lỗi Phổ Biến Cần Tránh

| Lỗi | Hậu Quả | Giải Pháp |
|---|---|---|
| Hardcode credentials trong code | Lộ secret qua git history | Dùng Secrets Manager/Parameter Store |
| Log giá trị secret | Lộ trong CloudWatch/S3 | Chỉ log secret ARN, không log giá trị |
| Không bật rotation | Secret không bao giờ thay đổi | Bật rotation với chu kỳ ≤ 90 ngày |
| Quyền quá rộng `secretsmanager:*` | Ứng dụng có thể xóa/sửa secret | Chỉ cấp `GetSecretValue` |
| Không dùng VPC Endpoint | Secret đi qua internet | Tạo VPC Interface Endpoint |
| Xóa secret ngay lập tức | Ứng dụng bị lỗi nếu còn dùng | Luôn dùng deletion window |

---

## 🎓 Câu Hỏi Phỏng Vấn Thường Gặp

1. **Secrets Manager vs Parameter Store — khi nào dùng cái nào?**
   → Xem [3-secrets-vs-parameter.md](3-secrets-vs-parameter.md)

2. **Giải thích cơ chế auto-rotation không gây downtime**
   → Xem [1-secrets-manager.md](1-secrets-manager.md) — phần multi-user rotation

3. **ACM certificate gia hạn tự động hoạt động như thế nào?**
   → Xem [4-acm-certificates.md](4-acm-certificates.md)

4. **Khi nào cần ACM Private CA thay vì ACM Public?**
   → Xem [5-acm-private-ca.md](5-acm-private-ca.md)

5. **Làm thế nào để ứng dụng truy xuất secret an toàn mà không hardcode?**
   → IAM role + SDK + VPC Endpoint

---

## 🚀 Bước Tiếp Theo

```
1. Đọc 1-secrets-manager.md — dịch vụ quan trọng nhất
2. So sánh với 2-parameter-store.md
3. Đọc 3-secrets-vs-parameter.md để biết khi nào dùng gì
4. Học về TLS/SSL qua 4-acm-certificates.md
5. Nếu cần PKI nội bộ → 5-acm-private-ca.md
```

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
