# AWS Organizations Setup — Thiết Lập Tổ Chức và Phân Cấp OU

> Hướng dẫn thực tế từng bước để tạo AWS Organizations, thiết kế OU hierarchy (phân cấp đơn vị tổ chức), mời/tạo tài khoản thành viên, và cấu hình baseline bảo mật cho toàn tổ chức.

---

## 📚 Mục Lục

1. [Khái Niệm Nền Tảng](#1-khái-niệm-nền-tảng)
2. [Tạo Organization](#2-tạo-organization)
3. [Thiết Kế OU Hierarchy](#3-thiết-kế-ou-hierarchy)
4. [Quản Lý Tài Khoản Thành Viên](#4-quản-lý-tài-khoản-thành-viên)
5. [Cấu Hình Baseline Bảo Mật](#5-cấu-hình-baseline-bảo-mật)
6. [Câu Hỏi Phỏng Vấn](#6-câu-hỏi-phỏng-vấn)

---

## 1. Khái Niệm Nền Tảng

### Cấu Trúc Organizations

```
Root
│  (Không thể xóa, không thể đổi tên)
│
├── OU (Organizational Unit — Đơn Vị Tổ Chức)
│   ├── Có thể lồng nhau tối đa 5 cấp (kể cả Root)
│   ├── Chứa accounts và/hoặc OUs con
│   └── SCPs áp dụng cho OU tự động kế thừa xuống
│
└── Account (Tài Khoản)
    ├── Management Account — tạo tổ chức, quản lý toàn bộ
    └── Member Accounts   — workloads thực tế
```

### Giới Hạn Quan Trọng (Service Limits)

| Mục | Giới Hạn Mặc Định | Ghi Chú |
|---|---|---|
| OUs per organization | 1,000 | Có thể tăng |
| Accounts per organization | 10 (mặc định) | Yêu cầu tăng qua support |
| SCPs per organization | 1,000 | — |
| SCPs per account/OU | 5 | Áp dụng tất cả cấp cha cộng lại |
| Cấp lồng OU tối đa | 5 (kể cả Root) | Root → OU → OU → OU → OU |
| Kích thước SCP | 5,120 ký tự | Per policy document |

### Feature Sets (Bộ Tính Năng)

```
All Features (Toàn Tính Năng) — Khuyến nghị
├── SCPs đầy đủ chức năng
├── Tag policies (Chính Sách Thẻ)
├── Backup policies (Chính Sách Sao Lưu)
├── AI services opt-out policies
└── Delegated administration

Consolidated Billing Only (Chỉ Hóa Đơn Hợp Nhất) — Hạn chế
└── Chỉ gộp hóa đơn, không có SCPs hay policy types khác
```

> **Lưu ý:** Sau khi bật All Features, không thể quay về Consolidated Billing Only mà không giải tán organization.

---

## 2. Tạo Organization

### Bước 1: Tạo Organization từ Management Account

```bash
# Tạo organization với toàn bộ tính năng
aws organizations create-organization --feature-set ALL

# Kết quả trả về:
# {
#   "Organization": {
#     "Id": "o-exampleorgid11",
#     "Arn": "arn:aws:organizations::123456789012:organization/o-exampleorgid11",
#     "FeatureSet": "ALL",
#     "MasterAccountId": "123456789012",
#     "MasterAccountArn": "arn:aws:organizations::123456789012:account/o-exampleorgid11/123456789012",
#     "AvailablePolicyTypes": [
#       {"Type": "SERVICE_CONTROL_POLICY", "Status": "ENABLED"}
#     ]
#   }
# }
```

### Bước 2: Bật SCP Policy Type

```bash
# Lấy Root ID
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)

# Bật SCPs (thường đã bật khi dùng ALL feature set)
aws organizations enable-policy-type \
  --root-id $ROOT_ID \
  --policy-type SERVICE_CONTROL_POLICY

# Bật Tag Policies (nếu cần quản lý tags đồng nhất)
aws organizations enable-policy-type \
  --root-id $ROOT_ID \
  --policy-type TAG_POLICY
```

### Bước 3: Bật Service Integrations

```bash
# Bật CloudTrail tập trung
aws organizations enable-aws-service-access \
  --service-principal cloudtrail.amazonaws.com

# Bật AWS Config tập trung
aws organizations enable-aws-service-access \
  --service-principal config.amazonaws.com

# Bật GuardDuty tập trung
aws organizations enable-aws-service-access \
  --service-principal guardduty.amazonaws.com

# Bật Security Hub tập trung
aws organizations enable-aws-service-access \
  --service-principal securityhub.amazonaws.com

# Bật IAM Access Analyzer
aws organizations enable-aws-service-access \
  --service-principal access-analyzer.amazonaws.com

# Kiểm tra danh sách tích hợp đang bật
aws organizations list-aws-service-access-for-organization
```

---

## 3. Thiết Kế OU Hierarchy

### Nguyên Tắc Thiết Kế

```
Nguyên tắc 1: Nhóm theo yêu cầu chính sách, không phải theo cơ cấu tổ chức
  ❌ OU: Engineering-Team-A / OU: Engineering-Team-B (theo team)
  ✅ OU: Production / OU: Non-Production (theo môi trường — cùng SCP)

Nguyên tắc 2: Tái sử dụng policy qua OU, không nhân bản
  → Accounts cần cùng guardrails → đặt vào cùng OU

Nguyên tắc 3: Giữ phẳng (flat) khi có thể
  → Mỗi cấp thêm vào tăng độ phức tạp quản lý

Nguyên tắc 4: Tách biệt Management Account hoàn toàn
  → Không đặt workloads, không áp SCPs hạn chế vào Management Account
```

### Tạo OU Hierarchy (Phân Cấp Đơn Vị Tổ Chức)

```bash
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)

# Tạo Security OU
SECURITY_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Security" \
  --query 'OrganizationalUnit.Id' --output text)

# Tạo Infrastructure OU
INFRA_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Infrastructure" \
  --query 'OrganizationalUnit.Id' --output text)

# Tạo Workloads OU
WORKLOADS_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Workloads" \
  --query 'OrganizationalUnit.Id' --output text)

# Tạo Sandbox OU
SANDBOX_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Sandbox" \
  --query 'OrganizationalUnit.Id' --output text)

# Tạo Suspended OU (tài khoản cần đóng — đặt vào đây để bị SCP chặn hoàn toàn)
SUSPENDED_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Suspended" \
  --query 'OrganizationalUnit.Id' --output text)

# Tạo OU con trong Workloads
PROD_OU=$(aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU \
  --name "Production" \
  --query 'OrganizationalUnit.Id' --output text)

NONPROD_OU=$(aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU \
  --name "Non-Production" \
  --query 'OrganizationalUnit.Id' --output text)

echo "Security OU: $SECURITY_OU"
echo "Infrastructure OU: $INFRA_OU"
echo "Workloads OU: $WORKLOADS_OU"
echo "Production OU: $PROD_OU"
echo "Non-Production OU: $NONPROD_OU"
echo "Sandbox OU: $SANDBOX_OU"
echo "Suspended OU: $SUSPENDED_OU"
```

### Xem Cấu Trúc Hiện Tại

```bash
# Liệt kê tất cả OUs dưới Root
aws organizations list-organizational-units-for-parent \
  --parent-id $ROOT_ID \
  --query 'OrganizationalUnits[*].[Name,Id]' \
  --output table

# Liệt kê accounts trong một OU
aws organizations list-accounts-for-parent \
  --parent-id $SECURITY_OU \
  --query 'Accounts[*].[Name,Id,Status]' \
  --output table
```

---

## 4. Quản Lý Tài Khoản Thành Viên

### Tạo Tài Khoản Mới Trong Organization

```bash
# Tạo tài khoản mới — AWS tự động tạo và thêm vào Organization
aws organizations create-account \
  --email "log-archive@company.com" \
  --account-name "Log-Archive" \
  --iam-user-access-to-billing DENY  # Root account không xem billing

# Kiểm tra trạng thái tạo account (bất đồng bộ)
CREATE_REQUEST_ID="car-exampleid"
aws organizations describe-create-account-status \
  --create-account-request-id $CREATE_REQUEST_ID
```

### Mời Tài Khoản Hiện Có

```bash
# Gửi lời mời tới tài khoản có sẵn
aws organizations invite-account-to-organization \
  --target '{"Type": "ACCOUNT", "Id": "987654321098"}' \
  --notes "Mời account vào Organization để quản lý tập trung"

# Tài khoản được mời cần chấp nhận từ account đó:
aws organizations accept-handshake --handshake-id "h-exampleid"
```

### Di Chuyển Account Giữa Các OU

```bash
# Di chuyển account vào Security OU
ACCOUNT_ID="111111111111"
CURRENT_PARENT=$(aws organizations list-parents \
  --child-id $ACCOUNT_ID \
  --query 'Parents[0].Id' --output text)

aws organizations move-account \
  --account-id $ACCOUNT_ID \
  --source-parent-id $CURRENT_PARENT \
  --destination-parent-id $SECURITY_OU
```

### Xóa Account Khỏi Organization

```bash
# Xóa account thành viên — account đó sẽ trở thành standalone
aws organizations remove-account-from-organization \
  --account-id "111111111111"
```

> **Lưu ý:** Trước khi xóa, account thành viên cần có payment method riêng nếu muốn tiếp tục dùng AWS.

---

## 5. Cấu Hình Baseline Bảo Mật

### OrganizationAccountAccessRole — Role Quản Trị Xuyên Tài Khoản

Khi tạo account mới qua Organizations, AWS tự động tạo role **OrganizationAccountAccessRole** trong account đó với full admin cho Management Account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::MANAGEMENT_ACCOUNT_ID:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
# Từ Management Account, assume role vào member account
aws sts assume-role \
  --role-arn "arn:aws:iam::MEMBER_ACCOUNT_ID:role/OrganizationAccountAccessRole" \
  --role-session-name "initial-setup"
```

### CloudTrail Organization Trail (Nhật Ký Kiểm Toán Toàn Tổ Chức)

```bash
# Tạo S3 bucket trong Log Archive account trước, sau đó:
aws cloudtrail create-trail \
  --name "org-audit-trail" \
  --s3-bucket-name "org-cloudtrail-logs-ACCOUNT_ID" \
  --is-multi-region-trail \
  --is-organization-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name "org-audit-trail"
```

**Bucket policy cho Log Archive account:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs-ACCOUNT_ID",
      "Condition": {
        "StringEquals": {
          "aws:SourceArn": "arn:aws:cloudtrail:REGION:MANAGEMENT_ACCOUNT_ID:trail/org-audit-trail"
        }
      }
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs-ACCOUNT_ID/AWSLogs/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "aws:SourceArn": "arn:aws:cloudtrail:REGION:MANAGEMENT_ACCOUNT_ID:trail/org-audit-trail"
        }
      }
    }
  ]
}
```

### Delegated Administrator (Quản Trị Viên Ủy Quyền)

```bash
# Ủy quyền Security Tooling account làm admin cho GuardDuty
aws organizations register-delegated-administrator \
  --account-id "SECURITY_TOOLING_ACCOUNT_ID" \
  --service-principal "guardduty.amazonaws.com"

# Ủy quyền Security Hub
aws organizations register-delegated-administrator \
  --account-id "SECURITY_TOOLING_ACCOUNT_ID" \
  --service-principal "securityhub.amazonaws.com"

# Kiểm tra delegated admins
aws organizations list-delegated-administrators \
  --query 'DelegatedAdministrators[*].[Name,Id]' \
  --output table
```

### Tag Policy (Chính Sách Thẻ) — Chuẩn Hóa Tags Toàn Tổ Chức

```json
{
  "tags": {
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": ["production", "staging", "development", "sandbox"]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "rds:db", "s3:bucket"]
      }
    },
    "CostCenter": {
      "tag_key": {
        "@@assign": "CostCenter"
      },
      "tag_value": {
        "@@assign": ["engineering", "data-platform", "security", "infrastructure"]
      }
    }
  }
}
```

```bash
# Áp dụng tag policy vào OU
aws organizations create-policy \
  --content file://tag-policy.json \
  --name "OrgTagStandards" \
  --type TAG_POLICY

aws organizations attach-policy \
  --policy-id "p-exampletagpolicyid" \
  --target-id $WORKLOADS_OU
```

---

## 6. Câu Hỏi Phỏng Vấn

**Q: Management Account có bị giới hạn bởi SCPs không?**

> Không. SCPs không áp dụng cho Management Account — đây là lý do chính tại sao Management Account không nên chạy workloads. Nếu credentials của Management Account bị xâm phạm, attacker có toàn quyền trên mọi tài khoản trong tổ chức.

**Q: Nếu muốn một account không thể tắt CloudTrail, làm thế nào?**

> Tạo SCP Deny `cloudtrail:StopLogging` và `cloudtrail:DeleteTrail`, áp dụng vào Root hoặc OU chứa account đó. Kể cả account root của member account cũng không thể tắt CloudTrail khi SCP này áp dụng.

**Q: Có thể áp SCP lên Management Account không?**

> Không. SCPs không bao giờ áp dụng cho Management Account. Đây là thiết kế có chủ đích của AWS — Management Account luôn có full control để có thể khôi phục tổ chức nếu SCP bị cấu hình sai.

**Q: OrganizationAccountAccessRole có nên xóa sau khi setup không?**

> Không nên xóa vội. Role này cần để Management Account có thể truy cập emergency nếu cần. Thay vào đó, hãy giám sát kỹ việc sử dụng role này qua CloudTrail, và có thể giới hạn principal được phép assume role từ Management Account bằng cách cập nhật trust policy.

**Q: Một account có thể thuộc nhiều OU không?**

> Không. Mỗi account chỉ có thể thuộc một OU (hoặc trực tiếp dưới Root). Tuy nhiên, account kế thừa SCPs từ toàn bộ chuỗi cha (chain of parent OUs) lên đến Root.

---

## 🔗 Tiếp Theo

- **[2-service-control-policies.md](2-service-control-policies.md)** — Viết và áp dụng SCPs cho các OU vừa tạo
- **[3-control-tower.md](3-control-tower.md)** — Tự động hóa toàn bộ quá trình trên
- **[../02-identity-federation/1-iam-identity-center.md](../02-identity-federation/1-iam-identity-center.md)** — Cấu hình SSO đa tài khoản
