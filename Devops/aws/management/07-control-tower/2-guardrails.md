# Guardrails — Rào Chắn Quản Trị Trong Control Tower

> **Guardrails** (Rào Chắn Quản Trị) là các chính sách quản trị cấp cao, được áp dụng tự động cho các OU (Organizational Unit — Đơn Vị Tổ Chức) trong Control Tower Landing Zone. Chúng bảo vệ môi trường đa tài khoản khỏi những thay đổi vi phạm chính sách bảo mật và tuân thủ.

---

## 📚 Mục Lục

1. [Guardrail là gì và cách hoạt động](#guardrail-là-gì-và-cách-hoạt-động)
2. [Phân Loại Theo Hành Vi: Preventive, Detective, Proactive](#phân-loại-theo-hành-vi)
3. [Phân Loại Theo Mức Độ Bắt Buộc: Mandatory, Strongly Recommended, Elective](#phân-loại-theo-mức-độ-bắt-buộc)
4. [Danh Sách Guardrails Quan Trọng](#danh-sách-guardrails-quan-trọng)
5. [Cách Áp Dụng Guardrail Cho OU](#cách-áp-dụng-guardrail-cho-ou)
6. [Guardrail Inheritance (Kế Thừa)](#guardrail-inheritance)
7. [Compliance Dashboard](#compliance-dashboard)
8. [Guardrail vs SCP vs Config Rule](#guardrail-vs-scp-vs-config-rule)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Guardrail là gì và cách hoạt động

### Định Nghĩa

Guardrail là một **high-level rule** (quy tắc cấp cao) trong Control Tower, mô tả ý định quản trị bằng ngôn ngữ tự nhiên, nhưng thực thi thông qua cơ chế kỹ thuật:

```
Guardrail (khái niệm cấp cao)
    ↓ triển khai thông qua
┌──────────────────────────────────────────┐
│ Preventive → SCP (Service Control Policy)│
│ Detective  → AWS Config Rule             │
│ Proactive  → CloudFormation Hook         │
└──────────────────────────────────────────┘
```

### Ví Dụ Thực Tế

```
Guardrail: "Disallow Changes to CloudTrail Logging"
(Không cho phép thay đổi CloudTrail Logging)

→ Triển khai bằng SCP deny:
  Deny action: cloudtrail:StopLogging
  Deny action: cloudtrail:DeleteTrail
  Áp dụng: Tất cả member accounts trong OU

Kết quả: Ngay cả tài khoản admin trong member account
         cũng KHÔNG thể tắt CloudTrail
```

---

## Phân Loại Theo Hành Vi

### 1. Preventive Guardrails (Rào Chắn Ngăn Ngừa)

**Cơ chế:** SCP — Service Control Policy (Chính Sách Kiểm Soát Dịch Vụ)

**Nguyên lý:** Deny trực tiếp API call trước khi hành động xảy ra.

```
Flow:
User/Role trong member account
    ↓ thực hiện action (vd: cloudtrail:StopLogging)
AWS Organizations SCP evaluation
    ↓ SCP từ Control Tower Guardrail
DENY — Action bị chặn ngay lập tức
    ↓
User nhận lỗi: "Access denied by Service Control Policy"
```

**Ưu điểm:**
- Bắt ngay lập tức, không có "damage window"
- Không tốn phí (SCP miễn phí)
- Hiệu quả với vi phạm nghiêm trọng

**Hạn chế:**
- Chỉ bắt được hành động thông qua AWS API
- Không giúp phát hiện cấu hình sai đã tồn tại từ trước
- Nếu SCP quá strict có thể block hành động hợp lệ

---

### 2. Detective Guardrails (Rào Chắn Phát Hiện)

**Cơ chế:** AWS Config Rule (Quy Tắc AWS Config)

**Nguyên lý:** Kiểm tra cấu hình tài nguyên sau khi tạo/thay đổi, báo cáo vi phạm.

```
Flow:
Tài nguyên được tạo hoặc thay đổi
    ↓
AWS Config ghi lại configuration change
    ↓
Config Rule đánh giá (evaluate)
    ↓
Nếu COMPLIANT: Status = ✅ Compliant
Nếu NON-COMPLIANT: Status = ❌ Non-compliant
    ↓ (nếu vi phạm)
Control Tower Dashboard hiển thị violation
EventBridge event → SNS notification → Alert team
```

**Ưu điểm:**
- Phát hiện cấu hình sai đã tồn tại
- Cung cấp audit trail rõ ràng
- Có thể tích hợp tự động remediation

**Hạn chế:**
- Không ngăn chặn — chỉ phát hiện sau khi xảy ra
- Tốn phí Config Rule evaluations
- Có độ trễ (configuration change → evaluation → notification)

---

### 3. Proactive Guardrails (Rào Chắn Chủ Động)

**Cơ chế:** AWS CloudFormation Hooks (Móc CloudFormation)

**Nguyên lý:** Kiểm tra CloudFormation templates trước khi deploy — reject resources không tuân thủ.

```
Flow:
Developer viết CloudFormation template
    ↓
cfn deploy / cfn create-stack
    ↓
CloudFormation Hook evaluation (Pre-provision)
    ↓
Nếu PASS: CloudFormation tiếp tục deploy
Nếu FAIL: CloudFormation REJECT stack — không deploy
    ↓ (nếu vi phạm)
Developer nhận lỗi mô tả rule bị vi phạm
```

**Ưu điểm:**
- Shift-left (phát hiện sớm trong pipeline)
- Developer biết ngay lý do reject
- Ngăn ngừa cấu hình sai ngay từ đầu

**Hạn chế:**
- Chỉ bắt tài nguyên deploy qua CloudFormation
- Không áp dụng cho tài nguyên tạo tay qua console/CLI
- Yêu cầu Control Tower phiên bản mới hơn

---

## Phân Loại Theo Mức Độ Bắt Buộc

### Mandatory Guardrails (Rào Chắn Bắt Buộc)

**Đặc điểm:**
- Luôn bật — **KHÔNG thể tắt**
- Áp dụng cho toàn bộ landing zone ngay từ khi thiết lập
- Bảo vệ tính toàn vẹn của Control Tower

```
Ví dụ Mandatory Guardrails:
┌─────────────────────────────────────────────────────────────────┐
│ PREVENTIVE — Mandatory                                          │
├─────────────────────────────────────────────────────────────────┤
│ • Disallow changes to CloudTrail logging                        │
│   (Không cho thay đổi CloudTrail)                              │
│ • Disallow deletion of Log Archive S3 buckets                  │
│   (Không cho xóa S3 bucket Log Archive)                        │
│ • Disallow changes to encryption of Log Archive S3 buckets     │
│   (Không cho đổi mã hóa Log Archive)                          │
│ • Disallow changes to CloudTrail logging in the Log Archive     │
│ • Disallow configuration changes to CloudTrail                  │
│ • Disallow deletion of Amazon CloudWatch Logs                   │
│ • Disallow changes to AWS Config rules for CT                   │
│ • Disallow changes to AWS IAM roles for Control Tower           │
│ • Disallow changes to AWS Lambda functions for Control Tower    │
│ • Disallow changes to Amazon SNS subscriptions for CT           │
│ • Disallow changes to Amazon SNS topics for CT                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### Strongly Recommended Guardrails (Rào Chắn Khuyến Nghị Mạnh)

**Đặc điểm:**
- Mặc định **tắt** — phải bật thủ công
- AWS khuyến nghị bật cho hầu hết môi trường
- Phổ biến với môi trường production

```
Ví dụ Strongly Recommended Guardrails:
┌─────────────────────────────────────────────────────────────────┐
│ DETECTIVE — Strongly Recommended                                │
├─────────────────────────────────────────────────────────────────┤
│ • Detect whether MFA is enabled for root user                   │
│   (Phát hiện nếu root user không bật MFA)                      │
│ • Detect whether public access to RDS DB instance is enabled   │
│   (Phát hiện RDS instance có public access)                    │
│ • Detect whether public access to RDS DB snapshots is enabled  │
│ • Detect whether storage encryption is enabled for RDS         │
│   (Phát hiện RDS không bật encryption)                        │
│ • Detect whether unrestricted internet connection from EC2     │
│   (Phát hiện EC2 expose port ra internet không hạn chế)       │
│ • Detect whether Amazon EBS volumes are attached to EC2        │
│ • Detect whether an Amazon S3 bucket policy allows public write│
└─────────────────────────────────────────────────────────────────┘
```

---

### Elective Guardrails (Rào Chắn Tùy Chọn)

**Đặc điểm:**
- Mặc định **tắt** — bật theo nhu cầu cụ thể
- Phù hợp với một số industry hoặc compliance framework nhất định
- Có thể quá strict cho sandbox/dev environments

```
Ví dụ Elective Guardrails:
┌─────────────────────────────────────────────────────────────────┐
│ PREVENTIVE — Elective                                           │
├─────────────────────────────────────────────────────────────────┤
│ • Disallow creation of access keys for root user                │
│   (Không cho tạo access key cho root)                          │
│ • Disallow actions as a root user                               │
│   (Không cho root thực hiện bất kỳ hành động nào)              │
│ • Disallow Amazon S3 buckets that are not versioning-enabled   │
│ • Require use of multi-factor authentication for IAM users     │
│                                                                 │
│ DETECTIVE — Elective                                            │
├─────────────────────────────────────────────────────────────────┤
│ • Detect whether MFA is enabled for all IAM users              │
│ • Detect whether S3 bucket versioning is enabled               │
│ • Detect whether encryption is enabled for Amazon EBS volumes  │
│ • Detect whether VPC flow logs are enabled                     │
│ • Detect whether public IP is assigned to EC2 instance         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Danh Sách Guardrails Quan Trọng

### Bảng Tổng Hợp Guardrails Phổ Biến Nhất

| Guardrail                                              | Loại        | Mức Độ           | Cơ Chế      |
| ------------------------------------------------------ | ----------- | ---------------- | ----------- |
| Disallow changes to CloudTrail logging                 | Preventive  | Mandatory        | SCP         |
| Disallow deletion of Log Archive buckets               | Preventive  | Mandatory        | SCP         |
| Disallow changes to CT IAM roles                       | Preventive  | Mandatory        | SCP         |
| Detect MFA disabled for root user                      | Detective   | Strongly Rec.    | Config Rule |
| Detect public RDS snapshots                            | Detective   | Strongly Rec.    | Config Rule |
| Detect unrestricted SSH/RDP in Security Groups         | Detective   | Strongly Rec.    | Config Rule |
| Detect S3 bucket with public write access              | Detective   | Strongly Rec.    | Config Rule |
| Disallow actions as root user                          | Preventive  | Elective         | SCP         |
| Detect EBS volumes not encrypted                       | Detective   | Elective         | Config Rule |
| Detect VPC flow logs disabled                          | Detective   | Elective         | Config Rule |

---

## Cách Áp Dụng Guardrail Cho OU

### Bật Guardrail Qua Console

```
Control Tower Console
  → Guardrails
  → Chọn guardrail cần bật
  → "Enable on OU"
  → Chọn OU muốn áp dụng
  → Confirm

Control Tower sẽ:
1. Tạo/update SCP trong Management Account (nếu Preventive)
2. Deploy Config Rule vào tất cả accounts trong OU (nếu Detective)
3. Cập nhật Dashboard trạng thái
```

### Bật Guardrail Qua API

```bash
# Liệt kê guardrails hiện có
aws controltower list-enabled-controls \
  --target-identifier "arn:aws:organizations::123456789:ou/o-xxxx/ou-xxxx-xxxx"

# Bật guardrail cho OU
aws controltower enable-control \
  --control-identifier "arn:aws:controltower:us-east-1::control/AWS-GR_DETECT_CLOUDTRAIL_ENABLED" \
  --target-identifier "arn:aws:organizations::123456789:ou/o-xxxx/ou-xxxx-xxxx"

# Tắt guardrail khỏi OU (chỉ áp dụng cho Elective/Strongly Recommended)
aws controltower disable-control \
  --control-identifier "arn:aws:controltower:us-east-1::control/AWS-GR_S3_VERSIONING_ENABLED" \
  --target-identifier "arn:aws:organizations::123456789:ou/o-xxxx/ou-xxxx-xxxx"
```

### Kiểm Tra Compliance Qua API

```bash
# Xem compliance status của guardrail trên OU
aws controltower get-control-operation \
  --operation-identifier "arn:aws:controltower:us-east-1:123456789:operation/xxxx"

# Liệt kê tất cả accounts không tuân thủ
aws controltower list-enabled-controls \
  --target-identifier "arn:aws:organizations::123456789:ou/o-xxxx/ou-xxxx-xxxx" \
  --query 'enabledControls[?driftStatusSummary.driftStatus==`DRIFTED`]'
```

---

## Guardrail Inheritance (Kế Thừa)

### Nguyên Tắc Kế Thừa

```
Root (Organization Root)
│
├── Security OU [Mandatory Guardrails áp dụng ở đây]
│   ├── Log Archive Account ← Kế thừa tất cả guardrails từ Security OU
│   └── Audit Account ← Kế thừa tất cả guardrails từ Security OU
│
└── Workloads OU [Strongly Rec. + Elective bật thêm]
    ├── Prod Sub-OU [thêm Elective guardrails nghiêm ngặt hơn]
    │   ├── Prod Account A ← Kế thừa Workloads OU + Prod Sub-OU guardrails
    │   └── Prod Account B ← Kế thừa Workloads OU + Prod Sub-OU guardrails
    │
    └── Dev Sub-OU [ít guardrails hơn để linh hoạt]
        └── Dev Account ← Kế thừa Workloads OU + Dev Sub-OU guardrails
```

### Nguyên Tắc Quan Trọng

```
1. Guardrail bật ở OU cha → tự động áp dụng cho tất cả OU con và accounts
2. Account kế thừa guardrails từ tất cả OU trên đường từ Root đến account
3. Không thể tắt Mandatory Guardrail ở bất kỳ cấp nào
4. Elective guardrail bật ở OU cha → accounts trong OU con kế thừa
```

---

## Compliance Dashboard

### Control Tower Dashboard

```
Control Tower Console → Dashboard

Hiển thị:
┌────────────────────────────────────────────────────────────┐
│ Landing Zone Status                                         │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐│
│ │  OUs          │ │  Accounts    │ │  Guardrail Violations ││
│ │  Managed: 5  │ │  Enrolled: 23│ │  Critical: 2          ││
│ │  Drifted: 0  │ │  Suspended: 1│ │  High: 5              ││
│ └──────────────┘ └──────────────┘ └──────────────────────┘│
│                                                            │
│ Non-Compliant Resources                                    │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ Account        │ Guardrail              │ Resource     │ │
│ │ prod-account-a │ Detect MFA for root   │ root user    │ │
│ │ dev-account-b  │ Detect public RDS     │ db-instance-1│ │
│ └────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### Tích Hợp SNS Notification

```bash
# Control Tower gửi notification khi có guardrail violation
# Cấu hình trong: Control Tower → Settings → Notifications

# Event structure từ SNS:
{
  "version": "0",
  "source": "aws.controltower",
  "detail-type": "AWS Control Tower Compliance Status Change",
  "detail": {
    "eventName": "ComplianceStatusChanged",
    "controlIdentifier": "arn:aws:controltower:us-east-1::control/AWS-GR_...",
    "accountId": "123456789",
    "organizationalUnitId": "ou-xxxx-xxxx",
    "controlComplianceStatus": "NON_COMPLIANT"
  }
}
```

---

## Guardrail vs SCP vs Config Rule

### Khi Nào Dùng Gì?

| Tiêu Chí                          | Control Tower Guardrail         | SCP Thuần Thủ Công             | Config Rule Thuần              |
| --------------------------------- | ------------------------------- | ------------------------------ | ------------------------------ |
| **Mức độ abstraction**            | Cao — policy intent             | Thấp — JSON policy thủ công    | Thấp — rule code               |
| **Dễ quản lý**                   | ✅ Dashboard, 1-click           | ❌ Phải viết JSON, track thủ công | ❌ Phải deploy từng account   |
| **Consistency**                   | ✅ Tự động áp dụng khi enroll  | ❌ Phải nhớ áp dụng thủ công   | ❌ Phải deploy lại              |
| **Flexibility**                   | ❌ Guardrail có sẵn, ít tùy biến| ✅ Viết bất kỳ policy nào      | ✅ Viết custom rule            |
| **Multi-account consistency**     | ✅ Tự động                      | ✅ Nhưng phải quản lý thủ công | ❌ Khó                         |
| **Audit trail**                   | ✅ Control Tower Dashboard      | ❌ Phải tự track               | ✅ Config Dashboard            |

**Kết Luận:**
- Dùng **Guardrail** cho các policies phổ biến đã có sẵn trong Control Tower
- Dùng **SCP thủ công** (qua CfCT) cho policies tùy chỉnh ngoài guardrail có sẵn
- Dùng **Custom Config Rule** (qua CfCT) cho compliance checks đặc thù của tổ chức

---

## Câu Hỏi Phỏng Vấn

### Q1: Preventive vs Detective Guardrails — khác nhau gì và khi nào dùng?

**Trả lời:**
- **Preventive:** Dùng SCP để chặn trực tiếp API call — áp dụng cho vi phạm **tuyệt đối không được phép** (ví dụ: không tắt CloudTrail, không leave Organization). Lợi thế là ngăn chặn ngay, không có khoảng thời gian vi phạm.
- **Detective:** Dùng Config Rule để phát hiện sau khi cấu hình sai đã tồn tại — áp dụng cho **vi phạm cần phát hiện và báo cáo** (ví dụ: S3 bucket public, RDS không encrypt). Lợi thế là phát hiện cả cấu hình cũ sai.
- Trong thực tế: dùng kết hợp — Preventive cho vi phạm nghiêm trọng, Detective cho compliance monitoring.

---

### Q2: Mandatory Guardrail có thể bị bypass không?

**Trả lời:** Không thể bypass theo cách thông thường. Mandatory Guardrails dùng SCP áp dụng ở cấp Organization, ngay cả Administrator trong member account cũng không thể override. Chỉ có thể:
1. Management Account Admin mới có thể gỡ SCP — nhưng đây là hành động bị theo dõi và vi phạm Control Tower contract.
2. Control Tower có drift detection — nếu ai sửa SCP do CT tạo, dashboard sẽ hiển thị "Drifted" ngay.

---

### Q3: Tại sao Control Tower có Proactive Guardrail ngoài Preventive và Detective?

**Trả lời:** Proactive Guardrail (qua CloudFormation Hooks) bổ sung cho gap mà hai loại kia bỏ qua:
- **Preventive SCP:** Chặn API calls — nhưng một số misconfiguration chỉ xuất hiện khi xem xét toàn bộ resource configuration, không chỉ từng API call.
- **Detective Config Rule:** Phát hiện sau khi deploy — vẫn có khoảng thời gian non-compliant resource tồn tại.
- **Proactive Hook:** Chặn **trước khi deploy** qua CloudFormation — developer nhận feedback ngay, không bao giờ có non-compliant resource trong môi trường.

---

### Q4: Nếu bật Guardrail mới sau khi đã có nhiều accounts, điều gì xảy ra?

**Trả lời:**
- **Preventive Guardrail:** SCP áp dụng ngay lập tức cho tất cả accounts trong OU — các API calls vi phạm sẽ bị chặn từ thời điểm đó.
- **Detective Guardrail:** Config Rule được deploy vào tất cả enrolled accounts trong OU — sẽ scan và báo cáo các tài nguyên non-compliant **hiện có** (kể cả tài nguyên cũ). Dashboard sẽ hiển thị danh sách vi phạm cần xử lý.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành

← [1-landing-zone-setup.md](1-landing-zone-setup.md) | [3-account-factory.md](3-account-factory.md) →
