# 3 — S3 Làm Nền Tảng Data Lake — Lifecycle, Storage Classes & Tối Ưu Chi Phí

> Amazon S3 (Simple Storage Service — Dịch Vụ Lưu Trữ Đơn Giản) là nền tảng lưu trữ của hầu hết data lake trên AWS. Phần này tập trung vào việc tối ưu chi phí và hiệu suất thông qua storage classes (lớp lưu trữ), lifecycle policies (chính sách vòng đời), và Intelligent-Tiering (Phân Tầng Thông Minh).

---

## 🗂️ S3 Storage Classes — Lớp Lưu Trữ S3

### Tổng Quan Các Lớp Lưu Trữ

```
Chi Phí Lưu Trữ (Storage Cost)
        ▲
        │  S3 Standard ─────────────────────────────── Truy cập thường xuyên
        │  (Tiêu Chuẩn)         ~$0.023/GB/tháng
        │
        │  S3 Intelligent-Tiering ────────────────────── Không rõ pattern
        │  (Phân Tầng Thông Minh) ~$0.023/GB → tự giảm
        │
        │  S3 Standard-IA ────────────────────────────── Ít truy cập (monthly)
        │  (Truy Cập Không Thường Xuyên - Chuẩn)
        │         ~$0.0125/GB/tháng + phí retrieval
        │
        │  S3 One Zone-IA ────────────────────────────── Ít truy cập + có thể tái tạo
        │  (Một Vùng - Ít Truy Cập)
        │         ~$0.01/GB/tháng
        │
        │  S3 Glacier Instant Retrieval ──────────────── Archive, lấy trong ms
        │  (Glacier Truy Xuất Tức Thì)
        │         ~$0.004/GB/tháng
        │
        │  S3 Glacier Flexible Retrieval ─────────────── Archive, lấy trong giờ
        │  (Glacier Truy Xuất Linh Hoạt)
        │         ~$0.0036/GB/tháng
        │
        │  S3 Glacier Deep Archive ───────────────────── Lưu trữ dài hạn (7-10 năm)
        │         ~$0.00099/GB/tháng
        ▼
Chi Phí Lưu Trữ Thấp Nhất
```

### Chi Tiết Từng Lớp

| Storage Class | Độ Trễ Truy Cập | Min Storage | Min Object | Dùng Cho |
| ------------- | --------------- | ----------- | ---------- | -------- |
| **Standard** | Milliseconds | Không | Không | Hot data — đọc hằng ngày |
| **Intelligent-Tiering** | Milliseconds - phút | Không | 128KB | Không rõ access pattern |
| **Standard-IA** | Milliseconds | 30 ngày | 128KB | Warm data — đọc hằng tháng |
| **One Zone-IA** | Milliseconds | 30 ngày | 128KB | Dữ liệu có thể tái tạo |
| **Glacier Instant** | Milliseconds | 90 ngày | 128KB | Archive nhưng cần truy cập nhanh |
| **Glacier Flexible** | Phút - giờ | 90 ngày | 40KB | Archive, truy cập vài lần/năm |
| **Glacier Deep Archive** | 12-48 giờ | 180 ngày | 40KB | Compliance, backup dài hạn |

---

## ♻️ S3 Lifecycle Policies — Chính Sách Vòng Đời

### Lifecycle Policy Là Gì?

**Lifecycle Policy** (Chính Sách Vòng Đời) tự động chuyển đổi (transition) hoặc xóa (expire) objects theo thời gian — không cần can thiệp thủ công.

### Lifecycle Policy Điển Hình Cho Data Lake

```
Ngày 0-30:    S3 Standard         → Data nóng, đọc thường xuyên
Ngày 31-90:   S3 Standard-IA      → Data ấm, đọc hàng tuần/tháng
Ngày 91-365:  S3 Glacier Instant  → Data nguội, truy cập dịp đặc biệt
Sau 365 ngày: S3 Glacier Deep     → Lưu trữ dài hạn / compliance
Sau 7 năm:    DELETE              → Hết retention requirement
```

### Cấu Hình Lifecycle Policy — AWS Console (Giao Diện Đồ Họa)

```json
{
  "Rules": [
    {
      "ID": "DataLakeLifecyclePolicy",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "raw/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    }
  ]
}
```

### Cấu Hình Qua AWS CLI

```bash
# Áp dụng lifecycle policy cho S3 bucket
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-data-lake \
  --lifecycle-configuration file://lifecycle-policy.json

# Kiểm tra lifecycle policy hiện tại
aws s3api get-bucket-lifecycle-configuration \
  --bucket my-data-lake

# Xem lifecycle transitions của một object
aws s3api head-object \
  --bucket my-data-lake \
  --key raw/source=crm/entity=customers/year=2023/month=01/customers.parquet
```

### Lifecycle Policy Theo Zone

```json
{
  "Rules": [
    {
      "ID": "RawZonePolicy",
      "Filter": {"Prefix": "raw/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 180, "StorageClass": "GLACIER_IR"},
        {"Days": 730, "StorageClass": "DEEP_ARCHIVE"}
      ]
    },
    {
      "ID": "ProcessedZonePolicy",
      "Filter": {"Prefix": "processed/"},
      "Transitions": [
        {"Days": 90, "StorageClass": "STANDARD_IA"},
        {"Days": 365, "StorageClass": "GLACIER_IR"}
      ]
    },
    {
      "ID": "CuratedZonePolicy",
      "Filter": {"Prefix": "curated/"},
      "Transitions": [
        {"Days": 180, "StorageClass": "STANDARD_IA"}
      ]
    },
    {
      "ID": "SandboxCleanup",
      "Filter": {"Prefix": "sandbox/"},
      "Expiration": {"Days": 30}
    }
  ]
}
```

---

## 🧠 S3 Intelligent-Tiering — Phân Tầng Thông Minh

### Intelligent-Tiering Hoạt Động Thế Nào?

**S3 Intelligent-Tiering** tự động di chuyển objects giữa các tầng dựa trên access pattern (mẫu truy cập) — không cần viết lifecycle rules thủ công:

```
┌────────────────────────────────────────────────────────────────────┐
│                  S3 Intelligent-Tiering Tiers                      │
│                                                                    │
│  Frequent Access Tier       ← Luôn ở đây khi mới upload           │
│  (Tầng Truy Cập Thường      ~ $0.023/GB/tháng                      │
│   Xuyên)                    [Truy cập bất kỳ lúc nào]             │
│         │ 30 ngày không truy cập                                   │
│         ▼                                                          │
│  Infrequent Access Tier     ← Tự động chuyển xuống                │
│  (Tầng Truy Cập Không        ~ $0.0125/GB/tháng                    │
│   Thường Xuyên)              [Truy cập bất kỳ lúc nào]            │
│         │ 90 ngày không truy cập (opt-in)                          │
│         ▼                                                          │
│  Archive Instant Access Tier                                       │
│  (Tầng Archive Tức Thì)      ~ $0.004/GB/tháng                    │
│         │ 180 ngày không truy cập (opt-in)                         │
│         ▼                                                          │
│  Archive Access Tier         ~ $0.0036/GB/tháng                   │
│  (Tầng Lưu Trữ)              [Cần restore: vài phút-giờ]          │
│         │ 730 ngày không truy cập (opt-in)                         │
│         ▼                                                          │
│  Deep Archive Access Tier    ~ $0.00099/GB/tháng                  │
│  (Tầng Lưu Trữ Sâu)          [Cần restore: 12-48 giờ]             │
└────────────────────────────────────────────────────────────────────┘

Khi có truy cập → Object tự động được đưa trở lại Frequent Access Tier
```

### Bật Intelligent-Tiering

```bash
# Bật Intelligent-Tiering với Archive tiers
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-data-lake \
  --id DataLakeIntelligentTiering \
  --intelligent-tiering-configuration '{
    "Id": "DataLakeIntelligentTiering",
    "Status": "Enabled",
    "OptionalFields": ["BucketKeyEnabled"],
    "Tierings": [
      {
        "Days": 90,
        "AccessTier": "ARCHIVE_ACCESS"
      },
      {
        "Days": 180,
        "AccessTier": "DEEP_ARCHIVE_ACCESS"
      }
    ]
  }'
```

### Khi Nào Dùng Intelligent-Tiering vs Lifecycle Policies

| Tình Huống | Khuyến Nghị |
| ---------- | ----------- |
| Access pattern không rõ ràng | **Intelligent-Tiering** |
| Dữ liệu mới, chưa biết sẽ được truy cập bao nhiêu | **Intelligent-Tiering** |
| Pattern rõ ràng: hot 30 ngày → archive sau đó | **Lifecycle Policy** |
| Object nhỏ (< 128KB) — không đáng để tiering | **Standard** hoặc **Standard-IA** cố định |
| Compliance cần kiểm soát tuyệt đối storage class | **Lifecycle Policy** |

> **Lưu ý chi phí:** Intelligent-Tiering tính thêm phí monitoring ~$0.0025/1000 objects/tháng. Không có ý nghĩa cho objects nhỏ hơn 128KB.

---

## 🔒 S3 Security — Bảo Mật S3 Cho Data Lake

### Encryption (Mã Hóa)

```
Encryption at Rest (Mã Hóa Khi Lưu Trữ):
┌─────────────────────────────────────────────────────┐
│  SSE-S3    — AWS quản lý key hoàn toàn              │
│  SSE-KMS   — Dùng AWS KMS (Key Management Service)  │
│              Kiểm soát được ai dùng key             │
│  SSE-C     — Bạn tự quản lý key, gửi kèm request   │
│  DSSE-KMS  — Dual-layer encryption (2 lớp mã hóa)   │
└─────────────────────────────────────────────────────┘

Encryption in Transit (Mã Hóa Khi Truyền):
- Bắt buộc dùng HTTPS (TLS 1.2+)
- Từ chối HTTP requests bằng bucket policy
```

```json
// Bucket policy bắt buộc HTTPS
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyHTTP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-data-lake",
        "arn:aws:s3:::my-data-lake/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### Bật SSE-KMS Cho Data Lake

```bash
# Đặt default encryption cho bucket
aws s3api put-bucket-encryption \
  --bucket my-data-lake \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789:key/abc-123"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'
```

> **Bucket Key** (Khóa Bucket) giảm chi phí KMS lên đến 99% bằng cách tạo một data key cho toàn bucket thay vì tạo key mới cho mỗi object.

### S3 Block Public Access (Chặn Truy Cập Công Khai)

```bash
# Luôn bật Block Public Access cho data lake bucket
aws s3api put-public-access-block \
  --bucket my-data-lake \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### S3 Object Lock (Khóa Đối Tượng) — Bảo Vệ Raw Zone

```bash
# Bật Object Lock khi tạo bucket (không thể bật sau)
aws s3api create-bucket \
  --bucket my-data-lake-raw \
  --object-lock-enabled-for-bucket \
  --create-bucket-configuration LocationConstraint=us-east-1

# Đặt retention policy: COMPLIANCE mode — không ai có thể xóa trong 1 năm
aws s3api put-object-retention \
  --bucket my-data-lake-raw \
  --key raw/source=crm/entity=orders/year=2024/orders.parquet \
  --retention '{
    "Mode": "COMPLIANCE",
    "RetainUntilDate": "2025-01-15T00:00:00Z"
  }'
```

---

## 📊 S3 Versioning — Phiên Bản Hóa

### Bật Versioning Cho Processed Zone

```bash
aws s3api put-bucket-versioning \
  --bucket my-data-lake \
  --versioning-configuration Status=Enabled

# Lifecycle rule để xóa old versions (phiên bản cũ)
# Giữ lại 3 phiên bản gần nhất, xóa phiên bản cũ hơn 90 ngày
```

```json
{
  "Rules": [
    {
      "ID": "CleanOldVersions",
      "Status": "Enabled",
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "STANDARD_IA"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90,
        "NewerNoncurrentVersions": 3
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

---

## 🏎️ S3 Performance Optimization — Tối Ưu Hiệu Suất

### Transfer Acceleration (Tăng Tốc Truyền Tải)

```bash
# Bật S3 Transfer Acceleration cho upload từ nhiều vùng địa lý
aws s3api put-bucket-accelerate-configuration \
  --bucket my-data-lake \
  --accelerate-configuration Status=Enabled

# Upload qua accelerated endpoint
aws s3 cp large-file.parquet \
  s3://my-data-lake/processed/large-file.parquet \
  --endpoint-url https://my-data-lake.s3-accelerate.amazonaws.com
```

### Multipart Upload (Tải Lên Nhiều Phần)

```python
import boto3
from boto3.s3.transfer import TransferConfig

s3_client = boto3.client('s3')

# Cấu hình multipart upload cho file lớn
config = TransferConfig(
    multipart_threshold=1024 * 25,    # Dùng multipart khi file > 25MB
    max_concurrency=10,               # 10 thread song song
    multipart_chunksize=1024 * 25,    # Mỗi chunk 25MB
    use_threads=True
)

# Upload file lớn với multipart
s3_client.upload_file(
    'large-dataset.parquet',
    'my-data-lake',
    'processed/domain=events/large-dataset.parquet',
    Config=config
)
```

### S3 Select — Lọc Dữ Liệu Ngay Tại S3

**S3 Select** cho phép chạy SQL đơn giản trực tiếp trên object S3 — chỉ trả về dữ liệu cần thiết thay vì tải toàn bộ file:

```python
# Chỉ lấy rows có amount > 1000 từ CSV file
response = s3_client.select_object_content(
    Bucket='my-data-lake',
    Key='raw/source=payments/transactions.csv.gz',
    ExpressionType='SQL',
    Expression="SELECT * FROM s3object WHERE CAST(amount AS FLOAT) > 1000",
    InputSerialization={
        'CSV': {'FileHeaderInfo': 'USE'},
        'CompressionType': 'GZIP'
    },
    OutputSerialization={'CSV': {}}
)

# Đọc kết quả
for event in response['Payload']:
    if 'Records' in event:
        print(event['Records']['Payload'].decode('utf-8'))
```

---

## 💰 Cost Optimization — Tối Ưu Chi Phí

### S3 Cost Calculator Đơn Giản

```
Ước Tính Chi Phí Cho 10TB Data Lake (us-east-1):

Raw Zone (3TB, lưu 1 năm, chuyển sang Glacier sau 90 ngày):
  - 90 ngày × 3TB × $0.023/GB/tháng   = ~$63
  - 275 ngày × 3TB × $0.004/GB/tháng  = ~$34
  Tổng Raw Zone: ~$97/năm

Processed Zone (5TB, Standard-IA sau 60 ngày):
  - 60 ngày × 5TB × $0.023/GB/tháng   = ~$114
  - 305 ngày × 5TB × $0.0125/GB/tháng = ~$580
  Tổng Processed Zone: ~$694/năm

Curated Zone (2TB, Standard):
  - 12 tháng × 2TB × $0.023/GB/tháng  = ~$552/năm

Tổng Ước Tính: ~$1,343/năm (so với ~$2,760/năm nếu toàn Standard)
Tiết Kiệm: ~51%
```

### Công Cụ Kiểm Tra Chi Phí

```bash
# Dùng S3 Storage Lens để phân tích usage
aws s3control get-storage-lens-dashboard \
  --account-id 123456789012 \
  --config-id MyStorageLensDashboard \
  --home-region us-east-1

# Dùng AWS Cost Explorer để xem chi phí theo bucket
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --filter '{
    "Dimensions": {
      "Key": "SERVICE",
      "Values": ["Amazon Simple Storage Service"]
    }
  }' \
  --group-by '[{"Type": "TAG", "Key": "bucket-name"}]' \
  --metrics BlendedCost
```

### Checklist Tối Ưu Chi Phí S3

```
✅ Bật Intelligent-Tiering cho dữ liệu không rõ access pattern
✅ Cấu hình lifecycle policies cho tất cả zones
✅ Bật S3 Bucket Key để giảm chi phí KMS
✅ Xóa incomplete multipart uploads sau 7 ngày
✅ Nén dữ liệu (Snappy/GZIP) trước khi upload
✅ Convert sang columnar format (Parquet) trong Processed Zone
✅ Compact small files định kỳ
✅ Dùng S3 Storage Lens để giám sát usage
✅ Xem xét dùng S3 Express One Zone cho hot data latency-sensitive
```

---

## 🔗 Tích Hợp S3 Với Analytics Services

```
┌─────────────────────────────────────────────────────────────────┐
│                     S3 Data Lake                                │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 Ingestion Layer                         │   │
│  │  Kinesis Firehose → S3   |   Glue Jobs → S3             │   │
│  │  AWS DMS → S3            |   AppFlow → S3               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│  ┌───────────────────────────▼──────────────────────────────┐  │
│  │                 S3 Storage                               │  │
│  │  raw/ → processed/ → curated/                           │  │
│  │  (Với Lake Formation quản lý permissions)                │  │
│  └───────────┬───────────────────────────────┬─────────────┘  │
│              │                               │                  │
│  ┌───────────▼──────────┐    ┌───────────────▼─────────────┐  │
│  │  Query Layer         │    │  Processing Layer            │  │
│  │  • Athena (SQL)      │    │  • Glue ETL Jobs             │  │
│  │  • Redshift Spectrum │    │  • EMR Spark                 │  │
│  └──────────────────────┘    └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### S3 Event Notifications — Kích Hoạt Pipeline Tự Động

```python
# Cấu hình S3 Event Notification để trigger Glue ETL khi có file mới
import boto3

s3_client = boto3.client('s3')

s3_client.put_bucket_notification_configuration(
    Bucket='my-data-lake',
    NotificationConfiguration={
        'LambdaFunctionConfigurations': [
            {
                'LambdaFunctionArn': 'arn:aws:lambda:us-east-1:123456789:function:TriggerGlueJob',
                'Events': ['s3:ObjectCreated:*'],
                'Filter': {
                    'Key': {
                        'FilterRules': [
                            {'Name': 'prefix', 'Value': 'raw/'},
                            {'Name': 'suffix', 'Value': '.parquet'}
                        ]
                    }
                }
            }
        ]
    }
)
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Khi nào dùng S3 Standard-IA thay vì Standard? Những trade-offs là gì?**

> Standard-IA (Infrequent Access — Truy Cập Không Thường Xuyên) rẻ hơn 45% về storage cost nhưng tính thêm phí retrieval ($0.01/GB) và có minimum duration 30 ngày. Phù hợp khi truy cập ít hơn một lần mỗi tháng. Nếu dữ liệu được đọc thường xuyên, Standard rẻ hơn tổng thể. Với data lake, Processed Zone thường phù hợp với Standard-IA (đọc khi cần analytics), còn Curated Zone phụ thuộc — BI dashboards đọc thường xuyên thì nên dùng Standard.

**Q: Giải thích S3 Intelligent-Tiering và khi nào không nên dùng nó?**

> Intelligent-Tiering tự động di chuyển objects giữa tầng hot và cold dựa trên access pattern. Phù hợp khi không biết trước dữ liệu sẽ được truy cập bao nhiêu lần. Không nên dùng cho: (1) Objects nhỏ hơn 128KB — phí monitoring xóa hết lợi thế tiết kiệm; (2) Dữ liệu sẽ bị xóa trong vòng 30 ngày — minimum storage charge; (3) Khi biết rõ access pattern — lifecycle policies rõ ràng và có thể dự đoán chi phí tốt hơn.

**Q: Làm thế nào để ngăn dữ liệu trong Raw Zone bị xóa hoặc sửa?**

> Dùng S3 Object Lock ở chế độ COMPLIANCE (Tuân Thủ) — một khi đặt retention period, kể cả root account cũng không thể xóa. Kết hợp với: (1) S3 Versioning để giữ history; (2) IAM policy từ chối `s3:DeleteObject` và `s3:PutObject` (chỉ allow IAM roles dùng cho ingestion); (3) MFA Delete (Xóa Xác Thực Hai Yếu Tố) để yêu cầu MFA khi thực hiện delete operation.

**Q: S3 Select giải quyết vấn đề gì, và khi nào Athena tốt hơn?**

> S3 Select push down filter xuống tận storage layer — chỉ trả về dữ liệu khớp với điều kiện, giảm bandwidth và chi phí. Phù hợp cho: query đơn giản (filter + project) trên một file, ứng dụng cần low latency. Athena tốt hơn khi: cần JOIN nhiều bảng, aggregate phức tạp, query nhiều files/partitions song song, cần SQL đầy đủ. S3 Select chỉ xử lý một object tại một thời điểm.

---

**Cập Nhật Lần Cuối:** 2026-05-17
