# CloudTrail cho AWS Storage — Audit Trail và Điều Tra Bảo Mật

> AWS CloudTrail — Dịch Vụ Theo Dõi API — ghi lại mọi hành động lên AWS resources, cung cấp audit trail — dấu vết kiểm toán — thiết yếu cho bảo mật và compliance

---

## 📋 Tổng Quan

CloudTrail là dịch vụ governance — quản trị — tự động ghi lại:
- **Ai** (IAM user, role, service)
- **Đã làm gì** (API call)
- **Lúc nào** (timestamp)
- **Từ đâu** (IP address, user-agent)
- **Kết quả là gì** (success/failure)

```
Mọi hành động → CloudTrail Event → S3 bucket (log storage) → Phân tích
```

---

## 📁 Ba Loại CloudTrail Events

### 1. Management Events — Sự Kiện Quản Lý

Ghi lại các **management plane** operations — thao tác cấu hình:

```
Ví dụ:
  CreateBucket        → Tạo bucket mới
  DeleteBucket        → Xóa bucket
  PutBucketPolicy     → Thay đổi bucket policy
  PutBucketAcl        → Thay đổi ACL
  CreateVolume        → Tạo EBS volume
  DeleteVolume        → Xóa EBS volume
  ModifyVolume        → Thay đổi volume type/size
  CreateFileSystem    → Tạo EFS file system
  DeleteFileSystem    → Xóa EFS file system
```

| Đặc Điểm | Chi Tiết |
|-----------|---------|
| **Mặc định** | Bật sẵn, miễn phí (90 ngày trong Event History) |
| **Lưu trữ dài hạn** | Cần tạo Trail → gửi vào S3 (tốn phí lưu trữ) |
| **Chi phí** | Miễn phí cho management events |

### 2. Data Events — Sự Kiện Dữ Liệu

Ghi lại các **data plane** operations — thao tác với dữ liệu:

```
S3 Data Events:
  GetObject           → Tải xuống object
  PutObject           → Tải lên object
  DeleteObject        → Xóa object
  CopyObject          → Sao chép object
  HeadObject          → Đọc metadata
  GetObjectAcl        → Đọc ACL của object
  PutObjectAcl        → Thay đổi ACL của object

EBS Data Events:
  CreateSnapshot      → Tạo snapshot
  DeleteSnapshot      → Xóa snapshot
  CopySnapshot        → Sao chép snapshot

Lambda Data Events:
  Invoke              → Gọi Lambda function
```

| Đặc Điểm | Chi Tiết |
|-----------|---------|
| **Mặc định** | TẮT — phải bật thủ công |
| **Chi phí** | ~$0.10 per 100,000 events |
| **Volume** | Có thể rất nhiều — S3 bucket lớn = hàng triệu events/ngày |

### 3. CloudTrail Insights Events — Sự Kiện Bất Thường

CloudTrail Insights dùng Machine Learning — Học Máy — để phát hiện **bất thường trong API call patterns**:

```
Ví dụ phát hiện:
  - Đột ngột DeleteObject nhiều hơn bình thường 10x
  - Burst PutObject từ IP lạ
  - IAM role chưa bao giờ dùng bỗng gọi GetObject hàng loạt
```

| Đặc Điểm | Chi Tiết |
|-----------|---------|
| **Chi phí** | ~$0.35 per 100,000 management events được phân tích |
| **Ứng dụng** | Phát hiện data exfiltration, credential compromise |

---

## ⚙️ Cấu Hình CloudTrail cho Storage

### Tạo Trail Gửi Log Vào S3

```bash
# Bước 1: Tạo S3 bucket cho CloudTrail logs
aws s3api create-bucket \
  --bucket my-cloudtrail-logs-123456789012 \
  --region us-east-1

# Bước 2: Gắn bucket policy cho CloudTrail
aws s3api put-bucket-policy \
  --bucket my-cloudtrail-logs-123456789012 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AWSCloudTrailAclCheck",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:GetBucketAcl",
        "Resource": "arn:aws:s3:::my-cloudtrail-logs-123456789012"
      },
      {
        "Sid": "AWSCloudTrailWrite",
        "Effect": "Allow",
        "Principal": {"Service": "cloudtrail.amazonaws.com"},
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::my-cloudtrail-logs-123456789012/AWSLogs/*",
        "Condition": {
          "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
        }
      }
    ]
  }'

# Bước 3: Tạo Trail
aws cloudtrail create-trail \
  --name my-storage-audit-trail \
  --s3-bucket-name my-cloudtrail-logs-123456789012 \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

# Bước 4: Bắt đầu ghi log
aws cloudtrail start-logging \
  --name my-storage-audit-trail
```

### Bật S3 Data Events

```bash
# Bật Data Events cho bucket cụ thể
aws cloudtrail put-event-selectors \
  --trail-name my-storage-audit-trail \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3::Object",
          "Values": [
            "arn:aws:s3:::my-sensitive-bucket/",
            "arn:aws:s3:::my-data-bucket/"
          ]
        }
      ]
    }
  ]'

# Bật Data Events cho TẤT CẢ bucket (cẩn thận — tốn kém)
aws cloudtrail put-event-selectors \
  --trail-name my-storage-audit-trail \
  --event-selectors '[
    {
      "ReadWriteType": "WriteOnly",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::S3::Object",
          "Values": ["arn:aws:s3"]
        }
      ]
    }
  ]'
```

> **Tip chi phí:** Chỉ bật `WriteOnly` cho Data Events nếu muốn biết ai ghi/xóa. Bật `All` (bao gồm Read) tốn gấp 10-100x do lượng GetObject rất nhiều.

### Bật Advanced Event Selectors — Bộ Lọc Sự Kiện Nâng Cao

```bash
# Lọc chính xác hơn — chỉ ghi Delete events
aws cloudtrail put-event-selectors \
  --trail-name my-storage-audit-trail \
  --advanced-event-selectors '[
    {
      "Name": "S3-Delete-Events-Only",
      "FieldSelectors": [
        {"Field": "eventCategory", "Equals": ["Data"]},
        {"Field": "resources.type", "Equals": ["AWS::S3::Object"]},
        {"Field": "eventName", "Equals": ["DeleteObject", "DeleteObjects"]}
      ]
    }
  ]'
```

---

## 📋 Cấu Trúc CloudTrail Event

Mỗi event là một JSON object. Ví dụ event DeleteObject:

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDACKCEVSQ6C2EXAMPLE",
    "arn": "arn:aws:iam::123456789012:user/alice",
    "accountId": "123456789012",
    "userName": "alice"
  },
  "eventTime": "2026-05-16T10:23:45Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteObject",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.10",
  "userAgent": "aws-cli/2.13.0",
  "requestParameters": {
    "bucketName": "my-data-bucket",
    "key": "sensitive-data/customer-records.csv"
  },
  "responseElements": null,
  "requestID": "EXAMPLE123456789",
  "eventID": "a1234567-89ab-cdef-0123-456789abcdef",
  "readOnly": false,
  "resources": [
    {
      "ARN": "arn:aws:s3:::my-data-bucket/sensitive-data/customer-records.csv",
      "accountId": "123456789012",
      "type": "AWS::S3::Object"
    }
  ],
  "eventType": "AwsApiCall",
  "managementEvent": false,
  "recipientAccountId": "123456789012"
}
```

**Các trường quan trọng:**

| Trường | Ý Nghĩa |
|--------|---------|
| `userIdentity` | Ai đã thực hiện hành động |
| `eventTime` | Khi nào |
| `eventName` | Hành động gì (API call) |
| `sourceIPAddress` | Từ IP nào |
| `requestParameters` | Chi tiết request (bucket, key) |
| `errorCode` + `errorMessage` | Lý do thất bại (nếu có) |
| `readOnly` | `true` = chỉ đọc, `false` = ghi/xóa |

---

## 🔍 Điều Tra Bảo Mật với CloudTrail

### Kịch Bản 1: Tìm Ai Đã Xóa Object

```bash
# Dùng CloudTrail Event History (chỉ 90 ngày, quản lý events)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteObject \
  --start-time 2026-05-10T00:00:00Z \
  --end-time 2026-05-16T23:59:59Z \
  --query 'Events[*].{Time:EventTime,User:Username,Event:CloudTrailEvent}' \
  --output table
```

### Kịch Bản 2: Tìm Ai Đã Thay Đổi Bucket Policy

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutBucketPolicy \
  --query 'Events[*].{
    Time:EventTime,
    User:Username,
    SourceIP:CloudTrailEvent
  }' \
  --output json | jq '.[] | {time: .Time, user: .User}'
```

### Kịch Bản 3: Phân Tích với Athena

```sql
-- Tạo table Athena cho CloudTrail logs
CREATE EXTERNAL TABLE cloudtrail_logs (
  eventVersion STRING,
  userIdentity STRUCT<
    type: STRING,
    principalId: STRING,
    arn: STRING,
    accountId: STRING,
    userName: STRING,
    sessionContext: STRUCT<
      sessionIssuer: STRUCT<
        type: STRING,
        principalId: STRING,
        arn: STRING,
        accountId: STRING,
        userName: STRING
      >
    >
  >,
  eventTime STRING,
  eventSource STRING,
  eventName STRING,
  awsRegion STRING,
  sourceIPAddress STRING,
  userAgent STRING,
  errorCode STRING,
  errorMessage STRING,
  requestParameters STRING,
  responseElements STRING,
  requestId STRING,
  eventId STRING,
  readOnly BOOLEAN,
  resources ARRAY<STRUCT<
    arn: STRING,
    accountId: STRING,
    type: STRING
  >>,
  eventType STRING,
  managementEvent BOOLEAN,
  recipientAccountId STRING
)
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-logs-123456789012/AWSLogs/123456789012/CloudTrail/'
TBLPROPERTIES ('classification'='cloudtrail');
```

**Queries phân tích bảo mật:**

```sql
-- Tìm tất cả DeleteObject trong 7 ngày qua
SELECT
  eventtime,
  useridentity.arn AS requester,
  sourceipaddress,
  json_extract_scalar(requestparameters, '$.bucketName') AS bucket,
  json_extract_scalar(requestparameters, '$.key') AS object_key
FROM cloudtrail_logs
WHERE eventsource = 's3.amazonaws.com'
  AND eventname IN ('DeleteObject', 'DeleteObjects')
  AND eventtime > date_format(current_timestamp - interval '7' day, '%Y-%m-%dT%H:%i:%sZ')
ORDER BY eventtime DESC;
```

```sql
-- Phát hiện access từ IP lạ (không phải private network)
SELECT
  eventtime,
  useridentity.arn AS requester,
  sourceipaddress,
  eventname,
  json_extract_scalar(requestparameters, '$.bucketName') AS bucket
FROM cloudtrail_logs
WHERE eventsource = 's3.amazonaws.com'
  AND sourceipaddress NOT LIKE '10.%'
  AND sourceipaddress NOT LIKE '172.16.%'
  AND sourceipaddress NOT LIKE '192.168.%'
  AND sourceipaddress != 'AWS Internal'
  AND readonly = false
  AND eventtime > date_format(current_timestamp - interval '24' hour, '%Y-%m-%dT%H:%i:%sZ')
ORDER BY eventtime DESC;
```

```sql
-- Top 10 action thực hiện nhiều nhất
SELECT eventname, COUNT(*) AS count
FROM cloudtrail_logs
WHERE eventsource = 's3.amazonaws.com'
  AND eventtime > date_format(current_timestamp - interval '30' day, '%Y-%m-%dT%H:%i:%sZ')
GROUP BY eventname
ORDER BY count DESC
LIMIT 10;
```

```sql
-- Tìm failed attempts (lỗi authorization)
SELECT
  eventtime,
  useridentity.arn AS requester,
  sourceipaddress,
  eventname,
  errormessage
FROM cloudtrail_logs
WHERE eventsource = 's3.amazonaws.com'
  AND errorcode IN ('AccessDenied', 'AuthorizationError', 'InvalidClientTokenId')
  AND eventtime > date_format(current_timestamp - interval '24' hour, '%Y-%m-%dT%H:%i:%sZ')
ORDER BY eventtime DESC;
```

---

## 🛡️ CloudTrail Lake — Nền Tảng Phân Tích Sự Kiện Quản Lý

CloudTrail Lake là managed analytics platform — nền tảng phân tích được quản lý — mới hơn Athena + S3:

| Tính Năng | Athena + S3 | CloudTrail Lake |
|-----------|------------|-----------------|
| **Setup** | Phức tạp (tạo table, partition) | Đơn giản (1-click) |
| **Retention** — Lưu trữ | Tùy cấu hình S3 lifecycle | 90 ngày – 7 năm |
| **Query** | SQL chuẩn | SQL tương tự |
| **Chi phí query** | Per data scanned | Per data scanned |
| **Chi phí lưu trữ** | S3 Standard | Cao hơn S3 |
| **Integration** | Tự build | AWS Security Hub, Detective |

```bash
# Tạo Event Data Store trong CloudTrail Lake
aws cloudtrail create-event-data-store \
  --name "StorageAuditDataStore" \
  --retention-period 365 \
  --advanced-event-selectors '[
    {
      "Name": "S3-All-Events",
      "FieldSelectors": [
        {"Field": "eventSource", "Equals": ["s3.amazonaws.com"]}
      ]
    },
    {
      "Name": "EBS-Events",
      "FieldSelectors": [
        {"Field": "eventSource", "Equals": ["ec2.amazonaws.com"]},
        {"Field": "eventCategory", "Equals": ["Management"]}
      ]
    }
  ]'
```

---

## 🔔 CloudWatch Alarms từ CloudTrail Events

Kết hợp CloudTrail với CloudWatch Logs để tạo alarm real-time:

### Cấu Hình CloudTrail → CloudWatch Logs

```bash
# Cập nhật trail để gửi log vào CloudWatch Logs
aws cloudtrail update-trail \
  --name my-storage-audit-trail \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:CloudTrail/StorageAudit:* \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789012:role/CloudTrail-CloudWatch-Role
```

### Tạo Metric Filter — Bộ Lọc Chỉ Số

```bash
# Metric filter: đếm số lần DeleteBucket
aws logs put-metric-filter \
  --log-group-name CloudTrail/StorageAudit \
  --filter-name DeleteBucketCount \
  --filter-pattern '{ ($.eventName = "DeleteBucket") }' \
  --metric-transformations '[
    {
      "metricName": "DeleteBucketCount",
      "metricNamespace": "StorageAudit",
      "metricValue": "1",
      "defaultValue": 0
    }
  ]'

# Alarm khi bucket bị xóa
aws cloudwatch put-metric-alarm \
  --alarm-name "Alert-S3-BucketDeleted" \
  --alarm-description "Bucket S3 bị xóa — điều tra ngay" \
  --metric-name DeleteBucketCount \
  --namespace StorageAudit \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --statistic Sum \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:security-critical
```

### Các Metric Filter Khuyến Nghị cho Security

```bash
# 1. Bucket policy thay đổi
{ ($.eventName = "PutBucketPolicy") || ($.eventName = "DeleteBucketPolicy") }

# 2. Public access được bật
{ ($.eventName = "PutBucketAcl") && ($.requestParameters.accessControlList.grants[*].grantee.uri = "*") }

# 3. Encryption bị tắt
{ ($.eventName = "DeleteBucketEncryption") }

# 4. Replication bị xóa
{ ($.eventName = "DeleteBucketReplication") }

# 5. EBS volume bị xóa
{ ($.eventSource = "ec2.amazonaws.com") && ($.eventName = "DeleteVolume") }

# 6. Snapshot bị xóa
{ ($.eventSource = "ec2.amazonaws.com") && ($.eventName = "DeleteSnapshot") }

# 7. Root account activity
{ ($.userIdentity.type = "Root") && ($.userIdentity.invokedBy NOT EXISTS) && ($.eventType != "AwsServiceEvent") }
```

---

## 📊 CloudTrail vs Server Access Logging — Chọn Cái Nào?

```
Câu hỏi                          Công Cụ
─────────────────────────────────────────────────────────
Ai đã tạo/xóa bucket?            CloudTrail Management Events
Ai đã xóa file cụ thể?           CloudTrail Data Events
Request nào bị 403?               S3 Server Access Logging
Tổng bytes downloaded hôm nay?    S3 Server Access Logging
Bucket policy đã thay đổi khi nào? CloudTrail Management Events
IP nào đang scan bucket?          S3 Server Access Logging
Root account có đang dùng không?  CloudTrail Management Events
Latency request cụ thể là bao nhiêu? S3 Server Access Logging
```

**Kết hợp tối ưu:**
- Bật **CloudTrail Management Events** cho tất cả (miễn phí, 90 ngày)
- Tạo **Trail** gửi vào S3 cho lưu trữ dài hạn (> 90 ngày)
- Bật **Data Events** chỉ cho bucket nhạy cảm (tốn phí)
- Bật **Server Access Logging** cho tất cả bucket (rẻ, chi tiết)
- Dùng **CloudTrail Lake** nếu cần query nhanh không setup nhiều

---

## 📝 Câu Hỏi Phỏng Vấn

**Q: Khách hàng báo có file bị xóa nhưng không biết ai xóa. Làm thế nào điều tra?**
A: Đầu tiên kiểm tra CloudTrail — lookup-events với EventName=DeleteObject. Nếu chưa bật Data Events, kiểm tra S3 Server Access Logging (nếu đã bật). Trong tương lai, nên bật cả hai cho bucket quan trọng và enable S3 Versioning để recovery.

**Q: CloudTrail Event History lưu bao lâu và giới hạn gì?**
A: 90 ngày, chỉ Management Events, không thể tìm theo resource (chỉ tìm theo event name, resource name, user, IP). Để lưu lâu hơn hoặc query phức tạp hơn, cần tạo Trail gửi vào S3 và dùng Athena.

**Q: Bật CloudTrail Data Events cho tất cả S3 bucket có ảnh hưởng hiệu suất không?**
A: Không ảnh hưởng hiệu suất. Chỉ ảnh hưởng chi phí — $0.10 per 100,000 events. Bucket có 100M request/ngày → ~$100/ngày chỉ cho Data Events. Nên bật selective cho bucket quan trọng, không phải tất cả.

---

**Tiếp Theo:** [6-cost-anomaly-detection.md](6-cost-anomaly-detection.md) — Phát hiện bất thường chi phí và cảnh báo tự động

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
