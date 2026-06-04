# 5 — Audit & Compliance — Kiểm Toán và Tuân Thủ

> Audit (Kiểm Toán) là khả năng trả lời câu hỏi: "Ai đã làm gì với database của chúng ta, khi nào, và từ đâu?" Compliance (Tuân Thủ) là đảm bảo hệ thống đáp ứng các tiêu chuẩn như PCI-DSS (Payment Card Industry Data Security Standard — Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán), HIPAA (Health Insurance Portability and Accountability Act — Luật Trách Nhiệm Giải Trình và Di Động Bảo Hiểm Y Tế), và GDPR (General Data Protection Regulation — Quy Định Bảo Vệ Dữ Liệu Chung).

---

## 📚 Mục Lục

1. [Tổng Quan Audit Database](#1-tổng-quan-audit-database)
2. [CloudTrail — Kiểm Toán AWS API](#2-cloudtrail--kiểm-toán-aws-api)
3. [Database Activity Streams — Luồng Hoạt Động Database](#3-database-activity-streams--luồng-hoạt-động-database)
4. [RDS Native Audit Logging](#4-rds-native-audit-logging)
5. [CloudWatch Logs & Alerting](#5-cloudwatch-logs--alerting)
6. [Compliance Frameworks](#6-compliance-frameworks)
7. [AWS Config & Security Hub](#7-aws-config--security-hub)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tổng Quan Audit Database

### Hai Tầng Audit Cần Thiết

```
TẦNG 1: AWS Control Plane Audit (Kiểm Toán Tầng Điều Khiển)
─────────────────────────────────────────────────────────────
Trả lời: Ai đã làm gì với AWS RESOURCE?

Ví dụ câu hỏi:
• Ai đã tạo/xóa RDS instance?
• Ai đã sửa Security Group của database?
• Ai đã download snapshot?
• Ai đã thay đổi KMS key policy?
• Ai đã tạo/xóa IAM user/role?

→ Công cụ: AWS CloudTrail

TẦNG 2: Database Data Plane Audit (Kiểm Toán Tầng Dữ Liệu)
──────────────────────────────────────────────────────────────
Trả lời: Ai đã truy cập/thay đổi DATA gì trong database?

Ví dụ câu hỏi:
• User nào đã chạy SELECT * FROM customers?
• Ai đã UPDATE bảng payments vào lúc 3 giờ sáng?
• Có SELECT nào lấy hơn 10,000 rows không?
• Ai đã DELETE records trong bảng orders?
• Có login failed liên tục từ IP nào không?

→ Công cụ: Database Activity Streams, RDS native audit logs
```

### Tại Sao Audit Quan Trọng

```
Business reasons:
• Phát hiện insider threats (nguy cơ từ nội bộ)
• Điều tra incidents (sự cố)
• Compliance bắt buộc (PCI-DSS, HIPAA yêu cầu audit log)
• Forensics (điều tra pháp lý) khi có breach

Technical reasons:
• Detect anomalies (bất thường) — query bất thường có thể là data theft
• Performance: audit logs giúp tìm slow queries
• Debugging: reproduce lại vấn đề từ audit trail
```

---

## 2. CloudTrail — Kiểm Toán AWS API

### CloudTrail Là Gì?

**CloudTrail** ghi lại mọi API call được thực hiện trong AWS account của bạn — từ Console, CLI, SDK, hoặc từ AWS service khác.

```
Mỗi CloudTrail event ghi lại:
├── WHO:   IAM identity (user/role/account) thực hiện
├── WHAT:  API action (CreateDBInstance, DeleteDBSnapshot, ...)
├── WHEN:  Timestamp (ISO 8601, UTC)
├── WHERE: Source IP address, region
├── HOW:   Request parameters
└── RESULT: Response (success/error)
```

### CloudTrail Event Ví Dụ

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROA123456789:john.doe@company.com",
    "arn": "arn:aws:sts::123456789012:assumed-role/DBA-Role/john.doe@company.com",
    "accountId": "123456789012"
  },
  "eventTime": "2026-05-15T03:22:15Z",
  "eventSource": "rds.amazonaws.com",
  "eventName": "DeleteDBSnapshot",    // Xóa snapshot — cần audit!
  "sourceIPAddress": "203.0.113.100", // IP ngoài VPC — suspicious!
  "requestParameters": {
    "dBSnapshotIdentifier": "prod-db-backup-2026-05-01"
  },
  "responseElements": {
    "dBSnapshot": {
      "status": "deleted"
    }
  }
}
```

### Setup CloudTrail Cho Database Security

```bash
# Tạo Trail ghi lại mọi management events
aws cloudtrail create-trail \
  --name "prod-database-audit-trail" \
  --s3-bucket-name "my-cloudtrail-logs-bucket" \
  --include-global-service-events \  # Capture IAM, STS events
  --is-multi-region-trail \          # Bắt buộc — cover tất cả regions
  --enable-log-file-validation \     # Detect tampering (can thiệp)
  --kms-key-id arn:aws:kms:...:key/mrk-audit  # Encrypt logs

# Enable trail
aws cloudtrail start-logging --name "prod-database-audit-trail"

# Bật Data Events cho DynamoDB (theo dõi data-level operations)
aws cloudtrail put-event-selectors \
  --trail-name "prod-database-audit-trail" \
  --event-selectors '[
    {
      "ReadWriteType": "All",
      "IncludeManagementEvents": true,
      "DataResources": [
        {
          "Type": "AWS::DynamoDB::Table",
          "Values": ["arn:aws:dynamodb:ap-southeast-1:123456789012:table/orders"]
        }
      ]
    }
  ]'
```

### CloudTrail S3 Bucket Security

```bash
# S3 Bucket Policy cho CloudTrail logs — chỉ CloudTrail được ghi
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-bucket"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-bucket/AWSLogs/*",
      "Condition": {
        "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
      }
    },
    {
      "Sid": "DenyDeleteCloudTrailLogs",
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["s3:DeleteObject", "s3:DeleteObjectVersion"],
      "Resource": "arn:aws:s3:::my-cloudtrail-logs-bucket/*"
      // Không ai được xóa audit logs — kể cả root
    }
  ]
}
```

### Query CloudTrail Bằng Athena

```sql
-- Tạo bảng Athena từ CloudTrail logs trên S3
CREATE EXTERNAL TABLE cloudtrail_logs (
    eventVersion STRING,
    userIdentity STRUCT<
        type:STRING,
        principalId:STRING,
        arn:STRING,
        accountId:STRING,
        userName:STRING
    >,
    eventTime STRING,
    eventSource STRING,
    eventName STRING,
    sourceIPAddress STRING,
    requestParameters STRING,
    responseElements STRING
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-logs-bucket/AWSLogs/123456789012/CloudTrail/';

-- Query: Ai đã xóa snapshots trong 30 ngày qua?
SELECT
    userIdentity.arn AS who,
    eventTime AS when,
    sourceIPAddress AS from_ip,
    requestParameters AS what
FROM cloudtrail_logs
WHERE eventName = 'DeleteDBSnapshot'
  AND eventTime >= date_add('day', -30, current_date)
ORDER BY eventTime DESC;

-- Query: Phát hiện bất thường — nhiều lần describe từ IP lạ
SELECT sourceIPAddress, COUNT(*) AS request_count
FROM cloudtrail_logs
WHERE eventSource = 'rds.amazonaws.com'
  AND eventName LIKE 'Describe%'
  AND eventTime >= date_add('hour', -1, current_timestamp)
GROUP BY sourceIPAddress
HAVING COUNT(*) > 100
ORDER BY request_count DESC;
```

---

## 3. Database Activity Streams — Luồng Hoạt Động Database

### Database Activity Streams Là Gì?

**Database Activity Streams** (Luồng Hoạt Động Database) capture mọi activity ở cấp độ database engine, bao gồm từng câu SQL, authenticated user, và results. Dữ liệu được stream qua Amazon Kinesis Data Streams (Luồng Dữ Liệu Kinesis) và có thể không bị database admin can thiệp.

```
Architecture của Database Activity Streams:

RDS/Aurora Instance
    │
    ├── Database Engine captures: SQL text, user, host, result
    │
    ├── Encrypt với KMS key (database admin KHÔNG thể decrypt — tamper-proof)
    │
    ▼
Kinesis Data Stream
    │
    ├──► CloudWatch Logs (lưu trữ, query)
    ├──► S3 (long-term storage cho compliance)
    ├──► SIEM tools (Splunk, IBM QRadar, Datadog)
    └──► Custom Lambda (real-time alerting)
```

### Hỗ Trợ Database Activity Streams

| Database | Hỗ Trợ | Mode |
|----------|--------|------|
| Aurora PostgreSQL | ✅ | Synchronous & Asynchronous |
| Aurora MySQL | ✅ | Asynchronous only |
| RDS for Oracle | ✅ | Synchronous & Asynchronous |
| RDS for PostgreSQL | ✅ | Synchronous & Asynchronous |
| RDS for SQL Server | ✅ | Asynchronous only |
| RDS for MySQL | ❌ | Không hỗ trợ (dùng native audit log) |

### Enable Database Activity Streams

```bash
# Enable cho Aurora cluster
aws rds start-activity-stream \
  --resource-arn arn:aws:rds:ap-southeast-1:123456789012:cluster:prod-aurora \
  --mode async \          # async: ít performance impact hơn sync
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789012:key/mrk-audit \
  --apply-immediately

# Response trả về:
# {
#   "KinesisStreamName": "aws-rds-das-cluster-ABCDEF",
#   "Status": "starting"
# }
```

### Synchronous vs Asynchronous Mode

```
Synchronous Mode (Chế Độ Đồng Bộ):
  ├── Query PHẢI được audit stream accept trước khi execute
  ├── Đảm bảo không missing events
  ├── Performance impact: thêm latency cho mỗi query (~1-5ms)
  └── Dùng cho: high-compliance workloads (PCI-DSS, HIPAA tier-1)

Asynchronous Mode (Chế Độ Bất Đồng Bộ):
  ├── Query execute trước, audit sau
  ├── Có thể miss events nếu instance crash
  ├── Performance impact: minimal
  └── Dùng cho: hầu hết production workloads cần balance

→ PCI-DSS tier-1 (cardholder data) yêu cầu Synchronous
→ Hầu hết use cases khác: Asynchronous là đủ
```

### Đọc và Phân Tích Activity Stream Events

```python
import boto3
import base64
import json
import zlib
import aws_encryption_sdk

kinesis_client = boto3.client('kinesis', region_name='ap-southeast-1')
kms_client = boto3.client('kms')

def process_activity_stream(stream_name: str):
    """Đọc và decrypt activity stream events"""
    shard_iterator = get_shard_iterator(stream_name)

    while True:
        response = kinesis_client.get_records(
            ShardIterator=shard_iterator,
            Limit=100
        )

        for record in response['Records']:
            # Decode base64
            payload = base64.b64decode(record['Data'])

            # Parse outer wrapper
            event = json.loads(payload)

            # Decrypt database activity record
            if 'databaseActivityEvents' in event:
                encrypted = base64.b64decode(event['databaseActivityEvents'])
                key_encrypted = base64.b64decode(event['key'])

                # Decrypt data encryption key
                dek = kms_client.decrypt(CiphertextBlob=key_encrypted)['Plaintext']

                # Decrypt activity data
                decrypted = decrypt_with_dek(encrypted, dek)
                activities = json.loads(zlib.decompress(decrypted))

                for activity in activities.get('databaseActivityEventList', []):
                    analyze_activity(activity)

        shard_iterator = response['NextShardIterator']


def analyze_activity(activity: dict):
    """Phân tích từng database activity event"""
    event_type = activity.get('type')       # 'record', 'heartbeat', 'startup'
    command = activity.get('command')       # SQL command type: SELECT, INSERT, UPDATE...
    object_name = activity.get('objectName') # Table name
    db_user = activity.get('dbUserName')   # Database username
    client_ip = activity.get('clientApplication')  # Client IP
    sql_text = activity.get('commandText')  # Actual SQL query
    rows = activity.get('rowCount')         # Số rows affected

    # Rule 1: Alert khi SELECT nhiều rows bất thường (potential data exfiltration)
    if command == 'SELECT' and rows and int(rows) > 10000:
        send_alert(f"Large data export: {db_user} selected {rows} rows from {object_name}")

    # Rule 2: Alert khi admin access ngoài giờ làm việc
    import datetime
    hour = datetime.datetime.utcnow().hour
    if db_user in ['admin', 'root', 'dba'] and (hour < 1 or hour > 11):  # Outside UTC 1-11 (UTC+7 8am-6pm)
        send_alert(f"Admin access outside business hours: {db_user} from {client_ip}")

    # Rule 3: Alert khi có DROP/TRUNCATE
    if command in ['DROP', 'TRUNCATE', 'DELETE']:
        send_alert(f"Destructive operation: {db_user} executed {command} on {object_name}")
```

---

## 4. RDS Native Audit Logging

### MySQL General Query Log & Audit Log

```bash
# Enable MySQL general query log (không recommend cho production — quá verbose)
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-mysql-params \
  --parameters "ParameterName=general_log,ParameterValue=1,ApplyMethod=immediate"

# MySQL Audit Log (dùng MariaDB Audit Plugin cho MySQL trên RDS)
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-mysql-params \
  --parameters \
    "ParameterName=server_audit_logging,ParameterValue=ON,ApplyMethod=immediate" \
    "ParameterName=server_audit_events,ParameterValue=CONNECT,QUERY_DDL,QUERY_DML_NO_SELECT,ApplyMethod=immediate" \
    # CONNECT: log kết nối/ngắt kết nối
    # QUERY_DDL: log CREATE/DROP/ALTER
    # QUERY_DML_NO_SELECT: log INSERT/UPDATE/DELETE (bỏ SELECT để giảm noise)
    "ParameterName=server_audit_excl_users,ParameterValue=rdsadmin,ApplyMethod=immediate"
    # Loại trừ user nội bộ của AWS
```

### PostgreSQL pgAudit Extension

```bash
# Enable pgAudit cho PostgreSQL
aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-pg-params \
  --parameters \
    "ParameterName=shared_preload_libraries,ParameterValue=pgaudit,ApplyMethod=pending-reboot" \
    "ParameterName=pgaudit.log,ParameterValue=ddl\,write,ApplyMethod=immediate"
    # ddl: audit CREATE/DROP/ALTER
    # write: audit INSERT/UPDATE/DELETE/TRUNCATE
    # read: audit SELECT (nhiều, cẩn thận)
    # role: audit GRANT/REVOKE
```

### Export Logs To CloudWatch

```bash
# Enable CloudWatch Logs export cho RDS MySQL
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql \
  --cloudwatch-logs-export-configuration '{
    "EnableLogTypes": ["audit", "error", "general", "slowquery"]
  }'

# Logs sẽ xuất hiện tại:
# /aws/rds/instance/prod-mysql/audit
# /aws/rds/instance/prod-mysql/error
# /aws/rds/instance/prod-mysql/slowquery
```

---

## 5. CloudWatch Logs & Alerting

### Log Groups Cho Database Monitoring

```
/aws/rds/instance/{db-name}/audit      → Audit events
/aws/rds/instance/{db-name}/error      → Database errors
/aws/rds/instance/{db-name}/slowquery  → Slow queries
/aws/rds/cluster/{cluster-name}/audit  → Aurora cluster audit
/aws/kms/                              → KMS API usage
CloudTrail/                            → AWS API audit
```

### CloudWatch Metric Filters — Bộ Lọc Chỉ Số

```bash
# Tạo metric filter phát hiện failed logins
aws logs put-metric-filter \
  --log-group-name "/aws/rds/instance/prod-mysql/error" \
  --filter-name "FailedLoginAttempts" \
  --filter-pattern "[timestamp, requestId, level=\"Access denied*\"]" \
  --metric-transformations \
    metricName=FailedLoginAttempts,metricNamespace=DatabaseSecurity,metricValue=1

# Tạo alarm khi nhiều failed logins (brute force attack)
aws cloudwatch put-metric-alarm \
  --alarm-name "DatabaseBruteForce" \
  --alarm-description "Multiple failed database login attempts" \
  --metric-name FailedLoginAttempts \
  --namespace DatabaseSecurity \
  --statistic Sum \
  --period 300 \          # 5 phút
  --evaluation-periods 1 \
  --threshold 10 \        # Hơn 10 lần thất bại trong 5 phút
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:ap-southeast-1:123456789012:security-alerts \
  --treat-missing-data notBreaching
```

### Security-Focused CloudWatch Alarms

```bash
# Alarm: RDS instance bị delete
aws cloudwatch put-metric-alarm \
  --alarm-name "RDSInstanceDeleted" \
  --metric-name "CallCount" \
  --namespace "CloudTrailMetrics" \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:...:security-critical \
  --dimensions Name=EventName,Value=DeleteDBInstance

# Alarm: KMS key disabled (có thể làm database không decrypt được)
aws cloudwatch put-metric-alarm \
  --alarm-name "KMSKeyDisabled" \
  --metric-name "CallCount" \
  --namespace "CloudTrailMetrics" \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:...:security-critical \
  --dimensions Name=EventName,Value=DisableKey

# Alarm: Security Group rule thay đổi
aws cloudwatch put-metric-alarm \
  --alarm-name "SecurityGroupChanged" \
  --metric-name "CallCount" \
  --namespace "CloudTrailMetrics" \
  --statistic Sum --period 300 --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:...:security-alerts \
  --dimensions Name=EventName,Value=AuthorizeSecurityGroupIngress
```

---

## 6. Compliance Frameworks

### PCI-DSS — Tiêu Chuẩn Bảo Mật Thẻ Thanh Toán

**PCI-DSS** (Payment Card Industry Data Security Standard) áp dụng cho mọi tổ chức xử lý, lưu trữ, hoặc truyền cardholder data (dữ liệu chủ thẻ).

```
PCI-DSS Requirements liên quan đến Database:

Requirement 3 — Protect Stored Cardholder Data:
✅ Mã hóa PAN (Primary Account Number — Số Tài Khoản Chính) khi lưu trữ
✅ Chỉ lưu minimum data cần thiết
✅ Mask PAN khi hiển thị (chỉ show 4 số cuối: XXXX-XXXX-XXXX-1234)
→ Implementation: KMS CMK, column-level encryption

Requirement 7 — Restrict Access by Business Need:
✅ Least privilege cho database accounts
✅ Document tất cả accounts và permissions
→ Implementation: IAM policies, database grants, regular reviews

Requirement 8 — Identify & Authenticate Access:
✅ Unique ID cho mỗi user truy cập database
✅ Strong authentication (no shared accounts)
✅ Rotate passwords ít nhất 90 ngày
→ Implementation: IAM Auth hoặc Secrets Manager rotation ≤90 days

Requirement 10 — Log & Monitor All Access:
✅ Audit log cho mọi access vào cardholder data environment
✅ Log retention ít nhất 12 tháng (3 tháng online, 9 tháng offline)
✅ Alert khi detect suspected compromises
→ Implementation: Database Activity Streams (Synchronous mode), CloudWatch Logs
```

### HIPAA — Bảo Vệ Dữ Liệu Y Tế

**HIPAA** áp dụng cho PHI (Protected Health Information — Thông Tin Y Tế Được Bảo Vệ): thông tin có thể identify một cá nhân và liên quan đến sức khỏe.

```
HIPAA Technical Safeguards (Biện Pháp Bảo Vệ Kỹ Thuật):

Access Control:
✅ Unique user identification
✅ Emergency access procedure (quy trình truy cập khẩn cấp)
✅ Automatic logoff (phiên tự động hết hạn)
✅ Encryption/decryption mechanisms
→ Implementation: IAM + Database Users + KMS

Audit Controls:
✅ Mechanism to record and examine activity in systems
    containing or using ePHI (electronic PHI — PHI điện tử)
→ Implementation: Database Activity Streams, CloudTrail

Transmission Security:
✅ Protect ePHI during transmission
✅ Encryption when appropriate (considered required)
→ Implementation: TLS 1.2+, enforce SSL

AWS HIPAA Eligible Services:
• RDS, Aurora, DynamoDB, ElastiCache đều có trong danh sách
• Cần ký BAA (Business Associate Agreement — Thỏa Thuận Đối Tác Kinh Doanh) với AWS
```

### GDPR — Bảo Vệ Dữ Liệu Cá Nhân EU

**GDPR** áp dụng cho dữ liệu của EU residents, bất kể hệ thống đặt ở đâu.

```
GDPR Articles liên quan đến Database:

Article 5 — Data minimization (Tối Giản Hóa Dữ Liệu):
✅ Chỉ collect và lưu data cần thiết
✅ Không lưu lâu hơn cần thiết (retention policy)
→ Implementation: Database TTL policies, automated deletion jobs

Article 17 — Right to Erasure (Quyền Xóa Dữ Liệu — "Right to be Forgotten"):
✅ Phải có khả năng xóa tất cả data của một user khi được yêu cầu
→ Implementation: Soft delete + scheduled hard delete,
                  document deletion process, confirm với audit log

Article 25 — Privacy by Design (Bảo Mật Theo Thiết Kế):
✅ Pseudonymization và encryption là default
→ Implementation: Encrypt PII fields, pseudonymize data trong analytics

Article 32 — Security of Processing (Bảo Mật Xử Lý):
✅ Encryption at rest và in transit
✅ Confidentiality, integrity, availability
✅ Regular security testing
→ Implementation: KMS + TLS + CloudTrail + penetration testing

Article 33 — Breach Notification (Thông Báo Vi Phạm):
✅ Phải thông báo DPA (Data Protection Authority) trong 72 giờ
✅ Phải thông báo individuals nếu high risk
→ Implementation: Incident response plan với 72h deadline,
                  GuardDuty + Security Hub alerts
```

### Compliance Matrix — Ma Trận Tuân Thủ

| Control | PCI-DSS | HIPAA | GDPR | AWS Service |
|---------|---------|-------|------|-------------|
| Encryption at rest | Req 3 | Technical Safeguard | Art 32 | KMS + RDS/DynamoDB encryption |
| Encryption in transit | Req 4 | Transmission Security | Art 32 | TLS/SSL |
| Access control | Req 7, 8 | Access Control | Art 25, 32 | IAM + DB Users |
| Audit logging | Req 10 | Audit Controls | Art 5, 30 | CloudTrail + Activity Streams |
| Password rotation | Req 8 | Access Control | Art 32 | Secrets Manager rotation |
| Vulnerability management | Req 6 | Technical Safeguard | Art 32 | Inspector + RDS patching |
| Data retention/deletion | Req 3 | Minimum Necessary | Art 5, 17 | S3 lifecycle + automated deletion |

---

## 7. AWS Config & Security Hub

### AWS Config — Theo Dõi Compliance Liên Tục

**AWS Config** liên tục kiểm tra resource configurations và đánh giá theo compliance rules:

```bash
# Bật AWS Config recording
aws configservice start-configuration-recorder \
  --configuration-recorder-name default

# Managed rules liên quan đến database security:

# Kiểm tra RDS không có public accessibility
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "rds-instance-public-access-check",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "RDS_INSTANCE_PUBLIC_ACCESS_CHECK"
    }
  }'

# Kiểm tra RDS đã bật encryption
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "rds-storage-encrypted",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "RDS_STORAGE_ENCRYPTED"
    }
  }'

# Kiểm tra RDS đã bật Multi-AZ
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "rds-multi-az-support",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "RDS_MULTI_AZ_SUPPORT"
    }
  }'

# Kiểm tra RDS snapshot không public
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "rds-snapshot-encrypted",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "RDS_SNAPSHOT_ENCRYPTED"
    }
  }'

# Khi rule bị vi phạm → CloudWatch alarm → SNS notification → on-call engineer
```

### AWS Security Hub — Trung Tâm Bảo Mật

**Security Hub** tổng hợp findings từ nhiều AWS security services vào một dashboard:

```
Security Hub aggregates findings từ:
├── Amazon GuardDuty (threat detection — phát hiện mối đe dọa)
├── Amazon Inspector (vulnerability scanning — quét lỗ hổng)
├── AWS Config (compliance violations — vi phạm tuân thủ)
├── AWS IAM Access Analyzer
├── Amazon Macie (data classification — phân loại dữ liệu)
└── Third-party tools (Splunk, CrowdStrike, ...)

Security Standards supported:
├── AWS Foundational Security Best Practices
├── CIS AWS Foundations Benchmark
├── PCI-DSS
└── NIST SP 800-53

Ví dụ findings liên quan đến database:
• [HIGH] RDS instance has publicly accessible endpoint
• [CRITICAL] RDS encryption is not enabled
• [MEDIUM] RDS instance has no deletion protection enabled
• [HIGH] KMS key rotation is not enabled
• [MEDIUM] Secrets Manager automatic rotation is not configured
```

### Amazon GuardDuty — Phát Hiện Mối Đe Dọa

```
GuardDuty threat types liên quan đến database:

RDS Protection (tính năng mới, cần enable riêng):
• RDS:IAMAnomalousBehavior — login bất thường vào RDS qua IAM
• RDS:LoginAttemptFromKnownMaliciousIP — login từ IP đã biết là độc hại
• RDS:TorIPCaller — login từ Tor network (thường là attacker ẩn danh)

General findings ảnh hưởng đến database security:
• UnauthorizedAccess:IAMUser/ConsoleLoginSuccess — login console bất thường
• CredentialAccess:IAMUser/AnomalousBehavior — IAM key dùng bất thường
• Exfiltration:S3/AnomalousBehavior — có thể backup data bị exfiltrate

→ Tích hợp GuardDuty findings vào Security Hub
→ Set SNS notification cho HIGH/CRITICAL severity findings
→ Lambda function auto-remediation cho một số findings
   (ví dụ: tự động revoke credentials khi detect compromised key)
```

---

## 8. Câu Hỏi Phỏng Vấn

**Q: Phân biệt CloudTrail và Database Activity Streams. Khi nào cần cả hai?**

> CloudTrail: ghi lại AWS API calls (control plane) — ai tạo/xóa RDS instance, thay đổi Security Group, lấy secret. Database Activity Streams: ghi lại database-level activity (data plane) — ai chạy câu SQL nào, trên table nào. Cần cả hai khi có compliance requirements như PCI-DSS hoặc HIPAA, vì audit trail phải cover cả "thay đổi cấu hình AWS" và "truy cập data thực tế trong database."

**Q: PCI-DSS yêu cầu gì về database audit logging?**

> PCI-DSS Requirement 10: (1) Log tất cả access vào cardholder data environment; (2) Log failed login attempts; (3) Alert về suspicious activity; (4) Sync timestamps với NTP (tất cả systems phải cùng time zone); (5) Bảo vệ logs khỏi bị modify (immutable storage — S3 Object Lock); (6) Retention: 12 tháng, với 3 tháng sẵn sàng để query ngay. Implementation trên AWS: Database Activity Streams (Synchronous mode) + CloudTrail + S3 với Object Lock Compliance mode.

**Q: Giải thích Database Activity Streams — tại sao "tamper-proof" (không thể giả mạo)?**

> Activity data được encrypt bằng KMS key ngay tại database engine, trước khi gửi ra Kinesis. Key policy của KMS key này chỉ cho phép database engine encrypt — database admin KHÔNG có quyền decrypt trực tiếp. Admin cần qua separate process với separate IAM permission và audit trail riêng để decrypt logs. Kết hợp với S3 Object Lock (immutable) cho logs đã archive → không ai (kể cả admin) có thể modify hay xóa audit history.

**Q: Làm thế nào implement "Right to be Forgotten" của GDPR cho RDS database?**

> Multi-step approach: (1) Design data model với user_id làm identifier; (2) Khi nhận deletion request: soft delete (set deleted_at, anonymize PII fields); (3) Scheduled job hard delete sau grace period (ví dụ 30 ngày); (4) Handle cascade: delete/anonymize data của user từ tất cả tables; (5) Handle backups: maintain deletion log để apply lại khi restore từ backup; (6) Confirm bằng audit log: CloudTrail record API call + Database Activity Streams record DELETE operations; (7) Trả lời requester với confirmation và timestamp. Phức tạp nhất là database backups — không thể "delete" từ snapshot, phải apply deletion khi restore.

---

## 📂 Điều Hướng

| ← Trước | Chủ Đề Hiện Tại | Tổng Quan |
|---------|-----------------|-----------|
| [4-secrets-management.md](./4-secrets-management.md) | **5-audit-compliance.md** | [README.md](./README.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15 | **Trạng Thái:** ✅ Hoàn Thành
