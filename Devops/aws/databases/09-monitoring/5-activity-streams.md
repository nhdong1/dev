# Database Activity Streams — Luồng Hoạt Động Cơ Sở Dữ Liệu

> Hướng dẫn toàn diện về Database Activity Streams (Luồng Hoạt Động Cơ Sở Dữ Liệu) — tính năng audit real-time (kiểm toán thời gian thực) cho Aurora và RDS, tích hợp với Kinesis Data Streams (Luồng Dữ Liệu Kinesis), SIEM (Security Information and Event Management — Quản Lý Thông Tin & Sự Kiện Bảo Mật), và đáp ứng yêu cầu compliance (tuân thủ) PCI-DSS, HIPAA, GDPR.

---

## 📚 Mục Lục

1. [Database Activity Streams là gì](#database-activity-streams-là-gì)
2. [Kiến Trúc Hoạt Động](#kiến-trúc-hoạt-động)
3. [Bật Activity Streams](#bật-activity-streams)
4. [Cấu Trúc Activity Record](#cấu-trúc-activity-record)
5. [Tích Hợp Kinesis & Downstream](#tích-hợp-kinesis--downstream)
6. [SIEM Integration](#siem-integration)
7. [Compliance Use Cases](#compliance-use-cases)
8. [Security Monitoring](#security-monitoring)
9. [Best Practices & Chi Phí](#best-practices--chi-phí)

---

## Database Activity Streams là gì

Database Activity Streams (DAS) là tính năng RDS/Aurora cung cấp **near real-time stream** (luồng gần thời gian thực) của **mọi hoạt động** xảy ra trong database — bao gồm DML (Data Manipulation Language — Ngôn Ngữ Thao Tác Dữ Liệu), DDL (Data Definition Language — Ngôn Ngữ Định Nghĩa Dữ Liệu), và các hoạt động quản trị.

### So Sánh Với CloudTrail

```
CloudTrail:
  → Log API calls đến AWS service (control plane)
  → "Ai đó gọi ModifyDBInstance lúc 3:00 PM"
  → Không thấy SQL queries trong database

Database Activity Streams:
  → Log mọi hoạt động trong database (data plane)
  → "User 'app_user' từ IP 10.0.1.5 chạy SELECT * FROM users WHERE id=123"
  → Thấy từng câu SQL, từng transaction

Dùng cả hai để có visibility toàn diện:
  CloudTrail = "Ai thay đổi cấu hình database?"
  Activity Streams = "Ai đọc/ghi dữ liệu gì trong database?"
```

### Tại Sao Cần Activity Streams

```
Use Cases chính:

1. Compliance & Audit (Tuân Thủ & Kiểm Toán):
   PCI-DSS: Phải track mọi access đến cardholder data
   HIPAA:   Phải audit mọi access đến protected health information
   GDPR:    Phải chứng minh ai có access đến personal data

2. Security Threat Detection (Phát Hiện Mối Đe Dọa Bảo Mật):
   SQL injection attempts (thử tấn công SQL injection)
   Unauthorized data access (truy cập dữ liệu trái phép)
   Privilege escalation (leo thang đặc quyền)
   Data exfiltration (lọc dữ liệu)

3. Operational Troubleshooting (Khắc Phục Sự Cố Vận Hành):
   Tìm câu SQL nào gây ra incident
   Trace user actions (theo dõi hành động người dùng) trong incident investigation
   Forensics (điều tra pháp lý) sau data breach
```

---

## Kiến Trúc Hoạt Động

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Database Activity Streams Flow                       │
│                   (Luồng Hoạt Động Cơ Sở Dữ Liệu)                      │
│                                                                          │
│  ┌──────────────────┐                                                    │
│  │   Application    │                                                    │
│  │   (Ứng Dụng)     │                                                    │
│  └────────┬─────────┘                                                    │
│           │ SQL queries                                                  │
│  ┌────────▼─────────────────────────────────────────────────────────┐   │
│  │           Aurora / RDS Database                                   │   │
│  │                                                                   │   │
│  │  ┌─────────────────────────────────────────────────────────┐     │   │
│  │  │  Activity Stream Agent (built-in)                       │     │   │
│  │  │  Ghi lại mọi SQL statement + metadata                   │     │   │
│  │  └──────────────────────┬──────────────────────────────────┘     │   │
│  └─────────────────────────┼─────────────────────────────────────────┘  │
│                            │ Encrypted JSON records                      │
│                            ▼                                             │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │        Amazon Kinesis Data Streams                               │    │
│  │        (Luồng Dữ Liệu Kinesis)                                  │    │
│  │        Encrypt với KMS CMK                                       │    │
│  └─────────┬──────────────┬──────────────┬──────────────────────────┘   │
│            │              │              │                               │
│            ▼              ▼              ▼                               │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐                │
│  │  AWS Lambda  │ │  Amazon S3   │ │  SIEM (Splunk /  │                │
│  │  (xử lý &   │ │  (lưu trữ   │ │  Datadog / etc.) │                │
│  │  alert)     │ │  dài hạn)   │ │                  │                │
│  └──────────────┘ └──────────────┘ └──────────────────┘                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Encryption Architecture (Kiến Trúc Mã Hóa)

```
Activity Streams mã hóa 2 lớp:

Lớp 1: Kinesis Data Streams mã hóa với AWS KMS
  → Data được mã hóa khi ghi vào Kinesis
  → DBA admin KHÔNG thể đọc stream trực tiếp

Lớp 2: Record payload mã hóa với Database CMK (Customer Managed Key)
  → Mỗi record JSON được mã hóa thêm
  → Chỉ consumer có access đến KMS key mới đọc được

Tại sao 2 lớp?
  → Ngăn chặn DBA hoặc AWS admin đọc data nhạy cảm
  → Đảm bảo separation of duties (phân tách nhiệm vụ)
  → Audit trail cho việc ai decrypt activity records
```

---

## Bật Activity Streams

### Hỗ Trợ Hiện Tại

```
Hỗ Trợ Activity Streams:
  ✅ Aurora MySQL (version 5.7+ và 8.0+)
  ✅ Aurora PostgreSQL (version 10.7+)
  ✅ RDS for Oracle (version 12.2+, 18c, 19c)
  ✅ RDS for PostgreSQL (version 10.7+)
  ✅ RDS for SQL Server (Enterprise/Standard edition)

KHÔNG hỗ trợ:
  ❌ Aurora Serverless v1
  ❌ RDS for MySQL (chỉ Aurora MySQL được hỗ trợ)
  ❌ RDS for MariaDB
  ❌ Aurora Serverless v2 (đang phát triển tính năng)
```

### Bật qua AWS Console

```
1. RDS Console → Databases → chọn Aurora cluster
2. Click "Actions" → "Start activity stream"
3. Cấu hình:
   - KMS key: Chọn CMK (Customer Managed Key — Khóa Quản Lý Bởi Khách Hàng)
   - Kinesis stream: Tự động tạo hoặc dùng existing
   - Mode:
     * Synchronous (Đồng Bộ): Đảm bảo ghi audit trước khi commit
       → Hiệu năng bị ảnh hưởng nhẹ
       → Dùng khi audit trail bắt buộc (PCI-DSS, HIPAA)
     * Asynchronous (Bất Đồng Bộ): Ghi audit sau commit
       → Ít ảnh hưởng hiệu năng hơn
       → Một số records có thể mất khi instance crash
4. Confirm bật (không cần restart database)
```

### Bật qua AWS CLI

```bash
# Lấy cluster ARN
aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-cluster \
  --query 'DBClusters[0].DBClusterArn'

# Bật Activity Streams
aws rds start-activity-stream \
  --resource-arn arn:aws:rds:ap-southeast-1:123456789:cluster:prod-aurora-cluster \
  --mode async \
  --kms-key-id arn:aws:kms:ap-southeast-1:123456789:key/abcd-1234 \
  --apply-immediately

# Kiểm tra trạng thái
aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-cluster \
  --query 'DBClusters[0].ActivityStreamStatus'
# Output: "started" | "starting" | "stopped" | "stopping"

# Xem Kinesis stream name
aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-cluster \
  --query 'DBClusters[0].ActivityStreamKinesisStreamName'
```

---

## Cấu Trúc Activity Record

### JSON Record Format

```json
{
  "type": "DatabaseActivityMonitoringRecord",
  "clusterId": "cluster-ABCDEFGHIJKLMNOP",
  "instanceId": "db-ABCDEFGHIJKLMNOP",
  "databaseActivityEventList": [
    {
      "class": "CONNECT",
      "type": "CONNECT",
      "clientApplication": "psql",
      "pid": 12345,
      "dbUserName": "app_user",
      "databaseName": "production_db",
      "remoteHost": "10.0.1.45",
      "remotePort": "52143",
      "command": "CONNECT",
      "commandText": null,
      "paramList": null,
      "objectType": "DATABASE",
      "objectName": "production_db",
      "statementId": 1,
      "substatementId": 1,
      "exitCode": "0",
      "sessionId": "3d5e7f11",
      "rowCount": null,
      "logTime": "2026-05-15T09:32:00.123456+00",
      "transactionId": "0",
      "isInternal": false,
      "errorMessage": null,
      "startTime": "2026-05-15T09:32:00.120000+00",
      "endTime": "2026-05-15T09:32:00.123456+00"
    },
    {
      "class": "READ",
      "type": "SELECT",
      "clientApplication": "psql",
      "pid": 12345,
      "dbUserName": "app_user",
      "databaseName": "production_db",
      "remoteHost": "10.0.1.45",
      "remotePort": "52143",
      "command": "SELECT",
      "commandText": "SELECT * FROM users WHERE user_id = $1",
      "paramList": ["12345"],
      "objectType": "TABLE",
      "objectName": "users",
      "statementId": 2,
      "substatementId": 1,
      "exitCode": "0",
      "sessionId": "3d5e7f11",
      "rowCount": 1,
      "logTime": "2026-05-15T09:32:01.234567+00",
      "transactionId": "789012",
      "isInternal": false,
      "errorMessage": null
    }
  ]
}
```

### Giải Thích Các Fields Quan Trọng

```
dbUserName    — Tên user database thực hiện câu lệnh
remoteHost    — IP của client (ứng dụng / DBA)
commandText   — Câu SQL thực tế (quan trọng nhất!)
paramList     — Parameters của prepared statement (tham số của câu lệnh chuẩn bị)
objectName    — Tên table/view được access
rowCount      — Số rows trả về/bị ảnh hưởng
exitCode      — "0" = thành công, khác = lỗi
errorMessage  — Nội dung lỗi nếu có
sessionId     — ID session để track toàn bộ user session
transactionId — ID transaction để group statements liên quan
logTime       — Timestamp chính xác (UTC)
isInternal    — True nếu là internal DB operation (không phải user)
```

### Event Classes (Lớp Sự Kiện)

```
CONNECT:    User kết nối vào database
DISCONNECT: User ngắt kết nối
READ:       SELECT statements
WRITE:      INSERT, UPDATE, DELETE, TRUNCATE
DDL:        CREATE, ALTER, DROP, RENAME
DCL:        GRANT, REVOKE (phân quyền)
```

---

## Tích Hợp Kinesis & Downstream

### Đọc Records Từ Kinesis

```python
# Python: Đọc và decrypt Activity Stream records
import boto3
import base64
import json
import zlib

kinesis_client = boto3.client('kinesis', region_name='ap-southeast-1')
kms_client = boto3.client('kms', region_name='ap-southeast-1')

stream_name = 'aws-rds-das-cluster-XXXXX'

# Lấy shard iterator (con trỏ shard)
shard_iterator = kinesis_client.get_shard_iterator(
    StreamName=stream_name,
    ShardId='shardId-000000000000',
    ShardIteratorType='LATEST'
)['ShardIterator']

# Đọc records
response = kinesis_client.get_records(
    ShardIterator=shard_iterator,
    Limit=100
)

for record in response['Records']:
    # Decode base64
    payload = base64.b64decode(record['Data'])

    # Parse outer structure
    outer = json.loads(payload)

    # Decrypt payload với KMS
    if outer.get('type') == 'DatabaseActivityMonitoringRecord':
        encrypted_payload = base64.b64decode(outer['databaseActivityEvents'])
        key_id = outer['key']

        # Decrypt với KMS
        decrypted = kms_client.decrypt(
            CiphertextBlob=encrypted_payload,
            EncryptionContext={'aws:rds:dbc-id': outer['clusterId']}
        )['Plaintext']

        # Decompress (giải nén) - payload được gzip compressed
        decompressed = zlib.decompress(decrypted, zlib.MAX_WBITS | 16)
        activity_record = json.loads(decompressed)

        for event in activity_record['databaseActivityEventList']:
            print(f"User: {event['dbUserName']}")
            print(f"IP:   {event['remoteHost']}")
            print(f"SQL:  {event['commandText']}")
            print(f"Time: {event['logTime']}")
            print("---")
```

### Lambda Consumer (Hàm Lambda Tiêu Thụ)

```python
# Lambda function triggered by Kinesis
import json
import base64
import zlib
import boto3

def lambda_handler(event, context):
    kms = boto3.client('kms')

    for record in event['Records']:
        # Kinesis record
        payload = base64.b64decode(record['kinesis']['data'])
        outer = json.loads(payload)

        if outer.get('type') != 'DatabaseActivityMonitoringRecord':
            continue

        # Decrypt
        encrypted = base64.b64decode(outer['databaseActivityEvents'])
        decrypted = kms.decrypt(
            CiphertextBlob=encrypted,
            EncryptionContext={'aws:rds:dbc-id': outer['clusterId']}
        )['Plaintext']

        # Decompress
        activity = json.loads(zlib.decompress(decrypted, zlib.MAX_WBITS | 16))

        for evt in activity['databaseActivityEventList']:
            # Detect suspicious activity (phát hiện hoạt động đáng ngờ)
            if is_suspicious(evt):
                send_security_alert(evt)

            # Archive to S3 for compliance (lưu trữ S3 cho compliance)
            archive_to_s3(evt)

def is_suspicious(event):
    # Phát hiện hành vi đáng ngờ
    suspicious_patterns = [
        event.get('commandText', '').upper().find('DROP TABLE') >= 0,
        event.get('commandText', '').upper().find('TRUNCATE') >= 0,
        int(event.get('rowCount') or 0) > 100000,  # Đọc quá nhiều rows
        event.get('exitCode') != '0',               # Lỗi đáng ngờ
    ]
    return any(suspicious_patterns)
```

### Lưu Trữ Long-term trên S3

```
Kinesis Data Firehose (Luồng Dữ Liệu Firehose) → S3:

  Activity Streams → Kinesis Data Streams
                          ↓
               Kinesis Data Firehose
                          ↓
               Amazon S3 (partitioned by date)
               s3://audit-logs/das/year=2026/month=05/day=15/

  S3 Lifecycle Policy (Chính Sách Vòng Đời S3):
    - 0-90 days: S3 Standard (truy cập thường xuyên)
    - 90-365 days: S3 Standard-IA (truy cập ít)
    - 365+ days: S3 Glacier (lưu trữ dài hạn, giảm chi phí 90%)

  Compliance requirement (yêu cầu tuân thủ):
    PCI-DSS: Giữ audit logs 1 năm minimum, 3 tháng online
    HIPAA:   Giữ 6 năm
    GDPR:    Giữ theo mục đích xử lý (thường 3-7 năm)
```

---

## SIEM Integration

### SIEM Supported Integrations (Tích Hợp SIEM Được Hỗ Trợ)

AWS Activity Streams tích hợp native với nhiều SIEM providers:

```
Certified SIEM Integrations:
  ├── IBM QRadar          — Enterprise SIEM
  ├── Splunk              — Most popular, native AWS add-on
  ├── McAfee MVISION Cloud— Cloud security platform
  ├── Micro Focus ArcSight— Enterprise security analytics
  └── Trustwave           — Managed security services

Custom Integration via Kinesis:
  → Bất kỳ system nào có thể consume Kinesis streams
  → Datadog, Elastic, Sumo Logic, etc.
```

### Splunk Integration Example

```
# Splunk AWS Add-on for Activity Streams
1. Cài đặt Splunk Add-on for AWS từ Splunkbase
2. Cấu hình Kinesis Data Stream input:
   Stream: aws-rds-das-cluster-XXXXX
   Region: ap-southeast-1
   KMS Decryption: Enable

3. Tạo Splunk search alert:
   | search sourcetype="aws:kinesis:das"
   | spath output=sql path=databaseActivityEventList{}.commandText
   | spath output=user path=databaseActivityEventList{}.dbUserName
   | search sql="*DROP*" OR sql="*TRUNCATE*" OR sql="*INTO OUTFILE*"
   | alert trigger: realtime
```

### Datadog Integration

```yaml
# datadog-agent/conf.d/kinesis.yaml
init_config:

instances:
  - stream_name: aws-rds-das-cluster-XXXXX
    region: ap-southeast-1
    kms_key_arn: arn:aws:kms:...

# Custom metric từ Activity Streams
# dd_sql_events: COUNT queries by user, table, operation
```

---

## Compliance Use Cases

### PCI-DSS — Payment Card Industry Data Security Standard

PCI-DSS (Tiêu Chuẩn Bảo Mật Dữ Liệu Ngành Thẻ Thanh Toán) yêu cầu:

```
Requirement 10.2 (Yêu Cầu 10.2): Implement audit trails for:
  ✅ All individual user access to cardholder data
     → Activity Streams ghi mọi SELECT trên bảng chứa card data

  ✅ All actions taken by root or administrative privileges
     → Activity Streams ghi mọi action của DBA users

  ✅ Access to all audit trails
     → CloudTrail ghi ai đọc audit records

  ✅ Invalid logical access attempts
     → Activity Streams với exitCode != "0"

  ✅ Use of and changes to identification and authentication mechanisms
     → Activity Streams CONNECT/DCL events

Requirement 10.5 (Yêu Cầu 10.5): Secure audit trails so they cannot be altered:
  ✅ Activity Streams → Kinesis → S3 với Object Lock
  ✅ KMS encryption prevents tampering
  ✅ IAM prevents direct modification of stream
```

### HIPAA — Health Insurance Portability and Accountability Act

HIPAA (Đạo Luật Về Tính Di Động và Trách Nhiệm Bảo Hiểm Y Tế) yêu cầu:

```
Access Controls:
  ✅ Monitor ai access vào bảng chứa PHI (Protected Health Information)
  ✅ Alert khi unusual access patterns
  ✅ Quarterly access review từ audit logs

Audit Controls:
  ✅ Activity Streams cung cấp record của mọi access
  ✅ 6-year retention trên S3 Glacier

Integrity Controls:
  ✅ Detect unauthorized modifications với Activity Streams
  ✅ Alert on DDL changes (DROP TABLE, ALTER TABLE)
```

### Tự Động Hóa Compliance Reports

```python
# Script tạo báo cáo compliance hàng tháng
import boto3
from datetime import datetime, timedelta

def generate_monthly_report(table_name: str, month: str):
    athena = boto3.client('athena')

    query = f"""
    SELECT
        dbUserName,
        remoteHost,
        command,
        COUNT(*) as access_count,
        SUM(CAST(rowCount AS BIGINT)) as total_rows_accessed
    FROM activity_stream_logs
    WHERE
        objectName = '{table_name}'
        AND date_partition = '{month}'
        AND class IN ('READ', 'WRITE')
    GROUP BY dbUserName, remoteHost, command
    ORDER BY total_rows_accessed DESC
    """

    response = athena.start_query_execution(
        QueryString=query,
        ResultConfiguration={'OutputLocation': 's3://compliance-reports/'}
    )

    return response['QueryExecutionId']

# Chạy hàng tháng → gửi email cho auditor
generate_monthly_report('credit_cards', '2026-05')
```

---

## Security Monitoring

### Phát Hiện SQL Injection Attempts

```python
# Patterns phổ biến trong SQL injection attempts
INJECTION_PATTERNS = [
    "'; DROP TABLE",
    "' OR '1'='1",
    "UNION SELECT",
    "xp_cmdshell",
    "INTO OUTFILE",
    "--",
    "/*",
    "0x",  # Hex encoding
]

def detect_sql_injection(command_text: str) -> bool:
    text_upper = (command_text or '').upper()
    return any(pattern.upper() in text_upper for pattern in INJECTION_PATTERNS)
```

### Phát Hiện Data Exfiltration (Lọc Dữ Liệu)

```python
# Ngưỡng phát hiện bất thường
THRESHOLDS = {
    'max_rows_single_query': 10000,   # > 10K rows 1 query là bất thường
    'max_rows_per_session': 100000,   # > 100K rows 1 session
    'off_hours_access': (0, 6),       # Truy cập lúc 12AM-6AM
    'unusual_source_ips': ['0.0.0.0/0']  # External IPs không expect
}

def check_data_exfiltration(event: dict) -> list:
    alerts = []

    row_count = int(event.get('rowCount') or 0)
    if row_count > THRESHOLDS['max_rows_single_query']:
        alerts.append(f"LARGE_READ: {row_count} rows by {event['dbUserName']}")

    hour = datetime.fromisoformat(event['logTime']).hour
    start, end = THRESHOLDS['off_hours_access']
    if start <= hour <= end:
        alerts.append(f"OFF_HOURS_ACCESS: {event['dbUserName']} at {hour}:00")

    return alerts
```

### Phát Hiện Privilege Escalation (Leo Thang Đặc Quyền)

```python
# Theo dõi GRANT statements
SENSITIVE_GRANTS = ['GRANT ALL', 'GRANT SUPER', 'GRANT ROOT']

def detect_privilege_escalation(event: dict) -> bool:
    if event.get('class') == 'DCL':
        text = (event.get('commandText') or '').upper()
        return any(grant in text for grant in SENSITIVE_GRANTS)
    return False
```

---

## Best Practices & Chi Phí

### Chi Phí Activity Streams

```
Activity Streams pricing (giá):
  - Kinesis Data Streams: $0.015/shard-hour + $0.014/1M PUT payload units
  - KMS API calls: $0.03/10,000 requests
  - CloudWatch Logs (nếu archive thêm)
  - S3 storage (long-term archive)

Ước tính chi phí:
  Database nhỏ (100 transactions/s):  ~$50-100/tháng
  Database trung bình (1000 TPS):      ~$200-500/tháng
  Database lớn (10,000 TPS):           ~$1,000-2,000/tháng

→ Chi phí này justified nếu cần compliance (PCI-DSS fine có thể lên $100,000+)
→ Có thể giảm chi phí bằng cách filter events trước khi archive
```

### Synchronous vs Asynchronous Mode

```
Synchronous Mode (Chế Độ Đồng Bộ):
  Hoạt động:
    Transaction → Activity Record ghi vào Kinesis → Commit (xác nhận)
  Ưu điểm:
    → Đảm bảo KHÔNG mất audit record nào
    → Activity record luôn consistent với database state
  Nhược điểm:
    → Latency tăng 1-3ms cho mỗi transaction
    → Nếu Kinesis có vấn đề → transactions bị delay

Asynchronous Mode (Chế Độ Bất Đồng Bộ):
  Hoạt động:
    Transaction → Commit ngay → Activity Record ghi vào Kinesis (sau)
  Ưu điểm:
    → Không ảnh hưởng latency của transactions
  Nhược điểm:
    → Có thể mất một số records khi instance crash
    → Không thể chứng minh 100% completeness cho audit

Chọn khi nào:
  PCI-DSS, HIPAA: Synchronous (đảm bảo completeness)
  General monitoring, troubleshooting: Asynchronous
```

### Checklist Activity Streams

```
□ Xác định databases chứa sensitive data cần Activity Streams
□ Tạo dedicated KMS CMK cho Activity Streams encryption
□ Bật Activity Streams (Async thông thường, Sync nếu compliance yêu cầu)
□ Thiết lập Kinesis → S3 pipeline qua Kinesis Firehose
□ Cấu hình S3 Object Lock (WORM) cho compliance data
□ Thiết lập S3 Lifecycle Policy (Standard → IA → Glacier)
□ Cấu hình Lambda hoặc SIEM consumer để phát hiện anomalies
□ Tạo CloudWatch Alarms cho Kinesis iterator age (đảm bảo consumer không lag)
□ Test: Chạy suspicious SQL và verify nó được capture
□ Document compliance mapping (PCI/HIPAA requirement → Activity Stream field)
□ Thiết lập monthly automated compliance reports
□ IAM policies: Chỉ security team và auditors có access decrypt records
```

---

## Câu Hỏi Phỏng Vấn

### Câu 1: Database Activity Streams khác gì với CloudTrail?

```
CloudTrail:
  → Audit AWS API calls (control plane)
  → "Ai tạo/xóa/modify RDS instance?"
  → Không thấy SQL bên trong database

Database Activity Streams:
  → Audit mọi database activity (data plane)
  → "Ai chạy câu SQL nào, đọc/ghi bảng nào?"
  → Thấy từng statement, user, IP, timestamp

Cần cả hai:
  CloudTrail → Infrastructure changes
  Activity Streams → Data access patterns
```

### Câu 2: Tại sao Activity Streams dùng encryption 2 lớp?

```
Vấn đề: DBA có thể access Kinesis stream trực tiếp
→ Đọc sensitive data (card numbers, SSN...) trong audit log

Giải pháp 2 lớp:
  Lớp 1: Kinesis Server-side Encryption → Bảo vệ data at rest
  Lớp 2: Record-level encryption với CMK → DBA KHÔNG thể decrypt

Chỉ security/compliance team với KMS key access mới đọc được records
→ True separation of duties (phân tách nhiệm vụ thực sự)
```

### Câu 3: Khi nào nên dùng Synchronous mode?

```
Synchronous mode:
  → Khi compliance BUỘC PHẢI có complete audit trail
  → PCI-DSS, HIPAA, SOX environments
  → Khi không thể chứng minh records bị mất là chấp nhận được

Asynchronous mode:
  → General security monitoring
  → Operational troubleshooting
  → Khi latency impact là critical concern
  → Test/staging environments

Rule: Nếu auditor hỏi "có 100% complete?" → cần Synchronous
```

---

**Xem Trước:** [4-dynamodb-monitoring.md](4-dynamodb-monitoring.md) — Giám Sát DynamoDB
**Xem Tiếp:** [../10-advanced/README.md](../10-advanced/README.md) — Chủ Đề Nâng Cao

**Cập Nhật Lần Cuối:** 2026-05-15
