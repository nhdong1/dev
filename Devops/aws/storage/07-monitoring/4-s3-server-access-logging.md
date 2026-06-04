# S3 Server Access Logging — Nhật Ký Truy Cập S3

> S3 Server Access Logging ghi lại chi tiết mọi HTTP request đến bucket — ai đã truy cập, lúc nào, làm gì, từ đâu — thiết yếu cho bảo mật, compliance và debug

---

## 📋 Tổng Quan

S3 Server Access Logging là tính năng **miễn phí** (chỉ trả phí lưu trữ log) ghi lại mọi request đến S3 bucket vào một bucket đích dưới dạng file log text.

| Đặc Điểm | Chi Tiết |
|-----------|---------|
| **Trạng thái mặc định** | Tắt — phải bật thủ công |
| **Chi phí kích hoạt** | Miễn phí |
| **Chi phí lưu trữ log** | Trả phí S3 Standard theo dung lượng log |
| **Độ trễ ghi log** | Vài giờ (best-effort — không đảm bảo real-time) |
| **Format** | Text, mỗi dòng = 1 request |
| **Phân phối log** | Theo batch, không phải từng request |

> **Khác với CloudTrail:** Server Access Logging ghi *data plane* requests (GetObject, PutObject). CloudTrail ghi *management plane* (CreateBucket, PutBucketPolicy). Để đầy đủ cần dùng cả hai.

---

## 🗂️ Kiến Trúc Logging

```
                    ┌──────────────────────┐
                    │   Source Bucket      │
                    │   (bucket gốc)       │
                    │   my-data-bucket     │
                    └──────────┬───────────┘
                               │ client requests
                               ▼
                    ┌──────────────────────┐
                    │   S3 Service         │
                    │   xử lý request      │
                    │   ghi log entry      │
                    └──────────┬───────────┘
                               │ ghi log (vài giờ)
                               ▼
                    ┌──────────────────────┐
                    │   Target Bucket      │
                    │   (bucket log)       │
                    │   my-access-logs     │
                    │   Prefix: s3-logs/   │
                    └──────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Amazon Athena      │
                    │   Query SQL          │
                    └──────────────────────┘
```

**Yêu cầu cấu hình:**
- Target bucket phải **cùng region** với source bucket
- Target bucket nên **khác** với source bucket (tránh vòng lặp logging)
- Cấp quyền cho S3 Log Delivery Group ghi vào target bucket

---

## ⚙️ Cấu Hình S3 Server Access Logging

### Bật qua AWS Console

1. Vào S3 Console → chọn source bucket
2. Tab **Properties** → phần **Server access logging**
3. Chọn **Enable**
4. Chọn Target bucket và Target prefix (ví dụ: `logs/my-data-bucket/`)
5. Save

### Bật qua AWS CLI

```bash
# Bước 1: Tạo target bucket (nếu chưa có)
aws s3api create-bucket \
  --bucket my-access-logs-bucket \
  --region us-east-1

# Bước 2: Cấp quyền cho Log Delivery service
aws s3api put-bucket-acl \
  --bucket my-access-logs-bucket \
  --grant-write URI=http://acs.amazonaws.com/groups/s3/LogDelivery \
  --grant-read-acp URI=http://acs.amazonaws.com/groups/s3/LogDelivery

# Bước 3: Bật logging trên source bucket
aws s3api put-bucket-logging \
  --bucket my-data-bucket \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "my-access-logs-bucket",
      "TargetPrefix": "logs/my-data-bucket/"
    }
  }'

# Kiểm tra cấu hình
aws s3api get-bucket-logging --bucket my-data-bucket
```

### Bật qua CloudFormation

```yaml
MySourceBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: my-data-bucket
    LoggingConfiguration:
      DestinationBucketName: !Ref MyLogBucket
      LogFilePrefix: logs/my-data-bucket/

MyLogBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: my-access-logs-bucket
    AccessControl: LogDeliveryWrite  # Cấp quyền cho Log Delivery
    LifecycleConfiguration:
      Rules:
        - Id: ExpireOldLogs
          Status: Enabled
          ExpirationInDays: 90  # Xóa log sau 90 ngày
```

---

## 📋 Format Log — Định Dạng Nhật Ký

Mỗi dòng log là một request, các trường cách nhau bằng dấu cách. Các trường không có giá trị hiển thị `-`.

### Ví Dụ Dòng Log Thực Tế

```
79a59df900b949e55d96a1e698fbacedfd6e09d98eacf8f8d5218e7cd47ef2be \
my-data-bucket [06/Feb/2019:00:00:38 +0000] \
192.0.2.3 \
arn:aws:iam::123456789012:user/alice \
3E57427F3EXAMPLE REST.GET.OBJECT photos/puppy.jpg \
"GET /my-data-bucket/photos/puppy.jpg?X-Amz-Security-Token=... HTTP/1.1" \
200 - 2797690 2797690 2699 1820 \
"-" \
"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_2) ..." \
- \
kDneHhmvQB0vFCGUhKLB7fTnfOgU7Y2vgzDkiWMoXfJaRwqYCkiSDgOl0Dq2G \
SigV4 ECDHE-RSA-AES128-GCM-SHA256 \
AuthHeader my-data-bucket.s3.amazonaws.com \
TLS_AES_128_GCM_SHA256
```

### Các Trường Quan Trọng

| Vị Trí | Tên Trường | Ví Dụ | Mô Tả |
|---------|------------|-------|-------|
| 1 | `bucket_owner` | `79a59...` | Canonical ID của bucket owner |
| 2 | `bucket` | `my-data-bucket` | Tên bucket |
| 3 | `time` | `[06/Feb/2019:00:00:38 +0000]` | Thời gian request |
| 4 | `remote_ip` | `192.0.2.3` | IP của client |
| 5 | `requester` | `arn:aws:iam::...` | IAM ARN hoặc `-` nếu anonymous |
| 6 | `request_id` | `3E57427F3EXAMPLE` | Request ID duy nhất |
| 7 | `operation` | `REST.GET.OBJECT` | Loại API call |
| 8 | `key` | `photos/puppy.jpg` | Object key |
| 9 | `request_uri` | `"GET /... HTTP/1.1"` | Full HTTP request |
| 10 | `http_status` | `200` | HTTP status code |
| 11 | `error_code` | `-` | S3 error code nếu có |
| 12 | `bytes_sent` | `2797690` | Bytes gửi về client |
| 13 | `object_size` | `2797690` | Kích thước object |
| 14 | `total_time` | `2699` | Tổng thời gian xử lý (ms) |
| 15 | `turn_around_time` | `1820` | Thời gian S3 xử lý (ms) |
| 16 | `referer` | `"-"` | HTTP Referer header |
| 17 | `user_agent` | `"Mozilla/5.0..."` | User-Agent |
| 18 | `version_id` | `-` | Version ID của object |
| 19 | `host_id` | `kDneH...` | Extended request ID |
| 20 | `signature_version` | `SigV4` | SigV2 hoặc SigV4 |
| 21 | `cipher_suite` | `ECDHE-RSA-AES128...` | TLS cipher |
| 22 | `authentication_type` | `AuthHeader` | Loại authentication |
| 23 | `host_header` | `my-data-bucket.s3...` | Host header |
| 24 | `tls_version` | `TLS_AES_128_GCM_SHA256` | TLS version |

---

## 🔍 Phân Tích Log với Amazon Athena

Athena — Dịch Vụ Query Không Máy Chủ — cho phép chạy SQL trực tiếp trên S3 log files.

### Bước 1: Tạo Athena Table

```sql
CREATE EXTERNAL TABLE s3_access_logs (
  bucket_owner STRING,
  bucket       STRING,
  request_time STRING,
  remote_ip    STRING,
  requester    STRING,
  request_id   STRING,
  operation    STRING,
  key          STRING,
  request_uri  STRING,
  http_status  INT,
  error_code   STRING,
  bytes_sent   BIGINT,
  object_size  BIGINT,
  total_time   INT,
  turn_around  INT,
  referer      STRING,
  user_agent   STRING,
  version_id   STRING,
  host_id      STRING,
  sig_version  STRING,
  cipher_suite STRING,
  auth_type    STRING,
  host_header  STRING,
  tls_version  STRING
)
ROW FORMAT REGEX
'([^ ]*) ([^ ]*) \[([^\]]*)\] ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) (\"[^\"]*\"|-) (-|[0-9]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) (\"[^\"]*\"|-) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*)'
LOCATION 's3://my-access-logs-bucket/logs/my-data-bucket/'
TBLPROPERTIES ('has_encrypted_data'='false');
```

### Bước 2: Câu Query Thường Dùng

**Tìm top 10 IP truy cập nhiều nhất:**

```sql
SELECT remote_ip, COUNT(*) AS request_count
FROM s3_access_logs
WHERE bucket = 'my-data-bucket'
GROUP BY remote_ip
ORDER BY request_count DESC
LIMIT 10;
```

**Tìm tất cả lỗi 403 Forbidden trong 24 giờ qua:**

```sql
SELECT request_time, remote_ip, requester, key, error_code
FROM s3_access_logs
WHERE bucket = 'my-data-bucket'
  AND http_status = 403
  AND parse_datetime(request_time, 'dd/MMM/yyyy:HH:mm:ss Z')
      > current_timestamp - interval '24' hour
ORDER BY request_time DESC;
```

**Tìm object bị xóa (DELETE operations):**

```sql
SELECT request_time, requester, key, http_status
FROM s3_access_logs
WHERE bucket = 'my-data-bucket'
  AND operation = 'REST.DELETE.OBJECT'
ORDER BY request_time DESC
LIMIT 100;
```

**Tính tổng lưu lượng download theo ngày:**

```sql
SELECT
  substr(request_time, 1, 11) AS date,
  SUM(bytes_sent) / 1073741824.0 AS total_gb_downloaded,
  COUNT(*) AS total_requests
FROM s3_access_logs
WHERE bucket = 'my-data-bucket'
  AND operation = 'REST.GET.OBJECT'
  AND http_status = 200
GROUP BY substr(request_time, 1, 11)
ORDER BY date DESC;
```

**Tìm user-agent bất thường (không phải browser hay AWS SDK):**

```sql
SELECT user_agent, COUNT(*) AS count
FROM s3_access_logs
WHERE bucket = 'my-data-bucket'
  AND user_agent NOT LIKE '%Mozilla%'
  AND user_agent NOT LIKE '%aws-sdk%'
  AND user_agent NOT LIKE '%boto%'
  AND user_agent != '-'
GROUP BY user_agent
ORDER BY count DESC
LIMIT 20;
```

---

## 💰 Quản Lý Chi Phí Log

Log S3 có thể tích lũy nhanh chóng, đặc biệt bucket có nhiều request.

**Ước tính kích thước log:**

```
Một request → ~400-600 bytes log
100,000 requests/ngày → ~50 MB/ngày → ~1.5 GB/tháng → ~$0.035/tháng
10,000,000 requests/ngày → ~5 GB/ngày → ~150 GB/tháng → ~$3.45/tháng
```

**Lifecycle rule để tự động xóa log cũ:**

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-access-logs-bucket \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "ExpireAccessLogs",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Expiration": {"Days": 90},
        "NoncurrentVersionExpiration": {"NoncurrentDays": 1}
      }
    ]
  }'
```

---

## 🔒 Bảo Mật Log

### Bảo Vệ Log Khỏi Bị Xóa

```bash
# Bật versioning trên log bucket
aws s3api put-bucket-versioning \
  --bucket my-access-logs-bucket \
  --versioning-configuration Status=Enabled

# Bật Object Lock (WORM) nếu cần compliance
aws s3api put-object-lock-configuration \
  --bucket my-access-logs-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 365
      }
    }
  }'
```

### Encrypt Log

```bash
# Bật SSE-KMS cho log bucket
aws s3api put-bucket-encryption \
  --bucket my-access-logs-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/your-key-id"
      }
    }]
  }'
```

---

## ⚖️ So Sánh: Server Access Logging vs CloudTrail

| Tiêu Chí | S3 Server Access Logging | CloudTrail Data Events |
|---------|------------------------|----------------------|
| **Loại events** | Tất cả HTTP requests | S3 API calls |
| **Độ chi tiết** | URL đầy đủ, byte count | API call level |
| **Độ trễ** | Vài giờ | ~15 phút |
| **Chi phí** | Chỉ lưu trữ log | Tốn phí per event (~$0.10/100k events) |
| **Query** | Athena (tự build table) | CloudTrail Lake (managed) |
| **Anonymous access** | Có ghi lại | Có ghi lại |
| **Completeness** | Best-effort | Best-effort |
| **Dùng cho** | Access pattern, billing debug | Security audit, compliance |

> **Khi nào dùng cái nào?**
> - Debug "tại sao request này bị 403?" → Server Access Logging
> - Audit "ai đã tạo/xóa bucket?" → CloudTrail Management Events
> - Điều tra "ai đã download file nhạy cảm?" → CloudTrail Data Events (chính xác hơn, có thể query nhanh qua CloudTrail Lake)

---

## 📝 Câu Hỏi Phỏng Vấn

**Q: S3 Server Access Logging và CloudTrail khác nhau thế nào?**
A: Server Access Logging ghi mọi HTTP request (kể cả anonymous) với thông tin chi tiết như byte count, latency. CloudTrail ghi API calls vào AWS API, không phải direct HTTP. Để điều tra bảo mật đầy đủ cần cả hai: Server Access Logging cho data access pattern, CloudTrail cho management actions.

**Q: Làm sao phân tích log S3 hiệu quả nhất?**
A: Dùng Athena với table được tạo trên log bucket. Athena serverless nên không tốn phí server, chỉ trả phí theo lượng data scan. Nên partition table theo ngày để giảm data scan và chi phí.

**Q: Nếu bucket có 1 tỷ request/ngày, làm sao manage log?**
A: Dùng lifecycle rule xóa log sau 30-90 ngày. Partition Athena table theo ngày (`/year=2026/month=05/day=16/`). Compress log với S3 intelligent-tiering. Chỉ enable Data Events trong CloudTrail cho bucket thực sự cần audit, không bật hết.

---

**Tiếp Theo:** [5-cloudtrail-for-storage.md](5-cloudtrail-for-storage.md) — CloudTrail audit trail và security investigation

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
