# DMS Heterogeneous Migration — Di Chuyển Khác Loại Cơ Sở Dữ Liệu

> **Heterogeneous Migration** — Di Chuyển Dị Cấu Trúc — là kịch bản di chuyển database từ một engine sang engine khác. Ví dụ phổ biến nhất: Oracle → Amazon Aurora PostgreSQL, SQL Server → Amazon Aurora MySQL. Đây là loại migration phức tạp và có giá trị kinh tế cao nhất — thường để **thoát khỏi license Oracle/SQL Server** tốn kém sang open-source engine trên AWS.

## 📚 Mục Lục

1. [Tại Sao Migration Khác Engine?](#tại-sao-migration-khác-engine)
2. [Quy Trình Hai Bước: SCT + DMS](#quy-trình-hai-bước)
3. [Kịch Bản Phổ Biến: Oracle → Aurora PostgreSQL](#oracle-aurora-postgresql)
4. [Kịch Bản: SQL Server → Aurora MySQL](#sql-server-aurora-mysql)
5. [Chuyển Đổi Data Types](#chuyển-đổi-data-types)
6. [Xử Lý Stored Procedures Và Triggers](#xử-lý-stored-procedures-và-triggers)
7. [Testing Strategy — Chiến Lược Kiểm Thử](#testing-strategy)
8. [Rủi Ro Và Cách Giảm Thiểu](#rủi-ro-và-cách-giảm-thiểu)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## 💰 Tại Sao Migration Khác Engine?

### Chi Phí License Là Động Lực Chính

```
Chi phí Oracle Database Enterprise Edition:
├── On-premises: ~$47,500 / processor core / năm
├── Với server 16-core: $760,000/năm chỉ tiền license
└── Cộng thêm support, infrastructure, DBA...

So sánh với Amazon Aurora PostgreSQL:
├── db.r5.4xlarge (16 vCPU, 128 GB): ~$2,376/tháng = $28,512/năm
└── Tiết kiệm: $760,000 - $28,512 = ~$731,000/năm
   → ROI cực kỳ hấp dẫn cho enterprise

Tương tự với SQL Server Enterprise:
├── $14,256 / core, 16 core = $228,096/năm
└── Amazon Aurora MySQL: ~$1,000/tháng = $12,000/năm
```

### Use Cases Phổ Biến

```
Oracle → Aurora PostgreSQL:
├── Phổ biến nhất trong enterprise
├── PostgreSQL: open-source, SQL-compatible cao
└── Aurora PostgreSQL: 3x performance so với standard PostgreSQL

Oracle → Amazon RDS PostgreSQL:
├── Tương tự Aurora nhưng không cần serverless/auto-scaling
└── Phù hợp workload ổn định, có thể dự đoán

SQL Server → Aurora MySQL:
├── Migrate từ Windows ecosystem sang Linux/MySQL
└── Chi phí giảm mạnh, không cần Windows Server license

DB2 (IBM) → RDS PostgreSQL:
├── Thoát khỏi IBM mainframe ecosystem
└── PostgreSQL hỗ trợ tốt các tính năng ANSI SQL của DB2

Sybase ASE → Amazon Aurora MySQL:
└── Di sản từ thập niên 90-2000
```

---

## 🔄 Quy Trình Hai Bước: SCT + DMS

Heterogeneous migration bắt buộc phải thực hiện theo **hai bước độc lập**:

```
Bước 1: SCT (Schema Conversion Tool) — Chuyển Đổi Schema
├── Chuyển đổi DDL (CREATE TABLE, CREATE VIEW...)
├── Chuyển đổi stored procedures, functions, triggers (nếu có thể)
├── Tạo assessment report (báo cáo đánh giá) về độ phức tạp
└── Output: SQL scripts để tạo schema trên target DB

Bước 2: DMS (Database Migration Service) — Di Chuyển Dữ Liệu
├── Schema đã sẵn sàng trên target (nhờ SCT)
├── DMS kết nối source và target
├── Full Load: tải toàn bộ data
└── CDC: đồng bộ thay đổi liên tục cho zero-downtime

Lưu ý:
└── SCT chuyển đổi SCHEMA (cấu trúc, code)
    DMS di chuyển DATA (dữ liệu thực tế)
    Hai công việc này SONG SONG và ĐỘC LẬP nhau
```

---

## 🔴 Kịch Bản Phổ Biến: Oracle → Aurora PostgreSQL

### Giai Đoạn 1: Đánh Giá (Assessment) — 1-2 Tuần

```
Cài AWS SCT → Kết nối Oracle source:
├── File → New Migration Project
├── Source: Oracle — điền hostname, port, credentials
├── Target: Amazon Aurora PostgreSQL
└── Chạy: Create Report → Assessment Report

Assessment Report cho biết:
├── Summary: % objects có thể tự động convert
│   ├── Simple (tables, indexes): 95-100% tự động
│   └── Complex (stored procedures): 40-80% tự động
├── Action items: danh sách objects cần sửa thủ công
│   ├── Red: không thể convert tự động — cần viết lại
│   ├── Orange: có thể convert nhưng cần review
│   └── Green: convert thành công, không cần can thiệp
└── Effort estimation: ước tính số ngày/tuần cần thiết
```

### Giai Đoạn 2: Chuyển Đổi Schema — 2-8 Tuần

```
Trong SCT:
1. Apply conversion tự động cho tất cả objects
2. Review từng object màu Red/Orange:
   ├── Xem Oracle code gốc bên trái
   ├── Xem PostgreSQL code được convert bên phải
   └── Sửa thủ công nếu conversion không đúng

3. Ví dụ: Oracle sequence → PostgreSQL sequence:

Oracle:
CREATE SEQUENCE order_seq
  START WITH 1
  INCREMENT BY 1
  NOCACHE
  NOCYCLE;

PostgreSQL (SCT tạo ra):
CREATE SEQUENCE order_seq
  START 1
  INCREMENT 1
  NO CYCLE;
-- Thay NUMBER trong column bằng BIGINT hoặc SERIAL

4. Ví dụ: Oracle ROWNUM → PostgreSQL LIMIT:

Oracle:
SELECT * FROM orders WHERE ROWNUM <= 10;

PostgreSQL:
SELECT * FROM orders LIMIT 10;
```

### Chuyển Đổi Data Types Oracle → PostgreSQL

```
| Oracle Type    | PostgreSQL Type         | Ghi Chú                              |
|---------------|------------------------|--------------------------------------|
| NUMBER(p,s)   | NUMERIC(p,s)           | Tương đương, SCT tự động             |
| NUMBER         | NUMERIC                | Hoặc DECIMAL                        |
| VARCHAR2(n)   | VARCHAR(n)             | Tương đương                          |
| NVARCHAR2(n)  | VARCHAR(n)             | PostgreSQL UTF-8 native              |
| CLOB           | TEXT                   | Character Large Object               |
| BLOB           | BYTEA                  | Binary Large Object                  |
| DATE           | TIMESTAMP              | Oracle DATE có time component!       |
| TIMESTAMP      | TIMESTAMP              | Tương đương                          |
| RAW(n)         | BYTEA                  |                                      |
| LONG           | TEXT                   | Deprecated Oracle type              |
| XMLTYPE        | XML                    |                                      |
| ROWID          | Không tương đương      | Phải redesign query dùng ROWID       |
| DUAL table     | Không cần              | SELECT 1; thay vì SELECT 1 FROM DUAL|
```

### Giai Đoạn 3: Apply Schema Lên Aurora

```sql
-- Từ SCT: Export SQL scripts
-- Apply lên Aurora PostgreSQL:
psql -h aurora-endpoint -U postgres -d retail < schema_converted.sql

-- Kiểm tra schema được tạo đúng:
\dt retail.*            -- liệt kê tables
\df retail.*            -- liệt kê functions
\dv retail.*            -- liệt kê views

-- Tạo thêm indexes (SCT có thể bỏ sót):
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders(created_at);
```

### Giai Đoạn 4: DMS Full Load + CDC

```
Cấu hình DMS:
├── Source Endpoint: Oracle
│   ├── Cần bật Oracle Supplemental Logging:
│   │   ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
│   └── DMS dùng LogMiner hoặc Binary Reader để đọc redo logs
│
├── Target Endpoint: Aurora PostgreSQL
│   └── Thông thường là Aurora cluster endpoint (writer)
│
└── Task: Full Load + CDC

Mapping đặc biệt cho Oracle → PostgreSQL:
├── Oracle schema "RETAIL" → PostgreSQL schema "retail" (lowercase)
├── Thêm transformation rule:
{
  "rule-type": "transformation",
  "rule-action": "convert-lowercase",
  "rule-target": "schema",
  "object-locator": {"schema-name": "%"}
}
```

### Giai Đoạn 5: Testing Ứng Dụng — Quan Trọng Nhất

```
Đây là giai đoạn tốn thời gian nhất (2-4 tuần):

1. Functional testing (Kiểm thử chức năng):
   ├── Chạy toàn bộ test suite của ứng dụng
   ├── Test từng API endpoint với Aurora backend
   └── So sánh kết quả với Oracle source

2. Performance testing (Kiểm thử hiệu năng):
   ├── Load test với traffic tương đương production
   ├── Phân tích slow queries với EXPLAIN ANALYZE
   └── Tạo thêm indexes cần thiết

3. Query compatibility:
   ├── Tìm tất cả queries dùng Oracle-specific syntax:
   │   SELECT ... FROM DUAL
   │   CONNECT BY PRIOR (hierarchical queries)
   │   NVL() → COALESCE()
   │   DECODE() → CASE WHEN
   │   SYSDATE → CURRENT_TIMESTAMP
   │   TO_CHAR/TO_DATE với Oracle format strings
   └── Cập nhật application code

4. Stored procedure testing:
   ├── Test từng procedure đã được convert bằng SCT
   └── So sánh output giữa Oracle và Aurora
```

---

## 🔵 Kịch Bản: SQL Server → Aurora MySQL

### Điểm Khác Biệt Chính

```
Data Types SQL Server → MySQL:

| SQL Server     | MySQL          | Ghi Chú                   |
|---------------|----------------|---------------------------|
| NVARCHAR(n)   | VARCHAR(n)     | MySQL 8 UTF-8 by default  |
| DATETIME      | DATETIME       | Tương đương               |
| DATETIME2     | DATETIME(6)    | Microsecond precision     |
| BIT           | TINYINT(1)     | 0/1                       |
| UNIQUEIDENTIFIER | CHAR(36)    | UUID string               |
| TEXT/NTEXT    | LONGTEXT       | Deprecated SQL Server type|
| VARBINARY(MAX)| LONGBLOB       |                           |
| MONEY         | DECIMAL(19,4)  | SCT tự chuyển             |
| IDENTITY      | AUTO_INCREMENT | SCT tự chuyển             |

Cú pháp SQL khác nhau:
├── SQL Server: SELECT TOP 10 * FROM orders
│   MySQL: SELECT * FROM orders LIMIT 10
├── SQL Server: ISNULL(col, default)
│   MySQL: IFNULL(col, default) hoặc COALESCE(col, default)
├── SQL Server: GETDATE()
│   MySQL: NOW() hoặc CURRENT_TIMESTAMP
└── SQL Server: STRING_AGG (2017+)
    MySQL: GROUP_CONCAT
```

### CDC Cho SQL Server

```
SQL Server hỗ trợ hai phương thức CDC trong DMS:

1. MS-CDC (SQL Server CDC Feature):
   ├── Bật trong SQL Server:
   │   USE retail;
   │   EXEC sys.sp_cdc_enable_db;
   │   EXEC sys.sp_cdc_enable_table
   │     @source_schema = N'dbo',
   │     @source_name = N'orders',
   │     @role_name = N'cdc_role';
   └── Khuyến nghị cho production

2. Backup mode (Log-based CDC):
   ├── Đọc từ SQL Server transaction log backup
   └── Phù hợp khi MS-CDC không thể bật
```

---

## 🔀 Chuyển Đổi Data Types

### Vấn Đề Thường Gặp Với Oracle NUMBER

```
Oracle NUMBER không có fixed precision → nhiều cách xử lý:

Oracle:
CREATE TABLE products (
    id       NUMBER,           -- integer-like, no scale
    price    NUMBER(10,2),     -- có precision
    quantity NUMBER(5)         -- có precision, no scale
);

PostgreSQL sau SCT:
CREATE TABLE products (
    id       NUMERIC,          -- có thể không chính xác
    price    NUMERIC(10,2),
    quantity NUMERIC(5,0)
);

Tốt hơn (cần sửa thủ công):
CREATE TABLE products (
    id       BIGINT,           -- rõ ràng hơn cho integer
    price    NUMERIC(10,2),
    quantity INTEGER
);

Lý do: NUMERIC không có precision kém hiệu năng hơn BIGINT/INTEGER
→ Cần review tất cả NUMBER columns trong Oracle schema
```

### Oracle DATE Vs PostgreSQL DATE

```
Đây là bẫy phổ biến nhất:

Oracle DATE = DATE + TIME (ví dụ: 2026-06-03 14:30:00)
PostgreSQL DATE = chỉ ngày (2026-06-03) — không có giờ!

Giải pháp:
├── Oracle DATE → PostgreSQL TIMESTAMP (không phải DATE)
└── SCT thường làm đúng điều này, nhưng cần kiểm tra

Ví dụ:
Oracle: SELECT TO_CHAR(order_date, 'YYYY-MM-DD HH24:MI:SS') FROM orders;
PostgreSQL: SELECT TO_CHAR(order_date, 'YYYY-MM-DD HH24:MI:SS') FROM orders;
-- TO_CHAR hoạt động tương tự — nhưng Oracle format mask đôi khi khác
```

---

## ⚙️ Xử Lý Stored Procedures Và Triggers

### Phần Khó Nhất Của Heterogeneous Migration

```
Tỷ lệ tự động conversion bởi SCT (ước tính):

| Object Type        | Tỷ Lệ Tự Động | Ghi Chú                               |
|--------------------|--------------|---------------------------------------|
| Tables             | ~99%         | Chủ yếu data type mapping             |
| Views              | ~85-95%      | Tùy thuộc vào Oracle-specific syntax  |
| Indexes            | ~95%         | Có thể cần điều chỉnh type            |
| Simple Procedures  | ~70-80%      | Logic đơn giản, cursor cơ bản         |
| Complex Procedures | ~30-50%      | Dynamic SQL, Oracle packages phức tạp |
| Triggers           | ~60-75%      | Tùy thuộc logic                       |
| Oracle Packages    | ~40-60%      | Không có tương đương trực tiếp trong PG|
```

### Ví Dụ: Oracle Package → PostgreSQL Schema/Functions

```sql
-- Oracle Package:
CREATE OR REPLACE PACKAGE retail_pkg AS
  PROCEDURE get_customer_orders(p_cust_id IN NUMBER, p_cursor OUT SYS_REFCURSOR);
  FUNCTION calc_discount(p_amount IN NUMBER) RETURN NUMBER;
END retail_pkg;

CREATE OR REPLACE PACKAGE BODY retail_pkg AS
  PROCEDURE get_customer_orders(p_cust_id IN NUMBER, p_cursor OUT SYS_REFCURSOR) IS
  BEGIN
    OPEN p_cursor FOR
      SELECT * FROM orders WHERE customer_id = p_cust_id;
  END;

  FUNCTION calc_discount(p_amount IN NUMBER) RETURN NUMBER IS
  BEGIN
    IF p_amount > 1000 THEN RETURN p_amount * 0.1;
    ELSE RETURN 0;
    END IF;
  END;
END retail_pkg;

-- PostgreSQL (SCT convert + manual fix):
-- Oracle packages → PostgreSQL schemas hoặc prefix naming

CREATE OR REPLACE FUNCTION retail_pkg_get_customer_orders(p_cust_id BIGINT)
RETURNS REFCURSOR AS $$
DECLARE
  v_cursor REFCURSOR := 'mycursor';
BEGIN
  OPEN v_cursor FOR
    SELECT * FROM orders WHERE customer_id = p_cust_id;
  RETURN v_cursor;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION retail_pkg_calc_discount(p_amount NUMERIC)
RETURNS NUMERIC AS $$
BEGIN
  IF p_amount > 1000 THEN RETURN p_amount * 0.1;
  ELSE RETURN 0;
  END IF;
END;
$$ LANGUAGE plpgsql;
```

### Ví Dụ: Oracle CONNECT BY → PostgreSQL Recursive CTE

```sql
-- Oracle hierarchical query:
SELECT employee_id, manager_id, name, LEVEL
FROM employees
START WITH manager_id IS NULL
CONNECT BY PRIOR employee_id = manager_id;

-- PostgreSQL (WITH RECURSIVE — Common Table Expression Đệ Quy):
WITH RECURSIVE employee_hierarchy AS (
  -- Base case: root nodes (managers with no parent)
  SELECT employee_id, manager_id, name, 1 AS level
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive case: find children
  SELECT e.employee_id, e.manager_id, e.name, eh.level + 1
  FROM employees e
  INNER JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level;
```

---

## 🧪 Testing Strategy — Chiến Lược Kiểm Thử

### Kiểm Thử Theo Tầng

```
Tầng 1: Schema Verification (Kiểm Tra Schema)
├── Đếm số tables, views, functions trên target
├── So sánh column names và data types
└── Kiểm tra constraints, indexes, foreign keys

Tầng 2: Data Verification (Kiểm Tra Dữ Liệu)
├── Row count per table: phải khớp 100%
├── Checksum validation bằng DMS hoặc manual scripts
└── Sample data check: lấy 100 row ngẫu nhiên và so sánh

Tầng 3: Stored Procedure Testing (Kiểm Tra Thủ Tục)
├── Chạy từng procedure/function với test data
├── So sánh output giữa Oracle và Aurora
└── Đặc biệt chú ý: numeric precision, date formatting

Tầng 4: Integration Testing (Kiểm Thử Tích Hợp)
├── Kết nối ứng dụng thực vào Aurora PostgreSQL
├── Chạy regression test suite
└── Kiểm tra performance (slow queries, missing indexes)

Tầng 5: Load Testing (Kiểm Thử Tải)
└── Mô phỏng production traffic để phát hiện bottlenecks
```

---

## ⚠️ Rủi Ro Và Cách Giảm Thiểu

| Rủi Ro | Khả Năng | Cách Giảm Thiểu |
| ------ | -------- | --------------- |
| **Query behavior thay đổi** | Cao | Kiểm thử toàn diện, null handling khác nhau |
| **Stored procedure lỗi** | Cao | Kiểm thử từng procedure, test data đầy đủ |
| **Performance regression** | Trung bình | EXPLAIN ANALYZE, tạo indexes bổ sung |
| **Character encoding issues** | Thấp-Trung | Đảm bảo UTF-8 nhất quán |
| **Sequence out of sync** | Thấp | Đặt sequence start value = MAX(id) + buffer |
| **Timezone differences** | Thấp | Kiểm tra TIMEZONE setting nhất quán |

### Vấn Đề NULL Handling Khác Nhau

```sql
-- Oracle: empty string = NULL
INSERT INTO test (name) VALUES ('');
SELECT name IS NULL FROM test; -- Oracle trả về TRUE

-- PostgreSQL: empty string ≠ NULL
SELECT name IS NULL FROM test; -- PostgreSQL trả về FALSE
SELECT name = '' FROM test;   -- PostgreSQL trả về TRUE

-- Giải pháp: Review tất cả NULL checks trong application code
-- Thêm NULLIF() hoặc COALESCE() nếu cần
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Tại sao heterogeneous migration phức tạp hơn homogeneous?**

> Heterogeneous migration cần chuyển đổi hai thứ thay vì một: không chỉ di chuyển dữ liệu (data migration) mà còn phải chuyển đổi cấu trúc schema và code business logic (schema + code migration). Mỗi DB engine có data types, cú pháp SQL, stored procedure language, và tính năng riêng biệt. Ví dụ, Oracle dùng PL/SQL với packages, CONNECT BY hierarchical queries, DATE type có component time; trong khi PostgreSQL dùng PL/pgSQL, WITH RECURSIVE, và có DATE type chỉ chứa ngày. SCT giúp tự động hóa phần lớn conversion nhưng không thể 100% — phần còn lại đòi hỏi DBA expertise và testing kỹ lưỡng.

**Q: SCT (Schema Conversion Tool) có thể tự động convert 100% Oracle code sang PostgreSQL không?**

> Không. SCT có thể tự động convert ~95% tables và views, nhưng chỉ ~40-80% stored procedures và triggers tùy độ phức tạp. Những phần SCT không thể tự động convert (đánh dấu màu đỏ) thường là: Oracle-specific features như CONNECT BY hierarchical queries, complex Oracle packages với state management, Dynamic SQL phức tạp, UTL_FILE / DBMS_PIPE package calls, và Oracle-specific analytic functions. Những phần này cần DBA viết lại thủ công bằng PL/pgSQL — đây thường là phần tốn thời gian nhất trong heterogeneous migration.

**Q: Quy trình tổng quát cho Oracle → Aurora PostgreSQL migration?**

> (1) Assessment: Dùng SCT tạo Assessment Report để biết độ phức tạp, ước tính effort. (2) Schema conversion: SCT convert tự động + DBA sửa thủ công phần phức tạp, apply lên Aurora test environment. (3) Data migration: DMS Full Load + CDC để tải dữ liệu và sync liên tục. (4) Testing: functional testing, performance testing, stored procedure testing so sánh Oracle vs Aurora. (5) Application changes: cập nhật queries dùng Oracle-specific syntax (ROWNUM → LIMIT, NVL → COALESCE, SYSDATE → NOW()). (6) Cutover: maintenance window, drain CDC lag về 0, chuyển connection string sang Aurora.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Trạng Thái:** ✅ Hoàn Thành
