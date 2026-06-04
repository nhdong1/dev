# S3 Object Lock — Khóa Đối Tượng WORM

> S3 Object Lock (Khóa Đối Tượng) thực hiện mô hình WORM (Write Once Read Many — Ghi Một Lần Đọc Nhiều Lần): object không thể bị xóa hoặc ghi đè trong suốt thời gian retention (lưu giữ). Là công cụ quan trọng cho compliance (tuân thủ) với SEC Rule 17a-4, FINRA, HIPAA.

## 📚 Mục Lục

1. [Object Lock Là Gì?](#1-object-lock-là-gì)
2. [Retention Mode — Chế Độ Lưu Giữ](#2-retention-mode--chế-độ-lưu-giữ)
3. [Legal Hold — Giữ Pháp Lý](#3-legal-hold--giữ-pháp-lý)
4. [Bật Object Lock](#4-bật-object-lock)
5. [Cấu Hình Và Quản Lý](#5-cấu-hình-và-quản-lý)
6. [Use Cases Thực Tế](#6-use-cases-thực-tế)
7. [Kết Hợp Với Các Tính Năng Khác](#7-kết-hợp-với-các-tính-năng-khác)
8. [Chi Phí](#8-chi-phí)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Object Lock Là Gì?

Object Lock bảo vệ object khỏi bị xóa hoặc ghi đè bằng cách áp dụng quy tắc lưu giữ bất biến (immutable retention). Hoạt động dựa trên Versioning — mỗi version (phiên bản) của object có thể có retention riêng.

### Hai Cơ Chế Bảo Vệ

```
Object Lock
├── Retention (Lưu Giữ) — dựa trên thời gian
│   ├── Governance Mode  — admin đặc biệt có thể bypass
│   └── Compliance Mode  — tuyệt đối, không ai bypass được
└── Legal Hold (Giữ Pháp Lý) — không thời hạn
    └── Bật/tắt độc lập với Retention
```

### Điều Kiện Tiên Quyết

```
✅ Versioning phải bật TRƯỚC KHI bật Object Lock
✅ Object Lock phải được bật khi TẠO BUCKET (không thể bật sau)
✅ Không thể tắt Object Lock sau khi đã bật
✅ Không thể tắt Versioning khi Object Lock đang bật
```

---

## 2. Retention Mode — Chế Độ Lưu Giữ

### 2.1 Governance Mode — Chế Độ Quản Trị

```
Bảo vệ: Cao — nhưng có thể bypass với quyền đặc biệt
Ai có thể xóa?
  - IAM user/role có quyền s3:BypassGovernanceRetention
  - Phải thêm header: x-amz-bypass-governance-retention: true trong request

Ai KHÔNG thể xóa?
  - Tất cả user thông thường, kể cả bucket owner
  - AWS account root user (trừ khi root có quyền bypass)

Thay đổi retention date?
  - Có thể kéo dài hoặc rút ngắn nếu có quyền bypass

Mục đích:
  - Kiểm tra cấu hình Object Lock trước khi chuyển sang Compliance Mode
  - Bảo vệ khỏi xóa nhầm trong khi vẫn linh hoạt cho admin
  - Workloads không yêu cầu regulatory compliance cứng
```

### 2.2 Compliance Mode — Chế Độ Tuân Thủ

```
Bảo vệ: Tuyệt đối — KHÔNG AI có thể bypass
Ai có thể xóa?
  - KHÔNG AI — kể cả AWS root user, kể cả AWS Support
  - Chỉ có thể xóa khi retention period đã hết hạn

Thay đổi retention date?
  - Chỉ có thể KÉO DÀI, KHÔNG thể rút ngắn
  - Không thể xóa retention hoàn toàn

Mục đích:
  - Regulatory compliance: SEC Rule 17a-4(f), CFTC, FINRA
  - HIPAA — Health Insurance Portability and Accountability Act
  - GDPR — cho dữ liệu phải giữ trong thời gian cố định
  - Bằng chứng pháp lý, hợp đồng, financial records
```

### Bảng So Sánh Governance vs Compliance

| Đặc Điểm | Governance Mode | Compliance Mode |
| --------- | --------------- | --------------- |
| Xóa object trong retention? | Có (cần quyền đặc biệt) | **Không** |
| Root account xóa được? | Có (với bypass header) | **Không** |
| Rút ngắn retention period? | Có (với bypass) | **Không** |
| Kéo dài retention period? | Có | Có |
| Dùng để test? | Phù hợp | Không nên |
| Compliance cứng? | Không | **Có** |

---

## 3. Legal Hold — Giữ Pháp Lý

Legal Hold (Giữ Pháp Lý) là cơ chế bảo vệ độc lập, không có thời hạn cố định.

### Đặc Điểm Legal Hold

```
✅ Không phụ thuộc vào Retention period
✅ Bật và tắt bởi user có quyền s3:PutObjectLegalHold
✅ Không có ngày hết hạn — duy trì cho đến khi được tắt thủ công
✅ Có thể áp dụng cùng lúc với Retention
❌ Nếu Legal Hold đang bật, KHÔNG thể xóa dù Retention đã hết hạn
```

### Khi Nào Dùng Legal Hold

```
1. Điều tra pháp lý (litigation hold — giữ tài liệu khi có vụ kiện)
2. Kiểm toán nội bộ đang diễn ra
3. Dữ liệu đang bị tranh chấp
4. Compliance investigation (điều tra tuân thủ)

Workflow:
- Pháp lý yêu cầu giữ tài liệu → Bật Legal Hold
- Điều tra kết thúc → Tắt Legal Hold
- Object bị bảo vệ trong toàn bộ quá trình điều tra, bất kể retention period
```

### Tương Tác Giữa Retention Và Legal Hold

```
Trạng thái              Xóa được không?
───────────────────────────────────────
Retention còn hạn + Legal Hold = ON    → KHÔNG
Retention còn hạn + Legal Hold = OFF   → KHÔNG (Retention bảo vệ)
Retention hết hạn + Legal Hold = ON    → KHÔNG (Legal Hold bảo vệ)
Retention hết hạn + Legal Hold = OFF   → CÓ THỂ XÓA
```

---

## 4. Bật Object Lock

### Bật Khi Tạo Bucket (Duy Nhất Có Thể)

**Qua AWS CLI:**
```bash
aws s3api create-bucket \
  --bucket my-compliance-bucket \
  --region us-east-1 \
  --object-lock-enabled-for-bucket
```

**Qua AWS CloudFormation:**
```yaml
Resources:
  ComplianceBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-compliance-bucket
      ObjectLockEnabled: true
      VersioningConfiguration:
        Status: Enabled
      ObjectLockConfiguration:
        ObjectLockEnabled: Enabled
        Rule:
          DefaultRetention:
            Mode: COMPLIANCE
            Years: 7
```

### Đặt Default Retention (Lưu Giữ Mặc Định)

Default Retention áp dụng tự động cho mọi object mới nếu không có retention riêng:

```bash
aws s3api put-object-lock-configuration \
  --bucket my-compliance-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Years": 7
      }
    }
  }'
```

Retention period có thể là `Days` hoặc `Years`, nhưng không dùng cả hai cùng lúc.

---

## 5. Cấu Hình Và Quản Lý

### 5.1 Áp Dụng Retention Khi Upload

```bash
# Chỉ định ngày hết hạn cụ thể
aws s3api put-object \
  --bucket my-compliance-bucket \
  --key financial-report-2026.pdf \
  --body report.pdf \
  --object-lock-mode COMPLIANCE \
  --object-lock-retain-until-date "2033-01-01T00:00:00Z"

# Hoặc dùng default retention từ bucket configuration (không cần chỉ định)
aws s3api put-object \
  --bucket my-compliance-bucket \
  --key audit-log-2026.csv \
  --body audit.csv
```

### 5.2 Quản Lý Legal Hold

```bash
# Bật Legal Hold
aws s3api put-object-legal-hold \
  --bucket my-compliance-bucket \
  --key contract-2026.pdf \
  --version-id "abc123xyz" \
  --legal-hold Status=ON

# Tắt Legal Hold (sau khi điều tra xong)
aws s3api put-object-legal-hold \
  --bucket my-compliance-bucket \
  --key contract-2026.pdf \
  --version-id "abc123xyz" \
  --legal-hold Status=OFF
```

### 5.3 Kiểm Tra Trạng Thái Object Lock

```bash
# Xem retention và legal hold của object
aws s3api get-object-retention \
  --bucket my-compliance-bucket \
  --key financial-report-2026.pdf \
  --version-id "abc123xyz"

aws s3api get-object-legal-hold \
  --bucket my-compliance-bucket \
  --key financial-report-2026.pdf \
  --version-id "abc123xyz"
```

### 5.4 Bypass Governance Mode (Với Quyền Đặc Biệt)

```bash
# Xóa object đang ở Governance Mode (cần IAM permission đặc biệt)
aws s3api delete-object \
  --bucket my-governance-bucket \
  --key test-object.txt \
  --version-id "xyz789" \
  --bypass-governance-retention
```

IAM policy cho phép bypass:
```json
{
  "Effect": "Allow",
  "Action": "s3:BypassGovernanceRetention",
  "Resource": "arn:aws:s3:::my-governance-bucket/*"
}
```

---

## 6. Use Cases Thực Tế

### Use Case 1: Financial Records — SEC Rule 17a-4(f)

```
Yêu cầu: Broker-dealer (môi giới chứng khoán) phải lưu giữ electronic records
6 năm, không thể xóa, không thể sửa.

Cấu hình:
- Object Lock: COMPLIANCE Mode
- Retention: 6 năm (2191 ngày)
- Default Retention: Bật để mọi object tự động được bảo vệ
- Encryption: SSE-KMS với KMS key riêng cho compliance bucket
- S3 Access Logs: Bật để audit trail đầy đủ

Kiến trúc:
Trading System → PUT records → compliance-records-bucket
                               (Object Lock COMPLIANCE 6 năm)
                                        │
                               CloudTrail API logs
                               S3 Access Logs
                               (Audit trail đầy đủ)
```

### Use Case 2: Healthcare Records — HIPAA

```
Yêu cầu: Patient records (hồ sơ bệnh nhân) phải giữ ít nhất 6 năm.

Cấu hình:
- Object Lock: COMPLIANCE Mode, 6 năm
- Encryption: SSE-KMS với CMK (Customer Managed Key — Khóa Khách Hàng Quản Lý)
- S3 Block Public Access: Bật toàn bộ
- VPC Endpoint: Chỉ cho phép truy cập từ VPC nội bộ

Legal Hold Workflow:
- Bệnh nhân khiếu nại → Bật Legal Hold cho record liên quan
- Giải quyết khiếu nại → Tắt Legal Hold
```

### Use Case 3: Backup Immutability — Bảo Vệ Backup

```
Bài toán: Ransomware (phần mềm tống tiền) mã hóa backup files
→ Attacker xóa backup cũ để ép nạn nhân trả tiền

Giải pháp:
- Backup bucket với Object Lock GOVERNANCE Mode
- Retention: 30 ngày (đủ để restore)
- Attacker không thể xóa backup trong 30 ngày
- Admin có thể xóa object cũ bình thường (sau 30 ngày hoặc với bypass)

Tại sao GOVERNANCE chứ không phải COMPLIANCE?
→ Admin cần xóa backup cũ hơn 30 ngày để tiết kiệm chi phí
→ COMPLIANCE quá cứng — không ai xóa được kể cả khi retention hết hạn sớm hơn dự kiến
```

### Use Case 4: Code Artifact Immutability

```
Bài toán: Container images, deployment artifacts phải bất biến.
Phiên bản đã deployed không được phép sửa đổi (supply chain security).

Cấu hình:
- artifact-bucket với Object Lock GOVERNANCE
- Retention: 90 ngày (thời gian deployment lifecycle)
- CI/CD pipeline: push artifact → bucket
- Deploy stage: chỉ đọc từ bucket, không ghi
```

---

## 7. Kết Hợp Với Các Tính Năng Khác

### Object Lock + Versioning

```
Object Lock yêu cầu Versioning. Mỗi version độc lập có retention riêng.

Version 1 (retention đến 2030) ← Protected
Version 2 (retention đến 2031) ← Protected
Version 3 (retention đến 2032) ← Protected (current)

Xóa current version → tạo delete marker (không xóa version thật)
Xóa delete marker cũng có thể được bảo vệ tùy cấu hình
```

### Object Lock + Replication

```
CRR/SRR replicate cả Object Lock metadata:
- Retention Mode và date được copy sang bucket đích
- Legal Hold status được copy
- Bucket đích phải có Object Lock bật

Dùng cho: Lưu giữ compliance record ở hai region
```

### Object Lock + Lifecycle

```
Lifecycle KHÔNG thể xóa object đang bị Object Lock bảo vệ.
Lifecycle sẽ bỏ qua object này cho đến khi retention hết hạn.

Sau khi retention hết hạn:
→ Lifecycle rules áp dụng bình thường (xóa hoặc chuyển tầng)
```

### Object Lock + S3 Batch Operations

```
Có thể dùng Batch Operations để:
- Áp dụng Object Lock cho hàng tỷ object một lúc
- Kéo dài retention period cho nhiều object
- Bật/tắt Legal Hold hàng loạt
```

---

## 8. Chi Phí

Object Lock **không tính phí riêng** — chỉ tính phí theo storage class và request thông thường.

Lưu ý chi phí gián tiếp:
- Object bị lock không thể xóa → storage cost tích lũy cho đến hết retention
- Kế hoạch storage cost phải tính trước dựa trên retention period
- Object Lock + Deep Archive: Cân bằng giữa compliance và chi phí lưu trữ dài hạn

```
Chiến lược tiết kiệm chi phí khi dùng Object Lock:
- Ngay khi upload → Standard hoặc Standard-IA
- Sau 30 ngày → Lifecycle chuyển sang Glacier Instant (nếu compliance cho phép)
- Sau 90 ngày → Lifecycle chuyển sang Deep Archive
- Object Lock vẫn hoạt động ở Deep Archive
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q1: Sự khác biệt chính giữa Governance Mode và Compliance Mode?**

> Governance Mode: User có quyền `s3:BypassGovernanceRetention` và thêm header đặc biệt có thể xóa hoặc rút ngắn retention — phù hợp để kiểm tra và bảo vệ linh hoạt. Compliance Mode: Tuyệt đối không ai có thể xóa hoặc rút ngắn retention kể cả root account — dành cho regulatory compliance cứng như SEC 17a-4. Một khi đặt Compliance Mode, chỉ có thể kéo dài retention, không thể rút ngắn.

**Q2: Tại sao Object Lock phải bật khi tạo bucket, không thể bật sau?**

> Đây là thiết kế có chủ ý của AWS. Nếu có thể bật Object Lock sau, attacker có thể tấn công dữ liệu trước khi admin kịp bật. Việc bật khi tạo bucket đảm bảo mọi object trong bucket đó đều được bảo vệ ngay từ đầu, đồng thời việc không thể tắt sau khi bật tạo sự đảm bảo pháp lý cho regulatory compliance.

**Q3: Legal Hold và Retention hoạt động độc lập như thế nào?**

> Legal Hold và Retention bảo vệ theo hai chiều khác nhau. Cả hai phải đồng thời không hoạt động thì mới có thể xóa object. Nếu Retention hết hạn nhưng Legal Hold còn bật → không xóa được. Nếu Legal Hold tắt nhưng Retention còn hạn → không xóa được. Chỉ khi Retention hết hạn VÀ Legal Hold tắt thì mới xóa được.

**Q4: Có thể dùng Object Lock với S3 Glacier không?**

> Có. Object Lock hoạt động với mọi storage class bao gồm Glacier và Deep Archive. Đây là lợi thế lớn — có thể archive dữ liệu compliance dài hạn với chi phí rất thấp ($0.00099/GB ở Deep Archive) trong khi vẫn duy trì tính bất biến. Lifecycle rules có thể chuyển object sang Glacier trong khi Object Lock vẫn bảo vệ.

**Q5: Làm sao backup protected bucket khi cần kiểm thử restore?**

> Dùng Governance Mode thay vì Compliance Mode cho môi trường backup/test. Admin có quyền bypass Governance Mode để restore và kiểm thử. Sau khi xác nhận restore quy trình hoạt động, dữ liệu production thật dùng Compliance Mode. Tốt hơn nữa, dùng S3 Batch Operations để copy object sang bucket test riêng không có Object Lock để kiểm thử thoải mái hơn.

---

## 📋 Checklist Compliance Deployment

- [ ] Tạo bucket với Object Lock khi khởi tạo (không thể thêm sau)
- [ ] Xác nhận Retention Mode đúng (Governance để test, Compliance để production)
- [ ] Tính toán retention period phù hợp với quy định (SEC 6 năm, HIPAA 6 năm, GDPR tùy loại data)
- [ ] Đặt Default Retention để tự động bảo vệ mọi object mới
- [ ] Bật Encryption với CMK (Customer Managed Key) để audit trail
- [ ] Bật S3 Access Logs và CloudTrail để audit ai đã truy cập gì
- [ ] Test quy trình Legal Hold: bật → xác nhận không xóa được → tắt → xác nhận xóa được
- [ ] Lên kế hoạch chi phí storage dài hạn (retention period = chi phí bắt buộc)
- [ ] Kết hợp Lifecycle rules để chuyển sang Glacier sau 90 ngày (tiết kiệm chi phí)

---

**Tiếp Theo:** [5-s3-event-notifications.md](./5-s3-event-notifications.md) — Event-driven architecture với S3
