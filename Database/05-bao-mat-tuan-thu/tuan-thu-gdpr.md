# Tuân Thủ GDPR

Quy định bảo vệ dữ liệu cá nhân của EU — áp dụng cho mọi tổ chức xử lý dữ liệu công dân EU.

## GDPR Là Gì?

```
GDPR (General Data Protection Regulation):
- Có hiệu lực từ tháng 5/2018
- Áp dụng cho mọi tổ chức xử lý dữ liệu cá nhân của công dân EU
- Phạt tối đa: €20 triệu hoặc 4% doanh thu toàn cầu

Các khái niệm chính:
- PII (Personally Identifiable Information): Tên, email, địa chỉ, IP, v.v.
- Data Subject: Người dùng có quyền về dữ liệu của họ
- Data Controller: Tổ chức quyết định mục đích xử lý data
- Data Processor: Tổ chức xử lý data thay cho controller (vendor)
- Legal Basis: Lý do hợp pháp để xử lý data (consent, contract, legal obligation...)
```

---

## Kiểm Kê Dữ Liệu (Data Inventory)

### Xác Định PII Trong Database

```sql
-- Gắn tag vào bảng và cột chứa PII:
COMMENT ON TABLE users IS 'GDPR: Chứa PII - email, tên, địa chỉ';
COMMENT ON COLUMN users.email IS 'GDPR: PII - dữ liệu liên lạc';
COMMENT ON COLUMN users.name IS 'GDPR: PII - danh tính';
COMMENT ON COLUMN users.phone IS 'GDPR: PII - dữ liệu liên lạc';
COMMENT ON COLUMN users.ip_address IS 'GDPR: PII - nhận dạng kỹ thuật số';
COMMENT ON COLUMN users.ssn IS 'GDPR: PII Nhạy cảm - số an sinh xã hội';

-- Xem tất cả PII columns:
SELECT
    c.table_name,
    c.column_name,
    pgd.description AS gdpr_classification
FROM information_schema.columns c
JOIN pg_description pgd ON pgd.objoid = (
    SELECT oid FROM pg_class WHERE relname = c.table_name
)
AND pgd.objsubid = c.ordinal_position
WHERE pgd.description LIKE 'GDPR%'
ORDER BY c.table_name, c.column_name;
```

### Data Map

```sql
-- Tạo bảng data map:
CREATE TABLE gdpr_data_map (
    id SERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    column_name TEXT NOT NULL,
    data_category TEXT NOT NULL,        -- 'contact', 'identity', 'financial', 'sensitive'
    legal_basis TEXT NOT NULL,          -- 'consent', 'contract', 'legal_obligation'
    retention_days INTEGER NOT NULL,
    encrypted BOOLEAN DEFAULT false,
    notes TEXT
);

INSERT INTO gdpr_data_map VALUES
    (DEFAULT, 'users', 'email', 'contact', 'contract', 2555, true, 'Cần để đăng nhập'),
    (DEFAULT, 'users', 'name', 'identity', 'contract', 2555, false, NULL),
    (DEFAULT, 'users', 'phone', 'contact', 'consent', 365, false, 'Opt-in only'),
    (DEFAULT, 'orders', 'shipping_address', 'contact', 'contract', 2555, false, 'Cần cho giao hàng'),
    (DEFAULT, 'analytics', 'ip_address', 'technical', 'legitimate_interest', 90, false, 'Xóa sau 90 ngày');
```

---

## Quyền Của Data Subject

### 1. Right to Access (Quyền Truy Cập)

```sql
-- User có quyền biết tất cả data ta có về họ:
CREATE OR REPLACE FUNCTION get_user_data_export(p_user_id INTEGER)
RETURNS JSONB AS $$
BEGIN
    RETURN jsonb_build_object(
        'user', (SELECT row_to_json(u) FROM (
            SELECT id, name, email, phone, created_at, updated_at
            FROM users WHERE id = p_user_id
        ) u),
        'orders', (SELECT jsonb_agg(row_to_json(o)) FROM (
            SELECT id, created_at, total, status, shipping_address
            FROM orders WHERE user_id = p_user_id
        ) o),
        'reviews', (SELECT jsonb_agg(row_to_json(r)) FROM (
            SELECT id, created_at, rating, comment
            FROM reviews WHERE user_id = p_user_id
        ) r),
        'export_date', NOW()
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Gọi khi user request data export:
SELECT get_user_data_export(123);
```

### 2. Right to Erasure (Quyền Xóa)

```sql
-- Xóa PII khi user yêu cầu (GDPR "Right to be Forgotten"):
CREATE OR REPLACE PROCEDURE delete_user_pii(p_user_id INTEGER)
LANGUAGE plpgsql AS $$
DECLARE
    v_deleted_at TIMESTAMPTZ := NOW();
BEGIN
    -- 1. Xóa dữ liệu nhạy cảm trực tiếp:
    UPDATE users SET
        email = 'deleted_' || p_user_id || '@deleted.invalid',
        name = 'Deleted User',
        phone = NULL,
        address = NULL,
        ssn_encrypted = NULL,
        deleted_at = v_deleted_at
    WHERE id = p_user_id;

    -- 2. Xóa dữ liệu không cần thiết trong orders:
    UPDATE orders SET
        shipping_address = '[Deleted]',
        billing_address = '[Deleted]'
    WHERE user_id = p_user_id;

    -- 3. Xóa sessions:
    DELETE FROM user_sessions WHERE user_id = p_user_id;

    -- 4. Xóa analytics với PII:
    DELETE FROM user_analytics WHERE user_id = p_user_id;

    -- 5. Log việc xóa (để chứng minh tuân thủ):
    INSERT INTO gdpr_deletion_log (user_id, deleted_at, requested_by)
    VALUES (p_user_id, v_deleted_at, current_user);

    RAISE NOTICE 'PII deleted for user %', p_user_id;
END;
$$;

-- Gọi khi nhận được yêu cầu:
CALL delete_user_pii(123);
```

### 3. Right to Portability (Quyền Chuyển Đổi)

```sql
-- Export data ở format chuẩn (JSON/CSV):
COPY (
    SELECT
        u.id,
        u.name,
        u.email,
        u.created_at,
        json_agg(json_build_object(
            'order_id', o.id,
            'date', o.created_at,
            'amount', o.total
        )) AS orders
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE u.id = 123
    GROUP BY u.id, u.name, u.email, u.created_at
) TO '/tmp/user_123_export.json' (FORMAT CSV);
```

### 4. Right to Rectification (Quyền Chỉnh Sửa)

```sql
-- User có thể yêu cầu sửa data không chính xác:
-- Đây chỉ là logic bình thường của ứng dụng
-- Quan trọng: Log mọi thay đổi PII (audit trail)

UPDATE users SET
    name = 'John Smith',
    phone = '+1234567890'
WHERE id = 123;

-- Trigger audit sẽ tự động log thay đổi này
```

---

## Chính Sách Lưu Giữ Dữ Liệu (Data Retention)

```sql
-- Bảng lưu retention policy:
CREATE TABLE data_retention_policies (
    table_name TEXT PRIMARY KEY,
    retention_days INTEGER NOT NULL,
    deletion_strategy TEXT NOT NULL,  -- 'hard_delete', 'anonymize', 'archive'
    last_run TIMESTAMPTZ
);

INSERT INTO data_retention_policies VALUES
    ('analytics_events', 90, 'hard_delete', NULL),
    ('user_sessions', 30, 'hard_delete', NULL),
    ('email_logs', 365, 'anonymize', NULL),
    ('deleted_users', 2555, 'hard_delete', NULL);  -- 7 năm

-- Job xóa dữ liệu hết hạn (chạy hàng ngày):
CREATE OR REPLACE PROCEDURE run_data_retention()
LANGUAGE plpgsql AS $$
DECLARE
    policy RECORD;
BEGIN
    FOR policy IN SELECT * FROM data_retention_policies LOOP
        IF policy.deletion_strategy = 'hard_delete' THEN
            EXECUTE format(
                'DELETE FROM %I WHERE created_at < NOW() - INTERVAL ''%s days''',
                policy.table_name,
                policy.retention_days
            );
        ELSIF policy.deletion_strategy = 'anonymize' THEN
            EXECUTE format(
                'UPDATE %I SET user_id = NULL, ip_address = NULL
                 WHERE created_at < NOW() - INTERVAL ''%s days''
                 AND user_id IS NOT NULL',
                policy.table_name,
                policy.retention_days
            );
        END IF;

        UPDATE data_retention_policies
        SET last_run = NOW()
        WHERE table_name = policy.table_name;

        RAISE NOTICE 'Retention job completed for %', policy.table_name;
    END LOOP;
END;
$$;

-- Cron job hàng ngày:
-- 0 2 * * * psql -d mydb -c "CALL run_data_retention();"
```

---

## Xử Lý Backup Với GDPR

```
Thách thức: Khi user yêu cầu xóa, data vẫn tồn tại trong backup!

Giải pháp:
1. Anonymize trong production → Backup chỉ chứa data đã ẩn danh
   (Tốt nhất nhưng phức tạp nếu cần restore)

2. Xóa backup cũ sau retention period
   (Backup < 90 ngày → Không vấn đề nếu retention = 90 ngày)

3. Legal Hold: Một số backup có thể giữ lâu hơn vì lý do pháp lý
   (Tranh chấp hợp đồng, điều tra tội phạm)
   → Document rõ ràng, giữ riêng biệt

4. Backup Encryption + Key Rotation:
   Mã hóa backup với key riêng theo khoảng thời gian
   → Xóa key = Data không thể decrypt (crypto-shredding)
```

```sql
-- Log backup retention:
CREATE TABLE backup_retention_log (
    id SERIAL PRIMARY KEY,
    backup_date DATE NOT NULL,
    backup_type TEXT NOT NULL,  -- 'full', 'incremental'
    location TEXT NOT NULL,     -- S3 path
    encryption_key_id TEXT,
    scheduled_deletion DATE NOT NULL,
    deleted_at TIMESTAMPTZ
);

-- Alert backup sắp đến ngày xóa:
SELECT *
FROM backup_retention_log
WHERE scheduled_deletion <= CURRENT_DATE + 7
  AND deleted_at IS NULL;
```

---

## Tuân Thủ Với Vendor (Data Processors)

```sql
-- Tracking vendor compliance:
CREATE TABLE vendor_data_agreements (
    vendor_name TEXT PRIMARY KEY,
    agreement_type TEXT,       -- 'DPA', 'SCCs'
    signed_date DATE,
    data_shared JSONB,         -- Loại data được share
    retention_days INTEGER,
    sub_processors JSONB,      -- Vendor của vendor
    review_date DATE
);

-- Kiểm tra vendor nào hết hạn review:
SELECT vendor_name, review_date, CURRENT_DATE - review_date AS days_overdue
FROM vendor_data_agreements
WHERE review_date < CURRENT_DATE
ORDER BY days_overdue DESC;
```

---

## Thông Báo Vi Phạm (Breach Notification)

```
GDPR yêu cầu: Thông báo trong 72 giờ nếu có vi phạm dữ liệu!

Quy trình:
T+0: Phát hiện vi phạm
T+1h: Điều tra ban đầu (scope là gì?)
T+4h: Quyết định có phải notify không
T+24h: Notify Data Protection Officer (DPO)
T+72h: Notify cơ quan quản lý (DPA)
T+72h+: Notify data subjects nếu cần

Phải document:
- Bản chất của vi phạm
- Loại và số lượng records bị ảnh hưởng
- Hậu quả có thể xảy ra
- Biện pháp khắc phục
```

```sql
-- Bảng incident tracking:
CREATE TABLE security_incidents (
    id SERIAL PRIMARY KEY,
    detected_at TIMESTAMPTZ NOT NULL,
    description TEXT NOT NULL,
    affected_records_count INTEGER,
    affected_data_types TEXT[],
    severity TEXT,              -- 'low', 'medium', 'high', 'critical'
    notified_dpo_at TIMESTAMPTZ,
    notified_dpa_at TIMESTAMPTZ,
    notified_users_at TIMESTAMPTZ,
    resolution_at TIMESTAMPTZ,
    root_cause TEXT,
    preventive_measures TEXT
);
```

---

## Checklist GDPR

**Kiểm Kê:**
- [ ] Tất cả PII được xác định và gắn tag
- [ ] Data map được tạo và cập nhật
- [ ] Legal basis cho mỗi loại data được document
- [ ] Vendor DPA được ký kết

**Quyền Data Subject:**
- [ ] Right to Access: Có thể export toàn bộ data của user
- [ ] Right to Erasure: Có quy trình xóa/ẩn danh hóa
- [ ] Right to Portability: Export ở format chuẩn
- [ ] Right to Rectification: Quy trình update data

**Kỹ Thuật:**
- [ ] PII được mã hóa (cột nhạy cảm)
- [ ] Retention policy được thiết lập và tự động hóa
- [ ] Data Minimization: Không thu thập data không cần thiết
- [ ] Pseudonymization/Anonymization được áp dụng

**Vận Hành:**
- [ ] DPO được chỉ định
- [ ] Quy trình breach notification < 72 giờ
- [ ] Training nhân viên về GDPR
- [ ] DPIA (Data Protection Impact Assessment) cho xử lý rủi ro cao

---

> **Điểm Mấu Chốt:** GDPR không chỉ là kỹ thuật — đó là văn hóa tổ chức. Kỹ thuật đúng (mã hóa, retention, audit) là cần thiết nhưng không đủ nếu không có quy trình và trách nhiệm rõ ràng.
