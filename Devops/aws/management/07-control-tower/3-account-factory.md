# Account Factory — Tạo Tài Khoản AWS Tự Động Theo Chuẩn

> **Account Factory** (Xưởng Tài Khoản) là tính năng của Control Tower cho phép tạo tài khoản AWS mới theo template chuẩn một cách tự động và nhất quán. Mỗi account mới được tạo ra đã có sẵn guardrails, logging, SSO, và baseline configuration — mà không cần cấu hình thủ công.

---

## 📚 Mục Lục

1. [Account Factory là gì?](#account-factory-là-gì)
2. [Cách Account Factory Hoạt Động](#cách-account-factory-hoạt-động)
3. [Tạo Account Qua Service Catalog](#tạo-account-qua-service-catalog)
4. [Account Factory for Terraform — AFT](#account-factory-for-terraform--aft)
5. [Account Customization Sau Khi Tạo](#account-customization-sau-khi-tạo)
6. [Quản Lý Vòng Đời Account](#quản-lý-vòng-đời-account)
7. [Account Enroll vs Account Factory](#account-enroll-vs-account-factory)
8. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Account Factory là gì?

### Vấn Đề Khi Tạo Account Thủ Công

```
Trước Account Factory (thủ công):
1. Tạo account mới trong AWS Organizations → 5 phút
2. Đăng nhập account → cấu hình root MFA → 5 phút
3. Tạo IAM roles cần thiết → 15 phút
4. Bật CloudTrail và gửi logs về S3 tập trung → 30 phút
5. Bật Config Recorder → cấu hình delivery channel → 20 phút
6. Tạo VPC mặc định theo chuẩn → 20 phút
7. Cấu hình IAM Identity Center cho account này → 15 phút
8. Apply SCPs đúng OU → 10 phút
9. Gắn vào OU phù hợp → 5 phút
10. Review và test → 30 phút

Tổng: 2–4 giờ PER ACCOUNT, dễ sai, không nhất quán
```

```
Sau Account Factory (tự động):
1. Điền form Account Factory: tên, email, OU, SSO user → 3 phút
2. Xác nhận → Chờ Control Tower thực thi → 20–30 phút

Tổng: 3 phút thao tác + 25 phút chờ
Nhất quán 100% — mọi account đều giống nhau
```

### Account Factory Làm Gì?

```
Account Factory tự động hóa:
┌─────────────────────────────────────────────────────────────┐
│ Khi tạo account mới qua Account Factory:                    │
│                                                             │
│ 1. Tạo AWS Account (Organizations CreateAccount API)        │
│ 2. Đặt account vào OU được chỉ định                        │
│ 3. Apply Guardrails của OU đó                              │
│ 4. Tạo IAM Roles cho Control Tower management              │
│ 5. Bật CloudTrail → log tới Log Archive Account            │
│ 6. Bật Config Recorder → gửi tới Log Archive Account       │
│ 7. Cấu hình IAM Identity Center (SSO) permission set       │
│ 8. Tạo VPC theo network baseline (nếu cấu hình)            │
│ 9. Xóa VPC mặc định (default VPC) — best practice bảo mật  │
│ 10. Gửi notification khi hoàn thành                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Cách Account Factory Hoạt Động

### Kiến Trúc Kỹ Thuật

```
User (Cloud Admin)
    ↓ Submit form
AWS Service Catalog
    ↓ Launch product "Account Vending Machine"
AWS CloudFormation (trong Management Account)
    ↓ CloudFormation Stack sử dụng Custom Resource
AWS Lambda (AWSControlTowerAccountFactory)
    ↓
AWS Organizations CreateAccount API
    ↓ Account tạo xong
Control Tower Automation (CloudFormation StackSets)
    ↓ Deploy baseline vào account mới:
    ├── AWS::CloudTrail::Trail
    ├── AWS::Config::ConfigurationRecorder
    ├── AWS::Config::DeliveryChannel
    ├── IAM Roles (AWSControlTowerExecution, etc.)
    └── VPC Configuration
    ↓
IAM Identity Center
    ↓ Tạo permission assignment
Account sẵn sàng sử dụng
```

### Thời Gian Thực Thi

| Giai Đoạn                              | Thời Gian    |
| -------------------------------------- | ------------ |
| Tạo AWS account                        | 2–5 phút     |
| Deploy CloudFormation StackSet baseline| 10–15 phút   |
| Cấu hình IAM Identity Center           | 2–3 phút     |
| Tổng                                   | ~20–30 phút  |

---

## Tạo Account Qua Service Catalog

### Bước 1: Truy Cập Account Factory

```
Cách 1: Control Tower Console
  → Control Tower → Account Factory → Create account

Cách 2: AWS Service Catalog
  → Service Catalog → Products → "AWS Control Tower Account Factory"
  → Launch product
```

### Bước 2: Điền Thông Tin Account

```
Thông tin bắt buộc:
┌─────────────────────────────────────────────────────┐
│ Account email:     prod-team-a@company.com          │
│                    (phải là email chưa dùng)        │
│                                                     │
│ Account name:      prod-team-a                      │
│                                                     │
│ SSO user email:    alice@company.com                │
│                    (user sẽ là admin của account)   │
│                                                     │
│ SSO user first name: Alice                          │
│ SSO user last name:  Smith                          │
│                                                     │
│ Managed OU:        Workloads/Production             │
│                    (OU đích trong Organizations)    │
│                                                     │
│ VPC configuration: (tùy chọn)                      │
│   - VPC CIDR: 10.1.0.0/16                          │
│   - Subnet CIDRs: ...                              │
└─────────────────────────────────────────────────────┘
```

### Bước 3: Review & Launch

```
Service Catalog hiển thị:
- Tóm tắt các thông tin đã nhập
- Estimated cost của provisioning
- Tags áp dụng cho account

→ Click "Launch product"
→ Theo dõi tiến trình trong Service Catalog → Provisioned products
→ Nhận email notification khi hoàn thành
```

### Sau Khi Tạo Xong

```
Alice nhận email:
"Your AWS account has been created. Sign in at:
https://[your-sso-domain].awsapps.com/start"

Alice đăng nhập IAM Identity Center portal
→ Thấy account "prod-team-a" với permission set được cấu hình
→ Truy cập account với quyền được phân
```

---

## Account Factory for Terraform — AFT

### AFT là gì?

**AFT — Account Factory for Terraform** (Xưởng Tài Khoản Cho Terraform) là giải pháp IaC — Infrastructure as Code (Hạ Tầng Dưới Dạng Mã) cho phép tạo và tùy chỉnh account Control Tower thông qua Terraform thay vì Service Catalog UI.

### Kiến Trúc AFT

```
Git Repository (Account Requests)
    ↓ git push / PR merge
AWS CodePipeline (AFT Pipeline)
    ↓
CodeBuild: Terraform Apply
    ↓
Control Tower Account Factory API
    ↓ Tạo account
Control Tower baseline automation
    ↓
Customization Pipeline
    ↓ Chạy account customizations
Account sẵn sàng
```

### Cấu Trúc Repository AFT

```
aft-account-request/           ← Request tạo account mới
├── terraform/
│   └── accounts.tf            ← Định nghĩa mỗi account
│
aft-global-customizations/     ← Customization áp dụng cho MỌI account
├── api_helpers/
│   └── python/                ← Python scripts chạy sau khi tạo
├── terraform/                 ← Terraform chạy trong mọi account mới
│
aft-account-customizations/    ← Customization theo từng account cụ thể
├── PROD-ACCOUNT-A/
│   ├── api_helpers/
│   └── terraform/
└── DEV-ACCOUNT-B/
    ├── api_helpers/
    └── terraform/
```

### Định Nghĩa Account Trong AFT

```hcl
# aft-account-request/terraform/accounts.tf

module "prod_team_a" {
  source = "./modules/aft-account-request"

  # Thông tin account
  control_tower_parameters = {
    AccountEmail              = "prod-team-a@company.com"
    AccountName               = "prod-team-a"
    ManagedOrganizationalUnit = "Workloads (ou-xxxx-xxxx)"
    SSOUserEmail              = "alice@company.com"
    SSOUserFirstName          = "Alice"
    SSOUserLastName           = "Smith"
  }

  # Metadata cho customization
  account_tags = {
    Environment = "production"
    Team        = "team-a"
    CostCenter  = "CC-1234"
  }

  # Custom fields cho customization pipeline
  custom_fields = {
    vpc_cidr       = "10.1.0.0/16"
    enable_guardduty = "true"
    backup_policy  = "daily"
  }

  # Chọn customization nào áp dụng
  account_customizations_name = "PROD-ACCOUNT-A"
}
```

### AFT Infrastructure Setup (Một Lần Duy Nhất)

```hcl
# Cài đặt AFT vào Management Account
module "aft" {
  source = "github.com/aws-ia/terraform-aws-control_tower_account_factory"

  ct_management_account_id    = "111111111111"  # Management Account
  log_archive_account_id      = "222222222222"  # Log Archive Account
  audit_account_id            = "333333333333"  # Audit Account
  aft_management_account_id   = "444444444444"  # Dedicated AFT Account

  ct_home_region              = "us-east-1"
  tf_backend_secondary_region = "us-west-2"

  vcs_provider                = "github"  # hoặc gitlab, bitbucket, codecommit
  account_request_repo_name   = "myorg/aft-account-request"
  global_customizations_repo_name    = "myorg/aft-global-customizations"
  account_customizations_repo_name   = "myorg/aft-account-customizations"
  account_provisioning_customizations_repo_name = "myorg/aft-account-provisioning-customizations"
}
```

### AFT vs Service Catalog UI — Khi Nào Dùng Gì?

| Tiêu Chí                           | Service Catalog UI (Account Factory) | AFT                          |
| ---------------------------------- | ------------------------------------ | ---------------------------- |
| **Độ phức tạp setup**              | Không cần setup thêm                 | Cần setup infrastructure     |
| **Yêu cầu kỹ năng**               | Không cần coding                     | Cần Terraform + Git          |
| **Tốc độ tạo account đơn lẻ**    | Nhanh hơn (UI form)                  | Phải commit code + wait CI   |
| **Tốc độ tạo 10+ accounts**       | Chậm (điền form từng cái)            | Nhanh (code định nghĩa)      |
| **Customization sau khi tạo**      | Hạn chế                              | Mạnh mẽ, tùy biến cao       |
| **Audit trail thay đổi**           | Service Catalog events               | Git history (đầy đủ hơn)    |
| **GitOps workflow**                | ❌                                   | ✅                            |
| **Phù hợp với**                   | Tổ chức nhỏ, ít account             | Tổ chức lớn, DevOps-mature   |

---

## Account Customization Sau Khi Tạo

### Vấn Đề: Account Factory Chỉ Tạo Baseline

Account Factory tạo baseline chuẩn, nhưng mỗi account thường cần thêm:
- VPC với CIDR cụ thể theo IP addressing plan
- Security Hub bật với specific standards
- GuardDuty bật và gửi findings về central account
- IAM Password Policy theo tiêu chuẩn công ty
- Default encryption settings
- Tagging defaults

### Giải Pháp 1: CfCT (Customizations for Control Tower)

Xem chi tiết tại [4-customizations-cfct.md](4-customizations-cfct.md)

### Giải Pháp 2: AFT Customizations Pipeline

```
AFT Customization Workflow:

1. Global Customizations (chạy với MỌI account)
   ├── Bật Security Hub với CIS AWS Foundations standard
   ├── Bật GuardDuty và gửi findings về Audit Account
   ├── Set IAM Password Policy theo chuẩn công ty
   └── Xóa default VPC

2. Account-specific Customizations
   ├── PROD-ACCOUNT: Tạo VPC với CIDR 10.1.0.0/16, enable flow logs
   ├── DEV-ACCOUNT: Tạo VPC với CIDR 10.2.0.0/16
   └── SANDBOX: Không tạo VPC (dùng default setup)
```

---

## Quản Lý Vòng Đời Account

### Update Account (Thay Đổi OU hoặc SSO)

```
Control Tower Console
  → Account Factory → Update account

Có thể thay đổi:
- OU đích (account sẽ được di chuyển sang OU mới)
- SSO permission set
- Email address (hạn chế)

KHÔNG thể thay đổi qua Account Factory:
- Account ID
- Account root email
```

### Suspend Account (Tạm Dừng)

```
Khi account không còn cần dùng nhưng chưa muốn xóa hoàn toàn:

1. Move account về "Suspended OU"
   → OU này thường có SCP deny tất cả actions
   → Account bị đóng băng — không ai dùng được

2. Account vẫn tồn tại trong Organizations
   → Vẫn tính phí tài nguyên còn lại
   → CloudTrail vẫn ghi log

Best practice: Xóa tài nguyên trước khi suspend
              để tránh phát sinh chi phí
```

### Close Account (Đóng Tài Khoản)

```
⚠️ KHÔNG THỂ KHÔI PHỤC sau khi close account!

Quy trình:
1. Backup tất cả dữ liệu cần giữ
2. Xóa tài nguyên để tránh phí tiếp theo
3. Remove account khỏi Control Tower
   → Control Tower → Account Factory → Unmanage account
4. Close account trong Organizations
   → Organizations → Close account

Lưu ý:
- Account sẽ bị suspend 90 ngày trước khi bị xóa hoàn toàn
- Trong 90 ngày có thể khôi phục
- Sau 90 ngày: xóa vĩnh viễn, không thể khôi phục
```

---

## Account Enroll vs Account Factory

### Sự Khác Biệt

| Tiêu Chí                          | Account Factory (Tạo Mới)          | Account Enroll (Đăng Ký Cũ)          |
| --------------------------------- | ---------------------------------- | ------------------------------------- |
| **Account nguồn gốc**             | Tạo mới hoàn toàn                  | Account đã tồn tại trong Organizations|
| **Baseline**                      | Áp dụng từ đầu                     | Áp dụng lên account đang chạy         |
| **Rủi ro**                        | Thấp — account trống               | Trung bình — có thể xung đột resource |
| **Kiểm tra trước**                | Không cần                          | Cần review resource hiện có           |
| **Dùng khi**                      | Account mới hoàn toàn              | Migrate account cũ vào CT             |

### Enroll Account Hiện Có Vào Control Tower

```
Điều kiện để enroll thành công:
□ Account đã là thành viên của Organizations
□ Không có Config Recorder xung đột
□ Region được chọn phải trong danh sách CT governed regions
□ Không có SCP chặn Control Tower operations
□ Account không phải Management Account

Quy trình:
Control Tower → Account Factory → Enroll account
  → Nhập Account ID hoặc email
  → Chọn OU đích
  → Control Tower apply baseline và guardrails
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Account Factory và Account Factory for Terraform (AFT) khác nhau gì?

**Trả lời:** Account Factory là tính năng có sẵn trong Control Tower, sử dụng Service Catalog UI để tạo account — phù hợp cho tổ chức nhỏ hoặc khi tạo account thỉnh thoảng. AFT là giải pháp bổ sung, dùng Terraform + Git + CodePipeline để tự động hóa việc tạo và tùy chỉnh account theo GitOps workflow — phù hợp cho tổ chức lớn cần tạo nhiều account, muốn audit trail đầy đủ qua git history, và cần customization phức tạp sau khi tạo account.

---

### Q2: Sau khi Account Factory tạo account xong, account đó có những gì mặc định?

**Trả lời:** Account mới từ Account Factory có sẵn:
1. **CloudTrail trail** ghi log tới S3 trong Log Archive Account
2. **AWS Config Recorder** ghi configuration changes tới Log Archive Account
3. **IAM Roles** cho Control Tower management (AWSControlTowerExecution, ReadOnly, AuditAdmin)
4. **VPC** theo cấu hình (hoặc default VPC bị xóa theo best practice)
5. **IAM Identity Center** permission set được assign cho SSO user chỉ định
6. **Guardrails** của OU đích đã được áp dụng (thông qua SCP + Config Rules)

---

### Q3: Tôi muốn thêm bước tùy chỉnh khi tạo account mới (ví dụ: bật GuardDuty, tạo VPC riêng). Làm thế nào?

**Trả lời:** Có hai cách:
1. **CfCT (Customizations for Control Tower):** Dùng CloudFormation template + SCP thêm, tự động chạy khi account được enroll vào OU. Phù hợp nếu đã dùng CloudFormation/CDK.
2. **AFT Customizations:** Dùng global customizations (áp dụng cho mọi account) và account-specific customizations (Terraform + Python scripts). Phù hợp nếu đã dùng Terraform. Đây là phương pháp được khuyến nghị cho tổ chức dùng Terraform.

---

### Q4: Tại sao Account Factory xóa default VPC?

**Trả lời:** Default VPC trong mỗi account mới có CIDR `172.31.0.0/16` giống nhau và đã có Internet Gateway, route table, subnet mở sẵn — đây là cấu hình không tuân thủ best practice bảo mật (expose resources ra internet mặc định). Account Factory xóa default VPC và tạo VPC mới theo cấu hình chuẩn của tổ chức (CIDR riêng, không có public subnet trừ khi cần, flow logs bật...).

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành

← [2-guardrails.md](2-guardrails.md) | [4-customizations-cfct.md](4-customizations-cfct.md) →
