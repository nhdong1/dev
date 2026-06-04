# Redshift Architecture — Kiến Trúc Amazon Redshift

> Hiểu sâu về kiến trúc MPP — Massively Parallel Processing (Xử Lý Song Song Đại Trà) của Redshift: Leader Node, Compute Nodes, Slices, columnar storage và cách query được thực thi phân tán.

---

## 📚 Mục Lục

1. [Kiến Trúc Tổng Thể](#1-kiến-trúc-tổng-thể)
2. [Leader Node — Nút Dẫn Đầu](#2-leader-node--nút-dẫn-đầu)
3. [Compute Nodes và Slices](#3-compute-nodes-và-slices)
4. [Columnar Storage — Lưu Trữ Theo Cột](#4-columnar-storage--lưu-trữ-theo-cột)
5. [Node Types — Loại Node](#5-node-types--loại-node)
6. [RA3 Nodes và Managed Storage](#6-ra3-nodes-và-managed-storage)
7. [Vòng Đời Thực Thi Query](#7-vòng-đời-thực-thi-query)
8. [Redshift Managed Storage — RMS](#8-redshift-managed-storage--rms)
9. [Cluster Networking và VPC](#9-cluster-networking-và-vpc)
10. [Backup và Disaster Recovery](#10-backup-và-disaster-recovery)
11. [Hands-on: Tạo Cluster và Query Đầu Tiên](#11-hands-on-tạo-cluster-và-query-đầu-tiên)

---

## 1. Kiến Trúc Tổng Thể

### Mô Hình MPP — Massively Parallel Processing

Redshift chia công việc ra nhiều node và thực thi song song. Đây là lý do vì sao Redshift có thể query hàng tỉ rows trong vài giây.

```
                    ┌───────────────────────────────┐
                    │          Client               │
                    │  (BI tool / SQL client / SDK) │
                    └──────────────┬────────────────┘
                                   │ SQL Query
                    ┌──────────────▼────────────────┐
                    │         Leader Node            │
                    │                                │
                    │  1. Parse SQL                  │
                    │  2. Build execution plan       │
                    │  3. Compile C++ code           │
                    │  4. Distribute to nodes        │
                    │  5. Aggregate results          │
                    └──────────────┬────────────────┘
                                   │ Compiled plans
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
┌─────────▼──────┐      ┌──────────▼─────┐      ┌──────────▼─────┐
│  Compute Node 1│      │  Compute Node 2│      │  Compute Node 3│
│                │      │                │      │                │
│  Slice 1       │      │  Slice 3       │      │  Slice 5       │
│  (CPU + Mem)   │      │  (CPU + Mem)   │      │  (CPU + Mem)   │
│  Data subset A │      │  Data subset C │      │  Data subset E │
│                │      │                │      │                │
│  Slice 2       │      │  Slice 4       │      │  Slice 6       │
│  (CPU + Mem)   │      │  (CPU + Mem)   │      │  (CPU + Mem)   │
│  Data subset B │      │  Data subset D │      │  Data subset F │
└────────────────┘      └────────────────┘      └────────────────┘
          │                        │                        │
          └────────────────────────┼────────────────────────┘
                                   │ Partial results
                    ┌──────────────▼────────────────┐
                    │         Leader Node            │
                    │   Merge + Return final result  │
                    └───────────────────────────────┘
```

---

## 2. Leader Node — Nút Dẫn Đầu

### Vai Trò Của Leader Node

Leader Node là điểm vào duy nhất cho tất cả kết nối client. Nó **không lưu trữ dữ liệu thực tế** mà chỉ điều phối.

| Nhiệm Vụ | Chi Tiết |
|----------|----------|
| **Query Parsing** | Phân tích cú pháp SQL, kiểm tra ngữ nghĩa |
| **Query Planning** | Tạo execution plan (kế hoạch thực thi) tối ưu |
| **Code Compilation** | Biên dịch plan thành mã C++ native để tăng tốc |
| **Task Distribution** | Phân phối compiled segments xuống Compute Nodes |
| **Result Aggregation** | Thu thập kết quả từ các node và trả về client |
| **Catalog Management** | Quản lý system tables và metadata |
| **Connection Handling** | Xử lý JDBC/ODBC connections từ client |

### Query Compilation — Biên Dịch Query

Redshift biên dịch query thành mã C++ native thay vì interpret SQL trực tiếp. Điều này cải thiện tốc độ đáng kể:

```
Lần đầu chạy query:
  SQL → Parse → Plan → Compile C++ → Execute
  (chậm hơn do compile, ~ vài trăm ms overhead)

Lần 2+ chạy query tương tự:
  SQL → Cache lookup → Execute
  (rất nhanh vì compiled code được cache)
```

---

## 3. Compute Nodes và Slices

### Compute Node

Mỗi Compute Node là một máy chủ riêng biệt với CPU, RAM và storage. Chúng thực hiện toàn bộ công việc tính toán và lưu trữ dữ liệu.

### Slice — Đơn Vị Xử Lý Song Song

Mỗi Compute Node được chia thành nhiều **Slice** (Lát Cắt — đơn vị xử lý song song). Mỗi Slice có:
- Một phần CPU
- Một phần RAM
- Một phần dữ liệu (data subset)

```
Ví dụ: Cluster với 3 Compute Nodes, mỗi node có 2 Slices = 6 Slices tổng

Bảng orders (100 triệu rows):
  Slice 1: rows 1 - 16.7M     (trên Node 1)
  Slice 2: rows 16.7M - 33.3M (trên Node 1)
  Slice 3: rows 33.3M - 50M   (trên Node 2)
  Slice 4: rows 50M - 66.7M   (trên Node 2)
  Slice 5: rows 66.7M - 83.3M (trên Node 3)
  Slice 6: rows 83.3M - 100M  (trên Node 3)

Query: SELECT SUM(amount) FROM orders
  → 6 Slices tính SUM song song → Leader Node cộng 6 partial sums
  → Nhanh hơn ~6x so với single-node
```

### Cách Dữ Liệu Được Phân Phối Cho Slices

Phân phối dữ liệu cho Slices phụ thuộc vào **DISTSTYLE** (Distribution Style — Kiểu Phân Phối) của bảng, được trình bày chi tiết trong [2-redshift-performance.md](./2-redshift-performance.md).

---

## 4. Columnar Storage — Lưu Trữ Theo Cột

### Tại Sao Columnar Storage Quan Trọng?

Redshift lưu dữ liệu theo **cột** thay vì theo **hàng** như các RDBMS (Relational Database Management System — Hệ Quản Trị Cơ Sở Dữ Liệu Quan Hệ) truyền thống.

```
Row-based Storage (Lưu Theo Hàng) — như MySQL, PostgreSQL:
  Block 1: [1, Alice, 1000, 2024-01-01] [2, Bob, 2000, 2024-01-02] [3, Carol, 1500, 2024-01-03]
  
  Query: SELECT SUM(amount) FROM orders
  → Phải đọc toàn bộ Block 1 (cả id, name, date) dù chỉ cần amount
  → I/O (Input/Output) lãng phí = chậm

Columnar Storage (Lưu Theo Cột) — như Redshift:
  Column id:     [1, 2, 3, ...]
  Column name:   [Alice, Bob, Carol, ...]
  Column amount: [1000, 2000, 1500, ...]   ← Chỉ đọc cột này
  Column date:   [2024-01-01, ...]
  
  Query: SELECT SUM(amount) FROM orders
  → Chỉ đọc Column amount → I/O tối thiểu → rất nhanh
```

### Lợi Ích Của Columnar Storage

| Lợi Ích | Giải Thích |
|---------|-----------|
| **Giảm I/O** | Chỉ đọc cột cần thiết, không đọc toàn bộ row |
| **Nén tốt hơn** | Cùng kiểu dữ liệu trong cột → nén hiệu quả hơn |
| **Cache hiệu quả** | Dữ liệu thường dùng được cache tốt hơn trong memory |
| **SIMD operations** | Vectorized processing (Xử Lý Vector) trên nhiều giá trị cùng lúc |

### Block Compression — Nén Khối

Redshift tự động áp dụng **encoding** (mã hóa nén) cho từng cột:

| Encoding | Phù Hợp Cho | Tỷ Lệ Nén |
|----------|-------------|-----------|
| **AZ64** | Numeric, DATE — mặc định cho hầu hết | Rất tốt |
| **ZSTD** | Text, VARCHAR — nén tốt nhất | Xuất sắc |
| **LZO** | VARCHAR, TEXT dài | Tốt |
| **Runlength** | Cột có nhiều giá trị lặp (ví dụ: status = 'active') | Tuyệt vời cho low cardinality |
| **Delta** | Timestamps, dates tăng dần đều | Tốt |
| **RAW** | Không nén — dùng khi giá trị rất khác nhau | Không nén |

```sql
-- Kiểm tra encoding đang dùng
SELECT tablename, "column", encoding, distkey, sortkey
FROM pg_table_def
WHERE tablename = 'orders';

-- Để Redshift tự chọn encoding tốt nhất (khuyến nghị)
COPY orders FROM 's3://...'
IAM_ROLE 'arn:aws:iam::...'
COMPUPDATE ON;  -- Tự động chọn encoding tối ưu
```

---

## 5. Node Types — Loại Node

### So Sánh Các Loại Node

| Node Type | vCPU | RAM | Storage | Tốt Nhất Cho |
|-----------|------|-----|---------|-------------|
| **ra3.xlplus** | 4 | 32 GB | 32 TB Managed | Workload nhỏ-vừa |
| **ra3.4xlarge** | 12 | 96 GB | 128 TB Managed | Workload vừa |
| **ra3.16xlarge** | 48 | 384 GB | 128 TB Managed | Workload lớn |
| **dc2.large** | 2 | 15 GB | 160 GB SSD | Dev/test, nhỏ |
| **dc2.8xlarge** | 32 | 244 GB | 2.56 TB SSD | Workload cũ |

### Khuyến Nghị Node Type

```
Workload hiện đại → Luôn dùng RA3 nodes:
  - Tách biệt storage và compute
  - Scale compute lên/xuống mà không mất dữ liệu
  - Managed Storage tự động tier sang S3

DC2 nodes (thế hệ cũ):
  - Vẫn dùng được nhưng không khuyến nghị cho project mới
  - Storage gắn liền với compute — scale bất tiện
```

---

## 6. RA3 Nodes và Managed Storage

### Kiến Trúc RA3 — Tách Storage Khỏi Compute

RA3 — Redshift Architecture 3 là thế hệ node mới nhất, cho phép **tách biệt storage và compute**:

```
Trước RA3 (DC2 nodes):
  Compute ←→ Storage (gắn liền)
  Muốn tăng storage = phải thêm node = tăng compute (lãng phí)

Với RA3 nodes:
  Compute Nodes (RA3) ←→ Redshift Managed Storage (RMS) ←→ Amazon S3
  
  Có thể scale:
  - Compute độc lập (thêm/bớt RA3 nodes)
  - Storage tự động tăng qua RMS → S3 (không giới hạn thực tế)
```

### Redshift Managed Storage — RMS

**RMS** (Redshift Managed Storage — Lưu Trữ Được Quản Lý Bởi Redshift) tự động quản lý dữ liệu:

```
Dữ liệu truy cập thường xuyên (hot):
  → Lưu trên NVMe SSD của RA3 node → truy cập nhanh

Dữ liệu ít truy cập (cold):
  → Tự động chuyển sang S3 → tiết kiệm chi phí

Điều phối tự động:
  → RMS theo dõi access pattern và di chuyển data thông minh
  → Người dùng không cần can thiệp
```

---

## 7. Vòng Đời Thực Thi Query

### Các Bước Xử Lý Query Đầy Đủ

```
Bước 1: Client gửi SQL query đến Leader Node qua JDBC/ODBC/API

Bước 2: Leader Node — Parsing (Phân Tích Cú Pháp)
  - Kiểm tra cú pháp SQL
  - Tạo AST — Abstract Syntax Tree (Cây Cú Pháp Trừu Tượng)

Bước 3: Leader Node — Query Optimization (Tối Ưu Hóa Query)
  - Tra cứu statistics (thống kê) từ system catalog
  - Chọn join order tối ưu (ưu tiên bảng nhỏ)
  - Quyết định redistribution strategy (chiến lược phân phối lại)
  - Tạo query plan (kế hoạch query) dạng cây

Bước 4: Leader Node — Compilation (Biên Dịch)
  - Biên dịch query plan thành C++ code
  - Compile C++ thành native machine code
  - Cache compiled code để reuse

Bước 5: Leader Node — Distribution (Phân Phối)
  - Chia query thành segments (phân đoạn)
  - Gửi compiled segments xuống từng Compute Node

Bước 6: Compute Nodes — Parallel Execution (Thực Thi Song Song)
  - Mỗi Slice thực thi segment trên data subset của nó
  - Đọc dữ liệu từ local storage (hoặc RMS/S3 cho RA3)
  - Thực hiện filter, aggregation, local join

Bước 7: Data Redistribution (Nếu Cần)
  - Khi JOIN yêu cầu data từ nhiều Slices khác nhau
  - Các Slices gửi rows cho nhau qua network
  - (Đây là bước tốn kém nhất — cần tối ưu với DISTKEY)

Bước 8: Compute Nodes gửi partial results về Leader Node

Bước 9: Leader Node — Final Aggregation
  - Merge các partial results
  - Áp dụng ORDER BY, LIMIT cuối cùng
  - Trả kết quả về client
```

### Query Plan — Đọc EXPLAIN Output

```sql
-- Xem query plan trước khi thực thi
EXPLAIN
SELECT c.customer_name, SUM(o.amount) AS total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.customer_name
ORDER BY total DESC
LIMIT 10;
```

```
Kết quả EXPLAIN (đọc từ dưới lên):
  XN Limit  (tối đa 10 rows)
    → XN Merge (merge sorted results từ nodes)
      → XN Network (gửi data về Leader Node)
        → XN Sort (sắp xếp theo total DESC)
          → XN HashAggregate (tính SUM và GROUP BY)
            → XN Hash Join  (DS_BCAST_INNER — broadcast bảng nhỏ)
                → XN Seq Scan on orders (filter order_date)
                → XN Seq Scan on customers

Chú ý:
  DS_BCAST_INNER = broadcast join → bảng customers nhỏ, broadcast đến tất cả nodes
  DS_DIST_BOTH   = cả hai bảng đều redistribute → tốn kém, cần optimize DISTKEY
  DS_DIST_NONE   = không cần redistribute → tốt nhất
```

---

## 8. Redshift Managed Storage — RMS

### Cơ Chế Hoạt Động

```
RA3 Node Local NVMe SSD (hot tier):
  ├── Block A (accessed 5 minutes ago)
  ├── Block B (accessed 1 hour ago)
  └── Block C (accessed 2 hours ago)

Amazon S3 — Redshift Managed Storage (cold tier):
  ├── Block D (accessed 3 days ago)
  ├── Block E (accessed 1 week ago)
  └── Block F (accessed 1 month ago)

Access Pattern:
  Query cần Block D → RMS tải từ S3 vào SSD → serve query
  → Block D bây giờ là "hot", một block "cold" hơn bị đẩy sang S3
```

### Multi-AZ Deployment — Triển Khai Đa Vùng Khả Dụng

Từ 2022, Redshift RA3 hỗ trợ **Multi-AZ deployment** (Triển Khai Đa Vùng Khả Dụng):

```
AZ-1 (Availability Zone 1 — Vùng Khả Dụng 1):
  Leader Node + Compute Nodes + Local Cache

AZ-2 (Availability Zone 2 — Vùng Khả Dụng 2):
  Leader Node (standby) + Compute Nodes + Local Cache

Shared:
  Redshift Managed Storage (S3-backed) — cả hai AZ đều truy cập được

→ Failover tự động khi AZ-1 gặp sự cố, chuyển sang AZ-2
→ RPO (Recovery Point Objective — Mục Tiêu Điểm Khôi Phục) = 0
→ RTO (Recovery Time Objective — Mục Tiêu Thời Gian Khôi Phục) < vài phút
```

---

## 9. Cluster Networking và VPC

### Kiến Trúc Mạng

```
┌─────────────────────────────────────────────────────────────────┐
│                        VPC của bạn                               │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │              Private Subnet (Subnet Riêng Tư)           │     │
│  │                                                         │     │
│  │  ┌───────────────────────────────────────────────────┐  │     │
│  │  │           Redshift Cluster                         │  │     │
│  │  │   Leader Node IP: 10.0.1.10                       │  │     │
│  │  │   Port: 5439 (PostgreSQL-compatible)              │  │     │
│  │  └───────────────────────────────────────────────────┘  │     │
│  │                                                         │     │
│  │  Security Group (Nhóm Bảo Mật):                        │     │
│  │    - Inbound: Port 5439 từ BI tools subnet             │     │
│  │    - Inbound: Port 5439 từ Glue ETL subnet             │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐      │
│  │  S3 VPC Endpoint (Điểm Cuối VPC S3)                    │      │
│  │  → Redshift truy cập S3 qua private network            │      │
│  │  → Không qua internet → bảo mật và nhanh hơn          │      │
│  └────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### Endpoint Redshift

```
Redshift JDBC connection string (chuỗi kết nối JDBC):
  jdbc:redshift://<cluster-id>.<account>.<region>.redshift.amazonaws.com:5439/<database>

Ví dụ:
  jdbc:redshift://my-cluster.abc123.ap-southeast-1.redshift.amazonaws.com:5439/analytics

Kết nối từ AWS Glue:
  - Dùng Glue JDBC Connection với Security Group cho phép port 5439
  - Hoặc dùng COPY command từ Glue qua S3 (khuyến nghị — nhanh hơn)
```

---

## 10. Backup và Disaster Recovery

### Automated Snapshots — Snapshot Tự Động

```
Redshift tự động tạo snapshot mỗi 8 giờ (mặc định):
  - Lưu trên S3 (không tính vào storage của cluster)
  - Giữ 1 ngày (mặc định, có thể tăng lên 35 ngày)
  - Incremental (chỉ lưu thay đổi, không sao chép toàn bộ)

Retention: 1-35 ngày cho automated snapshots
```

### Manual Snapshots — Snapshot Thủ Công

```sql
-- Tạo snapshot thủ công (giữ mãi cho đến khi xóa)
aws redshift create-cluster-snapshot \
  --cluster-identifier my-cluster \
  --snapshot-identifier my-snapshot-before-migration

-- Khôi phục từ snapshot
aws redshift restore-from-cluster-snapshot \
  --cluster-identifier my-restored-cluster \
  --snapshot-identifier my-snapshot-before-migration
```

### Cross-Region Snapshot Copy — Sao Chép Snapshot Giữa Region

```
Bật Cross-Region Snapshot Copy để Disaster Recovery (Khôi Phục Sau Thảm Họa):

Region chính (ap-southeast-1):
  Cluster → Auto Snapshot → S3 (ap-southeast-1)
                          ↓ Sao chép tự động
Region phụ (ap-northeast-1):
                          S3 (ap-northeast-1)
                          → Có thể restore nếu region chính bị sự cố
```

---

## 11. Hands-on: Tạo Cluster và Query Đầu Tiên

### Tạo Cluster Với AWS CLI

```bash
# Tạo subnet group (nhóm subnet) cho Redshift
aws redshift create-cluster-subnet-group \
  --cluster-subnet-group-name my-redshift-subnet-group \
  --description "Subnet group for Redshift cluster" \
  --subnet-ids subnet-xxxxxxxx subnet-yyyyyyyy

# Tạo Redshift cluster RA3
aws redshift create-cluster \
  --cluster-identifier my-analytics-cluster \
  --node-type ra3.xlplus \
  --number-of-nodes 2 \
  --master-username admin \
  --master-user-password "MySecurePass123!" \
  --db-name analytics \
  --cluster-subnet-group-name my-redshift-subnet-group \
  --vpc-security-group-ids sg-xxxxxxxx \
  --no-publicly-accessible \
  --automated-snapshot-retention-period 7 \
  --region ap-southeast-1
```

### Kết Nối và Tạo Bảng Đầu Tiên

```sql
-- Kết nối qua Redshift Query Editor v2 hoặc psql
-- psql -h <endpoint> -U admin -d analytics -p 5439

-- Tạo schema
CREATE SCHEMA IF NOT EXISTS sales;

-- Tạo bảng với DISTKEY và SORTKEY phù hợp
CREATE TABLE sales.orders (
    order_id     BIGINT         NOT NULL,
    customer_id  BIGINT         NOT NULL,
    product_id   INTEGER        NOT NULL,
    amount       DECIMAL(12, 2) NOT NULL,
    status       VARCHAR(20)    NOT NULL,
    region       VARCHAR(50),
    order_date   DATE           NOT NULL,
    created_at   TIMESTAMP      DEFAULT CURRENT_TIMESTAMP
)
DISTKEY (customer_id)   -- Phân phối dữ liệu theo customer_id
SORTKEY (order_date);   -- Sắp xếp dữ liệu theo ngày để query range nhanh

-- Load dữ liệu từ S3 với COPY command (nhanh nhất)
COPY sales.orders
FROM 's3://my-datalake/orders/year=2024/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftS3Role'
FORMAT AS PARQUET;

-- Chạy ANALYZE để cập nhật thống kê cho query optimizer
ANALYZE sales.orders;

-- Query đầu tiên
SELECT
    region,
    COUNT(*) AS order_count,
    SUM(amount) AS total_revenue,
    ROUND(AVG(amount), 2) AS avg_order_value
FROM sales.orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
  AND status = 'COMPLETED'
GROUP BY region
ORDER BY total_revenue DESC;
```

### Kiểm Tra Sức Khỏe Cluster

```sql
-- Xem thông tin về slices và nodes
SELECT node, slice, col, num_values, minvalue, maxvalue
FROM svv_diskusage
WHERE name = 'orders'
ORDER BY node, slice;

-- Kiểm tra phân phối dữ liệu trên các slices
SELECT slice, COUNT(*) AS row_count
FROM stv_slices
GROUP BY slice
ORDER BY slice;

-- Xem query đang chạy
SELECT pid, user_name, starttime, query
FROM stv_recents
WHERE status = 'Running';

-- Xem lịch sử query gần đây
SELECT query, starttime, endtime,
       DATEDIFF(seconds, starttime, endtime) AS duration_sec,
       aborted
FROM svl_qlog
ORDER BY starttime DESC
LIMIT 20;
```

---

## 🔑 Tóm Tắt Key Points

```
1. MPP = Phân tán query ra nhiều Compute Nodes & Slices → chạy song song → nhanh
2. Leader Node = điều phối, không lưu data; Compute Nodes = lưu data và tính toán
3. Columnar Storage = đọc chỉ cột cần → I/O tối thiểu → query analytics nhanh
4. RA3 nodes = tách storage (RMS/S3) khỏi compute → scale linh hoạt hơn DC2
5. COPY from S3 = cách load data nhanh nhất vào Redshift (song song, multi-node)
6. Query compile thành C++ native → lần đầu chậm, lần sau nhanh (cache)
7. Multi-AZ deployment với RA3 = HA (High Availability — Tính Sẵn Sàng Cao) cho production
8. Automated snapshots mỗi 8 giờ, giữ 1-35 ngày → backup tự động
```

---

**Tiếp Theo:** [2-redshift-performance.md](./2-redshift-performance.md) — DISTKEY, SORTKEY, DISTSTYLE và WLM — các kỹ thuật tối ưu hiệu suất Redshift
