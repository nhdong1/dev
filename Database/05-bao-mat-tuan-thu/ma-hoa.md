# Mã Hóa CSDL

Bảo vệ dữ liệu nhạy cảm cả khi lưu trữ (at rest) lẫn khi truyền tải (in transit).

## Các Tầng Mã Hóa

```
Tầng 1: MÃ HÓA TRUYỀN TẢI (In Transit)
  - TLS giữa ứng dụng và CSDL
  - TLS giữa primary và replica
  - Ngăn man-in-the-middle attacks

Tầng 2: MÃ HÓA LƯU TRỮ (At Rest)
  - Encryption toàn bộ volume/disk
  - Ngăn đọc dữ liệu khi lấy disk vật lý

Tầng 3: MÃ HÓA CỘT (Column-level)
  - Chỉ encrypt dữ liệu thực sự nhạy cảm (SSN, thẻ tín dụng)
  - Kiểm soát chi tiết ai giải mã được
```

---

## Mã Hóa Truyền Tải (TLS)

### Cấu Hình PostgreSQL

```bash
# Kiểm tra TLS đang bật:
SHOW ssl;  -- Phải là 'on'
SHOW ssl_cert_file;
SHOW ssl_key_file;
SHOW ssl_ca_file;

# postgresql.conf:
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_min_protocol_version = 'TLSv1.2'  # Không dùng TLS 1.0/1.1!

# Chỉ cho phép cipher mạnh:
ssl_ciphers = 'HIGH:!aNULL:!MD5'
```

### Bắt Buộc TLS Cho Tất Cả Connections

```ini
# pg_hba.conf:

# Thay vì:
host    mydb    app_user    10.0.1.0/24    md5

# Dùng hostssl (chỉ cho phép SSL):
hostssl    mydb    app_user    10.0.1.0/24    md5

# Từ chối non-SSL:
hostnossl  all     all         all            reject
```

### Connection String Với TLS

```python
# Python với psycopg2:
import psycopg2

conn = psycopg2.connect(
    host="db.example.com",
    database="mydb",
    user="myapp_api",
    password="secret",
    sslmode="verify-full",     # Xác minh certificate
    sslrootcert="/etc/ssl/certs/ca.crt"
)

# sslmode options:
# disable    - Không dùng SSL (NGUY HIỂM!)
# allow      - SSL nếu server yêu cầu
# prefer     - Ưu tiên SSL (mặc định, KHÔNG ĐỦ AN TOÀN!)
# require    - Bắt buộc SSL nhưng không verify cert
# verify-ca  - Bắt buộc SSL + verify CA
# verify-full - Bắt buộc SSL + verify CA + verify hostname (TỐT NHẤT)
```

```bash
# Kiểm tra TLS từ psql:
psql "sslmode=verify-full sslrootcert=/etc/ssl/certs/ca.crt \
      host=db.example.com dbname=mydb user=myapp_api"

# Kiểm tra kết nối hiện tại dùng SSL không:
SELECT ssl, version FROM pg_stat_ssl WHERE pid = pg_backend_pid();
```

---

## Mã Hóa Lưu Trữ (At Rest)

### Encryption at Volume Level

```bash
# AWS: Bật encryption khi tạo RDS instance
aws rds create-db-instance \
    --db-instance-identifier mydb \
    --storage-encrypted \
    --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc-def

# Azure: Transparent Data Encryption (TDE) - bật mặc định
# GCP: Encryption at rest - bật mặc định với Google-managed keys

# On-premise: LUKS encryption
cryptsetup luksFormat /dev/sdb
cryptsetup luksOpen /dev/sdb encrypted_db
mkfs.ext4 /dev/mapper/encrypted_db
mount /dev/mapper/encrypted_db /var/lib/postgresql
```

### Kiểm Tra Encryption

```sql
-- Kiểm tra PostgreSQL data directory có được encrypt không:
-- (Phụ thuộc vào infrastructure, không thể check từ SQL)
-- Kiểm tra ở tầng OS/Cloud console

-- Kiểm tra backup được encrypt:
-- pg_dump output có thể encrypt với:
pg_dump mydb | gpg --symmetric --cipher-algo AES256 > backup.sql.gpg
```

---

## Mã Hóa Cột (Column-Level Encryption)

### pgcrypto Extension

```sql
-- Cài đặt:
CREATE EXTENSION pgcrypto;

-- Mã hóa đối xứng (AES):
-- Thêm cột encrypted:
ALTER TABLE users ADD COLUMN ssn_encrypted bytea;

-- Mã hóa khi ghi:
UPDATE users
SET ssn_encrypted = pgp_sym_encrypt(ssn, 'encryption_key_here')
WHERE ssn IS NOT NULL;

-- Bỏ cột gốc (sau khi verify):
ALTER TABLE users DROP COLUMN ssn;

-- Giải mã khi đọc:
SELECT
    id,
    name,
    pgp_sym_decrypt(ssn_encrypted, 'encryption_key_here') AS ssn
FROM users
WHERE id = 123;
```

### Mã Hóa Bất Đối Xứng (Public/Private Key)

```sql
-- Mã hóa với public key (bất kỳ ai có public key đều có thể mã hóa):
UPDATE users
SET ssn_encrypted = pgp_pub_encrypt(ssn, dearmor('-----BEGIN PGP PUBLIC KEY BLOCK-----
...
-----END PGP PUBLIC KEY BLOCK-----'))
WHERE ssn IS NOT NULL;

-- Giải mã chỉ với private key (chỉ DBA/authorized service mới có thể giải mã):
SELECT pgp_pub_decrypt(
    ssn_encrypted,
    dearmor('-----BEGIN PGP PRIVATE KEY BLOCK-----
...
-----END PGP PRIVATE KEY BLOCK-----'),
    'passphrase'
) AS ssn
FROM users
WHERE id = 123;

-- Ưu điểm: Application có thể encrypt nhưng không decrypt
-- → Database bị compromise không expose được data
```

### Hashing (Một Chiều) Cho Password

```sql
-- KHÔNG bao giờ lưu mật khẩu plain text!
-- KHÔNG dùng MD5/SHA1 đơn giản!

-- Dùng bcrypt hoặc scrypt:
-- Tạo hash:
SELECT crypt('user_password', gen_salt('bf', 12));
-- Output: $2a$12$... (bcrypt hash)

-- Xác minh mật khẩu:
SELECT crypt('input_password', stored_hash) = stored_hash AS is_valid
FROM users WHERE username = 'johndoe';

-- Hoặc dùng application-level (khuyến nghị):
-- Python: bcrypt.hashpw(password, bcrypt.gensalt(rounds=12))
-- Node.js: bcrypt.hash(password, 12)
```

---

## Quản Lý Khóa Mã Hóa

### Nguyên Tắc Cơ Bản

```
KHÔNG bao giờ:
❌ Hardcode encryption key trong code
❌ Lưu key trong cùng database với data được encrypt
❌ Lưu key trong source control
❌ Dùng cùng key cho tất cả môi trường

LUÔN LUÔN:
✓ Lưu key trong Key Management Service (KMS/HSM)
✓ Key khác nhau cho dev/staging/production
✓ Rotate key định kỳ
✓ Audit mọi truy cập vào key
```

### AWS KMS Integration

```python
import boto3
import base64

kms_client = boto3.client('kms', region_name='us-east-1')
KEY_ID = 'arn:aws:kms:us-east-1:123456789:key/abc-def-ghi'

def encrypt_ssn(ssn: str) -> str:
    response = kms_client.encrypt(
        KeyId=KEY_ID,
        Plaintext=ssn.encode('utf-8')
    )
    return base64.b64encode(response['CiphertextBlob']).decode('utf-8')

def decrypt_ssn(encrypted_ssn: str) -> str:
    ciphertext = base64.b64decode(encrypted_ssn)
    response = kms_client.decrypt(
        CiphertextBlob=ciphertext,
        KeyId=KEY_ID
    )
    return response['Plaintext'].decode('utf-8')

# Dùng trong ứng dụng:
# encrypted = encrypt_ssn("123-45-6789")
# db.execute("INSERT INTO users (ssn_encrypted) VALUES (%s)", [encrypted])
```

### Envelope Encryption

```
Nguyên tắc: Dùng hai tầng key

Data Encryption Key (DEK):
  - Encrypt dữ liệu thực tế
  - Khác nhau cho mỗi record hoặc batch
  - Được encrypt bởi KEK

Key Encryption Key (KEK):
  - Encrypt DEK
  - Lưu trong KMS/HSM
  - Hiếm khi rotate

Lợi ích:
  - Re-encrypt toàn bộ data chỉ cần rotate KEK (không cần decrypt/re-encrypt data)
  - Granular access: Một số users chỉ access một số DEK
```

```python
# Envelope encryption với AWS KMS:
def generate_data_key():
    response = kms_client.generate_data_key(
        KeyId=KEY_ID,
        KeySpec='AES_256'
    )
    return {
        'plaintext_key': response['Plaintext'],        # Dùng để encrypt
        'encrypted_key': response['CiphertextBlob']    # Lưu vào DB
    }

def encrypt_with_envelope(data: str) -> dict:
    key = generate_data_key()
    # Encrypt data với plaintext DEK
    encrypted_data = symmetric_encrypt(data, key['plaintext_key'])
    return {
        'encrypted_data': encrypted_data,
        'encrypted_key': base64.b64encode(key['encrypted_key']).decode()
    }
```

---

## Lịch Rotate Key

```
Thông tin đăng nhập CSDL: Mỗi 90 ngày
Chứng chỉ TLS: Mỗi 1 năm (tự động với Let's Encrypt: 90 ngày)
Data Encryption Key (DEK): Mỗi 1 năm
Key Encryption Key (KEK): Mỗi 2 năm
Backup encryption key: Mỗi 2 năm
```

---

## Checklist Mã Hóa

**Truyền Tải:**
- [ ] TLS 1.2+ cho tất cả kết nối CSDL
- [ ] sslmode=verify-full trong connection strings
- [ ] pg_hba.conf chỉ cho phép hostssl connections
- [ ] TLS cho replication giữa primary/replica
- [ ] Certificate được monitor (không để hết hạn)

**Lưu Trữ:**
- [ ] Disk/volume encryption được bật
- [ ] Backup được encrypt
- [ ] Snapshot/export được encrypt

**Column-level:**
- [ ] PII (SSN, số thẻ, v.v.) được encrypt tại cột
- [ ] Password được hash với bcrypt/scrypt (không MD5!)
- [ ] Encryption key KHÔNG lưu trong database

**Key Management:**
- [ ] Keys lưu trong KMS/HSM hoặc Vault
- [ ] Keys khác nhau theo môi trường
- [ ] Key rotation schedule được thiết lập
- [ ] Truy cập key được audit

---

> **Điểm Mấu Chốt:** Mã hóa không thay thế kiểm soát truy cập — đó là tầng bổ sung. Ngay cả khi kẻ tấn công lấy được file database, dữ liệu vẫn vô dụng nếu không có key.
