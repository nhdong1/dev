# CloudTrail — Thiết Lập Nhật Ký Kiểm Toán

> **CloudTrail** ghi lại toàn bộ lời gọi API trong tài khoản AWS — là xương sống của mọi chiến lược bảo mật và kiểm toán.

---

## 🎯 CloudTrail Là Gì?

**AWS CloudTrail** là dịch vụ ghi nhật ký kiểm toán (audit logging) tự động ghi lại mọi hành động được thực hiện thông qua AWS Management Console, AWS CLI, AWS SDKs, và các dịch vụ AWS khác.

### Sự Kiện CloudTrail Ghi Lại Gì?

Mỗi bản ghi sự kiện (event record) trong CloudTrail chứa:

```json
{
  "eventTime": "2026-05-16T08:30:00Z",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDAEXAMPLE",
    "arn": "arn:aws:iam::123456789012:user/alice",
    "accountId": "123456789012",
    "userName": "alice"
  },
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteBucket",
  "awsRegion": "ap-southeast-1",
  "sourceIPAddress": "203.0.113.10",
  "userAgent": "aws-cli/2.x",
  "requestParameters": {
    "bucketName": "prod-sensitive-data"
  },
  "responseElements": null,
  "errorCode": null,
  "errorMessage": null
}
```

**Năm trường quan trọng nhất:**
1. `userIdentity` — Ai thực hiện hành động?
2. `eventName` — Hành động gì? (`CreateBucket`, `AssumeRole`, ...)
3. `sourceIPAddress` — Từ IP nào?
4. `eventTime` — Khi nào?
5. `errorCode` — Thành công hay thất bại?

---

## 📦 Các Loại Sự Kiện

### 1. Management Events — Sự Kiện Quản Lý

Ghi lại các thao tác **control plane** — tạo, sửa, xóa tài nguyên AWS.

```
Ví dụ:
✅ CreateBucket (tạo S3 bucket)
✅ RunInstances (khởi động EC2)
✅ AssumeRole (đảm nhận vai trò IAM)
✅ CreateUser (tạo IAM user)
✅ AuthorizeSecurityGroupIngress (mở port Security Group)
```

**Mặc định:** Được ghi lại tự động — không cần cấu hình thêm.

**Phân loại thêm:**
- **Read events** (Sự kiện đọc): `DescribeInstances`, `ListBuckets` — ít nhạy cảm hơn
- **Write events** (Sự kiện ghi): `TerminateInstances`, `DeleteBucket` — quan trọng cho audit

### 2. Data Events — Sự Kiện Dữ Liệu

Ghi lại các thao tác **data plane** — hoạt động trực tiếp trên dữ liệu trong tài nguyên.

```
Ví dụ:
📦 S3: GetObject, PutObject, DeleteObject
⚡ Lambda: Invoke
🔐 DynamoDB: GetItem, PutItem
🔑 KMS: Decrypt, GenerateDataKey
```

**Mặc định:** KHÔNG được ghi — phải bật thủ công (tốn thêm chi phí).

**Khi nào cần bật Data Events?**
- S3 bucket chứa dữ liệu nhạy cảm (PII, tài chính)
- Lambda function xử lý payment hoặc authentication
- KMS keys dùng trong môi trường regulated (PCI-DSS, HIPAA)

### 3. Insights Events — Sự Kiện Bất Thường

**CloudTrail Insights** — Bất Thường CloudTrail — tự động phát hiện hoạt động API bất thường so với baseline.

```
Ví dụ phát hiện:
⚠️  CreateSecurityGroup được gọi 50 lần trong 5 phút (bình thường: 2 lần/ngày)
⚠️  TerminateInstances spike lớn
⚠️  DeleteBucket được gọi hàng loạt
```

**Mặc định:** KHÔNG được bật — tốn thêm $0.35/100,000 events phân tích.

---

## ⚙️ Cấu Hình CloudTrail Trail

### Trail Là Gì?

**Trail** (Vết Kiểm Toán) là cấu hình chỉ định CloudTrail ghi sự kiện nào và gửi đến đâu.

### Hai Loại Trail

| Loại | Phạm vi | Khuyến nghị |
|---|---|---|
| **Single-region trail** | Chỉ một region | ❌ Không đủ cho production |
| **Multi-region trail** | Tất cả regions hiện tại và tương lai | ✅ Bắt buộc |

> **Tại sao cần multi-region?** Kẻ tấn công thường hoạt động ở region ít được giám sát. Multi-region trail bắt được mọi hành động dù ở region nào.

### Tạo Trail Chuẩn Qua CLI

```bash
# Bước 1: Tạo S3 bucket để lưu logs
aws s3api create-bucket \
  --bucket my-cloudtrail-logs-123456789012 \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Bước 2: Bật versioning và MFA Delete trên bucket
aws s3api put-bucket-versioning \
  --bucket my-cloudtrail-logs-123456789012 \
  --versioning-configuration \
    Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/root-device 123456"

# Bước 3: Tạo trail multi-region với log validation
aws cloudtrail create-trail \
  --name production-audit-trail \
  --s3-bucket-name my-cloudtrail-logs-123456789012 \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --include-global-service-events \
  --cloud-watch-logs-log-group-arn arn:aws:logs:ap-southeast-1:123456789012:log-group:CloudTrail/logs \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrailRole

# Bước 4: Bật trail
aws cloudtrail start-logging --name production-audit-trail

# Bước 5: (Tuỳ chọn) Bật Insights
aws cloudtrail put-insight-selectors \
  --trail-name production-audit-trail \
  --insight-selectors '[{"InsightType": "ApiCallRateInsight"}, {"InsightType": "ApiErrorRateInsight"}]'
```

### Tạo Trail Bằng Terraform

```hcl
# S3 bucket lưu CloudTrail logs
resource "aws_s3_bucket" "cloudtrail_logs" {
  bucket        = "cloudtrail-logs-${data.aws_caller_identity.current.account_id}"
  force_destroy = false
}

resource "aws_s3_bucket_versioning" "cloudtrail_logs" {
  bucket = aws_s3_bucket.cloudtrail_logs.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Chặn public access
resource "aws_s3_bucket_public_access_block" "cloudtrail_logs" {
  bucket = aws_s3_bucket.cloudtrail_logs.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Server-side encryption với KMS
resource "aws_s3_bucket_server_side_encryption_configuration" "cloudtrail_logs" {
  bucket = aws_s3_bucket.cloudtrail_logs.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.cloudtrail.arn
    }
  }
}

# CloudTrail trail
resource "aws_cloudtrail" "main" {
  name                          = "production-audit-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id
  is_multi_region_trail         = true
  enable_log_file_validation    = true   # Quan trọng: bật log integrity
  include_global_service_events = true   # Bao gồm IAM, STS, Route53

  # Gửi logs vào CloudWatch Logs để alert realtime
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cloudwatch.arn

  # Mã hóa logs bằng KMS
  kms_key_id = aws_kms_key.cloudtrail.arn

  # Bật CloudTrail Insights
  insight_selector {
    insight_type = "ApiCallRateInsight"
  }
  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }

  # Bật Data Events cho S3 bucket nhạy cảm
  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::prod-sensitive-data/"]
    }
  }

  tags = {
    Environment = "production"
    Purpose     = "security-audit"
  }
}
```

---

## 🔒 Bảo Vệ Tính Toàn Vẹn Của Log

### Log File Validation — Xác Thực File Log

**Log File Validation** (Xác Thực File Log) là tính năng CloudTrail tạo **digest file** (file tóm tắt) chứa hash SHA-256 của mỗi log file. Nếu log bị sửa đổi hoặc xóa, việc xác thực sẽ phát hiện ra.

```
Cách hoạt động:
1. CloudTrail ghi log file vào S3
2. Mỗi giờ, CloudTrail tạo digest file chứa SHA-256 hash
3. Digest file được ký bằng private key của AWS
4. Bạn xác thực bằng: aws cloudtrail validate-logs
```

**Kiểm tra tính toàn vẹn:**

```bash
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:ap-southeast-1:123456789012:trail/production-audit-trail \
  --start-time "2026-05-16T00:00:00Z" \
  --end-time "2026-05-16T23:59:59Z" \
  --verbose
```

Kết quả mẫu:
```
Validating log files for trail arn:aws:cloudtrail:...
2026/05/16T00:30:15Z: Valid
2026/05/16T01:30:22Z: Valid
2026/05/16T02:30:19Z: INVALID (file has been modified or deleted)
```

### Bảo Vệ S3 Bucket Chứa Log

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteLogs",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-*/AWSLogs/*"
    },
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutBucketAcl",
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalAccount": "123456789012"
        }
      }
    },
    {
      "Sid": "AllowCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-*/AWSLogs/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
```

**Các biện pháp bảo vệ bucket:**

| Biện pháp | Mục đích |
|---|---|
| MFA Delete | Xóa object yêu cầu MFA — ngăn xóa vô tình hoặc bị tấn công |
| Bucket Versioning | Khôi phục file bị ghi đè |
| S3 Object Lock | Ngăn chặn xóa trong thời gian retention |
| Bucket Policy Deny Delete | Từ chối mọi nỗ lực xóa log |
| Block Public Access | Ngăn bucket trở thành public |
| Server-side Encryption | Mã hóa log bằng KMS |

---

## 🏊 CloudTrail Lake — Lưu Trữ Và Truy Vấn Dài Hạn

### CloudTrail Lake Là Gì?

**CloudTrail Lake** (Hồ Sự Kiện CloudTrail) là managed data lake (kho dữ liệu được quản lý) cho phép lưu trữ, tổng hợp, và truy vấn sự kiện CloudTrail bằng SQL — không cần Athena hay Glue.

```
Truyền thống:   CloudTrail → S3 → Glue → Athena → Query
CloudTrail Lake: CloudTrail → Lake → SQL Query (trực tiếp)
```

### Ưu Điểm CloudTrail Lake

| Tính năng | CloudTrail + Athena | CloudTrail Lake |
|---|---|---|
| **Cài đặt** | Phức tạp (Glue catalog, Athena) | Đơn giản (vài click) |
| **Retention** | Tùy theo S3 lifecycle | 7 năm (configurable) |
| **Query** | SQL qua Athena | SQL tích hợp sẵn |
| **Federation** | Không | ✅ Import từ on-premises |
| **Chi phí** | Thấp hơn | Cao hơn (~$2.75/GB) |

### Tạo Event Data Store (Kho Dữ Liệu Sự Kiện)

```bash
# Tạo CloudTrail Lake Event Data Store
aws cloudtrail create-event-data-store \
  --name "SecurityAuditLake" \
  --retention-period 2557 \       # 7 năm (2557 ngày)
  --termination-protection-enabled \
  --multi-region-enabled \
  --organization-enabled \         # Bao gồm tất cả accounts trong Org
  --advanced-event-selectors '[
    {
      "Name": "AllManagementEvents",
      "FieldSelectors": [
        {"Field": "eventCategory", "Equals": ["Management"]}
      ]
    }
  ]'
```

### Query Mẫu Trên CloudTrail Lake

```sql
-- Tìm tất cả login thất bại trong 24 giờ qua
SELECT
  eventTime,
  userIdentity.userName,
  sourceIPAddress,
  errorCode,
  errorMessage
FROM cloudtrail_logs
WHERE
  eventName = 'ConsoleLogin'
  AND errorCode = 'Failed authentication'
  AND eventTime >= DATE_ADD('hour', -24, NOW())
ORDER BY eventTime DESC;

-- Phát hiện ai đã tắt CloudTrail (hành động đáng ngờ)
SELECT
  eventTime,
  userIdentity.arn,
  userIdentity.sessionContext.sessionIssuer.arn AS assumedRole,
  sourceIPAddress
FROM cloudtrail_logs
WHERE
  eventName IN ('StopLogging', 'DeleteTrail', 'UpdateTrail')
ORDER BY eventTime DESC;

-- Liệt kê tất cả IAM changes trong tuần qua
SELECT
  eventTime,
  eventName,
  userIdentity.arn AS actor,
  requestParameters
FROM cloudtrail_logs
WHERE
  eventSource = 'iam.amazonaws.com'
  AND eventName LIKE '%Policy%'
  AND eventTime >= DATE_ADD('day', -7, NOW())
ORDER BY eventTime DESC;
```

---

## 🏢 Cấu Hình CloudTrail Cho Môi Trường Đa Tài Khoản

### Kiến Trúc Centralized Logging

```
┌─────────────────────────────────────────────────────┐
│              AWS Organizations                       │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ Account  │  │ Account  │  │ Account  │           │
│  │   Dev    │  │  Staging │  │   Prod   │           │
│  │ CloudTrl │  │ CloudTrl │  │ CloudTrl │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │              │              │                │
│       └──────────────┴──────────────┘                │
│                      │                              │
│                      ▼                              │
│          ┌─────────────────────┐                    │
│          │    Log Archive      │                    │
│          │      Account        │                    │
│          │  (Central S3 Logs) │                    │
│          └─────────────────────┘                    │
└─────────────────────────────────────────────────────┘
```

### Organization Trail — Trail Toàn Tổ Chức

```bash
# Tạo Organization Trail từ management account
aws cloudtrail create-trail \
  --name org-audit-trail \
  --s3-bucket-name centralized-logs-archive-account \
  --is-organization-trail \          # Bao gồm tất cả member accounts
  --is-multi-region-trail \
  --enable-log-file-validation
```

**Quyền lợi của Organization Trail:**
- Một trail duy nhất bao phủ tất cả accounts
- Member accounts không thể tắt hoặc sửa trail từ management account
- Logs từ tất cả accounts gom về một S3 bucket trung tâm

---

## 🚨 CloudTrail Logs → CloudWatch Alarms Realtime

### Cấu Hình Cảnh Báo Bảo Mật Cơ Bản

```bash
# Tạo log group
aws logs create-log-group --log-group-name CloudTrail/SecurityAlerts

# Metric filter: Root account được dùng
aws logs put-metric-filter \
  --log-group-name CloudTrail/SecurityAlerts \
  --filter-name RootAccountUsage \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName=RootAccountUsageCount,metricNamespace=SecurityMetrics,metricValue=1

# Tạo alarm khi root account được dùng
aws cloudwatch put-metric-alarm \
  --alarm-name "RootAccountUsage" \
  --alarm-description "Root account được sử dụng - hành động đáng ngờ" \
  --metric-name RootAccountUsageCount \
  --namespace SecurityMetrics \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:SecurityAlerts
```

**Bộ cảnh báo security cơ bản cần thiết lập:**

| Tên Alarm | Filter Pattern | Mức Độ Nghiêm Trọng |
|---|---|---|
| Root account login | `$.userIdentity.type = "Root"` | 🔴 Critical |
| CloudTrail bị tắt | `$.eventName = "StopLogging"` | 🔴 Critical |
| Security Group thay đổi | `$.eventName = "AuthorizeSecurityGroupIngress"` | 🟡 Warning |
| MFA bị tắt | `$.eventName = "DeactivateMFADevice"` | 🟡 Warning |
| Policy mới được tạo | `$.eventName = "CreatePolicy"` | 🟡 Warning |
| Console login thất bại (>=5 lần) | `$.eventName = "ConsoleLogin" && $.errorMessage = "Failed"` | 🟡 Warning |
| KMS key bị xóa | `$.eventName = "DeleteAlias"` | 🟡 Warning |
| S3 bucket public access bật | `$.eventName = "PutBucketAcl"` | 🟠 High |

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Tại sao nên bật `enable-log-file-validation` trên CloudTrail?**

> Log file validation tạo digest file chứa SHA-256 hash của mỗi log file, được ký bởi AWS. Nếu kẻ tấn công cố xóa hoặc sửa log sau khi xâm phạm account, bạn có thể phát hiện ra bằng cách validate digest. Không có tính năng này, bạn không thể chứng minh log chưa bị giả mạo — quan trọng cho forensics và compliance.

**Q: Sự khác biệt giữa CloudTrail và VPC Flow Logs?**

> CloudTrail ghi **API calls** — ai gọi API nào tới dịch vụ AWS. VPC Flow Logs ghi **network traffic metadata** — IP nào kết nối tới IP nào, port nào, bytes bao nhiêu. Để điều tra bảo mật toàn diện, cần cả hai: CloudTrail để biết hành động quản lý, VPC Flow Logs để biết network communication.

**Q: Khi nào nên bật Data Events, và rủi ro không bật là gì?**

> Bật Data Events cho S3 buckets chứa dữ liệu nhạy cảm (credentials, PII, payment data) và Lambda functions xử lý authentication. Không bật: bạn không biết ai đã đọc/ghi object nào trong S3 — kẻ tấn công có thể exfiltrate (rò rỉ) dữ liệu mà không để lại dấu vết trong logs.

**Q: Làm sao ngăn SOC analyst xóa CloudTrail logs để che giấu hành động sai trái của họ?**

> Dùng S3 Object Lock với WORM (Write Once Read Many) compliance mode, kết hợp với MFA Delete. Gửi logs vào Log Archive account riêng, cấp quyền ghi cho CloudTrail service nhưng không ai trong tổ chức có quyền xóa. Dùng AWS Organizations SCPs để ngăn member accounts tắt CloudTrail.

---

**Tiếp Theo:** [2-cloudtrail-analysis.md](2-cloudtrail-analysis.md) — Phân tích log và phát hiện bất thường

---

**Cập Nhật Lần Cuối:** 2026-05-16
