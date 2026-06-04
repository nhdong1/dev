# Bảo Mật & Mạng cho Cloud Database

## 1. Network Architecture — Layered Defense

### AWS — Network Topology Chuẩn

```
Internet
    │
    ▼
┌───────────────────────────────────────────────┐
│                  VPC (10.0.0.0/16)            │
│                                               │
│  ┌────────────────────────────────────────┐   │
│  │        Public Subnets (AZ-a, AZ-b)     │   │
│  │  ┌──────────────┐  ┌───────────────┐   │   │
│  │  │ ALB / NAT GW │  │ Bastion Host  │   │   │
│  │  └──────────────┘  └───────────────┘   │   │
│  └────────────────────────────────────────┘   │
│                      │                        │
│  ┌────────────────────────────────────────┐   │
│  │       Private Subnets (App Tier)        │   │
│  │  ┌──────────┐  ┌──────────┐            │   │
│  │  │ App ECS  │  │ App ECS  │            │   │
│  │  │ (AZ-a)   │  │ (AZ-b)   │            │   │
│  │  └──────────┘  └──────────┘            │   │
│  └────────────────────────────────────────┘   │
│                      │                        │
│  ┌────────────────────────────────────────┐   │
│  │     Isolated Subnets (DB Tier)          │   │
│  │  ┌──────────┐  ┌──────────┐            │   │
│  │  │ RDS AZ-a │  │ RDS AZ-b │            │   │
│  │  │(Primary) │  │(Standby) │            │   │
│  │  └──────────┘  └──────────┘            │   │
│  └────────────────────────────────────────┘   │
└───────────────────────────────────────────────┘

Security Groups:
  sg-alb:      inbound 443 from 0.0.0.0/0
  sg-app:      inbound 8080 from sg-alb only
  sg-database: inbound 5432 from sg-app only
               inbound 5432 from sg-bastion only (admin)
```

### Azure — Network Topology Chuẩn

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────┐
│              Virtual Network (10.0.0.0/16)       │
│                                                  │
│  ┌─────────────────────────────────────────┐    │
│  │  public-subnet (10.0.1.0/24)            │    │
│  │  Application Gateway / Azure Firewall   │    │
│  └──────────────────┬──────────────────────┘    │
│                     │                           │
│  ┌──────────────────────────────────────────┐   │
│  │  app-subnet (10.0.2.0/24)                │   │
│  │  App Service / AKS / VMs                 │   │
│  │  Service Endpoint / Private Link         │   │
│  └──────────────────┬───────────────────────┘   │
│                     │                           │
│  ┌──────────────────────────────────────────┐   │
│  │  db-subnet (10.0.3.0/24)                 │   │
│  │  Azure DB for PostgreSQL (Private Access)│   │
│  │  Private DNS Zone                        │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘

NSG Rules cho db-subnet:
  Inbound:  Allow TCP 5432 from app-subnet (10.0.2.0/24)
  Inbound:  Deny All (default)
  Outbound: Allow All (Azure DB cần outbound cho health checks)
```

---

## 2. Encryption

### Encryption at Rest

```
AWS RDS:
  - Bật khi tạo instance (KHÔNG bật được sau khi tạo)
  - Dùng AWS KMS key (Customer Managed Key — CMK cho compliance)
  - Encryption tự động apply cho: storage, snapshots, read replicas, backups
  
  ⚠️ Nếu quên bật: phải snapshot → restore sang encrypted instance
  
  Customer Managed Key (CMK) advantages:
    - Audit key usage qua CloudTrail
    - Rotate key hàng năm tự động
    - Revoke access nếu cần (terminate data access)
    - Cross-account sharing

Azure Database for PostgreSQL:
  - Encryption at rest mặc định (Azure-managed keys)
  - Customer-managed key via Azure Key Vault (cho compliance)
  - "Double encryption" option (software + hardware layer)

DynamoDB:
  - Encrypted by default với AWS-owned key
  - Upgrade lên AWS Managed Key hoặc CMK nếu cần audit
  - Không tốn thêm phí với AWS-owned key

Cosmos DB:
  - Encrypted by default (Microsoft-managed keys)
  - Customer-managed keys qua Azure Key Vault
  - Bật "double encryption" cho regulated industries
```

### Encryption in Transit

```
AWS RDS:
  Bắt buộc TLS bằng cách set parameter:
    rds.force_ssl = 1 (PostgreSQL)
    require_secure_transport = ON (MySQL)
  
  Download RDS CA certificate bundle:
    https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
  
  Connection string:
    sslmode=verify-full
    sslrootcert=/path/to/global-bundle.pem

Azure Database for PostgreSQL:
  ssl_enforcement_enabled = Enabled (default)
  
  Certificate:
    DigiCert Global Root CA (download từ DigiCert)
  
  Connection string:
    sslmode=verify-full
    sslrootcert=/path/to/DigiCertGlobalRootCA.crt.pem

ElastiCache Redis / Azure Cache for Redis:
  - Bật TLS khi tạo (không bật sau được trên một số tiers)
  - Port TLS: 6380 (thay vì 6379)
  - Client: ssl=True trong connection string
```

---

## 3. IAM & Authentication

### AWS — IAM Database Authentication

```
Thay vì dùng password, dùng IAM token (15 phút expiry):

# Tạo IAM Policy cho EC2/Lambda:
{
  "Effect": "Allow",
  "Action": "rds-db:connect",
  "Resource": "arn:aws:rds-db:us-east-1:123456:dbuser:db-XXXXX/app_user"
}

# Tạo DB user với rds_iam:
CREATE USER app_user WITH LOGIN;
GRANT rds_iam TO app_user;

# Application lấy auth token:
import boto3
client = boto3.client('rds')
token = client.generate_db_auth_token(
    DBHostname='mydb.xxx.us-east-1.rds.amazonaws.com',
    Port=5432,
    DBUsername='app_user'
)
# Dùng token như password (hết hạn sau 15 phút)

Ưu điểm IAM Auth:
  ✓ Không lưu password trong code
  ✓ Token ngắn hạn (15 phút)
  ✓ Audit qua CloudTrail
  ✗ Overhead generate token mỗi connection
  ✗ Không hỗ trợ connection pooling tốt (vì token expire)
```

### Azure — Managed Identity Authentication

```
# Enable Managed Identity cho App Service / AKS
# Assign role "Reader" hoặc custom role trên Azure DB resource

# Tạo Azure AD user trong PostgreSQL:
CREATE ROLE "my-app-managed-identity" WITH LOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO "my-app-managed-identity";

# Application code:
from azure.identity import DefaultAzureCredential
import psycopg2

credential = DefaultAzureCredential()
token = credential.get_token("https://ossrdbms-aad.database.windows.net/.default")

conn = psycopg2.connect(
    host="server.postgres.database.azure.com",
    database="mydb",
    user="my-app-managed-identity",
    password=token.token,  # Azure AD token
    sslmode="require"
)
```

---

## 4. Audit & Compliance

### Bật Database Audit Log

```sql
-- AWS RDS PostgreSQL (qua Parameter Group):
log_connections = on
log_disconnections = on  
log_duration = on
log_statement = 'ddl'     -- log tất cả DDL (CREATE, ALTER, DROP)
log_min_duration_statement = 0  -- log mọi query (chỉ audit DB, không production)

-- Xuất log:
CloudWatch Logs → tạo Log Group "rds/postgresql"
  Bật "Log exports" trong RDS console: postgresql, upgrade

-- Azure PostgreSQL (Server Parameters):
pgaudit.log = 'ddl,role,connection'
pgaudit.log_catalog = on
pgaudit.log_relation = on
-- Log xuất sang Azure Monitor Log Analytics
```

### AWS CloudTrail cho Database API Calls

```
Tự động audit mọi AWS API call liên quan đến RDS:
  - Ai tạo/xóa instance
  - Ai thay đổi Security Group
  - Ai tạo/restore snapshot
  - Ai thay đổi Parameter Group

Cấu hình:
  CloudTrail → Create Trail → S3 bucket + CloudWatch Logs
  Bật Management Events + Data Events (tốn phí thêm)
```

---

## 5. Secrets Rotation

### AWS Secrets Manager Auto-Rotation

```
Setup rotation cho RDS password:
1. Tạo Secret: Secrets Manager → Store a new secret → RDS credentials
2. Enable rotation: 
   - Rotation schedule: 30 ngày (hoặc tùy policy)
   - Lambda rotation function: AWS cung cấp sẵn cho RDS
3. Application dùng secret ARN, không dùng hardcoded password

Code không cần restart khi rotate:
  # SDK tự động lấy secret mới khi gọi get_secret_value
  # Nếu dùng connection pool: thêm retry logic khi authentication fail
  
  def get_db_connection():
      secret = get_secret("prod/myapp/rds")  # luôn fetch mới
      return create_connection(secret)
```

### Azure Key Vault + Secret Rotation

```
1. Lưu DB password trong Key Vault
2. App dùng Managed Identity lấy secret
3. Tạo Event Grid trigger khi secret sắp hết hạn
4. Azure Function tự động rotate password

Cấu hình Azure App Service:
  App Settings → Key Vault Reference:
  @Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/db-password/)
  → App tự động lấy giá trị mới khi secret được update
```

---

## 6. Checklist Bảo Mật Trước Khi Go-Live

```
Network:
  ☐ Database không có public IP / Publicly Accessible = No
  ☐ Chỉ mở port DB từ app tier Security Group / NSG
  ☐ VPC/VNet có Flow Logs bật
  ☐ Bastion Host hoặc SSM Session Manager cho admin access

Encryption:
  ☐ Encryption at rest bật (với CMK cho PCI/HIPAA)
  ☐ SSL/TLS enforced (force_ssl = 1)
  ☐ Certificate validation ở client (verify-full)

Authentication:
  ☐ Không dùng master/admin user cho application
  ☐ Mỗi service có dedicated DB user với quyền tối thiểu
  ☐ Password lưu trong Secrets Manager / Key Vault
  ☐ Auto-rotation bật (30-90 ngày)

Audit:
  ☐ Audit logs bật và xuất ra CloudWatch / Azure Monitor
  ☐ CloudTrail / Azure Activity Log cho API calls
  ☐ Alerts cho: failed logins, unusual access patterns

Compliance:
  ☐ Backup encryption bật
  ☐ PITR bật
  ☐ Backup retention đủ theo quy định (vd: 7 năm cho tài chính)
  ☐ Data residency: database region đúng với yêu cầu GDPR
```
