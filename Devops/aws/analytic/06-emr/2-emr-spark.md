# Apache Spark trên Amazon EMR — Tối Ưu và Thực Chiến

> Apache Spark là framework xử lý dữ liệu phân tán phổ biến nhất trên EMR. Tài liệu này đi sâu vào cách Spark hoạt động, cách tối ưu hiệu suất và các pattern thực chiến trên AWS.

## 📚 Mục Lục

1. [Spark Architecture trên EMR](#spark-architecture-trên-emr)
2. [Deploy Modes — Chế Độ Triển Khai](#deploy-modes)
3. [Spark Configuration Tuning — Tinh Chỉnh Cấu Hình](#spark-configuration-tuning)
4. [Memory Management — Quản Lý Bộ Nhớ](#memory-management)
5. [Partitioning — Phân Vùng Dữ Liệu](#partitioning)
6. [Reading S3 Efficiently — Đọc S3 Hiệu Quả](#reading-s3-efficiently)
7. [Common Performance Issues — Vấn Đề Hiệu Suất Thường Gặp](#common-performance-issues)
8. [Spark Structured Streaming trên EMR](#spark-structured-streaming-trên-emr)
9. [Tích Hợp Glue Data Catalog](#tích-hợp-glue-data-catalog)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ⚡ Spark Architecture trên EMR

### Tổng Quan Luồng Thực Thi

```
User Code (PySpark / Scala)
          │
          ▼
    Spark Driver (Trình Điều Khiển Spark)
    ─ Chạy trên Primary Node (cluster mode)
    ─ Tạo SparkContext / SparkSession
    ─ Phân tích DAG (Directed Acyclic Graph — Đồ Thị Acyclic Có Hướng)
    ─ Lập kế hoạch thực thi (Execution Plan)
          │
          │  (Gửi tasks qua YARN)
          │
    YARN ResourceManager (Trình Quản Lý Tài Nguyên)
          │
    ┌─────┴──────┐
    ▼            ▼
Executor 1   Executor 2   ... (trên Core / Task Nodes)
─ Task Thread Pool (Nhóm Luồng Tác Vụ)
─ JVM Heap Memory (Bộ Nhớ Heap JVM)
─ On-heap / Off-heap cache (Bộ Nhớ Đệm)
```

### DAG — Directed Acyclic Graph (Đồ Thị Acyclic Có Hướng)

Spark biên dịch code thành DAG trước khi thực thi:

```
read(S3)
   │
   ▼
filter(country='VN')      ← Transformation (Phép Biến Đổi) — lazy (lười biếng)
   │
   ▼
groupBy(date)
   │
   ▼
agg(sum(revenue))         ← Action (Hành Động) — trigger thực thi thực sự
   │
   ▼
write(S3)
```

**Stages và Shuffles:**
- **Stage** (Giai Đoạn) — Tập hợp transformations không cần shuffle
- **Shuffle** (Hoán Vị) — Di chuyển dữ liệu giữa executors qua mạng → tốn kém nhất

```
Stage 1: read → filter → map     (no shuffle)
              ↓
         [SHUFFLE — groupBy gây network transfer]
              ↓
Stage 2: aggregate → write        (no shuffle)
```

---

## 🚀 Deploy Modes — Chế Độ Triển Khai

### Client Mode (Chế Độ Máy Khách)

```
Người Dùng (Laptop/EC2) ─── Driver chạy ở đây
                          │
                          │  (Giao tiếp qua mạng)
                          │
                     EMR Cluster ─── Executors
```

- **Driver chạy ngoài cluster** — Trên máy submit job
- Thích hợp cho **interactive development** (phát triển tương tác) — Jupyter, Zeppelin
- **Rủi ro:** Nếu kết nối bị ngắt → Driver chết → Job mất

### Cluster Mode (Chế Độ Cluster) — Khuyến Nghị Production

```
Người Dùng ─── Submit job ─── YARN RM ─── Spawn Driver trên cluster
                                    │
                                    │ Driver + Executors đều trong cluster
                                    │
                               Primary Node (Driver)
                               Core/Task Nodes (Executors)
```

- **Driver chạy trong cluster** — Trên một trong các nodes
- **An toàn hơn** — Không phụ thuộc máy submit
- **Phù hợp cho production** — ETL jobs, batch processing

```bash
# Chạy Spark ở cluster deploy mode (chế độ triển khai cluster)
spark-submit \
  --deploy-mode cluster \
  --master yarn \
  --class com.example.Main \
  s3://my-bucket/jars/app.jar \
  --input s3://my-bucket/input/ \
  --output s3://my-bucket/output/
```

---

## ⚙️ Spark Configuration Tuning — Tinh Chỉnh Cấu Hình

### Cấu Hình Executor (Đơn Vị Thực Thi)

**Công thức tính executor cho instance m5.4xlarge (16 vCPU, 64 GB RAM):**

```
Số executor mỗi node = (vCPU - 1) / executor_cores
                     = (16 - 1) / 5 = 3 executors

Bộ nhớ mỗi executor = (64 GB - overhead) / số_executor
                    = (64 - 2) / 3 ≈ 20 GB
```

> **Lý do trừ 1 vCPU:** Để lại 1 core cho OS và YARN NodeManager daemon.
> **Executor cores = 5:** Là con số thực nghiệm tốt — cân bằng giữa parallelism và HDFS throughput.

```python
# Cấu hình trong SparkSession
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("MyETLJob") \
    .config("spark.executor.cores", "5") \
    .config("spark.executor.memory", "18g") \
    .config("spark.executor.memoryOverhead", "2g") \
    .config("spark.driver.memory", "4g") \
    .config("spark.driver.memoryOverhead", "1g") \
    .config("spark.dynamicAllocation.enabled", "true") \
    .config("spark.dynamicAllocation.minExecutors", "2") \
    .config("spark.dynamicAllocation.maxExecutors", "50") \
    .getOrCreate()
```

---

### Dynamic Allocation — Phân Bổ Động

```
Workload thấp: 2 executors (min)
        │
        ▼
Nhiều task pending: scale up → 20 executors
        │
        ▼
Task hoàn thành, idle: scale down → 2 executors
```

```xml
<!-- emrfs-site.xml hoặc spark-defaults.conf -->
spark.dynamicAllocation.enabled=true
spark.dynamicAllocation.shuffleTracking.enabled=true
spark.dynamicAllocation.minExecutors=2
spark.dynamicAllocation.maxExecutors=100
spark.dynamicAllocation.initialExecutors=5
spark.dynamicAllocation.executorIdleTimeout=60s
spark.dynamicAllocation.schedulerBacklogTimeout=1s
```

**Lưu ý:** `shuffleTracking.enabled=true` cho phép dynamic allocation khi có shuffle data — không cần External Shuffle Service (Dịch Vụ Shuffle Bên Ngoài) như trước.

---

### EMR-Specific Spark Defaults (Mặc Định Đặc Biệt của EMR)

EMR tự động cấu hình một số thông số dựa trên instance type:

```bash
# Xem cấu hình được EMR tự động set
cat /etc/spark/conf/spark-defaults.conf

# Một số thông số EMR tự tối ưu:
# spark.executor.instances      (tính từ số core nodes)
# spark.executor.memory         (tính từ RAM instance)
# spark.executor.cores          (thường = 4 hoặc 5)
# spark.driver.memory
```

---

## 🧠 Memory Management — Quản Lý Bộ Nhớ

### Cấu Trúc Bộ Nhớ Executor

```
┌────────────────────────────────────────────────┐
│            Executor JVM Memory (ví dụ: 20 GB)  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │     Spark Memory (spark.memory.fraction  │  │
│  │            mặc định 60%)                 │  │
│  │                                          │  │
│  │  ┌─────────────────┐  ┌───────────────┐  │  │
│  │  │ Execution Mem   │  │  Storage Mem  │  │  │
│  │  │ (Shuffle, Sort, │  │  (DataFrame   │  │  │
│  │  │  Join, Agg)     │  │   Cache/      │  │  │
│  │  │                 │  │   Persist)    │  │  │
│  │  └─────────────────┘  └───────────────┘  │  │
│  │      (Có thể mượn lẫn nhau)              │  │
│  └──────────────────────────────────────────┘  │
│                                                │
│  ┌──────────────────────────────────────────┐  │
│  │   User Memory — Code, Data Structures    │  │
│  │          (40% còn lại)                   │  │
│  └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
    +
┌──────────────────────┐
│  Memory Overhead     │  (Ngoài JVM — Python worker, native lib)
│  spark.executor.     │  Thường = max(384MB, 10% executor mem)
│  memoryOverhead      │
└──────────────────────┘
```

### Khi Nào Tăng memoryOverhead?

- Dùng **PySpark** với UDF (User-Defined Function — Hàm Tự Định Nghĩa) nặng
- Dùng **Pandas UDF** (Arrow-based)
- Xử lý **nhiều cột** hoặc **schema phức tạp**
- Gặp lỗi **"Container killed by YARN"** (Container Bị YARN Hủy)

```python
# Tăng overhead khi dùng PySpark UDF nặng
spark.conf.set("spark.executor.memoryOverhead", "4g")  # Thay vì 2g
```

---

### Spill — Tràn Bộ Nhớ Sang Đĩa

Khi Execution Memory không đủ, Spark **spill** (tràn) dữ liệu ra đĩa:

```
Execution Memory đầy
        │
        ▼
Serialize (tuần tự hóa) dữ liệu và ghi ra local disk
        │
        ▼
Đọc lại từ đĩa khi cần → chậm hơn 10–100x so với RAM
```

**Dấu hiệu spill:**
- Spark UI → Stage → "Spill (Memory)" hoặc "Spill (Disk)" > 0
- Job chậm bất thường dù data không quá lớn

**Giải pháp:**
1. Tăng `spark.executor.memory`
2. Tăng `spark.memory.fraction` (nhưng ít user memory hơn)
3. Tăng số partitions (chia nhỏ data hơn)
4. Tối ưu join strategy (chiến lược join)

---

## 🔀 Partitioning — Phân Vùng Dữ Liệu

### Số Partitions Tối Ưu

```
Guideline (Hướng Dẫn):
- Mỗi partition nên có 100–200 MB data (sau khi đọc vào memory)
- Số partitions = max(2 × tổng_cores_cluster, data_size_MB / 128)

Ví dụ: 1 TB data, cluster 200 cores
- 1,000,000 MB / 128 = ~8,000 partitions
- 2 × 200 = 400 partitions
→ Dùng max(400, 8000) = 8,000 partitions
```

```python
# Đọc data từ S3 và repartition (phân vùng lại)
df = spark.read.parquet("s3://my-bucket/data/")

# Kiểm tra số partitions hiện tại
print(f"Số partitions: {df.rdd.getNumPartitions()}")

# Tăng partitions — dùng khi data quá lớn cho số partition hiện tại
df_repartitioned = df.repartition(1000)

# Giảm partitions — dùng khi output ra nhiều file nhỏ (small files problem)
df_coalesced = df.coalesce(100)  # coalesce không gây shuffle, nhanh hơn repartition
```

### Skew — Dữ Liệu Lệch

**Vấn đề skew (lệch dữ liệu):**
```
Partition 1: 1 GB  (bình thường)
Partition 2: 1 GB  (bình thường)
Partition 3: 50 GB (SKEWED — quá lớn → executor này chậm hơn nhiều)
Partition 4: 1 GB  (bình thường)

→ Toàn bộ job phải đợi Partition 3 hoàn thành
```

**Giải pháp salt trick (kỹ thuật muối):**

```python
import pyspark.sql.functions as F

# Thêm salt (muối — số ngẫu nhiên) vào key trước khi join
SALT_FACTOR = 100

# Table lớn — thêm salt ngẫu nhiên
large_df = large_df.withColumn(
    "salted_key",
    F.concat(F.col("join_key"), F.lit("_"), (F.rand() * SALT_FACTOR).cast("int"))
)

# Table nhỏ — explode (bùng nổ) để match với mọi salt value
small_df = small_df.withColumn("salt_range", F.array([F.lit(i) for i in range(SALT_FACTOR)]))
small_df = small_df.withColumn("salt", F.explode(F.col("salt_range")))
small_df = small_df.withColumn(
    "salted_key",
    F.concat(F.col("join_key"), F.lit("_"), F.col("salt"))
)

# Join với salted keys
result = large_df.join(small_df, "salted_key")
```

**Giải pháp khác:**

```python
# AQE — Adaptive Query Execution (Thực Thi Truy Vấn Thích Nghi) — tự động xử lý skew
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256mb")
```

---

## 📂 Reading S3 Efficiently — Đọc S3 Hiệu Quả

### Small Files Problem (Vấn Đề File Nhỏ)

```
Thư mục S3 có 100,000 files × 1 KB = 100 MB tổng cộng
→ Spark tạo 100,000 partitions → 100,000 tasks nhỏ → Overhead khổng lồ

vs.

Thư mục S3 có 10 files × 10 MB = 100 MB tổng cộng
→ Spark tạo 10 partitions → 10 tasks → Hiệu quả hơn nhiều
```

**Giải pháp:**

```python
# 1. Dùng Hadoop CombineFileInputFormat — nhóm nhiều file nhỏ vào 1 partition
spark.conf.set("spark.hadoop.mapreduce.input.fileinputformat.split.minsize", str(128 * 1024 * 1024))

# 2. Merge kết quả định kỳ (compaction job)
df = spark.read.parquet("s3://my-bucket/raw-events/")
df.coalesce(100).write.mode("overwrite").parquet("s3://my-bucket/compacted/")

# 3. Dùng Apache Hudi / Iceberg — tự động compaction (nén gộp)
```

### S3 Select Pushdown (Đẩy Lọc Xuống S3)

```python
# Bật S3 Select (Lọc tại S3) — S3 lọc data trước khi gửi về EMR
spark.conf.set("spark.hadoop.fs.s3.select.pushdown.enabled", "true")

# Chỉ đọc columns cần thiết — Column pruning (Cắt Tỉa Cột)
df = spark.read.parquet("s3://my-bucket/data/") \
    .select("user_id", "event_type", "timestamp") \
    .filter("country = 'VN'")  # Filter pushdown — lọc sớm nhất có thể
```

### Partition Pruning — Cắt Tỉa Phân Vùng

```python
# Cấu trúc S3 với Hive-style partitioning (phân vùng kiểu Hive)
# s3://my-bucket/events/year=2024/month=01/day=15/data.parquet

# Spark tự động chỉ đọc partitions cần thiết — không quét toàn bộ
df = spark.read.parquet("s3://my-bucket/events/") \
    .filter("year = 2024 AND month = 1 AND day >= 15")

# Kiểm tra query plan (kế hoạch truy vấn) — phải thấy "PartitionFilters"
df.explain(mode="formatted")
```

---

## 🔧 Common Performance Issues — Vấn Đề Hiệu Suất Thường Gặp

### 1. GC Pressure — Áp Lực Thu Gom Rác

**Triệu chứng:**
- Log có `GC overhead limit exceeded` (Vượt Giới Hạn Chi Phí GC)
- Executor mất nhiều thời gian trong GC thay vì xử lý
- Spark UI hiển thị "GC Time" cao

**Giải pháp:**

```python
# Dùng G1GC (Garbage-First Garbage Collector — Bộ Thu Gom Rác Ưu Tiên)
spark.conf.set("spark.executor.extraJavaOptions",
    "-XX:+UseG1GC -XX:G1HeapRegionSize=16m "
    "-XX:InitiatingHeapOccupancyPercent=35 "
    "-XX:+PrintGCDetails -XX:+PrintGCDateStamps")

# Giảm số objects bằng cách dùng primitive types (kiểu nguyên thủy)
# Thay List[String] bằng Array[String], tránh boxing/unboxing
```

### 2. OOM — Out of Memory (Hết Bộ Nhớ)

**Các loại OOM:**

```
1. Java heap OOM → Tăng spark.executor.memory
2. Off-heap OOM → Tăng spark.executor.memoryOverhead
3. Driver OOM → Tăng spark.driver.memory (thường do collect() quá nhiều data)
```

```python
# Tránh collect() trên dataset lớn (đưa toàn bộ data về Driver)
# BAD:
all_data = df.collect()  # OOM nếu data lớn hơn Driver memory

# GOOD:
df.write.parquet("s3://my-bucket/output/")  # Ghi trực tiếp từ Executors
# Hoặc dùng limit() nếu chỉ cần sample
sample = df.limit(1000).collect()
```

### 3. Shuffle Bottleneck — Tắc Nghẽn Shuffle

**Triệu chứng:**
- Spark UI: Stage tốn nhiều thời gian ở "shuffle write" / "shuffle read"
- Network bandwidth cao bất thường

**Giải pháp:**

```python
# 1. Broadcast Join — Join nhỏ (Broadcast Join — Phát Sóng Join)
# Khi một bảng nhỏ (< broadcast threshold), Spark gửi bản copy đến mọi executor
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", str(50 * 1024 * 1024))  # 50 MB

# Manual broadcast hint (gợi ý phát sóng thủ công)
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "join_key")

# 2. Tăng số shuffle partitions cho dữ liệu lớn
spark.conf.set("spark.sql.shuffle.partitions", "2000")  # Mặc định 200

# 3. AQE tự động điều chỉnh shuffle partitions (Spark 3.0+)
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
```

---

## 🌊 Spark Structured Streaming trên EMR

### Đọc từ Kinesis (Nguồn Streaming)

```python
# Đọc từ Kinesis Data Streams qua Kinesis-Spark connector
kinesis_df = spark \
    .readStream \
    .format("kinesis") \
    .option("streamName", "my-event-stream") \
    .option("startingPosition", "latest") \
    .option("region", "us-east-1") \
    .load()

# Xử lý
parsed_df = kinesis_df \
    .select(F.from_json(
        F.col("data").cast("string"),
        schema=event_schema
    ).alias("event")) \
    .select("event.*")

# Ghi ra S3 với Checkpoint (điểm kiểm tra — để recovery)
query = parsed_df \
    .writeStream \
    .format("parquet") \
    .option("path", "s3://my-bucket/streaming-output/") \
    .option("checkpointLocation", "s3://my-bucket/checkpoints/stream-1/") \
    .trigger(processingTime="1 minute") \
    .start()

query.awaitTermination()
```

---

## 🔗 Tích Hợp Glue Data Catalog

```python
# Dùng Glue Data Catalog (Danh Mục Dữ Liệu Glue) như Hive Metastore
spark = SparkSession.builder \
    .appName("EMR-Glue-Integration") \
    .config("spark.sql.catalogImplementation", "hive") \
    .config("hive.metastore.client.factory.class",
            "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory") \
    .enableHiveSupport() \
    .getOrCreate()

# Bây giờ có thể query Glue Catalog tables như Hive tables
spark.sql("SHOW DATABASES").show()
spark.sql("USE my_database")
spark.sql("SELECT * FROM my_table LIMIT 10").show()

# Tạo table mới trong Glue Catalog từ Spark
df.write \
    .mode("overwrite") \
    .format("parquet") \
    .partitionBy("year", "month") \
    .saveAsTable("my_database.processed_events")
```

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: Spark Driver và Executor khác nhau thế nào?**

> Driver là tiến trình điều phối — phân tích code, xây dựng DAG (Đồ Thị Thực Thi Có Hướng), phân chia thành stages và tasks, theo dõi tiến độ. Executor là tiến trình thực thi thực sự — nhận tasks từ Driver, đọc/ghi dữ liệu, tính toán và trả kết quả. Trong `cluster` deploy mode, Driver chạy trên cluster; trong `client` mode, Driver chạy trên máy submit job.

**Q: Tại sao Shuffle (Hoán Vị) là operation tốn kém nhất?**

> Shuffle đòi hỏi di chuyển dữ liệu giữa tất cả Executors qua mạng (network transfer). Dữ liệu phải được serialize (tuần tự hóa), ghi ra đĩa (shuffle write), gửi qua network, và đọc lại (shuffle read). Các operations gây shuffle: `groupBy`, `join`, `distinct`, `repartition`. Cách giảm: dùng broadcast join khi một bảng nhỏ, tăng số partitions để mỗi partition gọn hơn, dùng AQE để tự động tối ưu.

**Q: Khi nào dùng `coalesce` thay vì `repartition`?**

> `coalesce(n)` giảm số partitions mà không gây shuffle — nhanh hơn nhưng phân phối không đều. Dùng khi cần giảm số output files trước khi ghi. `repartition(n)` luôn gây shuffle — tốn hơn nhưng phân phối đều. Dùng khi cần tăng parallelism hoặc cân bằng dữ liệu giữa các partitions. Nguyên tắc: `coalesce` để giảm, `repartition` để tăng hoặc phân phối lại theo column.

**Q: AQE — Adaptive Query Execution là gì và khi nào dùng?**

> AQE (Adaptive Query Execution — Thực Thi Truy Vấn Thích Nghi) là tính năng của Spark 3.0+ cho phép tối ưu kế hoạch thực thi dựa trên thống kê thu thập tại runtime (thời gian chạy thực tế). Ba tính năng chính: (1) tự động gộp shuffle partitions nhỏ (coalesce), (2) tự động chuyển sort-merge join thành broadcast join khi thấy một bảng nhỏ, (3) tự động xử lý data skew. Bật bằng `spark.sql.adaptive.enabled=true`.

---

**Cập Nhật Lần Cuối:** 2026-05-17
**Trạng Thái:** ✅ Hoàn thành
