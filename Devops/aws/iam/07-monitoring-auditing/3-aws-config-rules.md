# AWS Config Rules — Kiểm Soát Tuân Thủ Và Tự Động Remediation

> **AWS Config** theo dõi trạng thái cấu hình tài nguyên và đánh giá tuân thủ liên tục — giúp bạn biết ngay khi nào cấu hình vi phạm chính sách bảo mật.

---

## 🎯 AWS Config Là Gì?

**AWS Config** là dịch vụ đánh giá, kiểm toán và theo dõi cấu hình tài nguyên AWS. Nó trả lời hai câu hỏi cốt lõi:

```
"Cấu hình tài nguyên này hiện tại là gì?"
"Cấu hình đã thay đổi như thế nào theo thời gian?"
```

### Phân Biệt AWS Config Và CloudTrail

| | **CloudTrail** | **AWS Config** |
|---|---|---|
| **Theo dõi** | Hành động (Actions / API calls) | Trạng thái cấu hình (Configuration state) |
| **Câu hỏi trả lời** | Ai làm gì? Khi nào? | Cấu hình đang là gì? Có tuân thủ không? |
| **Dữ liệu lưu** | Event log | Configuration snapshots |
| **Dùng cho** | Audit trail, forensics | Compliance, drift detection |
| **Ví dụ** | "Alice đã xóa S3 bucket lúc 9h" | "S3 bucket này có versioning không?" |

### Các Thành Phần Cốt Lõi

```
┌─────────────────────────────────────────────────────────┐
│                    AWS Config                            │
│                                                         │
│  ┌─────────────────┐    ┌──────────────────────────┐    │
│  │  Configuration  │    │      Config Rules        │    │
│  │    Recorder     │    │  (Quy Tắc Đánh Giá)     │    │
│  │  (Ghi trạng     │    │                          │    │
│  │   thái thay     │    │  ┌──────┐  ┌──────────┐  │    │
│  │   đổi)          │    │  │Mngd  │  │  Custom  │  │    │
│  └────────┬────────┘    │  │Rules │  │  Rules   │  │    │
│           │             │  └──────┘  └──────────┘  │    │
│           ▼             └──────────────┬─────────────┘   │
│  ┌─────────────────┐                  │                  │
│  │   Config S3     │       ┌───────────▼──────────┐      │
│  │  (lưu snapshots │       │     Remediation      │      │
│  │   và history)   │       │    (Tự Động Sửa)    │      │
│  └─────────────────┘       └──────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

---

## ⚙️ Thiết Lập AWS Config

### Bật Config Recorder

```bash
# Tạo S3 bucket để lưu Config snapshots
aws s3api create-bucket \
  --bucket config-history-123456789012 \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Tạo IAM role cho Config
aws iam create-role \
  --role-name AWSConfigRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "config.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach AWS managed policy
aws iam attach-role-policy \
  --role-name AWSConfigRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWS_ConfigRole

# Bật Config Recorder
aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::123456789012:role/AWSConfigRole",
    "recordingGroup": {
      "allSupported": true,
      "includeGlobalResourceTypes": true
    }
  }'

# Cấu hình delivery channel (nơi lưu snapshots)
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "config-history-123456789012",
    "snsTopicARN": "arn:aws:sns:ap-southeast-1:123456789012:ConfigAlerts",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }'

# Bật recorder
aws configservice start-configuration-recorder \
  --configuration-recorder-name default
```

### Terraform Setup

```hcl
resource "aws_config_configuration_recorder" "main" {
  name     = "main"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "main"
  s3_bucket_name = aws_s3_bucket.config_history.bucket
  sns_topic_arn  = aws_sns_topic.config_alerts.arn

  snapshot_delivery_properties {
    delivery_frequency = "TwentyFour_Hours"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true

  depends_on = [aws_config_delivery_channel.main]
}
```

---

## 📋 Managed Rules — Quy Tắc Được AWS Quản Lý

**Managed Rules** (Quy Tắc Được Quản Lý) là các quy tắc đánh giá được AWS xây dựng sẵn. Hiện có hơn 180 managed rules.

### Nhóm Quy Tắc Bảo Mật Quan Trọng Nhất

#### Nhóm IAM

| Rule Name | Mô tả | Trigger |
|---|---|---|
| `iam-root-access-key-check` | Root account không có access key | Định kỳ |
| `iam-user-mfa-enabled` | IAM users có MFA | Định kỳ |
| `iam-password-policy` | Password policy đủ mạnh | Định kỳ |
| `access-keys-rotated` | Access keys được xoay vòng < 90 ngày | Định kỳ |
| `iam-no-inline-policy-check` | Không dùng inline policies | Thay đổi |
| `iam-user-no-policies-check` | User không attach policy trực tiếp (phải qua group) | Thay đổi |
| `iam-policy-no-statements-with-admin-access` | Không có policy cho `*:*` | Thay đổi |

#### Nhóm S3

| Rule Name | Mô tả | Trigger |
|---|---|---|
| `s3-bucket-public-read-prohibited` | S3 bucket không public read | Thay đổi |
| `s3-bucket-public-write-prohibited` | S3 bucket không public write | Thay đổi |
| `s3-bucket-ssl-requests-only` | Chỉ cho phép HTTPS | Thay đổi |
| `s3-bucket-logging-enabled` | Access logging bật | Thay đổi |
| `s3-bucket-versioning-enabled` | Versioning bật | Thay đổi |
| `s3-bucket-server-side-encryption-enabled` | Encryption bật | Thay đổi |

#### Nhóm EC2 và VPC

| Rule Name | Mô tả | Trigger |
|---|---|---|
| `ec2-instance-no-public-ip` | EC2 không có public IP | Thay đổi |
| `ec2-security-group-attached-to-eni` | Security Groups được dùng | Thay đổi |
| `restricted-ssh` | Port 22 không mở với 0.0.0.0/0 | Thay đổi |
| `restricted-common-ports` | Ports nhạy cảm không mở public | Thay đổi |
| `vpc-flow-logs-enabled` | VPC Flow Logs bật | Định kỳ |
| `vpc-default-security-group-closed` | Default Security Group không có rules | Thay đổi |

#### Nhóm Encryption

| Rule Name | Mô tả | Trigger |
|---|---|---|
| `encrypted-volumes` | EBS volumes được mã hóa | Thay đổi |
| `rds-storage-encrypted` | RDS storage được mã hóa | Thay đổi |
| `kms-cmk-not-scheduled-for-deletion` | KMS keys không bị scheduled deletion | Định kỳ |
| `cloudtrail-kms-key-arn-check` | CloudTrail dùng KMS encryption | Định kỳ |
| `dynamodb-table-encrypted-at-rest` | DynamoDB encrypted | Thay đổi |

### Bật Managed Rules

```bash
# Bật rule kiểm tra S3 bucket public access
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }'

# Bật rule kiểm tra MFA cho IAM users
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "iam-user-mfa-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "IAM_USER_MFA_ENABLED"
    }
  }'
```

---

## 🔧 Custom Rules — Quy Tắc Tùy Chỉnh

Khi managed rules không đủ cho chính sách nội bộ, bạn tạo **Custom Rules** (Quy Tắc Tùy Chỉnh) bằng Lambda.

### Custom Rule: Kiểm Tra Tên Resource Theo Convention

```python
# Lambda function: check-naming-convention
import json
import boto3

config_client = boto3.client('config')

COMPLIANT = 'COMPLIANT'
NON_COMPLIANT = 'NON_COMPLIANT'
NOT_APPLICABLE = 'NOT_APPLICABLE'

def lambda_handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    configuration_item = invoking_event.get('configurationItem')
    
    if not configuration_item:
        return
    
    resource_type = configuration_item['resourceType']
    resource_id = configuration_item['resourceId']
    resource_name = configuration_item.get('resourceName', '')
    
    # Chỉ áp dụng cho EC2 instances
    if resource_type != 'AWS::EC2::Instance':
        compliance = NOT_APPLICABLE
    else:
        # Kiểm tra naming convention: phải có prefix env-
        environment = configuration_item.get('tags', {}).get('Environment', '')
        if not environment:
            compliance = NON_COMPLIANT
            annotation = f"EC2 instance thiếu tag 'Environment'"
        elif not resource_name.startswith(f"{environment.lower()}-"):
            compliance = NON_COMPLIANT
            annotation = f"Tên không theo convention: {environment.lower()}-*"
        else:
            compliance = COMPLIANT
            annotation = "Đúng naming convention"
    
    evaluation = {
        'ComplianceResourceType': resource_type,
        'ComplianceResourceId': resource_id,
        'ComplianceType': compliance,
        'Annotation': annotation if compliance != NOT_APPLICABLE else '',
        'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
    }
    
    config_client.put_evaluations(
        Evaluations=[evaluation],
        ResultToken=event['resultToken']
    )
```

```bash
# Đăng ký custom rule với Lambda
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "ec2-naming-convention",
    "Description": "EC2 instances phải theo naming convention env-*",
    "Source": {
      "Owner": "CUSTOM_LAMBDA",
      "SourceIdentifier": "arn:aws:lambda:ap-southeast-1:123456789012:function:check-naming-convention",
      "SourceDetails": [{
        "EventSource": "aws.config",
        "MessageType": "ConfigurationItemChangeNotification"
      }]
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Instance"]
    }
  }'
```

### Custom Rule: Kiểm Tra Tag Bắt Buộc

```python
# Lambda function: required-tags-check
import json
import boto3

REQUIRED_TAGS = ['Environment', 'Owner', 'CostCenter', 'Project']

def lambda_handler(event, context):
    invoking_event = json.loads(event['invokingEvent'])
    config_item = invoking_event.get('configurationItem')
    
    if not config_item:
        return
    
    tags = config_item.get('tags', {})
    missing_tags = [tag for tag in REQUIRED_TAGS if tag not in tags]
    
    if missing_tags:
        compliance = 'NON_COMPLIANT'
        annotation = f"Thiếu tags bắt buộc: {', '.join(missing_tags)}"
    else:
        compliance = 'COMPLIANT'
        annotation = "Đủ tất cả tags bắt buộc"
    
    boto3.client('config').put_evaluations(
        Evaluations=[{
            'ComplianceResourceType': config_item['resourceType'],
            'ComplianceResourceId': config_item['resourceId'],
            'ComplianceType': compliance,
            'Annotation': annotation,
            'OrderingTimestamp': config_item['configurationItemCaptureTime']
        }],
        ResultToken=event['resultToken']
    )
```

---

## 🔄 Auto-Remediation — Tự Động Khắc Phục Vi Phạm

**Auto-Remediation** (Tự Động Khắc Phục) cho phép Config tự động chạy SSM Automation document khi phát hiện vi phạm.

### Kiến Trúc Auto-Remediation

```
AWS Config phát hiện vi phạm
           │
           ▼
    Config Rule: NON_COMPLIANT
           │
           ▼
   Remediation Action
   (SSM Automation Document)
           │
           ▼
  ┌─────────────────────┐
  │  Auto (ngay lập tức)│  hoặc  │  Manual (chờ approve)│
  └─────────────────────┘         └──────────────────────┘
           │
           ▼
   Tài nguyên được sửa
           │
           ▼
   Config đánh giá lại: COMPLIANT
```

### Ví Dụ: Tự Động Tắt Public Access S3 Bucket

```bash
aws configservice put-remediation-configurations \
  --remediation-configurations '[{
    "ConfigRuleName": "s3-bucket-public-read-prohibited",
    "TargetType": "SSM_DOCUMENT",
    "TargetId": "AWS-DisableS3BucketPublicReadWrite",
    "Parameters": {
      "AutomationAssumeRole": {
        "StaticValue": {
          "Values": ["arn:aws:iam::123456789012:role/AutoRemediationRole"]
        }
      },
      "S3BucketName": {
        "ResourceValue": {
          "Value": "RESOURCE_ID"
        }
      }
    },
    "Automatic": false,    # Đặt true để tự động, false để manual approve
    "MaximumAutomaticAttempts": 3,
    "RetryAttemptSeconds": 60,
    "ExecutionControls": {
      "SsmControls": {
        "ConcurrentExecutionRatePercentage": 25,
        "ErrorPercentage": 20
      }
    }
  }]'
```

### Custom SSM Automation Document Cho Remediation

```yaml
# SSM Document: RevokeBroadSGRule
description: "Thu hồi Security Group rules cho phép 0.0.0.0/0 trên port nhạy cảm"
schemaVersion: '0.3'
assumeRole: "{{AutomationAssumeRole}}"
parameters:
  SecurityGroupId:
    type: String
    description: ID của Security Group cần sửa
  AutomationAssumeRole:
    type: String
    description: ARN của IAM role để thực thi

mainSteps:
  - name: GetSecurityGroupRules
    action: aws:executeAwsApi
    inputs:
      Service: ec2
      Api: DescribeSecurityGroupRules
      Filters:
        - Name: group-id
          Values:
            - "{{SecurityGroupId}}"
    outputs:
      - Name: SecurityGroupRules
        Selector: $.SecurityGroupRules
        Type: MapList

  - name: RevokePublicIngressRules
    action: aws:executeScript
    inputs:
      Runtime: python3.9
      Handler: revoke_public_rules
      InputPayload:
        SecurityGroupId: "{{SecurityGroupId}}"
        Rules: "{{GetSecurityGroupRules.SecurityGroupRules}}"
      Script: |
        import boto3
        
        def revoke_public_rules(event, context):
            ec2 = boto3.client('ec2')
            sg_id = event['SecurityGroupId']
            rules = event['Rules']
            
            sensitive_ports = [22, 3389, 1433, 3306, 5432, 6379, 27017]
            
            rules_to_revoke = []
            for rule in rules:
                if rule.get('IsEgress'):
                    continue
                cidr = rule.get('CidrIpv4', '')
                from_port = rule.get('FromPort', 0)
                to_port = rule.get('ToPort', 0)
                
                is_open_to_world = cidr in ['0.0.0.0/0', '::/0']
                hits_sensitive = any(
                    from_port <= p <= to_port
                    for p in sensitive_ports
                )
                
                if is_open_to_world and hits_sensitive:
                    rules_to_revoke.append(rule['SecurityGroupRuleId'])
            
            if rules_to_revoke:
                ec2.revoke_security_group_ingress(
                    GroupId=sg_id,
                    SecurityGroupRuleIds=rules_to_revoke
                )
                return {'RevokedRules': rules_to_revoke}
            
            return {'RevokedRules': []}
```

---

## 📦 Conformance Packs — Gói Tuân Thủ

**Conformance Packs** (Gói Tuân Thủ) là tập hợp Config rules và remediation actions đóng gói theo framework tuân thủ cụ thể.

### Các Conformance Pack Phổ Biến

| Pack | Framework | Rules |
|---|---|---|
| `AWS-Control-Tower-Detective-Guardrails` | Control Tower | ~30 rules |
| `CIS-AWS-Foundations-Benchmark-Level-1` | CIS Benchmark | ~30 rules |
| `CIS-AWS-Foundations-Benchmark-Level-2` | CIS Benchmark | ~50 rules |
| `Operational-Best-Practices-for-PCI-DSS` | PCI-DSS | ~60 rules |
| `Operational-Best-Practices-for-HIPAA-Security` | HIPAA | ~45 rules |
| `Operational-Best-Practices-for-NIST-CSF` | NIST CSF | ~60 rules |

### Triển Khai CIS AWS Foundations Benchmark

```bash
# Tạo Conformance Pack từ sample template
aws configservice put-conformance-pack \
  --conformance-pack-name "CISBenchmarkLevel1" \
  --template-s3-uri s3://aws-config-rules-packages-ap-southeast-1/conformancepack/CIS-AWS-Foundations-Benchmark-Level-1.yaml \
  --delivery-s3-bucket config-history-123456789012
```

### Tạo Custom Conformance Pack

```yaml
# custom-security-pack.yaml
Parameters:
  IAMPasswordMinLength:
    Type: String
    Default: "14"

Resources:
  # IAM Rules
  RootAccessKeyCheck:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-root-access-key-check
      Source:
        Owner: AWS
        SourceIdentifier: IAM_ROOT_ACCESS_KEY_CHECK

  MFAEnabledForRoot:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: root-account-mfa-enabled
      Source:
        Owner: AWS
        SourceIdentifier: ROOT_ACCOUNT_MFA_ENABLED

  PasswordPolicy:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: iam-password-policy
      Source:
        Owner: AWS
        SourceIdentifier: IAM_PASSWORD_POLICY
      InputParameters:
        MinimumPasswordLength: !Ref IAMPasswordMinLength
        RequireUppercaseCharacters: true
        RequireLowercaseCharacters: true
        RequireNumbers: true
        RequireSymbols: true
        MaxPasswordAge: 90
        PasswordReusePrevention: 10

  # S3 Rules
  S3PublicReadProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED

  S3EncryptionEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-default-encryption-kms
      Source:
        Owner: AWS
        SourceIdentifier: S3_DEFAULT_ENCRYPTION_KMS

  # Network Rules
  RestrictedSSH:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: restricted-ssh
      Source:
        Owner: AWS
        SourceIdentifier: INCOMING_SSH_DISABLED

  VPCFlowLogsEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: vpc-flow-logs-enabled
      Source:
        Owner: AWS
        SourceIdentifier: VPC_FLOW_LOGS_ENABLED
```

---

## 📊 Xem Kết Quả Đánh Giá Tuân Thủ

```bash
# Xem tổng quan compliance dashboard
aws configservice describe-compliance-by-config-rule \
  --compliance-types NON_COMPLIANT \
  --query 'ComplianceByConfigRules[*].{Rule:ConfigRuleName,Status:Compliance.ComplianceType}'

# Xem chi tiết tài nguyên vi phạm một rule
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name s3-bucket-public-read-prohibited \
  --compliance-types NON_COMPLIANT \
  --query 'EvaluationResults[*].{ResourceId:EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId}'

# Xem lịch sử cấu hình của một tài nguyên
aws configservice get-resource-config-history \
  --resource-type AWS::S3::Bucket \
  --resource-id my-bucket-name \
  --limit 10
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Config Rule trigger "Configuration change" vs "Periodic" khác nhau thế nào?**

> "Configuration change" (Thay đổi cấu hình) trigger đánh giá ngay khi tài nguyên thay đổi — phù hợp cho rules cần phát hiện ngay lập tức như S3 public access. "Periodic" (Định kỳ) đánh giá theo lịch (1h, 3h, 6h, 12h, 24h) — phù hợp cho rules liên quan đến trạng thái ngoài AWS Config như IAM password age hoặc access key rotation. Nhiều rules bảo mật quan trọng dùng cả hai trigger.

**Q: Khi nào nên dùng Auto Remediation vs Manual Remediation?**

> Auto Remediation tốt cho các vi phạm rõ ràng và không có rủi ro gián đoạn dịch vụ: xóa S3 public access, force-enable encryption. Manual Remediation tốt hơn cho những thay đổi có thể ảnh hưởng production: thu hồi Security Group rules (có thể ngắt kết nối), terminate EC2 instances. Nguyên tắc: nếu auto remediation có thể gây downtime, dùng manual với approval workflow.

**Q: Conformance Pack giải quyết vấn đề gì mà dùng riêng lẻ từng Config Rule không giải quyết được?**

> Conformance Pack đóng gói nhiều rules thành một đơn vị quản lý, deploy, và báo cáo. Lợi ích: (1) deploy một lần cho toàn organization thay vì deploy từng rule; (2) báo cáo compliance aggregate theo framework (ví dụ: "đạt 85% CIS Level 1"); (3) version control và audit trail cho policy changes; (4) rollback đồng loạt khi cần. Đặc biệt quan trọng khi cần chứng minh tuân thủ cho auditors.

---

**Tiếp Theo:** [4-access-analyzer.md](4-access-analyzer.md) — IAM Access Analyzer

---

**Cập Nhật Lần Cuối:** 2026-05-16
