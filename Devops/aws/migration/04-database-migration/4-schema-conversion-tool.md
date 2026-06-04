# AWS SCT — Schema Conversion Tool: Chuyển Đổi Schema Tự Động

> **AWS SCT** — Schema Conversion Tool (Công Cụ Chuyển Đổi Schema) — là ứng dụng desktop miễn phí giúp tự động chuyển đổi schema cơ sở dữ liệu từ một engine sang engine khác. SCT phân tích schema nguồn, tạo báo cáo đánh giá mức độ phức tạp, và sinh ra SQL scripts tương đương cho target DB. SCT là công cụ bắt buộc trong bất kỳ heterogeneous migration nào.

## 📚 Mục Lục

1. [SCT Là Gì? Khi Nào Dùng?](#sct-là-gì)
2. [Cài Đặt Và Thiết Lập](#cài-đặt-và-thiết-lập)
3. [Assessment Report — Báo Cáo Đánh Giá](#assessment-report)
4. [Quy Trình Chuyển Đổi Schema](#quy-trình-chuyển-đổi)
5. [Xử Lý Action Items](#xử-lý-action-items)
6. [Extension Pack — Gói Mở Rộng](#extension-pack)
7. [Các Nguồn Và Đích Được Hỗ Trợ](#các-nguồn-và-đích-được-hỗ-trợ)
8. [SCT Vs. Chuyển Đổi Thủ Công](#sct-vs-thủ-công)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 SCT Là Gì? Khi Nào Dùng?

### Chức Năng Của SCT

```
SCT thực hiện các nhiệm vụ sau:

1. Phân tích schema nguồn:
   ├── Kết nối vào source DB (Oracle, SQL Server, DB2...)
   ├── Extract toàn bộ metadata: tables, views, procedures, functions...
   └── Xây dựng dependency graph giữa các objects

2. Tạo Assessment Report:
   ├── Ước tính % objects có thể tự động convert
   ├── Liệt kê action items cần sửa thủ công
   └── Ước tính effort (số giờ/ngày cần thiết)

3. Tự động chuyển đổi schema:
   ├── Convert DDL (CREATE TABLE, CREATE INDEX...)
   ├── Convert stored procedures, functions, triggers
   └── Tạo SQL script cho target DB

4. Áp dụng schema lên target:
   ├── Apply converted schema trực tiếp lên target DB
   └── Hoặc export ra file .sql để review trước
```

### SCT Vs. DMS: Phân Biệt Rõ Ràng

```
SCT (Schema Conversion Tool):
├── Xử lý: SCHEMA (cấu trúc) — tables, views, procedures
├── Cài đặt: Desktop application trên máy tính của bạn
├── Output: SQL scripts
├── Phí: MIỄN PHÍ (tải xuống từ aws.amazon.com)
└── Khi dùng: Heterogeneous migration (khác engine)

DMS (Database Migration Service):
├── Xử lý: DATA (dữ liệu thực tế trong tables)
├── Cài đặt: Cloud service (không cài gì)
├── Output: Data trong target DB
├── Phí: Theo giờ (replication instance)
└── Khi dùng: Mọi loại DB migration

Quy tắc nhớ:
SCT = Kiến trúc sư vẽ bản thiết kế tòa nhà mới
DMS = Xe tải chở đồ đạc từ nhà cũ sang nhà mới
```

### Khi Nào KHÔNG Cần SCT?

```
Không cần SCT khi:
├── Homogeneous migration (cùng engine):
│   MySQL → RDS MySQL        (không cần)
│   PostgreSQL → Aurora PG   (không cần)
│   SQL Server → RDS MSSQL  (không cần)
└── Lý do: schema tương thích, DMS có thể đọc trực tiếp

Bắt buộc dùng SCT khi:
├── Oracle → Aurora PostgreSQL / Aurora MySQL
├── SQL Server → Aurora PostgreSQL / Aurora MySQL
├── DB2 → RDS PostgreSQL
└── Bất kỳ heterogeneous migration nào
```

---

## 💿 Cài Đặt Và Thiết Lập

### Yêu Cầu Hệ Thống

```
SCT chạy trên:
├── Windows: Windows 8.1 / 10 / Server 2012 trở lên
├── macOS: 10.14 (Mojave) trở lên
└── Linux: Ubuntu 16.04+, RHEL 7+

Yêu cầu:
├── Java: JDK 11 trở lên (Oracle JDK hoặc OpenJDK)
├── RAM: ít nhất 4 GB (khuyến nghị 8 GB+)
└── Disk: 5 GB+ cho SCT, drivers và temp files
```

### Cài Đặt

```bash
# 1. Tải SCT từ AWS:
# https://aws.amazon.com/dms/schema-conversion-tool/
# → Chọn hệ điều hành → Download

# 2. Tải JDBC drivers cho source và target DB:
# Oracle JDBC: ojdbc8.jar (từ Oracle website)
# PostgreSQL JDBC: postgresql-42.x.x.jar (từ jdbc.postgresql.org)
# SQL Server JDBC: mssql-jdbc-xxx.jar (từ Microsoft)
# MySQL JDBC: mysql-connector-java-xxx.jar

# 3. Cấu hình drivers trong SCT:
# Settings → Global Settings → Drivers
# → Chỉ định đường dẫn đến từng JDBC driver

# 4. (Tùy chọn) Cấu hình AWS credentials:
# Settings → AWS Service Profiles → Add
# → Access Key ID, Secret Access Key, Region
```

### Tạo Migration Project

```
Trong SCT:
1. File → New Migration Project
2. Đặt tên project (ví dụ: "oracle-to-aurora-retail")
3. Chọn Source:
   ├── Database engine: Oracle
   ├── Connection: hostname, port, SID/service name
   ├── Credentials: username/password
   └── SSL settings nếu cần
4. Chọn Target:
   ├── Database engine: Amazon Aurora PostgreSQL
   ├── Connection: Aurora endpoint
   └── Credentials
5. Click OK → SCT kết nối và load metadata
```

---

## 📊 Assessment Report — Báo Cáo Đánh Giá

### Tạo Assessment Report

```
Trong SCT:
1. Chọn schema trong Source Database panel
2. Right-click → Create Report
3. SCT phân tích tất cả objects (có thể mất 1-10 phút)
4. Report xuất hiện trong Report tab

Hoặc save ra file:
File → Save Report As → HTML hoặc PDF
→ Dùng để chia sẻ với team, stakeholders
```

### Đọc Assessment Report

```
Cấu trúc Assessment Report:

1. Executive Summary (Tóm Tắt Điều Hành):
   ├── Total objects: 1,234 objects được phân tích
   ├── Automatically converted: 85% (1,049 objects)
   ├── Manual action required: 15% (185 objects)
   └── Estimated effort: 120 person-hours

2. Summary by Category:
   ┌──────────────────┬──────────┬──────────┬──────────────┐
   │ Object Type      │ Total    │ Auto     │ Manual       │
   ├──────────────────┼──────────┼──────────┼──────────────┤
   │ Tables           │ 324      │ 323 (99%)│ 1 (1%)       │
   │ Views            │ 45       │ 40 (89%) │ 5 (11%)      │
   │ Stored Procs     │ 78       │ 45 (58%) │ 33 (42%)     │
   │ Functions        │ 34       │ 22 (65%) │ 12 (35%)     │
   │ Triggers         │ 56       │ 38 (68%) │ 18 (32%)     │
   │ Sequences        │ 28       │ 28 (100%)│ 0            │
   └──────────────────┴──────────┴──────────┴──────────────┘

3. Action Items (Danh Sách Việc Cần Làm):
   ├── CRITICAL (màu đỏ): Cannot be automatically converted
   │   → Cần DBA viết lại hoàn toàn
   ├── WARNING (màu vàng/cam): Partially converted, needs review
   │   → SCT đã convert nhưng có thể không chính xác
   └── INFO (màu xanh): Converted successfully with notes
       → Review để đảm bảo behavior đúng
```

### Phân Tích Action Items

```
Ví dụ Action Item điển hình:

[CRITICAL] Stored Procedure: RETAIL.PROCESS_ORDER
Issues:
├── Uses Oracle DBMS_PIPE package — không có tương đương trong PostgreSQL
├── Uses UTL_FILE for file I/O — cần thay bằng COPY command hoặc external tables
└── AUTONOMOUS_TRANSACTION pragma — không hỗ trợ trong PostgreSQL

Action required:
└── Rewrite procedure to use PostgreSQL equivalent:
    DBMS_PIPE → pg_notify() hoặc LISTEN/NOTIFY
    UTL_FILE → COPY TO/FROM hoặc file_fdw extension

[WARNING] View: RETAIL.VW_CUSTOMER_ORDERS
Issues:
└── Uses Oracle CONNECT BY hierarchical query
Action required:
└── Replace with WITH RECURSIVE CTE (already converted — please verify)
```

---

## 🔄 Quy Trình Chuyển Đổi Schema

### Bước 1: Chuyển Đổi Tự Động

```
Trong SCT:
1. Mở rộng Source Database tree → chọn schema cần convert
2. Right-click → Convert Schema
3. SCT tự động convert tất cả objects
4. Kết quả xuất hiện trong Target Database panel:
   ├── Objects màu xanh: converted thành công
   ├── Objects màu vàng: converted với cảnh báo
   └── Objects màu đỏ: không thể convert tự động
```

### Bước 2: Review Và Sửa Thủ Công

```
Quy trình review từng action item:

1. Click vào object màu đỏ/vàng
2. Xem Source code (Oracle) và Converted code (PostgreSQL) song song
3. Đọc Action Item description
4. Sửa code trong Target panel:
   ├── Double-click vào object trong Target tree
   ├── Chỉnh sửa SQL code trực tiếp trong editor
   └── Lưu thay đổi

5. Test convert:
   ├── Apply lên test Aurora instance
   ├── Chạy queries thủ công để verify
   └── Nếu đúng → đánh dấu "resolved"
```

### Bước 3: Áp Dụng Lên Target DB

```
Cách 1: Apply trực tiếp từ SCT:
1. Right-click target schema → Apply to Database
2. SCT kết nối Aurora và chạy tất cả CREATE scripts
3. Verify trong Aurora: \dt, \df, \dv

Cách 2: Export script và review:
1. Right-click target schema → Save as SQL
2. Lưu file oracle_to_aurora_schema.sql
3. Review file (có thể là 50,000+ dòng SQL)
4. Chạy thủ công: psql -h aurora -U postgres < oracle_to_aurora_schema.sql

Ưu tiên Cách 2 cho production:
└── Giúp team review trước khi apply
    Version control schema script bằng Git
    Chạy có kiểm soát, dễ debug
```

---

## 🔧 Xử Lý Action Items

### Oracle-Specific Features Thường Gặp

#### 1. CONNECT BY → WITH RECURSIVE

```sql
-- Oracle:
SELECT level, employee_id, name, manager_id
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id
ORDER SIBLINGS BY name;

-- PostgreSQL (SCT convert + verify):
WITH RECURSIVE emp_tree AS (
  SELECT 1 AS level, employee_id, name, manager_id
  FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT et.level + 1, e.employee_id, e.name, e.manager_id
  FROM employees e
  JOIN emp_tree et ON e.manager_id = et.employee_id
)
SELECT * FROM emp_tree ORDER BY level, name;
```

#### 2. Oracle ROWNUM → PostgreSQL ROW_NUMBER() / LIMIT

```sql
-- Oracle:
SELECT * FROM (
  SELECT orders.*, ROWNUM rn FROM orders ORDER BY created_at DESC
) WHERE rn BETWEEN 11 AND 20;

-- PostgreSQL:
SELECT * FROM orders ORDER BY created_at DESC
LIMIT 10 OFFSET 10;

-- Hoặc dùng window function nếu cần giữ logic phức tạp:
SELECT * FROM (
  SELECT orders.*, ROW_NUMBER() OVER (ORDER BY created_at DESC) AS rn
  FROM orders
) subq WHERE rn BETWEEN 11 AND 20;
```

#### 3. NVL → COALESCE

```sql
-- Oracle:
SELECT NVL(phone, 'N/A') FROM customers;
SELECT NVL2(phone, 'Has phone', 'No phone') FROM customers;

-- PostgreSQL:
SELECT COALESCE(phone, 'N/A') FROM customers;
SELECT CASE WHEN phone IS NOT NULL THEN 'Has phone' ELSE 'No phone' END FROM customers;
```

#### 4. DECODE → CASE WHEN

```sql
-- Oracle:
SELECT DECODE(status, 'A', 'Active', 'I', 'Inactive', 'Unknown') FROM orders;

-- PostgreSQL:
SELECT CASE status
  WHEN 'A' THEN 'Active'
  WHEN 'I' THEN 'Inactive'
  ELSE 'Unknown'
END FROM orders;
```

#### 5. Oracle Sequence → PostgreSQL SEQUENCE

```sql
-- Oracle:
CREATE SEQUENCE order_seq START WITH 1 INCREMENT BY 1 NOCACHE NOCYCLE;
-- Sử dụng: INSERT INTO orders (id) VALUES (order_seq.NEXTVAL);

-- PostgreSQL:
CREATE SEQUENCE order_seq START 1 INCREMENT 1 NO CYCLE;
-- Sử dụng: INSERT INTO orders (id) VALUES (NEXTVAL('order_seq'));

-- Hoặc dùng SERIAL / GENERATED ALWAYS AS IDENTITY (PostgreSQL 10+):
CREATE TABLE orders (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  ...
);

-- Đặt sequence value sau migration:
SELECT SETVAL('order_seq', (SELECT MAX(id) FROM orders) + 1000);
-- +1000 để buffer, tránh conflict
```

#### 6. TO_CHAR / TO_DATE Format Differences

```sql
-- Oracle:
SELECT TO_CHAR(order_date, 'DD/MM/YYYY HH24:MI:SS') FROM orders;
SELECT TO_DATE('03/06/2026', 'DD/MM/YYYY') FROM DUAL;

-- PostgreSQL:
SELECT TO_CHAR(order_date, 'DD/MM/YYYY HH24:MI:SS') FROM orders;
-- TO_CHAR tương tự, nhưng một số format masks khác:
-- Oracle 'HH24' → PostgreSQL 'HH24' (giống nhau)
-- Oracle 'YYYY' → PostgreSQL 'YYYY' (giống nhau)
-- Oracle 'J' (Julian) → PostgreSQL 'J' (giống nhau)

SELECT TO_DATE('03/06/2026', 'DD/MM/YYYY');
-- Giống nhau, không cần FROM DUAL trong PostgreSQL
```

---

## 📦 Extension Pack — Gói Mở Rộng

### SCT Extension Pack Là Gì?

**Extension Pack** — Gói Mở Rộng — là tập hợp các functions và procedures được SCT tạo ra trên target DB để **giả lập (emulate)** các Oracle/SQL Server built-in functions không có tương đương trực tiếp trong target engine.

```
Extension Pack cài vào target DB:
├── Schema: aws_oracle_ext (cho Oracle → PostgreSQL)
└── Chứa các functions như:
    ├── aws_oracle_ext.to_date() — Oracle-compatible TO_DATE
    ├── aws_oracle_ext.substr() — Oracle SUBSTR behavior
    ├── aws_oracle_ext.instr() — Oracle INSTR
    ├── aws_oracle_ext.nvl() — NVL function
    └── ... hàng trăm functions khác

Ưu điểm:
└── Code Oracle được convert minimum thay đổi
    NVL(x, y) → aws_oracle_ext.nvl(x, y)

Nhược điểm:
├── Performance thấp hơn native PostgreSQL functions
└── Dependency vào Extension Pack (phải giữ trên DB)
```

### Khi Nào Dùng Extension Pack?

```
Dùng Extension Pack khi:
├── Cần convert nhanh (tight deadline)
├── Quá nhiều stored procedures khó convert thủ công
└── Phase đầu của migration (refactor sau)

Không dùng Extension Pack (tốt hơn) khi:
├── Có thời gian để convert properly
├── Performance là yêu cầu cao
└── Muốn code sạch, native PostgreSQL

Best practice: Dùng Extension Pack để unblock migration,
sau đó dần dần thay thế bằng native PostgreSQL code.
```

---

## 📋 Các Nguồn Và Đích Được Hỗ Trợ

### Source Databases (DB Nguồn Được Hỗ Trợ)

| Source | Phiên Bản | Target Được Khuyến Nghị |
| ------ | --------- | ---------------------- |
| **Oracle** | 10g, 11g, 12c, 18c, 19c, 21c | Aurora PostgreSQL, RDS PostgreSQL, Aurora MySQL |
| **SQL Server** | 2008 R2 - 2022 | Aurora MySQL, Aurora PostgreSQL |
| **IBM DB2** | 9.1 - 11.5 (LUW) | RDS PostgreSQL, Aurora PostgreSQL |
| **SAP ASE** (Sybase) | 12.5 - 16.0 | Aurora MySQL, Aurora PostgreSQL |
| **MySQL** | 5.5 - 8.0 | Aurora PostgreSQL (heterogeneous) |
| **PostgreSQL** | 9.4 - 15 | Aurora MySQL (heterogeneous) |
| **Teradata** | 13.0 - 16.20 | Amazon Redshift (data warehouse) |
| **Greenplum** | 5.x, 6.x | Amazon Redshift |
| **Netezza** | 6.x, 7.x | Amazon Redshift |

### Target Databases (DB Đích)

| Target | Ghi Chú |
| ------ | ------- |
| **Amazon Aurora MySQL** | Phổ biến |
| **Amazon Aurora PostgreSQL** | Phổ biến nhất cho Oracle migration |
| **Amazon RDS MySQL** | |
| **Amazon RDS PostgreSQL** | |
| **Amazon RDS MariaDB** | |
| **Amazon RDS Oracle** | Homogeneous với Oracle source |
| **Amazon RDS SQL Server** | Homogeneous với SQL Server source |
| **Amazon Redshift** | Data warehouse migration |

---

## ⚖️ SCT Vs. Chuyển Đổi Thủ Công

| Tiêu Chí | Dùng SCT | Chuyển Đổi Thủ Công |
| -------- | -------- | ------------------- |
| **Tốc độ** | Nhanh (tự động 60-95%) | Chậm (100% thủ công) |
| **Chất lượng code** | Trung bình (cần review) | Cao (được optimize) |
| **Chi phí** | Thấp (công cụ miễn phí) | Cao (DBA hours) |
| **Phù hợp với** | Số lượng lớn objects | Số ít objects phức tạp |
| **Risk** | Có thể bỏ sót edge cases | Kiểm soát tốt hơn |

**Thực tế tốt nhất:** Dùng SCT cho toàn bộ schema tự động, sau đó tập trung effort thủ công vào những objects SCT không convert được.

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: SCT là gì? Nó khác DMS như thế nào?**

> SCT — Schema Conversion Tool — là công cụ desktop miễn phí để chuyển đổi schema database từ engine này sang engine khác. SCT xử lý phần cấu trúc: tables, views, stored procedures, functions, triggers — và xuất ra SQL scripts. DMS ngược lại xử lý phần dữ liệu: kết nối source và target DB, tải data thực tế và sync thay đổi liên tục. Trong heterogeneous migration, cả hai được dùng cùng nhau: SCT chuẩn bị schema trên target, DMS di chuyển data.

**Q: Assessment Report trong SCT cho biết điều gì?**

> Assessment Report cho biết độ phức tạp của việc chuyển đổi: tổng số objects được phân tích, tỷ lệ % có thể tự động convert (thường 85-99% cho tables, 40-80% cho procedures), danh sách action items phân theo màu (đỏ = không thể tự động, vàng = cần review, xanh = ok), và ước tính effort tính bằng person-hours. Báo cáo này là input quan trọng cho project planning: giúp team ước tính timeline và resource cần thiết trước khi bắt đầu migration.

**Q: Extension Pack trong SCT là gì? Khi nào nên dùng?**

> Extension Pack là tập hợp functions được SCT tự động cài vào target DB để emulate các built-in functions của source DB không có tương đương trong target engine. Ví dụ, khi convert Oracle → PostgreSQL, Extension Pack cung cấp `aws_oracle_ext.nvl()`, `aws_oracle_ext.substr()` để thay thế Oracle-specific functions. Nên dùng khi deadline tight và cần unblock migration nhanh. Tuy nhiên, về dài hạn nên thay thế bằng native PostgreSQL functions để có performance tốt hơn và code sạch hơn.

**Q: Tại sao SCT không thể tự động convert 100% stored procedures?**

> Stored procedures của Oracle (PL/SQL) có nhiều tính năng không có tương đương trực tiếp trong PostgreSQL (PL/pgSQL): Oracle Packages (gom nhiều procedures vào một namespace với state), DBMS_* system packages (DBMS_PIPE, DBMS_SCHEDULER, UTL_FILE...), AUTONOMOUS_TRANSACTION pragma, CONNECT BY hierarchical queries lồng nhau, và dynamic SQL phức tạp. Những tính năng này đòi hỏi DBA hiểu sâu cả business logic lẫn syntax của cả hai platforms để viết lại đúng behavior — máy không thể thay thế hoàn toàn được.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
