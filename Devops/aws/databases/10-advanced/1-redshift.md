# 1. Amazon Redshift — Data Warehouse Trên AWS

> Amazon Redshift là dịch vụ **data warehouse** (kho dữ liệu) được quản lý hoàn toàn trên đám mây, được tối ưu cho **OLAP** (Online Analytical Processing — Xử Lý Phân Tích Trực Tuyến) với khả năng phân tích petabyte dữ liệu.

## 📚 Mục Lục

1. [OLAP vs OLTP](#olap-vs-oltp)
2. [Kiến Trúc Redshift](#kiến-trúc-redshift)
3. [Distribution Keys — Khóa Phân Phối Dữ Liệu](#distribution-keys--khóa-phân-phối-dữ-liệu)
4. [Sort Keys — Khóa Sắp Xếp](#sort-keys--khóa-sắp-xếp)
5. [Redshift Spectrum](#redshift-spectrum)
6. [Concurrency Scaling](#concurrency-scaling)
7. [Redshift Serverless](#redshift-serverless)
8. [Data Loading & Ingestion](#data-loading--ingestion)
9. [Performance Optimization](#performance-optimization)
10. [Bảo Mật & Compliance](#bảo-mật--compliance)
11. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## OLAP vs OLTP

Hiểu sự khác biệt này là nền tảng để hiểu tại sao cần Redshift.

| Tiêu Chí | OLTP (Xử Lý Giao Dịch) | OLAP (Xử Lý Phân Tích) |
|----------|------------------------|------------------------|
| **Mục đích** | Ghi và đọc giao dịch thời gian thực | Phân tích dữ liệu lớn, báo cáo |
| **Workload** | Nhiều giao dịch nhỏ, concurrent | Ít truy vấn lớn, quét toàn bảng |
| **Dữ liệu** | Dữ liệu hiện tại, mới nhất | Dữ liệu lịch sử, terabyte/petabyte |
| **Ví dụ** | RDS, Aurora, DynamoDB | Redshift, Snowflake, BigQuery |
| **Query pattern** | `SELECT * WHERE id = 123` | `SELECT SUM(revenue) GROUP BY region` |
| **Row vs Column** | Row-oriented storage (Lưu theo hàng) | Column-oriented storage (Lưu theo cột) |

### Tại Sao Column-Oriented Storage Tốt Hơn Cho Analytics?

```
Row-oriented (RDS):
┌────┬──────────┬──────────┬─────────┐
│ id │ customer │ product  │ revenue │
├────┼──────────┼──────────┼─────────┤
│  1 │ Alice    │ Laptop   │  1500   │  ← đọc toàn hàng
│  2 │ Bob      │ Phone    │   800   │  ← dù chỉ cần revenue
│  3 │ Carol    │ Tablet   │  1200   │
└────┴──────────┴──────────┴─────────┘

Column-oriented (Redshift):
revenue: [1500, 800, 1200, ...]  ← chỉ đọc cột này
→ Ít I/O hơn, nén tốt hơn, tổng hợp nhanh hơn
```

---

## Kiến Trúc Redshift

### Các Thành Phần Chính

```
┌─────────────────────────────────────────────────────────┐
│                    REDSHIFT CLUSTER                      │
│                                                         │
│  ┌─────────────────┐    ┌──────────────────────────┐   │
│  │   Leader Node   │    │      Compute Nodes        │   │
│  │  (Nút Dẫn Đầu) │───▶│   (Các Nút Tính Toán)    │   │
│  │                 │    │  ┌──────┐  ┌──────┐       │   │
│  │ - Lập kế hoạch  │    │  │Node 1│  │Node 2│  ...  │   │
│  │   truy vấn      │    │  │Slice1│  │Slice3│       │   │
│  │ - Tổng hợp kết  │    │  │Slice2│  │Slice4│       │   │
│  │   quả           │    │  └──────┘  └──────┘       │   │
│  │ - Giao tiếp     │    └──────────────────────────┘   │
│  │   với clients   │                                    │
│  └─────────────────┘                                    │
└─────────────────────────────────────────────────────────┘
```

### Leader Node (Nút Dẫn Đầu)

- Tiếp nhận kết nối từ client (JDBC/ODBC)
- Parse và tối ưu hóa query
- Điều phối các Compute Nodes thực thi query song song
- Tổng hợp kết quả và trả về client

### Compute Nodes (Các Nút Tính Toán)

- Mỗi node chia thành các **slices** (lát cắt) — đơn vị xử lý song song
- Số slices phụ thuộc vào loại node (2, 4, 8, 16, 32 slices/node)
- Dữ liệu được phân phối (distribute) trên các slices theo distribution key

### Node Types (Loại Node)

| Loại | Đặc Điểm | Use Case |
|------|----------|----------|
| **RA3** | Storage ở S3, compute tách biệt | Linh hoạt, khuyên dùng |
| **DC2** (Dense Compute) | SSD local, tính toán nhanh | Dữ liệu dưới 1TB |
| **DS2** (Dense Storage) | HDD local, storage lớn | Legacy, dữ liệu lớn |

---

## Distribution Keys — Khóa Phân Phối Dữ Liệu

Distribution key quyết định **hàng nào lưu ở node/slice nào**. Chọn đúng distribution key giảm thiểu data movement (di chuyển dữ liệu) giữa các nodes trong query.

### Các Distribution Styles (Kiểu Phân Phối)

#### 1. KEY Distribution

```sql
CREATE TABLE orders (
    order_id    INT,
    customer_id INT,
    amount      DECIMAL(10,2),
    order_date  DATE
)
DISTKEY (customer_id);  -- Các hàng cùng customer_id → cùng slice
```

- Hàng có cùng `distkey` → lưu cùng slice
- Tốt nhất khi thường xuyên JOIN trên cột này
- **Nguy cơ:** Data skew (Lệch Dữ Liệu) nếu phân phối không đều

#### 2. ALL Distribution

```sql
CREATE TABLE dim_product (
    product_id   INT,
    product_name VARCHAR(200),
    category     VARCHAR(100)
)
DISTSTYLE ALL;  -- Bản sao đầy đủ ở mỗi node
```

- Toàn bộ bảng được sao chép (copy) tới **mỗi node**
- Tốt cho bảng nhỏ thường xuyên JOIN với bảng lớn (dimension tables)
- **Đánh đổi:** Tốn storage, write chậm hơn

#### 3. EVEN Distribution

```sql
CREATE TABLE raw_events (
    event_id   INT,
    event_data VARCHAR(MAX)
)
DISTSTYLE EVEN;  -- Phân phối đều (round-robin)
```

- Hàng phân phối **round-robin** (xoay vòng đều) trên các slices
- Không có JOIN key rõ ràng
- Tránh data skew nhưng tăng data movement khi JOIN

#### 4. AUTO Distribution (Khuyên Dùng)

```sql
CREATE TABLE sales_fact (
    sale_id     INT,
    customer_id INT,
    amount      DECIMAL
)
DISTSTYLE AUTO;  -- Redshift tự quyết định
```

- Redshift tự phân tích workload và chọn style phù hợp
- Bắt đầu với ALL (nếu bảng nhỏ), tự chuyển sang KEY khi bảng lớn

### Chọn Distribution Key Như Thế Nào?

```
1. Xác định bảng fact lớn nhất (largest fact table)
2. Tìm cột JOIN thường xuyên nhất giữa fact và dimension
3. Đặt DISTKEY trên cột đó
4. Đảm bảo cột có cardinality (số lượng giá trị phân biệt) cao
5. Tránh cột có skewed distribution (timestamp, boolean)
```

---

## Sort Keys — Khóa Sắp Xếp

Sort key quyết định thứ tự lưu trữ vật lý của dữ liệu trên disk, giúp Redshift **bỏ qua block** không cần thiết (zone map — bản đồ vùng).

### Compound Sort Key (Khóa Sắp Xếp Kết Hợp)

```sql
CREATE TABLE sales (
    sale_date   DATE,
    region      VARCHAR(50),
    product_id  INT,
    revenue     DECIMAL(10,2)
)
SORTKEY (sale_date, region);  -- Sắp xếp theo date → rồi theo region
```

- Hiệu quả nhất khi WHERE clause dùng đúng thứ tự sort key
- `WHERE sale_date = '2024-01-01'` → rất nhanh
- `WHERE region = 'US'` → ít hiệu quả hơn (không phải leading column)

### Interleaved Sort Key (Khóa Sắp Xếp Xen Kẽ)

```sql
CREATE TABLE sales (
    sale_date   DATE,
    region      VARCHAR(50),
    product_id  INT,
    revenue     DECIMAL(10,2)
)
INTERLEAVED SORTKEY (sale_date, region, product_id);
```

- Cân bằng hiệu quả cho nhiều cột khác nhau
- Phù hợp khi query dùng bất kỳ tổ hợp cột sort key
- **Đánh đổi:** VACUUM (dọn dẹp) chậm hơn compound sort key

---

## Redshift Spectrum

**Redshift Spectrum** cho phép truy vấn dữ liệu trực tiếp từ **S3** mà không cần load vào Redshift cluster.

### Kiến Trúc Spectrum

```
┌──────────────┐     ┌─────────────────────┐     ┌──────────────┐
│   Redshift   │────▶│  Spectrum Layer      │────▶│   S3 Data    │
│   Cluster    │     │  (Hàng nghìn nodes   │     │  (Parquet,   │
│  (Leader +   │◀────│   độc lập, parallel) │     │   ORC, CSV)  │
│  Compute)    │     └─────────────────────┘     └──────────────┘
└──────────────┘
         ▲
         │ AWS Glue Data Catalog
         │ (Danh Mục Dữ Liệu)
```

### Use Case Điển Hình

```sql
-- Truy vấn kết hợp dữ liệu trong Redshift và dữ liệu trên S3
SELECT 
    r.customer_name,
    s.total_revenue
FROM 
    redshift_table r                               -- Dữ liệu trong cluster
    JOIN spectrum.s3_sales_archive s               -- Dữ liệu trên S3
      ON r.customer_id = s.customer_id
WHERE 
    s.sale_year >= 2020;
```

### Khi Nào Dùng Spectrum?

| Scenario | Dùng Spectrum? |
|----------|----------------|
| Dữ liệu lịch sử (> 1 năm) ít truy vấn | ✅ Tiết kiệm chi phí |
| Hot data (dữ liệu mới, truy vấn thường xuyên) | ❌ Dùng Redshift cluster |
| Data lake query (Data Lake — Hồ Dữ Liệu) ad-hoc | ✅ Không cần ETL |
| GDPR data tiering (phân tầng dữ liệu) | ✅ Archive xuống S3 |

---

## Concurrency Scaling

**Concurrency Scaling** (Tự Động Mở Rộng Đồng Thời) tự động thêm cluster capacity khi có nhiều query đồng thời.

```
Bình thường: 1 cluster chính xử lý tất cả query
↓ Khi có nhiều query đồng thời:
Concurrency Scaling cluster được thêm tự động (vài giây)
↓ Query được route đến cluster phụ
↓ Khi load giảm: cluster phụ tắt tự động
```

- **Billing:** 1 giờ credit miễn phí/ngày; sau đó tính per-second
- Phù hợp cho workload có **peak hours** (giờ cao điểm) rõ ràng

---

## Redshift Serverless

**Redshift Serverless** (Redshift Không Máy Chủ) tự động scale compute, không cần quản lý cluster.

| Tiêu Chí | Redshift Cluster | Redshift Serverless |
|----------|------------------|---------------------|
| **Quản lý** | Phải chọn node type, số nodes | Không cần quản lý |
| **Pricing** | Pay per node-hour | Pay per RPU (Redshift Processing Unit) |
| **Scale** | Manual hoặc Elastic Resize | Tự động |
| **Cold start** | Không (cluster luôn chạy) | Có thể có |
| **Use case** | Predictable, high workload | Intermittent, variable workload |

---

## Data Loading & Ingestion

### COPY Command (Lệnh Sao Chép) — Khuyên Dùng

```sql
-- Load từ S3 (nhanh nhất, parallel)
COPY sales
FROM 's3://my-bucket/sales/2024/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftS3Role'
FORMAT AS PARQUET;

-- Load từ CSV với delimiter
COPY customers
FROM 's3://my-bucket/customers.csv'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftS3Role'
CSV
DELIMITER ','
IGNOREHEADER 1;
```

- **Parallel loading:** Tự động chia file ra load song song trên các slices
- **Best practice:** Chia file thành nhiều file nhỏ (1 file/slice là tốt nhất)

### INSERT vs COPY

```
INSERT: Tuần tự, từng hàng → CỰC CHẬM cho bulk load
COPY:   Song song, batch → NHANH hơn INSERT hàng trăm lần

→ Luôn dùng COPY cho data loading, không dùng INSERT
```

### AWS Data Pipeline Tools

| Công Cụ | Mô Tả | Use Case |
|---------|-------|----------|
| **AWS Glue** | ETL (Extract, Transform, Load) serverless | Transform data trước khi load |
| **Kinesis Firehose** | Stream data real-time vào Redshift | Real-time ingestion |
| **DMS** | Migrate từ OLTP database | One-time hoặc ongoing replication |

---

## Performance Optimization

### VACUUM — Dọn Dẹp Không Gian

```sql
-- Sau khi DELETE/UPDATE nhiều, chạy VACUUM để reclaim space
VACUUM FULL sales;

-- Chỉ sort lại (không reclaim space)
VACUUM SORT ONLY sales;

-- Chỉ reclaim space (không sort lại)
VACUUM DELETE ONLY sales;
```

### ANALYZE — Cập Nhật Thống Kê

```sql
-- Cập nhật statistics để query optimizer đưa ra kế hoạch tốt hơn
ANALYZE sales;
ANALYZE sales(sale_date, region);  -- Chỉ phân tích cột cụ thể
```

### WLM — Workload Management (Quản Lý Tải Công Việc)

**WLM** phân loại query vào các queue (hàng đợi) khác nhau với memory/concurrency riêng.

```json
{
  "queue_type": "auto",
  "queues": [
    {
      "name": "Short queries",
      "query_execution_time": 0,     // 0-10 giây
      "memory_percent_to_use": 20,
      "max_execution_time": 10000
    },
    {
      "name": "Long queries",
      "memory_percent_to_use": 80
    }
  ]
}
```

### Materialized Views (View Đã Vật Liệu Hóa)

```sql
-- Tạo materialized view để pre-compute kết quả phức tạp
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT 
    DATE_TRUNC('day', sale_date) AS sale_day,
    SUM(revenue)                 AS total_revenue,
    COUNT(*)                     AS num_orders
FROM sales
GROUP BY 1;

-- Tự động refresh
REFRESH MATERIALIZED VIEW daily_revenue;
```

---

## Bảo Mật & Compliance

### Encryption (Mã Hóa)

- **At rest:** AES-256, key managed by KMS hoặc HSM (Hardware Security Module — Module Bảo Mật Phần Cứng)
- **In transit:** SSL/TLS bắt buộc

### VPC Integration

- Redshift cluster chạy trong VPC, isolated khỏi public internet
- Dùng **Enhanced VPC Routing** để buộc traffic qua VPC (không bypass qua public internet)

### Row-Level Security (Bảo Mật Cấp Hàng)

```sql
-- Tạo policy cho phép mỗi region chỉ xem data của mình
CREATE RLS POLICY region_isolation
WITH (region VARCHAR)
USING (region = current_setting('app.region'));

ATTACH RLS POLICY region_isolation ON sales TO ROLE analyst;
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích DISTKEY và cách chọn?

> **Trả lời:** DISTKEY (Distribution Key — Khóa Phân Phối) quyết định hàng nào được lưu trên slice nào. Chọn DISTKEY tốt giúp giảm **data shuffling** (xáo trộn dữ liệu) khi JOIN. Nguyên tắc: chọn cột JOIN thường xuyên nhất, có **high cardinality** (nhiều giá trị phân biệt), phân phối đều. Tránh timestamp hay boolean vì gây **data skew** (lệch dữ liệu).

### Q2: Redshift Spectrum khác gì Athena?

> **Trả lời:** Cả hai đều query S3, nhưng:
> - **Redshift Spectrum:** Dùng khi đã có Redshift cluster, muốn JOIN S3 data với Redshift data; compute cost tính thêm ngoài cluster cost
> - **Athena (Truy Vấn Serverless trên S3):** Hoàn toàn serverless, không cần cluster; tính phí theo lượng data quét; phù hợp cho ad-hoc analytics không cần Redshift

### Q3: Khi nào dùng Redshift thay vì RDS?

> **Trả lời:** Dùng Redshift khi:
> - **Workload là OLAP** (tổng hợp, báo cáo, phân tích lịch sử)
> - **Dữ liệu lớn** (hàng trăm GB đến petabyte)
> - **Query phức tạp** quét nhiều cột của bảng lớn
> - Dùng RDS khi workload là OLTP (đọc/ghi giao dịch từng row, low latency)

### Q4: Column-oriented storage giúp gì cho analytics?

> **Trả lời:** Trong analytics, thường chỉ cần vài cột từ bảng rất rộng. Column-oriented storage:
> 1. Đọc **chỉ các cột cần thiết**, tránh đọc cột không liên quan
> 2. **Nén tốt hơn** vì dữ liệu cùng type nằm cạnh nhau (ví dụ: tất cả số revenue)
> 3. **SIMD operations** (phép toán vector) hiệu quả hơn trên column data

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Phiên Bản:** 1.0
