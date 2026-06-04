# Bảo Mật Mạng CSDL

Ngăn chặn truy cập trái phép từ mạng — tầng phòng thủ đầu tiên.

## Nguyên Tắc Network Security

```
Defense in Depth (Phòng thủ theo chiều sâu):

Internet
    ↓
[WAF / DDoS Protection]
    ↓
[Load Balancer / API Gateway]
    ↓
[Application Servers] — Private Subnet
    ↓
[Database] — Isolated Private Subnet
             (Không có route ra internet!)
```

---

## Phân Vùng Mạng (Network Segmentation)

### Architecture Cơ Bản

```
VPC/Network: 10.0.0.0/16

Public Subnet (10.0.0.0/24):
  - Load Balancer
  - Bastion/Jump Host
  - NAT Gateway

Private Subnet App (10.0.1.0/24):
  - Application servers
  - Không có public IP

Private Subnet DB (10.0.2.0/24):
  - Database servers
  - Không có public IP
  - Không có route ra internet
  - Chỉ app subnet được phép kết nối vào
```

### AWS Security Groups

```bash
# Security Group cho Database:
aws ec2 create-security-group \
    --group-name "sg-database" \
    --description "Database security group"

# Chỉ cho phép port 5432 từ app servers:
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxx \
    --protocol tcp \
    --port 5432 \
    --source-group sg-app-servers  # Security group của app servers

# Cho phép DBA từ bastion host:
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxx \
    --protocol tcp \
    --port 5432 \
    --cidr 10.0.0.100/32  # IP của bastion host

# KHÔNG cho phép 0.0.0.0/0 (internet)!
```

### PostgreSQL: Giới Hạn Listen Address

```ini
# postgresql.conf:

# Chỉ lắng nghe trên mạng nội bộ (KHÔNG lắng nghe 0.0.0.0!):
listen_addresses = '10.0.2.10'    # Chỉ IP nội bộ của DB server

# Hoặc nhiều interface:
listen_addresses = 'localhost,10.0.2.10'

# Không bao giờ:
# listen_addresses = '*'  ← Lắng nghe tất cả interface!
```

---

## pg_hba.conf: Kiểm Soát Kết Nối

### Cấu Trúc pg_hba.conf

```
# TYPE  DATABASE    USER        ADDRESS         METHOD
host    mydb        app_user    10.0.1.0/24     md5
host    mydb        dba_user    10.0.0.100/32   md5
local   all         postgres                    peer
```

### Ví Dụ Thực Tế

```ini
# pg_hba.conf

# Kết nối local (Unix socket): Chỉ cho postgres user
local   all             postgres                                peer

# Ứng dụng từ app servers:
host    mydb    myapp_api       10.0.1.0/24     scram-sha-256
host    mydb    myapp_worker    10.0.1.0/24     scram-sha-256

# Read replica từ reporting service:
host    mydb    myapp_reports   10.0.1.50/32    scram-sha-256

# DBA chỉ từ bastion host:
host    mydb    dba_user        10.0.0.100/32   scram-sha-256

# Replication từ replica servers:
host    replication     replicator      10.0.2.20/32    scram-sha-256

# TỪCHỐI tất cả kết nối khác:
host    all             all             all             reject
```

### Authentication Methods

```
trust      - Không cần mật khẩu (CHỈ dùng cho local dev!)
peer       - Dùng OS username (chỉ Unix socket)
md5        - Mật khẩu MD5 (không an toàn, dùng scram-sha-256)
scram-sha-256 - Mật khẩu SCRAM (KHUYẾN NGHỊ)
cert       - Client certificate (Rất an toàn, phức tạp hơn)
ldap       - LDAP authentication
radius     - RADIUS authentication
```

---

## SSL/TLS Configuration

### Server TLS

```bash
# Tạo self-signed certificate (chỉ cho dev):
openssl req -new -x509 -days 365 -nodes \
    -out server.crt \
    -keyout server.key \
    -subj "/CN=db.internal.example.com"

chmod 600 server.key
chown postgres:postgres server.key server.crt

# Production: Dùng Let's Encrypt hoặc internal CA
# Internal CA (cho private networks):
# 1. Tạo CA key và cert
# 2. Sign server certificate với CA
# 3. Distribute CA cert cho clients
```

```ini
# postgresql.conf:
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'ca.crt'           # CA cert cho verify client certs
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'HIGH:!aNULL:!MD5:!RC4'
ssl_prefer_server_ciphers = on

# pg_hba.conf: Bắt buộc SSL:
hostssl    mydb    all    10.0.1.0/24    scram-sha-256
hostnossl  all     all    all            reject
```

### Client Certificate Authentication

```bash
# Tạo client certificate cho app:
openssl req -new -nodes \
    -out myapp.csr \
    -keyout myapp.key \
    -subj "/CN=myapp_api"

# Sign với CA:
openssl x509 -req -days 365 \
    -in myapp.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out myapp.crt
```

```ini
# pg_hba.conf với client cert:
hostssl    mydb    myapp_api    10.0.1.0/24    cert clientcert=verify-full
# Chỉ cho phép kết nối nếu client có certificate được ký bởi CA
```

---

## VPN & Bastion Host

### Bastion Host Pattern

```
Internet
    ↓
[Bastion Host] ← DBA kết nối SSH vào đây
    ↓ (SSH tunnel)
[Database] ← Không có kết nối trực tiếp từ internet

Cách DBA kết nối:
1. SSH vào bastion: ssh -i key.pem user@bastion.example.com
2. Từ bastion: psql -h 10.0.2.10 -U dba_user mydb

Hoặc SSH tunnel:
ssh -L 5432:10.0.2.10:5432 user@bastion.example.com
psql -h localhost -p 5432 -U dba_user mydb
```

### AWS Session Manager (Không Cần Public Bastion)

```bash
# AWS SSM Session Manager: Kết nối không cần open port 22!

# Port forwarding đến database:
aws ssm start-session \
    --target i-xxxxxxxxxxxxx \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{"host":["10.0.2.10"],"portNumber":["5432"],"localPortNumber":["5432"]}'

# Sau đó kết nối local:
psql -h localhost -p 5432 -U dba_user mydb
```

### VPN Access

```bash
# OpenVPN hoặc WireGuard cho DBA access

# WireGuard server (trên bastion):
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]  # DBA laptop
PublicKey = DBA_PUBLIC_KEY
AllowedIPs = 10.8.0.2/32

# WireGuard client (DBA laptop):
[Interface]
Address = 10.8.0.2/24
PrivateKey = DBA_PRIVATE_KEY

[Peer]  # Server
PublicKey = SERVER_PUBLIC_KEY
AllowedIPs = 10.0.2.0/24  # Chỉ route database subnet qua VPN
Endpoint = vpn.example.com:51820
```

---

## Firewall Rules

### Iptables (On-Premise)

```bash
# Chỉ cho phép kết nối PostgreSQL từ app servers:
iptables -A INPUT -p tcp --dport 5432 \
    -s 10.0.1.0/24 -j ACCEPT

# Cho phép từ bastion:
iptables -A INPUT -p tcp --dport 5432 \
    -s 10.0.0.100/32 -j ACCEPT

# Từ chối tất cả còn lại:
iptables -A INPUT -p tcp --dport 5432 -j DROP

# Lưu rules:
iptables-save > /etc/iptables/rules.v4
```

### AWS Network ACL (Tầng Subnet)

```
Inbound Rules (cho Database Subnet):
Rule# | Protocol | Port | Source          | Allow/Deny
100   | TCP      | 5432 | 10.0.1.0/24    | ALLOW  (app servers)
110   | TCP      | 5432 | 10.0.0.100/32  | ALLOW  (bastion)
*     | ALL      | ALL  | 0.0.0.0/0      | DENY   (mọi thứ khác)

Outbound Rules:
100   | TCP      | 1024-65535 | 10.0.1.0/24 | ALLOW  (response traffic)
*     | ALL      | ALL         | 0.0.0.0/0   | DENY
```

---

## Connection Pooler Proxy

```
Không kết nối trực tiếp từ app đến database!
Dùng connection pooler làm proxy:

App Servers → PgBouncer (10.0.1.100:5432) → PostgreSQL (10.0.2.10:5432)

Lợi ích:
✓ Giảm số kết nối đến database
✓ Thêm tầng security (PgBouncer có auth riêng)
✓ Circuit breaker nếu DB quá tải
✓ Connection multiplexing
```

```ini
# pgbouncer.ini:
[databases]
mydb = host=10.0.2.10 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 10.0.1.100
listen_port = 5432

# TLS từ app đến pgbouncer:
client_tls_sslmode = require
client_tls_cert_file = /etc/pgbouncer/server.crt
client_tls_key_file = /etc/pgbouncer/server.key

# TLS từ pgbouncer đến postgresql:
server_tls_sslmode = verify-full
server_tls_ca_file = /etc/pgbouncer/ca.crt
```

---

## Giám Sát Kết Nối

```sql
-- Xem tất cả kết nối hiện tại:
SELECT
    pid,
    usename,
    client_addr,
    client_port,
    application_name,
    state,
    query_start,
    LEFT(query, 60) AS query
FROM pg_stat_activity
WHERE client_addr IS NOT NULL  -- Chỉ kết nối network (không phải local)
ORDER BY query_start;

-- Kết nối từ IP không quen:
SELECT DISTINCT client_addr, usename, COUNT(*) AS connections
FROM pg_stat_activity
WHERE client_addr NOT IN (
    '10.0.1.10', '10.0.1.11',  -- App servers đã biết
    '10.0.0.100'                 -- Bastion host
)
AND client_addr IS NOT NULL
GROUP BY 1, 2
ORDER BY connections DESC;
-- Nếu có kết quả → Điều tra ngay!
```

---

## Checklist Bảo Mật Mạng

- [ ] Database trong private subnet, không có public IP
- [ ] Security groups chỉ cho phép app servers và bastion
- [ ] listen_addresses giới hạn (không phải '*')
- [ ] pg_hba.conf chỉ whitelist IP đã biết
- [ ] TLS 1.2+ bắt buộc cho tất cả kết nối (hostssl)
- [ ] Không có hostnossl hoặc trust authentication trong production
- [ ] VPN hoặc bastion host cho DBA access
- [ ] Connection pooler làm proxy (không kết nối trực tiếp)
- [ ] Network ACL bảo vệ tầng subnet
- [ ] Monitor kết nối từ IP không quen
- [ ] Firewall rules được review và update định kỳ

---

> **Điểm Mấu Chốt:** Database không bao giờ nên tiếp xúc trực tiếp với internet. Nhiều lớp mạng — security group, network ACL, firewall, VPN — tạo nên phòng thủ chiều sâu. Nếu một lớp thất bại, các lớp khác vẫn bảo vệ.
