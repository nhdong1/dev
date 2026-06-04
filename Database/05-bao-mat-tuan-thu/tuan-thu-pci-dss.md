# Tuân Thủ PCI-DSS

Tiêu chuẩn bảo mật dữ liệu ngành thẻ thanh toán — bắt buộc cho mọi tổ chức xử lý dữ liệu thẻ.

## PCI-DSS Là Gì?

```
PCI-DSS (Payment Card Industry Data Security Standard):
- Được tạo bởi Visa, Mastercard, American Express, Discover, JCB
- Phiên bản hiện tại: PCI-DSS v4.0 (2022)
- Áp dụng cho: Mọi tổ chức lưu trữ, xử lý hoặc truyền dữ liệu thẻ

Cardholder Data (CHD):
- PAN (Primary Account Number): Số thẻ 16 chữ số
- Cardholder name
- Expiration date
- Service code

Sensitive Authentication Data (SAD - KHÔNG được lưu sau authorization):
- Full magnetic stripe data
- CVV/CVC/CAV (mã bảo mật)
- PIN
```

---

## Quy Tắc Cơ Bản: Không Lưu SAD

```sql
-- TUYỆT ĐỐI KHÔNG làm thế này:
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    card_number VARCHAR(16),      -- Vi phạm PCI-DSS! (PAN không được lưu plain)
    card_cvv VARCHAR(4),          -- Vi phạm nghiêm trọng! (SAD không được lưu)
    card_expiry DATE,             -- Có thể lưu nhưng cần xem xét
    cardholder_name VARCHAR(255), -- Có thể lưu nếu cần
    amount DECIMAL(10,2)
);

-- ĐÚNG: Tokenization
CREATE TABLE payments (
    id SERIAL PRIMARY KEY,
    payment_token VARCHAR(32),    -- Token thay cho số thẻ thực
    last_four CHAR(4),            -- Chỉ 4 số cuối (được phép)
    card_brand VARCHAR(20),       -- 'Visa', 'Mastercard', v.v.
    cardholder_name VARCHAR(255), -- Tùy use case
    amount DECIMAL(10,2)
);

-- Số thẻ thực chỉ lưu tại payment processor!
```

---

## Tokenization

### Nguyên Tắc

```
Thay vì lưu số thẻ thực:
4111 1111 1111 1111 → tok_abc123xyz789

Token:
- Không có giá trị nếu bị lấy cắp
- Ánh xạ đến số thẻ thực chỉ tại payment vault
- Merchant không bao giờ thấy số thẻ đầy đủ

Luồng thanh toán:
1. User nhập thẻ trên form (hosted fields hoặc iframe)
2. Thông tin thẻ gửi trực tiếp đến payment processor (KHÔNG qua server của bạn!)
3. Payment processor trả về token
4. Server nhận token, lưu token (không phải số thẻ)
5. Charge dùng token
```

### Tích Hợp Stripe

```python
import stripe
stripe.api_key = "sk_live_..."  # Lưu trong Vault, không hardcode!

# Frontend tạo payment method (Stripe Elements):
# const { paymentMethod } = await stripe.createPaymentMethod({
#     type: 'card',
#     card: cardElement,
# });
# Gửi paymentMethod.id (token) đến server

# Backend: Charge với token
def create_payment(user_id: int, payment_method_id: str, amount_cents: int):
    try:
        payment_intent = stripe.PaymentIntent.create(
            amount=amount_cents,
            currency='usd',
            payment_method=payment_method_id,
            confirm=True,
            customer=get_stripe_customer_id(user_id)
        )
        # Lưu vào database: CHỈ token và metadata
        db.execute("""
            INSERT INTO payments (user_id, payment_token, last_four, amount, status)
            VALUES (%s, %s, %s, %s, %s)
        """, [
            user_id,
            payment_method_id,
            payment_intent.payment_method.card.last4,
            amount_cents,
            payment_intent.status
        ])
        return payment_intent

    except stripe.error.CardError as e:
        # Log lỗi (KHÔNG log card details!)
        logger.error(f"Card error for user {user_id}: {e.code}")
        raise
```

---

## Truncation và Masking

```sql
-- PAN Truncation: Chỉ lưu 4-6 chữ số đầu và 4 số cuối:
-- 4111 1111 1111 1111 → 411111XXXXXX1111

-- Lưu trữ đúng cách:
CREATE TABLE payment_cards (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    token TEXT NOT NULL,          -- Token từ vault
    bin_prefix CHAR(6),           -- 6 chữ số đầu (Bank Identification Number)
    last_four CHAR(4),            -- 4 chữ số cuối
    card_brand TEXT,
    expiry_month INTEGER,
    expiry_year INTEGER,
    -- KHÔNG lưu CVV, không lưu full PAN!
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Function hiển thị masked card:
CREATE OR REPLACE FUNCTION mask_card_number(bin_prefix CHAR(6), last_four CHAR(4))
RETURNS TEXT AS $$
BEGIN
    RETURN bin_prefix || 'XXXXXX' || last_four;
    -- Output: 411111XXXXXX1111
END;
$$ LANGUAGE plpgsql;
```

---

## 12 Yêu Cầu PCI-DSS v4.0

### Nhóm 1: Build and Maintain Secure Network

```bash
# Requirement 1: Cài đặt và maintain network security controls

# Firewall rules:
iptables -A INPUT -p tcp --dport 5432 -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 5432 -j DROP

# Requirement 2: Apply secure configurations
# Thay đổi default passwords, disable unnecessary services
ALTER ROLE postgres PASSWORD 'new_strong_password';
```

### Nhóm 2: Protect Cardholder Data

```sql
-- Requirement 3: Protect stored account data
-- Không lưu SAD sau authorization

-- Kiểm tra có CHD nào bị lưu không đúng cách:
-- Tìm patterns giống số thẻ trong database:
SELECT tablename, attname
FROM pg_attribute a
JOIN pg_class c ON a.attrelid = c.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE n.nspname = 'public'
  AND a.atttypid IN (25, 1043)  -- text, varchar
  AND a.attname ILIKE '%card%' OR a.attname ILIKE '%pan%'
     OR a.attname ILIKE '%credit%';

-- Requirement 4: Protect cardholder data with strong cryptography during transmission
-- TLS cho tất cả kết nối (đã cover trong bao-mat-mang.md)
```

### Nhóm 3: Maintain Vulnerability Management

```bash
# Requirement 5: Protect all systems from malicious software
# OS patching, antivirus

# Requirement 6: Develop and maintain secure systems and software
# Penetration testing, vulnerability scanning
# Quarterly external vulnerability scan
nmap -sV --script ssl-enum-ciphers db.internal.example.com

# Code review cho SQL injection:
# Không bao giờ concatenate user input vào SQL!
# LUÔN dùng parameterized queries
```

### Nhóm 4: Implement Strong Access Control

```sql
-- Requirement 7: Restrict access to cardholder data by business need to know
CREATE ROLE payment_readonly;
GRANT SELECT (id, payment_token, last_four, card_brand, amount, status)
ON payments TO payment_readonly;
-- KHÔNG cấp SELECT trên toàn bộ bảng nếu có cột nhạy cảm

-- Requirement 8: Identify and authenticate access to system components
-- Unique IDs, không shared accounts
-- MFA cho admin access

-- Requirement 9: Restrict physical access to cardholder data
-- Physical security cho data centers (ngoài phạm vi database)
```

### Nhóm 5: Regularly Monitor and Test Networks

```sql
-- Requirement 10: Log and monitor all access to network resources and cardholder data
-- Audit logging (đã cover trong ghi-nhat-ky-kiem-tra.md)

-- Requirement 10.6: Retain audit logs for at least 12 months
-- 3 tháng gần nhất phải immediately available

CREATE TABLE pci_audit_log (
    id BIGSERIAL PRIMARY KEY,
    event_time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id TEXT NOT NULL,
    action TEXT NOT NULL,
    resource TEXT NOT NULL,
    outcome TEXT NOT NULL,     -- 'success', 'failure'
    client_ip INET,
    details JSONB
) PARTITION BY RANGE (event_time);

-- Requirement 11: Test security systems regularly
-- Quarterly internal/external vulnerability scan
-- Annual penetration test
```

### Nhóm 6: Maintain Information Security Policy

```
-- Requirement 12: Support information security with policies and programs
-- Written security policies
-- Annual risk assessment
-- Security awareness training
```

---

## Scope Reduction

```
Mục tiêu: Giảm "Cardholder Data Environment" (CDE) scope

Kỹ thuật:
1. Tokenization: Số thẻ chỉ tồn tại tại payment processor
   → Hầu hết systems out of scope

2. Hosted payment page: Checkout page do payment processor host
   → Payment page hoàn toàn out of scope

3. Network segmentation: Isolate CDE khỏi phần còn lại
   → Giảm scope audit

Kết quả: Ít systems cần audit → Chi phí compliance thấp hơn
```

---

## SAQ (Self-Assessment Questionnaire)

```
SAQ types (theo mức độ xử lý):

SAQ A: Chỉ dùng hosted payment page / tokenization
  → Đơn giản nhất, ~22 requirements

SAQ A-EP: E-commerce với JavaScript-based payment
  → Phức tạp hơn

SAQ D: Lưu, xử lý hoặc truyền CHD
  → Đầy đủ 300+ requirements

Mục tiêu: Luôn cố gắng đạt SAQ A
  → Tokenization + Hosted payment page
```

---

## QSA Audit Preparation

```sql
-- Chuẩn bị cho PCI audit:

-- 1. Document tất cả systems trong CDE:
SELECT
    current_database() AS database_name,
    version() AS pg_version,
    inet_server_addr() AS server_ip,
    current_setting('listen_addresses') AS listen_addresses,
    current_setting('ssl') AS ssl_enabled;

-- 2. Chứng minh không có CHD không đúng cách:
-- Chạy PAN scan tools:
-- - Braintree PAN finder
-- - Spirent CardView
-- - Custom regex scan

-- Regex tìm PAN trong database (chạy và chứng minh "not found"):
-- Pattern Luhn-valid 13-19 digit numbers
-- (Thực hiện bởi specialized tools, không phải simple regex)

-- 3. Chứng minh access control đúng:
SELECT grantee, privilege_type, table_name
FROM information_schema.role_table_grants
WHERE table_name = 'payments'
ORDER BY grantee;
```

---

## Checklist PCI-DSS

**Dữ Liệu:**
- [ ] KHÔNG lưu CVV/CVC/SAD sau authorization
- [ ] PAN được truncate hoặc tokenize
- [ ] Tokenization được triển khai
- [ ] Không có CHD trong logs, error messages

**Mã Hóa:**
- [ ] TLS 1.2+ cho tất cả truyền tải CHD
- [ ] PAN được mã hóa nếu phải lưu
- [ ] Key management process được document

**Truy Cập:**
- [ ] Quyền tối thiểu cho CDE systems
- [ ] MFA cho tất cả non-console admin access
- [ ] Unique IDs cho mỗi user (không shared accounts)

**Monitoring:**
- [ ] Audit logs cho tất cả truy cập CHD
- [ ] Logs giữ 12 tháng (3 tháng immediate access)
- [ ] Alert cho hành vi bất thường
- [ ] Quarterly vulnerability scan

**Testing:**
- [ ] Annual penetration test
- [ ] Quarterly vulnerability scan
- [ ] File integrity monitoring cho CDE

---

> **Điểm Mấu Chốt:** Cách tốt nhất để tuân thủ PCI-DSS là KHÔNG lưu dữ liệu thẻ. Dùng tokenization và hosted payment pages để đưa hầu hết systems ra ngoài scope CDE. Ít scope = Ít compliance burden = Ít rủi ro.
