# Quản Lý Bí Mật (Secret Management)

Quản lý credentials, API keys và encryption keys một cách an toàn, có kiểm soát và tự động.

## Vấn Đề Với Cách Truyền Thống

```
Cách xấu phổ biến:

1. Hardcode trong code:
   db_password = "MySecret123!"  ← Lộ khi push git!

2. Lưu trong config file:
   database.yml: password: "MySecret123!"  ← Trong repo!

3. Biến môi trường không bảo vệ:
   DB_PASSWORD=MySecret123!  ← Ai cũng thấy trong ps aux, logs

4. Dùng chung mật khẩu:
   Dev, staging, production dùng cùng password ← Sai!

Hậu quả:
- Credentials bị lộ trong git history
- Không biết ai có credentials
- Không thể rotate mà không downtime
- Không có audit trail
```

---

## HashiCorp Vault

### Cài Đặt và Khởi Tạo

```bash
# Cài Vault:
wget https://releases.hashicorp.com/vault/1.15.0/vault_1.15.0_linux_amd64.zip
unzip vault_1.15.0_linux_amd64.zip
mv vault /usr/local/bin/

# Khởi tạo (chỉ làm một lần):
vault operator init -key-shares=5 -key-threshold=3
# Tạo ra 5 unseal keys, cần 3 key để unseal

# Unseal Vault sau mỗi lần restart:
vault operator unseal KEY1
vault operator unseal KEY2
vault operator unseal KEY3

# Login:
vault login ROOT_TOKEN
```

### Lưu Database Credentials

```bash
# Kích hoạt KV secrets engine:
vault secrets enable -path=secret kv-v2

# Lưu credentials CSDL:
vault kv put secret/databases/production/mydb \
    username=myapp_api \
    password="GeneratedStrongPassword123!" \
    host=db.internal.example.com \
    port=5432 \
    dbname=mydb

# Lưu credentials theo môi trường:
vault kv put secret/databases/staging/mydb \
    username=myapp_api_staging \
    password="DifferentPassword456!" \
    host=staging-db.internal.example.com

# Đọc credentials:
vault kv get secret/databases/production/mydb

# Output:
# ====== Data ======
# Key        Value
# ---        -----
# dbname     mydb
# host       db.internal.example.com
# password   GeneratedStrongPassword123!
# port       5432
# username   myapp_api
```

### Dynamic Secrets (Credentials Tự Động Tạo)

```bash
# Vault có thể tự tạo PostgreSQL credentials tạm thời!

# Kích hoạt database secrets engine:
vault secrets enable database

# Cấu hình kết nối PostgreSQL:
vault write database/config/mydb \
    plugin_name=postgresql-database-plugin \
    allowed_roles="myapp-role" \
    connection_url="postgresql://{{username}}:{{password}}@db.internal.example.com/mydb?sslmode=verify-full" \
    username="vault_admin" \
    password="VaultAdminPass"

# Tạo role (định nghĩa quyền của credentials tạm thời):
vault write database/roles/myapp-role \
    db_name=mydb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' IN ROLE app_readwrite;" \
    default_ttl="1h" \
    max_ttl="24h"

# Ứng dụng request credentials:
vault read database/creds/myapp-role

# Output:
# Key                Value
# ---                -----
# lease_id           database/creds/myapp-role/AbCdEfGh
# lease_duration     1h
# lease_renewable    true
# password           A1s2d3f4-G5h6j7k8
# username           v-myapp-role-Abcdef-1234567890

# Credentials tự xóa sau 1 giờ!
# Nếu bị lộ → Revoke ngay:
vault lease revoke database/creds/myapp-role/AbCdEfGh
```

---

## Tích Hợp Vault Vào Ứng Dụng

### Python: Đọc Credentials Từ Vault

```python
import hvac
import psycopg2
import os

def get_db_connection():
    # Kết nối Vault
    vault_client = hvac.Client(
        url=os.environ['VAULT_ADDR'],    # https://vault.internal:8200
        token=os.environ['VAULT_TOKEN']  # App token (từ Vault AppRole)
    )

    # Lấy credentials (dynamic hoặc static)
    secret = vault_client.secrets.kv.v2.read_secret_version(
        path='databases/production/mydb'
    )
    creds = secret['data']['data']

    # Tạo connection
    return psycopg2.connect(
        host=creds['host'],
        port=creds['port'],
        database=creds['dbname'],
        user=creds['username'],
        password=creds['password'],
        sslmode='verify-full'
    )
```

### Vault Agent: Tự Động Renew Credentials

```hcl
# vault-agent.hcl
vault {
  address = "https://vault.internal:8200"
}

auto_auth {
  method "aws" {
    config = {
      role = "myapp-role"
    }
  }
}

template {
  source = "/etc/vault/templates/db.env.tpl"
  destination = "/etc/myapp/db.env"
  command = "systemctl restart myapp"  # Restart khi credentials mới
}
```

```
# db.env.tpl
{{- with secret "database/creds/myapp-role" -}}
DB_HOST=db.internal.example.com
DB_USER={{ .Data.username }}
DB_PASSWORD={{ .Data.password }}
{{- end -}}
```

---

## AWS Secrets Manager

```python
import boto3
import json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='us-east-1')

    secret_value = client.get_secret_value(
        SecretId='production/mydb/credentials'
    )

    secret = json.loads(secret_value['SecretString'])
    return secret  # {'username': '...', 'password': '...', 'host': '...'}

# Sử dụng:
creds = get_db_credentials()
conn = psycopg2.connect(
    host=creds['host'],
    database=creds['dbname'],
    user=creds['username'],
    password=creds['password']
)
```

```bash
# Lưu secret:
aws secretsmanager create-secret \
    --name "production/mydb/credentials" \
    --secret-string '{"username":"myapp_api","password":"StrongPass123!","host":"db.internal","dbname":"mydb"}'

# Tự động rotate mỗi 90 ngày:
aws secretsmanager rotate-secret \
    --secret-id "production/mydb/credentials" \
    --rotation-lambda-arn arn:aws:lambda:us-east-1:123:function:rotate-db-password \
    --rotation-rules AutomaticallyAfterDays=90
```

---

## Rotation Credentials CSDL

### Manual Rotation Procedure

```bash
#!/bin/bash
# rotate-db-password.sh

SECRET_NAME="production/mydb/credentials"
DB_USER="myapp_api"
DB_HOST="db.internal.example.com"

# 1. Generate new password
NEW_PASSWORD=$(openssl rand -base64 32)

# 2. Update password in PostgreSQL
psql -h $DB_HOST -U admin_user -c \
    "ALTER ROLE $DB_USER PASSWORD '$NEW_PASSWORD';"

# 3. Update secret in Vault/AWS Secrets Manager
vault kv put secret/databases/production/mydb \
    username=$DB_USER \
    password="$NEW_PASSWORD"

# 4. Reload application connections (graceful restart)
kubectl rollout restart deployment/myapp

echo "Password rotated successfully"
```

### Rotation Không Downtime

```sql
-- PostgreSQL hỗ trợ multiple passwords? Không trực tiếp.
-- Nhưng có thể dùng connection pool để rotate không downtime:

-- Bước 1: Tạo role mới với password mới
CREATE ROLE myapp_api_v2 WITH LOGIN PASSWORD 'NewPassword456!'
    IN ROLE app_readwrite;

-- Bước 2: Update ứng dụng để dùng myapp_api_v2
-- (Dần dần chuyển connections, không drop hết cùng lúc)

-- Bước 3: Khi tất cả connections đã chuyển, xóa role cũ
DROP ROLE myapp_api_v1;

-- Hoặc đơn giản hơn: Dùng Vault Dynamic Secrets
-- Credentials tự động expire → Rotate tự nhiên
```

---

## Kubernetes Secrets Integration

```yaml
# Dùng External Secrets Operator để sync từ Vault/AWS:
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp-role"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mydb-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: mydb-secret
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: databases/production/mydb
      property: password
  - secretKey: username
    remoteRef:
      key: databases/production/mydb
      property: username
```

---

## Lịch Xoay Vòng Bí Mật

```
Loại Bí Mật               | Tần Suất Rotate | Ghi Chú
--------------------------|-----------------|----------------------------------
Database passwords        | 90 ngày         | Tự động với Vault/Secrets Manager
API keys (external)       | 90 ngày         | Phụ thuộc vendor policy
TLS certificates          | 365 ngày        | Tự động với cert-manager
Encryption keys (DEK)     | 365 ngày        | Với envelope encryption
Encryption keys (KEK)     | 730 ngày        | Ảnh hưởng lớn, làm cẩn thận
Backup encryption keys    | 730 ngày        | Giữ key cũ để decrypt backup cũ!
Vault unseal keys         | Sau sự cố       | Hoặc khi nhân viên rời công ty
Root tokens               | Revoke ngay     | Không dùng trong vận hành thường
```

---

## Audit Secret Access

```bash
# Vault audit log:
vault audit enable file file_path=/var/log/vault/audit.log

# Mỗi lần đọc secret được log:
# {
#   "time": "2026-04-26T10:15:30Z",
#   "type": "response",
#   "request": {
#     "operation": "read",
#     "path": "secret/data/databases/production/mydb",
#     "remote_address": "10.0.1.10"
#   },
#   "auth": {
#     "client_token": "...",
#     "entity_id": "...",
#     "display_name": "myapp-api-server-1"
#   }
# }
```

---

## Checklist Quản Lý Bí Mật

**Không Được:**
- [ ] KHÔNG có credentials trong source code
- [ ] KHÔNG có credentials trong config files được commit
- [ ] KHÔNG có cùng credentials cho dev/staging/production
- [ ] KHÔNG có shared accounts giữa applications

**Thiết Lập:**
- [ ] Vault hoặc cloud secret manager được triển khai
- [ ] Dynamic secrets cho database credentials (nếu khả thi)
- [ ] Vault audit logging được bật
- [ ] App sử dụng AppRole hoặc cloud IAM (không hardcode Vault token)

**Vận Hành:**
- [ ] Rotation schedule được thiết lập và tự động hóa
- [ ] Alert khi secret sắp expire
- [ ] Quy trình emergency rotation được document và test
- [ ] Access review định kỳ (ai có thể đọc secret nào?)

**Ứng Phó Sự Cố:**
- [ ] Quy trình revoke ngay khi phát hiện credentials bị lộ
- [ ] List tất cả systems cần update khi rotate
- [ ] Runbook rotation không downtime

---

> **Điểm Mấu Chốt:** Secrets management là nền tảng của mọi chiến lược bảo mật. Nếu credentials bị lộ và không thể rotate nhanh, mọi biện pháp bảo mật khác đều vô nghĩa. Tự động hóa rotation ngay từ đầu — đừng chờ đến khi có sự cố.
