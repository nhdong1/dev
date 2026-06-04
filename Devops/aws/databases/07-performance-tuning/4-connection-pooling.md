# Connection Pooling — Gộp Kết Nối Database

> Connection pooling (gộp kết nối) là kỹ thuật tái sử dụng kết nối database thay vì tạo mới mỗi lần, giúp giảm overhead, tăng throughput và bảo vệ database khỏi quá nhiều kết nối đồng thời. RDS Proxy (Proxy RDS) là giải pháp fully-managed của AWS.

## 📚 Mục Lục

1. [Vấn Đề Kết Nối Database](#vấn-đề-kết-nối-database)
2. [Connection Pooling Là Gì?](#connection-pooling-là-gì)
3. [RDS Proxy — Proxy RDS](#rds-proxy--proxy-rds)
4. [Cấu Hình RDS Proxy](#cấu-hình-rds-proxy)
5. [pgBouncer — Giải Pháp Self-managed](#pgbouncer--giải-pháp-self-managed)
6. [Giới Hạn Kết Nối Theo Instance Class](#giới-hạn-kết-nối-theo-instance-class)
7. [Monitoring Connections](#monitoring-connections)
8. [Chuẩn Bị Phỏng Vấn](#chuẩn-bị-phỏng-vấn)

---

## 🔥 Vấn Đề Kết Nối Database

### Connection Overhead

Mỗi kết nối tới database đều có chi phí:

```
┌─────────────────────────────────────────────────────────────────┐
│  Chi Phí Mỗi Connection RDS                                      │
│                                                                   │
│  Memory:    ~10MB / connection (MySQL)                           │
│             ~5-10MB / connection (PostgreSQL)                     │
│                                                                   │
│  Ví dụ: db.r5.large (16 GB RAM)                                 │
│  - max_connections MySQL ≈ 1,341                                 │
│  - Nếu tất cả active: 1,341 × 10MB ≈ 13 GB chỉ cho connections │
│  - Còn lại cho buffer pool: ~3 GB → Cache miss nhiều            │
└─────────────────────────────────────────────────────────────────┘
```

### Vấn Đề Với Lambda + RDS

```
Kịch bản: E-commerce sale event
                                   RDS (max 1000 connections)
Lambda invocations  ─────────────► ┌──────────────────────────┐
     1000 concurrent  (mỗi Lambda  │  Thực tế cần:            │
                       tạo 1 conn)  │  - 1000 connections đồng │
                                    │    thời                   │
                                    │  - Vượt max_connections   │
                                    │  ❌ Connections bị từ chối│
                                    └──────────────────────────┘

Giải pháp: RDS Proxy
Lambda invocations  ──► RDS Proxy  ──► RDS
     1000 concurrent     (pool 100      (100 connections
                          connections)   thực tế)
```

### Khi Nào Connection Pooling Cần Thiết?

| Tình Huống | Vấn Đề | Giải Pháp |
|-----------|--------|---------|
| Serverless (Lambda, ECS) | Mỗi function/task tạo connection mới | RDS Proxy |
| Microservices nhiều service | Nhiều service × nhiều instances × nhiều threads | RDS Proxy hoặc pgBouncer |
| Spike traffic (đột biến lưu lượng) | Connections tăng đột biến | Connection pooling |
| max_connections bị đạt | Error: too many connections | Pooling hoặc scale instance |
| Connection setup latency cao | TLS handshake tốn thời gian | Persistent connection pool |

---

## 🔄 Connection Pooling Là Gì?

### Kiến Trúc Cơ Bản

```
BẮT ĐẦU — Không có pooling:
App Instance A ──[connect]──► RDS (10ms setup)
App Instance B ──[connect]──► RDS (10ms setup)
App Instance C ──[connect]──► RDS (10ms setup)
... N instances ──► N connections (mỗi query = 10ms overhead)

VỚI Connection Pool:
App Instance A ──► Pool ─[reuse conn]──► RDS
App Instance B ──► Pool ─[reuse conn]──► RDS
App Instance C ──► Pool ─[reuse conn]──► RDS
... N instances ──► M connections (M << N, không overhead)
```

### Connection Pool Hoạt Động Thế Nào?

```
Pool với 10 connections:

Initial: [conn1] [conn2] [conn3] ... [conn10]  ← Tất cả sẵn sàng

Request 1 đến: [BUSY:1] [conn2] [conn3] ... [conn10]
Request 2 đến: [BUSY:1] [BUSY:2] [conn3] ... [conn10]
...
Request 10 đến: [BUSY:1] [BUSY:2] [BUSY:3] ... [BUSY:10]

Request 11 đến: Chờ (wait_timeout) cho connection rảnh
→ Nếu timeout: error "Could not get connection from pool"

Request 1 xong: [conn1 RETURNED] [BUSY:2] ... [BUSY:10]
Request 11 nhận: [BUSY:11] [BUSY:2] ... [BUSY:10]
```

### Các Loại Pool Modes

| Mode | Hành Vi | Phù Hợp |
|------|--------|---------|
| **Session pooling** | 1 connection/client session | Nhiều kết nối idle |
| **Transaction pooling** | Connection được trả về pool sau mỗi transaction | OLTP, Lambda |
| **Statement pooling** | Connection được trả về sau mỗi statement | Không hỗ trợ multi-statement transactions |

---

## 🛡️ RDS Proxy — Proxy RDS

### RDS Proxy Là Gì?

RDS Proxy là fully-managed connection pooler (bộ gộp kết nối được quản lý hoàn toàn) chạy trong VPC (Virtual Private Cloud — Đám Mây Riêng Tư Ảo), nằm giữa ứng dụng và RDS/Aurora.

```
┌─────────────────────────────────────────────────────────────────┐
│  VPC                                                             │
│                                                                   │
│  ┌──────────┐          ┌─────────────────┐    ┌──────────────┐ │
│  │ Lambda   │──────────►                 │    │              │ │
│  │ Functions│          │   RDS Proxy     │────►  RDS/Aurora  │ │
│  │ (1000)  │          │                 │    │  (100 conns) │ │
│  └──────────┘          │ - Connection    │    └──────────────┘ │
│                        │   pooling       │                       │
│  ┌──────────┐          │ - IAM auth      │                       │
│  │ ECS      │──────────► - Secrets Mgr   │                       │
│  │ Services │          │ - Multi-AZ HA   │                       │
│  └──────────┘          │ - Failover fast │                       │
│                        └─────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

### Tính Năng Chính RDS Proxy

| Tính Năng | Mô Tả | Lợi Ích |
|----------|-------|--------|
| **Connection Pooling** | Duy trì pool connections tới DB | Giảm max connections lên 99% |
| **IAM Authentication** | Xác thực qua IAM thay vì username/password | Bảo mật tốt hơn, không lưu credentials trong code |
| **Secrets Manager Integration** | Tự động lấy credentials từ Secrets Manager | Rotation không ảnh hưởng ứng dụng |
| **Multi-AZ** | Proxy chạy ở nhiều AZ | HA cho proxy layer |
| **Faster Failover** | Phát hiện failover < 30 giây | RDS Multi-AZ failover nhanh hơn |
| **Read/Write Splitting** | Tự động route read/write đến đúng endpoint | Đơn giản hóa code ứng dụng |
| **TLS Enforcement** | Bắt buộc TLS cho tất cả connections | Encryption in transit |

### Hỗ Trợ Engine

| Engine | Hỗ Trợ |
|--------|--------|
| RDS MySQL 5.6, 5.7, 8.0 | ✅ |
| RDS PostgreSQL 10, 11, 12, 13, 14, 15 | ✅ |
| Aurora MySQL | ✅ |
| Aurora PostgreSQL | ✅ |
| RDS MariaDB | ✅ |
| Oracle / SQL Server | ❌ Chưa hỗ trợ |

### Giới Hạn & Lưu Ý

| Điểm | Chi Tiết |
|------|---------|
| **Không hỗ trợ** | `SET` statements, temporary tables (transaction pooling) |
| **Pinning** | Một số features gây "pinning" — connection bị chiếm cho suốt session |
| **Chi phí** | ~$0.015/giờ/vCPU của DB instance |
| **Latency** | Thêm ~1ms latency so với kết nối trực tiếp |
| **VPC only** | Không thể kết nối từ outside VPC mà không có VPC endpoint |

### Causes of Pinning — Nguyên Nhân Gây Pinning

Pinning xảy ra khi RDS Proxy buộc phải giữ nguyên connection cho cả session, vô hiệu hóa connection pooling:

```
MySQL — Pinning triggers:
- SET statements (SET @variable = value)
- Temporary tables (CREATE TEMPORARY TABLE)
- Table locks (LOCK TABLES)
- Prepared statements nhiều lần
- TEXT/BLOB parameters

PostgreSQL — Pinning triggers:
- SET statements
- Advisory locks
- Notify/Listen
- Extended query protocol với certain features
```

---

## ⚙️ Cấu Hình RDS Proxy

### Tạo RDS Proxy Qua Console

```
RDS Console → Proxies → Create Proxy

Cấu hình:
- Proxy identifier: myapp-proxy
- Engine: MySQL 8.0
- Require TLS: Yes
- Idle client connection timeout: 1800 giây
- Target group:
  - RDS instance: mydb
  - Connection pool max connections: 100%
  - Connection borrow timeout: 120 giây
  - Session pinning filters: EXCLUDE_VARIABLE_SETS
- Authentication:
  - IAM authentication: required
  - Secrets: myapp/db-credentials
- VPC & Security groups
```

### Tạo Qua Terraform

```hcl
resource "aws_db_proxy" "main" {
  name                   = "myapp-proxy"
  debug_logging          = false
  engine_family          = "MYSQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  vpc_subnet_ids         = aws_subnet.private[*].id

  auth {
    auth_scheme = "SECRETS"
    description = "DB credentials"
    iam_auth    = "REQUIRED"
    secret_arn  = aws_secretsmanager_secret.db_credentials.arn
  }
}

resource "aws_db_proxy_default_target_group" "main" {
  db_proxy_name = aws_db_proxy.main.name

  connection_pool_config {
    connection_borrow_timeout    = 120
    max_connections_percent      = 100
    max_idle_connections_percent = 50
    session_pinning_filters      = ["EXCLUDE_VARIABLE_SETS"]
  }
}

resource "aws_db_proxy_target" "main" {
  db_instance_identifier = aws_db_instance.main.id
  db_proxy_name          = aws_db_proxy.main.name
  target_group_name      = aws_db_proxy_default_target_group.main.name
}
```

### Kết Nối Qua RDS Proxy

```python
# Python — Kết nối qua RDS Proxy với IAM Auth
import boto3
import pymysql
import ssl

def get_auth_token():
    client = boto3.client('rds', region_name='ap-southeast-1')
    return client.generate_db_auth_token(
        DBHostname='myapp-proxy.proxy-xxxx.ap-southeast-1.rds.amazonaws.com',
        Port=3306,
        DBUsername='appuser',
        Region='ap-southeast-1'
    )

def connect_via_proxy():
    token = get_auth_token()
    ssl_context = ssl.create_default_context()
    
    conn = pymysql.connect(
        host='myapp-proxy.proxy-xxxx.ap-southeast-1.rds.amazonaws.com',
        user='appuser',
        password=token,        # IAM token thay cho password
        database='myapp',
        port=3306,
        ssl=ssl_context,
        connect_timeout=10
    )
    return conn
```

```javascript
// Node.js — Lambda với RDS Proxy
const mysql = require('mysql2/promise');
const AWS = require('aws-sdk');

const rds = new AWS.RDS.Signer();

// Tạo pool 1 lần ngoài handler (persist qua Lambda invocations)
let pool;

async function getPool() {
  if (!pool) {
    const token = await rds.getAuthToken({
      hostname: process.env.DB_PROXY_ENDPOINT,
      port: 3306,
      username: process.env.DB_USER
    });
    
    pool = mysql.createPool({
      host: process.env.DB_PROXY_ENDPOINT,
      user: process.env.DB_USER,
      password: token,
      database: process.env.DB_NAME,
      ssl: { rejectUnauthorized: true },
      connectionLimit: 10  // Lambda: giữ nhỏ để không overwhelm proxy
    });
  }
  return pool;
}

exports.handler = async (event) => {
  const pool = await getPool();
  const [rows] = await pool.execute('SELECT * FROM users WHERE id = ?', [event.userId]);
  return rows;
};
```

### Tham Số Connection Pool Configuration

| Tham Số | Mô Tả | Giá Trị |
|---------|-------|--------|
| `MaxConnectionsPercent` | % max_connections của DB instance mà Proxy có thể dùng | 10-100%, default 100% |
| `MaxIdleConnectionsPercent` | % connections idle được giữ trong pool | 0-100%, default 50% |
| `ConnectionBorrowTimeout` | Thời gian chờ lấy connection từ pool (giây) | 1-3600, default 120 |
| `SessionPinningFilters` | Filters để giảm pinning | `EXCLUDE_VARIABLE_SETS` |

---

## 🔧 pgBouncer — Giải Pháp Self-managed

### pgBouncer Là Gì?

pgBouncer là open-source connection pooler cho PostgreSQL, thường được deploy trên EC2 hoặc ECS trong VPC.

```
Application ──► pgBouncer (EC2/ECS) ──► RDS PostgreSQL
                (Pool 100 connections)    (Actual connections)
```

### Cấu Hình pgBouncer

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
myapp = host=mydb.cluster-xxxx.rds.amazonaws.com port=5432 dbname=myapp

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 5432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool mode
pool_mode = transaction          # transaction pooling (tốt nhất cho OLTP)

# Connection limits
max_client_conn = 1000           # Tổng client connections vào pgBouncer
default_pool_size = 20           # Connections tới DB mỗi database/user pair
min_pool_size = 5                # Connections tối thiểu luôn giữ
reserve_pool_size = 5            # Extra connections cho high load

# Timeouts
server_idle_timeout = 600        # Đóng idle server connection sau 10 phút
client_idle_timeout = 0          # Không timeout client (ứng dụng tự quản lý)
server_connect_timeout = 15      # Timeout khi connect tới DB

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1

# Stats
stats_period = 60
```

### pgBouncer vs RDS Proxy

| Tiêu Chí | pgBouncer | RDS Proxy |
|---------|----------|----------|
| Quản lý | Self-managed | Fully-managed AWS |
| Chi phí | EC2/ECS cost | ~$0.015/vCPU/giờ |
| Engine | PostgreSQL only | MySQL, PostgreSQL, MariaDB |
| Pool modes | Session, Transaction, Statement | Transaction pooling |
| HA | Cần tự setup | Built-in Multi-AZ |
| IAM Auth | Không có | Có |
| Secrets Manager | Không có | Có |
| Failover detection | Chậm hơn | < 30 giây |
| Phù hợp | Cần kiểm soát hoàn toàn, tiết kiệm chi phí | Serverless, cần managed service |

---

## 📏 Giới Hạn Kết Nối Theo Instance Class

### MySQL max_connections

```sql
-- Công thức:
-- max_connections = DBInstanceClassMemory / 12,582,880

-- Bảng tham khảo:
-- db.t3.micro    (1 GB)   ≈ 83 connections
-- db.t3.small    (2 GB)   ≈ 167 connections
-- db.t3.medium   (4 GB)   ≈ 334 connections
-- db.t3.large    (8 GB)   ≈ 668 connections
-- db.r5.large    (16 GB)  ≈ 1,341 connections
-- db.r5.xlarge   (32 GB)  ≈ 2,682 connections
-- db.r5.2xlarge  (64 GB)  ≈ 5,364 connections
-- db.r5.4xlarge  (128 GB) ≈ 10,728 connections

-- Kiểm tra giá trị hiện tại
SHOW VARIABLES LIKE 'max_connections';

-- Xem connections đang dùng
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
```

### PostgreSQL max_connections

```sql
-- Công thức:
-- max_connections ≈ DBInstanceClassMemory / 9,531,392

-- Bảng tham khảo:
-- db.t3.micro    (1 GB)   ≈ 112 connections
-- db.t3.medium   (4 GB)   ≈ 446 connections
-- db.r5.large    (16 GB)  ≈ 1,784 connections
-- db.r5.xlarge   (32 GB)  ≈ 3,568 connections

-- Lưu ý: PostgreSQL reserved connections
-- superuser_reserved_connections = 3 (mặc định)
-- Thực tế available = max_connections - 3

-- Kiểm tra
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
```

### Thực Hành Tốt Nhất — Connection Management

```
Nguyên tắc sizing:
1. Tổng connections = (instances × threads_per_instance) + pooler_overhead
2. Giữ connections thực tới DB ≤ 2-3× số vCPU
   Ví dụ: db.r5.xlarge (4 vCPU) → target 8-12 connections active

Với RDS Proxy:
- MaxConnectionsPercent = 100% (Proxy quản lý)
- Lambda pool size = 1-5 connections per function
- Tổng Lambda concurrency × pool size << DB max_connections

Với pgBouncer:
- default_pool_size = 2-5× vCPU của DB
- max_client_conn = 10× default_pool_size
```

---

## 📊 Monitoring Connections

### CloudWatch Metrics

```bash
# Theo dõi connections
aws cloudwatch get-metric-statistics \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=mydb \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 60 \
  --statistics Maximum

# RDS Proxy metrics
aws cloudwatch list-metrics \
  --namespace AWS/RDS \
  --dimensions Name=ProxyName,Value=myapp-proxy
```

### RDS Proxy Metrics Quan Trọng

| Metric | Ý Nghĩa | Ngưỡng Cảnh Báo |
|--------|--------|----------------|
| `ClientConnections` | Connections từ app tới Proxy | Monitor trend |
| `DBConnections` | Connections từ Proxy tới DB | Monitor trend |
| `DatabaseConnectionsSetupFailed` | Lỗi khi setup connection | > 0 |
| `ClientConnectionsClosed` | Connections bị đóng | Tăng đột biến |
| `QueryRequests` | Tổng queries qua Proxy | Monitor trend |
| `Pinned` | % connections bị pinned | > 50% cần tối ưu |

### SQL Monitoring (MySQL)

```sql
-- Xem connections theo user
SELECT user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;

-- Xem connection count theo state
SELECT command, count(*) as count
FROM information_schema.processlist
GROUP BY command;

-- Xem sleeping connections lâu
SELECT id, user, host, db, time, state
FROM information_schema.processlist
WHERE command = 'Sleep' AND time > 300
ORDER BY time DESC;

-- Kill sleeping connections quá lâu (> 1 giờ)
SELECT CONCAT('KILL ', id, ';')
FROM information_schema.processlist
WHERE command = 'Sleep' AND time > 3600;
```

### SQL Monitoring (PostgreSQL)

```sql
-- Connections theo state
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;

-- Long-running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
  AND state != 'idle'
ORDER BY duration DESC;

-- Blocked queries
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_statement,
       blocking_activity.query AS current_statement_in_blocking_process
FROM pg_catalog.pg_locks AS blocked_locks
JOIN pg_catalog.pg_stat_activity AS blocked_activity
  ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks AS blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
  AND blocking_locks.granted = true
  AND blocked_locks.granted = false
JOIN pg_catalog.pg_stat_activity AS blocking_activity
  ON blocking_activity.pid = blocking_locks.pid;
```

---

## 🎯 Chuẩn Bị Phỏng Vấn

### Q: Tại sao Lambda + RDS cần RDS Proxy?

```
Vấn đề không có Proxy:
1. Lambda có thể scale tới hàng nghìn concurrent executions
2. Mỗi Lambda execution cần 1 connection tới RDS
3. RDS có max_connections giới hạn (ví dụ: 1000 với db.r5.large)
4. Khi 1001 Lambda cùng chạy: "Too many connections" error

Chi phí mỗi connection:
- Setup time: 5-50ms (TCP + TLS + DB auth)
- Memory: ~10MB/connection
- 1000 connections × 10MB = 10GB chỉ cho connections

RDS Proxy giải quyết:
- Proxy duy trì pool 100 connections tới RDS
- 1000 Lambda share pool đó
- Lambda connect tới Proxy trong <1ms (persistent pool)
- RDS chỉ thấy 100 connections
- Failover: Proxy che giấu failover event, ứng dụng không cần handle
```

### Q: Connection pooling ảnh hưởng đến transactions như thế nào?

```
Session pooling (pgBouncer default):
- 1 connection dành riêng cho 1 client session
- Transaction hoạt động bình thường
- Ít hiệu quả hơn vì connection bị giữ kể cả khi idle

Transaction pooling (recommended):
- Connection được trả về pool sau mỗi COMMIT/ROLLBACK
- Multi-statement transaction: hoạt động bình thường trong 1 transaction
- Giữa transactions: connection có thể được dùng bởi client khác

Lưu ý quan trọng với transaction pooling:
- SET session variables bị mất khi connection được return về pool
- Solution: Chỉ SET trong transaction, hoặc dùng SET LOCAL
- Prepared statements cần được tạo lại mỗi session (với Proxy)
- pgBouncer với transaction pooling: không hỗ trợ LISTEN/NOTIFY

RDS Proxy luôn dùng transaction pooling.
Cần test kỹ ORM behavior trước khi enable.
```

### Q: Pinning trong RDS Proxy là gì và làm sao tránh?

```
Pinning = connection bị "ghim" vào 1 client session
→ Mất đi lợi ích của connection pooling cho session đó

Nguyên nhân phổ biến:
MySQL:
- SET @user_variable = value → pin cả session
- LOCK TABLES → pin đến khi UNLOCK
- CREATE TEMPORARY TABLE → pin đến khi drop

PostgreSQL:
- SET → pin cả session
- Advisory locks → pin đến khi release

Giải pháp:
1. SessionPinningFilters = EXCLUDE_VARIABLE_SETS
   → Proxy không pin khi gặp SET statements
   → Nhưng cần đảm bảo ứng dụng không phụ thuộc vào session variables

2. Tránh LOCK TABLES → dùng SELECT ... FOR UPDATE
3. Tránh SET @variables → pass values trực tiếp trong query
4. Monitor metric "Pinned" trong CloudWatch

Kiểm tra pinning:
aws cloudwatch get-metric-statistics \
  --metric-name Pinned \
  --namespace AWS/RDS \
  --dimensions Name=ProxyName,Value=myapp-proxy
```

---

**Tiếp Theo:** [5-dynamodb-performance.md](5-dynamodb-performance.md) — Hot Partitions, Throttling & DAX

**Cập Nhật Lần Cuối:** 2026-05-15
