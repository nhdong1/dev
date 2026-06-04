# Config Rules — Quy Tắc Đánh Giá Tuân Thủ

> **Config Rules** (Quy Tắc Config) định nghĩa trạng thái cấu hình "đúng" (desired configuration) của tài nguyên AWS, và liên tục đánh giá xem tài nguyên có **COMPLIANT** (Tuân Thủ) hay **NON_COMPLIANT** (Không Tuân Thủ) với quy tắc đó không.

---

## 📚 Mục Lục

1. [Config Rules Là Gì?](#config-rules-là-gì)
2. [AWS Managed Rules — Quy Tắc AWS Quản Lý](#aws-managed-rules--quy-tắc-aws-quản-lý)
3. [Custom Rules — Quy Tắc Tùy Chỉnh](#custom-rules--quy-tắc-tùy-chỉnh)
4. [Evaluation Scope — Phạm Vi Đánh Giá](#evaluation-scope--phạm-vi-đánh-giá)
5. [Evaluation Trigger — Thời Điểm Đánh Giá](#evaluation-trigger--thời-điểm-đánh-giá)
6. [Compliance States — Trạng Thái Tuân Thủ](#compliance-states--trạng-thái-tuân-thủ)
7. [Ví Dụ Rules Phổ Biến Theo Danh Mục](#ví-dụ-rules-phổ-biến-theo-danh-mục)
8. [Viết Custom Lambda Rule](#viết-custom-lambda-rule)
9. [CloudFormation Guard Rules](#cloudformation-guard-rules)
10. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Config Rules Là Gì?

### Định Nghĩa

Mỗi **Config Rule** mô tả trạng thái mong muốn (desired state) cho một loại tài nguyên:

```
Rule: "Tất cả S3 bucket phải tắt public access"
  ↓
AWS Config đánh giá mọi S3 bucket
  ↓
Bucket A — public access = OFF → COMPLIANT ✅
Bucket B — public access = ON  → NON_COMPLIANT ❌
```

### Hai Loại Rules

```
Config Rules
├── AWS Managed Rules     → AWS định nghĩa & maintain sẵn (>250 rules)
└── Custom Rules
    ├── Lambda-backed     → Function Lambda tùy viết
    └── Guard Rules       → CloudFormation Guard DSL (Domain Specific Language)
```

---

## AWS Managed Rules — Quy Tắc AWS Quản Lý

**AWS Managed Rules** (Quy Tắc AWS Quản Lý) là các rules được AWS tạo sẵn, kiểm thử và maintain. Không cần viết code, chỉ cần bật và cấu hình tham số.

### Cách Bật Managed Rule (AWS Console / CLI)

```bash
# Bật rule "s3-bucket-public-read-prohibited"
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
```

### Managed Rules Phổ Biến Theo Danh Mục

#### Bảo Mật (Security)

| Rule Name | Mô Tả | Trigger |
|-----------|-------|---------|
| `s3-bucket-public-read-prohibited` | S3 bucket không được phép public read | Thay đổi |
| `s3-bucket-public-write-prohibited` | S3 bucket không được phép public write | Thay đổi |
| `ec2-security-group-attached-to-eni` | Security group phải gắn với ENI (Elastic Network Interface) | Định kỳ |
| `iam-root-access-key-check` | Root account không được có access key | Định kỳ |
| `mfa-enabled-for-iam-console-access` | IAM user có console access phải bật MFA | Định kỳ |
| `restricted-ssh` | Security group không được mở SSH (port 22) từ 0.0.0.0/0 | Thay đổi |
| `access-keys-rotated` | Access key không được quá 90 ngày | Định kỳ |
| `secretsmanager-rotation-enabled-check` | Secrets trong Secrets Manager phải bật rotation | Thay đổi |

#### Mã Hóa (Encryption)

| Rule Name | Mô Tả | Trigger |
|-----------|-------|---------|
| `s3-bucket-server-side-encryption-enabled` | S3 bucket phải bật SSE (Server-Side Encryption) | Thay đổi |
| `encrypted-volumes` | EBS volume phải được mã hóa | Thay đổi |
| `rds-storage-encrypted` | RDS storage phải được mã hóa | Thay đổi |
| `cloudtrail-encryption-enabled` | CloudTrail log phải mã hóa bằng KMS | Định kỳ |
| `dynamodb-table-encrypted-at-rest` | DynamoDB table phải mã hóa at rest | Thay đổi |
| `efs-encrypted-check` | EFS file system phải mã hóa | Định kỳ |

#### Tính Sẵn Sàng Cao (High Availability)

| Rule Name | Mô Tả | Trigger |
|-----------|-------|---------|
| `rds-multi-az-support` | RDS phải bật Multi-AZ | Thay đổi |
| `elasticache-redis-cluster-automatic-backup-check` | Redis cluster phải bật backup tự động | Thay đổi |
| `db-instance-backup-enabled` | RDS phải bật automated backup | Thay đổi |
| `elb-cross-zone-load-balancing-enabled` | ELB (Elastic Load Balancer) phải bật cross-zone balancing | Thay đổi |

#### Logging & Audit

| Rule Name | Mô Tả | Trigger |
|-----------|-------|---------|
| `cloudtrail-enabled` | CloudTrail phải được bật | Định kỳ |
| `multi-region-cloudtrail-enabled` | Phải có ít nhất một multi-region trail | Định kỳ |
| `vpc-flow-logs-enabled` | VPC Flow Logs phải được bật | Định kỳ |
| `s3-bucket-logging-enabled` | S3 bucket phải bật access logging | Thay đổi |
| `cloud-trail-log-file-validation-enabled` | CloudTrail log file validation phải bật | Định kỳ |

#### Tagging (Gắn Nhãn)

| Rule Name | Mô Tả | Trigger |
|-----------|-------|---------|
| `required-tags` | Tài nguyên phải có tags bắt buộc | Thay đổi |
| `ec2-instance-detailed-monitoring-enabled` | EC2 phải bật detailed monitoring | Thay đổi |

### Tham Số (Parameters) Của Managed Rules

Một số rules cần tham số cấu hình:

```bash
# Rule "access-keys-rotated" với tham số maxAccessKeyAge
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "access-keys-rotated",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "ACCESS_KEYS_ROTATED"
    },
    "InputParameters": "{\"maxAccessKeyAge\": \"90\"}"
  }'
```

```bash
# Rule "required-tags" với các tag bắt buộc
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags-ec2",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Instance"]
    },
    "InputParameters": "{\"tag1Key\": \"Environment\", \"tag2Key\": \"Team\", \"tag3Key\": \"CostCenter\"}"
  }'
```

---

## Custom Rules — Quy Tắc Tùy Chỉnh

Dùng khi AWS Managed Rules không đủ cho logic nghiệp vụ phức tạp.

### Khi Nào Cần Custom Rules?

| Tình Huống | Giải Pháp |
|-----------|----------|
| Kiểm tra cross-resource (EC2 tag phải match danh sách trong DynamoDB) | Custom Lambda |
| Logic kinh doanh riêng (instance type phải theo danh sách approved) | Custom Lambda hoặc Guard |
| Kiểm tra bên thứ ba (tài nguyên phải đăng ký với CMDB nội bộ) | Custom Lambda |
| Rules đơn giản trên CloudFormation template syntax | Guard Rules |

---

## Evaluation Scope — Phạm Vi Đánh Giá

**Evaluation Scope** (Phạm Vi Đánh Giá) xác định *tài nguyên nào* sẽ bị đánh giá bởi một rule.

### Scope Theo Resource Types

```json
"Scope": {
  "ComplianceResourceTypes": [
    "AWS::EC2::Instance",
    "AWS::EC2::SecurityGroup"
  ]
}
```
→ Rule chỉ đánh giá EC2 Instances và Security Groups

### Scope Theo Resource ID Cụ Thể

```json
"Scope": {
  "ComplianceResourceTypes": ["AWS::S3::Bucket"],
  "ComplianceResourceId": "my-critical-bucket"
}
```
→ Rule chỉ đánh giá bucket có tên `my-critical-bucket`

### Scope Theo Tag

```json
"Scope": {
  "TagKey": "Environment",
  "TagValue": "production"
}
```
→ Rule chỉ đánh giá tài nguyên có tag `Environment=production`

### Scope Toàn Bộ (All Resources)

```json
"Scope": {}
```
→ Rule đánh giá tất cả tài nguyên được hỗ trợ

---

## Evaluation Trigger — Thời Điểm Đánh Giá

### Change-Triggered (Kích Hoạt Khi Thay Đổi)

```
Tài nguyên thay đổi → Configuration Item mới → Config Rule tự động re-evaluate
```

- Phản hồi gần real-time (vài giây đến vài phút)
- Chỉ evaluate tài nguyên vừa thay đổi
- Phù hợp: Rules cần phát hiện sớm vi phạm

```json
"Source": {
  "SourceDetails": [
    {
      "EventSource": "aws.config",
      "MessageType": "ConfigurationItemChangeNotification"
    }
  ]
}
```

### Periodic (Định Kỳ)

```
Lịch cố định → Config evaluate TẤT CẢ tài nguyên trong scope
```

- Tần suất: 1h / 3h / 6h / 12h / 24h
- Đánh giá toàn diện, kể cả tài nguyên không thay đổi
- Phù hợp: Kiểm tra trạng thái global (VD: MFA bật chưa, CloudTrail chạy chưa)

```json
"Source": {
  "SourceDetails": [
    {
      "EventSource": "aws.config",
      "MessageType": "ScheduledNotification",
      "MaximumExecutionFrequency": "TwentyFour_Hours"
    }
  ]
}
```

### Kết Hợp Cả Hai

```json
"Source": {
  "SourceDetails": [
    {
      "EventSource": "aws.config",
      "MessageType": "ConfigurationItemChangeNotification"
    },
    {
      "EventSource": "aws.config",
      "MessageType": "ScheduledNotification",
      "MaximumExecutionFrequency": "TwentyFour_Hours"
    }
  ]
}
```

---

## Compliance States — Trạng Thái Tuân Thủ

| Trạng Thái | Ý Nghĩa |
|-----------|---------|
| **COMPLIANT** | Tài nguyên đáp ứng đầy đủ quy tắc |
| **NON_COMPLIANT** | Tài nguyên vi phạm ít nhất một quy tắc |
| **NOT_APPLICABLE** | Rule không áp dụng cho resource type này |
| **INSUFFICIENT_DATA** | Chưa đủ dữ liệu để đánh giá (vừa tạo rule, chưa evaluate) |

### Aggregate Compliance Của Một Rule

Nếu một rule đánh giá 100 tài nguyên:
- 85 COMPLIANT, 15 NON_COMPLIANT → Rule ở trạng thái **NON_COMPLIANT**
- Chỉ khi **tất cả** tài nguyên COMPLIANT → Rule ở trạng thái **COMPLIANT**

---

## Ví Dụ Rules Phổ Biến Theo Danh Mục

### Bộ Rules Tối Thiểu Cho Production

```yaml
# CloudFormation — Triển khai bộ rules cơ bản
Rules:
  # 1. Bảo mật S3
  S3PublicReadProhibited:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: s3-bucket-public-read-prohibited
      Source:
        Owner: AWS
        SourceIdentifier: S3_BUCKET_PUBLIC_READ_PROHIBITED

  # 2. CloudTrail phải bật
  CloudTrailEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: cloudtrail-enabled
      Source:
        Owner: AWS
        SourceIdentifier: CLOUD_TRAIL_ENABLED

  # 3. MFA bắt buộc cho console access
  MFAEnabled:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: mfa-enabled-for-iam-console-access
      Source:
        Owner: AWS
        SourceIdentifier: MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS

  # 4. RDS phải Multi-AZ
  RDSMultiAZ:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: rds-multi-az-support
      Source:
        Owner: AWS
        SourceIdentifier: RDS_MULTI_AZ_SUPPORT
      Scope:
        ComplianceResourceTypes:
          - AWS::RDS::DBInstance

  # 5. EBS phải mã hóa
  EBSEncrypted:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: encrypted-volumes
      Source:
        Owner: AWS
        SourceIdentifier: ENCRYPTED_VOLUMES
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Volume

  # 6. Tags bắt buộc
  RequiredTags:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: required-tags-ec2
      Source:
        Owner: AWS
        SourceIdentifier: REQUIRED_TAGS
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Instance
      InputParameters:
        tag1Key: Environment
        tag2Key: Team
        tag3Key: CostCenter
```

---

## Viết Custom Lambda Rule

### Kiến Trúc Custom Lambda Rule

```
Config thay đổi / lịch
       ↓
AWS Config gọi Lambda (invoking-event)
       ↓
Lambda nhận event, đánh giá tài nguyên
       ↓
Lambda gọi put_evaluations() với kết quả
       ↓
AWS Config cập nhật compliance status
```

### Ví Dụ: Custom Rule Kiểm Tra EC2 Instance Type

```python
import json
import boto3

# Danh sách instance types được phép dùng
APPROVED_INSTANCE_TYPES = ['t3.micro', 't3.small', 't3.medium', 'm5.large', 'm5.xlarge']

config_client = boto3.client('config')


def lambda_handler(event, context):
    """
    Custom Config Rule: EC2 instance phải dùng instance type được phê duyệt
    """
    invoking_event = json.loads(event['invokingEvent'])
    rule_parameters = json.loads(event.get('ruleParameters', '{}'))

    # Lấy danh sách approved types từ parameter (nếu có) hoặc dùng mặc định
    approved_types = rule_parameters.get(
        'approvedInstanceTypes',
        ','.join(APPROVED_INSTANCE_TYPES)
    ).split(',')

    evaluations = []

    # Change-triggered: Chỉ evaluate tài nguyên vừa thay đổi
    if invoking_event.get('messageType') == 'ConfigurationItemChangeNotification':
        ci = invoking_event['configurationItem']
        evaluation = evaluate_ec2_instance(ci, approved_types)
        if evaluation:
            evaluations.append(evaluation)

    # Periodic: Evaluate tất cả EC2 instances
    elif invoking_event.get('messageType') == 'ScheduledNotification':
        evaluations = evaluate_all_ec2_instances(approved_types)

    if evaluations:
        config_client.put_evaluations(
            Evaluations=evaluations,
            ResultToken=event['resultToken']
        )

    return {'statusCode': 200}


def evaluate_ec2_instance(configuration_item, approved_types):
    """Đánh giá một EC2 instance"""
    # Bỏ qua nếu không phải EC2 Instance
    if configuration_item['resourceType'] != 'AWS::EC2::Instance':
        return None

    # Bỏ qua instance đã bị xóa
    if configuration_item['configurationItemStatus'] == 'ResourceDeleted':
        return {
            'ComplianceResourceType': configuration_item['resourceType'],
            'ComplianceResourceId': configuration_item['resourceId'],
            'ComplianceType': 'NOT_APPLICABLE',
            'Annotation': 'Instance đã bị xóa',
            'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
        }

    instance_type = configuration_item['configuration'].get('instanceType', '')
    is_compliant = instance_type in approved_types

    return {
        'ComplianceResourceType': configuration_item['resourceType'],
        'ComplianceResourceId': configuration_item['resourceId'],
        'ComplianceType': 'COMPLIANT' if is_compliant else 'NON_COMPLIANT',
        'Annotation': (
            f'Instance type {instance_type} được phê duyệt'
            if is_compliant
            else f'Instance type {instance_type} KHÔNG được phép. Chỉ cho phép: {", ".join(approved_types)}'
        ),
        'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
    }


def evaluate_all_ec2_instances(approved_types):
    """Đánh giá tất cả EC2 instances (cho periodic trigger)"""
    ec2_client = boto3.client('ec2')
    evaluations = []

    paginator = ec2_client.get_paginator('describe_instances')
    import datetime

    for page in paginator.paginate():
        for reservation in page['Reservations']:
            for instance in reservation['Instances']:
                if instance['State']['Name'] == 'terminated':
                    continue

                instance_type = instance['InstanceType']
                is_compliant = instance_type in approved_types

                evaluations.append({
                    'ComplianceResourceType': 'AWS::EC2::Instance',
                    'ComplianceResourceId': instance['InstanceId'],
                    'ComplianceType': 'COMPLIANT' if is_compliant else 'NON_COMPLIANT',
                    'Annotation': (
                        f'Instance type {instance_type} được phê duyệt'
                        if is_compliant
                        else f'Instance type {instance_type} không được phép'
                    ),
                    'OrderingTimestamp': datetime.datetime.utcnow().isoformat()
                })

    return evaluations
```

### Triển Khai Custom Lambda Rule Qua CloudFormation

```yaml
Resources:
  # Lambda Execution Role
  ConfigRuleLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: ConfigRulePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - config:PutEvaluations
                  - ec2:DescribeInstances
                Resource: '*'

  # Lambda Function
  EC2InstanceTypeCheckFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: config-ec2-instance-type-check
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt ConfigRuleLambdaRole.Arn
      Code:
        ZipFile: |
          # ... (code Lambda ở trên)
      Timeout: 60

  # Quyền cho Config gọi Lambda
  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !GetAtt EC2InstanceTypeCheckFunction.Arn
      Action: lambda:InvokeFunction
      Principal: config.amazonaws.com

  # Config Rule
  EC2InstanceTypeRule:
    Type: AWS::Config::ConfigRule
    Properties:
      ConfigRuleName: ec2-approved-instance-types
      Description: EC2 instances phải dùng instance type được phê duyệt
      Source:
        Owner: CUSTOM_LAMBDA
        SourceIdentifier: !GetAtt EC2InstanceTypeCheckFunction.Arn
        SourceDetails:
          - EventSource: aws.config
            MessageType: ConfigurationItemChangeNotification
          - EventSource: aws.config
            MessageType: ScheduledNotification
            MaximumExecutionFrequency: TwentyFour_Hours
      Scope:
        ComplianceResourceTypes:
          - AWS::EC2::Instance
      InputParameters:
        approvedInstanceTypes: t3.micro,t3.small,t3.medium,m5.large
    DependsOn: LambdaInvokePermission
```

---

## CloudFormation Guard Rules

**CloudFormation Guard** (cfn-guard) cho phép viết rules tuân thủ bằng ngôn ngữ DSL (Domain Specific Language — Ngôn Ngữ Đặc Thù Miền) đơn giản, không cần code Lambda.

### Ví Dụ Guard Rule

```
# File: ec2-rules.guard

# Rule: EC2 instance phải có tag Environment
rule EC2_MUST_HAVE_ENVIRONMENT_TAG when resourceType == "AWS::EC2::Instance" {
    configuration.tags exists
    configuration.tags[*].key contains "Environment"
    <<
    EC2 instance PHẢI có tag "Environment".
    Ví dụ: Environment = production | staging | development
    >>
}

# Rule: EC2 không được dùng instance type quá lớn
rule EC2_APPROVED_INSTANCE_TYPES when resourceType == "AWS::EC2::Instance" {
    configuration.instanceType in ["t3.micro", "t3.small", "t3.medium", "m5.large", "m5.xlarge"]
    <<
    Instance type không được phép. Chỉ sử dụng: t3.micro, t3.small, t3.medium, m5.large, m5.xlarge
    >>
}
```

### So Sánh Lambda Rule vs Guard Rule

| Tiêu Chí | Lambda-backed Rule | Guard Rule |
|---------|-------------------|-----------|
| **Ngôn ngữ** | Python, Node.js, Java... | Guard DSL |
| **Độ phức tạp** | Không giới hạn | Phù hợp logic đơn giản–trung bình |
| **Cross-resource** | ✅ Có thể gọi API khác | ❌ Chỉ dữ liệu Configuration Item |
| **Maintain** | Phức tạp hơn (Lambda lifecycle) | Đơn giản hơn (chỉ file text) |
| **Chi phí** | Lambda invocations + Config | Chỉ Config |
| **Debugging** | CloudWatch Logs | Khó hơn |

---

## Thực Hành Tốt Nhất

### 1. Ưu Tiên Managed Rules Trước

```
Quy trình lựa chọn:
1. Tìm trong 250+ Managed Rules xem có rule phù hợp không
2. Nếu không có → Xem xét Guard Rule cho logic đơn giản
3. Chỉ dùng Lambda khi cần cross-resource check hoặc external API
```

### 2. Đặt Tên Rule Nhất Quán

```
Pattern: {resource-type}-{what-is-checked}-{action/check}

Ví dụ:
  ec2-instance-approved-instance-types    ✅
  s3-bucket-public-access-blocked         ✅
  my-rule-1                               ❌ (không rõ ràng)
```

### 3. Thêm Annotation Chi Tiết

Annotation giúp người dùng hiểu phải làm gì để sửa:

```python
'Annotation': (
    'NON_COMPLIANT: S3 bucket có public read access. '
    'Hành động cần thiết: Tắt "Block Public Access" settings. '
    'Tham khảo: https://docs.aws.amazon.com/s3/...'
)
```

### 4. Nhóm Rules Theo Framework

Đừng bật rules ngẫu nhiên. Nhóm theo:
- **Baseline security** — Rules bắt buộc cho mọi môi trường
- **Production requirements** — Rules chỉ cần cho production
- **Compliance framework** — Rules theo PCI-DSS / HIPAA / CIS

### 5. Review & Dọn Dẹp Rules Định Kỳ

- Rules không còn relevant → Tắt để tiết kiệm chi phí
- Rules luôn COMPLIANT → Kiểm tra xem scope có đúng không
- Rules luôn INSUFFICIENT_DATA → Kiểm tra trigger configuration

---

## Câu Hỏi Phỏng Vấn

**Q: Config Rule và AWS Security Hub khác nhau thế nào?**

A: Config Rules đánh giá *cấu hình cụ thể* của từng tài nguyên. Security Hub tổng hợp *findings từ nhiều nguồn* (Config, GuardDuty, Inspector, Macie...) vào một giao diện và map sang security standards (CIS, AWS Foundational). Config là nguồn dữ liệu cho Security Hub, không phải đối thủ.

**Q: Tại sao Lambda Custom Rule của tôi luôn trả về INSUFFICIENT_DATA?**

A: Nguyên nhân phổ biến: (1) Lambda Role thiếu quyền `config:PutEvaluations`; (2) Lambda function bị lỗi (xem CloudWatch Logs); (3) `resultToken` trong `put_evaluations()` không khớp với token trong event; (4) Lambda timeout trước khi gọi `put_evaluations()`.

**Q: Periodic evaluation với tần suất 1 giờ có tốn nhiều tiền không?**

A: Tùy số lượng tài nguyên. Mỗi evaluation tính $0.001. Nếu có 500 tài nguyên trong scope, 1 giờ một lần = 500 evaluations × 24 × 30 = 360,000 evaluations/tháng = $360/tháng chỉ riêng rule đó. Hãy dùng 24h frequency trừ khi thực sự cần phát hiện nhanh.

---

**Tiếp Theo:** [3-conformance-packs.md](3-conformance-packs.md) — Conformance Packs: CIS, PCI-DSS, NIST, HIPAA
