# AWS Config — Tuân Thủ Cấu Hình Tài Nguyên

> **AWS Config** là dịch vụ theo dõi cấu hình (configuration) của tài nguyên AWS theo thời gian, đánh giá tuân thủ (compliance) tự động theo rules, và hỗ trợ tự động khắc phục (automated remediation) sai lệch. Đây là nền tảng của compliance-as-code trên AWS.

---

## 📚 Mục Lục

1. [AWS Config Là Gì?](#aws-config-là-gì)
2. [Kiến Trúc Tổng Thể](#kiến-trúc-tổng-thể)
3. [Các Thành Phần Chính](#các-thành-phần-chính)
4. [Luồng Hoạt Động](#luồng-hoạt-động)
5. [So Sánh Với CloudTrail & CloudWatch](#so-sánh-với-cloudtrail--cloudwatch)
6. [Pricing & Giới Hạn](#pricing--giới-hạn)
7. [Câu Hỏi Phỏng Vấn Trọng Tâm](#câu-hỏi-phỏng-vấn-trọng-tâm)
8. [Điều Hướng Nội Dung](#điều-hướng-nội-dung)

---

## AWS Config Là Gì?

**AWS Config** trả lời ba câu hỏi cốt lõi về hạ tầng:

| Câu Hỏi | AWS Config Trả Lời Như Thế Nào |
|---------|-------------------------------|
| **Cấu hình tài nguyên hiện tại là gì?** | Configuration Recorder ghi lại trạng thái tức thời |
| **Cấu hình có thay đổi gì theo thời gian?** | Configuration History lưu toàn bộ lịch sử thay đổi |
| **Cấu hình có đúng policy không?** | Config Rules đánh giá COMPLIANT / NON_COMPLIANT |

### Điểm Khác Biệt Cốt Lõi

```
CloudTrail  → "Ai đã làm gì với API?" (WHO + WHEN + WHAT action)
AWS Config  → "Cấu hình tài nguyên có đúng không?" (WHAT state + COMPLIANT?)
CloudWatch  → "Hiệu suất/sức khỏe hệ thống ra sao?" (PERFORMANCE + HEALTH)
```

---

## Kiến Trúc Tổng Thể

```
┌──────────────────────────────────────────────────────────────────┐
│                        AWS Account                               │
│                                                                  │
│  ┌─────────────┐    ┌──────────────────┐    ┌────────────────┐  │
│  │  Tài Nguyên │───▶│  Configuration   │───▶│  Config Rules  │  │
│  │  AWS        │    │  Recorder        │    │  (Quy Tắc)     │  │
│  │  (EC2, S3,  │    │  (Bộ Ghi Cấu    │    │                │  │
│  │   RDS, IAM) │    │   Hình)          │    │  COMPLIANT ✅  │  │
│  └─────────────┘    └────────┬─────────┘    │  NON_COMPLIANT │  │
│                              │               │  ❌            │  │
│                              ▼               └───────┬────────┘  │
│                   ┌──────────────────┐               │           │
│                   │   S3 Bucket      │               ▼           │
│                   │  (Configuration  │    ┌────────────────────┐ │
│                   │   History &      │    │  Remediation       │ │
│                   │   Snapshots)     │    │  Actions           │ │
│                   └──────────────────┘    │  (SSM Automation)  │ │
│                                           └────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                   ┌──────────────────┐
                   │   Aggregator     │
                   │  (Multi-Account  │
                   │   Compliance     │
                   │   Dashboard)     │
                   └──────────────────┘
```

---

## Các Thành Phần Chính

### 1. Configuration Recorder — Bộ Ghi Cấu Hình

Theo dõi và ghi lại thay đổi cấu hình của tài nguyên AWS.

- **Bật/tắt** thủ công hoặc qua CloudFormation/Terraform
- **Chọn resource types** cần theo dõi (tất cả hoặc chọn lọc)
- **Delivery channel** — Kênh giao nhận đến S3 bucket

> Chi tiết: [1-configuration-recorder.md](1-configuration-recorder.md)

### 2. Config Rules — Quy Tắc Đánh Giá Tuân Thủ

Định nghĩa trạng thái cấu hình "đúng" cho tài nguyên.

- **AWS Managed Rules** — Hơn 250 rules được AWS xây dựng sẵn
- **Custom Rules** — Lambda function tự viết, hoặc Guard rules
- **Evaluation trigger** — Thay đổi cấu hình hoặc theo lịch (periodic)

> Chi tiết: [2-config-rules.md](2-config-rules.md)

### 3. Conformance Packs — Gói Tuân Thủ Framework

Bộ rules đóng gói sẵn theo compliance framework tiêu chuẩn.

- **CIS AWS Foundations Benchmark**
- **PCI-DSS** — Payment Card Industry Data Security Standard
- **NIST 800-53** — National Institute of Standards and Technology
- **HIPAA** — Health Insurance Portability and Accountability Act
- **Tùy chỉnh** — Xây dựng conformance pack riêng

> Chi tiết: [3-conformance-packs.md](3-conformance-packs.md)

### 4. Remediation Actions — Hành Động Khắc Phục

Tự động hoặc thủ công khắc phục tài nguyên NON_COMPLIANT.

- **Manual Remediation** — Người dùng chủ động kích hoạt
- **Automatic Remediation** — Tự động kích hoạt khi phát hiện vi phạm
- **SSM Automation Documents** — Runbook thực hiện sửa lỗi

> Chi tiết: [4-remediation.md](4-remediation.md)

### 5. Aggregator — Tổng Hợp Đa Account/Region

Tập hợp dữ liệu compliance từ nhiều account và region vào một nơi.

- **Source accounts** — Tài khoản nguồn cần thu thập
- **Aggregator account** — Tài khoản trung tâm hiển thị dashboard
- **Tích hợp Organizations** — Tự động bao gồm tất cả member accounts

> Chi tiết: [5-aggregator-multiregion.md](5-aggregator-multiregion.md)

---

## Luồng Hoạt Động

### Kịch Bản: Phát Hiện S3 Bucket Public và Tự Động Khắc Phục

```
1. Developer tạo S3 bucket, bật public access
       ↓
2. Configuration Recorder phát hiện thay đổi
       ↓
3. Config Rule "s3-bucket-public-read-prohibited" đánh giá
       ↓
4. Trạng thái → NON_COMPLIANT ❌
       ↓
5. EventBridge nhận sự kiện compliance change
       ↓
6. SNS gửi thông báo cho Security team
       ↓
7. Remediation Action kích hoạt SSM Automation
       ↓
8. SSM Automation chạy script: Tắt public access
       ↓
9. Config Rule đánh giá lại → COMPLIANT ✅
       ↓
10. Ghi lại bằng chứng vào S3 (audit evidence)
```

---

## So Sánh Với CloudTrail & CloudWatch

| Tiêu Chí | CloudTrail | AWS Config | CloudWatch |
|---------|-----------|-----------|-----------|
| **Mục đích** | Ghi nhật ký API call | Theo dõi cấu hình & tuân thủ | Giám sát hiệu suất & sức khỏe |
| **Câu hỏi** | "Ai làm gì?" | "Cấu hình có đúng?" | "Hệ thống hoạt động thế nào?" |
| **Dữ liệu** | API events | Resource configuration | Metrics, Logs |
| **Phát hiện** | Thay đổi thủ công | Sai cấu hình (misconfiguration) | Ngưỡng hiệu suất |
| **Remediation** | Không | Có (tích hợp SSM) | Có (qua Alarms) |
| **Compliance** | Audit trail | Configuration compliance | Performance SLA |
| **Lưu trữ** | S3 | S3 + Config service | CloudWatch Logs + S3 |

### Phối Hợp Ba Dịch Vụ

```
CloudTrail  → Ghi lại: IAM user X đã ModifyDBInstance lúc 14:32
AWS Config  → Ghi lại: RDS instance mất Multi-AZ (NON_COMPLIANT)
CloudWatch  → Cảnh báo: RDS connection errors tăng 300%
```

Ba dịch vụ bổ sung cho nhau để tạo bức tranh toàn diện về trạng thái và lịch sử hệ thống.

---

## Pricing & Giới Hạn

### Chi Phí

| Thành Phần | Đơn Giá (us-east-1) |
|-----------|---------------------|
| Configuration items (mỗi lần ghi) | $0.003 / item |
| Config Rule evaluations | $0.001 / evaluation (Managed Rules) |
| Custom Rule evaluations | $0.001 / evaluation |
| Conformance Pack evaluations | $0.001 / evaluation |

> **Lưu ý:** S3 storage cho snapshots và history tính theo giá S3 thông thường.

### Giới Hạn Mặc Định (có thể tăng)

| Giới Hạn | Mặc Định |
|---------|---------|
| Config Rules per region | 150 |
| Remediation configurations | 150 |
| Conformance packs per account | 50 |
| Aggregators per account | 5 |

### Tối Ưu Chi Phí

- Chỉ bật recorder cho resource types cần thiết, không bật "all"
- Xem lại rules không còn relevant định kỳ
- Dùng periodic evaluation thay change-triggered khi có thể

---

## Câu Hỏi Phỏng Vấn Trọng Tâm

### Câu 1: AWS Config khác CloudTrail như thế nào?

**Trả lời:**
- **CloudTrail** ghi lại *hành động API* — ai đã gọi API gì, lúc nào, từ IP nào.
- **AWS Config** theo dõi *trạng thái cấu hình* tài nguyên theo thời gian và đánh giá xem cấu hình đó có tuân thủ policy hay không.
- **Ví dụ thực tế:** CloudTrail ghi "User A gọi ModifySecurityGroup lúc 15:00". Config ghi "Security group SG-123 hiện có rule mở port 0.0.0.0/0 — NON_COMPLIANT với rule `restricted-ssh`".

### Câu 2: Khi nào dùng Managed Rules, khi nào viết Custom Rules?

**Trả lời:**
- **Managed Rules** — Khi AWS đã có rule sẵn phù hợp (>250 rules). Ưu tiên dùng trước vì không cần maintain code.
- **Custom Lambda Rules** — Khi logic phức tạp, cần kiểm tra cross-resource (VD: EC2 phải có tag Cost-Center khớp với danh sách trong DynamoDB), hoặc kiểm tra tài nguyên bên thứ ba.
- **Custom Guard Rules** — Dùng ngôn ngữ policy-as-code CloudFormation Guard, phù hợp khi muốn tránh quản lý Lambda.

### Câu 3: Tự động remediation hoạt động như thế nào?

**Trả lời:**
AWS Config Remediation Action liên kết mỗi Config Rule với một SSM Automation Document. Khi tài nguyên được đánh giá NON_COMPLIANT:
1. Config phát sự kiện lên EventBridge
2. Remediation Action kích hoạt SSM Automation Document tương ứng
3. SSM Document thực thi các bước sửa lỗi (VD: `AWS-DisableS3BucketPublicReadWrite`)
4. Config đánh giá lại — nếu đã sửa thành công → COMPLIANT

### Câu 4: Conformance Pack là gì và khác Config Rule như thế nào?

**Trả lời:**
- **Config Rule** là đơn vị đơn lẻ đánh giá một khía cạnh cụ thể.
- **Conformance Pack** là *bộ sưu tập* nhiều Config Rules và Remediation Actions đóng gói lại theo một compliance framework (CIS, PCI-DSS, HIPAA...). Ưu điểm: triển khai/xóa toàn bộ bộ rules một lần, dễ map sang yêu cầu audit.

---

## Điều Hướng Nội Dung

| File | Nội Dung |
|------|---------|
| [1-configuration-recorder.md](1-configuration-recorder.md) | Recorder setup, resource types, delivery channel, S3 |
| [2-config-rules.md](2-config-rules.md) | Managed rules, Custom Lambda rules, evaluation scope |
| [3-conformance-packs.md](3-conformance-packs.md) | CIS, PCI-DSS, NIST, HIPAA conformance packs |
| [4-remediation.md](4-remediation.md) | Manual & automatic remediation với SSM Automation |
| [5-aggregator-multiregion.md](5-aggregator-multiregion.md) | Aggregator, multi-account compliance dashboard |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành
