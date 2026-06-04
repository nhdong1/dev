# Kiểm Soát Truy Cập CSDL

Nguyên tắc quyền tối thiểu, phân tách vai trò và bảo vệ dữ liệu theo từng hàng.

## Nguyên Tắc Quyền Tối Thiểu (Least Privilege)

```
Mỗi tài khoản chỉ được cấp đúng quyền cần thiết để thực hiện công việc.

Sai lầm phổ biến:
❌ Ứng dụng dùng tài khoản superuser/root
❌ Nhiều ứng dụng dùng chung một tài khoản CSDL
❌ Dev/staging dùng cùng thông tin đăng nhập với production

Đúng đắn:
✓ Mỗi ứng dụng có tài khoản riêng biệt
✓ Read-only service chỉ có quyền SELECT
✓ Background jobs chỉ có quyền INSERT/UPDATE trên bảng cụ thể
```

---

## Thiết Kế Role PostgreSQL

### Phân Tầng Role

```sql
-- Tầng 1: Base roles (không có LOGIN)
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;
CREATE ROLE app_admin;

-- Tầng 2: Service accounts (có LOGIN, được cấp base roles)
CREATE ROLE myapp_api WITH LOGIN PASSWORD 'strong_password_1';
CREATE ROLE myapp_worker WITH LOGIN PASSWORD 'strong_password_2';
CREATE ROLE myapp_reports WITH LOGIN PASSWORD 'strong_password_3';

-- Gán base roles cho service accounts:
GRANT app_readwrite TO myapp_api;
GRANT app_readwrite TO myapp_worker;
GRANT app_readonly TO myapp_reports;
```

### Cấp Quyền Theo Role

```sql
-- app_readonly: Chỉ đọc
GRANT CONNECT ON DATABASE mydb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;

-- Tự động cấp SELECT cho bảng mới:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO app_readonly;

-- app_readwrite: Đọc + Ghi
GRANT CONNECT ON DATABASE mydb TO app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;

-- app_admin: Đọc + Ghi + DDL (chỉ dùng cho migration)
GRANT app_readwrite TO app_admin;
GRANT CREATE ON SCHEMA public TO app_admin;
-- KHÔNG cấp SUPERUSER!
```

### Kiểm Tra Quyền Hiện Tại

```sql
-- Xem quyền của user:
\du myapp_api   -- Trong psql

-- Hoặc:
SELECT
    rolname,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolcanlogin,
    rolreplication
FROM pg_roles
WHERE rolname LIKE 'myapp%';

-- Xem quyền trên bảng:
SELECT
    grantee,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE grantee IN ('app_readonly', 'app_readwrite')
ORDER BY table_name, privilege_type;
```

---

## Row-Level Security (RLS)

### Khi Nào Dùng RLS?

```
Use cases điển hình:
- Multi-tenant SaaS: Mỗi tenant chỉ thấy data của mình
- HR system: Manager chỉ thấy nhân viên thuộc team
- Compliance: Chỉ một số role được thấy PII
```

### Thiết Lập RLS

```sql
-- Bật RLS trên bảng:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: Users chỉ thấy orders của mình
CREATE POLICY orders_user_isolation ON orders
    USING (user_id = current_setting('app.current_user_id')::integer);

-- Ứng dụng set context trước khi truy vấn:
SET app.current_user_id = 123;
SELECT * FROM orders;  -- Chỉ trả về orders của user 123

-- Thay thế với app user thực:
CREATE POLICY orders_role_isolation ON orders
    USING (user_id = (
        SELECT id FROM users WHERE username = current_user
    ));
```

### RLS Cho Multi-Tenant

```sql
-- Bảng với tenant_id:
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    name VARCHAR(255),
    price DECIMAL(10,2)
);

ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Policy tách biệt theo tenant:
CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- Policy riêng cho superadmin (bypass RLS):
CREATE POLICY admin_all_access ON products
    USING (current_user = 'superadmin');

-- Bypass RLS cho superuser (mặc định):
ALTER TABLE products FORCE ROW LEVEL SECURITY;  -- Kể cả table owner
```

### Kiểm Tra RLS

```sql
-- Xem các policy:
SELECT schemaname, tablename, policyname, cmd, qual
FROM pg_policies
WHERE tablename = 'orders';

-- Test policy:
SET ROLE myapp_api;
SET app.current_user_id = 123;
SELECT COUNT(*) FROM orders;  -- Chỉ đếm orders của user 123

RESET ROLE;
```

---

## Phân Tách Schema (Schema Separation)

```sql
-- Tách schema theo chức năng:
CREATE SCHEMA app;      -- Bảng ứng dụng
CREATE SCHEMA reports;  -- Views báo cáo (read-only)
CREATE SCHEMA audit;    -- Bảng audit log (append-only)

-- Ứng dụng chỉ truy cập schema app:
GRANT USAGE ON SCHEMA app TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_readwrite;

-- Reporting service chỉ truy cập schema reports:
GRANT USAGE ON SCHEMA reports TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA reports TO app_readonly;

-- Schema audit: Chỉ INSERT, không UPDATE/DELETE
GRANT USAGE ON SCHEMA audit TO app_readwrite;
GRANT INSERT ON ALL TABLES IN SCHEMA audit TO app_readwrite;
-- KHÔNG cấp UPDATE/DELETE → Audit log không thể bị xóa!
```

---

## Xem Xét Truy Cập Định Kỳ (Access Review)

```sql
-- Script kiểm tra: Ai có quyền gì?
SELECT
    r.rolname AS username,
    r.rolsuper AS is_superuser,
    r.rolcanlogin AS can_login,
    r.rolvaliduntil AS password_expires,
    ARRAY_AGG(m.rolname) AS member_of_roles
FROM pg_roles r
LEFT JOIN pg_auth_members am ON r.oid = am.member
LEFT JOIN pg_roles m ON am.roleid = m.oid
WHERE r.rolcanlogin = true
GROUP BY r.rolname, r.rolsuper, r.rolcanlogin, r.rolvaliduntil
ORDER BY r.rolname;

-- Kiểm tra accounts không có expire date:
SELECT rolname, rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true
  AND rolvaliduntil IS NULL;
-- Tất cả service accounts nên có password rotation policy!

-- Tìm accounts lâu không dùng:
SELECT
    usename,
    last_login,
    NOW() - last_login AS time_since_login
FROM pg_user_mappings;  -- Chỉ có trong một số phiên bản
-- Dùng audit log để tìm inactive accounts
```

---

## Bảo Vệ Superuser

```sql
-- Không bao giờ dùng superuser cho ứng dụng!
-- Superuser có thể:
-- - Bypass RLS
-- - Đọc tất cả dữ liệu
-- - Drop bảng/database
-- - Đọc pg_shadow (hash mật khẩu)

-- Tạo user superuser chỉ khi thực sự cần:
CREATE ROLE admin_dba WITH LOGIN SUPERUSER PASSWORD 'very_strong_password';

-- Giới hạn truy cập superuser:
-- pg_hba.conf:
-- host  mydb  admin_dba  10.0.2.50/32  md5
-- Chỉ cho phép từ jump host cụ thể

-- Bật MFA cho DBA access (qua jump host + VPN)
```

---

## Checklist Kiểm Soát Truy Cập

- [ ] Không có ứng dụng nào dùng superuser
- [ ] Mỗi ứng dụng/service có account riêng biệt
- [ ] Read-only services chỉ có quyền SELECT
- [ ] Password không dùng chung giữa môi trường
- [ ] Password có expiry date và được xoay vòng
- [ ] RLS được bật cho multi-tenant hoặc data có nhạy cảm
- [ ] Phân tách schema theo mức độ truy cập
- [ ] Access review hàng quý (kiểm tra ai có quyền gì)
- [ ] Accounts không dùng được vô hiệu hóa
- [ ] Superuser access chỉ qua jump host + VPN + MFA

---

> **Điểm Mấu Chốt:** Quyền tối thiểu không chỉ là best practice — nó là tuyến phòng thủ quan trọng khi bị tấn công. Nếu application account bị compromise, kẻ tấn công chỉ có thể làm những gì account đó được phép.
