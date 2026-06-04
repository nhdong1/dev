# Setup SQL Database trên Cloud — Chi Tiết & Lưu Ý

## 1. AWS RDS / Aurora — Checklist Setup

### Bước 1: Chọn Instance Type & Storage

```
Production workload:
  db.r7g.large   (8GB RAM)  → ứng dụng vừa
  db.r7g.xlarge  (16GB RAM) → tải trung bình
  db.r7g.2xlarge (32GB RAM) → tải nặng

Storage:
  gp3 (General Purpose SSD) → default, 3000 IOPS miễn phí
  io1/io2                    → IOPS cao, cần >12000 IOPS
  
  ⚠️ LƯU Ý: Bật "Storage Autoscaling" để tránh hết disk
             Nhưng đặt ngưỡng max (vd: 500GB) để không bị
             surprise billing
```

### Bước 2: Cấu hình Networking

```yaml
# Bắt buộc phải làm đúng:

VPC: Chọn VPC riêng, KHÔNG dùng Default VPC

Subnet Group:
  - Tạo DB Subnet Group với ít nhất 2 AZ
  - Chỉ dùng Private Subnet (không có Internet Gateway)

Security Group (SG):
  - Tạo SG riêng cho DB: "sg-database"
  - Inbound rule: chỉ cho phép từ SG của app server
  - KHÔNG mở port 5432/3306 ra Internet
  - KHÔNG dùng 0.0.0.0/0

Ví dụ Security Group Rule:
  Type: PostgreSQL (5432)
  Source: sg-app-server (SG của application tier)
  Description: Allow from app servers only
```

**Lưu ý QUAN TRỌNG — Publicly Accessible:**
```
"Publicly Accessible = No" là bắt buộc với production.
Nếu cần debug từ local → dùng SSH tunnel qua Bastion Host
hoặc AWS Systems Manager Session Manager, KHÔNG bật public access.
```

### Bước 3: Parameter Group

```sql
-- PostgreSQL quan trọng cần chỉnh:
shared_buffers = 25% RAM              -- vd: 2GB nếu instance 8GB RAM
effective_cache_size = 75% RAM        -- ước tính OS cache
work_mem = RAM / (max_connections * 4) -- RAM cho mỗi operation sort/hash
max_connections = 100-200             -- không nên quá cao, dùng RDS Proxy thay
wal_level = replica                   -- cần cho read replica
log_min_duration_statement = 1000    -- log query > 1000ms
log_connections = on                 -- log connection mới
log_disconnections = on
log_lock_waits = on                  -- log khi chờ lock > deadlock_timeout
```

```sql
-- MySQL/Aurora MySQL:
innodb_buffer_pool_size = 70-75% RAM
max_connections = 1000                -- Aurora quản lý tốt hơn RDS MySQL
slow_query_log = 1
long_query_time = 1
log_queries_not_using_indexes = 1
binlog_format = ROW                   -- cần cho replication
```

### Bước 4: Multi-AZ và Backup

```
Multi-AZ:
  ✓ Bật cho production (tăng ~$$/tháng nhưng SLA 99.95%)
  ✗ Có thể tắt cho dev/staging

Automated Backup:
  Retention: 7-35 ngày (khuyến nghị 14 ngày production)
  Backup Window: chọn giờ thấp điểm (vd: 02:00-03:00 UTC)
  
  ⚠️ LƯU Ý: Backup window và maintenance window KHÔNG được overlap

Maintenance Window:
  Chọn cuối tuần, giờ thấp điểm
  Tắt "Auto Minor Version Upgrade" nếu muốn kiểm soát timing
  
Point-in-Time Recovery (PITR):
  AWS tự động giữ transaction logs trong retention period
  Có thể restore đến bất kỳ giây nào trong window
```

### Bước 5: RDS Proxy (Khuyến nghị cho Production)

```
Tại sao cần RDS Proxy?
  - Lambda functions: mỗi invocation tạo connection mới → DB quá tải
  - Microservices: nhiều service, nhiều connection pool
  - Failover: Proxy handle reconnection, app không cần retry logic

Cách hoạt động:
  App → RDS Proxy (pool connections) → RDS Instance
  
  Proxy duy trì pool lên đến 100% max_connections
  App kết nối đến Proxy endpoint (không đổi sau failover)

Chi phí: ~$0.015/GB data processed + $0.015/hour per AZ
```

---

## 2. Azure Database for PostgreSQL Flexible Server — Setup

### Compute & Storage

```
Compute tier:
  Burstable (B1ms, B2s):  dev/test, tiết kiệm chi phí
  General Purpose (D4s):  production chuẩn
  Memory Optimized (E4s): workload cần nhiều RAM (analytics)

Storage:
  Premium SSD v2: linh hoạt IOPS (không cần provision IOPS tối đa)
  Premium SSD:    cố định IOPS theo storage size
  
  ⚠️ Storage chỉ TĂNG được, không giảm được
     Bật "Storage Auto-grow" để tự mở rộng khi đầy 95%
```

### High Availability

```
Zone Redundant HA:
  - Primary và Standby ở 2 Availability Zone khác nhau
  - Sync replication
  - Failover tự động ~60-120 giây
  - Cần tại các region hỗ trợ multiple AZ

Same Zone HA:
  - Primary và Standby cùng AZ
  - Rẻ hơn Zone Redundant ~30%
  - Không bảo vệ được khi AZ down

⚠️ HA chỉ có khi chọn từ đầu, không bật sau khi tạo
   (phải restore từ backup vào instance mới nếu muốn đổi)
```

### Networking — Private Access (Bắt buộc Production)

```
Private Access (VNet Integration):
  - Server nằm hoàn toàn trong VNet, không có public IP
  - Tạo Private DNS Zone: server.private.postgres.database.azure.com
  - Dùng Private Endpoint để kết nối từ VNet khác

Public Access:
  - Chỉ cho dev/test với IP Firewall Rule
  - Tuyệt đối không dùng 0.0.0.0 - 255.255.255.255 cho production

Kết nối từ Azure Services (App Service, AKS):
  - Dùng VNet Integration của App Service
  - AKS: tạo Pod Subnet và enable VNet injection
```

### Server Parameters Quan Trọng

```bash
# Bật qua Azure Portal > Server Parameters:
shared_buffers            = 25% RAM (tự tính theo vCores)
pg_qs.query_capture_mode  = ALL     # Query Store - monitor queries
pgms_wait_sampling.query_capture_mode = ALL  # Wait statistics
log_min_duration_statement = 1000   # ms
log_connections            = on
log_checkpoints            = on
connection_throttling      = on     # bảo vệ khi quá tải
```

---

## 3. Lưu Ý Quan Trọng Khi Setup SQL Cloud

### Connection String & SSL

```python
# AWS RDS — PHẢI dùng SSL production
import psycopg2
conn = psycopg2.connect(
    host="mydb.cluster-xxxxx.us-east-1.rds.amazonaws.com",
    database="mydb",
    user="app_user",
    password=get_secret("rds/password"),  # KHÔNG hardcode
    sslmode="verify-full",                # verify certificate
    sslrootcert="/etc/ssl/certs/aws-rds-ca.pem"
)

# Azure PostgreSQL — SSL enforced by default
conn_str = (
    f"host={server}.postgres.database.azure.com "
    f"dbname={database} "
    f"user={username} "
    f"password={get_secret('azure/db')} "
    f"sslmode=require"
)
```

### Quản lý Users & Permissions

```sql
-- Nguyên tắc Least Privilege
-- Tạo user riêng cho từng application, KHÔNG dùng admin account

-- AWS RDS PostgreSQL
CREATE USER app_readonly WITH PASSWORD 'xxx';
GRANT CONNECT ON DATABASE mydb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
  GRANT SELECT ON TABLES TO app_readonly;

CREATE USER app_readwrite WITH PASSWORD 'xxx';
GRANT CONNECT ON DATABASE mydb TO app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;

-- KHÔNG BAO GIỜ dùng master user (admin) cho application
-- master user chỉ dùng cho DDL operations (create table, migration)
```

### Secrets Management

```
AWS:
  ✓ AWS Secrets Manager → tự động rotate password
  ✓ RDS tích hợp Secrets Manager: enable "Manage master credentials"
  ✓ App dùng IAM Role để get secret (không cần AWS keys trong code)
  
  # Ví dụ IAM Policy cho app:
  {
    "Effect": "Allow",
    "Action": "secretsmanager:GetSecretValue",
    "Resource": "arn:aws:secretsmanager:region:account:secret:rds/myapp-*"
  }

Azure:
  ✓ Azure Key Vault → lưu connection string / password
  ✓ Managed Identity → app lấy secret từ Key Vault không cần credential
  ✓ Azure App Configuration → centralize config
  
  # Dùng Managed Identity:
  from azure.keyvault.secrets import SecretClient
  from azure.identity import DefaultAzureCredential
  
  client = SecretClient(vault_url=vault_url, credential=DefaultAzureCredential())
  secret = client.get_secret("db-password")
```

### Sizing & Capacity Planning

```
Công thức tính RAM cần thiết:
  shared_buffers ≈ 25% working dataset
  Nếu working dataset = 10GB → cần instance có ≥ 40GB RAM

Công thức tính IOPS:
  IOPS cần = (TPS * avg_pages_per_tx * 2)  # đọc + ghi
  gp3 cho 3000 IOPS miễn phí, đủ cho phần lớn workload

Dấu hiệu cần scale UP:
  CPU sustained > 70%          → scale compute
  FreeStorageSpace < 20%       → scale storage (hoặc autoscale)
  ReadLatency > 20ms           → check index, query
  DatabaseConnections ≈ max    → thêm RDS Proxy hoặc connection pooler
  SwapUsage > 0                → RAM không đủ, scale lên
```

---

## 4. Các Lỗi Setup Thường Gặp

```
❌ Lỗi 1: Để "Publicly Accessible = Yes" trên production
   → Fix: Migrate sang private subnet, dùng VPN/Bastion

❌ Lỗi 2: Không bật Automated Backup
   → Fix: Bật ngay, retention ≥ 7 ngày

❌ Lỗi 3: Dùng default parameter group
   → Fix: Tạo custom parameter group, chỉnh shared_buffers, work_mem

❌ Lỗi 4: max_connections quá cao (vd: 5000)
   → Fix: Giảm xuống 200-500, dùng PgBouncer hoặc RDS Proxy

❌ Lỗi 5: Không test failover
   → Fix: Reboot with failover hàng quý, đo thời gian recovery

❌ Lỗi 6: Storage không bật autoscaling → DB đầy, crash
   → Fix: Bật autoscaling với threshold 90%, set max limit

❌ Lỗi 7: Dùng master user cho application
   → Fix: Tạo dedicated app user với quyền tối thiểu

❌ Lỗi 8: Không monitor slow queries
   → Fix: Bật Performance Insights (AWS) hoặc Query Store (Azure)
```
