# 07 — Monitoring & Auditing (Giám Sát & Kiểm Toán)

> **Security Observability Strategy** — Chiến lược quan sát và kiểm toán bảo mật toàn diện trên AWS

---

## 🎯 Mục Tiêu Module

Sau khi hoàn thành module này, bạn có thể:

- Thiết kế và triển khai chiến lược **Security Observability** (Quan Sát Bảo Mật) đa tầng
- Cấu hình **CloudTrail** để ghi nhật ký kiểm toán không thể giả mạo
- Phân tích log bảo mật với **Athena** và **CloudTrail Lake**
- Triển khai **AWS Config Rules** (Quy Tắc Cấu Hình) để phát hiện vi phạm tuân thủ tự động
- Dùng **IAM Access Analyzer** (Bộ Phân Tích Truy Cập) để lộ quyền không mong muốn
- Xây dựng **Security Dashboard** (Bảng Điều Khiển Bảo Mật) với CloudWatch

---

## 📁 Cấu Trúc Module

```
07-monitoring-auditing/
├── README.md                   ✅ Tổng quan & chiến lược (file này)
├── 1-cloudtrail-setup.md       ✅ Trail config, S3 integrity, CloudTrail Lake
├── 2-cloudtrail-analysis.md    ✅ Phân tích log, Athena queries, phát hiện bất thường
├── 3-aws-config-rules.md       ✅ Managed rules, custom rules, auto-remediation
├── 4-access-analyzer.md        ✅ Phát hiện external/cross-account access
└── 5-security-dashboard.md     ✅ CloudWatch dashboards & alarms cho security
```

---

## 🗺️ Tổng Quan Kiến Trúc Security Observability

```
┌─────────────────────────────────────────────────────────────┐
│               AWS Security Observability Stack               │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  CloudTrail  │  │  AWS Config  │  │  Access Analyzer │  │
│  │ (API Logging)│  │(Config Track)│  │ (Access Review)  │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │            │
│         ▼                 ▼                    ▼            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              CloudWatch Logs + Metrics               │    │
│  │              (Tổng hợp & cảnh báo thời gian thực)   │    │
│  └──────────────────────────┬──────────────────────────┘    │
│                             │                               │
│         ┌───────────────────┼───────────────────┐          │
│         ▼                   ▼                   ▼          │
│  ┌────────────┐   ┌──────────────────┐  ┌─────────────┐   │
│  │  Athena    │   │ CloudTrail Lake   │  │  Security   │   │
│  │ (SQL Query)│   │  (Long-term SQL)  │  │  Dashboard  │   │
│  └────────────┘   └──────────────────┘  └─────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Ba Tầng Quan Sát

| Tầng | Dịch vụ | Mục đích |
|---|---|---|
| **Ghi nhận (Collection)** | CloudTrail, VPC Flow Logs, Config | Thu thập sự kiện thô |
| **Phân tích (Analysis)** | Athena, CloudTrail Lake, Logs Insights | Truy vấn và tìm kiếm mẫu |
| **Cảnh báo (Alerting)** | CloudWatch Alarms, EventBridge, SNS | Thông báo và phản hồi tự động |

---

## 🔑 Dịch Vụ Cốt Lõi

### 1. AWS CloudTrail — Nhật Ký Kiểm Toán API

**CloudTrail** ghi lại mọi lời gọi API trong tài khoản AWS — ai làm gì, khi nào, từ đâu, và kết quả là gì.

```
Không có CloudTrail = Không có bằng chứng kiểm toán
Có CloudTrail = Toàn bộ lịch sử hành động có thể truy vết
```

**Các loại sự kiện CloudTrail:**

| Loại | Ví dụ | Mặc định |
|---|---|---|
| **Management Events** (Sự kiện quản lý) | `CreateBucket`, `RunInstances`, `AssumeRole` | ✅ Bật sẵn |
| **Data Events** (Sự kiện dữ liệu) | S3 GetObject/PutObject, Lambda Invoke | ❌ Phải bật thêm |
| **Insights Events** (Sự kiện bất thường) | Phát hiện spike API bất thường | ❌ Phải bật thêm |

Chi tiết: [1-cloudtrail-setup.md](1-cloudtrail-setup.md)

---

### 2. AWS Config — Kiểm Soát Tuân Thủ Cấu Hình

**AWS Config** theo dõi trạng thái cấu hình của tài nguyên AWS theo thời gian và đánh giá tuân thủ so với rules.

```
Config ≠ CloudTrail
CloudTrail: "Ai đã làm gì?"  (hành động)
Config:     "Cấu hình đang là gì?" (trạng thái)
```

**Hai câu hỏi Config trả lời:**
1. *"S3 bucket này có public không?"* — Đánh giá thời gian thực
2. *"3 tuần trước, Security Group này trông như thế nào?"* — Lịch sử cấu hình

Chi tiết: [3-aws-config-rules.md](3-aws-config-rules.md)

---

### 3. IAM Access Analyzer — Phân Tích Quyền Truy Cập

**IAM Access Analyzer** — Bộ Phân Tích Truy Cập IAM — tự động tìm kiếm các tài nguyên chia sẻ với bên ngoài tổ chức hoặc ngoài ý muốn.

**Những gì Access Analyzer phát hiện:**

- S3 bucket được cấp public access
- IAM Role có trust policy cho phép external AWS account
- KMS key được chia sẻ với tài khoản khác
- Lambda function có resource-based policy mở
- SQS queue có public policy

Chi tiết: [4-access-analyzer.md](4-access-analyzer.md)

---

### 4. CloudWatch — Giám Sát Và Cảnh Báo

**Amazon CloudWatch** là nền tảng observability của AWS — thu thập metrics, logs, và kích hoạt alarms.

**Trong bối cảnh security:**

```
CloudTrail Logs → CloudWatch Logs → Metric Filters → Alarms → SNS
```

Ví dụ cảnh báo: Root account login, Security Group thay đổi, MFA tắt...

Chi tiết: [5-security-dashboard.md](5-security-dashboard.md)

---

## 🔗 Mối Quan Hệ Giữa Các Dịch Vụ

```
Sự cố bảo mật xảy ra
        │
        ▼
┌──────────────────────────────────────┐
│  CloudTrail ghi lại API call         │
│  (ai, làm gì, khi nào, từ IP nào)    │
└──────────────────┬───────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
┌───────────────┐    ┌─────────────────┐
│  Config ghi   │    │  CloudWatch     │
│  trạng thái   │    │  Logs alert     │
│  thay đổi     │    │  realtime       │
└───────┬───────┘    └────────┬────────┘
        │                     │
        ▼                     ▼
┌───────────────┐    ┌─────────────────┐
│  Config Rule  │    │  SNS → Slack /  │
│  KHÔNG_TUÂN   │    │  PagerDuty /   │
│  THỦ → Alert  │    │  Email         │
└───────────────┘    └─────────────────┘
```

---

## 📋 Checklist Thiết Lập Cơ Bản

### Mức Tối Thiểu (Bắt Buộc Cho Mọi Account)

- [ ] **CloudTrail**: Bật multi-region trail, bật log validation
- [ ] **S3 Bucket Logging**: Gửi CloudTrail logs vào S3 có MFA Delete
- [ ] **AWS Config**: Bật recorder, gửi notifications
- [ ] **CloudWatch Alarms**: Root login, API error spikes
- [ ] **Access Analyzer**: Tạo analyzer cho organization hoặc account

### Mức Nâng Cao (Production / Regulated Environment)

- [ ] **CloudTrail Lake**: Bật để truy vấn SQL dài hạn
- [ ] **CloudTrail Insights**: Bật phát hiện bất thường tự động
- [ ] **Config Conformance Packs**: CIS AWS Benchmark
- [ ] **Security Hub**: Tổng hợp findings (module 08)
- [ ] **Athena + Glue**: ETL pipeline phân tích log tự động
- [ ] **Custom Config Rules**: Lambda-based rules cho chính sách nội bộ

---

## ⚡ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: CloudTrail và AWS Config khác nhau như thế nào?**

> CloudTrail theo dõi **hành động** (API calls) — ai làm gì, khi nào. AWS Config theo dõi **trạng thái cấu hình** tài nguyên — cấu hình hiện tại là gì, có thay đổi gì không. Khi điều tra sự cố, bạn dùng CloudTrail để biết ai thay đổi, dùng Config để xem cấu hình trước và sau thay đổi.

**Q: Làm sao phát hiện root account bị dùng?**

> Tạo CloudWatch Metric Filter trên CloudTrail logs lọc `userIdentity.type = "Root"`, tạo Alarm khi count > 0, gửi SNS notification ngay lập tức.

**Q: Access Analyzer phát hiện được gì mà IAM Policy review không thể?**

> Access Analyzer dùng **automated reasoning** (lý luận tự động) để phân tích toàn bộ policy graph thay vì review thủ công từng policy. Nó phát hiện được các trường hợp cross-account access phức tạp mà human review có thể bỏ sót.

---

## 📊 Chi Phí Ước Tính

| Dịch vụ | Mô hình tính phí | Ghi chú |
|---|---|---|
| CloudTrail | $2/100,000 management events | Management events trong region đầu tiên miễn phí |
| CloudTrail Lake | $2.75/GB ingested | Retention 7 năm |
| AWS Config | $0.003/configuration item | Tính theo mỗi lần ghi nhận thay đổi |
| Config Rules | $1/active rule/month | + $0.001/evaluation |
| Access Analyzer | Miễn phí | Không tốn chi phí trực tiếp |
| CloudWatch Logs | $0.50/GB ingested | Lưu trữ $0.03/GB/month |

---

## 🔗 Điều Hướng

| File | Nội Dung |
|---|---|
| [1-cloudtrail-setup.md](1-cloudtrail-setup.md) | Cấu hình CloudTrail, bảo vệ log, CloudTrail Lake |
| [2-cloudtrail-analysis.md](2-cloudtrail-analysis.md) | Phân tích log với Athena, SQL queries, phát hiện bất thường |
| [3-aws-config-rules.md](3-aws-config-rules.md) | Config rules, remediation tự động, conformance packs |
| [4-access-analyzer.md](4-access-analyzer.md) | IAM Access Analyzer, phát hiện external access |
| [5-security-dashboard.md](5-security-dashboard.md) | CloudWatch dashboards, metric filters, alarms |

---

**Cập Nhật Lần Cuối:** 2026-05-16
**Phiên Bản:** 1.0
