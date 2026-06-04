# CloudTrail — Kiểm Toán & Ghi Nhật Ký API Toàn Diện

> **Amazon CloudTrail** (Dấu Vết Đám Mây) ghi lại mọi lời gọi API (API Call) trên AWS — ai làm gì, lúc nào, từ địa chỉ IP nào, thành công hay thất bại. CloudTrail là nền tảng của audit logging (ghi nhật ký kiểm toán), security investigation (điều tra bảo mật) và compliance (tuân thủ) trên AWS.

---

## 📚 Mục Lục

1. [Tại Sao Cần CloudTrail?](#tại-sao-cần-cloudtrail)
2. [Kiến Trúc Tổng Thể](#kiến-trúc-tổng-thể)
3. [Các Thành Phần Cốt Lõi](#các-thành-phần-cốt-lõi)
4. [CloudTrail vs CloudWatch vs AWS Config](#cloudtrail-vs-cloudwatch-vs-aws-config)
5. [Pricing Overview](#pricing-overview)
6. [Câu Hỏi Phỏng Vấn Thường Gặp](#câu-hỏi-phỏng-vấn-thường-gặp)
7. [Điều Hướng Module](#điều-hướng-module)

---

## Tại Sao Cần CloudTrail?

Trong môi trường cloud, hàng nghìn API call xảy ra mỗi giây — từ người dùng, ứng dụng, CI/CD pipelines, đến các dịch vụ AWS tự gọi nhau. Không có audit trail (dấu vết kiểm toán), bạn không thể trả lời:

- **"Ai đã xóa S3 bucket production lúc 3 giờ sáng?"**
- **"Ai đã thay đổi Security Group cho phép traffic từ 0.0.0.0/0?"**
- **"EC2 instance này được tạo bởi user nào, từ IP nào?"**
- **"Root account có được dùng trong 90 ngày qua không?"** (yêu cầu PCI-DSS)

CloudTrail là công cụ trả lời **"Ai đã làm gì?"** — câu hỏi cốt lõi của mọi security audit và incident investigation.

### Ba Câu Hỏi CloudTrail Trả Lời

| Câu Hỏi         | Ví Dụ Thực Tế                                                        |
| --------------- | --------------------------------------------------------------------- |
| **Who?**        | IAM user `john.doe`, role `CodeDeployRole`, AWS service `lambda.amazonaws.com` |
| **What?**       | `DeleteBucket`, `RunInstances`, `AttachRolePolicy`, `PutObject`       |
| **When & Where?** | `2026-05-17T03:42:17Z` từ IP `203.0.113.45` tại region `us-east-1` |

---

## Kiến Trúc Tổng Thể

```
┌─────────────────────────────────────────────────────────────────────┐
│                   API CALL SOURCES (Nguồn Gọi API)                  │
│                                                                     │
│  AWS Console │ AWS CLI │ SDK │ CloudFormation │ Lambda │ Services   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  Mọi API call đều đi qua
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS CONTROL PLANE                            │
│                   (Mặt Phẳng Điều Khiển AWS)                        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         CLOUDTRAIL                                  │
│                                                                     │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │ EVENT HISTORY   │  │     TRAILS       │  │    INSIGHTS       │  │
│  │ (90 ngày free)  │  │ (Ghi liên tục)   │  │ (Phát hiện bất   │  │
│  │                 │  │                  │  │  thường)          │  │
│  │ Management Evts │  │ Single-region    │  │                   │  │
│  │ Read-only view  │  │ Multi-region     │  │ Write rate spike  │  │
│  │ No config needed│  │ Organization     │  │ Error rate spike  │  │
│  └─────────────────┘  └────────┬─────────┘  └───────────────────┘  │
│                                │                                     │
└────────────────────────────────┼─────────────────────────────────────┘
                                 │
               ┌─────────────────┴──────────────────┐
               │                                    │
               ▼                                    ▼
┌──────────────────────┐              ┌─────────────────────────────┐
│     S3 BUCKET        │              │   CLOUDWATCH LOGS           │
│ (Lưu trữ dài hạn)   │              │  (Phân tích real-time)      │
│                      │              │                             │
│ Lưu JSON event logs  │              │ Metric Filters → Alarms     │
│ Encryption KMS       │              │ Logs Insights queries       │
│ Lifecycle policies   │              │                             │
└──────────────────────┘              └─────────────────────────────┘
         │                                          │
         ▼                                          ▼
┌──────────────────────┐              ┌─────────────────────────────┐
│  ATHENA / OPENSEARCH │              │   EVENTBRIDGE → LAMBDA      │
│ (Phân tích nâng cao) │              │  (Tự động hóa response)     │
└──────────────────────┘              └─────────────────────────────┘
```

---

## Các Thành Phần Cốt Lõi

### 1. Event Types — Ba Loại Sự Kiện

CloudTrail ghi lại ba loại event (sự kiện):

| Loại Event              | Mô Tả                                        | Mặc Định | Phí Thêm |
| ----------------------- | -------------------------------------------- | -------- | -------- |
| **Management Events**   | Thao tác trên tài nguyên AWS (control plane) | ✅ Có    | Không    |
| **Data Events**         | Thao tác trên dữ liệu (S3, Lambda, DynamoDB) | ❌ Không | Có       |
| **Insights Events**     | Phát hiện hoạt động API bất thường           | ❌ Không | Có       |

📄 **Chi tiết:** [1-event-types.md](./1-event-types.md)

---

### 2. Trails — Cấu Hình Ghi Liên Tục

**Trail** (Dấu Vết) là cấu hình để CloudTrail ghi event liên tục vào S3 và/hoặc CloudWatch Logs.

**Hai loại Trail:**
- **Single-region Trail** — Chỉ ghi event trong một region
- **Multi-region Trail** — Ghi event từ tất cả region vào một S3 bucket trung tâm (khuyên dùng)

**Best Practice:**
- Bật ít nhất một multi-region Trail trong mọi AWS account
- Bật log file validation (xác thực tính toàn vẹn) để phát hiện file bị sửa đổi
- Mã hóa logs bằng KMS CMK (Customer Managed Key — Khóa Do Khách Hàng Quản Lý)

📄 **Chi tiết:** [2-trails-configuration.md](./2-trails-configuration.md)

---

### 3. Organization Trail — Ghi Log Đa Tài Khoản

**Organization Trail** cho phép một Trail trong management account ghi lại event từ **tất cả member accounts** trong AWS Organization — không cần cấu hình từng account riêng.

**Lợi ích:**
- Tập trung log vào một S3 bucket duy nhất
- Member accounts không thể tắt Organization Trail
- Bắt buộc trong môi trường enterprise multi-account

📄 **Chi tiết:** [3-organization-trail.md](./3-organization-trail.md)

---

### 4. CloudTrail Insights — Phát Hiện Bất Thường

**CloudTrail Insights** dùng machine learning (học máy) phân tích API write rate (tốc độ ghi API) thông thường và cảnh báo khi phát hiện hoạt động bất thường:

- Đột ngột xóa nhiều resource (bulk delete)
- Spike trong IAM API calls (có thể là tấn công privilege escalation)
- Tăng đột biến EC2 API calls (cryptomining, data exfiltration)

📄 **Chi tiết:** [4-cloudtrail-insights.md](./4-cloudtrail-insights.md)

---

### 5. Forensics & Investigation — Điều Tra Sự Cố

CloudTrail là công cụ số một cho digital forensics (pháp chứng kỹ thuật số) trên AWS:

- Tái hiện chuỗi sự kiện dẫn đến sự cố
- Tìm "blast radius" (phạm vi tác động) của credential bị lộ
- Chứng minh tuân thủ (PCI-DSS, HIPAA, SOC2) qua audit trail không thể phủ nhận

📄 **Chi tiết:** [5-forensics-investigation.md](./5-forensics-investigation.md)

---

## CloudTrail vs CloudWatch vs AWS Config

Ba dịch vụ hay bị nhầm lẫn nhất trong AWS Management & Governance:

| Tiêu Chí              | CloudTrail                        | CloudWatch Logs                   | AWS Config                         |
| --------------------- | --------------------------------- | --------------------------------- | ---------------------------------- |
| **Câu hỏi trả lời**  | "Ai đã làm gì?"                   | "Ứng dụng đang hoạt động thế nào?" | "Cấu hình hiện tại có đúng không?" |
| **Dữ liệu ghi lại**  | API calls từ control plane        | Log text từ ứng dụng/OS           | Trạng thái cấu hình tài nguyên     |
| **Thời gian**         | Hành động tại thời điểm cụ thể   | Log real-time từ ứng dụng         | Snapshot cấu hình theo thời gian   |
| **Lưu trữ**           | S3 (JSON event logs)              | CloudWatch Log Groups             | S3 (configuration snapshots)       |
| **Use case chính**    | Audit, forensics, compliance      | Debugging, monitoring             | Compliance, drift detection        |
| **Tự động bật?**      | Event History: có. Trail: không   | Không (cần cấu hình)              | Không (cần bật recorder)           |

### Khi Nào Dùng Cái Nào?

```
"Ai xóa S3 bucket?"                → CloudTrail
"Tại sao Lambda lỗi?"              → CloudWatch Logs
"S3 bucket có bật encryption không?"→ AWS Config
"Khi nào Security Group bị thay đổi?"→ CloudTrail (who/when) + Config (what changed)
```

---

## Pricing Overview

| Tính Năng                            | Free Tier / Mặc Định                       | Giá Thêm                                    |
| ------------------------------------ | ------------------------------------------ | ------------------------------------------- |
| **Event History**                    | Miễn phí, 90 ngày, read-only              | -                                           |
| **Management Events (Trail)**        | 1 Trail miễn phí / region                 | Trail thứ 2 trở đi: $2.00/100k events      |
| **Data Events**                      | Không bao gồm trong free tier              | $0.10/100k events                           |
| **Insights Events**                  | Không bao gồm                              | $0.35/100k write management events analyzed |
| **S3 Storage**                       | Tính theo S3 pricing                       | ~$0.023/GB/tháng (Standard)                 |
| **CloudWatch Logs delivery**         | Tính theo CW pricing                       | $0.50/GB ingestion                          |

> **Tip tiết kiệm:** Mặc định một Trail per region miễn phí cho Management Events. Với Data Events (đặc biệt S3 GetObject ở production), chi phí có thể rất cao — chỉ bật cho bucket quan trọng.

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: CloudTrail có tự động bật không?

**Câu trả lời chuẩn:** Event History (Lịch Sử Sự Kiện) tự động bật và lưu 90 ngày Management Events gần nhất — không cần cấu hình, miễn phí. Tuy nhiên, **Trail** (để lưu lâu dài vào S3) phải tự cấu hình. Trong thực tế, mọi account production nên có ít nhất một multi-region Trail.

---

### Q2: Làm thế nào biết ai đã xóa S3 bucket?

**Bước 1:** Vào CloudTrail → Event History, filter theo:
- Event name: `DeleteBucket`
- Resource type: S3

**Bước 2:** Xem event detail để tìm:
- `userIdentity.arn` — Ai thực hiện
- `sourceIPAddress` — Từ đâu
- `eventTime` — Lúc nào
- `requestParameters.bucketName` — Bucket nào

**Bước 3 (nếu cần tìm sâu hơn):** Query CloudTrail logs trong S3 bằng Amazon Athena.

---

### Q3: Data Events và Management Events khác nhau thế nào?

**Management Events** (Sự Kiện Quản Lý) — Thao tác trên **infrastructure** (cơ sở hạ tầng): tạo, sửa, xóa tài nguyên. Ví dụ: `CreateBucket`, `RunInstances`, `AttachRolePolicy`. Ghi mặc định, miễn phí.

**Data Events** (Sự Kiện Dữ Liệu) — Thao tác trên **dữ liệu trong tài nguyên**: đọc/ghi object S3, invoke Lambda, query DynamoDB. Không ghi mặc định, phải bật thêm và tính phí. Volume rất lớn (có thể hàng triệu events/ngày).

---

### Q4: Làm thế nào đảm bảo CloudTrail logs không bị giả mạo?

**Log File Validation** (Xác Thực Tệp Nhật Ký): CloudTrail tạo digest file (file tóm lược) chứa hash SHA-256 của mỗi log file. Để xác minh:

```bash
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:123456789012:trail/my-trail \
  --start-time 2026-05-01T00:00:00Z
```

**Bảo vệ thêm:** S3 bucket với MFA Delete + Object Lock, IAM policy ngăn xóa/sửa logs, S3 versioning.

---

### Q5: Organization Trail là gì và tại sao nên dùng?

**Organization Trail** được tạo trong management account và tự động ghi event từ **tất cả member accounts** — kể cả account được tạo sau. Lợi ích:

- Tập trung toàn bộ audit log vào một nơi
- Member accounts **không thể tắt** (ngăn xóa bằng cách điều chỉnh quyền trong member account)
- Tuân thủ yêu cầu audit của enterprise (PCI-DSS, SOC2, HIPAA)
- Tiết kiệm công cấu hình từng account riêng

---

### Q6: CloudTrail Insights hoạt động thế nào?

CloudTrail Insights phân tích **lịch sử 7 ngày** của API write rate để xây dựng baseline (mức nền thông thường). Khi tốc độ vượt ngưỡng bất thường, Insights tạo Insight Event và có thể:
- Ghi vào S3
- Gửi lên CloudWatch Logs → Alarm
- Trigger EventBridge rule → Lambda response tự động

---

## Điều Hướng Module

| File                                                          | Nội Dung                                              |
| ------------------------------------------------------------- | ----------------------------------------------------- |
| [1-event-types.md](./1-event-types.md)                        | Management Events, Data Events, Insights Events       |
| [2-trails-configuration.md](./2-trails-configuration.md)      | Trail setup, single/multi-region, S3 + CW Logs        |
| [3-organization-trail.md](./3-organization-trail.md)          | Centralized logging cho multi-account                 |
| [4-cloudtrail-insights.md](./4-cloudtrail-insights.md)        | Anomaly detection trong API activity                  |
| [5-forensics-investigation.md](./5-forensics-investigation.md)| Điều tra sự cố, phân tích event history               |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
