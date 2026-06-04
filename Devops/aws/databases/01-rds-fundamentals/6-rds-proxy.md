# 6 — RDS Proxy — Proxy Cơ Sở Dữ Liệu

> RDS Proxy (Proxy RDS) là dịch vụ proxy fully managed (Được Quản Lý Hoàn Toàn) nằm giữa ứng dụng và RDS database. Nó giải quyết bài toán connection management (Quản Lý Kết Nối) — đặc biệt quan trọng với serverless architectures và microservices có nhiều kết nối ngắn hạn.

## 📚 Mục Lục

1. [Vấn Đề RDS Proxy Giải Quyết](#vấn-đề-rds-proxy-giải-quyết)
2. [Kiến Trúc RDS Proxy](#kiến-trúc-rds-proxy)
3. [Connection Pooling — Gộp Kết Nối](#connection-pooling--gộp-kết-nối)
4. [Tính Năng Nổi Bật](#tính-năng-nổi-bật)
5. [Cấu Hình RDS Proxy](#cấu-hình-rds-proxy)
6. [RDS Proxy với Lambda](#rds-proxy-với-lambda)
7. [Chi Phí và Trade-offs](#chi-phí-và-trade-offs)
8. [Khi Nào Dùng RDS Proxy](#khi-nào-dùng-rds-proxy)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Vấn Đề RDS Proxy Giải Quyết

### Vấn Đề 1: Connection Exhaustion (Cạn Kiệt Kết Nối)

```
Tình huống thực tế:
  - ECS tasks: 100 instances × 10 connections mỗi task = 1,000 connections
  - Lambda functions: 500 concurrent × 1 connection = 500 connections
  - Tổng: 1,500 connections đồng thời

Giới hạn RDS:
  - db.t3.medium: ~420 max connections
  - db.m7g.large: ~890 max connections
  - db.r7g.large: ~1,800 max connections

→ Vượt giới hạn → "Too many connections" error
→ Database bị quá tải dù không có query nào chạy
```

### Vấn Đề 2: Lambda Connection Churn (Xáo Trộn Kết Nối Lambda)

```
Mỗi Lambda invocation:
  1. Tạo kết nối mới đến database    (TCP handshake + auth: ~100ms)
  2. Chạy query                       (~10ms)
  3. Đóng kết nối                     (TCP teardown)

Với 1,000 invocations/giây:
  → 1,000 new connections/giây → database không thể handle
  → Overhead kết nối > thời gian query thực sự
```

### Vấn Đề 3: Failover Recovery (Phục Hồi Sau Chuyển Đổi Dự Phòng)

```
Khi failover xảy ra:
  - Tất cả connections bị ngắt đột ngột
  - Hàng trăm ứng dụng đồng thời reconnect
  - Database bị thundering herd (Bầy Đàn Sấm Sét) — quá tải ngay lúc vừa phục hồi
  - Thực tế failover recovery lâu hơn 120 giây do connection storm (Bão Kết Nối)
```

**RDS Proxy giải quyết cả 3 vấn đề này.**

---

## Kiến Trúc RDS Proxy

### Sơ Đồ Kiến Trúc

```
                    VPC (Mạng Ảo)
        ┌──────────────────────────────────────────────────┐
        │                                                   │
        │   Application Layer (Tầng Ứng Dụng)              │
        │   ┌──────────┐ ┌──────────┐ ┌──────────┐        │
        │   │Lambda #1 │ │Lambda #2 │ │ECS Task  │        │
        │   │1 conn    │ │1 conn    │ │5 conns   │        │
        │   └────┬─────┘ └────┬─────┘ └────┬─────┘        │
        │        │             │             │               │
        │        └─────────────┼─────────────┘               │
        │                      │ (1,000 short-lived connections) │
        │                      ▼                             │
        │   ┌──────────────────────────────────────────┐   │
        │   │              RDS Proxy                    │   │
        │   │  (Connection Pooling — Gộp Kết Nối)      │   │
        │   │  ● Multiplexing 1,000 → 50 connections   │   │
        │   │  ● Credentials via Secrets Manager        │   │
        │   │  ● Failover aware (Nhận Biết Failover)   │   │
        │   └──────────────────┬───────────────────────┘   │
        │                      │ (50 persistent connections) │
        │                      ▼                             │
        │   ┌──────────────────────────────────────────┐   │
        │   │         RDS / Aurora Instance             │   │
        │   │         (Primary — chính)                 │   │
        │   └──────────────────────────────────────────┘   │
        │                                                   │
        └──────────────────────────────────────────────────┘
```

### Điểm Mấu Chốt Kiến Trúc

1. **Multiplexing (Ghép Kênh):** Nhiều application connections dùng chung ít database connections
2. **Persistent connections (Kết Nối Bền Vững):** Proxy duy trì pool connection đến database — không tạo/xóa liên tục
3. **Seamless failover (Chuyển Đổi Dự Phòng Liền Mạch):** Proxy tự xử lý failover, ứng dụng chỉ thấy connection pause ngắn

---

## Connection Pooling — Gộp Kết Nối

### Cơ Chế Pooling

```
Không có RDS Proxy:
  App Connection 1  →  DB Connection 1
  App Connection 2  →  DB Connection 2
  App Connection N  →  DB Connection N
  → 1:1 mapping, N connections đến database

Với RDS Proxy:
  App Connection 1  ─┐
  App Connection 2  ─┤→  DB Connection 1 (reused)
  App Connection 3  ─┤
  App Connection 4  ─┘→  DB Connection 2 (reused)
  ...
  App Connection N  ─→  DB Connection M (M << N)
  → M:N multiplexing, M connections đến database
```

### Connection Borrow Flow (Luồng Mượn Kết Nối)

```
1. App gửi query đến RDS Proxy endpoint
2. Proxy kiểm tra pool — có connection rảnh không?
   a. Có → Mượn connection đó, gửi query
   b. Không → Chờ connection trong pool (theo max_connection_pool_size)
3. Proxy nhận kết quả từ database
4. Proxy trả connection về pool (không đóng)
5. App nhận kết quả
```

### Tham Số Quan Trọng Của Pool

| Tham Số                           | Giá Trị Mặc Định | Mô Tả                                    |
| ---------------------------------- | ---------------- | ---------------------------------------- |
| `MaxConnectionsPercent`            | 100%             | % max_connections của DB dành cho proxy  |
| `MaxIdleConnectionsPercent`        | 50%              | % connections giữ idle trong pool        |
| `ConnectionBorrowTimeout`          | 120 giây         | Thời gian chờ mượn connection trước khi timeout |
| `SessionPinningFilters`            | None             | Transactions phải pin (Ghim) vào 1 connection |

---

## Tính Năng Nổi Bật

### 1. Faster Failover (Chuyển Đổi Dự Phòng Nhanh Hơn)

```
Không có RDS Proxy:
  Failover  → Tất cả connections mất → App reconnect đồng thời
  Recovery: 120+ giây (do thundering herd)

Với RDS Proxy:
  Failover  → Proxy detect failover → Proxy reconnect đến new primary
  App connections chỉ thấy pause ngắn (~30s thay vì 120s)
  Recovery: 30-60 giây (proxy xử lý reconnection thay app)
```

### 2. IAM Authentication (Xác Thực IAM)

RDS Proxy bắt buộc ứng dụng xác thực qua IAM token, không cần hardcode password:

```python
import boto3
import pymysql

def get_rds_token(hostname, port, username, region):
    client = boto3.client('rds', region_name=region)
    token = client.generate_db_auth_token(
        DBHostname=hostname,
        Port=port,
        DBUsername=username,
        Region=region
    )
    return token  # Token hết hạn sau 15 phút — tự động renew

# Kết nối qua IAM
token = get_rds_token(PROXY_ENDPOINT, 3306, 'lambda_user', 'us-east-1')
conn = pymysql.connect(
    host=PROXY_ENDPOINT,
    user='lambda_user',
    password=token,
    ssl_ca='/path/to/rds-ca-bundle.pem',
    ssl=True
)
```

### 3. Secrets Manager Integration (Tích Hợp Quản Lý Bí Mật)

RDS Proxy tự động lấy credentials từ AWS Secrets Manager — bạn không cần manage rotation:

```
Secrets Manager → Rotate password tự động mỗi 30 ngày
RDS Proxy       → Tự động cập nhật password từ Secrets Manager
Application     → Không cần biết gì về password rotation
```

### 4. Session Pinning (Ghim Phiên)

Một số SQL statements yêu cầu tất cả queries trong session phải dùng cùng connection:

```sql
-- Các operations cần session pinning:
SET @variable = value;       -- Session variables
LOCK TABLES ...;             -- Table locks
START TRANSACTION;           -- Transactions
PREPARE statement;           -- Prepared statements
```

Proxy tự động detect và pin session khi cần — đảm bảo correctness.

---

## Cấu Hình RDS Proxy

### Tạo RDS Proxy

```bash
# Tạo RDS Proxy
aws rds create-db-proxy \
    --db-proxy-name myapp-proxy \
    --engine-family MYSQL \
    --auth '[{
        "AuthScheme": "SECRETS",
        "SecretArn": "arn:aws:secretsmanager:us-east-1:123:secret:mydb-creds",
        "IAMAuth": "REQUIRED"
    }]' \
    --role-arn arn:aws:iam::123:role/RDSProxyRole \
    --vpc-subnet-ids subnet-abc123 subnet-def456 \
    --vpc-security-group-ids sg-xyz789 \
    --require-tls true

# Gắn proxy với target DB instance
aws rds register-db-proxy-targets \
    --db-proxy-name myapp-proxy \
    --db-instance-identifiers mydb-instance

# Lấy endpoint của proxy
aws rds describe-db-proxies \
    --db-proxy-name myapp-proxy \
    --query 'DBProxies[0].Endpoint'
```

### IAM Policy Cho RDS Proxy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:us-east-1:123456789:dbuser:prx-xxxxxx/lambda_user"
    }
  ]
}
```

---

## RDS Proxy với Lambda

Đây là use case phổ biến nhất và quan trọng nhất của RDS Proxy.

### Vấn Đề Lambda + RDS Không Có Proxy

```python
# Lambda function — vấn đề khi không có proxy

import pymysql

# ❌ Anti-pattern: tạo connection mỗi invocation
def lambda_handler(event, context):
    conn = pymysql.connect(host=DB_HOST, ...)  # Tốn 100ms mỗi lần!
    # ... xử lý ...
    conn.close()
```

```python
# Tốt hơn nhưng vẫn có vấn đề: reuse connection trong warm Lambda
import pymysql

conn = None  # Connection-level variable

def lambda_handler(event, context):
    global conn
    if conn is None:
        conn = pymysql.connect(host=DB_HOST, ...)
    # ... xử lý ...
```

Vấn đề: 500 concurrent Lambdas → 500 connections đến RDS.

### Với RDS Proxy

```python
import pymysql
import boto3

PROXY_ENDPOINT = "myapp-proxy.proxy-xxxxxx.us-east-1.rds.amazonaws.com"

def get_connection():
    # Kết nối đến proxy thay vì trực tiếp đến RDS
    return pymysql.connect(
        host=PROXY_ENDPOINT,
        user='lambda_user',
        password=get_iam_token(),  # IAM auth
        database='myapp',
        ssl=True
    )

def lambda_handler(event, context):
    with get_connection() as conn:
        with conn.cursor() as cursor:
            cursor.execute("SELECT * FROM orders WHERE id = %s", (event['order_id'],))
            return cursor.fetchone()
```

Kết quả: 500 concurrent Lambdas → 500 app connections đến Proxy → 50 connections đến RDS.

### Best Practices Lambda + RDS Proxy

```python
# Tốt nhất — reuse connection với context variable
import pymysql

_connection = None

def get_connection():
    global _connection
    try:
        if _connection and _connection.open:
            _connection.ping(reconnect=False)  # Test connection
            return _connection
    except:
        pass
    _connection = create_new_connection()
    return _connection

def lambda_handler(event, context):
    conn = get_connection()
    # ...
```

---

## Chi Phí và Trade-offs

### Chi Phí RDS Proxy

```
Chi phí = Số vCPUs của DB instance × Giờ chạy × Giá/vCPU-giờ

Ví dụ: db.m7g.large (2 vCPUs), chạy 730 giờ/tháng
  Proxy cost = 2 × 730 × $0.015 = $21.9/tháng
  (Chi phí thêm khoảng 10-20% so với instance cost)
```

### Trade-offs

| Lợi Ích                                   | Hạn Chế                                      |
| ------------------------------------------ | --------------------------------------------- |
| Giảm connection count đến database         | Thêm latency nhỏ (~1-2ms) do hop qua proxy   |
| Failover nhanh hơn (30s vs 120s)           | Chi phí thêm                                 |
| IAM auth, Secrets Manager integration      | Thêm một điểm failure tiềm ẩn                |
| Bảo vệ database khỏi connection storm      | Chỉ hỗ trợ MySQL, PostgreSQL, SQL Server      |
| Giảm overhead TLS handshake               | Không hỗ trợ MariaDB, Oracle, Db2            |

---

## Khi Nào Dùng RDS Proxy

### Nên Dùng

```
✅ Lambda + RDS — bắt buộc nếu nhiều concurrent invocations
✅ Microservices với nhiều instances nhỏ — mỗi instance ít connections
✅ ECS/EKS với auto-scaling — số containers biến động nhiều
✅ Cần failover nhanh hơn 120 giây
✅ Muốn tích hợp IAM auth và Secrets Manager
✅ Ứng dụng không có connection pool của riêng (stateless)
```

### Không Cần Thiết

```
❌ Monolithic application với connection pool tốt (pgBouncer, HikariCP)
❌ Ứng dụng có ít connections (<50) và ổn định
❌ Cần engine không được hỗ trợ (MariaDB, Oracle)
❌ Tối ưu chi phí tối đa — proxy thêm 10-20% chi phí
```

---

## Câu Hỏi Phỏng Vấn

**Q: Tại sao Lambda + RDS cần RDS Proxy?**

Lambda tạo connection mới mỗi invocation (hoặc mỗi container cold start). Với 1,000 concurrent Lambdas, có 1,000 connections đến RDS — vượt quá `max_connections`. RDS Proxy multiplexes nhiều Lambda connections thành ít database connections hơn, duy trì pool bền vững. Kết quả: giảm latency (không phải tạo connection mới), tránh "too many connections" error.

**Q: RDS Proxy cải thiện failover như thế nào?**

Không có proxy: khi failover, tất cả application connections bị ngắt, hàng trăm ứng dụng reconnect đồng thời → thundering herd làm database quá tải ngay sau khi phục hồi → thực tế recovery lâu hơn 120 giây.

Với proxy: proxy tự xử lý reconnection đến primary mới, ứng dụng thấy pause ngắn (~30s) và proxy tự retry thay ứng dụng.

**Q: RDS Proxy có ảnh hưởng đến latency không?**

Có, nhưng rất nhỏ — thêm ~1-2ms do network hop qua proxy. Với Lambda, tiết kiệm được 50-100ms (không phải establish connection mới) >> overhead 1-2ms. Với traditional server apps có connection pool riêng, lợi ích ít hơn và overhead vẫn là 1-2ms.

**Q: Sự khác biệt giữa RDS Proxy và pgBouncer/ProxySQL?**

| Tiêu Chí          | RDS Proxy              | pgBouncer/ProxySQL      |
| ----------------- | ---------------------- | ----------------------- |
| Managed           | ✅ Fully managed       | ❌ Tự quản lý           |
| Chi phí           | Thêm ~10-20%           | Chỉ EC2 instance cost   |
| Setup             | Dễ (vài click)         | Phức tạp hơn            |
| HA cho proxy      | ✅ Built-in            | ❌ Phải tự cấu hình     |
| IAM Integration   | ✅ Native              | ❌ Cần custom solution  |
| Flexibility       | Thấp hơn               | Cao hơn (nhiều config)  |
| Hỗ trợ engine     | MySQL, PG, SQL Server  | pgBouncer: PG only; ProxySQL: MySQL |

---

## 🔗 Điều Hướng

- **Trước:** [5-parameter-groups.md](5-parameter-groups.md) — Parameter Groups
- **Chủ đề tiếp:** [../02-aurora/README.md](../02-aurora/README.md) — Aurora
- **Liên quan:** [../07-performance-tuning/4-connection-pooling.md](../07-performance-tuning/4-connection-pooling.md) — Connection Pooling chi tiết

---

**Cập Nhật Lần Cuối:** 2026-05-15
