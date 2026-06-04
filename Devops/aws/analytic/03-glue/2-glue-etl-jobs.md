# ⚙️ AWS Glue ETL Jobs — Công Việc ETL Serverless

> Glue ETL Jobs là nơi dữ liệu thực sự được transform (chuyển đổi) — từ raw JSON/CSV sang Parquet tối ưu, từ nhiều nguồn sang kho dữ liệu, từ dữ liệu bẩn sang dữ liệu sạch. Chạy trên Apache Spark serverless — không cần quản lý cluster.

## 📚 Mục Lục

1. [ETL Job Là Gì](#etl-job-là-gì)
2. [Các Loại Job](#các-loại-job)
3. [DPU và Mô Hình Tính Phí](#dpu-và-mô-hình-tính-phí)
4. [DynamicFrame vs DataFrame](#dynamicframe-vs-dataframe)
5. [Viết ETL Job Thực Tế](#viết-etl-job-thực-tế)
6. [Job Bookmarks](#job-bookmarks)
7. [Glue Studio — Visual ETL](#glue-studio--visual-etl)
8. [Tối Ưu Hiệu Suất và Chi Phí](#tối-ưu-hiệu-suất-và-chi-phí)
9. [Monitoring và Debugging](#monitoring-và-debugging)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 ETL Job Là Gì

**ETL — Extract, Transform, Load (Trích Xuất, Chuyển Đổi, Tải):**

```
EXTRACT (Trích Xuất)         TRANSFORM (Chuyển Đổi)          LOAD (Tải)
─────────────────────        ──────────────────────────       ─────────────────
S3 raw JSON files    ──►     Clean null values          ──►  S3 Parquet files
RDS production DB    ──►     Convert data types         ──►  Redshift table
DynamoDB streams     ──►     Join/aggregate data         ──►  OpenSearch index
Kinesis stream       ──►     Enrich with lookups         ──►  DynamoDB table
```

**Glue ETL Job** = Script chạy trên Apache Spark cluster được AWS quản lý, thực hiện ETL logic.

### Vòng Đời Một ETL Job

```
Trigger (Kích Hoạt) Job
    │ (manual / scheduled / event-based)
    ▼
AWS khởi động Spark cluster (cold start ~2 phút)
    │
    ▼
Job Script chạy:
    ├── Extract: đọc dữ liệu từ source
    ├── Transform: xử lý, chuyển đổi
    └── Load: ghi vào destination
    │
    ▼
Cluster tự động xóa
    │
    ▼
Glue ghi Job Run metrics vào CloudWatch
```

---

## 🗂️ Các Loại Job

### So Sánh Tổng Quan

| Loại Job              | Engine               | Max Runtime | DPU Tối Thiểu | Tốt Nhất Cho                              |
| ---------------------- | -------------------- | ----------- | -------------- | ----------------------------------------- |
| **Spark (Visual/Script)** | Apache Spark     | 48 giờ      | 2 DPU          | Xử lý GB → TB, transformation phức tạp   |
| **Python Shell**       | Python 3.x           | 3 giờ       | 0.0625 DPU     | Script nhỏ, API calls, file đơn giản      |
| **Spark Streaming**    | Spark Structured Streaming | Vô thời hạn | 2 DPU   | ETL từ Kinesis/Kafka liên tục             |
| **Ray**                | Ray (Python)         | 48 giờ      | 2 DPU          | ML preprocessing, distributed Python     |

---

### 1. Glue Spark Job

**Phù Hợp:** Xử lý dữ liệu từ hàng GB đến hàng TB, transformation phức tạp, join nhiều bảng lớn.

**Ngôn ngữ hỗ trợ:** PySpark (Python) hoặc Scala.

```python
# Cấu trúc cơ bản một Glue Spark Job
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql import functions as F

# ── 1. Khởi tạo Glue Context ──────────────────────────────────
args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_db', 'source_table', 'output_path'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# ── 2. Extract — Đọc từ Glue Data Catalog ─────────────────────
raw_dyf = glueContext.create_dynamic_frame.from_catalog(
    database=args['source_db'],
    table_name=args['source_table'],
    transformation_ctx="raw_dyf"
)
print(f"[EXTRACT] Đọc được {raw_dyf.count()} records")

# ── 3. Transform — Chuyển đổi dữ liệu ────────────────────────
# Convert sang DataFrame để dùng Spark SQL functions
raw_df = raw_dyf.toDF()

transformed_df = raw_df \
    .filter(F.col("user_id").isNotNull()) \
    .withColumn("event_date", F.to_date(F.col("event_timestamp"))) \
    .withColumn("year", F.year(F.col("event_date"))) \
    .withColumn("month", F.lpad(F.month(F.col("event_date")).cast("string"), 2, "0")) \
    .withColumn("day", F.lpad(F.dayofmonth(F.col("event_date")).cast("string"), 2, "0")) \
    .drop("raw_timestamp")

print(f"[TRANSFORM] Sau xử lý còn {transformed_df.count()} records")

# ── 4. Load — Ghi ra S3 dạng Parquet có partition ─────────────
transformed_df.write \
    .mode("append") \
    .partitionBy("year", "month", "day") \
    .parquet(args['output_path'])

print(f"[LOAD] Ghi thành công vào {args['output_path']}")

# ── 5. Hoàn thành job ─────────────────────────────────────────
job.commit()
```

---

### 2. Python Shell Job

**Phù Hợp:** Script Python đơn giản, gọi API, xử lý file nhỏ, task không cần Spark.

**Ưu điểm lớn:** Chỉ 0.0625 DPU (1/32 so với Spark) → **tiết kiệm 96% chi phí** cho task đơn giản.

```python
# Glue Python Shell Job — không cần SparkContext
import boto3
import json
import pandas as pd
from io import BytesIO
from datetime import datetime, timedelta

# Python Shell job chạy như script Python thông thường
s3 = boto3.client('s3')
glue = boto3.client('glue')

def process_config_file():
    """
    Đọc config từ S3, xử lý, ghi lại — không cần Spark
    """
    # Đọc file JSON nhỏ từ S3
    response = s3.get_object(
        Bucket='my-bucket',
        Key='config/service_mapping.json'
    )
    config = json.loads(response['Body'].read())

    # Đọc CSV nhỏ bằng pandas (không cần Spark)
    csv_response = s3.get_object(
        Bucket='my-bucket',
        Key='input/daily_summary.csv'
    )
    df = pd.read_csv(BytesIO(csv_response['Body'].read()))

    # Xử lý đơn giản
    df['service_name'] = df['service_id'].map(config)
    df['processed_at'] = datetime.now().isoformat()

    # Ghi lại S3
    output_buffer = BytesIO()
    df.to_parquet(output_buffer, index=False)
    s3.put_object(
        Bucket='my-bucket',
        Key=f'output/daily_summary_{datetime.now().strftime("%Y%m%d")}.parquet',
        Body=output_buffer.getvalue()
    )
    print(f"Xử lý xong {len(df)} records")

process_config_file()
```

---

### 3. Glue Streaming Job

**Phù Hợp:** ETL liên tục từ Kinesis Data Streams hoặc Apache Kafka (MSK).

```python
# Glue Streaming Job — đọc từ Kinesis, ghi ra S3
from awsglue.context import GlueContext
from awsglue import DynamicFrame
from pyspark.context import SparkContext
from pyspark.sql import functions as F

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

# Kết nối Kinesis Data Stream
kinesis_options = {
    "streamARN": "arn:aws:kinesis:ap-southeast-1:123456789:stream/events-stream",
    "startingPosition": "TRIM_HORIZON",   # Đọc từ đầu (hoặc LATEST)
    "inferSchema": "true",
    "classification": "json"
}

# Tạo streaming DynamicFrame
streaming_dyf = glueContext.create_data_frame.from_options(
    connection_type="kinesis",
    connection_options=kinesis_options,
    transformation_ctx="streaming_dyf"
)

# Định nghĩa transform function cho mỗi micro-batch
def process_batch(data_frame, batch_id):
    if data_frame.count() > 0:
        dyf = DynamicFrame.fromDF(data_frame, glueContext, "transform_node")

        # Transform
        enriched_df = data_frame \
            .filter(F.col("event_type").isNotNull()) \
            .withColumn("processed_at", F.current_timestamp())

        # Ghi ra S3 theo partition giờ
        enriched_df.write \
            .mode("append") \
            .partitionBy("year", "month", "day", "hour") \
            .parquet("s3://my-bucket/streaming-output/")

# Chạy streaming query
glueContext.forEachBatch(
    frame=streaming_dyf,
    batch_function=process_batch,
    options={
        "windowSize": "100 seconds",    # Micro-batch mỗi 100 giây
        "checkpointLocation": "s3://my-bucket/checkpoints/events-job/"
    }
)
```

---

## 💰 DPU và Mô Hình Tính Phí

### DPU — Data Processing Unit (Đơn Vị Xử Lý Dữ Liệu)

```
1 DPU = 4 vCPU + 16 GB RAM

Mỗi Glue Spark Job mặc định: 10 DPU
→ 10 × (4 vCPU + 16 GB RAM) = 40 vCPU + 160 GB RAM
```

### Tính Phí

```
Chi phí = Số DPU × Thời gian chạy (giờ) × Đơn giá

Đơn giá (khu vực ap-southeast-1, tháng 5/2026):
  Glue Spark Job:       $0.44 / DPU-giờ
  Python Shell Job:     $0.44 / DPU-giờ (0.0625 DPU tối thiểu)
  Glue Flex (ưu đãi):   $0.29 / DPU-giờ

Ví dụ:
  Spark Job 10 DPU × 2 giờ = 20 DPU-giờ × $0.44 = $8.80
  Python Shell 0.0625 DPU × 0.5 giờ = 0.03125 DPU-giờ × $0.44 = $0.014
```

### Glue Flex Execution (Thực Thi Linh Hoạt)

```
Glue Flex = Sử dụng Spot Instance capacity dư thừa → giảm ~34% chi phí
Hạn chế:
  - Không đảm bảo thời gian bắt đầu chính xác
  - Job có thể bị interrupted (gián đoạn) nếu capacity không đủ
  - Không phù hợp cho SLA-sensitive jobs (job có yêu cầu thời gian nghiêm ngặt)

Khi nào dùng Flex:
  ✅ Batch job ban đêm, không khẩn cấp
  ✅ Development/testing jobs
  ✅ Data quality profiling jobs
  ❌ Production pipeline cần hoàn thành đúng giờ
```

---

## 🔵 DynamicFrame vs DataFrame

### DynamicFrame (Khung Dữ Liệu Động) — Glue Native

**Ưu điểm:**
- Xử lý **schema linh hoạt** — không enforce schema cứng nhắc
- Tích hợp native với Glue Catalog
- Hỗ trợ **ChoiceType** — khi một cột có nhiều kiểu dữ liệu
- Built-in transforms (ResolveChoice, DropNullFields, ApplyMapping...)

```python
from awsglue.transforms import *

# DynamicFrame xử lý được dữ liệu "lộn xộn"
# Ví dụ: cột "price" có lúc là "10.5" (string), lúc là 10.5 (double)
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="raw", table_name="messy_orders"
)

# ResolveChoice — thống nhất kiểu dữ liệu
# "cast:double" → ép kiểu về double, null nếu không convert được
resolved = ResolveChoice.apply(dyf, specs=[("price", "cast:double")])

# ApplyMapping — đổi tên cột và kiểu dữ liệu
mapped = ApplyMapping.apply(
    frame=resolved,
    mappings=[
        ("order_id", "string", "order_id", "string"),
        ("user_id", "string", "user_id", "string"),
        ("price", "double", "total_price", "double"),      # đổi tên
        ("created_ts", "string", "created_at", "timestamp")  # đổi kiểu
    ]
)

# DropNullFields — xóa cột toàn NULL (schema thừa từ Crawler)
cleaned = DropNullFields.apply(frame=mapped)
```

### DataFrame (Khung Dữ Liệu) — Spark Native

**Ưu điểm:**
- API quen thuộc, phong phú hơn
- Tốt hơn cho complex SQL operations, window functions
- Tối ưu hóa tốt hơn với Catalyst Optimizer (Trình Tối Ưu Catalyst)

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

# Convert DynamicFrame → DataFrame để dùng Spark SQL
df = dyf.toDF()

# Complex transformation với window function
window_spec = Window.partitionBy("user_id").orderBy(F.desc("created_at"))

result_df = df \
    .withColumn("rank", F.rank().over(window_spec)) \
    .filter(F.col("rank") == 1) \
    .drop("rank") \
    .withColumn("year", F.year("created_at")) \
    .withColumn("month", F.lpad(F.month("created_at").cast("string"), 2, "0"))

# Convert ngược lại để dùng glueContext.write_dynamic_frame
from awsglue import DynamicFrame
result_dyf = DynamicFrame.fromDF(result_df, glueContext, "result")
```

### Khi Nào Dùng Cái Nào?

```
DynamicFrame:
  ✅ Đọc từ Glue Catalog (to_catalog)
  ✅ Ghi vào Glue Catalog, S3 qua glueContext
  ✅ Dữ liệu có schema không nhất quán (ChoiceType)
  ✅ Dùng Glue built-in transforms

DataFrame:
  ✅ Complex SQL, window functions
  ✅ ML feature engineering
  ✅ Joins phức tạp với nhiều điều kiện
  ✅ Pandas UDF (User-Defined Function — Hàm Do Người Dùng Định Nghĩa)
```

---

## 🔖 Job Bookmarks (Dấu Trang Job)

**Job Bookmarks** là cơ chế giúp Glue Job nhớ đã xử lý đến đâu — chỉ xử lý dữ liệu **mới** trong lần chạy tiếp theo.

### Cơ Chế Hoạt Động

```
Lần chạy 1 (08:00):
  S3: file1.parquet, file2.parquet
  → Job xử lý file1, file2
  → Bookmark ghi: "đã xử lý đến đây"

Lần chạy 2 (09:00):
  S3: file1.parquet, file2.parquet, file3.parquet (file mới)
  → Job CHỈ xử lý file3.parquet
  → Bookmark cập nhật

→ Không xử lý lại file1, file2 → tiết kiệm thời gian và DPU
```

### Bật Job Bookmarks

```python
# Bật bookmark khi tạo job qua boto3
glue.create_job(
    Name='etl-events-job',
    Role='arn:aws:iam::123456789:role/GlueETLRole',
    Command={
        'Name': 'glueetl',
        'ScriptLocation': 's3://my-scripts/etl_events.py',
        'PythonVersion': '3'
    },
    DefaultArguments={
        '--job-bookmark-option': 'job-bookmark-enable',  # Bật bookmark
        '--enable-metrics': 'true',
        '--enable-continuous-cloudwatch-log': 'true'
    },
    GlueVersion='4.0',
    NumberOfWorkers=10,
    WorkerType='G.1X'
)
```

### Bookmark States (Trạng Thái Bookmark)

```
job-bookmark-enable:   Bật bookmark — chỉ xử lý dữ liệu mới
job-bookmark-disable:  Tắt bookmark — xử lý lại toàn bộ mỗi lần
job-bookmark-pause:    Tạm dừng — ghi nhớ vị trí nhưng xử lý toàn bộ

Reset bookmark khi cần xử lý lại từ đầu:
aws glue reset-job-bookmark --job-name etl-events-job
```

---

## 🎨 Glue Studio — Visual ETL

**Glue Studio** là giao diện đồ họa kéo-thả để thiết kế ETL pipeline mà không cần viết code.

```
┌────────────────────────────────────────────────────────────────┐
│                     GLUE STUDIO CANVAS                          │
│                                                                  │
│  [S3 Source]    [RDS Source]                                     │
│       │               │                                          │
│       └───────┬───────┘                                          │
│               │                                                  │
│          [Join Node]  ← Join hai bảng                           │
│               │                                                  │
│       [Filter Node]   ← Lọc records                             │
│               │                                                  │
│  [ApplyMapping Node]  ← Đổi tên/kiểu cột                       │
│               │                                                  │
│     [S3 Target]       ← Ghi Parquet ra S3                       │
│                                                                  │
│  → Glue Studio tự sinh code PySpark từ visual flow             │
└────────────────────────────────────────────────────────────────┘
```

**Khi nào dùng Glue Studio:**
- Onboarding team member mới chưa biết PySpark
- ETL pipeline đơn giản, không có logic phức tạp
- Prototype nhanh trước khi optimize code

---

## 🚀 Tối Ưu Hiệu Suất và Chi Phí

### Worker Types (Loại Worker)

| Worker Type  | vCPU | RAM    | DPU  | Tốt Cho                                   |
| ------------- | ---- | ------ | ---- | ----------------------------------------- |
| **Standard**  | 4    | 16 GB  | 1    | Workload thông thường                     |
| **G.1X**      | 4    | 16 GB  | 1    | Memory-intensive, thay thế Standard       |
| **G.2X**      | 8    | 32 GB  | 2    | Large dataset, complex joins              |
| **G.4X**      | 16   | 64 GB  | 4    | Very large dataset, ML workloads          |
| **G.8X**      | 32   | 128 GB | 8    | Cực lớn, machine learning heavy           |
| **Z.2X**      | 8    | 64 GB  | 2    | Memory-optimized (Apache Zeppelin)        |

### Số Workers Tối Ưu

```python
# Công thức ước tính số workers cần thiết:
# Số workers = (Tổng dung lượng dữ liệu GB) / (Dung lượng xử lý mỗi worker GB)
# Mỗi G.1X worker xử lý hiệu quả ~5-10 GB

# Ví dụ: 500 GB data, G.1X workers
# → 500 / 10 = 50 workers (ước tính ban đầu)
# → Test thực tế và điều chỉnh

# Dynamic allocation — Spark tự điều chỉnh số executor
spark.conf.set("spark.dynamicAllocation.enabled", "true")
spark.conf.set("spark.dynamicAllocation.minExecutors", "2")
spark.conf.set("spark.dynamicAllocation.maxExecutors", "20")
```

### Kỹ Thuật Tối Ưu

```python
# 1. REPARTITION — cân bằng phân phối dữ liệu giữa các executor
# Khi một số partition quá lớn (skew — lệch)
df = df.repartition(200, "user_id")  # Phân phối theo user_id, 200 partitions

# 2. COALESCE — giảm số partitions (output files nhỏ → ít files)
# Không shuffle data như repartition → nhanh hơn
df = df.coalesce(10)  # Gộp thành 10 files output

# 3. BROADCAST JOIN — join bảng nhỏ (< 200 MB)
# Broadcast bảng nhỏ đến mọi executor → tránh shuffle tốn kém
from pyspark.sql.functions import broadcast

large_df.join(broadcast(small_df), "user_id")

# 4. PREDICATE PUSHDOWN — đẩy filter xuống gần nguồn đọc
# Spark tự động làm, nhưng cần đảm bảo filter trên partition columns
df = spark.read.parquet("s3://bucket/events/") \
    .filter("year='2024' AND month='01'")  # Pushdown filter

# 5. CACHE khi dùng nhiều lần
df.cache()
df.count()  # Trigger materialization
# ... nhiều operations khác trên df ...
df.unpersist()  # Giải phóng bộ nhớ khi xong
```

### Anti-patterns Cần Tránh

```python
# ❌ TRÁNH: collect() toàn bộ data về driver
all_data = df.collect()  # OutOfMemoryError với big data!

# ✅ THAY THẾ: Dùng distributed operations
df.write.parquet("s3://output/")

# ❌ TRÁNH: UDF (User-Defined Function) Python khi có thể dùng Spark built-in
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType

@udf(returnType=StringType())
def slow_upper(s):  # Python UDF chậm hơn 10x so với Spark SQL functions
    return s.upper() if s else None

# ✅ THAY THẾ: Dùng Spark SQL functions
from pyspark.sql.functions import upper
df.withColumn("name_upper", upper(F.col("name")))

# ❌ TRÁNH: Quá nhiều small files input
# S3 có 100,000 files nhỏ 1 KB → Spark tạo 100,000 tasks → overhead lớn

# ✅ THAY THẾ: Compact files trước (dùng EMR hoặc Glue job riêng)
```

---

## 📊 Monitoring và Debugging

### CloudWatch Metrics Quan Trọng

```
glue.driver.aggregate.numFailedTasks   → Số tasks thất bại
glue.driver.aggregate.numCompletedTasks → Số tasks hoàn thành
glue.driver.ExecutorAllocationManager.executors.numberAllExecutors → Số executors
glue.ALL.jvm.heap.used                 → Bộ nhớ JVM đang dùng (cảnh báo nếu > 80%)
glue.driver.BlockManager.disk.diskSpaceUsed_MB → Disk spill (tràn đĩa)
```

### Continuous Logging (Ghi Log Liên Tục)

```python
# Bật trong job arguments
DefaultArguments={
    '--enable-continuous-cloudwatch-log': 'true',
    '--enable-continuous-log-filter': 'true',
    '--continuous-log-logGroup': '/aws-glue/jobs/etl-events-job'
}

# Trong script — log có cấu trúc
import logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

logger.info(f"[EXTRACT] Đọc {record_count} records từ {source_path}")
logger.info(f"[TRANSFORM] Sau filter còn {filtered_count} records")
logger.warning(f"[VALIDATE] Phát hiện {null_count} records có user_id null")
logger.error(f"[LOAD] Lỗi khi ghi vào {target_path}: {error_msg}")
```

### Xử Lý Lỗi và Retry (Thử Lại)

```python
# Cấu hình retry khi tạo job
glue.create_job(
    Name='etl-events-job',
    MaxRetries=2,              # Thử lại tối đa 2 lần nếu job fail
    Timeout=120,               # Timeout sau 120 phút
    NotificationProperty={
        'NotifyDelayAfter': 60  # Cảnh báo nếu job chạy > 60 phút
    },
    ...
)

# Trong script — try/except để log lỗi cụ thể
try:
    result_df = source_df.join(lookup_df, "product_id", "left")
    logger.info(f"Join thành công: {result_df.count()} records")
except Exception as e:
    logger.error(f"Lỗi khi join: {str(e)}")
    raise  # Re-raise để Glue đánh dấu job failed và trigger retry
```

---

## ❓ Câu Hỏi Phỏng Vấn

**Q: Job Bookmark trong Glue là gì? Khi nào nên dùng?**
> Job Bookmark ghi nhớ trạng thái xử lý của job — lần chạy sau chỉ xử lý dữ liệu mới, không xử lý lại dữ liệu cũ. Nên dùng cho incremental ETL jobs (xử lý gia tăng) chạy định kỳ: ví dụ job chạy mỗi giờ để xử lý log mới. Không phù hợp cho full-refresh jobs cần tái xử lý toàn bộ.

**Q: DynamicFrame có ưu điểm gì so với DataFrame trong Glue?**
> DynamicFrame xử lý được schema không nhất quán (ChoiceType — khi cùng một cột có nhiều kiểu dữ liệu khác nhau trong các file khác nhau). DynamicFrame cũng tích hợp sẵn với Glue Catalog và có các built-in transforms như ResolveChoice, ApplyMapping. DataFrame mạnh hơn cho complex transformations, window functions và tối ưu hóa Spark.

**Q: Glue Job bị chậm — bạn sẽ điều tra thế nào?**
> Kiểm tra theo thứ tự: (1) CloudWatch metrics — xem executor utilization, disk spill, GC time; (2) Spark UI — xem job stages, task distribution, skew; (3) Kiểm tra data skew — nếu một partition lớn hơn nhiều lần, repartition; (4) Xem có UDF Python không — thay bằng Spark built-in functions; (5) Kiểm tra số files input — quá nhiều small files → tăng overhead.

**Q: Cách giảm chi phí DPU cho Glue Jobs?**
> (1) Chọn đúng loại job — dùng Python Shell (0.0625 DPU) thay Spark khi dữ liệu nhỏ; (2) Dùng Glue Flex execution cho non-urgent jobs (tiết kiệm 34%); (3) Bật Job Bookmark để không xử lý lại dữ liệu cũ; (4) Đặt số workers hợp lý — đừng dùng 10 DPU mặc định khi 3 DPU đủ; (5) Set job timeout để tránh job treo vô thời hạn tốn phí.

**Q: Khi nào chọn Glue ETL thay vì Lambda để transform dữ liệu?**
> Lambda khi: dữ liệu nhỏ (< vài trăm MB), xử lý event-driven, thời gian chạy < 15 phút, không cần Spark. Glue khi: dữ liệu lớn (GB → TB), cần Apache Spark, thời gian chạy dài, cần join nhiều dataset lớn. Lambda không phù hợp cho big data ETL vì giới hạn memory (10 GB) và timeout (15 phút).

---

**Cập Nhật Lần Cuối:** 2026-05-17
**File:** `03-glue/2-glue-etl-jobs.md`
**Xem Tiếp:** `3-glue-crawlers.md` — Crawlers tự động phát hiện schema
