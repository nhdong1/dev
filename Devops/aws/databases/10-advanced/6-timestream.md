# 6. Amazon Timestream — Cơ Sở Dữ Liệu Chuỗi Thời Gian

> Amazon Timestream là dịch vụ **time-series database** (cơ sở dữ liệu chuỗi thời gian) được quản lý hoàn toàn, được tối ưu để lưu trữ và phân tích dữ liệu theo thời gian — ví dụ: IoT sensor data (dữ liệu cảm biến IoT), application metrics (số liệu ứng dụng), operational telemetry (đo từ xa vận hành).

## 📚 Mục Lục

1. [Time-Series Data là Gì?](#time-series-data-là-gì)
2. [Kiến Trúc Timestream](#kiến-trúc-timestream)
3. [Data Model — Mô Hình Dữ Liệu](#data-model--mô-hình-dữ-liệu)
4. [Ingest Data — Nhập Dữ Liệu](#ingest-data--nhập-dữ-liệu)
5. [Query với SQL-Like Syntax](#query-với-sql-like-syntax)
6. [Scheduled Queries — Truy Vấn Theo Lịch](#scheduled-queries--truy-vấn-theo-lịch)
7. [Storage Tiers — Phân Tầng Lưu Trữ](#storage-tiers--phân-tầng-lưu-trữ)
8. [Use Cases Thực Tế](#use-cases-thực-tế)
9. [Tích Hợp Với AWS Services](#tích-hợp-với-aws-services)
10. [So Sánh Timestream vs Các Lựa Chọn Khác](#so-sánh-timestream-vs-các-lựa-chọn-khác)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Time-Series Data là Gì?

### Đặc Điểm Của Time-Series Data

Time-series data (dữ liệu chuỗi thời gian) là dữ liệu được gắn với **timestamp** (dấu thời gian) và thường:

```
Đặc điểm:
✅ Append-only (chỉ thêm mới, không sửa) — sensor ghi liên tục
✅ Ordered by time (sắp xếp theo thời gian)
✅ High write throughput (lượng ghi cao) — hàng nghìn events/giây
✅ Mostly recent reads (đọc chủ yếu dữ liệu mới)
✅ Aggregation over time (tổng hợp theo thời gian) — avg/max/min theo giờ/ngày

Ví dụ dữ liệu:
  time=10:00:01, sensor=sensor-001, temperature=25.3, humidity=60
  time=10:00:02, sensor=sensor-001, temperature=25.4, humidity=61
  time=10:00:02, sensor=sensor-002, temperature=22.1, humidity=55
  time=10:00:03, sensor=sensor-001, temperature=25.3, humidity=60
  ...
```

### Tại Sao Cần Database Chuyên Biệt?

| Thách Thức | Relational DB (RDS) | Timestream |
|------------|---------------------|------------|
| **Write throughput** | Bị giới hạn bởi WAL (Write-Ahead Log) | Tối ưu cho millions writes/giây |
| **Storage cost** | Giá cao, không tự phân tầng | Tự động chuyển hot→cold→S3 |
| **Time-based queries** | Cần nhiều index, chậm | Native time functions, siêu nhanh |
| **Data retention** | Tự quản lý việc xóa old data | Tự động theo retention policy |
| **Compression** | Standard row compression | Đặc biệt tối ưu cho time-series |

---

## Kiến Trúc Timestream

### Hai Tầng Lưu Trữ

```
┌─────────────────────────────────────────────────────────────────┐
│                      TIMESTREAM ARCHITECTURE                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              MEMORY STORE (Bộ Nhớ Nhanh)               │   │
│  │  - SSD-backed, in-memory processing                     │   │
│  │  - Dữ liệu mới nhất (configurable: vài giờ đến vài ngày)│   │
│  │  - Chi phí cao hơn, query cực nhanh                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │ Auto-tiering (Phân Tầng Tự Động)     │
│                          ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           MAGNETIC STORE (Lưu Trữ Từ Tính)              │   │
│  │  - SSD-based cold storage                               │   │
│  │  - Dữ liệu cũ hơn (vài tháng đến nhiều năm)            │   │
│  │  - Chi phí thấp hơn nhiều, query hơi chậm hơn          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Auto-scaling, serverless — không cần provision capacity        │
└─────────────────────────────────────────────────────────────────┘
```

### Serverless Model

Timestream là **serverless** — không cần chọn instance type, không cần provision:
- Tự động scale write/read capacity
- Pay-per-use (trả theo lượng dùng): theo writes, queries, storage
- Không có idle cost (phí khi không dùng) nếu không có data

---

## Data Model — Mô Hình Dữ Liệu

### Các Thành Phần

| Thành Phần | Tương Đương SQL | Mô Tả |
|------------|-----------------|-------|
| **Database** | Database | Container cho tables |
| **Table** | Table | Lưu time-series data |
| **Dimension** | Tag/Label | Metadata không đổi — sensor_id, region |
| **Measure** | Column | Giá trị thay đổi — temperature, pressure |
| **Timestamp** | Timestamp column | Thời gian của data point |

### Cấu Trúc Record (Bản Ghi)

```
Record = {
    dimensions: {           ← Metadata (ít thay đổi, dùng để filter)
        sensor_id: "sensor-001",
        region:    "us-east-1",
        device:    "raspberry-pi-4"
    },
    measure_name:  "temperature",     ← Tên metric
    measure_value: 25.3,              ← Giá trị metric
    time:          "2024-01-15T10:00:01Z"
}
```

### Multi-Measure Records (Bản Ghi Đa Metric) — Khuyên Dùng

```python
# Ghi nhiều metrics trong 1 record (hiệu quả hơn)
record = {
    "Dimensions": [
        {"Name": "sensor_id", "Value": "sensor-001"},
        {"Name": "region",    "Value": "us-east-1"}
    ],
    "MeasureName": "environment_metrics",
    "MeasureValueType": "MULTI",
    "MeasureValues": [
        {"Name": "temperature", "Value": "25.3",  "Type": "DOUBLE"},
        {"Name": "humidity",    "Value": "60",    "Type": "BIGINT"},
        {"Name": "pressure",    "Value": "1013.25", "Type": "DOUBLE"}
    ],
    "Time": "1705312801000",  # Unix milliseconds
    "TimeUnit": "MILLISECONDS"
}
```

---

## Ingest Data — Nhập Dữ Liệu

### Python SDK

```python
import boto3
from datetime import datetime

timestream = boto3.client('timestream-write', region_name='us-east-1')

def write_sensor_data(sensor_id: str, temperature: float, humidity: int):
    current_time = str(int(datetime.now().timestamp() * 1000))
    
    records = [{
        'Dimensions': [
            {'Name': 'sensor_id', 'Value': sensor_id},
            {'Name': 'region',    'Value': 'us-east-1'}
        ],
        'MeasureName': 'environment',
        'MeasureValueType': 'MULTI',
        'MeasureValues': [
            {'Name': 'temperature', 'Value': str(temperature), 'Type': 'DOUBLE'},
            {'Name': 'humidity',    'Value': str(humidity),    'Type': 'BIGINT'}
        ],
        'Time': current_time,
        'TimeUnit': 'MILLISECONDS'
    }]
    
    try:
        response = timestream.write_records(
            DatabaseName='iot-database',
            TableName='sensor-data',
            Records=records
        )
        return response
    except timestream.exceptions.RejectedRecordsException as e:
        # Xử lý records bị reject (timestamp ngoài range, etc.)
        print(f"Rejected records: {e.response['RejectedRecords']}")

# Batch write (ghi theo lô) — tối đa 100 records mỗi lần
def write_batch(records_list):
    timestream.write_records(
        DatabaseName='iot-database',
        TableName='sensor-data',
        Records=records_list  # Tối đa 100 records
    )
```

### Late Arrival Data (Dữ Liệu Đến Muộn)

Timestream cho phép ghi dữ liệu có timestamp trong quá khứ (late arrival):

```python
# Cấu hình khi tạo table — cho phép dữ liệu đến muộn tối đa 1 ngày
timestream.create_table(
    DatabaseName='iot-database',
    TableName='sensor-data',
    MagneticStoreWriteProperties={
        'EnableMagneticStoreWrites': True,  # Ghi vào magnetic store
        'MagneticStoreRejectedDataLocation': {
            'S3Configuration': {
                'BucketName': 'my-rejected-data-bucket',
                'EncryptionOption': 'SSE_S3'
            }
        }
    }
)
```

---

## Query với SQL-Like Syntax

Timestream hỗ trợ **SQL-like query language** với các hàm time-series chuyên biệt:

### Query Cơ Bản

```sql
-- Đọc dữ liệu 1 giờ gần nhất
SELECT time, sensor_id, temperature, humidity
FROM "iot-database"."sensor-data"
WHERE time BETWEEN ago(1h) AND now()
ORDER BY time DESC
LIMIT 100
```

### Time-Series Functions (Hàm Chuỗi Thời Gian)

```sql
-- Tính trung bình nhiệt độ theo 5 phút (time binning)
SELECT 
    sensor_id,
    bin(time, 5m)        AS time_window,    -- Nhóm theo 5 phút
    AVG(temperature)     AS avg_temp,
    MAX(temperature)     AS max_temp,
    MIN(temperature)     AS min_temp
FROM "iot-database"."sensor-data"
WHERE time BETWEEN ago(1h) AND now()
GROUP BY sensor_id, bin(time, 5m)
ORDER BY sensor_id, time_window
```

### Interpolate — Nội Suy Dữ Liệu Thiếu

```sql
-- Nội suy các điểm dữ liệu thiếu (sensor bị ngắt kết nối)
SELECT 
    time,
    sensor_id,
    INTERPOLATE_LINEAR(
        CREATE_TIME_SERIES(time, temperature),
        SEQUENCE(min(time), max(time), 1m)  -- Khoảng cách 1 phút
    ) AS interpolated_temp
FROM "iot-database"."sensor-data"
WHERE sensor_id = 'sensor-001'
  AND time BETWEEN ago(6h) AND now()
GROUP BY sensor_id
```

### Anomaly Detection (Phát Hiện Bất Thường)

```sql
-- Tìm các điểm nhiệt độ bất thường (> 2 standard deviations từ mean)
WITH stats AS (
    SELECT 
        sensor_id,
        AVG(temperature) AS mean_temp,
        STDDEV(temperature) AS std_temp
    FROM "iot-database"."sensor-data"
    WHERE time BETWEEN ago(24h) AND now()
    GROUP BY sensor_id
)
SELECT d.time, d.sensor_id, d.temperature, s.mean_temp, s.std_temp
FROM "iot-database"."sensor-data" d
JOIN stats s ON d.sensor_id = s.sensor_id
WHERE ABS(d.temperature - s.mean_temp) > 2 * s.std_temp
  AND d.time BETWEEN ago(1h) AND now()
```

---

## Scheduled Queries — Truy Vấn Theo Lịch

**Scheduled Queries** (Truy Vấn Theo Lịch) tự động chạy query theo định kỳ và lưu kết quả — tương tự materialized views:

```python
timestream.create_scheduled_query(
    Name='hourly-temperature-summary',
    QueryString="""
        SELECT 
            sensor_id,
            bin(time, 1h) AS hour,
            AVG(temperature) AS avg_temp,
            MAX(temperature) AS max_temp
        FROM "iot-database"."sensor-data"
        WHERE time BETWEEN @scheduled_runtime - 1h AND @scheduled_runtime
        GROUP BY sensor_id, bin(time, 1h)
    """,
    ScheduleConfiguration={
        'ScheduleExpression': 'rate(1 hour)'  # Chạy mỗi giờ
    },
    TargetConfiguration={
        'TimestreamConfiguration': {
            'DatabaseName': 'iot-database',
            'TableName': 'hourly-summaries',  # Lưu kết quả vào table này
            'TimeColumn': 'hour',
            'DimensionMappings': [
                {'Name': 'sensor_id', 'DimensionValueType': 'VARCHAR'}
            ],
            'MultiMeasureMappings': {
                'TargetMultiMeasureName': 'temperature_stats',
                'MultiMeasureAttributeMappings': [
                    {'SourceColumn': 'avg_temp', 'MeasureValueType': 'DOUBLE'},
                    {'SourceColumn': 'max_temp', 'MeasureValueType': 'DOUBLE'},
                ]
            }
        }
    },
    NotificationConfiguration={
        'SnsConfiguration': {'TopicArn': 'arn:aws:sns:us-east-1:123456789:timestream-alerts'}
    },
    ScheduledQueryExecutionRoleArn='arn:aws:iam::123456789:role/TimestreamScheduledQueryRole'
)
```

---

## Storage Tiers — Phân Tầng Lưu Trữ

### Cấu Hình Retention Policy (Chính Sách Lưu Giữ)

```python
timestream.create_table(
    DatabaseName='iot-database',
    TableName='sensor-data',
    RetentionProperties={
        'MemoryStoreRetentionPeriodInHours': 24,    # 24 giờ trong Memory Store
        'MagneticStoreRetentionPeriodInDays': 365   # 1 năm trong Magnetic Store
    }
)
```

### Chi Phí Theo Tier

| Tier | Mô Tả | Chi Phí Tương Đối |
|------|-------|-------------------|
| **Memory Store** | SSD-backed, ultra-fast queries | $$$ (cao nhất) |
| **Magnetic Store** | Cost-optimized, fast queries | $ (thấp hơn ~10x) |

### Best Practice Về Retention

```
Thiết kế retention dựa trên access pattern:
  - Dữ liệu < 1 ngày: Truy vấn real-time, alerting   → Memory Store (24h)
  - Dữ liệu < 1 tháng: Trend analysis, reporting     → Magnetic Store (30 ngày)
  - Dữ liệu < 1 năm: Long-term analysis               → Magnetic Store (365 ngày)
  - Dữ liệu cũ hơn 1 năm: Archive, compliance        → Export sang S3
```

---

## Use Cases Thực Tế

### 1. IoT Sensor Monitoring (Giám Sát Cảm Biến IoT)

```
Nodes: 10,000 sensors
Frequency: Mỗi sensor ghi 1 record/giây
Volume: 10,000 records/giây = 864M records/ngày

Architecture:
  IoT Device → AWS IoT Core → Kinesis → Lambda → Timestream
                                                      ↓
  Grafana Dashboard ← Timestream Query ←─────────────┘
```

### 2. Application Performance Monitoring — APM (Giám Sát Hiệu Năng Ứng Dụng)

```sql
-- Phân tích latency của API endpoints theo giờ
SELECT 
    endpoint,
    bin(time, 1h)       AS hour,
    AVG(latency_ms)     AS avg_latency,
    APPROX_PERCENTILE(latency_ms, 0.95) AS p95_latency,
    APPROX_PERCENTILE(latency_ms, 0.99) AS p99_latency,
    COUNT(*)            AS request_count
FROM "app-monitoring"."api-metrics"
WHERE time BETWEEN ago(24h) AND now()
GROUP BY endpoint, bin(time, 1h)
HAVING AVG(latency_ms) > 500  -- Chỉ endpoints chậm > 500ms
ORDER BY avg_latency DESC
```

### 3. DevOps Metrics (Số Liệu DevOps)

```
Sources: CloudWatch metrics, container metrics, custom app metrics
Frequency: 1-60 giây/metric
Use case: 
  - Real-time dashboard trong Grafana
  - Alert khi CPU > 80% liên tục 5 phút
  - Capacity planning dựa trên historical trends
```

### 4. Financial Market Data (Dữ Liệu Thị Trường Tài Chính)

```sql
-- Tính VWAP (Volume-Weighted Average Price — Giá Trung Bình Theo Khối Lượng)
SELECT 
    symbol,
    bin(time, 5m) AS time_window,
    SUM(price * volume) / SUM(volume) AS vwap,
    SUM(volume) AS total_volume
FROM "market-data"."trades"
WHERE symbol IN ('AMZN', 'GOOG', 'MSFT')
  AND time BETWEEN ago(1h) AND now()
GROUP BY symbol, bin(time, 5m)
```

---

## Tích Hợp Với AWS Services

### Ingest Sources (Nguồn Nhập Dữ Liệu)

```
AWS IoT Core    → Rules Engine → Timestream         (IoT use case)
Kinesis         → Lambda       → Timestream         (Stream processing)
CloudWatch      → Metric Streams → Kinesis → Lambda → Timestream
MSK (Kafka)     → Lambda       → Timestream         (Event streaming)
AWS SDK         →              → Timestream         (Custom applications)
```

### Visualization & Analytics (Trực Quan Hóa & Phân Tích)

```
Timestream → Grafana       (Real-time dashboards)
Timestream → QuickSight    (BI dashboards)
Timestream → SageMaker     (ML trên time-series data)
Timestream → S3            (Export cho long-term archival)
```

### Grafana Integration

```yaml
# Grafana datasource config
datasources:
  - name: Amazon Timestream
    type: grafana-timestream-datasource
    jsonData:
      defaultRegion: us-east-1
      defaultDatabase: iot-database
      defaultTable: sensor-data
```

---

## So Sánh Timestream vs Các Lựa Chọn Khác

### Timestream vs InfluxDB (Self-Managed)

| Tiêu Chí | Timestream | InfluxDB (self-managed) |
|----------|------------|-------------------------|
| **Quản lý** | Fully managed | Tự quản lý |
| **Scale** | Tự động | Phải plan trước |
| **Cost** | Pay-per-use | Fixed infrastructure cost |
| **Integration** | AWS native | Generic |
| **Query language** | SQL-like | Flux hoặc InfluxQL |

### Timestream vs DynamoDB (cho time-series)

```
DynamoDB cho time-series:
  - Sort key là timestamp
  - Phải tự quản lý TTL deletion
  - Không có native time aggregation functions
  - Tốt khi dữ liệu cần strong consistency

Timestream:
  - Tối ưu hóa cho time-series (compression, tiering, functions)
  - Tự động tiering hot→cold
  - Rich time functions (bin, interpolate, anomaly detection)
  - Dùng khi time-series là core workload
```

### Timestream vs CloudWatch Metrics

```
CloudWatch: Tốt cho AWS infrastructure monitoring
  - Tự động collect metrics từ AWS services
  - Built-in alerting, dashboards
  - Giới hạn 15 tháng retention

Timestream: Tốt cho custom application metrics
  - Flexible schema, bất kỳ data
  - Retention lên đến nhiều năm
  - SQL queries linh hoạt hơn
  - Giá thấp hơn cho large volume custom metrics
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao cần time-series database thay vì dùng DynamoDB?

> **Trả lời:** Time-series database như Timestream được tối ưu đặc biệt cho time-series data:
> 1. **Compression:** Lưu trữ thay đổi delta, nén tốt hơn RDS/DynamoDB ~90%
> 2. **Auto-tiering:** Tự động chuyển dữ liệu cũ từ Memory (nhanh, đắt) sang Magnetic (rẻ hơn)
> 3. **Native time functions:** `bin(time, 5m)`, `interpolate_linear()`, `approx_percentile()`
> 4. **Write optimization:** Tối ưu cho append-only high-throughput writes
> 5. **Retention management:** Tự động xóa dữ liệu hết hạn
>
> DynamoDB không có những tính năng này natively — phải tự implement.

### Q2: Memory Store và Magnetic Store khác nhau thế nào?

> **Trả lời:** Memory Store (Bộ Nhớ Nhanh) lưu dữ liệu gần đây (cấu hình vài giờ đến vài ngày), được backed bởi SSD — query cực nhanh nhưng giá cao. Magnetic Store (Lưu Trữ Từ Tính) lưu dữ liệu lịch sử lâu hơn, giá thấp hơn ~10x. Timestream tự động tier dữ liệu từ Memory sang Magnetic khi đến retention threshold. Ứng dụng query không cần biết data ở tier nào — transparent.

### Q3: Timestream phù hợp cho IoT use case không?

> **Trả lời:** Rất phù hợp. IoT có đặc điểm: hàng nghìn đến hàng triệu sensors ghi liên tục (append-only), cần phân tích theo thời gian (average/max theo giờ/ngày), dữ liệu cũ ít truy vấn hơn. Timestream xử lý tốt tất cả: serverless scale cho ingestion spike, auto-tiering giảm chi phí dữ liệu cũ, SQL-like queries với time functions cho analytics. Tích hợp tự nhiên với AWS IoT Core qua Rules Engine.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
