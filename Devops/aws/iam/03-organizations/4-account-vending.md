# Account Vending — Tự Động Tạo và Chuẩn Hóa Tài Khoản AWS

> **Account Vending** (Phân Phối Tài Khoản Tự Động) là quy trình tự động tạo, cấu hình và bàn giao tài khoản AWS mới đúng chuẩn — nhanh chóng, nhất quán, và không cần can thiệp thủ công từ Platform Team.

---

## 📚 Mục Lục

1. [Vấn Đề Account Vending Giải Quyết](#1-vấn-đề-account-vending-giải-quyết)
2. [Account Vending Machine (AVM)](#2-account-vending-machine-avm)
3. [Account Factory for Terraform (AFT)](#3-account-factory-for-terraform-aft)
4. [Tự Xây Pipeline Vending Tùy Chỉnh](#4-tự-xây-pipeline-vending-tùy-chỉnh)
5. [Account Baseline Chuẩn](#5-account-baseline-chuẩn)
6. [Vòng Đời Tài Khoản](#6-vòng-đời-tài-khoản)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề Account Vending Giải Quyết

### Tình Huống Không Có Account Vending

```
Developer yêu cầu tài khoản mới:
→ Gửi email/ticket cho Platform Team
→ Platform Team tạo account thủ công: 2–5 ngày
→ Platform Team cấu hình thủ công: CloudTrail, Config, GuardDuty...
→ Cấu hình SSO thủ công
→ Tạo VPC, subnets thủ công
→ Developer nhận account sau 1 tuần
→ 6 tháng sau: cấu hình mỗi account khác nhau, khó audit
```

### Sau Khi Có Account Vending

```
Developer điền form/YAML/PR:
→ Pipeline chạy tự động: 15–30 phút
→ Account tạo với đầy đủ guardrails, SSO, CloudTrail, Config
→ VPC baseline với subnets, routing chuẩn
→ GuardDuty, Security Hub bật sẵn
→ Developer nhận email + login instructions
→ Mọi account đều giống nhau, audit dễ dàng
```

### Tiêu Chí Account Vending Tốt

```
✅ Tự động hoàn toàn  — không cần manual steps
✅ Idempotent         — chạy nhiều lần kết quả như nhau
✅ Version-controlled — thay đổi cấu hình qua git
✅ Auditable          — biết ai tạo account nào, khi nào
✅ Testable           — có thể test pipeline trước khi dùng production
✅ Extensible         — dễ thêm customization theo team/product
```

---

## 2. Account Vending Machine (AVM)

### AVM — Kiến Trúc Cổ Điển (Trước Khi Có AFT)

**AVM** là giải pháp tự xây phổ biến trước khi AWS ra Account Factory for Terraform. Nhiều tổ chức vẫn dùng pattern này:

```
AVM Architecture:
├── Service Catalog Product    — giao diện tạo account
│   └── CloudFormation Template
│
├── Lambda Function (Orchestrator)
│   ├── Gọi Organizations API tạo account
│   ├── Assume OrganizationAccountAccessRole
│   ├── Deploy baseline CloudFormation vào account mới
│   └── Cấu hình SSO
│
├── S3 Bucket
│   └── Baseline templates, scripts
│
└── SNS → Email notification khi hoàn thành
```

### Luồng AVM Đơn Giản

```python
# Lambda orchestrator — pseudocode
def create_account(event):
    params = event['ResourceProperties']
    
    # 1. Tạo account trong Organizations
    response = organizations.create_account(
        Email=params['AccountEmail'],
        AccountName=params['AccountName'],
        IamUserAccessToBilling='DENY'
    )
    account_id = wait_for_account_creation(response['CreateAccountStatus']['Id'])
    
    # 2. Di chuyển vào OU đúng
    organizations.move_account(
        AccountId=account_id,
        SourceParentId=get_root_id(),
        DestinationParentId=params['DestinationOuId']
    )
    
    # 3. Assume role vào account mới
    credentials = assume_role(
        f"arn:aws:iam::{account_id}:role/OrganizationAccountAccessRole"
    )
    
    # 4. Deploy baseline stack
    cfn = boto3.client('cloudformation', **credentials)
    cfn.create_stack(
        StackName='account-baseline',
        TemplateURL=params['BaselineTemplateUrl'],
        Capabilities=['CAPABILITY_NAMED_IAM'],
        Parameters=[
            {'ParameterKey': 'Environment', 'ParameterValue': params['Environment']},
            {'ParameterKey': 'CostCenter', 'ParameterValue': params['CostCenter']}
        ]
    )
    
    # 5. Cấu hình SSO
    assign_permission_sets(account_id, params['SSOGroupName'])
    
    return {'AccountId': account_id}
```

---

## 3. Account Factory for Terraform (AFT)

### AFT — Giải Pháp AWS Chính Thức

**AFT** (Account Factory for Terraform) là giải pháp AWS open-source tích hợp Control Tower, cho phép quản lý account vending bằng Terraform và GitOps.

### Kiến Trúc AFT

```
┌─────────────────── AFT Management Account ──────────────────┐
│                                                               │
│  GitHub/CodeCommit Repo                                       │
│  ├── account-requests/      ──→  CodePipeline                │
│  │   └── new-account.tf         ├── Source (git)             │
│  ├── global-customizations/     ├── Build (CodeBuild/TF)     │
│  └── account-customizations/    └── Deploy (AWS APIs)        │
│                                                               │
│  DynamoDB: AFT state & request tracking                       │
│  S3: Terraform state backend                                  │
│  SSM Parameter Store: Configuration values                    │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼ Creates & configures
              ┌───────────────────────────────┐
              │    New Member Account          │
              │    (Automatically configured) │
              └───────────────────────────────┘
```

### Cài Đặt AFT

```hcl
# aft-bootstrap/main.tf — chạy từ Management Account
module "aft" {
  source  = "aws-ia/control_tower_aft/aws"
  version = "1.11.1"

  # Control Tower home region
  ct_home_region = "ap-southeast-1"

  # AFT management account (riêng, không phải CT management account)
  aft_management_account_id = "111111111111"

  # Log Archive và Audit accounts (được CT tạo sẵn)
  log_archive_account_id = "222222222222"
  audit_account_id       = "333333333333"

  # Git provider cho account requests
  vcs_provider                                  = "github"
  account_request_repo_name                     = "myorg/aft-account-requests"
  global_customizations_repo_name               = "myorg/aft-global-customizations"
  account_customizations_repo_name              = "myorg/aft-account-customizations"
  account_provisioning_customizations_repo_name = "myorg/aft-account-provisioning-customizations"

  # Terraform version
  terraform_version      = "1.5.7"
  terraform_distribution = "oss"  # hoặc "tfc" cho Terraform Cloud

  # CloudWatch Logs retention
  cloudwatch_log_group_retention = "90"
}
```

### Tạo Account Yêu Cầu

```hcl
# account-requests/product-b-prod.tf
module "product_b_prod" {
  source = "./modules/aft-account-request"

  control_tower_parameters = {
    AccountEmail = "product-b-prod@company.com"
    AccountName  = "Product-B-Production"

    # OU đích — phải là OU được Control Tower quản lý
    ManagedOrganizationalUnit = "Workloads/Production"

    # SSO user sẽ nhận AdminAccess permission set
    SSOUserEmail     = "team-b@company.com"
    SSOUserFirstName = "Team"
    SSOUserLastName  = "B"
  }

  account_tags = {
    Environment = "production"
    Product     = "product-b"
    CostCenter  = "team-b"
    Owner       = "team-b@company.com"
    CreatedBy   = "aft"
  }

  # Customization templates áp dụng riêng cho account này
  account_customizations_name = "production-workload"

  change_management_parameters = {
    change_requested_by = "Platform Team"
    change_reason       = "New product B production environment"
  }
}
```

### Global Customizations (Áp Dụng Mọi Account)

```hcl
# global-customizations/terraform/main.tf
# Chạy trong MỌI account sau khi tạo

# Bật IMDSv2 bắt buộc cho EC2 (ngăn SSRF attacks)
resource "aws_ec2_instance_metadata_defaults" "require_imdsv2" {
  http_tokens = "required"
}

# Tắt public access cho S3 ở cấp account
resource "aws_s3_account_public_access_block" "default" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Bật EBS encryption mặc định
resource "aws_ebs_encryption_by_default" "default" {
  enabled = true
}

# Bật GuardDuty
resource "aws_guardduty_detector" "default" {
  enable                       = true
  finding_publishing_frequency = "FIFTEEN_MINUTES"
}

# Tag mặc định cho tất cả tài nguyên (khi dùng Terraform)
provider "aws" {
  default_tags {
    tags = {
      ManagedBy   = "aft"
      Environment = var.environment
    }
  }
}
```

### Account Customizations (Áp Dụng Riêng Từng Loại)

```hcl
# account-customizations/production-workload/terraform/main.tf
# Chạy cho các account có customizations_name = "production-workload"

# VPC chuẩn cho production
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "production-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  enable_vpn_gateway   = false
  enable_dns_hostnames = true
  enable_dns_support   = true

  # Flow logs vào CloudWatch
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group = true
  create_flow_log_cloudwatch_iam_role  = true
}

# Budget alert cho production
resource "aws_budgets_budget" "monthly" {
  name              = "monthly-production-budget"
  budget_type       = "COST"
  limit_amount      = "5000"
  limit_unit        = "USD"
  time_unit         = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["finops@company.com"]
  }
}
```

---

## 4. Tự Xây Pipeline Vending Tùy Chỉnh

### Khi Nào Tự Xây

```
Cân nhắc tự xây khi:
├── Không dùng Control Tower
├── Yêu cầu workflow phê duyệt phức tạp (multi-step approval)
├── Tích hợp với ITSM (ServiceNow, Jira Service Management)
├── Cần provisioning logic rất đặc thù
└── Sử dụng CDK thay vì Terraform
```

### Pattern: Step Functions + Lambda

```
Account Request (API/Form/PR)
          │
          ▼
┌─── AWS Step Functions ──────────────────────────────────────┐
│                                                               │
│  1. ValidateRequest         — kiểm tra tham số đầu vào       │
│       │                                                       │
│  2. RequestApproval         — gửi email, chờ approval        │
│       │ (Manual approval step)                                │
│  3. CreateAccount           — Organizations.CreateAccount    │
│       │                                                       │
│  4. WaitForAccountReady     — poll status, retry loop        │
│       │                                                       │
│  5. MoveToOU                — di chuyển vào OU đúng          │
│       │                                                       │
│  6. DeployBaseline          — CloudFormation/CDK baseline     │
│       │                                                       │
│  7. ConfigureSSO            — gán permission sets             │
│       │                                                       │
│  8. RunCustomizations       — customizations theo type        │
│       │                                                       │
│  9. NotifyRequester         — gửi email thông báo hoàn thành │
└───────────────────────────────────────────────────────────────┘
```

### Step Functions Definition (Tóm Tắt)

```json
{
  "Comment": "Account Vending Pipeline",
  "StartAt": "ValidateRequest",
  "States": {
    "ValidateRequest": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:validate-account-request",
      "Next": "RequestApproval",
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "Next": "NotifyValidationFailure"
        }
      ]
    },
    "RequestApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Parameters": {
        "QueueUrl": "https://sqs.REGION.amazonaws.com/ACCOUNT/approvals",
        "MessageBody": {
          "TaskToken.$": "$$.Task.Token",
          "Input.$": "$"
        }
      },
      "HeartbeatSeconds": 86400,
      "Next": "CreateAccount"
    },
    "CreateAccount": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:create-aws-account",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 30,
          "MaxAttempts": 3
        }
      ],
      "Next": "WaitForAccountReady"
    },
    "WaitForAccountReady": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "CheckAccountStatus"
    },
    "CheckAccountStatus": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT:function:check-account-status",
      "Next": "IsAccountReady"
    },
    "IsAccountReady": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.status",
          "StringEquals": "SUCCEEDED",
          "Next": "MoveToOU"
        },
        {
          "Variable": "$.status",
          "StringEquals": "IN_PROGRESS",
          "Next": "WaitForAccountReady"
        }
      ],
      "Default": "NotifyCreationFailure"
    }
  }
}
```

---

## 5. Account Baseline Chuẩn

### Danh Sách Tài Nguyên Baseline Tối Thiểu

```
Account Baseline (Cấu Hình Nền Tảng Tối Thiểu):

Security:
├── ✅ GuardDuty detector bật
├── ✅ Security Hub bật + join tổ chức
├── ✅ IAM Access Analyzer (zone of trust = organization)
├── ✅ S3 account-level public access block
├── ✅ EBS encryption by default
├── ✅ IMDSv2 required cho EC2
└── ✅ Password policy chuẩn cho IAM users

Logging & Auditing:
├── ✅ CloudTrail trail (nhận qua organization trail)
├── ✅ Config recorder + delivery channel
├── ✅ VPC Flow Logs (khi tạo VPC)
└── ✅ CloudWatch log group retention policy

Networking:
├── ✅ Delete default VPC (optional nhưng khuyến nghị)
├── ✅ VPC baseline theo chuẩn tổ chức
└── ✅ VPC Endpoints cho S3, DynamoDB (tiết kiệm cost và tăng security)

Tagging:
├── ✅ Environment tag
├── ✅ CostCenter tag
├── ✅ Owner tag
└── ✅ ManagedBy tag

Notifications:
├── ✅ Budget alert (80%, 100%)
└── ✅ Root activity alert (CloudWatch Events → SNS)
```

### Baseline CloudFormation Template (Tóm Tắt)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Account Security Baseline

Parameters:
  Environment:
    Type: String
    AllowedValues: [production, staging, development, sandbox]
  
  CostCenter:
    Type: String

  AlertEmail:
    Type: String

Resources:
  # S3 Block Public Access
  S3AccountPublicAccessBlock:
    Type: AWS::S3::AccountPublicAccessBlock
    Properties:
      BlockPublicAcls: true
      BlockPublicPolicy: true
      IgnorePublicAcls: true
      RestrictPublicBuckets: true

  # GuardDuty
  GuardDutyDetector:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: true
      FindingPublishingFrequency: FIFTEEN_MINUTES

  # IAM Password Policy
  IAMPasswordPolicy:
    Type: AWS::IAM::AccountPasswordPolicy
    Properties:
      MinimumPasswordLength: 14
      RequireUppercaseCharacters: true
      RequireLowercaseCharacters: true
      RequireNumbers: true
      RequireSymbols: true
      MaxPasswordAge: 90
      PasswordReusePrevention: 24
      HardExpiry: false

  # Budget Alert
  MonthlyBudget:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetName: !Sub "monthly-budget-${Environment}"
        BudgetType: COST
        TimeUnit: MONTHLY
        BudgetLimit:
          Amount: !If [IsProduction, 10000, 1000]
          Unit: USD
      NotificationsWithSubscribers:
        - Notification:
            NotificationType: ACTUAL
            ComparisonOperator: GREATER_THAN
            Threshold: 80
          Subscribers:
            - SubscriptionType: EMAIL
              Address: !Ref AlertEmail

  # Root Activity Alert
  RootActivityRule:
    Type: AWS::Events::Rule
    Properties:
      Name: RootActivityAlert
      Description: Alert khi root account được dùng
      EventPattern:
        source:
          - aws.signin
        detail-type:
          - AWS Console Sign In via CloudTrail
        detail:
          userIdentity:
            type:
              - Root
      State: ENABLED
      Targets:
        - Id: SNSAlert
          Arn: !Ref SecurityAlertTopic

  SecurityAlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: security-alerts
      Subscription:
        - Protocol: email
          Endpoint: !Ref AlertEmail

Conditions:
  IsProduction: !Equals [!Ref Environment, production]
```

---

## 6. Vòng Đời Tài Khoản

### Trạng Thái Tài Khoản

```
REQUESTED ──→ IN_REVIEW ──→ APPROVED ──→ PROVISIONING ──→ ACTIVE
                │                               │
                └──→ REJECTED              ──→ FAILED
                                               │
                                               ▼
                                          MANUAL INTERVENTION

ACTIVE ──→ UNDER_INVESTIGATION (khi phát hiện breach)
       ──→ SUSPENDED (vi phạm policy)
       ──→ SCHEDULED_FOR_CLOSURE
       ──→ CLOSED
```

### Đóng/Xóa Tài Khoản

```bash
# Bước 1: Di chuyển account vào Suspended OU (SCP chặn mọi hoạt động)
aws organizations move-account \
  --account-id $ACCOUNT_ID \
  --source-parent-id $CURRENT_OU \
  --destination-parent-id $SUSPENDED_OU

# Bước 2: Lưu trữ audit logs trước khi xóa
# - Export CloudTrail logs về Log Archive
# - Export Config history
# - Snapshot RDS/EBS nếu cần

# Bước 3: Sau 90 ngày, đóng account
aws organizations close-account \
  --account-id $ACCOUNT_ID

# Lưu ý: AWS giữ account 90 ngày trước khi xóa hoàn toàn
# Trong 90 ngày đó, account có thể được khôi phục
```

### Account Lifecycle Automation

```python
# Lambda: account-lifecycle-manager
import boto3
from datetime import datetime, timedelta

def check_accounts_for_closure(event, context):
    """Kiểm tra accounts được đánh dấu đóng và xử lý."""
    org = boto3.client('organizations')
    
    # Lấy tất cả accounts trong Suspended OU
    suspended_accounts = get_accounts_in_ou(SUSPENDED_OU_ID)
    
    for account in suspended_accounts:
        account_id = account['Id']
        tags = get_account_tags(account_id)
        
        suspension_date = tags.get('SuspensionDate')
        if not suspension_date:
            continue
        
        # Nếu đã suspend > 90 ngày → đóng account
        suspended_at = datetime.fromisoformat(suspension_date)
        if datetime.now() - suspended_at > timedelta(days=90):
            print(f"Closing account {account_id} suspended since {suspension_date}")
            org.close_account(AccountId=account_id)
            notify_closure(account_id, tags.get('Owner'))
```

---

## 7. Câu Hỏi Phỏng Vấn

**Q: Platform Team của bạn nhận 20 yêu cầu tạo account mỗi tuần. Làm thế nào thiết kế hệ thống xử lý hiệu quả?**

> Xây Account Vending Machine dùng Step Functions + Lambda, hoặc AFT nếu đang dùng Control Tower. Account request được submit qua git PR hoặc self-service portal → pipeline tự động chạy, không cần manual intervention. Cấu hình baseline (GuardDuty, Config, SSO) được đóng gói sẵn vào customization templates, đảm bảo mọi account đều giống nhau.

**Q: Làm thế nào đảm bảo account mới luôn đạt compliance ngay khi tạo?**

> Tích hợp baseline deployment vào account vending pipeline — không hoàn thành vending nếu baseline chưa apply xong. Dùng Control Tower guardrails (mandatory) để enforce những gì không thể bỏ qua. Bật Config recorder ngay từ đầu để mọi thay đổi đều được track từ ngày 0. Cuối cùng, chạy compliance check (Security Hub score) cuối pipeline và fail nếu dưới ngưỡng.

**Q: AFT và Account Factory Console khác nhau thế nào, nên chọn cái nào?**

> AFT là GitOps-based, mọi account request và customization là code trong git — phù hợp cho platform teams muốn version control, review process, và CI/CD. Account Factory Console (Service Catalog) là UI-based, nhanh hơn để bắt đầu nhưng khó scale và audit. Nếu tổ chức đã dùng Terraform, AFT là lựa chọn tự nhiên hơn.

**Q: Khi account bị compromise, quy trình xử lý là gì?**

> 1. Ngay lập tức: Di chuyển account vào Suspended OU với SCP chặn mọi hoạt động (trừ billing/support) — ngăn attacker làm thêm damage. 2. Investigate: Dùng CloudTrail logs (từ Log Archive account — attacker không thể xóa), GuardDuty findings, và Detective để hiểu phạm vi breach. 3. Recover: Tạo account mới sạch qua vending pipeline, migrate workloads được. 4. Post-mortem: Xác định root cause và cập nhật guardrails để ngăn tái diễn.

---

## 🔗 Tiếp Theo

- **[../02-identity-federation/1-iam-identity-center.md](../02-identity-federation/1-iam-identity-center.md)** — Cấu hình SSO được account vending thiết lập
- **[2-service-control-policies.md](2-service-control-policies.md)** — SCPs được áp dụng tự động trong vending pipeline
- **[../10-advanced/4-security-automation.md](../10-advanced/4-security-automation.md)** — EventBridge + Lambda auto-remediation tương tự pattern này
