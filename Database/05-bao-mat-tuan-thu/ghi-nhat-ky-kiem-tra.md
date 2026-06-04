# Ghi Nhật Ký Kiểm Tra (Audit Logging)

Theo dõi mọi hành động trên CSDL để phát hiện vi phạm, điều tra sự cố và tuân thủ quy định.

## Tại Sao Cần Audit Logging?

```
Yêu cầu quy định:
- GDPR: Chứng minh ai đã truy cập PII
- PCI-DSS: Log tất cả truy cập dữ liệu cardholder
- HIPAA: Audit trail cho dữ liệu y tế
- SOC 2: Kiểm soát và giám sát truy cập

Yêu cầu vận hành:
- Điều tra: "Ai đã xóa dữ liệu này?"
- Phát hiện bất thường: Query lạ lúc 3 giờ sáng
- Forensics sau sự cố bảo mật
```

---

## pgaudit Extension

### Cài Đặt

```sql
-- Ubuntu/Debian:
-- apt-get install postgresql-15-pgaudit

-- postgresql.conf:
shared_preload_libraries = 'pgaudit'

-- Reload:
SELECT pg_reload_conf();

-- Bật extension trong database:
CREATE EXTENSION pgaudit;
```

### Cấu Hình Cơ Bản

```sql
-- Log tất cả DDL (CREATE, ALTER, DROP):
ALTER SYSTEM SET pgaudit.log = 'DDL';

-- Log DDL + DML (INSERT, UPDATE, DELETE):
ALTER SYSTEM SET pgaudit.log = 'DDL, WRITE';

-- Log mọi thứ (CẢNH BÁO: Rất nhiều log!):
ALTER SYSTEM SET pgaudit.log = 'ALL';

-- Áp dụng:
SELECT pg_reload_conf();

-- Các giá trị log:
-- READ     → SELECT, COPY FROM
-- WRITE    → INSERT, UPDATE, DELETE, TRUNCATE, COPY TO
-- FUNCTION → Gọi function
-- ROLE     → GRANT, REVOKE, CREATE/ALTER/DROP ROLE/USER
-- DDL      → Tất cả DDL không thuộc ROLE
-- MISC     → FETCH, VACUUM, SET, ...
-- ALL      → Tất cả ở trên
```

### Audit Theo Role (Object-Level Auditing)

```sql
-- Tạo audit role:
CREATE ROLE audit_role;

-- Cấu hình audit cho role đó:
ALTER SYSTEM SET pgaudit.role = 'audit_role';
SELECT pg_reload_conf();

-- Cấp quyền audit cho bảng nhạy cảm:
GRANT SELECT ON users TO audit_role;          -- Audit mọi SELECT trên users
GRANT SELECT ON payments TO audit_role;       -- Audit mọi SELECT trên payments
GRANT SELECT ON medical_records TO audit_role;

-- Bây giờ mọi SELECT trên các bảng này đều được log
-- Kể cả khi dùng qua role khác
```

### Xem Audit Log

```bash
# Log được ghi vào PostgreSQL log file:
tail -f /var/log/postgresql/postgresql.log | grep AUDIT

# Ví dụ log entry:
# AUDIT: SESSION,1,1,DDL,ALTER TABLE,,,"ALTER TABLE users ADD COLUMN ssn_encrypted bytea",<not logged>
# AUDIT: OBJECT,1,1,READ,TABLE,public,users,"SELECT id, name, ssn FROM users WHERE id = 123",<not logged>
# AUDIT: SESSION,1,1,WRITE,DELETE,,,DELETE FROM sessions WHERE user_id = 456,<not logged>

# Tìm ai đã truy cập bảng nhạy cảm:
grep "AUDIT.*TABLE.*users" /var/log/postgresql/postgresql.log | \
  grep "$(date +%Y-%m-%d)" | \
  awk '{print $1, $2, $NF}' | head -50
```

---

## Audit Table Tự Xây Dựng

### Bảng Audit Log

```sql
-- Tạo bảng audit log:
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    event_time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    table_name TEXT NOT NULL,
    operation TEXT NOT NULL,          -- INSERT, UPDATE, DELETE
    user_name TEXT NOT NULL,
    app_user_id INTEGER,              -- User từ application context
    old_data JSONB,                   -- Dữ liệu trước thay đổi
    new_data JSONB,                   -- Dữ liệu sau thay đổi
    changed_fields TEXT[],            -- Tên các cột bị thay đổi
    client_ip INET,
    query_text TEXT
);

-- Index cho truy vấn audit:
CREATE INDEX idx_audit_table_time ON audit_log(table_name, event_time DESC);
CREATE INDEX idx_audit_user ON audit_log(user_name, event_time DESC);
CREATE INDEX idx_audit_app_user ON audit_log(app_user_id, event_time DESC);
```

### Trigger Audit Tự Động

```sql
-- Hàm audit generic:
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (
            table_name, operation, user_name,
            app_user_id, new_data, client_ip
        ) VALUES (
            TG_TABLE_NAME, 'INSERT', current_user,
            current_setting('app.current_user_id', true)::integer,
            row_to_json(NEW)::jsonb,
            inet_client_addr()
        );
        RETURN NEW;

    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (
            table_name, operation, user_name,
            app_user_id, old_data, new_data, changed_fields, client_ip
        ) VALUES (
            TG_TABLE_NAME, 'UPDATE', current_user,
            current_setting('app.current_user_id', true)::integer,
            row_to_json(OLD)::jsonb,
            row_to_json(NEW)::jsonb,
            -- Chỉ log các cột thực sự thay đổi:
            ARRAY(
                SELECT key
                FROM jsonb_each(row_to_json(NEW)::jsonb)
                WHERE value IS DISTINCT FROM (row_to_json(OLD)::jsonb)->key
            ),
            inet_client_addr()
        );
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (
            table_name, operation, user_name,
            app_user_id, old_data, client_ip
        ) VALUES (
            TG_TABLE_NAME, 'DELETE', current_user,
            current_setting('app.current_user_id', true)::integer,
            row_to_json(OLD)::jsonb,
            inet_client_addr()
        );
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Gán trigger cho bảng nhạy cảm:
CREATE TRIGGER audit_users
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();

CREATE TRIGGER audit_payments
    AFTER INSERT OR UPDATE OR DELETE ON payments
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

---

## Truy Vấn Audit Log

```sql
-- Tìm ai đã sửa user cụ thể:
SELECT
    event_time,
    operation,
    user_name,
    app_user_id,
    changed_fields,
    old_data,
    new_data
FROM audit_log
WHERE table_name = 'users'
  AND (old_data->>'id' = '123' OR new_data->>'id' = '123')
ORDER BY event_time DESC;

-- Tất cả thay đổi trong 24 giờ qua:
SELECT
    event_time,
    table_name,
    operation,
    user_name,
    app_user_id
FROM audit_log
WHERE event_time > NOW() - INTERVAL '24 hours'
ORDER BY event_time DESC;

-- Tìm DELETE bất thường (nhiều xóa trong thời gian ngắn):
SELECT
    DATE_TRUNC('minute', event_time) AS minute,
    user_name,
    COUNT(*) AS delete_count
FROM audit_log
WHERE operation = 'DELETE'
  AND event_time > NOW() - INTERVAL '1 hour'
GROUP BY 1, 2
HAVING COUNT(*) > 100
ORDER BY delete_count DESC;

-- Điều tra sau sự cố: Ai đã truy cập SSN?
SELECT
    event_time,
    user_name,
    app_user_id,
    client_ip,
    operation
FROM audit_log
WHERE table_name = 'users'
  AND operation = 'SELECT'  -- Cần pgaudit cho SELECT
ORDER BY event_time DESC
LIMIT 100;
```

---

## Bảo Vệ Audit Log

```sql
-- Audit log phải bất biến: Không được xóa/sửa!

-- Cách 1: Chỉ cấp INSERT, không cấp UPDATE/DELETE
REVOKE ALL ON audit_log FROM app_readwrite;
GRANT INSERT ON audit_log TO app_readwrite;

-- Cách 2: Dùng append-only schema riêng
CREATE SCHEMA audit;
ALTER TABLE audit_log SET SCHEMA audit;

GRANT USAGE ON SCHEMA audit TO app_readwrite;
GRANT INSERT ON audit.audit_log TO app_readwrite;
-- Không có UPDATE/DELETE permission

-- Cách 3: Row Security Policy
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;
CREATE POLICY audit_insert_only ON audit_log
    FOR INSERT WITH CHECK (true);
CREATE POLICY audit_select_all ON audit_log
    FOR SELECT USING (current_user IN ('audit_reader', 'admin'));
-- Không có UPDATE/DELETE policy → Không ai xóa được

-- Cách 4: Ship logs ra ngoài (tốt nhất)
-- Gửi logs đến SIEM/Elasticsearch ngay lập tức
-- Kẻ tấn công compromise DB không xóa được logs bên ngoài
```

---

## Tích Hợp SIEM

```bash
# Gửi PostgreSQL logs đến Elasticsearch (ELK Stack):

# Cài filebeat:
# filebeat.yml:
filebeat.inputs:
- type: log
  paths:
    - /var/log/postgresql/postgresql.log
  multiline.pattern: '^\d{4}-\d{2}-\d{2}'
  multiline.negate: true
  multiline.match: after

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "postgresql-audit-%{+YYYY.MM.dd}"

# Gửi đến Splunk:
# Cài Splunk Universal Forwarder
# Monitor /var/log/postgresql/

# Alert trong SIEM khi phát hiện:
# - Login thất bại liên tiếp (brute force)
# - DDL lúc ngoài giờ hành chính
# - Query lớn bất thường (data exfiltration)
# - Đăng nhập từ IP không quen
```

---

## Lưu Giữ Audit Log

```sql
-- Chính sách lưu giữ (ví dụ GDPR/PCI-DSS yêu cầu 1-7 năm):

-- Partition audit_log theo tháng để dễ archive:
CREATE TABLE audit_log (
    id BIGSERIAL,
    event_time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ...
) PARTITION BY RANGE (event_time);

-- Tạo partition hàng tháng:
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Archive partition cũ ra cold storage (S3/GCS):
-- pg_dump -t audit_log_2024_01 mydb | gzip > audit_2024_01.sql.gz
-- Upload to S3 với server-side encryption

-- Xóa partition đã archive (sau retention period):
-- DROP TABLE audit_log_2024_01;  -- Chỉ sau khi xác nhận đã archive!
```

---

## Checklist Audit Logging

**Cấu Hình:**
- [ ] pgaudit đã cài và bật cho DDL và WRITE
- [ ] pgaudit.role cấu hình cho SELECT audit trên bảng nhạy cảm
- [ ] log_connections = on (audit đăng nhập)
- [ ] log_disconnections = on (audit đăng xuất)

**Bảo Vệ Log:**
- [ ] Audit log append-only (không có UPDATE/DELETE)
- [ ] Audit log được ship ra SIEM ngay lập tức
- [ ] Logs không thể bị xóa bởi app account

**Nội Dung:**
- [ ] Timestamp với timezone
- [ ] Username (database và application)
- [ ] IP address của client
- [ ] Câu lệnh SQL (hoặc loại operation)
- [ ] Dữ liệu trước và sau thay đổi (cho trigger-based audit)

**Vận Hành:**
- [ ] Retention policy được thiết lập và tuân thủ
- [ ] Alert cho hành vi bất thường
- [ ] Quy trình review audit log định kỳ
- [ ] Test restore từ audit log (có thể truy vấn được không?)

---

> **Điểm Mấu Chốt:** Audit log vô nghĩa nếu không ai đọc chúng. Thiết lập alert tự động cho hành vi bất thường và review định kỳ là phần quan trọng nhất của audit strategy.
