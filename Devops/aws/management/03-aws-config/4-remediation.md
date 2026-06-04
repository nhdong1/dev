# Remediation Actions — Tự Động Khắc Phục Vi Phạm

> **Remediation Actions** (Hành Động Khắc Phục) là khả năng của AWS Config để tự động hoặc thủ công sửa chữa tài nguyên bị đánh giá là **NON_COMPLIANT** (Không Tuân Thủ). Bằng cách tích hợp với **SSM Automation** (AWS Systems Manager Automation — Tự Động Hóa Vận Hành), Config tạo thành vòng lặp compliance khép kín: phát hiện → thông báo → khắc phục.

---

## 📚 Mục Lục

1. [Remediation Là Gì?](#remediation-là-gì)
2. [Kiến Trúc Remediation](#kiến-trúc-remediation)
3. [Manual vs Automatic Remediation](#manual-vs-automatic-remediation)
4. [SSM Automation Documents](#ssm-automation-documents)
5. [Cấu Hình Remediation Action](#cấu-hình-remediation-action)
6. [Retry Logic — Thử Lại Khi Thất Bại](#retry-logic--thử-lại-khi-thất-bại)
7. [Remediation Thông Qua EventBridge + Lambda](#remediation-thông-qua-eventbridge--lambda)
8. [Ví Dụ Thực Tế Theo Kịch Bản](#ví-dụ-thực-tế-theo-kịch-bản)
9. [IAM Permissions Cần Thiết](#iam-permissions-cần-thiết)
10. [Monitoring Remediation](#monitoring-remediation)
11. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
12. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Remediation Là Gì?

### Vấn Đề Cần Giải Quyết

```
Không có Remediation:
  Developer tạo S3 bucket public → NON_COMPLIANT → Alert → Manual Fix → 2 giờ sau mới sửa
  Trong 2 giờ đó: Dữ liệu bị lộ

Với Automatic Remediation:
  Developer tạo S3 bucket public → NON_COMPLIANT → Tự động tắt public access → <2 phút
  Thiệt hại: Gần như bằng 0
```

### Hai Loại Remediation

```
Remediation Actions
├── Manual Remediation
│   └── Người dùng kích hoạt thủ công trên console hoặc CLI
│       Khi nào dùng: Cần review trước khi sửa, thay đổi có rủi ro cao
│
└── Automatic Remediation
    └── AWS Config tự kích hoạt khi phát hiện NON_COMPLIANT
        Khi nào dùng: Tác động thấp, cần phản hồi nhanh, quy trình rõ ràng
```

---

## Kiến Trúc Remediation

### Luồng Automatic Remediation

```
1. Tài nguyên thay đổi (VD: S3 bucket bật public access)
             ↓
2. Configuration Recorder ghi Configuration Item mới
             ↓
3. Config Rule "s3-bucket-public-read-prohibited" evaluate
             ↓
4. Kết quả: NON_COMPLIANT ❌
             ↓
5. Config kiểm tra: Có Remediation Action được cấu hình không?
             ↓ Có
6. Config invoke SSM Automation Document
             ↓
7. SSM Automation chạy steps (VD: gọi S3 PutPublicAccessBlock API)
             ↓
8. Tài nguyên được sửa: Public access = OFF
             ↓
9. Config Rule evaluate lại → COMPLIANT ✅
             ↓
10. EventBridge sự kiện: ComplianceChangeNotification (COMPLIANT)
```

### Luồng Với EventBridge (Linh Hoạt Hơn)

```
Config NON_COMPLIANT event
             ↓
EventBridge Rule (lọc theo resource type, rule name)
             ↓
Target: Lambda Function / Step Functions / SNS / SQS
             ↓
Lambda: Gửi Slack alert + Tạo Jira ticket + Gọi SSM
```

---

## Manual vs Automatic Remediation

### Manual Remediation

Người dùng chủ động kích hoạt từ console hoặc CLI:

```bash
# Kích hoạt remediation thủ công cho resource cụ thể
aws configservice start-remediation-execution \
  --config-rule-name "s3-bucket-public-read-prohibited" \
  --resource-keys '[
    {
      "resourceType": "AWS::S3::Bucket",
      "resourceId": "my-leaky-bucket"
    }
  ]'

# Kiểm tra trạng thái remediation
aws configservice describe-remediation-execution-status \
  --config-rule-name "s3-bucket-public-read-prohibited" \
  --resource-keys '[
    {
      "resourceType": "AWS::S3::Bucket",
      "resourceId": "my-leaky-bucket"
    }
  ]'
```

**Dùng Manual Remediation khi:**
- Sửa lỗi có thể gây downtime (VD: restart service)
- Cần approval trước khi thực thi
- Muốn review từng case trước khi sửa hàng loạt
- Đang trong giai đoạn pilot, chưa tin tưởng automation

### Automatic Remediation

Config tự kích hoạt không cần can thiệp người dùng:

```yaml
# CloudFormation: Cấu hình Automatic Remediation
RemediationConfiguration:
  Type: AWS::Config::RemediationConfiguration
  Properties:
    ConfigRuleName: !Ref S3PublicReadRule
    Automatic: true                          # Bật tự động
    MaximumAutomaticAttempts: 3             # Thử tối đa 3 lần
    RetryAttemptSeconds: 60                  # Chờ 60s giữa mỗi lần thử
    TargetType: SSM_DOCUMENT
    TargetId: AWS-DisableS3BucketPublicReadWrite
    TargetVersion: '1'
    Parameters:
      S3BucketName:
        ResourceValue:
          Value: RESOURCE_ID              # Tự điền tên bucket từ evaluation
    ExecutionControls:
      SsmControls:
        ConcurrentExecutionRatePercentage: 25
        ErrorPercentage: 10
```

**Dùng Automatic Remediation khi:**
- Hành động rõ ràng, không gây side effect nguy hiểm
- Cần phản hồi nhanh (bảo mật)
- Tài nguyên ảnh hưởng ít người dùng
- Đã test kỹ lưỡng trong môi trường staging

---

## SSM Automation Documents

**SSM Automation Documents** (còn gọi là **Runbooks** — Sổ Chạy Lệnh) là các tài liệu định nghĩa các bước thực thi để sửa lỗi.

### AWS Managed Automation Documents Phổ Biến

| Document Name | Mô Tả | Config Rule Phù Hợp |
|--------------|-------|---------------------|
| `AWS-DisableS3BucketPublicReadWrite` | Tắt public read/write trên S3 bucket | `s3-bucket-public-read-prohibited` |
| `AWS-EnableS3BucketEncryption` | Bật SSE trên S3 bucket | `s3-bucket-server-side-encryption-enabled` |
| `AWS-PublishSNSNotification` | Gửi SNS notification | Mọi rule |
| `AWS-EnableVpcFlowLogs` | Bật VPC Flow Logs | `vpc-flow-logs-enabled` |
| `AWS-ModifyRDSSnapshotAttribute` | Thay đổi snapshot visibility | RDS compliance rules |
| `AWS-TerminateEC2Instance` | Terminate EC2 instance (dùng cẩn thận!) | Policy violations nghiêm trọng |
| `AWS-CreateSnapshot` | Tạo EBS snapshot | `db-instance-backup-enabled` |
| `AWS-EnableCloudTrail` | Bật CloudTrail | `cloudtrail-enabled` |
| `AWS-RevokeUserPermissions` | Thu hồi IAM permissions | IAM policy violations |

### Tìm Documents Có Sẵn

```bash
# Liệt kê tất cả AWS managed automation documents
aws ssm list-documents \
  --filters '[
    {"Key": "DocumentType", "Values": ["Automation"]},
    {"Key": "Owner", "Values": ["Amazon"]}
  ]' \
  --query 'DocumentIdentifiers[].{Name:Name, Description:Description}' \
  --output table

# Xem chi tiết document
aws ssm describe-document \
  --name "AWS-DisableS3BucketPublicReadWrite"
```

### Tạo Custom SSM Automation Document

```yaml
# custom-tag-ec2-instance.yaml
description: "Thêm tag 'NonCompliant=true' vào EC2 instance vi phạm"
schemaVersion: '0.3'
assumeRole: "{{ AutomationAssumeRole }}"

parameters:
  InstanceId:
    type: String
    description: "ID của EC2 instance cần tag"
  AutomationAssumeRole:
    type: String
    description: "ARN của IAM role để Automation assume"

mainSteps:
  - name: TagNonCompliantInstance
    action: aws:createTags
    inputs:
      ResourceType: EC2
      ResourceIds:
        - "{{ InstanceId }}"
      Tags:
        - Key: NonCompliant
          Value: "true"
        - Key: RemediationRequired
          Value: "true"
        - Key: DetectedAt
          Value: "{{ global:DATE }}"

  - name: NotifySecurityTeam
    action: aws:executeAwsApi
    inputs:
      Service: sns
      Api: Publish
      TopicArn: "arn:aws:sns:ap-southeast-1:123456789012:security-alerts"
      Message: >
        EC2 instance {{ InstanceId }} đã được đánh dấu NON_COMPLIANT.
        Cần xem xét và khắc phục.
      Subject: "[CONFIG ALERT] EC2 Instance Vi Phạm Policy"
```

```bash
# Upload và đăng ký custom document
aws ssm create-document \
  --name "Custom-TagNonCompliantEC2" \
  --document-type "Automation" \
  --document-format "YAML" \
  --content file://custom-tag-ec2-instance.yaml
```

---

## Cấu Hình Remediation Action

### Qua AWS Console

```
Config → Rules → Chọn rule → Actions → Manage remediation
  → Configure remediation action
    → SSM Document: [chọn document]
    → Resource ID parameter: RESOURCE_ID (tự điền)
    → Automatic execution: Enable/Disable
    → Max automatic attempts: 3
    → Retry interval: 60 seconds
```

### Qua CloudFormation

```yaml
Resources:
  # Config Rule
  S3PublicReadRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED

  # Remediation Configuration
  S3PublicReadRemediation:
    Type: AWS::Config::RemediationConfiguration
    Properties:
      ConfigRuleName: !Ref S3PublicReadRule
      Automatic: true
      MaximumAutomaticAttempts: 5
      RetryAttemptSeconds: 60
      TargetType: SSM_DOCUMENT
      TargetId: AWS-DisableS3BucketPublicReadWrite
      TargetVersion: '1'
      Parameters:
        S3BucketName:
          ResourceValue:
            Value: RESOURCE_ID
        AutomationAssumeRole:
          StaticValue:
            Values:
              - !GetAtt RemediationRole.Arn
      ExecutionControls:
        SsmControls:
          ConcurrentExecutionRatePercentage: 25  # Tối đa 25% resources cùng lúc
          ErrorPercentage: 20                     # Dừng nếu 20% thất bại

  # IAM Role Cho Remediation
  RemediationRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ConfigRemediationRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ssm.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: RemediationPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:PutBucketPublicAccessBlock
                  - s3:GetBucketPublicAccessBlock
                Resource: '*'
```

### Tham Số Trong Remediation Configuration

| Tham Số | Loại | Mô Tả |
|--------|------|-------|
| `RESOURCE_ID` | ResourceValue | ID của tài nguyên NON_COMPLIANT (tự điền) |
| `RESOURCE_TYPE` | ResourceValue | Loại tài nguyên (VD: AWS::S3::Bucket) |
| `StaticValue` | StaticValue | Giá trị cố định (VD: ARN của SNS topic) |

---

## Retry Logic — Thử Lại Khi Thất Bại

### Cấu Hình Retry

```yaml
Automatic: true
MaximumAutomaticAttempts: 5      # Tổng số lần thử (bao gồm lần đầu)
RetryAttemptSeconds: 300         # Chờ 5 phút giữa các lần thử
```

### Vòng Đời Retry

```
Lần 1: Tự động kích hoạt remediation
  → Thất bại (VD: throttling, permission error)
  → Chờ RetryAttemptSeconds (300s)

Lần 2: Thử lại tự động
  → Thất bại lần 2
  → Chờ 300s

...

Lần 5: Thử lại lần cuối
  → Thất bại → Dừng, ghi log lỗi
  → Tài nguyên vẫn NON_COMPLIANT
  → Cần manual intervention
```

### Khi Nào Remediation Thất Bại?

| Nguyên Nhân | Dấu Hiệu | Giải Pháp |
|-------------|---------|----------|
| IAM permission thiếu | `AccessDenied` trong SSM logs | Cập nhật IAM policy cho Remediation Role |
| SSM Document parameters sai | `InvalidParameters` | Kiểm tra lại parameter mapping |
| Tài nguyên đã bị xóa | `ResourceNotFound` | Bình thường, bỏ qua |
| API throttling | `ThrottlingException` | Tăng `RetryAttemptSeconds` |
| Custom document logic lỗi | Tùy error | Debug qua SSM Automation execution history |

---

## Remediation Thông Qua EventBridge + Lambda

Khi cần logic phức tạp hơn SSM Documents, dùng EventBridge để trigger Lambda:

### Kiến Trúc EventBridge + Lambda Remediation

```
Config NON_COMPLIANT event
         ↓
EventBridge Rule (filter by rule name + resource type)
         ↓
Lambda Function
         ├── Gửi alert: Slack, PagerDuty, Email
         ├── Tạo ticket: Jira, ServiceNow
         ├── Log chi tiết: CloudWatch, S3
         └── Gọi SSM/API để sửa (nếu cần)
```

### EventBridge Rule Pattern

```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "messageType": ["ComplianceChangeNotification"],
    "newEvaluationResult": {
      "complianceType": ["NON_COMPLIANT"]
    },
    "configRuleName": [
      "s3-bucket-public-read-prohibited",
      "encrypted-volumes",
      "mfa-enabled-for-iam-console-access"
    ]
  }
}
```

### Lambda Handler Ví Dụ

```python
import json
import boto3
import os

ssm_client = boto3.client('ssm')
sns_client = boto3.client('sns')


def lambda_handler(event, context):
    """
    Xử lý Config NON_COMPLIANT event và thực hiện remediation
    """
    detail = event['detail']
    rule_name = detail['configRuleName']
    resource_type = detail['resourceType']
    resource_id = detail['resourceId']
    compliance_type = detail['newEvaluationResult']['complianceType']

    print(f"[CONFIG] Rule: {rule_name} | Resource: {resource_id} | Status: {compliance_type}")

    if compliance_type != 'NON_COMPLIANT':
        return

    # Định tuyến đến handler phù hợp
    handlers = {
        's3-bucket-public-read-prohibited': handle_s3_public_access,
        'encrypted-volumes': handle_unencrypted_ebs,
        'mfa-enabled-for-iam-console-access': handle_mfa_missing,
    }

    handler = handlers.get(rule_name)
    if handler:
        handler(resource_id, resource_type)
    else:
        # Rule không có automatic handler → chỉ gửi alert
        send_alert(rule_name, resource_id, resource_type)


def handle_s3_public_access(bucket_name, resource_type):
    """Tắt public access cho S3 bucket"""
    s3_client = boto3.client('s3')
    try:
        s3_client.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        print(f"[REMEDIATED] S3 bucket {bucket_name}: Public access đã tắt")
        send_success_notification(bucket_name, 's3-bucket-public-read-prohibited')
    except Exception as e:
        print(f"[ERROR] Không thể sửa {bucket_name}: {e}")
        send_alert('s3-bucket-public-read-prohibited', bucket_name, resource_type, error=str(e))


def handle_unencrypted_ebs(volume_id, resource_type):
    """Gửi alert cho EBS volume không mã hóa (không thể sửa tự động — cần tạo lại)"""
    message = f"""
    EBS Volume {volume_id} KHÔNG được mã hóa.
    Hành động cần thiết (thủ công):
    1. Tạo snapshot từ volume hiện tại
    2. Copy snapshot với encryption bật
    3. Tạo volume mới từ encrypted snapshot
    4. Detach volume cũ, attach volume mới
    5. Xóa volume cũ (sau khi verify)
    """
    send_alert('encrypted-volumes', volume_id, resource_type, custom_message=message)


def handle_mfa_missing(username, resource_type):
    """Thông báo và tạm thời vô hiệu hóa console access của user không có MFA"""
    iam_client = boto3.client('iam')
    try:
        # Liệt kê login profiles (console access)
        iam_client.get_login_profile(UserName=username)
        # Gửi alert khẩn cấp
        send_alert('mfa-enabled-for-iam-console-access', username, resource_type,
                   custom_message=f"IAM User {username} có console access nhưng CHƯA bật MFA!")
    except iam_client.exceptions.NoSuchEntityException:
        pass  # User không có console access → không cần xử lý


def send_alert(rule_name, resource_id, resource_type, error=None, custom_message=None):
    """Gửi SNS alert"""
    topic_arn = os.environ.get('ALERT_TOPIC_ARN')
    message = custom_message or f"NON_COMPLIANT: {resource_type} {resource_id} vi phạm rule {rule_name}"
    if error:
        message += f"\nLỗi khi remediate: {error}"

    sns_client.publish(
        TopicArn=topic_arn,
        Subject=f"[AWS Config] NON_COMPLIANT: {rule_name}",
        Message=message
    )


def send_success_notification(resource_id, rule_name):
    """Gửi thông báo remediation thành công"""
    topic_arn = os.environ.get('ALERT_TOPIC_ARN')
    sns_client.publish(
        TopicArn=topic_arn,
        Subject=f"[AWS Config] REMEDIATED: {rule_name}",
        Message=f"Đã tự động khắc phục: {resource_id} — {rule_name}"
    )
```

---

## Ví Dụ Thực Tế Theo Kịch Bản

### Kịch Bản 1: S3 Bucket Bị Mở Public Access

```
Rule: s3-bucket-public-read-prohibited
SSM Document: AWS-DisableS3BucketPublicReadWrite
Thời gian phát hiện → sửa: < 2 phút
Rủi ro remediation: Thấp (chỉ tắt public access, không xóa dữ liệu)
Khuyến nghị: Automatic Remediation ✅
```

### Kịch Bản 2: EC2 Instance Không Có Tag Bắt Buộc

```
Rule: required-tags
SSM Document: Custom — thêm default tag + gửi alert
Thời gian phát hiện → sửa: < 5 phút
Rủi ro remediation: Thấp (chỉ thêm tag)
Khuyến nghị: Automatic Remediation ✅ (với default tag values)
```

### Kịch Bản 3: IAM User Không Có MFA

```
Rule: mfa-enabled-for-iam-console-access
SSM Document: Không có (không thể tự bật MFA thay user)
Hành động: Gửi alert → Người dùng tự bật MFA
Thời gian: Phụ thuộc user action
Khuyến nghị: Manual + Alert qua EventBridge + Lambda ⚠️
Extreme case: Vô hiệu hóa console access nếu quá 24h không comply
```

### Kịch Bản 4: RDS Multi-AZ Bị Tắt

```
Rule: rds-multi-az-support
SSM Document: Custom — Gọi ModifyDBInstance API bật Multi-AZ
Thời gian: 10-30 phút (RDS cần thời gian để provision)
Rủi ro remediation: Trung bình (có thể gây brief connection interruption)
Khuyến nghị: Manual Remediation ⚠️ + Alert ngay lập tức
```

### Kịch Bản 5: Security Group Mở Port SSH từ 0.0.0.0/0

```
Rule: restricted-ssh
SSM Document: AWS-DisablePublicAccessForSecurityGroup
Thời gian phát hiện → sửa: < 2 phút
Rủi ro remediation: Thấp (chỉ revoke rule, có thể thêm lại rule đúng sau)
Lưu ý: Cần thêm specific IP trước khi xóa nếu không bị lock out
Khuyến nghị: Automatic + Alert đồng thời ✅
```

---

## IAM Permissions Cần Thiết

### IAM Role Cho Config Remediation (Phạm Vi Rộng)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSMStartAutomation",
      "Effect": "Allow",
      "Action": [
        "ssm:StartAutomationExecution",
        "ssm:GetAutomationExecution"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowResourceRemediation",
      "Effect": "Allow",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:GetBucketPublicAccessBlock",
        "s3:PutEncryptionConfiguration",
        "ec2:CreateTags",
        "ec2:DescribeSecurityGroups",
        "ec2:RevokeSecurityGroupIngress",
        "rds:ModifyDBInstance",
        "iam:CreateVirtualMFADevice"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowSNSPublish",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:*:*:security-alerts"
    }
  ]
}
```

### Trust Policy — Cho Phép SSM Assume Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ssm.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "123456789012"
        }
      }
    }
  ]
}
```

---

## Monitoring Remediation

### Xem Trạng Thái Remediation

```bash
# Xem tất cả remediation executions của một rule
aws configservice describe-remediation-execution-status \
  --config-rule-name "s3-bucket-public-read-prohibited"

# Kết quả
{
  "RemediationExecutionStatuses": [
    {
      "ResourceKey": {
        "ResourceType": "AWS::S3::Bucket",
        "ResourceId": "my-bucket-name"
      },
      "State": "SUCCEEDED",
      "StepDetails": [
        {
          "Name": "DisablePublicAccess",
          "State": "SUCCEEDED",
          "StartTime": "2026-05-17T10:30:00Z",
          "StopTime": "2026-05-17T10:30:05Z"
        }
      ]
    }
  ]
}
```

### CloudWatch Metrics Cho Remediation

```bash
# Tạo CloudWatch Alarm cho remediation failures
aws cloudwatch put-metric-alarm \
  --alarm-name "ConfigRemediationFailures" \
  --alarm-description "Cảnh báo khi Config Remediation thất bại" \
  --metric-name "RemediationFailures" \
  --namespace "AWS/Config" \
  --statistic "Sum" \
  --period 300 \
  --threshold 1 \
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:ap-southeast-1:123456789012:ops-alerts"
```

### Dashboard Theo Dõi

```json
{
  "widgets": [
    {
      "type": "metric",
      "title": "Config Compliance Overview",
      "properties": {
        "metrics": [
          ["AWS/Config", "CompliantRulesCount"],
          ["AWS/Config", "NonCompliantRulesCount"],
          ["AWS/Config", "TotalRemediations"],
          ["AWS/Config", "SuccessfulRemediations"],
          ["AWS/Config", "FailedRemediations"]
        ],
        "period": 3600
      }
    }
  ]
}
```

---

## Thực Hành Tốt Nhất

### 1. Nguyên Tắc Least Risk (Rủi Ro Thấp Nhất)

```
Automatic OK khi:
  ✅ Hành động có thể đảo ngược (reversible)
  ✅ Không gây gián đoạn dịch vụ
  ✅ Không xóa dữ liệu
  ✅ Không thay đổi network connectivity
  ✅ Đã test trong staging

Manual khi:
  ⚠️ Gây downtime (VD: bật Multi-AZ RDS)
  ⚠️ Xóa tài nguyên (VD: terminate instance)
  ⚠️ Thay đổi networking có thể lock out
  ⚠️ Production resource quan trọng
```

### 2. Luôn Gửi Thông Báo Song Song Với Automatic Remediation

```
Automatic remediation kích hoạt
  ├── SSM Automation sửa tài nguyên
  └── SNS/Slack gửi alert: "Đã tự động sửa bucket X — [details]"
```

Nhóm security cần biết ngay cả khi đã được sửa tự động — đây là signal cần review.

### 3. Test Remediation Trong Môi Trường Non-Production Trước

```bash
# Tạo S3 bucket test và bật public access
aws s3api create-bucket --bucket test-remediation-staging
aws s3api put-bucket-acl --bucket test-remediation-staging --acl public-read

# Chờ Config phát hiện và remediate
# Verify sau 2-5 phút
aws s3api get-bucket-acl --bucket test-remediation-staging
```

### 4. Đặt Execution Controls Hợp Lý

```yaml
ExecutionControls:
  SsmControls:
    ConcurrentExecutionRatePercentage: 25  # Không sửa > 25% resources cùng lúc
    ErrorPercentage: 10                    # Dừng toàn bộ nếu >10% thất bại
```

Tránh tình huống: 1000 buckets bị sửa cùng lúc → API throttling → Tất cả thất bại.

### 5. Document Hành Động Remediation

Mỗi rule nên có tài liệu:

```markdown
## Rule: s3-bucket-public-read-prohibited

**Tác động khi NON_COMPLIANT:** Dữ liệu S3 có thể bị truy cập công khai
**Remediation:** Automatic — AWS-DisableS3BucketPublicReadWrite
**Tác động remediation:** Tắt public access, không mất dữ liệu
**Thông báo:** SNS → security-alerts, Slack #security-channel
**Escalation:** Nếu remediation thất bại 3 lần → PagerDuty
**Approved by:** Security Team Lead (2026-01-15)
```

---

## Câu Hỏi Phỏng Vấn

**Q: Thiết kế hệ thống tự động remediation an toàn cho production. Cần chú ý gì?**

A: Năm điểm chính:
1. **Least risk principle** — Chỉ automatic với hành động reversible, không gây downtime
2. **Notify luôn** — Gửi alert song song dù tự động hay thủ công; security team cần biết
3. **Execution controls** — Giới hạn concurrency và error threshold để tránh cascade failures
4. **Test trước** — Test remediation trong staging với tài nguyên giả trước khi bật production
5. **IAM least privilege** — Remediation role chỉ có quyền cần thiết, scope hẹp nhất có thể

**Q: SSM Automation Document là gì và tại sao dùng với Config?**

A: SSM Automation Document (Runbook) là tập hợp các bước thực thi được định nghĩa trước. Config dùng nó làm "tay thực thi" remediation: Config phát hiện NON_COMPLIANT → kích hoạt SSM Document → Document chạy các bước API call để sửa tài nguyên. AWS cung cấp sẵn hàng chục managed documents cho các tình huống phổ biến; chỉ cần viết custom document khi không có sẵn.

**Q: Có thể dùng Config Remediation để xóa tài nguyên không?**

A: Kỹ thuật là có thể (dùng SSM Document gọi `aws:deleteStack` hoặc terminate EC2). Nhưng về mặt thực hành: **không nên dùng Automatic Remediation để xóa tài nguyên production**. Xóa tài nguyên là hành động không thể đảo ngược, cần manual approval. Chỉ áp dụng automatic delete trong môi trường sandbox có chính sách rõ ràng (VD: EC2 không tag sau 1 giờ sẽ bị terminate tự động).

---

**Tiếp Theo:** [5-aggregator-multiregion.md](5-aggregator-multiregion.md) — Aggregator: Tổng Hợp Compliance Đa Account/Region
