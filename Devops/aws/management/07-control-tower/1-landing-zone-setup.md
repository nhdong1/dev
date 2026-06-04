# Landing Zone — Thiết Lập Môi Trường Đa Tài Khoản Chuẩn

> **Landing Zone** (Vùng Hạ Cánh) là môi trường đa tài khoản AWS được cấu hình sẵn theo best practice bảo mật, tuân thủ và vận hành — bao gồm cấu trúc account, OU hierarchy (phân cấp đơn vị tổ chức), logging tập trung, và IAM Identity Center (SSO — Single Sign-On).

---

## 📚 Mục Lục

1. [Landing Zone là gì?](#landing-zone-là-gì)
2. [Các Account Bắt Buộc](#các-account-bắt-buộc)
3. [OU Structure Mặc Định](#ou-structure-mặc-định)
4. [Quá Trình Thiết Lập Landing Zone](#quá-trình-thiết-lập-landing-zone)
5. [Log Archive Account — Chi Tiết](#log-archive-account--chi-tiết)
6. [Audit Account — Chi Tiết](#audit-account--chi-tiết)
7. [IAM Identity Center Trong Landing Zone](#iam-identity-center-trong-landing-zone)
8. [Landing Zone Version & Update](#landing-zone-version--update)
9. [Troubleshooting Thường Gặp](#troubleshooting-thường-gặp)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Landing Zone là gì?

Landing Zone không phải một dịch vụ riêng mà là **kết quả** của quá trình Control Tower thiết lập — một tập hợp account, OU, guardrail, logging và SSO được cấu hình nhất quán.

### Những Gì Landing Zone Thiết Lập Tự Động

```
Landing Zone Setup (20–60 phút)
│
├── AWS Organizations
│   ├── Tạo Root Organization (nếu chưa có)
│   ├── Tạo Security OU
│   └── Tạo Sandbox OU
│
├── Accounts Đặc Biệt
│   ├── Log Archive Account (tạo mới hoặc chỉ định)
│   └── Audit Account (tạo mới hoặc chỉ định)
│
├── CloudTrail Organization Trail
│   └── Ghi log từ TẤT CẢ accounts → S3 trong Log Archive Account
│
├── AWS Config
│   └── Bật Configuration Recorder cho mọi account đã enroll
│
├── IAM Identity Center (SSO)
│   └── Kích hoạt và cấu hình directory
│
├── Mandatory Guardrails
│   └── Áp dụng cho toàn bộ landing zone ngay lập tức
│
└── Control Tower Dashboard
    └── Hiển thị compliance status toàn bộ accounts
```

---

## Các Account Bắt Buộc

Control Tower yêu cầu ít nhất **3 account** trong landing zone:

### 1. Management Account (Tài Khoản Quản Lý)

```
Đặc điểm:
- Account gốc kích hoạt Control Tower
- Là Organizations Management Account (Root)
- KHÔNG nên chạy workloads production ở đây
- Chứa Control Tower console, Account Factory, Dashboard

Tài nguyên Control Tower tạo trong Management Account:
- IAM roles cho Control Tower service
- CloudFormation StackSets để deploy vào member accounts
- SNS topics cho notifications
- S3 bucket cho CloudFormation artifacts
```

### 2. Log Archive Account (Tài Khoản Lưu Trữ Nhật Ký)

```
Đặc điểm:
- Dedicated account CHỈ để lưu trữ logs
- Mọi CloudTrail logs từ tất cả accounts → S3 ở đây
- AWS Config history và snapshots → S3 ở đây
- Không ai được xóa logs (Guardrail bắt buộc)
- Chỉ security/audit team có quyền đọc

S3 Buckets tự động tạo:
- aws-controltower-logs-{account-id}-{region}
- aws-controltower-s3-access-logs-{account-id}-{region}

Guardrails bảo vệ:
- Không cho phép xóa S3 bucket logging
- Không cho phép thay đổi S3 bucket policy
- Không cho phép tắt CloudTrail
```

### 3. Audit Account (Tài Khoản Kiểm Toán)

```
Đặc điểm:
- Dedicated account cho security và audit operations
- Cross-account read-only access vào TẤT CẢ member accounts
- Nơi chạy security tools: Security Hub, GuardDuty (nếu dùng)
- Security team sử dụng để điều tra sự cố

IAM Roles được tạo tự động (trong mỗi member account):
- AWSControlTowerExecution: Full admin (cho Control Tower automation)
- aws-controltower-ReadOnlyExecutionRole: Read-only cho Audit Account
- aws-controltower-AuditAdministratorRole: Admin từ Audit Account

Cross-account access flow:
Security Engineer
  → Đăng nhập Audit Account
  → AssumeRole vào member account (read-only)
  → Điều tra sự cố mà không cần credentials riêng
```

---

## OU Structure Mặc Định

Control Tower tạo sẵn 2 OU, nhưng bạn có thể thêm nhiều OU khác:

```
Root (Organization Root)
│
├── Security OU  ← Control Tower tạo và quản lý
│   ├── Log Archive Account
│   └── Audit Account
│   [Guardrails cứng nhất — không thể enroll accounts thường ở đây]
│
├── Sandbox OU  ← Control Tower tạo
│   ├── Dành cho dev/test, innovation
│   └── Guardrails nới lỏng hơn (cho phép thử nghiệm)
│
└── [OU bổ sung — tự tạo]  ← Bạn thêm theo nhu cầu
    ├── Infrastructure OU
    │   ├── Network Account (VPC hub, Transit Gateway)
    │   └── Shared Services Account (Active Directory, artifact repo)
    │
    ├── Workloads OU
    │   ├── Production Account (app-prod, db-prod...)
    │   └── Staging Account
    │
    ├── Policy Staging OU  [quan trọng khi test guardrails mới]
    │
    └── Suspended OU  [tạm dừng account mà chưa muốn xóa]
```

### Best Practice OU Design

```
Nguyên tắc thiết kế OU:
1. OU phản ánh CHÍNH SÁCH (policy), không phải tổ chức (org chart)
2. Accounts cùng OU có cùng bộ guardrails
3. Không nên quá nhiều OU (khó quản lý SCP phức tạp)
4. Tối đa 5 cấp OU lồng nhau

Mô hình phổ biến:
├── Security OU     → Audit, Log Archive
├── Infrastructure OU → Network, Shared Services
├── Workloads OU    → Production workloads
│   ├── Prod Sub-OU → Strict guardrails
│   └── SDLC Sub-OU → Dev/Staging/Test
└── Sandbox OU      → Experimentation, learning
```

---

## Quá Trình Thiết Lập Landing Zone

### Điều Kiện Tiên Quyết

```
Trước khi bật Control Tower:
□ Management Account chưa có Organizations hoặc Organizations mới tạo
□ Không có SCP nào đang xung đột với Control Tower
□ Có quyền AdministratorAccess trong Management Account
□ Region được chọn là Control Tower Home Region hỗ trợ
□ Đã xác nhận email cho Log Archive và Audit Account
  (hoặc có 2 email sẵn sàng để tạo account mới)
```

### Các Bước Thiết Lập

```
Bước 1: Vào Control Tower Console
  → AWS Console → Control Tower → Set up landing zone

Bước 2: Chọn Home Region
  - Home Region: Region chính cho Control Tower metadata
  - Không thể thay đổi sau khi thiết lập
  - Nên chọn Region gần workloads chính

Bước 3: Cấu hình Regions bổ sung
  - Chọn các AWS Regions muốn Control Tower quản lý
  - Config và CloudTrail sẽ được bật tại tất cả regions này

Bước 4: Cấu hình OU
  - Đặt tên Security OU (mặc định: "Security")
  - Đặt tên Sandbox OU (mặc định: "Sandbox")

Bước 5: Cấu hình Shared Accounts
  - Log Archive Account: Tạo mới hoặc dùng account hiện có
  - Audit Account: Tạo mới hoặc dùng account hiện có
  - Nhập email cho từng account (phải là email chưa dùng)

Bước 6: Cấu hình IAM Identity Center
  - Control Tower kích hoạt IAM Identity Center tự động
  - Chọn identity source (AWS IAM Identity Center / External IdP)

Bước 7: Review và Launch
  - Control Tower hiển thị tóm tắt những gì sẽ được tạo
  - Xác nhận → Quá trình tạo mất 20–60 phút
```

---

## Log Archive Account — Chi Tiết

### Kiến Trúc Logging Tập Trung

```
Management Account
  ├── CloudTrail Management Events → S3 (Log Archive)
  └── AWS Config → S3 (Log Archive)

Member Account A (Dev)
  ├── CloudTrail → Organization Trail → S3 (Log Archive)
  └── AWS Config → S3 (Log Archive)

Member Account B (Prod)
  ├── CloudTrail → Organization Trail → S3 (Log Archive)
  └── AWS Config → S3 (Log Archive)

             ↓ (tất cả tập trung vào)

Log Archive Account
  └── S3 Bucket: aws-controltower-logs-{account-id}-{region}
      ├── CloudTrail/
      │   ├── AWSLogs/{org-id}/{account-id}/CloudTrail/{region}/
      │   │   └── 2026/05/17/
      │   │       └── {account-id}_CloudTrail_{region}_*.json.gz
      ├── Config/
      │   └── AWSLogs/{account-id}/Config/{region}/
      │       └── ConfigHistory/ và ConfigSnapshot/
      └── [S3 Access Logs trong bucket riêng]
```

### S3 Bucket Policy trong Log Archive

Control Tower tự động tạo bucket policy với các nguyên tắc:
- Chỉ cho phép CloudTrail và Config service delivery
- Không ai được xóa objects (Object lock hoặc deny delete)
- Chỉ Audit Account có quyền đọc cross-account
- MFA Delete được khuyến nghị bật thêm

```json
// Ví dụ fragment của bucket policy do Control Tower tạo
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": ["s3:DeleteBucket", "s3:DeleteObject"],
  "Resource": [
    "arn:aws:s3:::aws-controltower-logs-*",
    "arn:aws:s3:::aws-controltower-logs-*/*"
  ],
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalArn": "arn:aws:iam::*:root"
    }
  }
}
```

### Retention (Lưu Giữ) Logs

```
Mặc định Control Tower không đặt lifecycle policy
→ Bạn cần tự đặt retention policy theo yêu cầu compliance:

PCI-DSS: 1 năm online + 3 năm archive
HIPAA:   6 năm
SOC 2:   Tối thiểu 1 năm

Khuyến nghị lifecycle:
- 0–90 ngày: S3 Standard (truy cập thường xuyên khi điều tra)
- 90–365 ngày: S3 Standard-IA (ít truy cập)
- 365+ ngày: S3 Glacier Instant Retrieval (lưu trữ dài hạn)
```

---

## Audit Account — Chi Tiết

### Cross-Account Roles Được Tạo Tự Động

Trong mỗi member account enrolled vào Control Tower:

```
aws-controltower-ReadOnlyExecutionRole
  - Trust: Audit Account
  - Policy: ReadOnlyAccess
  - Dùng cho: Security review, compliance check

aws-controltower-AuditAdministratorRole
  - Trust: Audit Account
  - Policy: AdministratorAccess
  - Dùng cho: Automated remediation từ Audit Account
  - ⚠️ Chỉ dùng cho automation, không phải người dùng

AWSControlTowerExecution
  - Trust: Management Account
  - Policy: AdministratorAccess
  - Dùng cho: Control Tower automation (tạo/update guardrails, baseline)
```

### Security Operations Từ Audit Account

```
Điển hình sử dụng Audit Account:

1. Security Investigation (Điều Tra Bảo Mật)
   Audit Account → AssumeRole ReadOnly vào member account
   → Xem CloudTrail events, IAM policies, Security Groups
   → Không cần tạo IAM user riêng trong mỗi account

2. Automated Security Scanning
   Chạy Security Hub Aggregator từ Audit Account
   → Thu thập findings từ tất cả member accounts

3. Centralized GuardDuty
   GuardDuty Delegated Admin = Audit Account
   → Manage detector từ một nơi cho toàn bộ org

4. Cross-Account Config Aggregator
   Config Aggregator trong Audit Account
   → Dashboard compliance toàn bộ accounts
```

---

## IAM Identity Center Trong Landing Zone

### Cấu Trúc SSO Sau Khi Landing Zone Thiết Lập

```
IAM Identity Center (trong Management Account)
  │
  ├── Identity Store (kho người dùng)
  │   ├── Users: alice@company.com, bob@company.com...
  │   └── Groups: Cloud-Admins, Developers, Security-Team...
  │
  ├── Permission Sets (bộ quyền)
  │   ├── AdministratorAccess (cho cloud admins)
  │   ├── PowerUserAccess (cho developers)
  │   ├── ReadOnlyAccess (cho auditors)
  │   └── SecurityAuditAccess (custom cho security team)
  │
  └── Account Assignments (phân quyền account)
      ├── Group "Cloud-Admins" + AdministratorAccess → [all accounts]
      ├── Group "Developers" + PowerUserAccess → [dev accounts]
      └── Group "Security-Team" + SecurityAuditAccess → [all accounts]
```

### Tích Hợp Với External Identity Provider

```
Control Tower hỗ trợ kết nối IAM Identity Center với:
- Azure Active Directory (Microsoft Entra ID)
- Okta
- Google Workspace
- Ping Identity
- Bất kỳ IdP nào hỗ trợ SAML 2.0

Flow khi dùng external IdP:
User → Azure AD login → SAML assertion
  → IAM Identity Center → Tạo session
  → Truy cập AWS Account được phân quyền
```

---

## Landing Zone Version & Update

### Versioning

Control Tower có phiên bản Landing Zone (vd: 3.0, 3.1, 3.2...) với các cải tiến:
- Bổ sung Mandatory Guardrails mới
- Hỗ trợ region mới
- Cải thiện Account Factory

### Cập Nhật Landing Zone

```
Control Tower Console → Landing Zone settings → Update

Quá trình update:
1. Control Tower kiểm tra pre-conditions
2. Apply thay đổi vào Management Account
3. Update guardrails trên tất cả enrolled OUs
4. Update baseline trong tất cả enrolled accounts
(có thể mất 1–2 giờ cho landing zone lớn)

⚠️ Lưu ý: Không update khi có account đang provision
```

---

## Troubleshooting Thường Gặp

### Lỗi "StackSet Failed" Khi Thiết Lập

```
Nguyên nhân: SCP hiện có chặn CloudFormation StackSets
Giải pháp: Kiểm tra SCPs trong Management Account, tạm thời gỡ
            SCPs xung đột, thiết lập xong rồi áp dụng lại
```

### Lỗi "Account Enrollment Failed"

```
Nguyên nhân phổ biến:
1. Account đã có Config Recorder không tương thích
2. SCP trong OU chặn Control Tower baseline
3. VPC limit reached trong account

Giải pháp:
1. Xóa Config Recorder cũ trước khi enroll
2. Kiểm tra và điều chỉnh SCP cho phép Control Tower operations
3. Request quota increase cho VPC
```

### Drift Sau Khi Thay Đổi Thủ Công

```
Nếu ai đó sửa trực tiếp tài nguyên do Control Tower tạo:
→ Control Tower Dashboard hiển thị "Drifted" status

Cách fix: Re-register OU hoặc Re-enroll Account
→ Control Tower sẽ restore lại trạng thái đúng
```

---

## Câu Hỏi Phỏng Vấn

### Q: Tại sao cần tách Log Archive và Audit thành 2 account riêng?

**Trả lời:** Nguyên tắc least privilege (đặc quyền tối thiểu) và separation of concerns (phân tách trách nhiệm):
- **Log Archive** chỉ được ghi vào bởi dịch vụ AWS (CloudTrail, Config). Ngay cả admin cũng không được xóa. Đây là immutable audit trail.
- **Audit Account** có quyền đọc cross-account nhưng không có quyền ghi vào Log Archive. Tách biệt để nếu Audit Account bị compromise, attacker không thể xóa evidence trong Log Archive.

---

### Q: Nếu tôi enroll account đã có tài nguyên vào Control Tower, điều gì xảy ra?

**Trả lời:** Control Tower sẽ:
1. Apply guardrails (SCPs) cho OU mà account đó thuộc về
2. Tạo IAM roles cần thiết (AWSControlTowerExecution, etc.)
3. Bật/cấu hình CloudTrail để gửi logs về Log Archive
4. Bật Config Recorder (nếu chưa có)
5. Tài nguyên hiện có của account vẫn giữ nguyên

Rủi ro: SCP mới có thể chặn các operation đang chạy. Luôn test trong Policy Staging OU trước.

---

### Q: Home Region trong Control Tower nghĩa là gì?

**Trả lời:** Home Region là region chính nơi Control Tower lưu metadata, dashboard, và thực hiện các operations. Một số tài nguyên Control Tower (như S3 bucket cho Log Archive, CloudTrail trail) được tạo trong Home Region. **Không thể thay đổi sau khi thiết lập**, vì vậy chọn region gần nhất với workloads chính hoặc theo yêu cầu data residency (lưu trú dữ liệu).

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành

← [README.md](README.md) | [2-guardrails.md](2-guardrails.md) →
