# CloudTrail Analysis — Phân Tích Log Và Phát Hiện Bất Thường

> Biết cách thu thập log chỉ là bước đầu — phân tích hiệu quả mới tạo ra giá trị bảo mật thực sự.

---

## 🎯 Tại Sao Phân Tích CloudTrail Log?

CloudTrail log chứa hàng triệu sự kiện mỗi ngày. Không phân tích thì log chỉ là kho lưu trữ thụ động. Mục tiêu phân tích:

1. **Phát hiện sự cố** (Incident Detection) — tìm hành vi bất thường trước khi gây hại
2. **Điều tra sau sự cố** (Post-incident Investigation) — truy vết ai làm gì khi nào
3. **Tuân thủ kiểm toán** (Compliance Audit) — chứng minh kiểm soát truy cập hoạt động
4. **Phân tích xu hướng** (Trend Analysis) — phát hiện leo thang đặc quyền dần dần

---

## 🔍 Phân Tích Với Amazon Athena

### Kiến Trúc Athena + CloudTrail

```
CloudTrail Logs (S3)
        │
        ▼
  AWS Glue Crawler
  (tạo schema tự động)
        │
        ▼
   Glue Data Catalog
   (metadata / schema)
        │
        ▼
  Amazon Athena
  (SQL query engine)
        │
        ▼
   Kết quả phân tích
```

### Thiết Lập Glue Crawler Cho CloudTrail Logs

```bash
# Tạo Glue database
aws glue create-database \
  --database-input '{"Name": "cloudtrail_analysis"}'

# Tạo crawler để tự động nhận dạng schema CloudTrail
aws glue create-crawler \
  --name cloudtrail-log-crawler \
  --role arn:aws:iam::123456789012:role/GlueCrawlerRole \
  --database-name cloudtrail_analysis \
  --targets '{"S3Targets": [{"Path": "s3://my-cloudtrail-logs/AWSLogs/"}]}' \
  --schedule 'cron(0 2 * * ? *)'  # Chạy lúc 2AM mỗi ngày

# Chạy crawler lần đầu
aws glue start-crawler --name cloudtrail-log-crawler
```

### Tạo Bảng Athena Cho CloudTrail (Thủ Công)

```sql
CREATE EXTERNAL TABLE cloudtrail_logs (
    eventVersion STRING,
    userIdentity STRUCT<
        type: STRING,
        principalId: STRING,
        arn: STRING,
        accountId: STRING,
        invokedBy: STRING,
        accessKeyId: STRING,
        userName: STRING,
        sessionContext: STRUCT<
            attributes: STRUCT<
                mfaAuthenticated: STRING,
                creationDate: STRING>,
            sessionIssuer: STRUCT<
                type: STRING,
                principalId: STRING,
                arn: STRING,
                accountId: STRING,
                userName: STRING>>>,
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
    resources ARRAY<STRUCT<
        arn: STRING,
        accountId: STRING,
        type: STRING>>,
    eventType STRING,
    apiVersion STRING,
    readOnly BOOLEAN,
    recipientAccountId STRING,
    managementEvent BOOLEAN,
    annotations ARRAY<STRUCT<content: STRING>>
)
PARTITIONED BY (region STRING, year STRING, month STRING, day STRING)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-logs/AWSLogs/123456789012/CloudTrail/';

-- Nạp partitions (phân vùng) vào bảng
MSCK REPAIR TABLE cloudtrail_logs;
```

---

## 📊 Bộ Câu Hỏi Athena Cần Nhớ

### Nhóm 1: Phát Hiện Xâm Phạm Tài Khoản

```sql
-- Tìm tất cả ConsoleLogin (đăng nhập Console) không dùng MFA
SELECT
    eventTime,
    userIdentity.userName,
    sourceIPAddress,
    userAgent,
    additionalEventData
FROM cloudtrail_logs
WHERE
    eventName = 'ConsoleLogin'
    AND JSON_EXTRACT_SCALAR(additionalEventData, '$.MFAUsed') = 'No'
    AND year = '2026' AND month = '05'
ORDER BY eventTime DESC;
```

```sql
-- Phát hiện đăng nhập từ nhiều IP trong cùng 1 giờ (credential stuffing)
SELECT
    userIdentity.userName,
    COUNT(DISTINCT sourceIPAddress) AS unique_ips,
    COUNT(*) AS total_logins,
    MIN(eventTime) AS first_login,
    MAX(eventTime) AS last_login
FROM cloudtrail_logs
WHERE
    eventName = 'ConsoleLogin'
    AND errorCode IS NULL
    AND year = '2026' AND month = '05' AND day = '16'
GROUP BY userIdentity.userName
HAVING COUNT(DISTINCT sourceIPAddress) > 3
ORDER BY unique_ips DESC;
```

```sql
-- Phát hiện AssumeRole từ IP lạ (không phải IP công ty)
SELECT
    eventTime,
    userIdentity.arn,
    sourceIPAddress,
    requestParameters
FROM cloudtrail_logs
WHERE
    eventName = 'AssumeRole'
    AND sourceIPAddress NOT LIKE '203.0.113.%'   -- IP của công ty
    AND sourceIPAddress NOT LIKE '10.%'           -- Private network
    AND year = '2026' AND month = '05'
ORDER BY eventTime DESC;
```

### Nhóm 2: Phát Hiện Hoạt Động Đặc Quyền

```sql
-- Các hành động IAM được thực hiện trong 7 ngày qua
SELECT
    eventTime,
    eventName,
    userIdentity.arn AS actor,
    userIdentity.sessionContext.sessionIssuer.arn AS assumed_role,
    sourceIPAddress
FROM cloudtrail_logs
WHERE
    eventSource = 'iam.amazonaws.com'
    AND eventName IN (
        'CreateUser', 'DeleteUser',
        'CreateRole', 'DeleteRole',
        'AttachRolePolicy', 'DetachRolePolicy',
        'PutRolePolicy', 'DeleteRolePolicy',
        'CreateAccessKey', 'DeleteAccessKey',
        'UpdateLoginProfile'
    )
    AND year = '2026' AND month = '05'
ORDER BY eventTime DESC;
```

```sql
-- Tìm tất cả lần dùng access key của IAM user bị terminate
-- (trường hợp offboarding không xóa access key)
SELECT
    eventTime,
    userIdentity.accessKeyId,
    userIdentity.arn,
    eventName,
    sourceIPAddress
FROM cloudtrail_logs
WHERE
    userIdentity.accessKeyId IN (
        'AKIAIOSFODNN7EXAMPLE',
        'AKIAI44QH8DHBEXAMPLE'
    )
ORDER BY eventTime DESC
LIMIT 100;
```

### Nhóm 3: Phát Hiện Data Exfiltration (Rò Rỉ Dữ Liệu)

```sql
-- S3 GetObject từ IP ngoài (Data Events phải được bật)
SELECT
    eventTime,
    userIdentity.arn,
    sourceIPAddress,
    JSON_EXTRACT_SCALAR(requestParameters, '$.bucketName') AS bucket,
    JSON_EXTRACT_SCALAR(requestParameters, '$.key') AS object_key
FROM cloudtrail_logs
WHERE
    eventName = 'GetObject'
    AND eventSource = 's3.amazonaws.com'
    AND sourceIPAddress NOT LIKE '10.%'
    AND sourceIPAddress NOT LIKE '172.16.%'
    AND sourceIPAddress NOT LIKE '192.168.%'
    AND year = '2026' AND month = '05'
ORDER BY eventTime DESC
LIMIT 1000;
```

```sql
-- Phát hiện ai tạo S3 presigned URL (có thể chia sẻ data ra ngoài)
SELECT
    eventTime,
    userIdentity.arn,
    sourceIPAddress,
    requestParameters
FROM cloudtrail_logs
WHERE
    eventName = 'GetObject'
    AND userAgent LIKE '%AWSSDKGoV2%'      -- SDK thường dùng cho presign
    AND sourceIPAddress NOT IN (
        SELECT DISTINCT sourceIPAddress
        FROM cloudtrail_logs
        WHERE eventName = 'ConsoleLogin'   -- IPs đã từng đăng nhập console
    )
ORDER BY eventTime DESC;
```

### Nhóm 4: Phát Hiện Tấn Công Và Trinh Sát

```sql
-- Phát hiện reconnaissance (trinh sát) — gọi Describe/List nhiều dịch vụ
SELECT
    userIdentity.arn,
    eventSource,
    COUNT(*) AS api_call_count,
    COUNT(DISTINCT eventName) AS distinct_operations
FROM cloudtrail_logs
WHERE
    eventName LIKE 'Describe%'
    OR eventName LIKE 'List%'
    OR eventName LIKE 'Get%'
    AND readOnly = TRUE
    AND year = '2026' AND month = '05' AND day = '16'
GROUP BY userIdentity.arn, eventSource
HAVING COUNT(*) > 100
ORDER BY api_call_count DESC;
```

```sql
-- Phát hiện privilege escalation attempt (cố leo thang quyền)
-- Khi entity gọi nhiều API tạo/sửa policy liên tiếp
SELECT
    userIdentity.arn,
    MIN(eventTime) AS start_time,
    MAX(eventTime) AS end_time,
    COUNT(*) AS privilege_actions,
    ARRAY_AGG(DISTINCT eventName) AS actions_taken
FROM cloudtrail_logs
WHERE
    eventName IN (
        'CreatePolicy', 'CreatePolicyVersion',
        'SetDefaultPolicyVersion',
        'PutUserPolicy', 'PutGroupPolicy', 'PutRolePolicy',
        'AttachUserPolicy', 'AttachGroupPolicy', 'AttachRolePolicy',
        'AddUserToGroup', 'CreateGroup',
        'UpdateAssumeRolePolicy'
    )
    AND year = '2026' AND month = '05' AND day = '16'
GROUP BY userIdentity.arn
HAVING COUNT(*) >= 3
ORDER BY privilege_actions DESC;
```

### Nhóm 5: Kiểm Toán Compliance (Tuân Thủ)

```sql
-- Kiểm tra tất cả thay đổi Security Group trong 30 ngày
SELECT
    eventTime,
    userIdentity.arn,
    eventName,
    awsRegion,
    JSON_EXTRACT_SCALAR(requestParameters, '$.groupId') AS security_group_id,
    requestParameters
FROM cloudtrail_logs
WHERE
    eventName IN (
        'AuthorizeSecurityGroupIngress',
        'AuthorizeSecurityGroupEgress',
        'RevokeSecurityGroupIngress',
        'RevokeSecurityGroupEgress',
        'CreateSecurityGroup',
        'DeleteSecurityGroup'
    )
    AND year = '2026' AND month = '05'
ORDER BY eventTime DESC;
```

```sql
-- Tổng hợp báo cáo hàng tháng: Top 20 API operations
SELECT
    eventName,
    eventSource,
    COUNT(*) AS call_count,
    COUNT(DISTINCT userIdentity.arn) AS unique_actors
FROM cloudtrail_logs
WHERE
    year = '2026' AND month = '05'
    AND errorCode IS NULL
GROUP BY eventName, eventSource
ORDER BY call_count DESC
LIMIT 20;
```

---

## 🔍 CloudWatch Logs Insights — Truy Vấn Realtime

**CloudWatch Logs Insights** (Phân Tích Log CloudWatch) cho phép truy vấn log theo thời gian thực bằng ngôn ngữ query riêng của CloudWatch — đơn giản hơn SQL nhưng đủ mạnh cho monitoring.

### Syntax Cơ Bản

```
fields @timestamp, @message
| filter eventName = "ConsoleLogin"
| filter errorMessage like "Failed"
| stats count(*) as failures by userIdentity.userName
| sort failures desc
| limit 20
```

### Queries Hữu Ích

```
# Top 10 IAM users gây ra lỗi nhất (có thể bị brute force)
fields @timestamp, userIdentity.userName, errorCode, sourceIPAddress
| filter errorCode like "AccessDenied"
| stats count(*) as denied_count by userIdentity.userName, sourceIPAddress
| sort denied_count desc
| limit 10
```

```
# Tìm tất cả thay đổi CloudTrail (ai đang cố che giấu?)
fields @timestamp, userIdentity.arn, eventName, sourceIPAddress
| filter eventName in ["StopLogging", "DeleteTrail", "UpdateTrail", "PutEventSelectors"]
| sort @timestamp desc
```

```
# Phát hiện API calls từ Tor exit nodes hoặc VPN đáng ngờ
fields @timestamp, userIdentity.arn, sourceIPAddress, eventName
| filter sourceIPAddress like "185.220."  # Dải IP Tor thường gặp
| stats count(*) as calls by sourceIPAddress, userIdentity.arn
| sort calls desc
```

```
# Theo dõi usage của temporary credentials (STS)
fields @timestamp, userIdentity.sessionContext.sessionIssuer.userName,
       userIdentity.accessKeyId, eventName, awsRegion
| filter userIdentity.type = "AssumedRole"
| stats count(*) as operations by userIdentity.sessionContext.sessionIssuer.userName, awsRegion
| sort operations desc
```

---

## 🤖 CloudTrail Insights — Phát Hiện Bất Thường Tự Động

**CloudTrail Insights** dùng machine learning (học máy) để phát hiện hoạt động API bất thường so với baseline lịch sử 7 ngày.

### Cách Hoạt Động

```
Bước 1: CloudTrail Insights học baseline
        (volume API calls bình thường trong 7 ngày đầu)

Bước 2: Theo dõi liên tục
        (so sánh rate hiện tại với baseline)

Bước 3: Phát hiện bất thường
        (rate cao hơn baseline đáng kể = Insights Event)

Bước 4: Ghi Insights Event vào S3 và CloudWatch
        (event riêng biệt, không lẫn với management events)
```

### Hai Loại Insights

| Loại | Phát hiện | Ví dụ |
|---|---|---|
| **ApiCallRateInsight** | Tốc độ gọi API tăng đột biến | `TerminateInstances` 100 lần trong 5 phút thay vì 2 lần/ngày |
| **ApiErrorRateInsight** | Tốc độ lỗi API tăng đột biến | Nhiều `AccessDenied` — có thể đang brute force permissions |

### Bật CloudTrail Insights

```bash
aws cloudtrail put-insight-selectors \
  --trail-name production-audit-trail \
  --insight-selectors \
    '[
      {"InsightType": "ApiCallRateInsight"},
      {"InsightType": "ApiErrorRateInsight"}
    ]'
```

### Xử Lý Insights Events Qua EventBridge

```json
{
  "source": ["aws.cloudtrail"],
  "detail-type": ["AWS Insight via CloudTrail"],
  "detail": {
    "eventType": ["AwsCloudTrailInsight"]
  }
}
```

EventBridge rule này kích hoạt Lambda để:
1. Phân tích loại bất thường
2. Gửi Slack/PagerDuty alert
3. Tự động tạo ticket trong Jira/ServiceNow

---

## 🧩 Phân Tích Sự Cố Thực Tế — Case Study

### Kịch Bản: Phát Hiện Credential Compromise (Tài Khoản Bị Xâm Phạm)

**Tình huống:** GuardDuty cảnh báo `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration`

**Quy trình điều tra với Athena:**

```sql
-- Bước 1: Xác định access key bị xâm phạm và lịch sử sử dụng
SELECT
    eventTime,
    eventName,
    eventSource,
    sourceIPAddress,
    awsRegion,
    errorCode
FROM cloudtrail_logs
WHERE
    userIdentity.accessKeyId = 'ASIAIOSFODNN7EXAMPLE'   -- Key từ GuardDuty finding
ORDER BY eventTime ASC;
```

```sql
-- Bước 2: Tìm pattern địa lý bất thường
SELECT
    sourceIPAddress,
    COUNT(*) AS call_count,
    MIN(eventTime) AS first_seen,
    MAX(eventTime) AS last_seen,
    ARRAY_AGG(DISTINCT eventName) AS operations
FROM cloudtrail_logs
WHERE
    userIdentity.accessKeyId = 'ASIAIOSFODNN7EXAMPLE'
GROUP BY sourceIPAddress
ORDER BY call_count DESC;
```

```sql
-- Bước 3: Xem kẻ tấn công đã làm gì (tìm persistence mechanisms)
SELECT
    eventTime,
    eventName,
    requestParameters,
    responseElements,
    sourceIPAddress
FROM cloudtrail_logs
WHERE
    userIdentity.accessKeyId = 'ASIAIOSFODNN7EXAMPLE'
    AND eventName IN (
        -- Persistence qua IAM
        'CreateUser', 'CreateAccessKey',
        'AttachRolePolicy', 'PutRolePolicy',
        -- Persistence qua EC2
        'CreateKeyPair', 'ImportKeyPair',
        'RunInstances',
        -- Data access
        'GetObject', 'ListBuckets',
        -- Hiding tracks
        'StopLogging', 'DeleteTrail'
    )
ORDER BY eventTime;
```

```sql
-- Bước 4: Kiểm tra có resource nào được tạo ra không (cần dọn dẹp)
SELECT
    eventTime,
    eventName,
    responseElements,
    awsRegion
FROM cloudtrail_logs
WHERE
    userIdentity.accessKeyId = 'ASIAIOSFODNN7EXAMPLE'
    AND eventName LIKE 'Create%'
    AND errorCode IS NULL
ORDER BY eventTime;
```

### Timeline Phân Tích (Mục Tiêu: < 30 Phút)

```
T+0m:  GuardDuty finding xuất hiện
T+2m:  Xác định access key từ finding
T+5m:  Chạy query Bước 1 — xem toàn bộ activity
T+10m: Chạy query Bước 2 — xác định IPs bất thường
T+15m: Chạy query Bước 3 — tìm persistence mechanisms
T+20m: Chạy query Bước 4 — liệt kê resources cần dọn
T+25m: Revoke access key, khởi động remediation
T+30m: Báo cáo sơ bộ cho incident commander
```

---

## 📈 Dashboard Phân Tích Với QuickSight

**Amazon QuickSight** có thể kết nối trực tiếp với Athena để tạo dashboard visual từ CloudTrail logs.

### Dataset Hữu Ích Cho Security Dashboard

```sql
-- Dataset: API Errors Hourly (cho biểu đồ xu hướng lỗi)
SELECT
    DATE_TRUNC('hour', CAST(eventTime AS TIMESTAMP)) AS hour,
    errorCode,
    COUNT(*) AS error_count
FROM cloudtrail_logs
WHERE
    errorCode IS NOT NULL
    AND year = '2026' AND month = '05'
GROUP BY 1, 2
ORDER BY 1;
```

```sql
-- Dataset: Geographic Distribution of API Calls
SELECT
    sourceIPAddress,
    COUNT(*) AS call_count,
    COUNT(DISTINCT userIdentity.arn) AS unique_users
FROM cloudtrail_logs
WHERE
    year = '2026' AND month = '05'
GROUP BY sourceIPAddress
ORDER BY call_count DESC
LIMIT 100;
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Làm thế nào phát hiện một IAM user đang thực hiện privilege escalation (leo thang quyền)?**

> Dùng Athena query tìm user thực hiện nhiều IAM actions trong khoảng thời gian ngắn — đặc biệt là `CreatePolicy`, `AttachRolePolicy`, `PutRolePolicy`, `UpdateAssumeRolePolicy`. Kết hợp với CloudTrail Insights để phát hiện spike trong `ApiCallRateInsight`. Đặt CloudWatch Alarm cho pattern này và tích hợp với GuardDuty để có context đầy đủ hơn.

**Q: CloudTrail Lake vs Athena + S3 — khi nào chọn cái nào?**

> CloudTrail Lake tốt hơn khi cần setup nhanh, muốn query ngay không cần cấu hình Glue, cần long-term retention (7 năm) và có multi-account/organization scope. Athena + S3 tốt hơn khi cần tối ưu chi phí (S3 rẻ hơn Lake), muốn tích hợp với data lake hiện có, hoặc cần custom transformations với Glue ETL. Nhiều tổ chức dùng cả hai: Lake cho query ad-hoc realtime, Athena cho batch analytics và reporting.

**Q: Làm sao phân biệt giữa API call từ legitimate automation và attacker?**

> Xem `userAgent` field — automation hợp lệ thường có user agent nhất quán (Terraform, boto3, aws-cli). Attacker thường dùng user agent generic hoặc thay đổi. Xem `sourceIPAddress` — internal automation dùng IP nội bộ hoặc VPC endpoints (không có IP). Xem `userIdentity.sessionContext` — service accounts có session name cố định, attacker thường dùng session name random.

---

**Tiếp Theo:** [3-aws-config-rules.md](3-aws-config-rules.md) — AWS Config Rules và auto-remediation

---

**Cập Nhật Lần Cuối:** 2026-05-16
