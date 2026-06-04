# CloudTrail Trails — Cấu Hình Ghi Nhật Ký Liên Tục

> **Trail** (Dấu Vết) là cấu hình để CloudTrail ghi event liên tục vào S3 và/hoặc CloudWatch Logs — khác với Event History (chỉ lưu 90 ngày, read-only). Trail là thành phần bắt buộc cho mọi môi trường production cần audit log lâu dài.

---

## 📚 Mục Lục

1. [Event History vs Trail — Khác Biệt Cốt Lõi](#event-history-vs-trail--khác-biệt-cốt-lõi)
2. [Single-Region vs Multi-Region Trail](#single-region-vs-multi-region-trail)
3. [Cấu Hình S3 Destination](#cấu-hình-s3-destination)
4. [Cấu Hình CloudWatch Logs](#cấu-hình-cloudwatch-logs)
5. [Log File Validation — Xác Thực Tính Toàn Vẹn](#log-file-validation--xác-thực-tính-toàn-vẹn)
6. [Encryption với KMS](#encryption-với-kms)
7. [SNS Notification](#sns-notification)
8. [Trail Best Practices](#trail-best-practices)
9. [Tạo Trail Bằng AWS CLI & CloudFormation](#tạo-trail-bằng-aws-cli--cloudformation)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Event History vs Trail — Khác Biệt Cốt Lõi

| Đặc Điểm              | Event History (Lịch Sử Sự Kiện)     | Trail (Dấu Vết)                         |
| --------------------- | ------------------------------------ | --------------------------------------- |
| **Cấu hình cần thiết** | Không — tự động bật                 | Phải tự tạo                             |
| **Thời gian lưu**     | 90 ngày gần nhất                     | Không giới hạn (lưu trong S3)           |
| **Loại event**        | Chỉ Management Events                | Management + Data + Insights Events     |
| **Xuất dữ liệu**      | Không thể export tự động             | Tự động gửi vào S3, CW Logs            |
| **Phân tích nâng cao** | Không                               | Có — Athena, OpenSearch, Splunk         |
| **Chi phí**           | Miễn phí                             | S3 + CW Logs storage                    |
| **Phù hợp**           | Debug nhanh, tra cứu ad-hoc          | Compliance, audit lâu dài, monitoring   |

---

## Single-Region vs Multi-Region Trail

### Single-Region Trail

Chỉ ghi event xảy ra trong **một region cụ thể**.

```
Account: 123456789012
  us-east-1: Trail A → S3 bucket (us-east-1 events)
  us-west-2: (không có Trail — blind spot!)
  eu-west-1: (không có Trail — blind spot!)
```

**Vấn đề:** Kẻ tấn công có thể thực hiện hành động ở region không có Trail để tránh bị ghi nhận.

### Multi-Region Trail (Khuyên Dùng)

Ghi event từ **tất cả region** (kể cả Global Services như IAM, STS, Route53) vào một S3 bucket trung tâm.

```
Account: 123456789012
  Multi-Region Trail:
    us-east-1 events ──┐
    us-west-2 events ──┤→ S3: my-audit-logs-us-east-1/
    eu-west-1 events ──┤    (central logging bucket)
    ap-southeast-1 ────┤
    Global services ───┘
```

**Lợi ích:**
- Không có blind spot theo region
- Một S3 bucket duy nhất để query và phân tích
- Global services (IAM, STS, CloudFront, Route53) chỉ ghi ở `us-east-1` — multi-region trail tự động bao gồm

### Global Service Events

**Global services** như IAM, STS, AWS Support ghi event về `us-east-1`, bất kể bạn ở region nào.

- Multi-region Trail: Tự động bao gồm global service events
- Single-region Trail: Chỉ bắt được nếu trail đặt ở `us-east-1`

---

## Cấu Hình S3 Destination

### Cấu Trúc Thư Mục S3

CloudTrail lưu log theo cấu trúc:

```
s3://my-cloudtrail-bucket/
└── AWSLogs/
    └── {AccountId}/
        └── CloudTrail/
            └── {Region}/
                └── {Year}/
                    └── {Month}/
                        └── {Day}/
                            └── {AccountId}_CloudTrail_{Region}_{Timestamp}_{UniqueString}.json.gz
```

Ví dụ file thực tế:
```
123456789012_CloudTrail_us-east-1_20260517T0300Z_Ab12Cd34Ef56.json.gz
```

### S3 Bucket Policy Cần Thiết

CloudTrail cần quyền `PutObject` vào S3 bucket. Bucket policy mẫu:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::my-cloudtrail-bucket"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-cloudtrail-bucket/AWSLogs/123456789012/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
```

### S3 Bucket Security Best Practices

```
✅ Block all public access — CloudTrail logs không được public
✅ MFA Delete — Yêu cầu MFA khi xóa object
✅ Object Lock — WORM (Write Once Read Many) cho compliance
✅ S3 Versioning — Khôi phục nếu bị xóa
✅ Lifecycle Policy — Chuyển sang S3 Glacier sau 90 ngày
✅ S3 Access Logging — Ghi lại ai access CloudTrail bucket
✅ Deny delete cho CloudTrail service principal
```

**S3 Object Lock Policy (cho compliance yêu cầu immutable logs):**
```json
{
  "Sid": "DenyDeleteCloudTrailLogs",
  "Effect": "Deny",
  "Principal": "*",
  "Action": [
    "s3:DeleteObject",
    "s3:DeleteObjectVersion"
  ],
  "Resource": "arn:aws:s3:::my-cloudtrail-bucket/AWSLogs/*"
}
```

### S3 Lifecycle Policy Tiết Kiệm Chi Phí

```json
{
  "Rules": [{
    "ID": "CloudTrailLogsLifecycle",
    "Status": "Enabled",
    "Transitions": [
      {"Days": 90, "StorageClass": "STANDARD_IA"},
      {"Days": 365, "StorageClass": "GLACIER"}
    ],
    "Expiration": {"Days": 2555}
  }]
}
```

---

## Cấu Hình CloudWatch Logs

Ngoài S3 (lưu trữ), Trail có thể gửi event sang **CloudWatch Logs** để:
- Tạo Metric Filters phát hiện hành vi nguy hiểm
- Đặt CloudWatch Alarms cảnh báo real-time
- Query bằng Logs Insights

### Thiết Lập CloudWatch Logs trong Trail

```
Trail Settings:
  CloudWatch Logs Log Group: /cloudtrail/management
  IAM Role: CloudTrailToCloudWatchRole
    → Quyền: logs:CreateLogStream, logs:PutLogEvents
```

### Metric Filters Quan Trọng

Tạo metric từ CloudTrail logs để cảnh báo các hành động nguy hiểm:

```
Filter Pattern                                    Cảnh Báo Khi
─────────────────────────────────────────────────────────────────────
{ $.eventName = "ConsoleLogin" &&
  $.additionalEventData.MFAUsed = "No" }         Login không có MFA

{ $.eventSource = "iam.amazonaws.com" &&
  ($.eventName = "CreateAccessKey" ||
   $.eventName = "CreateUser") }                  Tạo IAM user/key mới

{ $.eventName = "DeleteTrail" ||
  $.eventName = "StopLogging" }                   CloudTrail bị tắt!

{ $.userIdentity.type = "Root" &&
  $.userIdentity.invokedBy NOT EXISTS }           Root account login

{ $.eventName = "AuthorizeSecurityGroupIngress" &&
  $.requestParameters.ipPermissions.items[0]
  .ipRanges.items[0].cidrIp = "0.0.0.0/0" }     SG mở cho internet

{ ($.eventName = "CreateBucket" ||
   $.eventName = "DeleteBucket") }                S3 bucket thay đổi

{ $.errorCode = "AccessDenied" }                  Truy cập bị từ chối
```

### Tạo Alarm Từ Metric Filter

```bash
# Tạo metric filter
aws logs put-metric-filter \
  --log-group-name /cloudtrail/management \
  --filter-name "RootAccountUsage" \
  --filter-pattern '{ $.userIdentity.type = "Root" }' \
  --metric-transformations \
    metricName=RootAccountUsageCount,metricNamespace=CloudTrailMetrics,metricValue=1

# Tạo alarm từ metric
aws cloudwatch put-metric-alarm \
  --alarm-name "RootAccountUsageAlarm" \
  --alarm-description "Root account was used" \
  --metric-name RootAccountUsageCount \
  --namespace CloudTrailMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:security-alerts
```

---

## Log File Validation — Xác Thực Tính Toàn Vẹn

**Log File Validation** (Xác Thực Tệp Nhật Ký) đảm bảo CloudTrail logs không bị giả mạo sau khi ghi.

### Cơ Chế Hoạt Động

```
CloudTrail ghi log file → SHA-256 hash của file
                         → Tạo digest file (tóm lược)
                         → Ký digest bằng private key của AWS
                         → Ghi digest vào S3

Để xác minh:
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:... \
  --start-time 2026-05-01T00:00:00Z

CloudTrail:
  1. Tải digest files
  2. Verify chữ ký AWS bằng public key
  3. Tính lại SHA-256 của mỗi log file
  4. So sánh với hash trong digest
  → "All 127 log files validated successfully"
```

### Cấu Trúc Digest File

```
s3://my-cloudtrail-bucket/
└── AWSLogs/{AccountId}/CloudTrail-Digest/{Region}/{Year}/{Month}/{Day}/
    └── {AccountId}_CloudTrail-Digest_{Region}_{TrailName}_{Timestamp}.json.gz
```

Digest file chứa:
```json
{
  "digestStartTime": "2026-05-17T00:00:00Z",
  "digestEndTime": "2026-05-17T01:00:00Z",
  "logFiles": [
    {
      "s3Bucket": "my-cloudtrail-bucket",
      "s3Object": "AWSLogs/.../file.json.gz",
      "hashValue": "abc123...",
      "hashAlgorithm": "SHA-256"
    }
  ],
  "previousDigestHashValue": "def456..."
}
```

> **Quan trọng:** Digest files tạo thành chuỗi hash chain — nếu kẻ tấn công xóa log, hash chain sẽ bị gãy và phát hiện được.

---

## Encryption với KMS

Mặc định CloudTrail logs được mã hóa bằng **SSE-S3** (Server-Side Encryption với S3 Managed Keys). Để bảo mật cao hơn, dùng **KMS CMK** (Customer Managed Key — Khóa Do Khách Hàng Quản Lý):

### Lợi Ích KMS Encryption

- Kiểm soát hoàn toàn: Rotate key, revoke access, audit key usage
- Separation of duties (phân tách trách nhiệm): Người có S3 access vẫn không đọc được log nếu không có KMS quyền
- CloudTrail tự động ghi lại ai giải mã log (thông qua KMS API calls chính trong CloudTrail)

### KMS Key Policy Cho CloudTrail

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Allow CloudTrail to encrypt logs",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "kms:GenerateDataKey*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:123456789012:trail/*"
        }
      }
    },
    {
      "Sid": "Allow decrypt for authorized users",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/SecurityAuditRole"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## SNS Notification

Trail có thể gửi **SNS notification** mỗi khi có log file mới được gửi vào S3 — hữu ích để trigger processing pipeline:

```
CloudTrail → S3 (log file) → SNS Topic → Lambda (process log)
                                        → SQS (queue processing)
                                        → Kinesis Firehose → OpenSearch
```

> **Lưu ý:** SNS notification cho mỗi file log (không phải mỗi event). Nếu cần real-time alerting từng event, dùng CloudWatch Logs + Metric Filter.

---

## Trail Best Practices

### Checklist Trail Setup Chuẩn

```
☑ Tạo Multi-Region Trail (bao gồm global service events)
☑ Bật Log File Validation
☑ Mã hóa bằng KMS CMK (không dùng SSE-S3 cho production)
☑ S3 bucket: Block public access + MFA Delete + Versioning
☑ S3 bucket riêng cho CloudTrail (không mix với app data)
☑ Gửi sang CloudWatch Logs để real-time alerting
☑ Tạo Metric Filters cho các event nguy hiểm
☑ Bật CloudWatch Alarm cho Root account usage
☑ S3 Lifecycle Policy để chuyển sang Glacier sau 90 ngày
☑ Deny quyền xóa Trail cho tất cả user (kể cả admin)
☑ Bật Organization Trail nếu dùng AWS Organizations
```

### Ngăn Xóa Trail — SCP (Service Control Policy)

Trong môi trường Organizations, dùng SCP để ngăn member accounts tắt CloudTrail:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyStopCloudTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Tạo Trail Bằng AWS CLI & CloudFormation

### AWS CLI

```bash
# Tạo S3 bucket cho CloudTrail
aws s3 mb s3://company-cloudtrail-logs-123456789012 --region us-east-1

# Áp dụng bucket policy (từ file policy.json)
aws s3api put-bucket-policy \
  --bucket company-cloudtrail-logs-123456789012 \
  --policy file://cloudtrail-bucket-policy.json

# Tạo Multi-Region Trail
aws cloudtrail create-trail \
  --name company-multi-region-trail \
  --s3-bucket-name company-cloudtrail-logs-123456789012 \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:/cloudtrail/management:* \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrailToCloudWatchRole \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012

# Bật Trail (mặc định tạo xong chưa bật)
aws cloudtrail start-logging --name company-multi-region-trail

# Kiểm tra trạng thái
aws cloudtrail get-trail-status --name company-multi-region-trail
```

### CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudTrail Multi-Region Trail Setup'

Resources:
  CloudTrailBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'cloudtrail-logs-${AWS::AccountId}'
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
              KMSMasterKeyID: !Ref CloudTrailKMSKey
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      LifecycleConfiguration:
        Rules:
          - Id: MoveToGlacier
            Status: Enabled
            Transitions:
              - TransitionInDays: 90
                StorageClass: STANDARD_IA
              - TransitionInDays: 365
                StorageClass: GLACIER

  CloudTrailBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref CloudTrailBucket
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AWSCloudTrailAclCheck
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: s3:GetBucketAcl
            Resource: !GetAtt CloudTrailBucket.Arn
          - Sid: AWSCloudTrailWrite
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: s3:PutObject
            Resource: !Sub '${CloudTrailBucket.Arn}/AWSLogs/${AWS::AccountId}/*'
            Condition:
              StringEquals:
                s3:x-amz-acl: bucket-owner-full-control

  CloudTrailKMSKey:
    Type: AWS::KMS::Key
    Properties:
      Description: 'KMS key for CloudTrail log encryption'
      EnableKeyRotation: true
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          - Sid: Allow CloudTrail to encrypt
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: kms:GenerateDataKey*
            Resource: '*'

  CloudTrailLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /cloudtrail/management
      RetentionInDays: 90

  CloudTrailRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: CloudTrailToCloudWatchPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: !GetAtt CloudTrailLogGroup.Arn

  MultiRegionTrail:
    Type: AWS::CloudTrail::Trail
    DependsOn: CloudTrailBucketPolicy
    Properties:
      TrailName: company-multi-region-trail
      S3BucketName: !Ref CloudTrailBucket
      IsLogging: true
      IsMultiRegionTrail: true
      IncludeGlobalServiceEvents: true
      EnableLogFileValidation: true
      CloudWatchLogsLogGroupArn: !GetAtt CloudTrailLogGroup.Arn
      CloudWatchLogsRoleArn: !GetAtt CloudTrailRole.Arn
      KMSKeyId: !Ref CloudTrailKMSKey
      EventSelectors:
        - ReadWriteType: All
          IncludeManagementEvents: true
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao nên dùng Multi-Region Trail thay vì nhiều Single-Region Trails?

**Quản lý đơn giản hơn:** Một Trail, một S3 bucket, một policy. Không cần nhớ bật Trail ở mỗi region mới.

**Bảo mật tốt hơn:** Kẻ tấn công không thể thực hiện hành động ở region "orphan" không có Trail.

**Global services:** IAM, STS, Route53 chỉ ghi ở `us-east-1`. Multi-region Trail tự động bao gồm — single-region trail ở region khác sẽ bỏ sót.

**Chi phí tương đương:** Giá dựa trên số events, không phải số regions.

---

### Q2: Sau bao lâu thì event xuất hiện trong S3?

CloudTrail gửi log vào S3 **trong vòng 15 phút** sau khi event xảy ra. Thường sẽ sớm hơn (2–5 phút), nhưng SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) là 15 phút.

Nếu cần gần real-time hơn: dùng CloudWatch Logs (cùng latency 15 phút) kết hợp Metric Filters và Alarms.

---

### Q3: Log File Validation có thể phát hiện gì?

Phát hiện hai loại vấn đề:
1. **File bị sửa đổi** — Hash SHA-256 không khớp
2. **File bị xóa** — Hash chain bị gãy (digest file tham chiếu log file không tồn tại)

Không phát hiện được: Nếu kẻ tấn công **xóa cả digest file lẫn log file** đồng thời. Đó là lý do cần S3 Object Lock + MFA Delete.

---

### Q4: Làm thế nào phân tích CloudTrail logs hiệu quả?

**Phương pháp 1 — Athena (khuyên dùng cho scale lớn):**
```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
  eventVersion STRING,
  userIdentity STRUCT<...>,
  eventTime STRING,
  eventSource STRING,
  eventName STRING,
  ...
)
ROW FORMAT SERDE 'org.apache.hive.hcatalog.data.JsonSerDe'
LOCATION 's3://my-cloudtrail-bucket/AWSLogs/';
```

**Phương pháp 2 — CloudWatch Logs Insights (real-time, query đơn giản):**
```
fields eventTime, userIdentity.arn, eventName
| filter eventName = "DeleteBucket"
| sort eventTime desc
| limit 20
```

**Phương pháp 3 — Event History UI** (nhanh, ad-hoc, 90 ngày)

---

### Q5: Làm thế nào ngăn admin tắt CloudTrail?

**Cách 1 — SCP trong Organizations:**
Deny `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail` cho tất cả accounts.

**Cách 2 — Config Rule:**
Rule `cloud-trail-enabled` phát hiện khi CloudTrail bị tắt → trigger remediation tự động bật lại.

**Cách 3 — CloudWatch Alarm:**
Metric Filter trên `StopLogging` event → Alarm → SNS → PagerDuty alert security team.

**Kết hợp cả 3 là best practice cho production.**

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
**Liên Kết:** [1-event-types.md](./1-event-types.md) | [3-organization-trail.md](./3-organization-trail.md)
