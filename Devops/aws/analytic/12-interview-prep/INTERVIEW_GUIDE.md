# 🎯 Top 20 Câu Hỏi Phỏng Vấn AWS Analytics

> Danh sách 20 câu hỏi thường gặp nhất trong phỏng vấn Data Engineer / Analytics Engineer vị trí liên quan đến AWS Analytics — kèm gợi ý trả lời chi tiết, điểm cần nhấn mạnh và bẫy thường gặp.

## 📚 Mục Lục

- [Nhóm 1: Streaming & Real-time (Câu 1-5)](#nhóm-1-streaming--real-time)
- [Nhóm 2: ETL & Data Catalog (Câu 6-9)](#nhóm-2-etl--data-catalog)
- [Nhóm 3: Query & Data Warehouse (Câu 10-14)](#nhóm-3-query--data-warehouse)
- [Nhóm 4: Kiến Trúc & Thiết Kế (Câu 15-18)](#nhóm-4-kiến-trúc--thiết-kế)
- [Nhóm 5: Chi Phí & Vận Hành (Câu 19-20)](#nhóm-5-chi-phí--vận-hành)

---

## Nhóm 1: Streaming & Real-time

---

### Câu 1: Kinesis Data Streams vs Kinesis Data Firehose — Khi nào dùng cái nào?

**Tại sao hay hỏi:** Đây là câu phân biệt cơ bản nhất trong hệ sinh thái Kinesis — hai dịch vụ có tên gần giống nhau nhưng mục đích khác nhau hoàn toàn.

#### Gợi Ý Trả Lời

**Kinesis Data Streams — KDS (Luồng Dữ Liệu Kinesis):**
- Dịch vụ streaming thô, bạn phải tự quản lý consumer (người tiêu thụ)
- Lưu data trong Shard (Mảnh — đơn vị capacity) từ 1 đến 365 ngày (Extended Retention)
- Throughput (Thông Lượng): 1 MB/s hoặc 1,000 records/s per Shard khi ghi; 2 MB/s per Shard khi đọc
- Nhiều consumer cùng đọc độc lập — mỗi consumer group đọc toàn bộ stream
- Dùng khi: cần xử lý real-time tùy chỉnh, custom business logic, nhiều consumer song song

**Kinesis Data Firehose — KDF (Vòi Dữ Liệu Kinesis):**
- Managed delivery service (Dịch Vụ Giao Nhận Dữ Liệu Được Quản Lý) — không cần quản lý infrastructure
- Tự động buffer, compress, encrypt và deliver vào S3, Redshift, OpenSearch, Splunk
- Không lưu data lâu dài — chỉ buffer tối đa 15 phút hoặc 128 MB trước khi flush
- Dùng khi: chỉ cần đưa data vào storage đích, không cần real-time processing phức tạp

**Khi Nào Chọn Cái Nào:**

| Tiêu Chí | Kinesis Data Streams | Kinesis Firehose |
| --------- | -------------------- | ---------------- |
| Xử lý real-time tùy chỉnh | ✅ | ❌ |
| Nhiều consumer độc lập | ✅ | ❌ (1 destination) |
| Delivery đến S3/Redshift | Cần tự code | ✅ Tự động |
| Latency | Milliseconds | 60 giây — 15 phút |
| Quản lý infrastructure | Phải tự quản lý Shard | Fully managed |
| Chi phí | Shard-hour + dữ liệu | Chỉ theo dung lượng |

**Điểm Cần Nhấn Mạnh:**
> "KDS và KDF thường dùng kết hợp — KDS nhận và xử lý real-time, KDF ghi vào long-term storage. KDS cũng có thể là source cho KDF."

**Bẫy Thường Gặp:**
- Nhầm rằng Firehose = Streams nhanh hơn — thực ra Firehose có built-in buffer delay
- Quên rằng KDS cần bạn tự code consumer (Lambda, KDA, EC2)

---

### Câu 2: Giải thích Shard trong Kinesis Data Streams — Cách tính số Shard cần thiết?

**Tại sao hay hỏi:** Shard là core concept của KDS — sai Shard count dẫn đến throttling (Hạn Chế Tốc Độ) hoặc lãng phí tiền.

#### Gợi Ý Trả Lời

**Shard là gì:**
- Đơn vị capacity cơ bản của KDS — mỗi Shard là một ordered sequence of records (Chuỗi Record Có Thứ Tự)
- Mỗi Shard hỗ trợ:
  - **Ghi (Write):** 1 MB/s hoặc 1,000 records/s (whichever comes first)
  - **Đọc (Read):** 2 MB/s chia đều cho tất cả consumer của Shard đó
- Records cùng Partition Key (Khóa Phân Vùng) luôn vào cùng một Shard → đảm bảo thứ tự per key

**Công Thức Tính Số Shard:**

```
Number of Shards = max(
  ceil(incoming_write_bandwidth_MB_per_s / 1),
  ceil(incoming_write_records_per_s / 1000),
  ceil(outgoing_read_bandwidth_MB_per_s / 2)
)
```

**Ví Dụ Thực Tế:**
```
Yêu cầu:
  - 3,000 records/giây, mỗi record 500 bytes = 1.5 MB/s write
  - 2 consumer, mỗi consumer cần đọc đủ throughput

Write shards needed = max(ceil(1.5/1), ceil(3000/1000)) = max(2, 3) = 3 Shards
Read shards needed = 2 consumer × 1.5 MB/s / 2 MB/s per Shard = 1.5 → 2 Shards

Kết quả: Cần 3 Shards (write bottleneck thắng)
```

**Lưu Ý Quan Trọng:**
- Với Enhanced Fan-Out (Quạt Ra Nâng Cao) — mỗi consumer có dedicated 2 MB/s per Shard → không chia sẻ bandwidth đọc
- Có thể Shard Splitting (Tách Shard) và Merging (Gộp Shard) để scale lên/xuống
- On-Demand mode (Chế Độ Theo Yêu Cầu) tự động scale, giới hạn 200 Shards ban đầu

**Điểm Cần Nhấn Mạnh:**
> "Hot Shard xảy ra khi Partition Key distribution (Phân Phối Khóa Phân Vùng) không đều — ví dụ dùng user ID tập trung vào VIP users. Giải pháp: thêm random suffix vào Partition Key để phân tán."

---

### Câu 3: Kinesis vs MSK (Apache Kafka) — Trade-offs và tiêu chí lựa chọn?

**Tại sao hay hỏi:** Đây là quyết định kiến trúc quan trọng — không có câu trả lời đúng tuyệt đối, nhà tuyển dụng muốn thấy bạn phân tích.

#### Gợi Ý Trả Lời

**Tổng Quan:**
- **MSK** — Amazon Managed Streaming for Apache Kafka (Kafka Được Quản Lý) — là fully managed Kafka
- **Kinesis** — dịch vụ streaming native AWS với API riêng

**So Sánh Chi Tiết:**

| Tiêu Chí | Kinesis Data Streams | Amazon MSK (Kafka) |
| --------- | -------------------- | ------------------- |
| **Ecosystem** | AWS-native only | Kafka ecosystem rộng (Kafka Connect, Kafka Streams, Flink) |
| **Consumer Groups** | Hạn chế | Không giới hạn consumer groups |
| **Retention** | Tối đa 365 ngày | Không giới hạn (tuỳ disk) |
| **Message Size** | Tối đa 1 MB | Tối đa 1 MB (default), cấu hình lên được |
| **Ordering** | Per-Shard ordering | Per-Partition ordering |
| **Scaling** | Thêm/bớt Shard theo API | Thêm Broker — phức tạp hơn |
| **Serverless** | ✅ On-demand mode | ✅ MSK Serverless |
| **Cost Model** | Shard-hour | EC2 instance cho Brokers |
| **Ops Overhead** | Thấp | Trung bình (MSK quản lý Zookeeper/KRaft) |
| **Multi-region** | Phức tạp | Kafka MirrorMaker 2 hỗ trợ tốt |

**Khi Chọn Kinesis:**
- Toàn bộ stack là AWS native
- Team không có Kafka expertise
- Cần tích hợp nhanh với Lambda, Firehose, KDA
- Workload tương đối đơn giản, ít consumer types

**Khi Chọn MSK:**
- Đã có Kafka on-premises muốn migrate
- Cần Kafka Connect (Connectors) với nhiều data source/sink
- Cần consumer groups độc lập với replay capabilities tùy ý
- High-throughput workload > 1 GB/s với nhiều topics phức tạp
- Cần Kafka Streams hoặc ksqlDB cho stream processing

**Điểm Cần Nhấn Mạnh:**
> "Trong môi trường multi-cloud hoặc hybrid, MSK có lợi thế vì Kafka là open standard. Nếu 100% AWS và cần đơn giản hóa operations, Kinesis thường là lựa chọn tốt hơn."

---

### Câu 4: Làm thế nào để đảm bảo Exactly-once Processing trong streaming pipeline?

**Tại sao hay hỏi:** Đây là thách thức cốt lõi của distributed streaming — thể hiện bạn hiểu distributed systems.

#### Gợi Ý Trả Lời

**Ba Delivery Semantics (Ngữ Nghĩa Giao Nhận):**

```
At-most-once (Nhiều Nhất Một Lần):
  → Data có thể bị mất, không bao giờ duplicate
  → Dùng cho: metrics telemetry, logs không critical

At-least-once (Ít Nhất Một Lần):
  → Không mất data, nhưng có thể duplicate
  → Cần idempotent consumer (Người Tiêu Thụ Bất Biến)

Exactly-once (Chính Xác Một Lần):
  → Không mất, không duplicate — khó nhất
  → Chi phí cao hơn về latency và complexity
```

**Cách Đạt Exactly-once Với Kinesis:**

1. **Idempotent Consumer (Người Tiêu Thụ Bất Biến):**
   - Dùng sequence number của record làm idempotency key
   - Lưu sequence number đã xử lý vào DynamoDB
   - Trước khi xử lý, check xem sequence number đã có chưa

2. **Kinesis Data Analytics — KDA với Apache Flink:**
   - Flink có built-in exactly-once semantics với checkpointing
   - Dùng Flink's two-phase commit (Cam Kết Hai Pha) với Kinesis source

3. **Dead Letter Queue — DLQ (Hàng Đợi Thư Chết):**
   - Khi processing fail, ghi vào DLQ thay vì retry vô hạn
   - Consumer đọc DLQ để retry có kiểm soát

**Với Kafka/MSK:**
- Kafka Transactions (Giao Dịch Kafka) với `transactional.id`
- Idempotent producer với `enable.idempotence=true`
- Exactly-once across topics với transaction coordinator

**Điểm Cần Nhấn Mạnh:**
> "Trong thực tế, at-least-once + idempotent consumer thường đủ và đơn giản hơn. Exactly-once thực sự chỉ cần thiết khi downstream không thể handle duplicates — ví dụ financial transactions."

---

### Câu 5: Thiết kế hệ thống xử lý 1 triệu events/giây với latency < 1 giây?

**Tại sao hay hỏi:** Câu hỏi system design mở — kiểm tra tư duy kiến trúc end-to-end.

#### Gợi Ý Trả Lời

**Bước 1: Hỏi rõ requirements**
```
- Events là gì? Size trung bình bao nhiêu?
- Latency end-to-end hay chỉ processing latency?
- Destination: real-time dashboard hay long-term storage?
- Cần exactly-once hay at-least-once?
- Budget có giới hạn không?
```

**Bước 2: Phác thảo kiến trúc**

```
Producers (nhiều nguồn)
    ↓
[Kinesis Data Streams]
  → 1M records/s × 500B = 500 MB/s
  → Cần: ceil(500/1) = 500 Shards
  → Hoặc dùng On-Demand mode
    ↓
[Kinesis Data Analytics — Apache Flink]
  → Stream processing: filter, aggregate, enrich
  → Windowing: 5-giây tumbling window
    ↓ (nhiều sink song song)
    ├── [ElastiCache Redis] → Real-time dashboard (< 10ms)
    ├── [DynamoDB] → Per-user state (< 10ms)
    └── [Kinesis Firehose → S3] → Long-term storage (batch)
```

**Bước 3: Giải thích từng thành phần**

- **Kinesis Shards:** 1M records/s ÷ 1,000 records/s per Shard = 1,000 Shards
  - Thực tế: dùng aggregation để giảm số records → fewer Shards
  - KPL (Kinesis Producer Library) hỗ trợ aggregation tự động

- **Flink Processing:** Stateful stream processing với exactly-once checkpoint
  - Flink auto-scales với KDA managed service

- **Dual Output Pattern:** Real-time path (Redis/DynamoDB) + Cold path (S3)

**Chi Phí Ước Tính Nhanh:**
```
1,000 Shards × $0.015/Shard-hour × 24h = $360/ngày
→ Cần optimize: KPL aggregation giảm xuống ~100 Shards = $36/ngày
```

**Điểm Cần Nhấn Mạnh:**
> "Bottleneck thường không phải ở streaming layer mà ở downstream — Redis capacity, DynamoDB write throughput. Cần capacity plan toàn bộ chain."

---

## Nhóm 2: ETL & Data Catalog

---

### Câu 6: AWS Glue Data Catalog là gì và tại sao cần nó?

**Tại sao hay hỏi:** Glue Catalog là trung tâm metadata của AWS analytics ecosystem — hiểu nó = hiểu cách các dịch vụ liên kết với nhau.

#### Gợi Ý Trả Lời

**Glue Data Catalog (Danh Mục Dữ Liệu Glue) là gì:**
- Centralized metadata repository (Kho Lưu Trữ Metadata Tập Trung) — biết data đang ở đâu, có schema gì
- Lưu trữ: databases, tables, partitions, schemas, data types, locations
- Compatible với Hive Metastore (Kho Metadata Hive) — các tool dùng Hive đều tích hợp được

**Tại Sao Cần:**
```
Trước khi có Catalog:
  - Athena phải biết S3 path và schema của bạn = dễ sai, khó maintain
  - Mỗi team tự track schema riêng = data silos (Cô Lập Dữ Liệu)
  - Khi schema thay đổi, phải update code ở nhiều nơi

Với Glue Catalog:
  - Một nơi định nghĩa schema — Athena, Redshift Spectrum, EMR, Glue Jobs đều đọc từ đây
  - Glue Crawlers tự động discover và update schema
  - Schema versioning — biết schema thay đổi khi nào, thay đổi gì
```

**Các Thành Phần:**

```
Glue Catalog
├── Database (logical grouping of tables)
│   ├── Table (metadata về một dataset)
│   │   ├── Schema (column names, types)
│   │   ├── Location (S3 path hoặc JDBC connection)
│   │   ├── Partition keys (Khóa Phân Vùng)
│   │   └── SerDe (Serializer/Deserializer — cách đọc format)
│   └── ...
└── Connections (thông tin kết nối đến JDBC, MongoDB, v.v.)
```

**Tích Hợp Với Các Dịch Vụ:**
- **Athena:** Đọc schema từ Catalog để query → không cần khai báo schema thủ công
- **Redshift Spectrum:** Mount external tables từ Catalog → query S3 data
- **EMR / Spark:** SparkSQL sử dụng Catalog như Hive Metastore
- **Glue ETL Jobs:** Đọc và ghi metadata khi transform data

**Điểm Cần Nhấn Mạnh:**
> "Glue Catalog là 'single source of truth' cho metadata. Khi Crawler discover partition mới trong S3, Athena tự biết → không cần MSCK REPAIR TABLE thủ công nếu cấu hình đúng."

---

### Câu 7: Glue ETL Job vs Amazon EMR — Khi nào dùng cái nào?

**Tại sao hay hỏi:** Cả hai đều dùng Spark — sự khác biệt nằm ở mức độ control và complexity.

#### Gợi Ý Trả Lời

**So Sánh Nhanh:**

| Tiêu Chí | AWS Glue ETL | Amazon EMR |
| --------- | ------------ | ---------- |
| **Infrastructure** | Serverless — không quản lý cluster | Bạn quản lý cluster (Master/Core/Task nodes) |
| **Spark Version** | AWS-managed, phiên bản cố định | Toàn quyền chọn version |
| **Custom Libraries** | Hạn chế (chỉ Python packages từ S3) | Cài bất kỳ dependency nào |
| **Scaling** | Auto (theo DPU — Data Processing Unit) | Manual hoặc Auto Scaling |
| **Startup Time** | 2-10 phút | 5-15 phút |
| **Cost Model** | DPU-hour (tính đến phần nhỏ) | EC2 instance-hour (tính cả khi idle) |
| **Max Job Duration** | 48 giờ | Không giới hạn |
| **Monitoring** | Glue Console + CloudWatch | Spark History Server, CloudWatch |
| **Tối ưu** | Hạn chế (ít knobs) | Toàn quyền tuning |

**Khi Chọn Glue:**
- ETL pipeline tiêu chuẩn: transform, join, convert format
- Team không có Spark expertise sâu
- Muốn serverless, không muốn quản lý cluster
- Tích hợp chặt với Glue Catalog và Glue Workflows

**Khi Chọn EMR:**
- Cần custom Spark configurations (memory settings, executor tuning)
- Job chạy lâu (>48h) hoặc liên tục (long-running cluster)
- Cần Hive, Presto, HBase cùng với Spark
- ML pipeline với PySpark MLlib cần dependencies phức tạp
- Chi phí tối ưu hóa cao với Spot Instances + Instance Fleets

**EMR Serverless — Trung Dung:**
- Không quản lý cluster nhưng vẫn chạy Spark/Hive
- Tốt hơn Glue khi cần fine-grained Spark config mà không muốn manage cluster

**Điểm Cần Nhấn Mạnh:**
> "Quy tắc thực tế: dùng Glue khi bạn muốn xong việc nhanh; dùng EMR khi bạn cần control để đạt hiệu suất cao hoặc chi phí thấp nhất."

---

### Câu 8: Giải thích Glue DPU (Data Processing Unit) và cách tối ưu chi phí?

**Tại sao hay hỏi:** DPU là đơn vị tính phí của Glue — không hiểu dẫn đến bill shock.

#### Gợi Ý Trả Lời

**DPU là gì:**
- Data Processing Unit (Đơn Vị Xử Lý Dữ Liệu) = 4 vCPUs + 16 GB RAM
- Giá: ~$0.44 per DPU-hour (Standard Worker)
- Minimum: 2 DPUs per job, tính phí tối thiểu 10 phút

**Các Loại Worker:**

| Worker Type | vCPU | RAM | Disk | Phù Hợp |
| ----------- | ---- | --- | ---- | -------- |
| Standard | 4 | 16 GB | 50 GB | General ETL |
| G.1X | 4 | 16 GB | 64 GB | Memory-optimized |
| G.2X | 8 | 32 GB | 128 GB | Large shuffles, joins |
| G.4X | 16 | 64 GB | 256 GB | Very large datasets |
| G.8X | 32 | 128 GB | 512 GB | Extreme scale |
| Z.2X | 8 | 64 GB | 128 GB | ML transforms |

**Kỹ Thuật Tối Ưu Chi Phí:**

1. **Job Bookmarks (Dấu Trang Job):**
   - Chỉ xử lý data mới từ lần chạy trước → giảm volume xử lý
   - Bật: `job.init()` với `'--job-bookmark-option': 'job-bookmark-enable'`

2. **Pushdown Predicates (Đẩy Điều Kiện Xuống):**
   ```python
   datasource = glueContext.create_dynamic_frame.from_catalog(
       database="my_db", table_name="my_table",
       push_down_predicate="(year='2024' and month='01')"  # Chỉ đọc partition cần thiết
   )
   ```

3. **Đúng Worker Type:**
   - G.1X thay vì G.2X nếu không cần RAM nhiều — giảm 50% chi phí
   - Tăng DPU count thay vì dùng G.2X khi cần song song

4. **Glue Studio Visual ETL:**
   - Ước tính DPU usage trước khi chạy

5. **Python Shell Jobs:**
   - Tác vụ không cần Spark (API calls, metadata operations) → 0.0625 DPU — rẻ hơn rất nhiều

6. **Flex Execution (Thực Thi Linh Hoạt):**
   - Dùng spare capacity (năng lực dự phòng) → giảm 34% chi phí nhưng có thể bị delay

**Điểm Cần Nhấn Mạnh:**
> "Job Bookmark là tính năng tiết kiệm chi phí lớn nhất — tránh reprocessing toàn bộ S3 mỗi lần chạy. Kết hợp với partition pruning (Cắt Tỉa Phân Vùng) bằng pushdown predicate."

---

### Câu 9: Glue Crawler (Trình Thu Thập) hoạt động thế nào? Có nên dùng Crawler cho production không?

**Tại sao hay hỏi:** Crawler tiện nhưng có trade-offs quan trọng trong production.

#### Gợi Ý Trả Lời

**Crawler Hoạt Động Thế Nào:**
```
1. Crawler scan S3 path (hoặc JDBC source, DynamoDB, v.v.)
2. Sample một số files để infer schema (suy luận schema)
3. Phát hiện file format: Parquet, CSV, JSON, ORC, Avro
4. Phát hiện partition structure từ folder hierarchy
5. Tạo/update table metadata trong Glue Catalog
```

**Ưu Điểm:**
- Tự động phát hiện schema mới — không cần code thủ công
- Phát hiện partition mới khi data được thêm vào S3
- Schedule chạy định kỳ (hourly, daily)

**Nhược Điểm Trong Production:**

1. **Schema Merge Conflict (Xung Đột Gộp Schema):**
   - Nếu files có schema khác nhau, Crawler tự động merge — có thể sai
   - Ví dụ: file cũ có 10 cột, file mới có 12 cột → Crawler tạo table với 12 cột, nhưng query cột mới trên data cũ sẽ trả về null

2. **Chậm:** Crawler phải scan S3 → tốn thời gian với data lớn

3. **Không Deterministic (Không Xác Định):** Schema inferred có thể thay đổi nếu sample files khác nhau

**Best Practice Cho Production:**

```python
# Option 1: Crawler + Schema locking
# → Chạy Crawler lần đầu để tạo table
# → Sau đó tắt Crawler, quản lý schema manually

# Option 2: Manual table definition trong Glue Catalog
# → Đảm bảo schema chính xác 100%
# → Dùng Glue API hoặc CloudFormation/Terraform

# Option 3: Crawler với custom classifier
# → Viết custom classifier để Crawler infer schema theo quy tắc bạn định
```

**Điểm Cần Nhấn Mạnh:**
> "Crawler tuyệt vời cho development và discovery. Trong production, nên dùng Infrastructure as Code (Terraform hoặc CloudFormation) để định nghĩa Glue tables một cách explicit và version-controlled."

---

## Nhóm 3: Query & Data Warehouse

---

### Câu 10: Làm thế nào để tối ưu Athena query để giảm chi phí?

**Tại sao hay hỏi:** Athena tính phí theo dữ liệu quét (scan) — tối ưu = tiết kiệm tiền trực tiếp.

#### Gợi Ý Trả Lời

**Athena Tính Phí Thế Nào:**
- $5 per TB dữ liệu quét
- Tối thiểu 10 MB per query
- Không tính phí nếu query thất bại hoặc bị cancel

**Kỹ Thuật Tối Ưu (Theo Mức Độ Ảnh Hưởng):**

**1. Columnar Format — Parquet hoặc ORC (Ảnh Hưởng Lớn Nhất)**
```sql
-- Thay vì CSV (quét toàn bộ file)
-- Dùng Parquet (chỉ đọc columns cần thiết)

-- Trước: 100 columns × 1 TB = 1 TB scan = $5
-- Sau (Parquet, SELECT 5 columns): ~50 GB scan = $0.25
-- Tiết kiệm: 95%
```

**2. Partitioning (Phân Vùng) — Filter Giảm Data Quét**
```sql
-- S3 structure: s3://bucket/logs/year=2024/month=01/day=15/

-- Query KHÔNG optimize (full scan):
SELECT * FROM logs WHERE created_at >= '2024-01-15'
-- → Quét toàn bộ table = tốn tiền

-- Query ĐÚNG (sử dụng partition columns):
SELECT * FROM logs WHERE year='2024' AND month='01' AND day='15'
-- → Chỉ quét 1 ngày = 1/365 chi phí
```

**3. Partition Projection (Chiếu Phân Vùng) — Không Cần MSCK REPAIR**
```sql
-- Cấu hình trong table properties → Athena tự tính partition values
-- Không cần Crawler update khi có partition mới
ALTER TABLE logs SET TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.year.type' = 'integer',
  'projection.year.range' = '2020,2030'
);
```

**4. Compression (Nén Dữ Liệu)**
- Parquet với Snappy: giảm 60-70% kích thước so với CSV raw
- ORC với Zlib: giảm mạnh hơn nhưng CPU nhiều hơn

**5. File Size Optimization (Tối Ưu Kích Thước File)**
- Athena hoạt động tốt nhất với files 128 MB — 1 GB
- Quá nhiều files nhỏ (small file problem — Vấn Đề File Nhỏ) = overhead scan lớn
- Dùng Glue hoặc Spark để compact (gộp) files nhỏ thành lớn

**6. Workgroup Query Result Reuse (Tái Sử Dụng Kết Quả)**
- Bật trong Workgroup settings → query giống nhau trong 7 ngày không quét lại S3

**Điểm Cần Nhấn Mạnh:**
> "Thứ tự ưu tiên: Parquet format → Partitioning → Compression → File size. Chuyển từ CSV sang Parquet + partitioning thường giảm 90%+ chi phí và tăng tốc 10-50x."

---

### Câu 11: Redshift DISTKEY và SORTKEY — Cách chọn phù hợp?

**Tại sao hay hỏi:** Đây là kiến thức phân biệt Data Engineer trung cấp với nâng cao trong Redshift.

#### Gợi Ý Trả Lời

**DISTKEY (Khóa Phân Phối):**
- Quyết định rows được phân phối đến compute node nào
- Mục tiêu: giảm data movement (di chuyển dữ liệu) khi JOIN
- Chọn sai → full table redistributed (phân phối lại toàn bảng) khi JOIN = chậm

```sql
-- Bảng orders có DISTKEY là customer_id
-- Bảng customers có DISTKEY là customer_id
-- → Khi JOIN, matching rows đã ở cùng node → không cần network transfer
CREATE TABLE orders (
    order_id BIGINT,
    customer_id INT DISTKEY,
    amount DECIMAL(10,2)
) DISTSTYLE KEY;
```

**DISTSTYLE Options:**

| DISTSTYLE | Cách Hoạt Động | Dùng Khi |
| --------- | -------------- | --------- |
| `KEY` | Phân phối theo giá trị DISTKEY | Bảng lớn JOIN với bảng lớn cùng key |
| `ALL` | Copy toàn bộ bảng đến mọi node | Bảng nhỏ (< 10 MB) hay JOIN với bảng lớn |
| `EVEN` | Round-robin phân phối đều | Bảng không bao giờ JOIN |
| `AUTO` | Redshift tự chọn (default) | Mặc định, để Redshift quyết định |

**SORTKEY (Khóa Sắp Xếp):**
- Quyết định thứ tự rows được lưu trên disk per node
- Mục tiêu: Zone map (Bản Đồ Vùng) skip blocks không liên quan khi filter

```sql
-- Compound SORTKEY: sắp xếp theo nhiều cột, theo thứ tự
CREATE TABLE events (
    event_date DATE,
    user_id INT,
    event_type VARCHAR(50),
    value FLOAT
)
SORTKEY (event_date, user_id);  -- Compound: filter theo date trước, rồi user_id

-- Interleaved SORTKEY: mọi cột có weight bằng nhau
-- Tốt khi không biết query patterns
-- Nhưng tốn thêm chi phí VACUUM (tái cấu trúc)
INTERLEAVED SORTKEY (event_date, user_id, event_type);
```

**Hướng Dẫn Chọn:**

```
DISTKEY:
  ✅ Chọn cột thường được JOIN (customer_id, product_id, order_id)
  ✅ Chọn cột có high cardinality (số lượng giá trị riêng biệt cao) để phân phối đều
  ❌ Tránh cột có skewed values (nhiều rows cùng value) → hot node

SORTKEY:
  ✅ Chọn cột thường trong WHERE clause (date, timestamp)
  ✅ Compound: khi query patterns nhất quán (date THEN user_id)
  ✅ Interleaved: khi nhiều query patterns khác nhau
  ❌ Interleaved tốn VACUUM thêm → chỉ dùng khi thực sự cần
```

**Điểm Cần Nhấn Mạnh:**
> "Redshift Advisor (Cố Vấn Redshift) trong console có thể suggest DISTKEY và SORTKEY dựa trên query history thực tế — rất hữu ích khi migrate."

---

### Câu 12: Athena vs Redshift — Khi nào dùng cái nào?

**Tại sao hay hỏi:** Cả hai đều query data — nhưng chúng khác nhau hoàn toàn về kiến trúc và use case.

#### Gợi Ý Trả Lời

**So Sánh Tổng Quan:**

| Tiêu Chí | Amazon Athena | Amazon Redshift |
| --------- | ------------- | ---------------- |
| **Kiến Trúc** | Serverless, query S3 trực tiếp | MPP cluster, data stored internally |
| **Data Location** | Data luôn ở S3 | Data loaded vào Redshift nodes |
| **Setup** | Không cần setup (zero config) | Cần provision cluster hoặc Serverless |
| **Latency** | 5 giây — vài phút | Sub-second — vài giây với đúng config |
| **Cost Model** | $5/TB scanned | Cluster-hour (fixed cost) hoặc RPU |
| **Best For** | Ad-hoc query, exploration | OLAP workloads, BI dashboards, repeat queries |
| **Max Data** | Không giới hạn (S3) | Tùy node storage |
| **Concurrent Users** | Tốt | Cần WLM (Workload Management) tuning |
| **Complex Joins** | Chậm (scan lại S3) | Nhanh (data trên local disk, MPP) |

**Khi Chọn Athena:**
- Data scientists, analysts cần ad-hoc exploration
- Query không thường xuyên (< 10 queries/ngày per analyst)
- Data chưa được làm sạch, format khác nhau
- Budget thấp và volume query nhỏ
- Query logs, audit data rất ít khi cần

**Khi Chọn Redshift:**
- BI tools (Tableau, Power BI, QuickSight) kết nối thường xuyên
- Hàng trăm users concurrent query
- Report cần chạy trong vài giây
- Data đã được làm sạch và có schema rõ ràng
- Complex OLAP workloads với nhiều JOINs

**Pattern Kết Hợp Tốt Nhất:**
```
S3 Data Lake (Raw Data)
    ↓ Glue ETL
S3 (Processed Data, Parquet)
    ├── Athena → Ad-hoc query, data science
    └── Redshift COPY hoặc Spectrum → BI dashboard, production reports
```

**Điểm Cần Nhấn Mạnh:**
> "Redshift Spectrum (Phổ Redshift) là middle ground — query S3 directly từ Redshift, không cần COPY data vào. Nhưng vẫn tốn Redshift cluster cost."

---

### Câu 13: Giải thích Redshift Serverless — Khác gì so với Provisioned Redshift?

**Tại sao hay hỏi:** Redshift Serverless là hướng phát triển mới — cần biết khi nào nên migrate.

#### Gợi Ý Trả Lời

**Provisioned Redshift (Redshift Được Cấp Phát):**
- Bạn chọn node type (ra3.xlplus, ra3.4xlarge, v.v.) và số node
- Trả tiền theo giờ kể cả khi không có query
- RA3 nodes: compute và storage tách biệt (storage ở Redshift Managed Storage)
- Cần quản lý: resize, snapshot, maintenance windows

**Redshift Serverless (Redshift Không Máy Chủ):**
- Bạn chỉ cần cấu hình RPU — Redshift Processing Units (Đơn Vị Xử Lý Redshift)
- AWS tự scale compute lên/xuống theo workload
- Chỉ trả tiền khi query đang chạy (per RPU-second)
- Pause tự động khi idle

**So Sánh:**

| | Provisioned | Serverless |
| - | ----------- | ---------- |
| **Billing** | Per node-hour (24/7) | Per RPU-second (chỉ khi active) |
| **Scaling** | Manual resize hoặc Elastic Resize | Tự động |
| **Cold Start** | Không có (luôn sẵn sàng) | Vài giây nếu đã pause |
| **Max RPU** | Cố định theo cluster | Có thể set max RPU limit |
| **Cost (low usage)** | Đắt hơn (pay 24/7) | Rẻ hơn |
| **Cost (high usage)** | Rẻ hơn (fixed rate) | Có thể đắt hơn |
| **WLM** | Manual configuration | Tự động |

**Khi Chọn Serverless:**
- Workload không ổn định, có peaks và troughs
- Dev/test environments (tiết kiệm khi idle)
- New projects chưa biết exact capacity
- Analytical workloads không cần sub-second latency liên tục

**Khi Chọn Provisioned:**
- Workload ổn định, cao đều (>70% utilization)
- Cần Reserved Instance discount (giảm giá cam kết)
- Cần full control over WLM queues
- Latency yêu cầu rất cao, không chấp nhận cold start

**Điểm Cần Nhấn Mạnh:**
> "Rule of thumb: nếu cluster chạy < 30% thời gian trong ngày, Serverless thường rẻ hơn. Nếu chạy > 70%, Provisioned + Reserved Instance rẻ hơn tới 60%."

---

### Câu 14: Giải thích WLM (Workload Management) trong Redshift?

**Tại sao hay hỏi:** WLM là chìa khóa để Redshift phục vụ nhiều loại query cùng lúc mà không bị nghẽn.

#### Gợi Ý Trả Lời

**WLM — Workload Management (Quản Lý Tải Công Việc):**
- Hệ thống phân bổ resources (memory, concurrency) cho các loại query khác nhau
- Mục tiêu: query analytics nặng không block dashboard queries nhẹ

**Automatic WLM (WLM Tự Động):**
- Redshift tự quản lý memory và concurrency
- Tốt cho hầu hết workloads — recommend dùng mặc định

**Manual WLM Queues:**
```sql
-- Ví dụ: 3 queues
Queue 1 — BI Dashboards:
  Priority: Highest
  Concurrency: 15 slots
  Memory: 20% total
  Query timeout: 30 giây
  User groups: dashboard_users

Queue 2 — ETL Jobs:
  Priority: Medium
  Concurrency: 5 slots
  Memory: 60% total
  Query timeout: 3600 giây
  User groups: etl_users

Queue 3 — Ad-hoc Queries:
  Priority: Low (default queue)
  Concurrency: 5 slots
  Memory: 20% total
  Query timeout: 600 giây
```

**Short Query Acceleration — SQA (Tăng Tốc Truy Vấn Ngắn):**
- Tự động detect và prioritize queries dự kiến hoàn thành nhanh
- Bật theo mặc định với Automatic WLM
- Giúp dashboard queries không bị queue sau ETL jobs lâu

**Điểm Cần Nhấn Mạnh:**
> "Concurrency Scaling (Mở Rộng Song Song) cho phép Redshift tự động thêm cluster capacity khi queue đầy — trả phí theo thời gian scale out, tặng 1 giờ free per ngày."

---

## Nhóm 4: Kiến Trúc & Thiết Kế

---

### Câu 15: Lambda Architecture vs Kappa Architecture — Khác nhau thế nào và khi nào chọn cái nào?

**Tại sao hay hỏi:** Đây là câu hỏi kiến trúc kinh điển — thể hiện độ sâu tư duy hệ thống dữ liệu.

#### Gợi Ý Trả Lời

**Lambda Architecture (Kiến Trúc Lambda):**

```
Data Source
    ↓ (split)
    ├── Batch Layer (Lớp Lô):
    │     → Xử lý toàn bộ historical data
    │     → Accuracy cao, latency cao (giờ đến ngày)
    │     → Tool: Spark on EMR, Glue ETL
    │     → Output: Batch Views (Chế Độ Xem Lô)
    │
    └── Speed Layer (Lớp Tốc Độ):
          → Chỉ xử lý data mới nhất
          → Accuracy thấp hơn (approximate), latency thấp (giây đến phút)
          → Tool: Kinesis + Lambda/Flink
          → Output: Real-time Views (Chế Độ Xem Thời Gian Thực)

Serving Layer (Lớp Phục Vụ):
  → Merge Batch Views + Real-time Views để trả kết quả
  → Tool: Redshift, DynamoDB, Redis
```

**Kappa Architecture (Kiến Trúc Kappa):**

```
Data Source
    ↓
Stream Processing Layer (Lớp Xử Lý Luồng) — DUY NHẤT:
  → Xử lý TẤT CẢ data (historical + real-time) qua stream
  → Reprocess historical data bằng cách replay stream từ đầu
  → Tool: Kafka/MSK + Flink hoặc Spark Streaming
    ↓
Serving Layer
```

**So Sánh:**

| Tiêu Chí | Lambda | Kappa |
| --------- | ------- | ----- |
| **Complexity** | Cao (2 code paths) | Thấp hơn (1 code path) |
| **Accuracy** | Cao (batch corrects errors) | Phụ thuộc vào stream processing accuracy |
| **Historical Reprocessing** | Chạy lại batch job | Replay toàn bộ stream từ offset 0 |
| **Operational Cost** | Cao (duy trì 2 systems) | Thấp hơn |
| **Latency** | Mix: batch (cao) + stream (thấp) | Thấp (stream only) |
| **Tools** | Phức tạp | Flink hoặc Spark Streaming |

**Khi Chọn Lambda:**
- Cần absolute accuracy (tài chính, compliance)
- Team đã có batch system, thêm streaming sau
- Business cho phép eventual consistency (Nhất Quán Dần Dần)

**Khi Chọn Kappa:**
- Data đủ đơn giản để xử lý hoàn toàn bằng stream
- Muốn giảm operational complexity
- Kafka/MSK với retention dài → có thể replay lịch sử

**Điểm Cần Nhấn Mạnh:**
> "Xu hướng hiện tại nghiêng về Kappa với Flink vì mature hơn và tránh được vấn đề sync giữa batch và speed layer. Medallion Architecture trên S3 + Iceberg đang thay thế Lambda Architecture trong nhiều use case."

---

### Câu 16: Giải thích Medallion Architecture (Kiến Trúc Huy Chương) — Bronze, Silver, Gold?

**Tại sao hay hỏi:** Đây là pattern data lake phổ biến nhất hiện tại, hay gặp trong cả interview và thực tế.

#### Gợi Ý Trả Lời

**Tổng Quan:**

```
Bronze (Đồng) → Silver (Bạc) → Gold (Vàng)
   Raw Data    → Cleaned Data  → Business-ready Data
```

**Bronze Layer (Tầng Đồng — Raw):**
```
Mục tiêu: Lưu data gốc không chỉnh sửa — source of truth tuyệt đối

Đặc điểm:
  - Data nguyên gốc từ source systems (databases, APIs, Kafka, files)
  - Thêm metadata: ingestion timestamp, source system, batch ID
  - Schema: flat, không normalize
  - Format: Parquet hoặc Delta Lake (để audit trail)
  - Retention: lâu dài (1-7 năm) — dùng S3 Glacier cho cold data

S3 path ví dụ: s3://datalake/bronze/ecommerce/orders/year=2024/month=01/
```

**Silver Layer (Tầng Bạc — Cleaned):**
```
Mục tiêu: Data đã làm sạch, đã validate, đã normalize

Đặc điểm:
  - Remove duplicates (xóa bản ghi trùng lặp)
  - Standardize formats (chuẩn hóa định dạng: dates, phone numbers)
  - Apply business rules validation
  - Join với reference tables (dimension tables)
  - Thêm derived columns (cột được tính toán)
  - Conform to common data model (mô hình dữ liệu chung)

Ai dùng: Data engineers, data scientists cần data sạch
Format: Parquet hoặc Delta Lake với schema enforcement
```

**Gold Layer (Tầng Vàng — Business-ready):**
```
Mục tiêu: Data đã aggregate, ready cho BI và reporting

Đặc điểm:
  - Pre-aggregated metrics (số liệu đã tổng hợp sẵn)
  - Denormalized (phi chuẩn hóa) cho performance
  - Business-specific views (Daily Sales, Monthly Active Users)
  - Optimized cho query tools: Athena, Redshift, QuickSight
  - SLA nghiêm ngặt hơn (data freshness requirement)

Ai dùng: Business analysts, executives, BI tools
Ví dụ: daily_revenue_by_product, monthly_user_retention
```

**Pipeline Thực Tế:**

```
Kafka (events) → Kinesis Firehose → Bronze (S3/Parquet)
                                        ↓ Glue ETL Job
                                    Silver (S3/Parquet, cleaned)
                                        ↓ Glue ETL Job
                                    Gold (S3/Parquet, aggregated)
                                        ↓
                              ┌─────────────────────┐
                              │ Athena (ad-hoc)     │
                              │ Redshift Spectrum   │
                              │ QuickSight          │
                              └─────────────────────┘
```

**Điểm Cần Nhấn Mạnh:**
> "Medallion với Apache Iceberg (hoặc Delta Lake) thêm ACID transactions (Giao Dịch ACID) và time travel (Du Hành Thời Gian — query data tại point-in-time bất kỳ) — đây là tương lai của data lakehouse."

---

### Câu 17: Data Mesh là gì? Tại sao cần và triển khai thế nào trên AWS?

**Tại sao hay hỏi:** Data Mesh là paradigm mới đang được nhiều công ty lớn áp dụng — câu hỏi dành cho senior engineer.

#### Gợi Ý Trả Lời

**Vấn Đề Data Mesh Giải Quyết:**
```
Mô hình truyền thống (Centralized Data Lake — Hồ Dữ Liệu Tập Trung):
  - 1 team data engineering quản lý toàn bộ pipelines
  - Mỗi domain (Sales, Marketing, Product) phải xin data qua central team
  - Bottleneck: central team không thể hiểu sâu business của từng domain
  - Chậm: data request mất vài tuần để được serve
  - Data quality: central team không biết data nên trông như thế nào
```

**Data Mesh — 4 Nguyên Tắc Cốt Lõi:**

1. **Domain Ownership (Sở Hữu Theo Domain):**
   - Mỗi domain (Sales, Marketing, Product) tự sở hữu và quản lý data của mình
   - Domain team hiểu business logic → data chính xác hơn

2. **Data as a Product (Dữ Liệu Như Sản Phẩm):**
   - Mỗi domain publish "data products" — có SLA, documentation, quality guarantees
   - Consumers (người dùng) đăng ký sử dụng data products

3. **Self-serve Data Platform (Nền Tảng Tự Phục Vụ):**
   - Platform team cung cấp tools để domain teams dễ dàng build và serve data
   - Không phải AI infrastructure — mà là tooling, templates, governance

4. **Federated Computational Governance (Quản Trị Tính Toán Liên Kết):**
   - Policies áp dụng tự động — không cần trung tâm approve từng request
   - Global standards (bảo mật, format, SLA) nhưng domain autonomy

**Triển Khai Trên AWS:**

```
Self-serve Platform (Platform Team xây):
  └── AWS Lake Formation → Centralized access control policies
  └── AWS Glue Catalog → Central metadata registry, mỗi domain có database riêng
  └── S3 (per-domain buckets) → Domain data products
  └── Redshift data sharing → Cross-domain analytics

Domain A (Sales Team):
  └── KDS/MSK → ingest sales events
  └── Glue ETL → transform to data product format
  └── Catalog: register table vào central catalog với ownership tag
  └── Lake Formation: grant read access cho approved consumers

Domain B (Marketing Team) muốn dùng Sales data:
  └── Request access qua Lake Formation (có thể tự serve nếu public product)
  └── Query sales data product qua Athena hoặc Redshift Spectrum
  └── Không cần xin phép Sales team → đã có policy tự động
```

**Điểm Cần Nhấn Mạnh:**
> "Data Mesh không phải về technology — mà về organizational design (thiết kế tổ chức). Có thể fail nếu thiếu domain ownership culture, dù có tools tốt đến đâu."

---

### Câu 18: Thiết kế Real-time Analytics Pipeline cho ứng dụng e-commerce — nhận đơn hàng, cập nhật tồn kho, báo cáo real-time?

**Tại sao hay hỏi:** System design tổng hợp — kiểm tra khả năng kết hợp nhiều dịch vụ.

#### Gợi Ý Trả Lời

**Requirements Clarification:**
```
- Volume: 10,000 đơn hàng/phút peak
- Latency: Dashboard refresh < 30 giây
- Inventory: cập nhật trong < 5 giây sau khi đặt hàng
- Historical: lưu 2 năm, query được
- Budget: phải hợp lý cho startup
```

**Kiến Trúc Đề Xuất:**

```
[Mobile App / Web] → [API Gateway + Lambda]
                            ↓ (publish events)
                  [Kinesis Data Streams]
                   (3 topics/streams)
                    ↓           ↓           ↓
            [Orders Stream] [Inventory] [Analytics Stream]
                    ↓           ↓           ↓
         [Lambda Consumer]  [Lambda]  [Kinesis Firehose]
                ↓               ↓           ↓
        [DynamoDB: Orders]  [DynamoDB:  [S3: Raw Data]
         (< 10ms reads)      Inventory]      ↓
                                        [Glue ETL Job]
                                             ↓
                                     [S3: Parquet, partitioned]
                                             ↓
                                    [Athena + QuickSight]
                                    (refresh 5 phút)
```

**Real-time Dashboard (< 30 giây):**
```
Kinesis → Kinesis Data Analytics (Flink)
  → Aggregate: orders per minute, revenue per category
  → Output → ElastiCache Redis (Bộ Nhớ Cache Redis)
  → Dashboard app reads Redis → < 1 giây
```

**Inventory Update (< 5 giây):**
```
Order event → Lambda Consumer
  → DynamoDB atomic counter: DECREMENT inventory
  → Publish "low_inventory" event if threshold reached
  → SNS alert → email/Slack notification
```

**Historical Analytics (2 năm):**
```
Kinesis Firehose → S3 (raw JSON)
  ↓ Glue ETL (mỗi giờ)
S3 (Parquet, partitioned by year/month/day)
  ↓
Athena (ad-hoc) + Redshift Spectrum (BI reports)
  ↓
QuickSight dashboards
```

**Điểm Cần Nhấn Mạnh:**
> "Tách biệt real-time path (Kinesis → Flink → Redis) và batch path (Firehose → S3 → Athena) — không cần sacrifice một cái cho cái kia. Redis cho dashboard sub-second, Athena cho analysis."

---

## Nhóm 5: Chi Phí & Vận Hành

---

### Câu 19: Làm thế nào để giảm chi phí AWS Analytics mà không ảnh hưởng performance?

**Tại sao hay hỏi:** Cost optimization là kỹ năng thực chiến quan trọng — data pipeline dễ phát sinh chi phí ngoài kiểm soát.

#### Gợi Ý Trả Lời

**Theo Từng Dịch Vụ:**

**S3 (Nền Tảng Lưu Trữ):**
```
- S3 Intelligent-Tiering: tự động chuyển data sang tier rẻ hơn khi ít truy cập
- Lifecycle Policies: sau 30 ngày → Infrequent Access; sau 90 ngày → Glacier
- Parquet + Snappy compression: giảm 60-80% storage size
- Multipart upload + cleanup: tránh incomplete uploads tích lũy chi phí
```

**Athena:**
```
- Parquet + partitioning: giảm 90%+ data scanned
- Workgroup Query Result Reuse: cache kết quả 7 ngày
- Workgroup budgets: alert khi spend vượt ngưỡng
- Partition Projection: giảm metadata overhead
```

**Glue:**
```
- Job Bookmarks: chỉ process data mới
- Flex Execution: 34% discount với spot-like capacity
- Python Shell cho non-Spark tasks: 0.0625 DPU thay vì 2+ DPU
- Right-size Worker Type: đừng dùng G.2X khi G.1X đủ
```

**EMR:**
```
- Spot Instances cho Task nodes: 60-90% discount
- Instance Fleets: mix On-Demand (Master/Core) + Spot (Task)
- Transient Clusters: tắt sau khi job xong, không để cluster idle
- EMR Serverless: pay per job, không pay khi idle
```

**Kinesis:**
```
- On-Demand mode: không over-provision Shards
- KPL Aggregation: gộp nhiều records nhỏ → giảm Shard count cần thiết
- Enhanced Fan-Out chỉ khi thực sự cần (thêm $0.015/shard-hour)
```

**Redshift:**
```
- Reserved Instances: 1-3 năm → tiết kiệm 40-75%
- Redshift Serverless cho workload không ổn định
- Auto Pause khi idle (Serverless)
- Spectrum thay vì COPY toàn bộ historical data vào Redshift
- WLM tuning: tránh long-running queries chiếm hết memory
```

**Chiến Lược Tổng Thể:**
```
1. Tag mọi resource với cost center → biết chi phí theo team/project
2. Dùng AWS Cost Explorer + Cost Anomaly Detection
3. Set budget alerts cho từng dịch vụ
4. Review Trusted Advisor recommendations hàng tuần
5. Optimize theo thứ tự ảnh hưởng: Storage → Compute → Networking
```

**Điểm Cần Nhấn Mạnh:**
> "Storage optimization (Parquet + compression) thường có ROI cao nhất và dễ implement nhất. Chuyển toàn bộ data sang Parquet thường giảm 70%+ S3 cost và tăng tốc Athena lên 10-50x."

---

### Câu 20: Làm thế nào để xử lý sự cố phổ biến trong analytics pipeline? (Kinesis throttling, Glue job failure, Athena slow query)

**Tại sao hay hỏi:** Vận hành thực tế — không chỉ biết build mà phải biết debug.

#### Gợi Ý Trả Lời

**Kinesis Throttling (Hạn Chế Tốc Độ Kinesis):**

```
Triệu Chứng:
  - ProvisionedThroughputExceededException errors
  - Consumer lag tăng cao
  - Records bị drop hoặc retry

Nguyên Nhân + Giải Pháp:

1. Hot Shard (Mảnh Nóng) — quá nhiều records vào cùng 1 Shard:
   → Diagnosis: CloudWatch metric "IncomingBytes per Shard"
   → Fix: Thêm random suffix vào Partition Key để phân tán tốt hơn
   → Ví dụ: user_id + "_" + random(0,9) → 10x phân tán

2. Not Enough Shards (Không Đủ Mảnh):
   → Diagnosis: IncomingBytes/Records consistently > 80% limit
   → Fix: UpdateShardCount API → tăng số Shard
   → Hoặc: Chuyển sang On-Demand mode

3. Consumer Reading Too Slow:
   → Diagnosis: GetRecords.IteratorAgeMilliseconds tăng
   → Fix: Thêm consumer instances, dùng Enhanced Fan-Out
```

**Glue Job Failure (Lỗi Glue Job):**

```
Nguyên Nhân Phổ Biến:

1. OOM (Out Of Memory — Hết Bộ Nhớ):
   → CloudWatch Logs: java.lang.OutOfMemoryError
   → Fix: Tăng Worker Type (G.1X → G.2X) hoặc tăng số Workers
   → Hoặc: Repartition data trước khi join: df.repartition(200)

2. Job Timeout:
   → Default timeout: 2880 phút (48 giờ)
   → Fix: Kiểm tra không có infinite loop, tối ưu Spark shuffle
   → Thêm partition pruning để giảm data volume

3. Schema Mismatch:
   → Glue DynamicFrame vs Spark DataFrame type conflicts
   → Fix: Dùng ResolveChoice để handle mixed types:
     resolvechoice = ResolveChoice.apply(
         frame=datasource,
         choice="cast:double",
         specs=[("price", "cast:double")]
     )

4. S3 Rate Limit:
   → Fix: Add retry logic, random exponential backoff
   → Distribute writes across multiple prefixes
```

**Athena Slow Query (Truy Vấn Athena Chậm):**

```
Diagnosis Steps:

1. Check query execution plan:
   EXPLAIN SELECT ...
   → Tìm Full Table Scan (Quét Toàn Bộ Bảng)

2. Check data stats:
   SHOW TBLPROPERTIES "my_table";
   → Partitions, file count, total size

Nguyên Nhân + Giải Pháp:

1. No Partitioning — Full Table Scan:
   → Fix: Add partition columns, MSCK REPAIR TABLE
   → Long-term: Redesign S3 folder structure

2. Small Files Problem (Vấn Đề File Nhỏ):
   → Triệu chứng: Query nhanh plan nhưng chậm thực thi
   → Fix: Compact files với Glue Job:
     df.coalesce(1).write.parquet("s3://...")  # Cẩn thận với file quá lớn
     # Hoặc dùng số files vừa phải: 128MB-1GB per file

3. Wrong Format — CSV thay vì Parquet:
   → Fix: Convert sang Parquet với Glue ETL job

4. Complex Cross-Join hoặc Cartesian Product:
   → Fix: Rewrite query, thêm explicit JOIN conditions
   → Athena không optimize cross-joins tốt

5. Missing Statistics:
   → Fix: ANALYZE TABLE để update statistics
```

**Monitoring Chủ Động:**

```
Kinesis: Alert khi IteratorAgeMilliseconds > 60,000 (1 phút lag)
Glue: Alert khi JobRunState = FAILED
Athena: Alert khi QueryExecutionTime > ngưỡng (ví dụ 300 giây)
Redshift: Alert khi QueuedQueries > 10, CPUUtilization > 90%
```

**Điểm Cần Nhấn Mạnh:**
> "Monitoring và alerting là 50% của vận hành tốt. Đừng đợi user báo cáo mới biết pipeline có vấn đề — CloudWatch Alarms + SNS nên được thiết lập ngay từ đầu."

---

## 📊 Bảng Tóm Tắt Nhanh

### So Sánh Các Dịch Vụ Streaming

| | KDS | KDF | MSK | KDA/Flink |
| - | --- | --- | --- | --------- |
| **Dùng Cho** | Custom consumer | Delivery to storage | Kafka ecosystem | Stream analytics |
| **Latency** | Milliseconds | 60s-15m | Milliseconds | Seconds |
| **Managed** | Partially | Fully | Partially | Fully |
| **Complexity** | Medium | Low | High | Medium |

### So Sánh Các Dịch Vụ Query/Storage

| | Athena | Redshift | EMR/Spark | Glue ETL |
| - | ------ | -------- | --------- | --------- |
| **Dùng Cho** | Ad-hoc query | OLAP/BI | Big data ETL | Serverless ETL |
| **Cost Model** | Per TB scan | Per node/hour | Per instance/hour | Per DPU/hour |
| **Setup** | Zero config | Provision cluster | Provision cluster | Serverless |
| **Best For** | Exploration | Production BI | Complex transforms | Standard ETL |

---

**Cập Nhật Lần Cuối:** 2026-05-18
**Module:** 12 — Interview Prep
**Trạng Thái:** ✅ Hoàn thành — 20 câu hỏi
