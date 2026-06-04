# Multi-Account Patterns — Mô Hình Thiết Kế Đa Tài Khoản

> **Multi-Account Patterns** (Mô Hình Đa Tài Khoản) là các pattern kiến trúc đã được kiểm chứng trong thực tế để tổ chức AWS accounts hiệu quả. Từ OU design (Thiết Kế Đơn Vị Tổ Chức) đến account vending (Cấp Phát Tài Khoản Tự Động) và các loại account chuyên dụng — những pattern này là nền tảng của Landing Zone chuẩn enterprise.

---

## 📚 Mục Lục

1. [Phân Loại Account Theo Chức Năng](#phân-loại-account-theo-chức-năng)
2. [OU Design Patterns — Mẫu Thiết Kế OU](#ou-design-patterns)
3. [Account Vending — Cấp Phát Tài Khoản Tự Động](#account-vending)
4. [Networking Patterns — Mẫu Kết Nối Mạng](#networking-patterns)
5. [Identity & Access Patterns — Mẫu Quản Lý Danh Tính](#identity--access-patterns)
6. [Data Perimeter Pattern — Ranh Giới Dữ Liệu](#data-perimeter-pattern)
7. [Anti-Patterns Phổ Biến](#anti-patterns-phổ-biến)
8. [Case Study: 50-Account Organization](#case-study-50-account-organization)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Phân Loại Account Theo Chức Năng

### Account Types Chuẩn (Theo AWS Well-Architected)

```
1. Management Account (Tài Khoản Quản Lý)
   ├── Purpose: Organizations, billing, SCPs
   ├── Who: Cloud Operations team (read-only)
   └── Rule: KHÔNG chạy workload

2. Log Archive Account (Tài Khoản Lưu Trữ Nhật Ký)
   ├── Purpose: Centralized audit logs
   ├── Who: Security team (read-only)
   ├── Contains: CloudTrail, Config, VPC Flow Logs → S3
   └── Rule: S3 Object Lock (WORM), không ai được xóa logs

3. Security Tooling Account (Tài Khoản Công Cụ Bảo Mật)
   ├── Purpose: Security operations center
   ├── Who: Security Operations team
   ├── Contains: GuardDuty Admin, Security Hub Admin, Macie
   └── Rule: Delegated Admin cho security services

4. Shared Services Account (Tài Khoản Dịch Vụ Dùng Chung)
   ├── Purpose: Shared infrastructure cho tất cả accounts
   ├── Who: Platform/Infrastructure team
   ├── Contains: Active Directory, DNS, internal tools, AMI sharing
   └── Rule: Chỉ shared services, không có workload riêng

5. Network Account (Tài Khoản Mạng)
   ├── Purpose: Centralized network hub
   ├── Who: Network/Infrastructure team
   ├── Contains: Transit Gateway, VPN, Direct Connect, Egress VPC
   └── Rule: Hub của mọi kết nối mạng trong Org

6. Production Workload Account (Tài Khoản Workload Production)
   ├── Purpose: Chạy production applications
   ├── Who: Application team (với quyền hạn chế qua SCP)
   ├── Contains: Application resources, databases
   └── Rule: SCP nghiêm ngặt, MFA bắt buộc, no root user

7. Non-Production Account (Tài Khoản Không Phải Production)
   ├── Purpose: Dev, Staging, Testing
   ├── Who: Development team
   ├── Contains: Dev/Staging resources
   └── Rule: SCP thoải mái hơn, cho phép experiment

8. Sandbox Account (Tài Khoản Thử Nghiệm Tự Do)
   ├── Purpose: Học tập, POC (Proof of Concept — Thử Nghiệm Khả Thi)
   ├── Who: Individual engineers
   ├── Contains: Bất cứ thứ gì engineer muốn thử
   └── Rule: Budget limit cứng, auto-cleanup hàng tuần, region restricted
```

---

## OU Design Patterns

### Pattern 1: AWS Recommended OU Structure (Khuyến Nghị)

```
Organization Root
│
├── OU: Security
│   ├── Account: log-archive
│   └── Account: security-tooling
│
├── OU: Infrastructure
│   ├── Account: shared-services
│   └── Account: network
│
├── OU: Workloads
│   ├── OU: Production
│   │   ├── Account: app-prod
│   │   ├── Account: api-prod
│   │   └── Account: data-prod
│   │
│   └── OU: SDLC (Software Development Life Cycle)
│       ├── Account: app-dev
│       ├── Account: app-staging
│       └── Account: data-dev
│
├── OU: Policy Staging (Thử Nghiệm Policy)
│   └── Account: policy-test          ← Test SCP mới trước khi áp vào Production
│
├── OU: Suspended (Tạm Ngừng)
│   └── (Accounts cũ chờ xóa — SCP Deny tất cả)
│
└── OU: Individual Business Units (Đơn Vị Kinh Doanh)
    ├── OU: TeamA
    └── OU: TeamB
```

### Pattern 2: Environment-First vs Domain-First

#### Environment-First (Theo Môi Trường — Phổ Biến Hơn)

```
OU: Production     → Tất cả production accounts
OU: Staging        → Tất cả staging accounts
OU: Development    → Tất cả dev accounts

Ưu điểm:
✅ SCP áp dụng đồng nhất theo môi trường
✅ Dễ enforce "không ai vào production trực tiếp"
✅ Cost Explorer grouping theo OU dễ hơn
✅ Audit: tất cả production workloads ở một nơi

Nhược điểm:
❌ Khó thấy tất cả accounts của một team/domain
```

#### Domain-First (Theo Domain/Team)

```
OU: Payments
  ├── Account: payments-prod
  ├── Account: payments-staging
  └── Account: payments-dev

OU: Analytics
  ├── Account: analytics-prod
  └── Account: analytics-dev

Ưu điểm:
✅ Team tự quản lý accounts của họ
✅ Dễ delegate OU-level management cho team lead
✅ Phù hợp khi các domain có compliance requirements rất khác nhau
   (Payments: PCI-DSS, Healthcare: HIPAA...)

Nhược điểm:
❌ SCP phải duplicate hoặc phức tạp hóa để áp production-level control
❌ Khó enforce đồng nhất SCP cho "tất cả production accounts"
```

#### Hybrid Pattern (Thực Tế Phổ Biến Nhất)

```
OU: Security                       ← Luôn tách riêng
OU: Infrastructure                 ← Luôn tách riêng
OU: Production
  ├── OU: Payments (PCI-DSS)       ← Domain riêng vì compliance khác biệt
  └── OU: General Production       ← Accounts sản xuất thông thường
OU: NonProduction
  └── (chia theo team nếu cần)
OU: Sandbox                        ← Luôn tách riêng
```

### SCP Mapping Theo OU

```
Root SCP:
  ✅ FullAWSAccess (Allow *)
  ✅ DenyLeaveOrganization
  ✅ DenyCloseAccount
  ✅ DenyDisableCloudTrail
  ✅ DenyDisableGuardDuty

OU: Production (thêm):
  ✅ DenyNonApprovedRegions
  ✅ RequireMFA
  ✅ DenyRootUser
  ✅ DenyDeletionWithoutApproval (CloudTrail, Config, GuardDuty)
  ✅ RequireTagsOnCreate (EC2, RDS, S3)
  ✅ DenyPurchaseUnauthorizedRIs

OU: Sandbox (thêm):
  ✅ DenyExpensiveInstances (deny r5.4xlarge và lớn hơn)
  ✅ BudgetHardLimit (deny actions khi vượt $500/tháng qua budget action)
  ✅ DenyProductionData (deny copy từ production account)

OU: Security:
  ✅ DenyModifySecurityTools
  ✅ RequireVPCEndpoint (deny S3 access nếu không qua VPC Endpoint)
```

---

## Account Vending

### Account Vending (Cấp Phát Tài Khoản Tự Động) Là Gì?

**Account Vending** là quy trình **tự động hóa** tạo và cấu hình account mới theo template chuẩn — giống máy bán hàng tự động, developer "order" một account và nhận được account đã được cấu hình đầy đủ.

```
Không có Account Vending:
  1. Tạo account thủ công → email thủ công
  2. Cấu hình billing thủ công
  3. Setup VPC thủ công
  4. Tạo IAM roles thủ công
  5. Enable GuardDuty thủ công
  6. Apply tags thủ công
  → 2-4 giờ mỗi account, dễ mắc lỗi

Với Account Vending:
  1. Developer điền form (team name, environment, cost center)
  2. Pipeline chạy tự động:
     - Tạo account (Organizations API)
     - Di chuyển vào OU đúng
     - Apply baseline CloudFormation StackSet
     - Enable security tools (GuardDuty, Security Hub)
     - Setup VPC từ template chuẩn
     - Create IAM roles, SSO assignments
     - Apply tags, budget alerts
  → 15-30 phút, zero manual steps
```

### Công Cụ Account Vending

#### 1. AWS Control Tower Account Factory (Khuyến Nghị)

```
Control Tower Account Factory:
├── Console hoặc Service Catalog interface
├── Account Factory for Terraform (AFT)
│   └── Terraform-based, GitHub Actions tích hợp
└── Tự động: baseline CFN, SCPs, SSO, logging
```

#### 2. Account Factory Tự Xây Với AWS Lambda

```python
# Ví dụ Lambda handler cho Account Vending Pipeline
import boto3

def create_account(event, context):
    org_client = boto3.client('organizations')
    
    # 1. Tạo account
    response = org_client.create_account(
        Email=event['email'],
        AccountName=event['account_name'],
        RoleName='OrganizationAccountAccessRole',
        IamUserAccessToBilling='ALLOW'
    )
    
    request_id = response['CreateAccountStatus']['Id']
    
    # 2. Chờ account được tạo
    waiter_response = wait_for_account_creation(org_client, request_id)
    account_id = waiter_response['AccountId']
    
    # 3. Di chuyển vào OU đúng
    ou_id = get_ou_for_environment(event['environment'])
    org_client.move_account(
        AccountId=account_id,
        SourceParentId=get_root_id(org_client),
        DestinationParentId=ou_id
    )
    
    # 4. Assume role và cấu hình account
    assume_role_and_configure(account_id, event)
    
    return {'AccountId': account_id, 'Status': 'Created'}
```

#### 3. Terraform (HashiCorp) + AWS Organizations

```hcl
# Tạo account với Terraform
resource "aws_organizations_account" "workload" {
  name      = "App Production"
  email     = "app-prod@company.com"
  role_name = "OrganizationAccountAccessRole"
  
  parent_id = aws_organizations_organizational_unit.production.id
  
  tags = {
    Environment = "Production"
    Team        = "Platform"
    CostCenter  = "CC-001"
  }
}

# Deploy baseline StackSet vào account mới
resource "aws_cloudformation_stack_set_instance" "baseline" {
  stack_set_name = aws_cloudformation_stack_set.baseline.name
  account_id     = aws_organizations_account.workload.id
  region         = "ap-southeast-1"
}
```

### Baseline Account Configuration (Cấu Hình Cơ Bản)

```
Mỗi account mới được tạo phải có:

Security:
├── GuardDuty detector (tự động qua Organization)
├── Security Hub enabled (tự động qua Organization)
├── CloudTrail — Organization Trail đã cover
├── AWS Config — Organization Config đã cover
└── VPC Flow Logs enabled

Networking:
├── Default VPC xóa (tránh dùng default VPC)
├── VPC chuẩn từ template (public/private/data subnets)
└── VPC Endpoint cho S3, STS (tránh egress cost + security)

IAM:
├── IAM Password Policy cứng
├── Break-glass role (emergency access)
├── Service roles cần thiết
└── SSO Permission Sets assigned

Monitoring:
├── CloudWatch alarms cơ bản
├── Budget alert (notify khi 80% budget)
└── Cost allocation tags enabled

Compliance:
└── Tags bắt buộc: Environment, Team, CostCenter
```

---

## Networking Patterns

### Hub-and-Spoke với Transit Gateway

```
                    Internet
                       │
              ┌────────┴────────┐
              │   Egress VPC    │
              │  (Network Acc.) │
              │  NAT Gateway    │
              └────────┬────────┘
                       │
              ┌────────┴────────┐
              │ Transit Gateway │  ← Shared từ Network Account
              │  (Network Acc.) │     qua RAM (Resource Access Manager)
              └──────┬──┬──┬───┘
                     │  │  │
        ┌────────────┘  │  └────────────┐
        │               │               │
┌───────┴──────┐ ┌──────┴──────┐ ┌─────┴───────┐
│  App Prod    │ │  API Prod   │ │  Data Prod  │
│   VPC        │ │    VPC      │ │    VPC      │
│  10.0.0.0/16 │ │ 10.1.0.0/16 │ │ 10.2.0.0/16 │
└──────────────┘ └─────────────┘ └─────────────┘

→ Tất cả ra internet qua Egress VPC (centralized inspection)
→ Accounts không có direct internet gateway
→ Non-routable với nhau trừ khi có Transit Gateway routing rule
```

### VPC Peering vs Transit Gateway

| Tiêu Chí | VPC Peering | Transit Gateway |
|---------|-------------|-----------------|
| **Số kết nối** | Tăng theo n² | Hub-and-spoke, tuyến tính |
| **Transitive routing** | ❌ Không | ✅ Có |
| **Cross-account** | ✅ Có | ✅ Có (qua RAM) |
| **Bandwidth limit** | Không giới hạn | 50 Gbps/AZ |
| **Chi phí** | Thấp (2 VPCs) | Cao hơn (per attachment + data) |
| **Dùng khi** | ≤3 VPCs cần kết nối | ≥4 VPCs, many accounts |

---

## Identity & Access Patterns

### IAM Identity Center (SSO) Centralized

```
Active Directory / Okta / Azure AD
              │ SAML / SCIM sync
              ▼
    IAM Identity Center
    (Management Account or Delegated Admin)
    ├── Users & Groups (sync từ IdP)
    ├── Permission Sets (quyền preset)
    │   ├── AdministratorAccess (chỉ Break-glass)
    │   ├── PowerUserAccess (Dev team)
    │   ├── ReadOnlyAccess (Auditor)
    │   └── DatabaseAdmin (DBA team)
    └── Account Assignments
        ├── Group: Platform-Admin → Account: Management → AdministratorAccess
        ├── Group: AppTeam-A → Account: app-prod → PowerUserAccess
        └── Group: Auditors → All accounts → ReadOnlyAccess
```

### Permission Boundary Pattern

```
Scenario: Dev team tự tạo IAM roles cho ứng dụng,
nhưng không được tạo role có quyền cao hơn họ

Giải pháp: Permission Boundary (Ranh Giới Quyền)

1. Platform team tạo Permission Boundary policy:
   Allow: ec2:*, s3:*, rds:*, lambda:*, cloudwatch:*
   Deny: iam:*, organizations:*, billing:*

2. SCP đảm bảo mọi IAM role mới đều có Permission Boundary:
   Condition: {"Null": {"iam:PermissionsBoundary": "true"}} → DENY

3. Dev team tạo IAM role:
   Phải attach Permission Boundary → role bị giới hạn trong boundary
   → Không thể leo thang quyền (privilege escalation)
```

---

## Data Perimeter Pattern

### Data Perimeter (Ranh Giới Dữ Liệu) Là Gì?

**Data Perimeter** đảm bảo dữ liệu nhạy cảm chỉ được truy cập bởi:
- **Trusted Identities** (Danh Tính Đáng Tin) — chỉ principals trong Organization của bạn
- **Trusted Resources** (Tài Nguyên Đáng Tin) — chỉ resources thuộc Organization
- **Expected Networks** (Mạng Kỳ Vọng) — chỉ từ IP/VPC được phê duyệt

### Áp Dụng Data Perimeter Với SCP

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceDataPerimeter",
      "Effect": "Deny",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        },
        "BoolIfExists": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }
  ]
}
```

### S3 Bucket Policy Cho Data Perimeter

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyOrganizationAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-sensitive-bucket",
        "arn:aws:s3:::my-sensitive-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxxxx"
        }
      }
    }
  ]
}
```

---

## Anti-Patterns Phổ Biến

### Anti-Pattern 1: God Account (Tài Khoản Toàn Quyền)

```
❌ Sai:
  Management Account chạy production workload
  + quản lý Organizations
  + có billing access
  + nhiều engineer đăng nhập hàng ngày

✅ Đúng:
  Management Account: chỉ Organizations + billing
  Ít người có quyền vào, MFA bắt buộc, không có workload
```

### Anti-Pattern 2: One Account Per Application (Một Account Cho Mỗi Ứng Dụng)

```
❌ Quá nhiều accounts:
  account-user-service-prod, account-user-service-dev
  account-order-service-prod, account-order-service-dev
  account-payment-service-prod, account-payment-service-dev
  ...
  → 50+ accounts cho một medium-size company
  → Operational overhead cực cao
  → IAM Identity Center assignment phức tạp

✅ Cân bằng:
  app-prod (chứa nhiều microservices, tách biệt bằng IAM, VPC)
  app-dev (tất cả dev services)
  data-prod (databases, data lake)
  data-dev

  Dùng namespace/tagging trong account thay vì tạo account riêng
  cho mọi microservice nhỏ
```

### Anti-Pattern 3: Flat OU Structure

```
❌ Flat (không có hierarchy):
  Root → OU: All-Accounts → 100 accounts

  Vấn đề: SCP phải apply cho từng account riêng
  Không thể apply policy đồng nhất theo nhóm

✅ Hierarchy theo purpose:
  Root → OU: Production → SCP production standards
       → OU: Dev → SCP dev standards
       → OU: Security → SCP security standards
```

### Anti-Pattern 4: Bỏ Qua Policy Staging

```
❌ Test SCP trực tiếp trên Production OU:
  → SCP mới có lỗi → block tất cả production resources
  → Outage ngay lập tức

✅ Policy Staging OU:
  1. Tạo Policy Staging OU với account sandbox
  2. Apply SCP mới vào Policy Staging → test thật
  3. Chỉ khi OK → apply vào Production OU
```

### Anti-Pattern 5: Không Có Automation Cho Account Creation

```
❌ Tạo account thủ công:
  → Mỗi account cấu hình khác nhau
  → Missing security baseline (GuardDuty, Config...)
  → Mất nhiều giờ mỗi account
  → Con người mắc lỗi

✅ Account Vending Pipeline:
  → Tất cả accounts identical từ ngày 1
  → Security baseline guaranteed
  → 15-30 phút một account
  → Fully auditable (IaC trong Git)
```

---

## Case Study: 50-Account Organization

### Bối Cảnh

```
Công ty: Fintech với 200 engineer, 20 team
Yêu cầu: PCI-DSS, SOC2
Số accounts: 50+ accounts, tăng thêm ~2/tháng
```

### OU Structure

```
Organization Root (o-xxxxxxxxxx)
│
├── OU: Security (SCP: Deny tất cả, chỉ Allow security services)
│   ├── Account: log-archive (999000000001)
│   └── Account: security-tooling (999000000002)
│
├── OU: Infrastructure (SCP: Deny tạo public resources)
│   ├── Account: shared-services (999000000003)
│   ├── Account: network-hub (999000000004)
│   └── Account: backup-central (999000000005)
│
├── OU: Workloads
│   │
│   ├── OU: PCI-DSS Scope (SCP extra: DenyNonPCIResources, RequireEncryption)
│   │   ├── Account: payments-prod (111000000001)
│   │   └── Account: payments-staging (111000000002)
│   │
│   ├── OU: Production (SCP: DenyNonApprovedRegions, RequireMFA, DenyRootUser)
│   │   ├── Account: app-prod (222000000001)
│   │   ├── Account: api-prod (222000000002)
│   │   ├── Account: data-prod (222000000003)
│   │   └── Account: analytics-prod (222000000004)
│   │
│   └── OU: Non-Production (SCP: DenyExpensiveInstances, BudgetLimit)
│       ├── Account: app-dev (333000000001)
│       ├── Account: app-staging (333000000002)
│       ├── Account: data-dev (333000000003)
│       └── Account: analytics-dev (333000000004)
│
├── OU: Sandbox (SCP: RegionRestrict ap-southeast-1 only, NoBudgetOver$500)
│   ├── Account: sandbox-team-platform (444000000001)
│   ├── Account: sandbox-team-data (444000000002)
│   └── (mỗi team 1 sandbox account)
│
└── OU: Policy Staging (cho test SCP mới)
    └── Account: policy-test (555000000001)
```

### SCP Strategy

```
FullAWSAccess (gắn vào Root — AWS default, giữ nguyên)
DenyOrgLeave (gắn vào Root)
DenyCloudTrailModify (gắn vào Root)
DenyGuardDutyModify (gắn vào Root)

DenyNonApprovedRegions (gắn vào OU: Production, OU: PCI-DSS)
RequireMFAForConsoleAccess (gắn vào OU: Production, OU: PCI-DSS)
DenyRootUserActions (gắn vào OU: Production, OU: PCI-DSS)
RequireS3Encryption (gắn vào OU: PCI-DSS)
RequireEC2Encryption (gắn vào OU: PCI-DSS)

DenyLargeInstances (gắn vào OU: Sandbox)
BudgetHardStop (gắn vào OU: Sandbox)
```

### Delegated Administrators

```
security-tooling (999000000002):
├── GuardDuty Administrator
├── Security Hub Administrator
├── AWS Config Aggregator
├── Macie Administrator
└── Inspector Administrator

shared-services (999000000003):
└── CloudFormation StackSets (deploy baseline vào accounts mới)

network-hub (999000000004):
└── RAM (Resource Access Manager) — chia sẻ Transit Gateway
```

---

## Câu Hỏi Phỏng Vấn

### Cơ Bản

**Q: Khi nào nên tạo account mới thay vì dùng chung account?**
> Nên tạo account mới khi: (1) **Security boundary** cần thiết — môi trường prod/dev/sandbox; (2) **Compliance scope isolation** — PCI-DSS, HIPAA account riêng; (3) **Blast radius** — muốn sự cố không lan sang; (4) **Service quota isolation** — team lớn có thể cần quota riêng. Không nên tạo account riêng cho mỗi microservice nhỏ — overhead quá lớn. Trong account, dùng VPC/IAM/tags để tách biệt.

**Q: Làm thế nào đảm bảo account mới luôn được cấu hình đúng chuẩn bảo mật?**
> Dùng **Account Vending Pipeline**: (1) CloudFormation StackSets với auto-deployment — khi account join Organization, StackSet tự deploy baseline resources (IAM roles, Config rules, CloudWatch alarms); (2) Trusted Access + Organization-level GuardDuty/Security Hub tự động enable; (3) Organization Trail cover tất cả accounts ngay khi join; (4) SCP từ OU áp dụng ngay khi account vào OU.

**Q: Tại sao cần OU Policy Staging riêng biệt?**
> Để **test SCP an toàn** trước khi apply vào production. SCP sai có thể block mọi action trong production accounts — gây outage ngay lập tức. Policy Staging OU có một sandbox account — team Platform test SCP mới ở đây, verify không có side effect, rồi mới apply vào Production OU.

### Nâng Cao

**Q: Thiết kế OU cho fintech có PCI-DSS, làm thế nào để giữ compliance scope nhỏ nhất?**
> Tạo **OU: PCI-DSS Scope** riêng chứa chỉ accounts xử lý payment card data. Apply SCP nghiêm ngặt nhất ở đây (mandatory encryption, logging, region restriction). Accounts khác trong Organization không nằm trong PCI scope — giảm chi phí audit. Transit Gateway routing rules đảm bảo PCI accounts chỉ communicate qua approved paths. CloudTrail, Config, GuardDuty cho PCI accounts có enhanced monitoring và alerting riêng. Quarterly review ai có access vào PCI OU.

**Q: Account Vending Pipeline nên build bằng gì — Lambda tự làm, Control Tower Account Factory, hay Terraform?**
> Phụ thuộc vào context: **Control Tower Account Factory** — nếu dùng Control Tower, đây là lựa chọn natural, ít code nhất, maintained bởi AWS. **AFT (Account Factory for Terraform)** — nếu đội đã dùng Terraform và muốn IaC-native. **Lambda tự làm** — khi có requirements rất đặc thù mà Control Tower không support, hoặc không muốn dependency vào Control Tower. Trong hầu hết cases, **Control Tower + AFT** là lựa chọn balance tốt nhất giữa tính năng, maintainability và không phải tự viết tất cả.

---

## 🔗 Điều Hướng

| Trước | File Này | Tiếp Theo |
|-------|---------|-----------|
| [4-delegated-admin.md](./4-delegated-admin.md) | **5-multi-account-patterns.md** | [07-control-tower/README.md](../07-control-tower/README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
