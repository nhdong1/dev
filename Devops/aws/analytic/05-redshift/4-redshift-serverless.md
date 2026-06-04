# Redshift Serverless — Không Quản Lý Cluster

> Redshift Serverless cho phép chạy analytics workload mà không cần provision (cung cấp) hay quản lý cluster. Tự động scale theo workload và chỉ tính phí khi thực sự xử lý queries.

---

## 📚 Mục Lục

1. [Redshift Serverless Là Gì?](#1-redshift-serverless-là-gì)
2. [Kiến Trúc Serverless](#2-kiến-trúc-serverless)
3. [RPU — Redshift Processing Units](#3-rpu--redshift-processing-units)
4. [Namespace và Workgroup](#4-namespace-và-workgroup)
5. [Thiết Lập Redshift Serverless](#5-thiết-lập-redshift-serverless)
6. [Auto Scaling Và Cơ Chế Tính Phí](#6-auto-scaling-và-cơ-chế-tính-phí)
7. [Serverless vs Provisioned — So Sánh Toàn Diện](#7-serverless-vs-provisioned--so-sánh-toàn-diện)
8. [Use Cases Phù Hợp](#8-use-cases-phù-hợp)
9. [Performance Tuning Cho Serverless](#9-performance-tuning-cho-serverless)
10. [Migration Từ Provisioned Sang Serverless](#10-migration-từ-provisioned-sang-serverless)
11. [Monitoring Serverless](#11-monitoring-serverless)

---

## 1. Redshift Serverless Là Gì?

Redshift Serverless (Ra mắt 2021, GA tháng 7/2022) là tùy chọn triển khai Redshift mà:

- **Không cần chọn node type hay số lượng node**
- **Tự động scale** capacity lên/xuống theo workload
- **Không tính phí khi idle** (không có query chạy)
- **Tự động pause** sau khoảng thời gian không hoạt động

```
Redshift Provisioned (cũ):
  Bạn chọn: 2 x ra3.4xlarge nodes
  → Chạy 24/7 dù có query hay không
  → Trả tiền theo giờ × số nodes
  → $0.96/giờ × 2 nodes × 24h × 30 ngày = ~$1,382/tháng

Redshift Serverless (mới):
  Bạn chọn: Base RPU = 32
  → Chỉ chạy khi có query
  → Trả tiền theo RPU-giây thực tế dùng
  → 8 giờ query/ngày × $0.36/RPU-giờ × 32 RPU × 30 ngày = ~$2,765/tháng (nếu dùng nhiều)
  → 2 giờ query/ngày = ~$691/tháng
  → Không có query: $0
```

---

## 2. Kiến Trúc Serverless

### Mô Hình Tổng Thể

```
                    ┌────────────────────────────────────┐
                    │         Client / BI Tools           │
                    └──────────────────┬─────────────────┘
                                       │ JDBC/ODBC/API
                    ┌──────────────────▼─────────────────┐
                    │       Redshift Serverless           │
                    │                                     │
                    │  ┌───────────────────────────────┐  │
                    │  │         Workgroup             │  │
                    │  │  (Endpoint + Compute Config)  │  │
                    │  │                               │  │
                    │  │  Base RPU: 32 (configurable)  │  │
                    │  │  Auto Scaling: tự động        │  │
                    │  │  VPC + Security Groups        │  │
                    │  └───────────────────────────────┘  │
                    │               │                     │
                    │  ┌────────────▼──────────────────┐  │
                    │  │         Namespace             │  │
                    │  │  (Database + Users + Schemas) │  │
                    │  │                               │  │
                    │  │  - Databases                  │  │
                    │  │  - Users & Permissions        │  │
                    │  │  - Schemas & Tables           │  │
                    │  └───────────────────────────────┘  │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │     Redshift Managed Storage         │
                    │     (S3-backed, tự động tăng)        │
                    └─────────────────────────────────────┘
```

### So Sánh Mô Hình Control Plane

```
Provisioned:                    Serverless:
  Leader Node → Bạn quản lý      Leader Node → AWS quản lý hoàn toàn
  Compute Nodes → Bạn chọn       Compute → AWS auto-scale
  Scaling → Manual resize         Scaling → Tự động
  Pause → Manual                 Pause → Tự động sau idle
  Failover → Cần Multi-AZ setup  Failover → Built-in, tự động
```

---

## 3. RPU — Redshift Processing Units

**RPU** (Redshift Processing Units — Đơn Vị Xử Lý Redshift) là đơn vị đo lường capacity trong Serverless.

### RPU Là Gì?

```
1 RPU = một đơn vị tính toán gồm CPU + RAM + Networking

Cấu hình:
  - Base Capacity (Năng Lực Cơ Sở): 32 RPU (mặc định, tối thiểu)
  - Maximum Capacity (Năng Lực Tối Đa): 512 RPU (mặc định, tùy chỉnh được)
  - Bước nhảy: tăng/giảm theo bội số của 8 RPU

Mapping gần đúng:
  32 RPU  ≈ Cluster nhỏ (2 x ra3.xlplus)
  64 RPU  ≈ Cluster vừa (2 x ra3.4xlarge)
  128 RPU ≈ Cluster lớn (2 x ra3.16xlarge)
  512 RPU ≈ Cluster rất lớn
```

### Cách RPU Scale

```
Khi workload nhẹ (1 query nhỏ):
  RPU = Base Capacity = 32 RPU

Khi workload cao (10 queries phức tạp cùng lúc):
  RPU tự động scale lên → tối đa Max Capacity

Khi không có query trong > X phút:
  Cluster tự động suspend (dừng) → không tính phí RPU
  Query mới đến → cluster resume trong vài giây
```

### Giới Hạn RPU và Workload

```
RPU thấp hơn:
  + Rẻ hơn
  - Queries lớn chạy chậm hơn
  - Ít concurrent queries hơn

RPU cao hơn:
  + Queries nhanh hơn
  + Nhiều concurrent queries hơn
  - Đắt hơn

Cài đặt gợi ý:
  Dev/Test:     Base = 32 RPU, Max = 64 RPU
  Nhỏ-vừa:    Base = 32 RPU, Max = 128 RPU (mặc định)
  Production:  Base = 64 RPU, Max = 256-512 RPU
```

---

## 4. Namespace và Workgroup

### Namespace — Không Gian Tên

**Namespace** chứa tất cả objects database: databases, schemas, tables, users, permissions và data storage.

```
Namespace tương đương với "cluster data" trong Provisioned:
  - Databases và schemas
  - Users và quyền hạn
  - Dữ liệu (lưu trong Redshift Managed Storage)
  - Audit logs và snapshots

Một Namespace có thể được kết nối với nhiều Workgroups:
  → Ví dụ: Namespace prod_data kết nối với:
     - Workgroup production (RPU cao, VPC prod)
     - Workgroup reporting (RPU trung bình, VPC analytics)
```

### Workgroup — Nhóm Làm Việc

**Workgroup** chứa cấu hình compute và network:

```
Workgroup cấu hình:
  - Base RPU và Max RPU
  - VPC và Security Groups
  - Subnet Groups
  - Enhanced VPC Routing
  - Query limits (timeout, rows, bytes)
  - Parameter groups (tương tự Provisioned)

Một Workgroup kết nối đến một Namespace:
  workgroup_endpoint → Namespace data
```

```bash
# Tạo namespace
aws redshift-serverless create-namespace \
  --namespace-name prod-namespace \
  --admin-username admin \
  --admin-user-password "MySecurePass123!" \
  --db-name analytics \
  --log-exports '["userlog", "connectionlog", "useractivitylog"]'

# Tạo workgroup
aws redshift-serverless create-workgroup \
  --workgroup-name prod-workgroup \
  --namespace-name prod-namespace \
  --base-capacity 64 \
  --max-capacity 256 \
  --subnet-ids subnet-xxxxxxxx subnet-yyyyyyyy \
  --security-group-ids sg-xxxxxxxx \
  --publicly-accessible false
```

---

## 5. Thiết Lập Redshift Serverless

### Qua AWS Console (Nhanh Nhất)

```
1. Vào Amazon Redshift → Serverless dashboard
2. Click "Get started" → "Create workgroup"
3. Đặt tên workgroup: my-analytics-workgroup
4. Chọn Base RPU: 32 (dev) hoặc 64 (production)
5. Chọn VPC và Subnets
6. Tạo mới Namespace hoặc chọn có sẵn
7. Đặt Admin username/password
8. Click Create
→ Workgroup ready trong ~1-2 phút
```

### Kết Nối Tới Serverless

```python
import redshift_connector

# Kết nối qua JDBC/ODBC endpoint
conn = redshift_connector.connect(
    host='my-workgroup.123456789.ap-southeast-1.redshift-serverless.amazonaws.com',
    port=5439,
    database='analytics',
    user='admin',
    password='MySecurePass123!',
    # Hoặc dùng IAM authentication:
    # iam=True,
    # region='ap-southeast-1'
)

cursor = conn.cursor()
cursor.execute("SELECT current_user, current_database()")
print(cursor.fetchone())
conn.close()
```

### Kết Nối Với IAM Authentication (Khuyến Nghị)

```bash
# Lấy temporary credentials qua IAM
aws redshift-serverless get-credentials \
  --workgroup-name my-workgroup \
  --db-name analytics \
  --duration-seconds 3600

# Hoặc dùng trong connection string:
# jdbc:redshift:iam://my-workgroup.123456789.ap-southeast-1.redshift-serverless.amazonaws.com:5439/analytics
```

---

## 6. Auto Scaling Và Cơ Chế Tính Phí

### Cơ Chế Auto Scaling

```
Trigger Scale Up (Tăng RPU):
  - Queue queries chờ quá 30 giây
  - Memory pressure (áp lực bộ nhớ) cao
  - Disk I/O bottleneck
  → Tăng RPU tự động (trong vài giây đến vài phút)

Trigger Scale Down (Giảm RPU):
  - Workload giảm
  - Queue trống
  → Giảm RPU từ từ để tránh impact queries đang chạy

Trigger Suspend (Dừng Hoàn Toàn):
  - Không có query nào trong X phút (có thể cấu hình idle timeout)
  → Suspend tất cả compute
  → Tính phí $0 trong lúc suspend

Auto Resume (Tiếp Tục Tự Động):
  - Query mới đến khi cluster đang suspend
  → Resume tự động trong ~3-5 giây
  → Query chờ trong lúc resume (cold start latency)
```

### Mô Hình Tính Phí Chi Tiết

```
Phí Serverless:
  $0.36 / RPU-hour = $0.0001 / RPU-giây

Ví dụ tính toán:
  Query chạy 10 giây với 64 RPU:
  → 64 RPU × 10 giây × $0.0001/RPU-giây = $0.064

  1,000 queries/ngày × 10 giây × 64 RPU:
  → 1,000 × 10 × 64 × $0.0001 = $64/ngày = ~$1,920/tháng

  Không có query: $0

  Storage (dữ liệu trong Namespace):
  → $0.024/GB-tháng (giống Provisioned RA3)

So với Provisioned:
  2 x ra3.xlplus (32 RPU equivalent):
  → 2 nodes × $0.96/node-giờ × 730 giờ/tháng = $1,402/tháng (dù có query hay không)
```

### Tối Ưu Chi Phí Serverless

```sql
-- Bật usage limits để tránh chi phí bất ngờ
-- Trong AWS Console hoặc:
aws redshift-serverless put-resource-policy \
  --resource-arn arn:aws:redshift-serverless:... \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [...]
  }'

-- Đặt query timeout để tránh queries chạy quá lâu
CREATE EXTERNAL SCHEMA ...;

-- Thiết lập usage limit qua CLI
aws redshift-serverless create-usage-limit \
  --resource-arn arn:aws:redshift-serverless:ap-southeast-1:123456789:workgroup/my-workgroup \
  --usage-type serverless-compute \
  --amount 100 \      # 100 RPU-hours
  --period monthly \
  --breach-action emit-metric  # hoặc deactivate (tắt workgroup khi vượt limit)
```

---

## 7. Serverless vs Provisioned — So Sánh Toàn Diện

| Tiêu Chí | Provisioned | Serverless |
|----------|-------------|------------|
| **Quản lý** | Phải chọn node type, số node | Chỉ chọn Base/Max RPU |
| **Chi phí khi idle** | Tính phí 24/7 dù không dùng | Miễn phí khi suspend |
| **Chi phí khi dùng nhiều** | Thường rẻ hơn | Có thể đắt hơn |
| **Scaling** | Manual hoặc Elastic Resize | Tự động, trong giây |
| **Cold start** | Không (luôn running) | 3-5 giây sau suspend |
| **Concurrency** | Giới hạn bởi WLM | Linh hoạt hơn |
| **DISTKEY/SORTKEY** | Đầy đủ hỗ trợ | Đầy đủ hỗ trợ |
| **Spectrum** | Hỗ trợ | Hỗ trợ |
| **RA3/Multi-AZ** | Có | Built-in |
| **Tốt nhất cho** | Workload ổn định, dự đoán được | Workload không đều, intermittent |

### Decision Tree — Cây Quyết Định

```
Workload của bạn có thể dự đoán không?
  │
  ├── Có, chạy 8+ giờ/ngày, query phức tạp thường xuyên
  │     → Provisioned có thể rẻ hơn
  │
  └── Không, hoặc workload không đều
        │
        ├── Dev/Test environment → Serverless (pay as you go)
        ├── Reporting chạy theo schedule → Serverless (chỉ tốn phí khi chạy)
        ├── Startup/MVP chưa biết traffic → Serverless (không overcommit)
        └── Production với peak time ngắn → Serverless (scale khi cần)
```

---

## 8. Use Cases Phù Hợp

### ✅ Serverless Phù Hợp

```
1. Dev/Test/Staging Environments (Môi Trường Phát Triển/Kiểm Thử):
   → Dev chỉ dùng 2-4 giờ/ngày → tiết kiệm 80% so với Provisioned
   → Không cần pause/resume thủ công

2. Scheduled Reporting (Báo Cáo Theo Lịch):
   → Report chạy lúc 6 giờ sáng trong 30 phút
   → 23.5 giờ còn lại không tốn phí
   → Provisioned sẽ tốn phí 24/7

3. Exploratory Analytics (Phân Tích Khám Phá):
   → Data scientists query bất thường, không đoán được
   → Serverless tự scale khi cần, tự dừng khi không dùng

4. Multi-tenant Analytics (Phân Tích Đa Khách Hàng):
   → Mỗi khách hàng có Namespace riêng
   → Workgroup chung hoặc riêng tùy yêu cầu

5. Startup / Dự Án Mới:
   → Không biết trước traffic sẽ như thế nào
   → Không muốn overcommit infrastructure
```

### ❌ Provisioned Phù Hợp Hơn

```
1. Workload ổn định 24/7:
   → Nhiều BI dashboards, query liên tục
   → Provisioned thường rẻ hơn khi utilization > 60-70%

2. Latency-sensitive Production:
   → Cold start 3-5 giây không chấp nhận được
   → SLA < 1 giây cho mọi query

3. Very Large Workloads:
   → Cần > 512 RPU liên tục
   → Provisioned lớn thường rẻ hơn Serverless ở scale này

4. Cần Tùy Chỉnh Sâu:
   → Custom WLM queue phức tạp
   → Parameter tuning chi tiết
```

---

## 9. Performance Tuning Cho Serverless

### Cải Thiện Query Performance

```sql
-- 1. Vẫn cần DISTKEY và SORTKEY trong Serverless (hoạt động tương tự Provisioned)
CREATE TABLE orders (
    order_id    BIGINT,
    customer_id BIGINT,
    amount      DECIMAL(12,2),
    order_date  DATE
)
DISTKEY (customer_id)
SORTKEY (order_date);

-- 2. ANALYZE để optimizer hoạt động tốt
ANALYZE orders;

-- 3. Tăng RPU nếu query thường xuyên chậm
aws redshift-serverless update-workgroup \
  --workgroup-name my-workgroup \
  --base-capacity 64  -- Tăng từ 32 lên 64 RPU

-- 4. Xem query performance
SELECT query_id, status, error,
       compute_seconds,
       queue_time / 1000000 AS queue_sec,
       execution_time / 1000000 AS exec_sec
FROM sys_query_history
WHERE start_time > CURRENT_TIMESTAMP - INTERVAL '24 hours'
ORDER BY start_time DESC
LIMIT 20;
```

### Query Limits — Giới Hạn Query

```sql
-- Tránh runaway queries (query chạy không kiểm soát được) với query limits
-- Cấu hình trong Workgroup:

aws redshift-serverless update-workgroup \
  --workgroup-name my-workgroup \
  --config-parameters '[
    {"parameterKey": "max_query_execution_time", "parameterValue": "600"},
    {"parameterKey": "datestyle", "parameterValue": "ISO, MDY"},
    {"parameterKey": "enable_user_activity_logging", "parameterValue": "true"}
  ]'
```

---

## 10. Migration Từ Provisioned Sang Serverless

### Các Phương Pháp Migration

#### Phương Pháp 1: Snapshot Restore (Khôi Phục Từ Snapshot)

```bash
# Tạo snapshot từ Provisioned cluster
aws redshift create-cluster-snapshot \
  --cluster-identifier my-provisioned-cluster \
  --snapshot-identifier migration-snapshot-$(date +%Y%m%d)

# Restore snapshot vào Serverless Namespace
aws redshift-serverless restore-from-snapshot \
  --namespace-name my-serverless-namespace \
  --workgroup-name my-serverless-workgroup \
  --snapshot-arn arn:aws:redshift:ap-southeast-1:123456789:snapshot:my-provisioned-cluster/migration-snapshot-20240101
```

#### Phương Pháp 2: UNLOAD/COPY Qua S3

```bash
# Bước 1: Trong Provisioned cluster, UNLOAD dữ liệu ra S3
# (Chạy với psql hoặc Redshift Query Editor)

# Bước 2: Trong Serverless, COPY từ S3
# psql -h serverless-endpoint -U admin -d analytics

COPY target_table
FROM 's3://migration-bucket/table-name/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftServerlessRole'
FORMAT AS PARQUET;
```

### Checklist Migration

```
Trước migration:
  □ Kiểm tra tất cả table definitions (DISTKEY, SORTKEY, ENCODE)
  □ Export users và permissions
  □ Document các stored procedures và views
  □ Đo performance baseline của Provisioned

Trong quá trình:
  □ Chạy song song: test queries trên cả hai
  □ So sánh kết quả và performance
  □ Test với production-like workload

Sau migration:
  □ Update connection strings trong BI tools
  □ Update Glue JDBC connections
  □ Monitor CloudWatch metrics tuần đầu
  □ Tắt Provisioned cluster sau khi confirm OK
```

---

## 11. Monitoring Serverless

### CloudWatch Metrics Quan Trọng

```bash
# Xem compute usage (RPU usage)
aws cloudwatch get-metric-statistics \
  --namespace AWS/Redshift-Serverless \
  --metric-name ComputeSeconds \
  --dimensions Name=Workgroup,Value=my-workgroup \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Sum
```

| Metric | Ý Nghĩa | Alert Khi |
|--------|---------|----------|
| **ComputeSeconds** | Tổng RPU-giây đã dùng | Vượt budget |
| **QueriesCompletedPerSecond** | Throughput query | Giảm đột ngột |
| **QueriesFailedPerSecond** | Tỷ lệ query thất bại | > 0 liên tục |
| **QueryDuration** | Thời gian query trung bình | Tăng đột biến |
| **DatabaseConnections** | Số connections hiện tại | Tiệm cận limit |

### SQL Monitoring Trong Serverless

```sql
-- Xem tất cả queries đang chạy
SELECT query_id, user_name, status,
       DATEDIFF(seconds, start_time, GETDATE()) AS running_sec,
       TRIM(SUBSTRING(query_text, 1, 100)) AS query_preview
FROM sys_query_history
WHERE status = 'running'
ORDER BY start_time;

-- Top queries tốn nhiều compute nhất
SELECT query_id, user_name,
       compute_seconds,
       ROUND(compute_seconds / 3600.0 * 0.36, 4) AS estimated_cost_usd,
       TRIM(SUBSTRING(query_text, 1, 150)) AS query_preview
FROM sys_query_history
WHERE start_time > CURRENT_TIMESTAMP - INTERVAL '7 days'
  AND status = 'completed'
ORDER BY compute_seconds DESC
LIMIT 10;

-- Phân tích usage theo giờ trong ngày (để quyết định base RPU)
SELECT
    EXTRACT(HOUR FROM start_time) AS hour_of_day,
    COUNT(*) AS query_count,
    SUM(compute_seconds) AS total_compute_sec,
    MAX(compute_seconds) AS max_single_query_sec
FROM sys_query_history
WHERE start_time > CURRENT_TIMESTAMP - INTERVAL '30 days'
  AND status = 'completed'
GROUP BY 1
ORDER BY 1;
```

---

## 🔑 Tóm Tắt Key Points

```
1. Serverless = không quản lý cluster, scale tự động, pause tự động → zero cost khi idle
2. RPU (Redshift Processing Units) = đơn vị capacity; 1 RPU ≈ 1 vCPU + RAM proportional
3. Base RPU = minimum capacity luôn sẵn sàng; Max RPU = giới hạn scale-up
4. Namespace = database objects + storage; Workgroup = compute + network config
5. Tính phí: $0.36/RPU-hour — chỉ tính khi query đang chạy, không tính khi suspend
6. Cold start 3-5 giây sau suspend — không phù hợp cho SLA < 1 giây
7. Serverless rẻ hơn Provisioned khi utilization < 40-50% thời gian làm việc
8. DISTKEY, SORTKEY, ENCODE vẫn quan trọng như Provisioned — cùng query engine
9. Snapshot restore là cách nhanh nhất để migrate từ Provisioned sang Serverless
10. Usage limits tránh chi phí bất ngờ — luôn cài đặt cho production
```

---

## 🆚 Tổng Kết Module 05-redshift

```
Đã học trong module này:
  1-redshift-architecture.md  → MPP, Leader/Compute Nodes, Columnar Storage, RA3
  2-redshift-performance.md   → DISTKEY, SORTKEY, DISTSTYLE, WLM, VACUUM
  3-redshift-spectrum.md      → Query S3 từ Redshift, Data Lakehouse pattern
  4-redshift-serverless.md    → Auto scaling, RPU, Provisioned vs Serverless

Redshift trong hệ sinh thái AWS Analytics:
  S3 (data lake) → Glue (ETL) → Redshift (data warehouse) → QuickSight (BI)
                             ↗
  Kinesis (streaming) → Firehose → Redshift (real-time ingestion)
                                 ↗
  Redshift Spectrum → S3 (cold data) = Data Lakehouse

Câu hỏi phỏng vấn thường gặp:
  Q: Chọn DISTKEY như thế nào?
  A: Chọn cột JOIN thường xuyên nhất, cardinality cao, tránh data skew

  Q: Redshift Spectrum vs Athena?
  A: Spectrum khi cần JOIN với Redshift data; Athena khi serverless pure, không cần cluster

  Q: Serverless khi nào?
  A: Workload không đều, dev/test, scheduled batch — khi idle cost của Provisioned lãng phí

  Q: VACUUM khi nào?
  A: Sau nhiều DELETE/UPDATE (thu hồi không gian), khi sorted_rows% thấp (sort lại)
```

---

**Quay Lại:** [README.md](./README.md) — Tổng quan module Redshift

**Module Tiếp Theo:** [../06-emr/README.md](../06-emr/README.md) — Amazon EMR — Big Data Processing với Spark
