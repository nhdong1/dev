# Configuration Recorder — Bộ Ghi Cấu Hình AWS Config

> **Configuration Recorder** (Bộ Ghi Cấu Hình) là thành phần trung tâm của AWS Config, chịu trách nhiệm phát hiện và ghi lại mọi thay đổi về cấu hình của tài nguyên AWS vào **Configuration History** (Lịch Sử Cấu Hình) và **Configuration Snapshots** (Ảnh Chụp Cấu Hình).

---

## 📚 Mục Lục

1. [Configuration Recorder Là Gì?](#configuration-recorder-là-gì)
2. [Configuration Item — Đơn Vị Dữ Liệu](#configuration-item--đơn-vị-dữ-liệu)
3. [Resource Types Được Hỗ Trợ](#resource-types-được-hỗ-trợ)
4. [Delivery Channel — Kênh Giao Nhận](#delivery-channel--kênh-giao-nhận)
5. [Cấu Hình Recorder](#cấu-hình-recorder)
6. [Configuration Snapshot vs Configuration History](#configuration-snapshot-vs-configuration-history)
7. [Tích Hợp Với Các Dịch Vụ Khác](#tích-hợp-với-các-dịch-vụ-khác)
8. [Thực Hành Tốt Nhất](#thực-hành-tốt-nhất)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Configuration Recorder Là Gì?

**Configuration Recorder** theo dõi tài nguyên AWS và tạo ra **Configuration Items** (Mục Cấu Hình) mỗi khi có thay đổi.

### Cơ Chế Hoạt Động

```
Tài nguyên thay đổi
       ↓
AWS Config phát hiện qua AWS API polling + CloudTrail events
       ↓
Tạo Configuration Item (CI) mô tả trạng thái mới
       ↓
Lưu CI vào:
  ├── Config Service (để query & evaluate rules)
  └── S3 Bucket (qua Delivery Channel)
```

### Khi Nào Configuration Recorder Ghi Lại?

| Sự Kiện | Recorder Ghi? |
|---------|--------------|
| Tài nguyên được tạo mới | ✅ |
| Tài nguyên bị xóa | ✅ |
| Cấu hình tài nguyên thay đổi | ✅ |
| Tag của tài nguyên thay đổi | ✅ (nếu bật recording tags) |
| Tài nguyên không thay đổi | ❌ (chỉ ghi khi có thay đổi) |

> **Lưu ý quan trọng:** Recorder chỉ theo dõi *cấu hình*, không theo dõi *dữ liệu bên trong* (VD: nội dung file trong S3, dữ liệu trong RDS).

---

## Configuration Item — Đơn Vị Dữ Liệu

**Configuration Item (CI)** là bản ghi (record) mô tả đầy đủ trạng thái của một tài nguyên tại một thời điểm cụ thể.

### Cấu Trúc Configuration Item

```json
{
  "configurationItemVersion": "1.3",
  "configurationItemCaptureTime": "2026-05-17T10:30:00.000Z",
  "configurationItemStatus": "OK",
  "configurationStateId": "1716000600000",

  "resourceType": "AWS::EC2::Instance",
  "resourceId": "i-0abc123def456789",
  "resourceName": "production-web-server",
  "ARN": "arn:aws:ec2:ap-southeast-1:123456789012:instance/i-0abc123def456789",
  "awsRegion": "ap-southeast-1",
  "availabilityZone": "ap-southeast-1a",
  "resourceCreationTime": "2026-01-15T08:00:00.000Z",

  "tags": {
    "Environment": "production",
    "Team": "backend",
    "CostCenter": "eng-001"
  },

  "configuration": {
    "instanceType": "t3.medium",
    "imageId": "ami-0abcdef1234567890",
    "state": { "code": 16, "name": "running" },
    "privateIpAddress": "10.0.1.100",
    "securityGroups": [
      { "groupId": "sg-0123456789abcdef0", "groupName": "web-sg" }
    ],
    "iamInstanceProfile": {
      "arn": "arn:aws:iam::123456789012:instance-profile/EC2-SSM-Role"
    }
  },

  "relationships": [
    {
      "resourceType": "AWS::EC2::VPC",
      "resourceId": "vpc-0abc123",
      "name": "Is contained in Vpc"
    },
    {
      "resourceType": "AWS::EC2::Subnet",
      "resourceId": "subnet-0abc123",
      "name": "Is contained in Subnet"
    }
  ],

  "supplementaryConfiguration": {}
}
```

### Các Trạng Thái configurationItemStatus

| Trạng Thái | Ý Nghĩa |
|-----------|---------|
| `OK` | Ghi thành công, tài nguyên tồn tại |
| `ResourceDiscovered` | Tài nguyên được phát hiện lần đầu |
| `ResourceNotRecorded` | Resource type không được bật trong recorder |
| `ResourceDeleted` | Tài nguyên đã bị xóa |
| `ResourceDeletedNotRecorded` | Tài nguyên bị xóa nhưng không được record |

---

## Resource Types Được Hỗ Trợ

### Chọn Lọc Resource Types

Recorder có thể cấu hình theo ba chế độ:

#### Chế Độ 1: Tất Cả Resource Types (All Resources)

```yaml
# CloudFormation snippet
RecordingGroup:
  AllSupported: true
  IncludeGlobalResourceTypes: true  # IAM Users, Roles, Policies
```

- Tự động bao gồm resource types mới khi AWS release
- Chi phí cao hơn nếu có nhiều tài nguyên
- Phù hợp: Compliance đầy đủ, không biết trước cần theo dõi gì

#### Chế Độ 2: Chỉ Resource Types Chọn Lọc (Specific Resources)

```yaml
RecordingGroup:
  AllSupported: false
  ResourceTypes:
    - AWS::EC2::Instance
    - AWS::EC2::SecurityGroup
    - AWS::S3::Bucket
    - AWS::IAM::Role
    - AWS::RDS::DBInstance
    - AWS::Lambda::Function
```

- Chi phí tối ưu hơn
- Phải thêm thủ công khi cần theo dõi resource type mới
- Phù hợp: Biết chính xác cần tuân thủ resource types nào

#### Chế Độ 3: Exclude Specific Resource Types (Loại Trừ)

```yaml
RecordingGroup:
  AllSupported: true
  ExclusionByResourceTypes:
    ResourceTypes:
      - AWS::CloudWatch::Alarm  # Quá nhiều thay đổi, không cần track
      - AWS::Config::ResourceCompliance
```

### Resource Types Quan Trọng Nhất Cho Compliance

| Danh Mục | Resource Types |
|---------|---------------|
| **Compute** | EC2::Instance, EC2::SecurityGroup, EC2::NetworkInterface |
| **Storage** | S3::Bucket, EBS::Volume, EFS::FileSystem |
| **Database** | RDS::DBInstance, RDS::DBCluster, ElastiCache::CacheCluster |
| **IAM** | IAM::User, IAM::Role, IAM::Policy, IAM::Group |
| **Network** | VPC, Subnet, RouteTable, InternetGateway, NatGateway |
| **Serverless** | Lambda::Function, ApiGateway::RestApi |
| **Encryption** | KMS::Key, ACM::Certificate |

### Global Resource Types — Tài Nguyên Toàn Cục

IAM là **global resource** — tồn tại ở tầng account, không gắn với region cụ thể. Để ghi lại IAM resources:

```yaml
RecordingGroup:
  AllSupported: true
  IncludeGlobalResourceTypes: true  # Phải bật thêm
```

> **Lưu ý:** Global resources chỉ nên bật trong **một region** (thường là region chính) để tránh tính phí trùng lặp.

---

## Delivery Channel — Kênh Giao Nhận

**Delivery Channel** (Kênh Giao Nhận) xác định nơi AWS Config gửi Configuration Items và Snapshots.

### Thành Phần Delivery Channel

```
Delivery Channel
├── S3 Bucket              ← Lưu trữ configuration history & snapshots
├── S3 Key Prefix          ← Tiền tố đường dẫn trong S3 (tùy chọn)
├── SNS Topic              ← Thông báo khi có thay đổi (tùy chọn)
└── Snapshot Frequency     ← Tần suất gửi snapshot (1, 3, 6, 12, 24 giờ)
```

### Cấu Hình Delivery Channel Qua AWS CLI

```bash
# Tạo delivery channel
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "my-config-bucket-123456789012",
    "s3KeyPrefix": "config/",
    "snsTopicARN": "arn:aws:sns:ap-southeast-1:123456789012:config-notifications",
    "configSnapshotDeliveryProperties": {
      "deliveryFrequency": "TwentyFour_Hours"
    }
  }'
```

### Cấu Trúc S3 Bucket

```
s3://my-config-bucket-123456789012/
└── config/
    └── AWSLogs/
        └── 123456789012/
            └── Config/
                └── ap-southeast-1/
                    ├── ConfigHistory/
                    │   └── AWS::EC2::Instance/
                    │       └── 123456789012_Config_ap-southeast-1_ConfigHistory_AWS::EC2::Instance_20260517.json.gz
                    └── ConfigSnapshot/
                        └── 123456789012_Config_ap-southeast-1_ConfigSnapshot_20260517T103000Z_abc123.json.gz
```

### Yêu Cầu Quyền S3 Bucket

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSConfigBucketPermissionsCheck",
      "Effect": "Allow",
      "Principal": { "Service": "config.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::my-config-bucket-123456789012",
      "Condition": {
        "StringEquals": { "AWS:SourceAccount": "123456789012" }
      }
    },
    {
      "Sid": "AWSConfigBucketDelivery",
      "Effect": "Allow",
      "Principal": { "Service": "config.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-config-bucket-123456789012/config/AWSLogs/123456789012/Config/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "AWS:SourceAccount": "123456789012"
        }
      }
    }
  ]
}
```

---

## Cấu Hình Recorder

### Bật Recorder Qua CloudFormation

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS Config Recorder Setup

Resources:
  # IAM Role cho Config
  ConfigRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: AWSConfigRole
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: config.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWS_ConfigRole

  # S3 Bucket lưu trữ
  ConfigBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'aws-config-${AWS::AccountId}-${AWS::Region}'
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms

  # Bucket Policy
  ConfigBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref ConfigBucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AWSConfigBucketPermissionsCheck
            Effect: Allow
            Principal:
              Service: config.amazonaws.com
            Action: s3:GetBucketAcl
            Resource: !GetAtt ConfigBucket.Arn
          - Sid: AWSConfigBucketDelivery
            Effect: Allow
            Principal:
              Service: config.amazonaws.com
            Action: s3:PutObject
            Resource: !Sub '${ConfigBucket.Arn}/AWSLogs/${AWS::AccountId}/Config/*'
            Condition:
              StringEquals:
                s3:x-amz-acl: bucket-owner-full-control

  # Configuration Recorder
  ConfigRecorder:
    Type: AWS::Config::ConfigurationRecorder
    Properties:
      Name: default
      RoleARN: !GetAtt ConfigRole.Arn
      RecordingGroup:
        AllSupported: true
        IncludeGlobalResourceTypes: true

  # Delivery Channel
  DeliveryChannel:
    Type: AWS::Config::DeliveryChannel
    Properties:
      Name: default
      S3BucketName: !Ref ConfigBucket
      ConfigSnapshotDeliveryProperties:
        DeliveryFrequency: TwentyFour_Hours
    DependsOn: ConfigBucketPolicy
```

### Bật/Tắt Recorder Qua AWS CLI

```bash
# Bật recorder
aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# Tắt recorder
aws configservice stop-configuration-recorder \
  --configuration-recorder-name default

# Kiểm tra trạng thái
aws configservice describe-configuration-recorder-status \
  --configuration-recorder-names default
```

---

## Configuration Snapshot vs Configuration History

### Configuration Snapshot — Ảnh Chụp Cấu Hình

**Snapshot** là ảnh chụp *toàn bộ* trạng thái cấu hình của tất cả tài nguyên tại một thời điểm.

```
Snapshot = "Trạng thái toàn bộ hạ tầng tại 00:00 ngày 17/05/2026"

Nội dung: Tất cả Configuration Items của mọi resource
         đang được theo dõi, trong một file JSON duy nhất
```

**Khi nào snapshot được tạo:**
- Theo lịch tự động (Delivery Frequency): 1h / 3h / 6h / 12h / 24h
- Khi gọi thủ công: `aws configservice deliver-config-snapshot`

**Dùng snapshot khi:**
- Cần toàn cảnh hạ tầng tại một thời điểm cụ thể (point-in-time audit)
- Import vào Athena/QuickSight để phân tích quy mô lớn
- Backup cấu hình trước khi thực hiện thay đổi lớn

### Configuration History — Lịch Sử Thay Đổi

**History** là chuỗi thời gian (timeline) của các Configuration Items cho *một tài nguyên cụ thể*.

```
History của EC2 instance i-0abc123:
  T1: 09:00 - Tạo mới, instanceType=t3.small
  T2: 11:30 - Thay đổi SecurityGroup
  T3: 14:00 - Modify instanceType=t3.medium
  T4: 16:45 - Thêm tag CostCenter
```

**Dùng history khi:**
- Điều tra "cấu hình thay đổi lúc nào?"
- Audit trail cho một tài nguyên cụ thể
- Khôi phục cấu hình về trạng thái trước đó

### Bảng So Sánh

| Tiêu Chí | Snapshot | History |
|---------|---------|---------|
| Phạm vi | Toàn bộ tài nguyên | Một tài nguyên |
| Thời gian | Một thời điểm | Chuỗi thời gian |
| Tạo theo | Lịch định kỳ | Mỗi lần có thay đổi |
| Dùng cho | Point-in-time audit | Change tracking |
| Lưu trữ | S3 (file lớn) | S3 (nhiều file nhỏ) |

---

## Tích Hợp Với Các Dịch Vụ Khác

### Config + CloudTrail

```
CloudTrail: "User admin@example.com gọi ModifyDBInstance lúc 15:32"
Config:     "RDS instance db-prod: Multi-AZ thay đổi true → false lúc 15:33"

Kết hợp: Biết AI làm gì (CloudTrail) + Hệ quả cấu hình là gì (Config)
```

### Config + Amazon Athena — Phân Tích Quy Mô Lớn

```sql
-- Query từ Config snapshots trong S3 bằng Athena
SELECT
  resourceType,
  resourceId,
  configuration.state.name AS instance_state,
  tags['Environment'] AS environment
FROM
  aws_config_configuration_snapshot
WHERE
  resourceType = 'AWS::EC2::Instance'
  AND tags['Environment'] = 'production'
  AND dt = '2026-05-17'
```

### Config + Security Hub

AWS Config là **nguồn dữ liệu chính** cho Security Hub findings:
- Security Hub đọc kết quả Config Rule evaluations
- Tự động map sang CIS AWS Foundations Benchmark controls
- Tập trung findings từ nhiều account vào Security Hub master

### Config + EventBridge — Tự Động Hóa Phản Hồi

```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"],
  "detail": {
    "messageType": ["ComplianceChangeNotification"],
    "newEvaluationResult": {
      "complianceType": ["NON_COMPLIANT"]
    },
    "resourceType": ["AWS::S3::Bucket"]
  }
}
```

→ Trigger Lambda function gửi Slack alert + tạo Jira ticket

---

## Thực Hành Tốt Nhất

### 1. Chọn Resource Types Có Chủ Đích

```
❌ Không tốt: AllSupported=true mà không xem xét chi phí
✅ Tốt hơn: Bắt đầu với AllSupported=true, sau một tháng review
            xem resource types nào không có rules → loại trừ để tiết kiệm
```

### 2. Tách S3 Bucket Riêng Cho Config

- Không dùng chung bucket với CloudTrail hay application logs
- Bật versioning và MFA Delete trên bucket
- Bật S3 Object Lock nếu cần immutable audit records (WORM — Write Once Read Many)
- Mã hóa bằng KMS customer-managed key

### 3. Bật Global Resources Một Lần Duy Nhất

```yaml
# Region chính (ap-southeast-1): Bật global resources
IncludeGlobalResourceTypes: true

# Các region phụ: Tắt global resources
IncludeGlobalResourceTypes: false
```

### 4. Giữ Recorder Luôn Bật

- Tắt recorder → mất lịch sử thay đổi trong thời gian tắt
- Khoảng trống lịch sử làm yếu audit trail, vi phạm compliance
- Nếu cần tiết kiệm chi phí: Loại trừ resource types, không tắt recorder

### 5. Retention — Lưu Giữ Dữ Liệu

| Dữ Liệu | Lưu Trữ Mặc Định | Tùy Chỉnh |
|---------|-----------------|-----------|
| Config trong service | 7 năm | Không thể tùy chỉnh |
| S3 snapshots/history | Theo S3 Lifecycle policy | Tùy ý |
| Athena queries | Theo S3 retention | Tùy ý |

---

## Câu Hỏi Phỏng Vấn

**Q: Điều gì xảy ra nếu tắt Configuration Recorder?**

A: AWS Config ngừng theo dõi thay đổi trong khoảng thời gian recorder bị tắt. Tạo ra "khoảng trống" (gap) trong lịch sử cấu hình. Config Rules cũng ngừng đánh giá (không trigger khi có thay đổi). Khi bật lại, recorder sẽ chụp trạng thái hiện tại nhưng không thể khôi phục lịch sử đã bị bỏ qua.

**Q: Configuration Snapshot tốn nhiều tiền không?**

A: Bản thân việc tạo snapshot không tính phí riêng — phí đã bao gồm trong phí Configuration Items. Chi phí thêm là S3 storage. Snapshot thường rất lớn (hàng trăm MB đến vài GB nếu có nhiều tài nguyên) nên nên đặt S3 Lifecycle policy tự động chuyển sang Glacier sau 90 ngày.

**Q: Có thể dùng Configuration Snapshot để khôi phục tài nguyên không?**

A: AWS Config không có tính năng rollback tự động từ snapshot. Snapshot chỉ là dữ liệu mô tả cấu hình. Để khôi phục, cần dùng thông tin từ snapshot kết hợp với CloudFormation/Terraform/AWS CLI để tái tạo cấu hình cũ thủ công.

---

**Tiếp Theo:** [2-config-rules.md](2-config-rules.md) — Config Rules: Managed Rules, Custom Lambda Rules, Evaluation Scope
