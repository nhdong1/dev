# AWS Organizations — Cấu Trúc Account & OU Hierarchy

> **Account Structure** (Cấu Trúc Tài Khoản) trong AWS Organizations là nền tảng của mọi chiến lược đa tài khoản. Hiểu rõ Management Account, Member Account và cách tổ chức OU (Organizational Unit — Đơn Vị Tổ Chức) giúp thiết kế môi trường an toàn, dễ quản trị và tuân thủ chuẩn enterprise.

---

## 📚 Mục Lục

1. [Các Thành Phần Cơ Bản](#các-thành-phần-cơ-bản)
2. [Management Account (Tài Khoản Quản Lý)](#management-account)
3. [Member Account (Tài Khoản Thành Viên)](#member-account)
4. [Root — Gốc Tổ Chức](#root--gốc-tổ-chức)
5. [OU — Organizational Unit](#ou--organizational-unit)
6. [Hierarchy — Cây Phân Cấp](#hierarchy--cây-phân-cấp)
7. [Quản Lý Account — Tạo, Mời, Di Chuyển](#quản-lý-account)
8. [Feature Sets — Tính Năng Organizations](#feature-sets)
9. [Giới Hạn Và Quota](#giới-hạn-và-quota)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Các Thành Phần Cơ Bản

```
Organization
│
├── Root (1 duy nhất)
│   └── SCP gắn vào Root áp dụng cho toàn bộ Organization
│
├── Management Account (Tài Khoản Quản Lý)
│   ├── Luôn ở cấp Root, không nằm trong OU
│   ├── Quản lý toàn bộ Organizations
│   └── Không bị SCP giới hạn
│
└── OU (Organizational Unit — Đơn Vị Tổ Chức)
    ├── OU con (lồng nhau tối đa 5 cấp từ Root)
    └── Member Account (Tài Khoản Thành Viên)
        └── Leaf node — không chứa OU con
```

### Tóm Tắt Nhanh

| Thành Phần | Vai Trò | Số Lượng |
|-----------|---------|----------|
| **Root** | Gốc của toàn bộ cây | 1 duy nhất |
| **Management Account** | Điều hành Organization | 1 duy nhất |
| **OU** | Nhóm account theo logic | Tối đa 5 cấp sâu |
| **Member Account** | Chạy workload thực tế | Không giới hạn (soft limit 10.000) |

---

## Management Account

### Vai Trò Và Trách Nhiệm

**Management Account** (trước đây gọi là **Master Account**) là account đặc biệt có toàn quyền điều hành AWS Organizations:

```
Management Account có thể:
├── Tạo, xóa, mời Member Account
├── Tạo và quản lý OU
├── Tạo và gắn SCP, Tag Policy, AI Opt-out Policy
├── Xem billing hợp nhất của toàn bộ Organization
├── Enable/disable trusted access cho AWS services
└── Designate Delegated Administrator cho từng service
```

### Đặc Điểm Bảo Mật Quan Trọng

```
⚠️ SCP KHÔNG áp dụng cho Management Account
   → Root user và IAM user trong Management Account
     có thể làm mọi thứ, bất kể SCP

⚠️ Không thể chuyển Management Account sang account khác
   (chỉ có thể migrate Organization, rất phức tạp)

⚠️ Không thể rời Organization từ Management Account
   (phải xóa toàn bộ Organization)
```

### Best Practice Cho Management Account

```
✅ Bật MFA bắt buộc cho root user và mọi IAM user
✅ Không tạo IAM user trong Management Account (dùng IAM Identity Center)
✅ KHÔNG chạy workload ứng dụng trong Management Account
✅ Chỉ dùng để quản trị Organizations và billing
✅ Bật CloudTrail Organization Trail từ Management Account
✅ Thiết lập budget alert cho toàn Organization từ đây
✅ Số lượng người có quyền vào Management Account càng ít càng tốt
```

### Thực Hành: Tạo Organization

```bash
# Tạo Organization với ALL features (khuyến nghị)
aws organizations create-organization --feature-set ALL

# Xem thông tin Organization
aws organizations describe-organization

# Kết quả:
# {
#   "Organization": {
#     "Id": "o-xxxxxxxxxxxx",
#     "MasterAccountId": "123456789012",
#     "MasterAccountEmail": "management@company.com",
#     "FeatureSet": "ALL",
#     "AvailablePolicyTypes": [
#       {"Type": "SERVICE_CONTROL_POLICY", "Status": "ENABLED"},
#       {"Type": "TAG_POLICY", "Status": "ENABLED"}
#     ]
#   }
# }
```

---

## Member Account

### Tạo Member Account Mới

Có **3 cách** để thêm account vào Organization:

#### Cách 1: Tạo Account Mới Trực Tiếp (Account Vending)

```bash
# Tạo account mới từ Management Account
aws organizations create-account \
  --email app-production@company.com \
  --account-name "App Production" \
  --iam-user-access-to-billing ALLOW \
  --role-name OrganizationAccountAccessRole

# Kết quả trả về CreateAccountStatus với RequestId
# Kiểm tra trạng thái
aws organizations describe-create-account-status \
  --create-account-request-id car-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**Lưu ý:** Account mới tự động tạo IAM Role `OrganizationAccountAccessRole` với full admin permissions — cho phép Management Account assume role vào.

#### Cách 2: Mời Account Đã Tồn Tại (Join Bằng Invitation)

```bash
# Gửi lời mời từ Management Account
aws organizations invite-account-to-organization \
  --target '{"Type": "EMAIL", "Id": "existing-account@company.com"}'

# Account nhận được email — phải accept invitation
# Từ account được mời:
aws organizations accept-handshake \
  --handshake-id h-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

#### Cách 3: Account Factory (Control Tower)

Tự động hóa tạo account theo template chuẩn — xem `07-control-tower/3-account-factory.md`.

### Quyền Truy Cập Cross-Account (Truy Cập Chéo Account)

```bash
# Assume role từ Management Account vào Member Account
aws sts assume-role \
  --role-arn "arn:aws:iam::MEMBER_ACCOUNT_ID:role/OrganizationAccountAccessRole" \
  --role-session-name "AdminSession"

# Hoặc dùng AWS CLI profile
# ~/.aws/config
[profile member-account]
role_arn = arn:aws:iam::123456789012:role/OrganizationAccountAccessRole
source_profile = management
```

### Rời Organization (Leave Organization)

```bash
# Từ Member Account — tự rời
aws organizations leave-organization

# Từ Management Account — xóa account khỏi Organization
aws organizations remove-account-from-organization \
  --account-id 123456789012
```

**Lưu ý:** Account sau khi rời trở thành standalone account — phải có payment method riêng.

---

## Root — Gốc Tổ Chức

**Root** là container cấp cao nhất trong cây Organizations. Mỗi Organization chỉ có **1 Root duy nhất**.

```bash
# Lấy Root ID
aws organizations list-roots

# Kết quả:
# {
#   "Roots": [{
#     "Id": "r-xxxx",
#     "Arn": "arn:aws:organizations::123456789012:root/o-xxxx/r-xxxx",
#     "Name": "Root",
#     "PolicyTypes": [
#       {"Type": "SERVICE_CONTROL_POLICY", "Status": "ENABLED"}
#     ]
#   }]
# }
```

### Enable Policy Types Trên Root

```bash
# Enable SCP (cần trước khi tạo và gắn SCP)
aws organizations enable-policy-type \
  --root-id r-xxxx \
  --policy-type SERVICE_CONTROL_POLICY

# Enable Tag Policy
aws organizations enable-policy-type \
  --root-id r-xxxx \
  --policy-type TAG_POLICY
```

---

## OU — Organizational Unit

### Tạo Và Quản Lý OU

```bash
# Tạo OU dưới Root
aws organizations create-organizational-unit \
  --parent-id r-xxxx \
  --name "Security"

# Tạo OU con (lồng nhau)
aws organizations create-organizational-unit \
  --parent-id ou-xxxx-aaaaaaaa \   # Parent OU ID
  --name "Production"

# Liệt kê OU con của một parent
aws organizations list-children \
  --parent-id r-xxxx \
  --child-type ORGANIZATIONAL_UNIT

# Đổi tên OU
aws organizations update-organizational-unit \
  --organizational-unit-id ou-xxxx-aaaaaaaa \
  --name "Security-Updated"

# Xóa OU (phải rỗng — không có account hoặc OU con)
aws organizations delete-organizational-unit \
  --organizational-unit-id ou-xxxx-aaaaaaaa
```

### Di Chuyển Account Vào OU

```bash
# Di chuyển account từ Root vào OU
aws organizations move-account \
  --account-id 123456789012 \
  --source-parent-id r-xxxx \
  --destination-parent-id ou-xxxx-aaaaaaaa

# Di chuyển từ OU này sang OU khác
aws organizations move-account \
  --account-id 123456789012 \
  --source-parent-id ou-xxxx-aaaaaaaa \
  --destination-parent-id ou-xxxx-bbbbbbbb
```

---

## Hierarchy — Cây Phân Cấp

### Giới Hạn Độ Sâu

```
Root (cấp 0)
└── OU (cấp 1) ← Tối đa 5 cấp OU từ Root
    └── OU (cấp 2)
        └── OU (cấp 3)
            └── OU (cấp 4)
                └── OU (cấp 5) ← Sâu nhất có thể
                    └── Member Account ← Luôn là leaf node
```

### Thiết Kế Hierarchy Theo AWS Best Practice

```
Organization Root
│
├── Management Account (không thuộc OU nào)
│
├── OU: Security
│   ├── Account: log-archive          ← Centralized logging (CloudTrail, Config, VPC Flow Logs)
│   └── Account: security-tooling     ← GuardDuty admin, Security Hub, Macie
│
├── OU: Infrastructure
│   ├── Account: shared-services      ← Active Directory, DNS, NTP
│   ├── Account: network              ← Transit Gateway, VPN, Direct Connect
│   └── Account: backup               ← AWS Backup centralized
│
├── OU: Workloads
│   ├── OU: Production
│   │   ├── Account: app-prod
│   │   ├── Account: data-prod
│   │   └── Account: payments-prod    ← PCI-DSS scope tách riêng
│   │
│   └── OU: Non-Production
│       ├── Account: app-dev
│       ├── Account: app-staging
│       └── Account: data-dev
│
└── OU: Sandbox
    ├── Account: engineer-alice-sandbox
    └── Account: engineer-bob-sandbox
```

### Tại Sao Thiết Kế Theo Môi Trường Thay Vì Theo Team?

```
❌ Thiết kế theo team (không khuyến nghị):
   OU: TeamA → account-teama-dev, account-teama-prod
   OU: TeamB → account-teamb-dev, account-teamb-prod
   
   Vấn đề: SCP cho production phải lặp lại cho mỗi OU team
   Khó audit: production account nằm ở nhiều OU khác nhau

✅ Thiết kế theo môi trường (khuyến nghị):
   OU: Production → tất cả production accounts của mọi team
   OU: NonProd → dev, staging của mọi team
   
   Ưu điểm:
   - 1 SCP nghiêm ngặt cho Production OU — áp dụng cho tất cả
   - 1 SCP thoải mái hơn cho NonProd OU
   - Dễ audit tất cả production workloads
   - Dễ apply tag policy đồng nhất
```

---

## Quản Lý Account

### Thông Tin Account Trong Organization

```bash
# Liệt kê tất cả accounts
aws organizations list-accounts

# Xem thông tin một account
aws organizations describe-account --account-id 123456789012

# Liệt kê accounts trong một OU
aws organizations list-accounts-for-parent \
  --parent-id ou-xxxx-aaaaaaaa

# Tìm OU chứa một account
aws organizations list-parents \
  --child-id 123456789012

# Xem tag của account trong Organization
aws organizations list-tags-for-resource \
  --resource-id 123456789012
```

### Tag Account Để Quản Lý

```bash
# Gán tag cho account (từ Management Account)
aws organizations tag-resource \
  --resource-id 123456789012 \
  --tags '[
    {"Key": "Environment", "Value": "Production"},
    {"Key": "Team", "Value": "Platform"},
    {"Key": "CostCenter", "Value": "CC-001"},
    {"Key": "DataClassification", "Value": "Confidential"}
  ]'
```

---

## Feature Sets

### CONSOLIDATED_BILLING vs ALL

```
CONSOLIDATED_BILLING (Tính Năng Tối Thiểu):
├── Hợp nhất hóa đơn từ nhiều account
├── Volume discount tự động
├── RI/Savings Plans sharing giữa accounts
└── ❌ KHÔNG có SCP, Tag Policy, AI Opt-Out

ALL (Khuyến Nghị Cho Production):
├── Tất cả tính năng CONSOLIDATED_BILLING
├── ✅ SCP — Service Control Policy
├── ✅ Tag Policy — chuẩn hóa tags
├── ✅ AI Services Opt-Out Policy
└── ✅ Backup Policy (gắn với AWS Backup)
```

```bash
# Chuyển từ CONSOLIDATED_BILLING sang ALL
aws organizations enable-all-features
# ⚠️ Cần tất cả member accounts accept — không thể rollback dễ dàng
```

---

## Giới Hạn Và Quota

| Tài Nguyên | Giới Hạn Mặc Định | Có Thể Tăng? |
|-----------|-------------------|--------------|
| Accounts trong Organization | 10 (mới tạo) → 10.000 (sau verify) | ✅ Request AWS |
| OU trong Organization | 1.000 | ✅ Request AWS |
| Cấp sâu OU (từ Root) | 5 | ❌ |
| SCP trong Organization | 300 | ✅ |
| SCP gắn vào một target | 5 | ❌ |
| Ký tự trong SCP | 5.120 | ❌ |
| Delegated Administrators | 3 per service | ❌ |

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Management Account có bị SCP giới hạn không?**
> **Không.** SCP hoàn toàn không áp dụng cho Management Account, kể cả root user. Đây là lý do critical: không chạy workload production trong Management Account — compromise Management Account là compromise toàn bộ Organization.

**Q: Một Member Account có thể thuộc nhiều OU cùng lúc không?**
> **Không.** Mỗi account chỉ nằm trong **1 parent** tại một thời điểm (1 OU hoặc Root). Không có multi-parent như một số hệ thống directory khác. Khi di chuyển account sang OU mới, nó rời OU cũ.

**Q: Có thể đổi Management Account không?**
> **Không trực tiếp.** Không có tính năng "transfer management account". Giải pháp phức tạp: tạo Organization mới trong account muốn làm management, rồi mời từng member account cũ join. Không có migration tool chính thức.

### Nâng Cao

**Q: Khi xóa Member Account, điều gì xảy ra?**
> Account bị xóa khỏi Organization trở thành **standalone account** — cần có payment method riêng, mất consolidated billing benefits. Tài nguyên trong account **không bị xóa** — chỉ account rời khỏi Organization. Để xóa hoàn toàn account, phải làm thủ tục riêng (rất phức tạp, hiếm khi cần).

**Q: Service Quota trong Organizations hoạt động thế nào?**
> Mỗi Member Account có **quota riêng** cho mọi AWS service — hoàn toàn độc lập. Account A dùng hết EC2 quota không ảnh hưởng Account B. Đây là một lợi thế lớn của multi-account strategy. Tuy nhiên, quota mới tạo account thường rất thấp — cần request tăng quota sớm cho production accounts.

---

## 🔗 Điều Hướng

| Trước | File Này | Tiếp Theo |
|-------|---------|-----------|
| [README.md](./README.md) | **1-account-structure.md** | [2-scp-policies.md](./2-scp-policies.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
