# 1 — Engine Types — Các Loại Engine RDS

> RDS hỗ trợ 6 engine cơ sở dữ liệu quan hệ phổ biến. Lựa chọn engine phù hợp ảnh hưởng trực tiếp đến hiệu năng, chi phí, và khả năng tương thích với ứng dụng của bạn.

## 📚 Mục Lục

1. [Tổng Quan 6 Engine](#tổng-quan-6-engine)
2. [MySQL on RDS](#mysql-on-rds)
3. [PostgreSQL on RDS](#postgresql-on-rds)
4. [MariaDB on RDS](#mariadb-on-rds)
5. [Oracle on RDS](#oracle-on-rds)
6. [SQL Server on RDS](#sql-server-on-rds)
7. [So Sánh Engine](#so-sánh-engine)
8. [Migration Paths — Lộ Trình Di Chuyển](#migration-paths--lộ-trình-di-chuyển)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan 6 Engine

| Engine         | License (Giấy Phép) | Phiên Bản (2026)     | Multi-AZ | Read Replica | Aurora-compatible |
| -------------- | ------------------- | -------------------- | -------- | ------------ | ----------------- |
| **MySQL**      | GPL (Mã Nguồn Mở)   | 8.0, 8.4, 5.7        | ✅        | ✅ (tối đa 5) | ✅ MySQL-compatible |
| **PostgreSQL** | PostgreSQL (Mã Nguồn Mở) | 16, 15, 14, 13  | ✅        | ✅ (tối đa 5) | ✅ PostgreSQL-compatible |
| **MariaDB**    | GPL (Mã Nguồn Mở)   | 10.11, 10.6, 10.5    | ✅        | ✅ (tối đa 5) | ❌                 |
| **Oracle**     | Commercial (Thương Mại) | 19c, 21c         | ✅        | ✅            | ❌                 |
| **SQL Server** | Commercial (Thương Mại) | 2019, 2022, 2017 | ✅        | ✅            | ❌                 |
| **Db2**        | Commercial (Thương Mại) | 11.5             | ✅        | ✅            | ❌                 |

---

## MySQL on RDS

### Tổng Quan

MySQL là engine phổ biến nhất trên RDS — phù hợp với hầu hết web applications, CMS (Content Management System — Hệ Thống Quản Lý Nội Dung), và e-commerce.

### Phiên Bản Được Hỗ Trợ

- **MySQL 8.4** (LTS — Long Term Support — Hỗ Trợ Dài Hạn): Khuyến nghị cho dự án mới
- **MySQL 8.0**: Phổ biến nhất, nhiều tính năng mới
- **MySQL 5.7**: Legacy, sắp kết thúc hỗ trợ — nên nâng cấp

### Tính Năng Nổi Bật trên RDS

- **InnoDB** là storage engine mặc định — hỗ trợ ACID transactions đầy đủ
- **Automated backups** (Sao Lưu Tự Động) và PITR (Point-in-Time Recovery — Khôi Phục Theo Thời Điểm)
- **Read Replicas** trong cùng region và cross-region (Xuyên Vùng)
- **Performance Schema** (Lược Đồ Hiệu Năng) — công cụ profiling query
- **Binary log** (Nhật Ký Nhị Phân) — dùng cho replication và CDC (Change Data Capture — Thu Thập Thay Đổi Dữ Liệu)

### Hạn Chế trên RDS

- Không có quyền truy cập `SUPER` privilege
- Không thể chỉnh sửa một số tham số hệ thống sâu
- Không hỗ trợ `LOAD DATA INFILE` từ local file

### Cấu Hình Quan Trọng (Parameter Groups)

```ini
# Các tham số quan trọng cho MySQL
innodb_buffer_pool_size = 75% của RAM    # Cache dữ liệu và index trong RAM
max_connections = 500                    # Số kết nối tối đa đồng thời
slow_query_log = 1                       # Bật slow query log
long_query_time = 1                      # Log query chậm hơn 1 giây
innodb_flush_log_at_trx_commit = 1       # Đảm bảo durability (Tính Bền Vững)
```

---

## PostgreSQL on RDS

### Tổng Quan

PostgreSQL là engine mạnh nhất về tính năng SQL nâng cao — phù hợp với applications cần JSONB (JSON Binary — JSON Nhị Phân), full-text search (Tìm Kiếm Toàn Văn), và các kiểu dữ liệu đặc biệt.

### Phiên Bản Được Hỗ Trợ

- **PostgreSQL 16**: Phiên bản mới nhất với nhiều cải tiến hiệu năng
- **PostgreSQL 15**: Ổn định, phổ biến trong production
- **PostgreSQL 14/13/12**: Các phiên bản cũ hơn vẫn được hỗ trợ

### Tính Năng Nổi Bật

- **JSONB** — Lưu và query JSON hiệu quả với index
- **Full-text search** — Tích hợp sẵn không cần Elasticsearch cho use cases đơn giản
- **Extensions (Phần Mở Rộng)**: `pg_stat_statements`, `PostGIS` (GIS — Geographic Information System — Hệ Thống Thông Tin Địa Lý), `uuid-ossp`
- **Logical replication** (Sao Chép Logic) — cho phép streaming replication chi tiết hơn
- **Table partitioning** (Phân Vùng Bảng) — native support từ PG 10+
- **Window functions** (Hàm Cửa Sổ) và CTEs (Common Table Expressions — Biểu Thức Bảng Chung) nâng cao

### Extensions Được Hỗ Trợ trên RDS

RDS PostgreSQL hỗ trợ hơn 85 extensions. Quan trọng nhất:

| Extension               | Mục Đích                                            |
| ----------------------- | --------------------------------------------------- |
| `pg_stat_statements`    | Theo dõi thống kê query — cần thiết cho tuning      |
| `PostGIS`               | Dữ liệu địa lý (geospatial)                         |
| `pgvector`              | Vector embeddings — cho AI/ML workloads             |
| `pg_partman`            | Quản lý partition tự động                           |
| `uuid-ossp`             | Tạo UUID (Universally Unique Identifier — Định Danh Duy Nhất Toàn Cầu) |

### Cấu Hình Quan Trọng

```ini
shared_buffers = 25% của RAM            # Buffer cache chính của PostgreSQL
work_mem = 64MB                          # RAM cho mỗi sort/hash operation
max_connections = 200                    # Thấp hơn MySQL vì mỗi connection tốn nhiều RAM hơn
effective_cache_size = 75% của RAM       # Gợi ý cho query planner
checkpoint_completion_target = 0.9       # Trải đều I/O khi checkpoint
```

---

## MariaDB on RDS

### Tổng Quan

MariaDB là fork (Phân Nhánh) của MySQL, được tạo bởi người sáng lập MySQL sau khi Oracle mua lại. Tương thích gần như hoàn toàn với MySQL.

### Khi Nào Dùng MariaDB Thay MySQL

- Cần tính năng mà MariaDB có sẵn nhưng MySQL chưa có (ví dụ: `SEQUENCE`, columnar storage với `Aria` engine)
- Policy công ty yêu cầu dùng community/open-source hoàn toàn
- Đã có kinh nghiệm với MariaDB trên on-premises

### Điểm Khác Biệt Với MySQL

- **Spider storage engine** — sharding ngang tích hợp sẵn
- **Galera Cluster** — multi-master replication (Sao Chép Đa Chính)
- **Virtual columns** (Cột Ảo) — computed columns không cần trigger
- **Progress reporting** — hiển thị tiến độ cho các tác vụ dài

> **Lưu ý:** MariaDB KHÔNG tương thích với Aurora — nếu cần scale lên Aurora sau này, dùng MySQL thay thế.

---

## Oracle on RDS

### Tổng Quan

Oracle on RDS dành cho enterprise applications cũ (legacy) có dependency với Oracle-specific features — trong khi vẫn muốn AWS quản lý infrastructure.

### License Models (Mô Hình Cấp Phép)

RDS Oracle có 2 mô hình:

| Mô Hình                                   | Mô Tả                                    | Chi Phí     |
| ----------------------------------------- | ---------------------------------------- | ----------- |
| **License Included (LI)**                 | AWS cung cấp license trong giá RDS       | Cao hơn     |
| **BYOL (Bring Your Own License — Mang Giấy Phép Riêng)** | Dùng license Oracle hiện có của bạn | Thấp hơn nếu đã có license |

### Editions (Phiên Bản)

- **Oracle Standard Edition Two (SE2)**: Tính năng cơ bản, giới hạn 2 socket
- **Oracle Enterprise Edition (EE)**: Đầy đủ tính năng, dùng cho workloads phức tạp

### Hạn Chế Quan Trọng

- Không hỗ trợ tất cả Oracle features (ví dụ: Oracle RAC — Real Application Clusters không được hỗ trợ)
- Không thể truy cập OS level
- Một số packages Oracle không khả dụng

### Khi Nào Nên Xem Xét Migration ra khỏi Oracle

Oracle trên RDS thường là giải pháp tạm thời — long-term nên xem xét:
- **PostgreSQL** — tương thích nhiều tính năng Oracle, miễn phí
- **Aurora PostgreSQL** — với AWS SCT (Schema Conversion Tool — Công Cụ Chuyển Đổi Schema)

---

## SQL Server on RDS

### Tổng Quan

SQL Server trên RDS dành cho ứng dụng trong hệ sinh thái Microsoft .NET/Azure đã có — chạy trên AWS infrastructure.

### Editions Được Hỗ Trợ

| Edition                    | Giới Hạn                               | Chi Phí     |
| -------------------------- | -------------------------------------- | ----------- |
| **Express Edition**        | 10 GB DB, 1 GB RAM                     | Rất thấp    |
| **Web Edition**            | Dành cho web apps, không SSAS/SSRS     | Thấp        |
| **Standard Edition (SE)**  | Tính năng cơ bản đầy đủ               | Trung bình  |
| **Enterprise Edition (EE)**| Đầy đủ tính năng, HA nâng cao         | Cao         |

### Tính Năng Nổi Bật

- **Always On** — Multi-AZ built trên SQL Server Always On Availability Groups
- **SQL Server Agent** — job scheduling (lập lịch tác vụ)
- **SSRS (SQL Server Reporting Services — Dịch Vụ Báo Cáo)**: Hỗ trợ trên một số editions
- **Transparent Data Encryption (TDE — Mã Hóa Dữ Liệu Trong Suốt)**: Thông qua Option Groups

### Cấu Hình Đặc Biệt

```sql
-- Kiểm tra collation (Đối Chiếu Ký Tự) — quan trọng với tiếng Việt
SELECT SERVERPROPERTY('Collation');
-- Khuyến nghị: SQL_Latin1_General_CP1_CI_AS hoặc Vietnamese_CI_AS
```

---

## So Sánh Engine

### Ma Trận Tính Năng

| Tính Năng                          | MySQL | PostgreSQL | MariaDB | Oracle | SQL Server |
| ---------------------------------- | ----- | ---------- | ------- | ------ | ---------- |
| ACID Transactions                  | ✅     | ✅          | ✅       | ✅      | ✅          |
| JSON/JSONB Support                 | ✅     | ✅✅        | ✅       | ✅      | ✅          |
| Full-text Search                   | ✅     | ✅✅        | ✅       | ✅      | ✅          |
| Partitioning Native                | ✅     | ✅          | ✅       | ✅      | ✅          |
| Window Functions                   | ✅     | ✅          | ✅       | ✅      | ✅          |
| Stored Procedures                  | ✅     | ✅          | ✅       | ✅✅    | ✅          |
| Aurora-compatible                  | ✅     | ✅          | ❌       | ❌      | ❌          |
| Open Source (Mã Nguồn Mở)         | ✅     | ✅          | ✅       | ❌      | ❌          |
| Geospatial (GIS)                   | ✅     | ✅✅        | ✅       | ✅      | ✅          |
| Table Inheritance                  | ❌     | ✅          | ❌       | ❌      | ❌          |

### Hiệu Năng Tương Đối (OLTP Workloads)

```
MySQL 8.0:      ████████░░  Tốt — cân bằng giữa tốc độ và tính năng
PostgreSQL 16:  ███████░░░  Tốt — chậm hơn MySQL trong simple queries
                             nhưng nhanh hơn trong complex queries
MariaDB 10.11:  ████████░░  Tương đương MySQL
Oracle 19c:     ████████░░  Tốt với enterprise features
SQL Server:     ███████░░░  Tốt trong hệ sinh thái Microsoft
```

### Khuyến Nghị Lựa Chọn

```
┌─────────────────────────────────────────────────────────────┐
│ Câu hỏi chọn engine:                                        │
│                                                             │
│ 1. Đang migrate từ on-premises?                             │
│    → Dùng cùng engine để tránh rủi ro                      │
│                                                             │
│ 2. Dự án mới, web/mobile app?                               │
│    → MySQL 8.0 hoặc PostgreSQL 15/16                        │
│                                                             │
│ 3. Cần JSONB, GIS, vector search?                          │
│    → PostgreSQL                                             │
│                                                             │
│ 4. Cần scale lên Aurora sau?                               │
│    → MySQL hoặc PostgreSQL (Aurora-compatible)              │
│                                                             │
│ 5. Enterprise, Oracle-specific features?                    │
│    → Oracle (và lên kế hoạch migration dài hạn)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Migration Paths — Lộ Trình Di Chuyển

### Homogeneous Migration (Di Chuyển Cùng Loại)

Ví dụ: MySQL on-premises → MySQL on RDS

```
Phương pháp:
1. mysqldump / pg_dump — cho databases nhỏ
2. AWS DMS (Database Migration Service) — cho databases lớn
3. Replication-based — zero-downtime migration
```

### Heterogeneous Migration (Di Chuyển Khác Loại)

Ví dụ: Oracle → PostgreSQL

```
Phương pháp:
1. AWS SCT (Schema Conversion Tool) — chuyển đổi schema tự động
2. AWS DMS — migrate data
3. Application code changes — thay stored procedures, functions
4. Testing — kiểm tra kỹ ứng dụng sau migration
```

### Oracle → PostgreSQL Migration Path

Oracle và PostgreSQL có nhiều điểm tương đồng nhưng khác về syntax:

| Oracle Syntax                | PostgreSQL Equivalent              |
| ---------------------------- | ---------------------------------- |
| `VARCHAR2(n)`                | `VARCHAR(n)` hoặc `TEXT`           |
| `NUMBER(p,s)`                | `NUMERIC(p,s)` hoặc `DECIMAL(p,s)` |
| `SYSDATE`                    | `NOW()` hoặc `CURRENT_TIMESTAMP`   |
| `NVL(x, y)`                  | `COALESCE(x, y)`                   |
| `DECODE(expr, ...)`          | `CASE WHEN ... THEN ... END`       |
| `ROWNUM`                     | `LIMIT n` hoặc `ROW_NUMBER() OVER` |
| `CONNECT BY` (Hierarchical)  | Recursive CTE (`WITH RECURSIVE`)   |

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao bạn chọn PostgreSQL thay vì MySQL?**

PostgreSQL phù hợp khi:
- Cần JSONB với index hiệu năng cao
- Cần extensions đặc biệt (PostGIS, pgvector)
- Query phức tạp với subqueries, CTEs, window functions
- Cần table inheritance hoặc custom types

MySQL phù hợp khi:
- Read-heavy workload đơn giản
- Team quen MySQL hơn
- Cần Aurora-compatible với MySQL syntax

**Q: Khi nào nên migrate từ Oracle sang PostgreSQL?**

- Chi phí license Oracle quá cao (đặc biệt với BYOL)
- Muốn tận dụng open-source ecosystem
- Chức năng Oracle dùng không quá phức tạp (không dùng RAC, Exadata-specific features)
- Có thời gian và nguồn lực để test kỹ

**Q: SQL Server trên RDS có hỗ trợ Always On không?**

Có — RDS SQL Server Multi-AZ được xây dựng trên SQL Server Always On Availability Groups nhưng được AWS quản lý hoàn toàn. Bạn không cần cấu hình thủ công.

---

## 🔗 Điều Hướng

- **Trước:** [README.md](README.md) — Tổng Quan RDS
- **Tiếp theo:** [2-instance-storage.md](2-instance-storage.md) — Instance Classes & Storage
- **Liên quan:** [../08-migration/README.md](../08-migration/README.md) — Database Migration

---

**Cập Nhật Lần Cuối:** 2026-05-15
