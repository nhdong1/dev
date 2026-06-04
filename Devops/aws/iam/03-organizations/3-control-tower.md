# AWS Control Tower — Landing Zone Tự Động

> **AWS Control Tower** là dịch vụ tự động thiết lập và quản trị môi trường multi-account an toàn theo đúng best practices AWS. Thay vì cấu hình thủ công Organizations, SCPs, CloudTrail, Config, IAM Identity Center — Control Tower thực hiện tất cả trong vài click.

---

## 📚 Mục Lục

1. [Control Tower Là Gì](#1-control-tower-là-gì)
2. [Landing Zone — Những Gì Được Tạo Tự Động](#2-landing-zone--những-gì-được-tạo-tự-động)
3. [Guardrails — Phòng Vệ Tự Động](#3-guardrails--phòng-vệ-tự-động)
4. [Account Factory — Tạo Tài Khoản Chuẩn](#4-account-factory--tạo-tài-khoản-chuẩn)
5. [Drift Detection và Remediation](#5-drift-detection-và-remediation)
6. [Control Tower vs Cấu Hình Thủ Công](#6-control-tower-vs-cấu-hình-thủ-công)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Control Tower Là Gì

### Vấn Đề Control Tower Giải Quyết

```
Thiết lập thủ công multi-account tốn kém:
  ├── Tạo Organizations + OU hierarchy
  ├── Gán SCPs vào từng OU
  ├── Tạo Log Archive account + S3 policy phức tạp
  ├── Tạo Audit/Security account
  ├── Bật CloudTrail organization-wide
  ├── Bật Config tập trung
  ├── Cài IAM Identity Center + permission sets
  ├── Cấu hình tài khoản mới mỗi khi tạo
  └── Kiểm tra liên tục xem có ai tắt guardrails không

Control Tower tự động hóa TẤT CẢ các bước trên.
```

### Control Tower — Kiến Trúc Tổng Quan

```
AWS Control Tower
├── Landing Zone (Vùng Đáp)
│   ├── Organizations + OU structure chuẩn
│   ├── Management Account (tài khoản tạo Control Tower)
│   ├── Log Archive Account (tạo tự động)
│   └── Audit Account (tạo tự động)
│
├── Guardrails (Rào Chắn)
│   ├── Mandatory Guardrails   — luôn bật, không thể tắt
│   ├── Strongly Recommended   — AWS khuyến nghị mạnh
│   └── Elective Guardrails    — tùy chọn theo nhu cầu
│
├── Account Factory (Nhà Máy Tài Khoản)
│   ├── Tạo tài khoản mới theo chuẩn
│   └── Tích hợp Service Catalog
│
└── Dashboard (Bảng Điều Khiển)
    ├── Compliance status toàn tổ chức
    ├── Non-compliant resources
    └── Guardrail violations
```

---

## 2. Landing Zone — Những Gì Được Tạo Tự Động

### Cấu Trúc OU Mặc Định

```
Root
├── Security OU (bảo mật — không thể xóa)
│   ├── Log Archive Account  — tất cả CloudTrail/Config logs
│   └── Audit Account        — Security Hub, Config Aggregator
│
└── Sandbox OU (tùy chọn)
    └── (Các tài khoản sandbox)
```

> **Lưu ý:** Control Tower tạo OU tên "Security" (không đổi tên được) và OU "Sandbox" ban đầu. Bạn có thể tạo thêm OUs sau.

### Tài Nguyên Được Tạo Tự Động

**Log Archive Account:**

```
├── S3 bucket: aws-controltower-logs-ACCOUNT_ID-REGION
│   ├── CloudTrail organization trail logs
│   └── AWS Config snapshots & history
│
├── Bucket policy: Chỉ CloudTrail/Config service mới ghi được
├── KMS key: Mã hóa logs tự động
└── CloudWatch Logs: Thêm tầng lưu trữ
```

**Audit Account:**

```
├── AWS Config Aggregator: Tổng hợp config từ mọi account
├── SNS Topics:
│   ├── aws-controltower-AggregateSecurityNotifications — mọi alerts
│   └── aws-controltower-AllConfigNotifications         — mọi config changes
├── IAM Roles cho cross-account access:
│   ├── AWSControlTowerExecution — Control Tower dùng để quản lý
│   └── aws-controltower-AuditRole — đọc resources trong member accounts
└── CloudWatch Events Rules
```

**Mọi Tài Khoản Thành Viên:**

```
├── IAM Roles:
│   ├── AWSControlTowerExecution (Control Tower management)
│   └── aws-controltower-ReadOnlyExecutionRole
│
├── AWS Config:
│   ├── Configuration recorder (bật ngay khi tạo account)
│   └── Delivery channel → Log Archive S3 bucket
│
└── CloudTrail:
    └── Tự động ghi vào organization trail của Log Archive
```

### Tài Khoản Được Cấu Hình SSO Ngay

Khi Control Tower setup xong, IAM Identity Center được cấu hình với:

```
Permission Sets (Bộ Quyền) mặc định:
├── AWSAdministratorAccess  — Admin của từng account
├── AWSReadOnlyAccess       — Đọc toàn bộ resources
└── AWSServiceCatalogAdminFullAccess — Dùng Account Factory

Groups mặc định:
├── AWSAccountFactory           — tạo account mới
├── AWSAuditAccountAdmins       — quản lý Audit account
├── AWSCloudTrailAdmins         — quản lý CloudTrail
├── AWSControlTowerAdmins       — toàn quyền Control Tower
├── AWSLogArchiveAdmins         — quản lý Log Archive
└── AWSSecurityAuditors         — đọc toàn tổ chức
```

---

## 3. Guardrails — Phòng Vệ Tự Động

### Phân Loại Guardrails

**Mandatory Guardrails (Bắt Buộc — Không Thể Tắt):**

| Guardrail | Loại | Mô Tả |
|---|---|---|
| Disallow changes to CloudTrail | SCP Deny | Ngăn tắt hoặc xóa CloudTrail trail |
| Disallow deletion of Log Archive | SCP Deny | Bảo vệ S3 logs |
| Enable AWS CloudTrail | Config Rule | Kiểm tra CloudTrail bật |
| Enable AWS Config | Config Rule | Kiểm tra Config đang ghi |
| Disallow public read access to log archive S3 | Config Rule | Bảo vệ tính riêng tư của logs |
| Disallow internet connection through RDP/SSH | Config Rule | Hạn chế port nguy hiểm |

**Strongly Recommended Guardrails (Khuyến Nghị Mạnh):**

```
├── Detect public read access to S3 buckets — DETECTIVE
├── Detect public write access to S3 buckets — DETECTIVE
├── Detect whether MFA for root account is enabled — DETECTIVE
├── Detect whether EC2 instances are launched in unapproved AMIs — DETECTIVE
├── Disallow internet access for Lambda in VPC — PREVENTIVE (SCP)
└── Disallow changes to AWS Config rules — PREVENTIVE (SCP)
```

**Elective Guardrails (Tùy Chọn):**

```
├── Disallow Amazon S3 access via non-SSL endpoints
├── Disallow Amazon EC2 instance types that are not approved
├── Detect whether S3 buckets have versioning enabled
└── Detect whether an RDS DB instance has multi-AZ enabled
...và nhiều guardrails khác
```

### Hai Loại Guardrail Theo Cơ Chế

```
Preventive Guardrails (Rào Chắn Phòng Ngừa):
  → Triển khai qua SCPs
  → Chặn hành động ngay lập tức
  → Ví dụ: Deny xóa CloudTrail

Detective Guardrails (Rào Chắn Phát Hiện):
  → Triển khai qua AWS Config Rules
  → Phát hiện vi phạm sau khi xảy ra
  → Báo cáo "NON_COMPLIANT" trên dashboard
  → Ví dụ: S3 bucket có public access → báo cáo vi phạm
```

### Áp Dụng Guardrail vào OU

```
Qua AWS Console:
1. Control Tower → Guardrails
2. Chọn guardrail muốn bật
3. "Enable on OU" → chọn OU
4. Confirm → Control Tower deploy SCP/Config Rule tự động

Guardrail áp dụng cho:
├── OU được chọn
└── Mọi accounts hiện tại và tương lai trong OU đó
```

---

## 4. Account Factory — Tạo Tài Khoản Chuẩn

### Account Factory Là Gì

**Account Factory** (Nhà Máy Tài Khoản) là tính năng của Control Tower cho phép tạo tài khoản AWS mới với tất cả cấu hình chuẩn (guardrails, SSO, CloudTrail, Config) được áp dụng tự động.

### Luồng Tạo Tài Khoản Qua Account Factory

```
1. Người dùng có quyền mở AWS Service Catalog
2. Chọn product "AWS Control Tower Account Factory"
3. Điền thông tin:
   ├── Account name
   ├── Email address (phải unique)
   ├── OU đích (nơi account sẽ được đặt)
   ├── SSO user email (ai sẽ là admin của account mới)
   └── Account configurations (VPC CIDR, etc.)
4. Service Catalog chạy CloudFormation
5. Control Tower tạo account, áp guardrails, cấu hình SSO
6. Admin mới nhận email + link đăng nhập SSO
```

### Customize Account Factory

**Account Factory Customization (AFC)** — tùy chỉnh tài nguyên được tạo trong mỗi account mới:

```yaml
# account-factory-customization.yaml
# CloudFormation template chạy trong account mới sau khi tạo

AWSTemplateFormatVersion: '2010-09-09'
Description: Baseline resources cho account mới

Resources:
  # Bật GuardDuty ngay khi tạo account
  GuardDutyDetector:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: true
      FindingPublishingFrequency: FIFTEEN_MINUTES

  # Tạo default VPC security group rules
  DefaultVPCSecurityGroupRestriction:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: vpc-default-security-group-closed
      Automatic: true
      TargetType: SSM_DOCUMENT
      TargetId: AWS-DisablePublicAccessForSecurityGroup

  # Tag mặc định cho account
  AccountTagPolicy:
    Type: AWS::Organizations::Policy
    Properties:
      Type: TAG_POLICY
      Content: !Sub |
        {
          "tags": {
            "AccountCreatedBy": {
              "tag_key": {"@@assign": "AccountCreatedBy"},
              "tag_value": {"@@assign": ["ControlTower"]}
            }
          }
        }
```

### Account Factory for Terraform (AFT)

AWS cung cấp **AFT** (Account Factory for Terraform — Nhà Máy Tài Khoản cho Terraform) — phiên bản GitOps-native của Account Factory:

```
AFT Architecture:
├── AFT Management Account (pipeline)
│   ├── CodePipeline — CI/CD pipeline tạo account
│   ├── CodeBuild    — thực thi Terraform
│   └── S3/DynamoDB  — Terraform state
│
└── GitHub/CodeCommit Repository
    ├── account-requests/          — YAML files mô tả account cần tạo
    ├── global-customizations/     — Terraform áp dụng cho mọi account
    └── account-customizations/    — Terraform áp dụng cho account cụ thể
```

**Tạo account bằng AFT:**

```yaml
# account-requests/product-a-prod.yaml
module "product_a_prod" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail              = "product-a-prod@company.com"
    AccountName               = "Product-A-Production"
    ManagedOrganizationalUnit = "Workloads/Production"
    SSOUserEmail              = "platform-team@company.com"
    SSOUserFirstName          = "Platform"
    SSOUserLastName           = "Team"
  }

  account_tags = {
    Environment = "production"
    Product     = "product-a"
    CostCenter  = "engineering"
  }

  account_customizations_name = "production-baseline"
}
```

---

## 5. Drift Detection và Remediation

### Drift (Sai Lệch) Là Gì

**Drift** xảy ra khi ai đó thay đổi tài nguyên bên ngoài Control Tower (ví dụ: xóa OU, sửa SCP thủ công, xóa IAM role của Control Tower).

```
Ví dụ drift:
├── Admin xóa một OU → Control Tower mất track
├── Developer sửa SCP trực tiếp → Guardrail không còn đúng
├── Ai đó xóa AWSControlTowerExecution role trong member account
└── Ai đó di chuyển account ra khỏi OU được Control Tower quản lý
```

### Phát Hiện và Xử Lý Drift

```
Control Tower tự động phát hiện drift:
├── Dashboard hiển thị: "DRIFTED" status
├── SNS notification được gửi
└── Affected resources được highlight

Xử lý drift:
1. Control Tower Console → "Repair" (sửa chữa tự động)
2. Control Tower chạy lại provisioning để khôi phục đúng trạng thái
3. Nếu không tự sửa được → phải can thiệp thủ công theo hướng dẫn
```

### Các Loại Drift Phổ Biến

| Loại Drift | Nguyên Nhân | Cách Xử Lý |
|---|---|---|
| OU bị xóa | Admin xóa OU có account | Tạo lại OU, di chuyển account về |
| SCP bị sửa | Thay đổi nội dung SCP thủ công | Repair từ Control Tower console |
| Account di chuyển | Move account ra ngoài managed OU | Repair hoặc re-enroll account |
| IAM Role bị xóa | Xóa AWSControlTowerExecution | Tạo lại role theo spec của Control Tower |
| Account không còn managed | Unmanaged account trong managed OU | Re-enroll account |

---

## 6. Control Tower vs Cấu Hình Thủ Công

### So Sánh Chi Tiết

| Tiêu Chí | Control Tower | Thủ Công |
|---|---|---|
| **Thời gian setup ban đầu** | ~1 giờ | 2–5 ngày |
| **Tính nhất quán** | Đảm bảo theo AWS best practices | Phụ thuộc người cấu hình |
| **Drift detection** | Tự động | Phải tự kiểm tra |
| **Account vending** | Tự động, chuẩn hóa | Script/manual |
| **Linh hoạt tùy chỉnh** | Có giới hạn | Hoàn toàn tự do |
| **Phù hợp cho** | ≥5 accounts, đội nhỏ | Yêu cầu đặc thù, IaC-first |
| **Cost** | Không phí thêm (dịch vụ underlying tính phí) | Như nhau |

### Khi Nào Dùng Control Tower

```
✅ Dùng Control Tower khi:
   - Mới bắt đầu multi-account journey
   - Đội nhỏ không có chuyên gia AWS security
   - Muốn AWS best practices ngay lập tức
   - 5–100+ accounts cần quản lý
   - Cần compliance baseline nhanh

❌ Cân nhắc cấu hình thủ công khi:
   - Đã có Organizations phức tạp trước đó
   - Yêu cầu tuân thủ đặc thù (ví dụ: GovCloud)
   - Cần kiểm soát 100% mọi thứ bằng Terraform/CDK
   - Architecture rất khác với Control Tower defaults
```

### Adopting Control Tower Trên Organization Có Sẵn

```
Thách thức khi migrate sang Control Tower:
1. Control Tower cần quản lý phần OUs nó tạo
2. Existing accounts phải được "enrolled" (đăng ký)
3. Existing SCPs có thể xung đột với Control Tower SCPs
4. Log Archive và Audit accounts phải mới (không dùng cái cũ)

Quy trình:
1. Tạo Control Tower với accounts mới
2. Enroll existing accounts từng cái một
3. Di chuyển accounts vào OUs được Control Tower quản lý
4. Kiểm tra drift và sửa sau mỗi bước
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Control Tower tạo bao nhiêu tài khoản khi setup?**

> Control Tower tạo 2 tài khoản mới: **Log Archive** (lưu CloudTrail và Config logs) và **Audit** (Security Hub aggregator, Config aggregator, cross-account readonly access). Cộng với Management Account có sẵn, tổng là 3 accounts ngay sau khi setup.

**Q: Guardrail Mandatory và Elective khác nhau như thế nào?**

> Mandatory guardrails luôn được bật và không thể tắt — chúng bảo vệ tính toàn vẹn của landing zone (ví dụ: không cho xóa CloudTrail). Elective guardrails có thể bật/tắt theo từng OU dựa trên nhu cầu cụ thể (ví dụ: chặn EC2 instance type không phê duyệt).

**Q: Drift trong Control Tower nghĩa là gì và cách xử lý?**

> Drift xảy ra khi ai đó thay đổi tài nguyên được Control Tower quản lý bên ngoài Control Tower (xóa OU, sửa SCP, xóa IAM role). Control Tower phát hiện qua periodic checks và báo trên dashboard. Xử lý bằng cách dùng tính năng "Repair" trong console, hoặc can thiệp thủ công theo hướng dẫn nếu tự động repair không được.

**Q: Account Factory for Terraform (AFT) khác Account Factory Console như thế nào?**

> Account Factory Console dùng Service Catalog — phù hợp cho người không quen Terraform, tạo account qua UI. AFT là GitOps-based — account được định nghĩa bằng Terraform/YAML trong git repo, tạo tự động qua CI/CD pipeline. AFT phù hợp hơn cho platform teams muốn IaC-first, version control toàn bộ account configuration.

---

## 🔗 Tiếp Theo

- **[4-account-vending.md](4-account-vending.md)** — Tự động hóa tạo account ở tầng sâu hơn với AFT và custom pipelines
- **[../02-identity-federation/1-iam-identity-center.md](../02-identity-federation/1-iam-identity-center.md)** — Cấu hình SSO được Control Tower thiết lập
- **[../07-monitoring-auditing/](../07-monitoring-auditing/README.md)** — Khai thác CloudTrail và Config logs mà Control Tower tổng hợp
