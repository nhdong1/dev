# AWS SCT — Schema Conversion Tool: Chuyển Đổi Schema

> AWS SCT (Schema Conversion Tool — Công Cụ Chuyển Đổi Schema) là công cụ desktop miễn phí giúp chuyển đổi database schema (cấu trúc cơ sở dữ liệu) từ một engine sang engine khác. SCT tự động hóa phần lớn công việc chuyển đổi trong heterogeneous migration (di chuyển khác loại engine) và báo cáo chi tiết những gì cần xử lý thủ công.

## 📚 Mục Lục

1. [Tổng Quan SCT](#tổng-quan-sct)
2. [Luồng Làm Việc với SCT](#luồng-làm-việc-với-sct)
3. [Báo Cáo Đánh Giá](#báo-cáo-đánh-giá)
4. [Chuyển Đổi Các Loại Object](#chuyển-đổi-các-loại-object)
5. [Extension Pack](#extension-pack)
6. [Use Cases Phổ Biến](#use-cases-phổ-biến)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 🎯 Tổng Quan SCT

### SCT Là Gì?

SCT là công cụ chạy trên máy tính (Windows/macOS/Linux) — không phải dịch vụ AWS cloud. Bạn cài đặt SCT, kết nối đến source database và target database, rồi SCT phân tích và chuyển đổi.

### Khi Nào Dùng SCT?

```
Cần dùng SCT khi di chuyển giữa các engine KHÁC loại:

  Oracle      ──▶  Aurora PostgreSQL / Aurora MySQL / MySQL
  SQL Server  ──▶  Aurora PostgreSQL / Aurora MySQL
  Teradata    ──▶  Redshift
  Oracle DW   ──▶  Redshift

KHÔNG cần SCT khi cùng loại engine:
  MySQL       ──▶  MySQL on RDS / Aurora MySQL
  PostgreSQL  ──▶  PostgreSQL on RDS / Aurora PostgreSQL
```

### SCT vs DMS — Khác Biệt Chính

| Tiêu Chí              | SCT                                       | DMS                                        |
| --------------------- | ----------------------------------------- | ------------------------------------------ |
| **Mục đích**          | Chuyển đổi **schema** (cấu trúc)          | Di chuyển **data** (dữ liệu)               |
| **Cài đặt**           | Desktop app (cài trên máy local)          | Cloud service (managed bởi AWS)            |
| **Chi phí**           | Miễn phí                                  | Trả tiền theo giờ replication instance     |
| **Xử lý**             | Stored procs, triggers, views, DDL        | INSERT/UPDATE/DELETE, CDC                  |
| **Kết quả**           | SQL scripts để tạo schema trên target     | Data đã được copy sang target              |

**Thứ tự thực hiện:** SCT trước (tạo schema), DMS sau (copy data).

---

## 🔄 Luồng Làm Việc với SCT

### Bước 1: Cài Đặt & Kết Nối

```
1. Tải SCT từ AWS: aws.amazon.com/dms/schema-conversion-tool/
2. Cài đặt trên máy có thể kết nối đến cả source và target
3. Tạo project mới
4. Kết nối source database:
   - Engine: Oracle / SQL Server / ...
   - Hostname, port, username, password
5. Kết nối target database:
   - Engine: Aurora PostgreSQL / Aurora MySQL / Redshift / ...
   - Hostname, port, username, password
```

### Bước 2: Phân Tích Schema

SCT quét toàn bộ schema và phân loại từng object:

```
Green  ██  — Tự động chuyển đổi hoàn toàn, không cần chỉnh
Yellow ██  — Chuyển đổi được nhưng cần kiểm tra, có thể có thay đổi ngữ nghĩa
Orange ██  — Chuyển đổi một phần, cần điều chỉnh thủ công
Red    ██  — Không thể chuyển đổi tự động, phải viết lại hoàn toàn
```

### Bước 3: Đọc Assessment Report (Báo Cáo Đánh Giá)

SCT tạo report chi tiết với:
- Tổng số objects theo màu (green/yellow/orange/red)
- **Conversion complexity score** (điểm độ phức tạp chuyển đổi) — 0-100%
- Danh sách từng object cụ thể với lý do không chuyển được
- Ước tính thời gian xử lý thủ công

### Bước 4: Apply Conversion (Áp Dụng Chuyển Đổi)

```
1. Click "Convert Schema" trên tree view
2. SCT tạo SQL DDL scripts cho target engine
3. Review từng object — đặc biệt yellow/orange
4. Apply sang target database
   - Trực tiếp qua SCT (kết nối target)
   - Hoặc export SQL scripts và chạy thủ công
```

### Bước 5: Xử Lý Objects Không Chuyển Được

Với red objects (không tự động chuyển):
```
1. Xem lý do cụ thể trong SCT report
2. Viết lại logic tương đương trong engine đích
   Ví dụ: Oracle CONNECT BY (cây phân cấp) → PostgreSQL WITH RECURSIVE
3. Test kỹ output giống nhau không
4. Tài liệu hóa các thay đổi cho team review
```

---

## 📊 Báo Cáo Đánh Giá

### Cấu Trúc Assessment Report

```
Executive Summary (Tóm Tắt):
  - Tổng số database objects: 1,245
  - Tự động chuyển được:     1,089 (87%)
  - Cần xem xét:               112 (9%)
  - Cần viết lại thủ công:      44 (4%)
  
  Ước tính thời gian thủ công: 32 giờ

Object Breakdown (Phân Tích Đối Tượng):
  Tables:             450 / 450 (100% green)
  Views:              120 / 135 (89% green)
  Stored Procedures:   89 / 210 (42% green)
  Functions:           67 / 120 (56% green)
  Triggers:            20 /  80 (25% green)
  Sequences:           10 /  10 (100% green)
  Packages:             0 /  40 ( 0% — Oracle-specific, không có tương đương)
```

### Đọc Hiểu Report Theo Nhóm

**Tables & Columns (Bảng & Cột) — Thường Green**

- Data type mapping (ánh xạ kiểu dữ liệu) thường tự động tốt
- Chú ý: NUMBER(p,s) Oracle → NUMERIC(p,s) PostgreSQL (fine)
- Chú ý: Oracle DATE chứa cả thời gian → PostgreSQL TIMESTAMP

**Stored Procedures & Functions (Thủ Tục Lưu Trữ & Hàm) — Thường Yellow/Red**

- Oracle PL/SQL → PostgreSQL PL/pgSQL: cú pháp khác nhau nhiều
- SQL Server T-SQL → PostgreSQL: logic xử lý lỗi khác
- Cursors, dynamic SQL thường cần viết lại

**Triggers (Bộ Kích Hoạt) — Thường Orange/Red**

- Oracle BEFORE/AFTER triggers vs PostgreSQL trigger functions
- System triggers (DDL triggers) không có tương đương
- Instead-of triggers (SQL Server) → rule/trigger khác nhau

---

## 🔁 Chuyển Đổi Các Loại Object

### Data Type Mapping — Ánh Xạ Kiểu Dữ Liệu

**Oracle → Aurora PostgreSQL:**

| Oracle Type            | PostgreSQL Type         | Ghi Chú                                  |
| ---------------------- | ----------------------- | ----------------------------------------- |
| NUMBER(p,s)            | NUMERIC(p,s)            | Tương đương                               |
| NUMBER (không scale)   | BIGINT hoặc FLOAT8      | Tùy ngữ cảnh                             |
| VARCHAR2(n)            | VARCHAR(n)              | Tương đương                               |
| NVARCHAR2(n)           | VARCHAR(n)              | PostgreSQL Unicode natively               |
| DATE                   | TIMESTAMP               | Oracle DATE có time component!            |
| CLOB                   | TEXT                    | Tương đương                               |
| BLOB                   | BYTEA                   | Tương đương                               |
| XMLTYPE                | XML                     | Tương đương nhưng functions khác          |
| ROWID                  | OID / ctid              | Không có tương đương trực tiếp            |

**SQL Server → Aurora MySQL:**

| SQL Server Type  | MySQL Type              | Ghi Chú                       |
| ---------------- | ----------------------- | ----------------------------- |
| NVARCHAR(n)      | VARCHAR(n) CHARACTER SET utf8mb4 | Giữ được Unicode      |
| DATETIME2        | DATETIME(6)             | Microsecond precision          |
| UNIQUEIDENTIFIER | CHAR(36)                | UUID không có native type      |
| BIT              | TINYINT(1)              | Boolean representation         |
| MONEY            | DECIMAL(19,4)           | Tương đương                   |

### Code Conversion Examples (Ví Dụ Chuyển Đổi Code)

**Oracle PL/SQL → PostgreSQL PL/pgSQL:**

```sql
-- Oracle: Stored procedure
CREATE OR REPLACE PROCEDURE get_employee(
  p_id IN NUMBER,
  p_name OUT VARCHAR2
) AS
BEGIN
  SELECT emp_name INTO p_name
  FROM employees
  WHERE emp_id = p_id;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    p_name := NULL;
END;

-- PostgreSQL: Function (không có OUT params như Oracle)
CREATE OR REPLACE FUNCTION get_employee(p_id INTEGER)
RETURNS TEXT AS $$
DECLARE
  v_name TEXT;
BEGIN
  SELECT emp_name INTO v_name
  FROM employees
  WHERE emp_id = p_id;
  RETURN v_name;
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

**Oracle CONNECT BY → PostgreSQL Recursive CTE:**

```sql
-- Oracle: Hierarchical query (truy vấn phân cấp)
SELECT emp_id, emp_name, manager_id, LEVEL
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR emp_id = manager_id;

-- PostgreSQL: Recursive CTE (Common Table Expression đệ quy)
WITH RECURSIVE org_chart AS (
  -- Base case: root nodes (nút gốc)
  SELECT emp_id, emp_name, manager_id, 1 AS level
  FROM employees
  WHERE manager_id IS NULL
  
  UNION ALL
  
  -- Recursive case (trường hợp đệ quy)
  SELECT e.emp_id, e.emp_name, e.manager_id, oc.level + 1
  FROM employees e
  INNER JOIN org_chart oc ON e.manager_id = oc.emp_id
)
SELECT * FROM org_chart;
```

---

## 📦 Extension Pack — Gói Mở Rộng

### Extension Pack Là Gì?

Extension Pack (Gói Mở Rộng) là tập hợp các functions và procedures mà SCT tạo ra trên target database để **giả lập hành vi của source engine**.

Ví dụ: Oracle có built-in function `TO_DATE()` với format strings khác PostgreSQL. SCT tạo một function `AWS_ORACLE_EXT.TO_DATE()` trên PostgreSQL để tương thích.

### Cài Đặt Extension Pack

```
SCT → Extension Pack → Apply to Target Database

Extension Pack tạo schema riêng:
- AWS_ORACLE_EXT — cho Oracle → PostgreSQL migration
- AWS_SQLSERVER_EXT — cho SQL Server → MySQL/PostgreSQL
- AWS_TERADATA_EXT — cho Teradata → Redshift
```

### Hạn Chế

- Extension Pack thêm overhead (chi phí xử lý thêm) vào runtime
- Sau khi ổn định, nên refactor code để dùng native functions của engine đích
- Extension Pack là giải pháp tạm thời để migration, không phải long-term

---

## 🗂️ Use Cases Phổ Biến

### Oracle → Aurora PostgreSQL (Phổ Biến Nhất)

**Lý do di chuyển:** Oracle license rất đắt (~$50,000-100,000/core/năm). Aurora PostgreSQL rẻ hơn 90% với hiệu năng tương đương cho nhiều workloads.

```
Thách thức:
  - Oracle Packages → PostgreSQL không có Packages
    Giải pháp: Split thành functions/procedures riêng lẻ
  - Oracle Sequences → PostgreSQL Sequences (tương đương nhưng cú pháp khác)
  - Oracle hint (optimizer hints) như /*+ INDEX */ → không có tương đương
  - Oracle ROWNUM → PostgreSQL LIMIT/ROW_NUMBER()

Timeline thực tế cho Oracle ~500 GB:
  Week 1: Assessment, SCT report
  Week 2-3: Schema conversion, manual fixes
  Week 4: Application testing
  Week 5: DMS full load + CDC setup
  Week 6: Cutover
```

### SQL Server → Aurora MySQL

```
Thách thức:
  - T-SQL specific syntax (TOP, NOLOCK hints)
  - Identity columns → AUTO_INCREMENT
  - Temp tables (#temp) → MySQL temp tables (khác cú pháp)
  - Collation (bảng mã) phải match giữa source và target

Thường dễ hơn Oracle → PostgreSQL
```

### Teradata → Redshift

SCT có chế độ đặc biệt cho Data Warehouse migration:
- Phân tích distribution keys (khóa phân phối) tối ưu cho Redshift
- Chuyển đổi Teradata SQL dialect sang Redshift SQL
- Ước tính tiết kiệm chi phí

---

## 🎯 Câu Hỏi Phỏng Vấn

**Q: SCT (Schema Conversion Tool) là gì và khi nào cần dùng?**

> SCT là công cụ desktop miễn phí của AWS giúp chuyển đổi database schema từ engine này sang engine khác trong heterogeneous migration. Cần dùng SCT khi di chuyển giữa các engine khác loại, ví dụ Oracle sang Aurora PostgreSQL, SQL Server sang Aurora MySQL, hoặc Teradata sang Redshift. SCT tự động phân tích mức độ tương thích, chuyển đổi tables/views/indexes tự động, báo cáo các stored procedures và triggers cần viết lại thủ công, và tạo extension pack để giả lập hành vi của source engine.

**Q: Sau khi SCT báo cáo 20% objects là "red" (cần viết lại thủ công), bạn xử lý thế nào?**

> Trước tiên phân tích loại objects đó là gì — thường là stored procedures, triggers, hoặc Oracle-specific syntax như packages. Với mỗi red object, đọc lý do SCT báo và viết lại logic tương đương trong engine đích (ví dụ: Oracle CONNECT BY → PostgreSQL WITH RECURSIVE). Cần test kỹ output giống nhau với production data samples. Thêm vào estimation timeline — 20% red thường mất 1-2 tuần thủ công tùy complexity.

**Q: Extension Pack là gì và có vấn đề gì không?**

> Extension Pack là tập hợp functions SCT tạo ra trên target database để giả lập behavior của source engine, ví dụ `AWS_ORACLE_EXT.TO_DATE()` thay thế Oracle's `TO_DATE()` trên PostgreSQL. Vấn đề là extension pack thêm overhead vào runtime và tạo dependency vào AWS-specific code. Long-term nên refactor dần để dùng native functions của engine đích, nhưng ngắn hạn extension pack giúp migration nhanh hơn bằng cách giảm lượng code cần viết lại.

---

**Liên Kết:**
- [1-dms-overview.md](./1-dms-overview.md) — AWS DMS chi tiết
- [3-cdc-online-migration.md](./3-cdc-online-migration.md) — CDC và zero-downtime
- [4-migration-strategies.md](./4-migration-strategies.md) — Chiến lược migration

**Cập Nhật Lần Cuối:** 2026-05-15
