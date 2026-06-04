# 🕷️ AWS Glue Crawlers — Trình Thu Thập Tự Động Schema

> Glue Crawlers là "thám tử" tự động — kết nối đến data store, đọc mẫu dữ liệu, suy luận schema và phân vùng, rồi cập nhật tất cả vào Glue Data Catalog. Không cần viết DDL thủ công, không cần biết trước cấu trúc dữ liệu.

## 📚 Mục Lục

1. [Crawler Là Gì](#crawler-là-gì)
2. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
3. [Cấu Hình Crawler](#cấu-hình-crawler)
4. [Data Sources Được Hỗ Trợ](#data-sources-được-hỗ-trợ)
5. [Schema Inference](#schema-inference)
6. [Partition Discovery](#partition-discovery)
7. [Schema Change Behavior](#schema-change-behavior)
8. [Lịch Chạy và Event-driven](#lịch-chạy-và-event-driven)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Crawler Là Gì

**Glue Crawler** là chương trình tự động:

```
Bước 1: Kết nối đến data store (S3, RDS, DynamoDB, ...)
    │
    ▼
Bước 2: Liệt kê (list) các file/table tồn tại
    │
    ▼
Bước 3: Đọc mẫu (sample) dữ liệu để suy luận schema
    │
    ▼
Bước 4: Phát hiện partitions (phân vùng) từ cấu trúc folder
    │
    ▼
Bước 5: Cập nhật/tạo mới tables trong Glue Data Catalog
    │
    ▼
Kết quả: Athena, Redshift, EMR có thể query ngay
```

### Vấn Đề Crawler Giải Quyết

| Tình Huống                                    | Không Có Crawler               | Có Crawler                      |
| --------------------------------------------- | ------------------------------- | --------------------------------- |
| 500 bảng CSV mới từ hệ thống ERP              | Viết 500 CREATE TABLE thủ công  | Một Crawler chạy 5-10 phút xong  |
| Schema thêm cột mới mỗi tuần                  | Cập nhật DDL thủ công mỗi lần  | Crawler tự phát hiện, tự cập nhật|
| Mới nhận data từ đối tác, không biết schema   | Phải đọc tài liệu, guess format | Crawler đọc và suy luận tự động  |
| S3 có hàng nghìn partitions theo ngày          | Đăng ký từng partition thủ công | Crawler phát hiện và đăng ký tất cả |

---

## ⚙️ Cơ Chế Hoạt Động

### Thuật Toán Crawler

```
1. CLASSIFY (Phân Loại):
   Crawler đọc vài KB đầu của file → xác định format:
   - JSON, CSV, Parquet, ORC, Avro, XML, Ion
   - Gzip compressed? Snappy? Uncompressed?
   - Custom classifiers nếu format lạ

2. INFER SCHEMA (Suy Luận Schema):
   Crawler đọc mẫu dữ liệu (thường 2 MB đầu hoặc 10 rows):
   - Xác định tên cột (header của CSV, key của JSON)
   - Suy luận kiểu dữ liệu (string, int, double, boolean, timestamp)
   - Xử lý nested types (struct, array, map) trong JSON/Parquet

3. DISCOVER PARTITIONS (Phát Hiện Phân Vùng):
   Phân tích cấu trúc folder S3:
   s3://bucket/data/year=2024/month=01/day=01/ → partition: year, month, day
   s3://bucket/data/2024/01/01/               → partition: dạng date (tự suy luận)

4. GROUP FILES (Nhóm File):
   Nhóm các file có cùng schema vào cùng một table
   File khác schema → table mới
```

### Sampling Strategy (Chiến Lược Lấy Mẫu)

```
Glue Crawler không đọc toàn bộ dữ liệu — chỉ lấy mẫu:

S3 folder với 10,000 files:
  → Crawler đọc: đầu tiên ~2 MB của mỗi file, tối đa vài file mỗi partition
  → KHÔNG đọc toàn bộ → Crawler chạy nhanh kể cả với data lake lớn

Hệ quả:
  ✅ Chạy nhanh (vài phút cho hàng TB dữ liệu)
  ⚠️ Schema inference có thể không chính xác 100% với dữ liệu không đồng đều
     → Giải pháp: dùng Custom Classifier hoặc định nghĩa schema thủ công
```

---

## 🔧 Cấu Hình Crawler

### Tạo Crawler Qua Console / AWS CLI

```bash
# Tạo Crawler qua AWS CLI
aws glue create-crawler \
    --name "events-crawler" \
    --role "arn:aws:iam::123456789:role/GlueCrawlerRole" \
    --database-name "analytics_raw" \
    --targets '{
        "S3Targets": [
            {
                "Path": "s3://my-data-lake/raw/events/",
                "Exclusions": ["**/_temporary/**", "**/.spark-staging/**"]
            }
        ]
    }' \
    --schema-change-policy '{
        "UpdateBehavior": "UPDATE_IN_DATABASE",
        "DeleteBehavior": "LOG"
    }' \
    --recrawl-policy '{"RecrawlBehavior": "CRAWL_NEW_FOLDERS_ONLY"}' \
    --schedule "cron(0 2 * * ? *)"  # Chạy lúc 2:00 AM mỗi ngày
```

### Tạo Crawler Qua boto3 (Python SDK)

```python
import boto3

glue = boto3.client('glue', region_name='ap-southeast-1')

glue.create_crawler(
    Name='orders-crawler',
    Role='arn:aws:iam::123456789:role/GlueCrawlerRole',
    DatabaseName='analytics_processed',
    Description='Crawler cho bảng orders sau ETL',

    # Data sources
    Targets={
        'S3Targets': [
            {
                'Path': 's3://my-data-lake/processed/orders/',
                'Exclusions': [
                    '**/_SUCCESS',
                    '**/*.crc',
                    '**/.spark-*'
                ]
            }
        ]
    },

    # Hành vi khi schema thay đổi
    SchemaChangePolicy={
        'UpdateBehavior': 'UPDATE_IN_DATABASE',  # Cập nhật schema
        'DeleteBehavior': 'LOG'                   # Log thay vì xóa
    },

    # Chỉ crawl folder mới (tăng tốc)
    RecrawlPolicy={
        'RecrawlBehavior': 'CRAWL_NEW_FOLDERS_ONLY'
    },

    # Table prefix để phân biệt (tùy chọn)
    TablePrefix='processed_',

    # Lịch crawl
    Schedule='cron(30 3 * * ? *)',  # 3:30 AM mỗi ngày

    # Tags quản lý
    Tags={
        'Environment': 'production',
        'Team': 'data-engineering'
    }
)

print("Crawler 'orders-crawler' đã được tạo!")
```

---

## 🗂️ Data Sources Được Hỗ Trợ

### 1. Amazon S3 (Phổ Biến Nhất)

```python
# S3 targets — hỗ trợ nhiều paths
Targets={
    'S3Targets': [
        {'Path': 's3://bucket/raw/events/'},
        {'Path': 's3://bucket/raw/users/'},
        {
            'Path': 's3://bucket/raw/logs/',
            'Exclusions': [
                '**/archive/**',     # Bỏ qua folder archive
                '**/*.json.gz.tmp',  # Bỏ qua file đang ghi
                '**/year=2020/**'    # Bỏ qua data cũ
            ],
            'SampleSize': 5         # Lấy mẫu 5 file mỗi folder (mặc định: 2)
        }
    ]
}

# Format được hỗ trợ tự động:
# JSON, CSV, Parquet, ORC, Avro, XML, Ion, Grok
```

### 2. JDBC Databases (MySQL, PostgreSQL, SQL Server, Oracle, Redshift)

```python
Targets={
    'JdbcTargets': [
        {
            'ConnectionName': 'prod-mysql-connection',  # Glue Connection đã tạo
            'Path': 'ecommerce/orders',   # database/table hoặc schema/table
            'Exclusions': ['ecommerce/temp_%']  # Bỏ qua bảng temp
        }
    ]
}
```

### 3. Amazon DynamoDB

```python
Targets={
    'DynamoDBTargets': [
        {
            'Path': 'users-table',   # Tên DynamoDB table
            'scanAll': True,         # Quét toàn bộ (chậm nhưng chính xác)
            'scanRate': 0.5          # Sử dụng 50% read capacity
        }
    ]
}
```

### 4. Delta Lake, Apache Iceberg, Apache Hudi (Các Table Format Hiện Đại)

```python
# Crawler hỗ trợ các open table formats
Targets={
    'S3Targets': [
        {
            'Path': 's3://bucket/iceberg-tables/orders/',
            'DlqEventQueueArn': 'arn:aws:sqs:...:glue-dead-letter-queue'
        }
    ]
},
Configuration='{"Version": 1.0, "CrawlerOutput": {"Partitions": {"AddOrUpdateBehavior": "InheritFromTable"}}}'
```

---

## 🧠 Schema Inference (Suy Luận Schema)

### Quy Tắc Suy Luận Kiểu Dữ Liệu

```
Ví dụ Crawler đọc JSON:
{
    "user_id": "u-12345",
    "age": 28,
    "score": 9.5,
    "is_premium": true,
    "created_at": "2024-01-15T10:30:00Z",
    "tags": ["sports", "tech"],
    "address": {"city": "Hanoi", "country": "VN"}
}

Crawler suy luận:
  user_id    → string
  age        → int
  score      → double
  is_premium → boolean
  created_at → string (!) — cần ApplyMapping để convert sang timestamp
  tags       → array<string>
  address    → struct<city:string, country:string>
```

**Lưu ý quan trọng:** Crawler thường suy luận `timestamp` dạng string — ETL Job phải convert sau.

### Custom Classifiers (Bộ Phân Loại Tùy Chỉnh)

Khi format không chuẩn, tạo Custom Classifier để hướng dẫn Crawler:

```python
# Custom Grok Classifier — cho log format tùy chỉnh
glue.create_classifier(
    GrokClassifier={
        'Classification': 'nginx-access-log',
        'Name': 'nginx-log-classifier',
        'GrokPattern': '%{IPORHOST:client_ip} %{USER:ident} %{USER:auth} '
                       '\\[%{HTTPDATE:timestamp}\\] "%{WORD:method} %{URIPATHPARAM:path} '
                       'HTTP/%{NUMBER:http_version}" %{NUMBER:status_code} %{NUMBER:bytes}',
        'CustomPatterns': 'URIPATHPARAM %{URIPATH}(?:%{URIPARAM})?'
    }
)

# Custom JSON Classifier — chỉ crawl một nested path
glue.create_classifier(
    JsonClassifier={
        'Name': 'api-response-classifier',
        'JsonPath': '$.data.items[*]'  # Chỉ lấy phần data.items
    }
)

# Custom CSV Classifier — CSV không có header
glue.create_classifier(
    CsvClassifier={
        'Name': 'fixed-csv-classifier',
        'Delimiter': '|',           # Pipe-delimited
        'QuoteSymbol': '"',
        'ContainsHeader': 'ABSENT', # Không có header row
        'Header': ['user_id', 'event_type', 'timestamp', 'value'],
        'AllowSingleColumn': False,
        'DisableValueTrimming': False
    }
)
```

---

## 📂 Partition Discovery (Phát Hiện Phân Vùng)

### Hive-style Partitions (Phổ Biến Nhất)

```
Cấu trúc folder Hive-style:
s3://bucket/events/year=2024/month=01/day=15/
s3://bucket/events/year=2024/month=01/day=16/
s3://bucket/events/year=2024/month=02/day=01/

Crawler tự động phát hiện:
  Partition keys: year, month, day
  Partition values: (2024, 01, 15), (2024, 01, 16), (2024, 02, 01)
  → Đăng ký vào Glue Catalog ngay
```

### Non-Hive Partitions (Cần Custom Logic)

```
Cấu trúc folder không chuẩn:
s3://bucket/events/2024/01/15/
s3://bucket/events/2024/01/16/

Crawler suy luận (có thể không chính xác):
  partition_0: "2024"
  partition_1: "01"
  partition_2: "15"
  → Tên column không có ý nghĩa

Giải pháp: Sau khi crawl, đổi tên partition columns thủ công
hoặc tổ chức lại folder theo Hive-style
```

### RecrawlPolicy (Chính Sách Tái Crawl)

```python
# CRAWL_EVERYTHING: Crawl toàn bộ, bao gồm folder đã crawl
# → Chậm nhưng đảm bảo phát hiện mọi thay đổi
RecrawlPolicy={'RecrawlBehavior': 'CRAWL_EVERYTHING'}

# CRAWL_NEW_FOLDERS_ONLY: Chỉ crawl folder mới chưa từng crawl
# → Nhanh, phù hợp khi data chỉ append (thêm mới, không sửa cũ)
RecrawlPolicy={'RecrawlBehavior': 'CRAWL_NEW_FOLDERS_ONLY'}

# CRAWL_EVENT_MODE: Dùng S3 event notifications để trigger crawl
# → Gần real-time, không cần schedule cố định
RecrawlPolicy={'RecrawlBehavior': 'CRAWL_EVENT_MODE'}
```

---

## 🔄 Schema Change Behavior (Hành Vi Khi Schema Thay Đổi)

### UpdateBehavior (Hành Vi Cập Nhật)

```
UPDATE_IN_DATABASE (Khuyến Nghị):
  - Thêm column mới → cập nhật table definition
  - Đổi kiểu dữ liệu → cập nhật
  - Phù hợp: schema evolves nhưng vẫn tương thích ngược

LOG (An Toàn Hơn):
  - Phát hiện thay đổi → ghi log vào CloudWatch
  - KHÔNG tự động thay đổi Catalog
  - Phù hợp: production tables quan trọng, không muốn tự động thay đổi
```

### DeleteBehavior (Hành Vi Khi Dữ Liệu Biến Mất)

```
LOG (Mặc Định và An Toàn):
  - Folder/file không còn tồn tại → ghi log
  - Table vẫn tồn tại trong Catalog
  - Phù hợp: Tránh xóa nhầm table đang dùng

DELETE_FROM_DATABASE (Nguy Hiểm):
  - Folder/file không còn → xóa table khỏi Catalog
  - Dùng khi chắc chắn dữ liệu đã bị xóa có chủ đích

DEPRECATE_IN_DATABASE:
  - Đánh dấu table là deprecated (Đã Lỗi Thời)
  - Table vẫn tồn tại nhưng có cảnh báo
```

### Minh Họa Schema Change

```
Scenario: Thêm cột mới vào data từ ngày 01/02/2024

File trước 01/02/2024:
  {"order_id": "1", "user_id": "u1", "amount": 100.0}

File từ 01/02/2024 trở đi:
  {"order_id": "2", "user_id": "u2", "amount": 150.0, "discount_code": "SAVE10"}

Crawler chạy và phát hiện "discount_code" column mới:

Với UPDATE_IN_DATABASE:
  → Table được cập nhật: thêm cột discount_code STRING
  → File cũ (trước 01/02): discount_code = NULL
  → Athena query bình thường, chỉ file cũ thiếu giá trị

Với LOG:
  → Không cập nhật, chỉ log warning
  → Cần DBA/Engineer xem xét và cập nhật thủ công
```

---

## ⏰ Lịch Chạy và Event-driven

### Cron Schedule (Lịch Cố Định)

```bash
# Cron syntax trong Glue (khác Unix cron — có thêm field "Year")
# Cú pháp: cron(Minutes Hours Day-of-month Month Day-of-week Year)

# Mỗi ngày lúc 2:00 AM (UTC)
"cron(0 2 * * ? *)"

# Mỗi giờ vào phút thứ 15
"cron(15 * * * ? *)"

# Thứ Hai đến Thứ Sáu lúc 6:00 AM
"cron(0 6 ? * MON-FRI *)"

# Mỗi 4 giờ
"cron(0 0/4 * * ? *)"
```

### On-Demand (Theo Yêu Cầu)

```python
# Chạy Crawler ngay lập tức
glue.start_crawler(Name='events-crawler')

# Kiểm tra trạng thái
import time

def wait_for_crawler(crawler_name, timeout_minutes=30):
    start_time = time.time()
    while time.time() - start_time < timeout_minutes * 60:
        response = glue.get_crawler(Name=crawler_name)
        state = response['Crawler']['State']

        if state == 'READY':
            last_run = response['Crawler'].get('LastCrawl', {})
            if last_run.get('Status') == 'SUCCEEDED':
                print(f"✅ Crawler hoàn thành thành công")
                return True
            elif last_run.get('Status') == 'FAILED':
                print(f"❌ Crawler thất bại: {last_run.get('ErrorMessage')}")
                return False

        print(f"⏳ Crawler đang {state}...")
        time.sleep(30)

    print("⚠️ Timeout chờ Crawler")
    return False

wait_for_crawler('events-crawler')
```

### Event-driven với Lambda + S3 Event Notification

```
Kiến trúc event-driven:
─────────────────────────────────────────────────────
S3 bucket nhận file mới
    │
    ▼ (S3 Event Notification)
Amazon SQS Queue (hàng đợi thông báo)
    │
    ▼ (trigger)
AWS Lambda Function
    │ → glue.start_crawler(Name='events-crawler')
    ▼
Glue Crawler tự động cập nhật Catalog
    │
    ▼
Athena có thể query dữ liệu mới ngay
─────────────────────────────────────────────────────
```

```python
# Lambda function để trigger Crawler từ S3 event
import boto3
import json

glue = boto3.client('glue')

def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        print(f"File mới: s3://{bucket}/{key}")

    # Trigger crawler (có debounce logic để tránh trigger quá nhiều lần)
    response = glue.get_crawler(Name='events-crawler')
    if response['Crawler']['State'] == 'READY':
        glue.start_crawler(Name='events-crawler')
        print("✅ Đã trigger Crawler")
    else:
        print(f"⏭️ Crawler đang {response['Crawler']['State']}, bỏ qua")

    return {'statusCode': 200}
```

---

## ✅ Best Practices (Thực Hành Tốt Nhất)

### 1. Một Crawler Cho Nhiều Paths Liên Quan

```python
# ✅ TỐT: Một crawler cho toàn bộ raw layer cùng schema
Targets={
    'S3Targets': [
        {'Path': 's3://bucket/raw/events/'},
        {'Path': 's3://bucket/raw/pageviews/'},
        {'Path': 's3://bucket/raw/clicks/'}
    ]
}

# ❌ TRÁNH: Tạo crawler riêng cho mỗi bảng khi không cần thiết
# → Tốn chi phí Crawler DPU, khó quản lý
```

### 2. Exclusion Patterns (Mẫu Loại Trừ) Quan Trọng

```python
# Loại trừ các file không cần thiết
Exclusions=[
    '**/_SUCCESS',          # File Spark success marker
    '**/_committed_*',      # File Spark commit
    '**/*.crc',             # CRC checksum files
    '**/.spark-staging/**', # Spark temporary staging
    '**/_temporary/**',     # Temporary files
    '**/archive/**',        # Data đã archive (nếu không cần index)
    '**/*.tmp'              # File đang được ghi
]
```

### 3. Tách Crawler Theo Layer (Raw / Processed / Aggregated)

```
crawler-raw-data:
  → S3: s3://bucket/raw/**
  → Database: analytics_raw
  → Chạy: Sau mỗi ingestion job

crawler-processed-data:
  → S3: s3://bucket/processed/**
  → Database: analytics_processed
  → Chạy: Sau mỗi ETL job hoàn thành

crawler-aggregated-data:
  → S3: s3://bucket/aggregated/**
  → Database: analytics_aggregated
  → Chạy: Sau mỗi aggregation job
```

### 4. Table Prefix Để Phân Biệt

```python
# Dùng prefix để biết table đến từ đâu
TablePrefix='raw_'
# → raw_events, raw_orders, raw_users

TablePrefix='processed_'
# → processed_events, processed_orders
```

### 5. Chạy Crawler SAU ETL Job (Trong Glue Workflow)

```
Workflow (Luồng Công Việc) Glue:
─────────────────────────────────────────
[Trigger: Daily 2AM]
         │
         ▼
[Glue ETL Job: transform raw → processed]
         │ (chỉ chạy khi Job thành công)
         ▼
[Glue Crawler: crawl processed data]
         │ (cập nhật schema mới vào Catalog)
         ▼
[Glue ETL Job: load processed → aggregated]
         │
         ▼
[Glue Crawler: crawl aggregated data]
─────────────────────────────────────────
```

---

## 🔧 Troubleshooting (Xử Lý Sự Cố)

### Crawler Không Phát Hiện Partitions

```
Triệu Chứng: Crawler chạy nhưng table trong Catalog không có partitions

Nguyên Nhân Phổ Biến:
1. Folder structure không theo Hive-style
   ✗ s3://bucket/2024/01/15/
   ✓ s3://bucket/year=2024/month=01/day=15/

2. Crawler target path quá cụ thể
   ✗ Target: s3://bucket/events/year=2024/
     → Crawler thấy đây là root, không phát hiện partition
   ✓ Target: s3://bucket/events/
     → Crawler thấy cả folder structure bên trong

3. Exclusion pattern quá rộng
   Kiểm tra: Exclusion có vô tình loại trừ partition folder không?
```

### Schema Suy Luận Sai Kiểu Dữ Liệu

```
Triệu Chứng: Cột number bị suy luận thành string

Nguyên Nhân: Mẫu dữ liệu đầu tiên có giá trị trống hoặc "N/A"
  {"price": "N/A"}  → Crawler suy luận price là string
  {"price": 10.5}   → Phần sau của data

Giải Pháp:
1. Dùng Custom Classifier với schema được định nghĩa sẵn
2. Dùng ApplyMapping trong ETL job để ép kiểu
3. Đảm bảo dữ liệu đầu file không có "null string" như "N/A"
```

### Crawler Chạy Lâu Không Xong

```
Kiểm Tra:
1. Số lượng file quá lớn?
   → Dùng CRAWL_NEW_FOLDERS_ONLY hoặc chia nhỏ target paths

2. Có file lớn bất thường?
   → Một file 100 GB sẽ mất nhiều thời gian để đọc sample

3. Exclusion patterns giúp bỏ qua nhiều file không cần thiết?
   → Thêm exclusions cho file không cần catalog

4. Nhiều file nhỏ (small file problem)?
   → Hàng triệu file nhỏ → listing S3 chậm
   → Nên compact files trước
```

### IAM Permission Errors (Lỗi Quyền IAM)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::my-data-lake",
                "arn:aws:s3:::my-data-lake/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "glue:CreateTable",
                "glue:UpdateTable",
                "glue:CreatePartition",
                "glue:BatchCreatePartition",
                "glue:GetDatabase",
                "glue:GetTable"
            ],
            "Resource": [
                "arn:aws:glue:ap-southeast-1:123456789:catalog",
                "arn:aws:glue:ap-southeast-1:123456789:database/*",
                "arn:aws:glue:ap-southeast-1:123456789:table/*/*"
            ]
        }
    ]
}
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Khi nào dùng Crawler, khi nào dùng manual schema definition?**
> **Crawler:** Khi schema chưa biết trước, khi có nhiều bảng cần catalog hóa nhanh, khi schema thường xuyên thay đổi (thêm cột mới). **Manual definition (DDL):** Khi schema cố định và cần kiểm soát chính xác kiểu dữ liệu (ví dụ: timestamp thay vì string), khi cần partition projection, khi Crawler suy luận sai kiểu dữ liệu quan trọng.

**Q: CRAWL_NEW_FOLDERS_ONLY khác gì CRAWL_EVERYTHING?**
> `CRAWL_NEW_FOLDERS_ONLY` chỉ crawl folder chưa từng crawl trước — nhanh hơn nhiều cho data lake lớn chỉ append data mới. `CRAWL_EVERYTHING` crawl lại toàn bộ — đảm bảo phát hiện mọi thay đổi kể cả trong folder cũ nhưng chậm hơn. Dùng `CRAWL_NEW_FOLDERS_ONLY` cho daily/hourly partitioned data, `CRAWL_EVERYTHING` khi cần resync hoàn toàn.

**Q: Crawler phát hiện schema thay đổi — điều gì xảy ra với các query đang chạy?**
> Thay đổi schema trong Glue Catalog không ảnh hưởng đến query đang chạy vì Athena đọc schema tại thời điểm bắt đầu query. Query mới sau khi Catalog cập nhật sẽ thấy schema mới. Cột mới thêm → query cũ không bị lỗi (cột mới = NULL cho file cũ). Cột bị đổi kiểu → có thể gây lỗi nếu query expect kiểu cũ.

**Q: Crawler tính phí như thế nào? Làm sao tối ưu chi phí Crawler?**
> Crawler tính phí theo DPU-giờ (Data Processing Unit — Đơn Vị Xử Lý Dữ Liệu), tương tự ETL job. Tối ưu: (1) Dùng `CRAWL_NEW_FOLDERS_ONLY` thay vì `CRAWL_EVERYTHING`; (2) Gộp nhiều data sources vào một Crawler; (3) Tăng khoảng cách giữa các lần crawl nếu data không thay đổi thường xuyên; (4) Dùng Exclusion patterns để bỏ qua file không cần thiết.

**Q: Làm sao để Crawler và ETL Job chạy theo đúng thứ tự?**
> Dùng **Glue Workflows** — cho phép định nghĩa dependency giữa các Crawlers, ETL Jobs và Triggers. ETL Job chỉ chạy sau khi Crawler thành công, Crawler downstream chỉ chạy sau khi ETL Job thành công. Ngoài ra có thể dùng Step Functions hoặc Apache Airflow (MWAA) để orchestrate phức tạp hơn.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** `03-glue/3-glue-crawlers.md`
**Xem Tiếp:** `4-glue-data-quality.md` — Kiểm tra chất lượng dữ liệu tự động
