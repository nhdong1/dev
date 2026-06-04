# Customizations for Control Tower — CfCT

> **CfCT — Customizations for Control Tower** (Tùy Chỉnh Cho Control Tower) là giải pháp mã nguồn mở của AWS, cho phép mở rộng Control Tower Landing Zone với các tài nguyên CloudFormation và SCP — Service Control Policy (Chính Sách Kiểm Soát Dịch Vụ) tùy chỉnh, tự động triển khai khi account mới được enroll hoặc OU được cập nhật.

---

## 📚 Mục Lục

1. [CfCT là gì và tại sao cần?](#cfct-là-gì-và-tại-sao-cần)
2. [Kiến Trúc CfCT](#kiến-trúc-cfct)
3. [Cấu Trúc Repository CfCT](#cấu-trúc-repository-cfct)
4. [manifest.yaml — File Cấu Hình Chính](#manifestyaml--file-cấu-hình-chính)
5. [Triển Khai CloudFormation Qua CfCT](#triển-khai-cloudformation-qua-cfct)
6. [Triển Khai SCP Tùy Chỉnh Qua CfCT](#triển-khai-scp-tùy-chỉnh-qua-cfct)
7. [So Sánh CfCT vs AFT](#so-sánh-cfct-vs-aft)
8. [Best Practices](#best-practices)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## CfCT là gì và tại sao cần?

### Giới Hạn Của Control Tower Mặc Định

Control Tower cung cấp Landing Zone chuẩn tốt, nhưng mọi tổ chức đều có yêu cầu riêng:

```
Control Tower có sẵn:
✅ Guardrails phổ biến (khoảng 400+ guardrails)
✅ Account Factory tạo account cơ bản
✅ Log Archive + Audit account setup

Control Tower KHÔNG cung cấp:
❌ Tùy chỉnh VPC phức tạp (Transit Gateway attachment, custom routing)
❌ SCP tùy chỉnh theo policy riêng của công ty
❌ Bật Security Hub với specific standards cho từng OU
❌ Tạo IAM roles tùy chỉnh trong mọi account
❌ Cấu hình Budget alerts tự động trong mỗi account mới
❌ Triển khai monitoring stack tùy chỉnh
```

### CfCT Lấp Đầy Khoảng Trống

```
CfCT cho phép:
✅ Deploy bất kỳ CloudFormation template nào vào accounts/OUs
✅ Áp dụng SCP tùy chỉnh (ngoài guardrails mặc định)
✅ Tự động trigger khi account mới được enroll
✅ Trigger lại khi manifest.yaml thay đổi (pipeline CI/CD)
✅ Quản lý toàn bộ qua code (GitOps)
```

---

## Kiến Trúc CfCT

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Management Account                              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    CfCT Pipeline                             │  │
│  │                                                              │  │
│  │  Git Repo (CodeCommit/GitHub)                               │  │
│  │    ├── manifest.yaml           ← File cấu hình chính        │  │
│  │    ├── templates/              ← CloudFormation templates    │  │
│  │    └── policies/               ← SCP JSON files             │  │
│  │         ↓ git push                                          │  │
│  │  AWS CodePipeline                                           │  │
│  │    ├── Source: Git repo                                     │  │
│  │    ├── Build: CodeBuild (validate templates)               │  │
│  │    └── Deploy: AWS Lambda (CfCT Orchestrator)              │  │
│  │         ↓                                                   │  │
│  │  CloudFormation StackSets                                   │  │
│  │    ├── Deploy templates vào target accounts                 │  │
│  │    └── Deploy vào OUs                                       │  │
│  │                                                             │  │
│  │  AWS Organizations (SCP APIs)                               │  │
│  │    └── Create/Update/Attach SCPs tùy chỉnh                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  Trigger 1: git push vào repository                                │
│  Trigger 2: Control Tower lifecycle event (account enroll/update)  │
│    → EventBridge → Lambda → Khởi động pipeline                    │
└─────────────────────────────────────────────────────────────────────┘
                    ↓ Deploy đến
         ┌──────────────────────────────────┐
         │  Member Accounts / OUs           │
         │  ├── Account A (Prod)           │
         │  │   └── Custom resources       │
         │  ├── Account B (Dev)            │
         │  │   └── Custom resources       │
         │  └── Audit Account              │
         │      └── Security tooling       │
         └──────────────────────────────────┘
```

### Trigger Events

CfCT tự động chạy khi:

```
1. Git push vào repository
   → Pipeline khởi động ngay

2. Control Tower Lifecycle Events (EventBridge):
   - CreateManagedAccount (account mới tạo qua Account Factory)
   - UpdateManagedAccount (account được update)
   - RegisterOrganizationalUnit (OU mới được đăng ký với CT)

Điều này đảm bảo:
- Account mới tự động nhận customizations
- Không cần trigger thủ công
```

---

## Cấu Trúc Repository CfCT

```
customizations-for-control-tower/
│
├── manifest.yaml                    ← File cấu hình chính — BẮT ĐẦU ĐÂY
│
├── templates/                       ← CloudFormation templates
│   ├── security-hub-setup.yaml
│   ├── guardduty-setup.yaml
│   ├── budget-alerts.yaml
│   ├── vpc-flow-logs.yaml
│   └── custom-iam-roles.yaml
│
├── policies/                        ← SCP JSON files
│   ├── deny-root-actions.json
│   ├── require-mfa.json
│   ├── restrict-regions.json
│   └── deny-costly-services.json
│
└── parameters/                      ← Parameter files (tùy chọn)
    ├── prod-params.json
    └── dev-params.json
```

---

## manifest.yaml — File Cấu Hình Chính

### Cấu Trúc Tổng Quan

```yaml
# manifest.yaml — CfCT Configuration File

region: us-east-1            # Home region của Control Tower
version: 2021-03-15          # Phiên bản schema manifest

# Danh sách resources cần deploy
resources:

  # Phần 1: Deploy CloudFormation Templates
  - name: security-hub-baseline
    resource_file: templates/security-hub-setup.yaml
    deploy_method: stack_set              # Dùng CloudFormation StackSets
    deployment_targets:
      organizational_units:              # Deploy vào TẤT CẢ accounts trong OU
        - Workloads
        - Sandbox
    regions:
      - us-east-1
      - eu-west-1

  # Phần 2: Deploy SCP Tùy Chỉnh
  - name: deny-root-user-actions
    resource_file: policies/deny-root-actions.json
    deploy_method: scp                   # Đây là SCP, không phải CloudFormation
    deployment_targets:
      organizational_units:
        - Root                           # Áp dụng cho toàn bộ Organization
    regions:
      - us-east-1                        # SCP chỉ cần 1 region (toàn cục)
```

### Deploy Theo Target Phức Tạp

```yaml
resources:
  # Deploy vào accounts CỤ THỂ (không phải cả OU)
  - name: audit-account-siem-setup
    resource_file: templates/siem-integration.yaml
    deploy_method: stack_set
    deployment_targets:
      accounts:                          # Chỉ deploy vào accounts này
        - "222222222222"                 # Audit Account
    regions:
      - us-east-1

  # Deploy vào OU nhưng loại TRỪ một số accounts
  - name: budget-alerts-all-workloads
    resource_file: templates/budget-alerts.yaml
    deploy_method: stack_set
    deployment_targets:
      organizational_units:
        - Workloads
      exclude_accounts:                  # Loại trừ accounts này
        - "999999999999"                 # Legacy account không cần budget alert
    parameters:
      - parameter_key: BudgetAmount
        parameter_value: "1000"
    regions:
      - us-east-1
```

### Sử Dụng Parameters

```yaml
resources:
  - name: vpc-standard-setup
    resource_file: templates/vpc-setup.yaml
    deploy_method: stack_set
    deployment_targets:
      organizational_units:
        - Workloads/Production
    parameters:
      - parameter_key: VpcCidr
        parameter_value: "10.0.0.0/16"
      - parameter_key: EnableFlowLogs
        parameter_value: "true"
      - parameter_key: FlowLogRetentionDays
        parameter_value: "90"
    regions:
      - us-east-1
      - eu-west-1
```

---

## Triển Khai CloudFormation Qua CfCT

### Ví Dụ: Bật Security Hub Trong Mọi Account

```yaml
# templates/security-hub-setup.yaml

AWSTemplateFormatVersion: '2010-09-09'
Description: 'Security Hub baseline setup for all accounts'

Parameters:
  EnableCISStandard:
    Type: String
    Default: 'true'
    AllowedValues: ['true', 'false']
  EnableAWSFoundationsStandard:
    Type: String
    Default: 'true'
    AllowedValues: ['true', 'false']

Resources:
  # Bật Security Hub
  SecurityHub:
    Type: AWS::SecurityHub::Hub
    Properties:
      Tags:
        ManagedBy: "control-tower-cfct"

  # Bật CIS AWS Foundations Benchmark
  CISStandard:
    Type: AWS::SecurityHub::Standard
    Condition: EnableCIS
    Properties:
      StandardsArn: !Sub "arn:aws:securityhub:${AWS::Region}::standards/cis-aws-foundations-benchmark/v/1.4.0"
    DependsOn: SecurityHub

Conditions:
  EnableCIS: !Equals [!Ref EnableCISStandard, 'true']
```

### Ví Dụ: Budget Alert Cho Mỗi Account

```yaml
# templates/budget-alerts.yaml

AWSTemplateFormatVersion: '2010-09-09'
Description: 'Standard budget alert for each account'

Parameters:
  BudgetAmount:
    Type: Number
    Default: 500
  AlertEmail:
    Type: String
    Default: "cloud-billing@company.com"

Resources:
  MonthlyBudget:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetName: !Sub "monthly-budget-${AWS::AccountId}"
        BudgetType: COST
        TimeUnit: MONTHLY
        BudgetLimit:
          Amount: !Ref BudgetAmount
          Unit: USD
      NotificationsWithSubscribers:
        - Notification:
            NotificationType: ACTUAL
            ComparisonOperator: GREATER_THAN
            Threshold: 80          # Cảnh báo khi đạt 80% ngân sách
          Subscribers:
            - SubscriptionType: EMAIL
              Address: !Ref AlertEmail
        - Notification:
            NotificationType: FORECASTED
            ComparisonOperator: GREATER_THAN
            Threshold: 100         # Cảnh báo khi dự báo vượt 100%
          Subscribers:
            - SubscriptionType: EMAIL
              Address: !Ref AlertEmail
```

### Ví Dụ: GuardDuty Với Centralized Findings

```yaml
# templates/guardduty-setup.yaml

AWSTemplateFormatVersion: '2010-09-09'
Description: 'GuardDuty setup sending findings to central account'

Parameters:
  CentralAccountId:
    Type: String
    Description: "Audit Account ID to receive findings"

Resources:
  GuardDutyDetector:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: true
      FindingPublishingFrequency: FIFTEEN_MINUTES
      DataSources:
        S3Logs:
          Enable: true
        Kubernetes:
          AuditLogs:
            Enable: true

  # Gửi findings về Audit Account
  GuardDutyFindingsFilter:
    Type: AWS::Events::Rule
    Properties:
      Name: "guardduty-findings-to-central"
      EventPattern:
        source: ["aws.guardduty"]
        detail-type: ["GuardDuty Finding"]
      State: ENABLED
      Targets:
        - Arn: !Sub "arn:aws:events:${AWS::Region}:${CentralAccountId}:event-bus/default"
          Id: "CentralEventBus"
          RoleArn: !GetAtt EventBridgeCrossAccountRole.Arn

  EventBridgeCrossAccountRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: events.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: "AllowEventBusPut"
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: events:PutEvents
                Resource: !Sub "arn:aws:events:${AWS::Region}:${CentralAccountId}:event-bus/default"
```

---

## Triển Khai SCP Tùy Chỉnh Qua CfCT

### Ví Dụ: Chặn Root User Actions

```json
// policies/deny-root-actions.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

### Ví Dụ: Giới Hạn Region (Data Residency)

```json
// policies/restrict-regions.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOnlyApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "acm:*",
        "aws-marketplace-management:*",
        "aws-marketplace:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "config:*",
        "cur:*",
        "directconnect:*",
        "ec2:DescribeRegions",
        "ec2:DescribeTransitGateways",
        "ecr-public:*",
        "fms:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "kms:*",
        "mobileanalytics:*",
        "networkmanager:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "s3:GetAccountPublicAccessBlock",
        "shield:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*",
        "waf-regional:*",
        "waf:*",
        "wafv2:*",
        "wellarchitected:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "ap-southeast-1",
            "us-east-1"
          ]
        }
      }
    }
  ]
}
```

### Ví Dụ: Chặn Các Dịch Vụ Không Được Phê Duyệt

```json
// policies/deny-unapproved-services.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedServices",
      "Effect": "Deny",
      "Action": [
        "rekognition:*",
        "lex:*",
        "polly:*",
        "transcribe:*",
        "comprehend:*",
        "textract:*",
        "blockchain:*",
        "groundstation:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalTag/ExceptionApproved": "true"
        }
      }
    }
  ]
}
```

---

## So Sánh CfCT vs AFT

Đây là câu hỏi phỏng vấn phổ biến — hai giải pháp có mục đích khác nhau nhưng có thể dùng kết hợp.

| Tiêu Chí                              | CfCT                                     | AFT                                      |
| ------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| **Mục đích chính**                    | Customize resources trong accounts đã có | Tạo account mới + customize             |
| **IaC tool**                          | CloudFormation                           | Terraform                                |
| **Trigger**                           | Git push + CT lifecycle events           | Git push + CT lifecycle events           |
| **Phạm vi áp dụng**                  | OU hoặc accounts cụ thể (linh hoạt)     | Từng account riêng lẻ                    |
| **SCP tùy chỉnh**                    | ✅ Có (native)                           | ❌ Không — phải dùng CfCT hoặc tự code  |
| **Customization pipeline**            | CloudFormation StackSets                 | CodePipeline + Terraform                 |
| **Yêu cầu kỹ năng**                  | CloudFormation                           | Terraform + Python                       |
| **Phù hợp khi**                      | Dùng CloudFormation, cần SCP custom     | Dùng Terraform, cần tạo account nhiều   |
| **Kết hợp nhau được không?**         | ✅ Dùng AFT tạo account + CfCT customize |                                          |

### Kết Hợp CfCT + AFT

```
Mô hình tốt nhất (Best of both worlds):

AFT: Chịu trách nhiệm tạo account + baseline Terraform resources
     ├── Tạo account mới theo template
     ├── Global customization bằng Terraform (VPC, GuardDuty...)
     └── Account-specific customization bằng Terraform

CfCT: Chịu trách nhiệm SCP tùy chỉnh + CloudFormation resources
     ├── Deploy custom SCPs lên OUs
     └── Deploy CloudFormation templates cần StackSets features

Không overlap — mỗi tool làm phần nó giỏi nhất
```

---

## Best Practices

### 1. Quản Lý Repository

```
✅ Dùng Git branching strategy (main + feature branches)
✅ Pull Request review trước khi merge vào main
✅ Bảo vệ branch main: require review + test pass
✅ Semantic versioning cho manifest
✅ Mô tả rõ mục đích mỗi template trong README
```

### 2. Testing Templates Trước Khi Deploy

```
Chiến lược test:
1. Policy Staging OU
   → Tạo một OU "Policy-Staging" không có workloads thật
   → Deploy customizations vào đây trước
   → Verify kết quả → merge vào main để deploy production

2. cfn-lint (CloudFormation Linter)
   → Validate syntax template trước khi push

3. cfn-nag (CloudFormation Security Scanner)
   → Kiểm tra security issues trong template

4. Dry run bằng CloudFormation Change Sets
   → Xem trước impact trước khi apply
```

### 3. Quản Lý SCP An Toàn

```
⚠️ SCP sai có thể lock out mọi người khỏi accounts!

Best practices:
1. Test SCP với Policy Staging OU TRƯỚC
2. Dùng "NotAction" thay "Action" cho deny list
   → Liệt kê những gì không chặn, chặn tất cả phần còn lại
   → Tránh vô tình chặn dịch vụ cần thiết
3. Luôn exclude Management Account khỏi restrictive SCPs
4. Có emergency plan: Management Account luôn có quyền gỡ SCP
5. Log và alert khi SCP thay đổi
```

### 4. Naming Convention

```yaml
# Tên resource CfCT nên:
resources:
  - name: security-hub-all-workloads        # clear, descriptive
  - name: deny-root-user-scp-root-ou        # indicate loại + scope
  - name: budget-alert-1000-usd-monthly     # include tham số quan trọng
  - name: vpc-setup-prod-ou                 # indicate target OU

# TRÁNH:
  - name: setup1                            # không rõ
  - name: my-template                       # không đủ thông tin
```

---

## Câu Hỏi Phỏng Vấn

### Q1: CfCT là gì và giải quyết vấn đề gì của Control Tower?

**Trả lời:** CfCT — Customizations for Control Tower là giải pháp mã nguồn mở của AWS (triển khai qua AWS Solution) cho phép mở rộng Control Tower Landing Zone với CloudFormation templates và SCPs tùy chỉnh. Control Tower chỉ cung cấp guardrails và baseline chuẩn; CfCT cho phép tổ chức thêm tài nguyên tùy chỉnh (Security Hub với standard riêng, budget alerts, custom IAM roles, custom VPC setup) và SCPs theo policy riêng của tổ chức — tất cả tự động áp dụng khi account mới được enroll.

---

### Q2: CfCT khác gì với CloudFormation StackSets thông thường?

**Trả lời:** StackSets là cơ chế deploy; CfCT là orchestration layer trên StackSets:
- **StackSets:** Deploy template vào danh sách account/OU — nhưng không tự động biết khi account mới được tạo.
- **CfCT:** Lắng nghe Control Tower lifecycle events (EventBridge), tự động trigger pipeline khi account mới được enroll, quản lý toàn bộ qua Git repository, hỗ trợ cả CloudFormation lẫn SCPs trong cùng một manifest file.

---

### Q3: Khi nào nên chọn CfCT thay vì AFT?

**Trả lời:**
- Chọn **CfCT** khi: team dùng CloudFormation (không dùng Terraform), cần deploy SCP tùy chỉnh (AFT không hỗ trợ native), cần deploy resources vào OUs toàn bộ (không phải từng account), tổ chức đã có nhiều account và cần standardize chúng.
- Chọn **AFT** khi: team dùng Terraform, cần customization phức tạp per-account, muốn GitOps workflow đầy đủ với git history cho mọi account.
- Trong thực tế, nhiều tổ chức lớn dùng **kết hợp**: AFT tạo account + baseline, CfCT deploy SCPs và CloudFormation resources vào OUs.

---

### Q4: Nếu manifest.yaml có lỗi và pipeline fail, accounts hiện tại có bị ảnh hưởng không?

**Trả lời:** Không — CfCT pipeline là idempotent (bất biến khi chạy lại) và atomic theo từng resource:
- Pipeline fail ở bước nào → dừng tại đó, không rollback resources đã deploy trước đó.
- Accounts hiện có đang chạy không bị ảnh hưởng — chỉ changes mới không được apply.
- Fix lỗi trong manifest → push lại → pipeline chạy lại từ đầu (idempotent — resources đã đúng rồi thì skip).
- Account mới enroll trong thời gian pipeline fail → khi pipeline pass lần sau sẽ nhận customizations.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Phiên Bản:** 1.0
**Trạng Thái:** ✅ Hoàn thành

← [3-account-factory.md](3-account-factory.md) | [../08-trusted-advisor/](../08-trusted-advisor/)
