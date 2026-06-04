# 1 — VPC & Security Groups — Cô Lập Mạng Cho Database

> VPC (Virtual Private Cloud — Đám Mây Riêng Ảo) và Security Groups (Nhóm Bảo Mật) tạo thành lớp bảo vệ đầu tiên và quan trọng nhất cho database. Không có Network Isolation (Cô Lập Mạng) đúng cách, mọi lớp bảo mật khác đều vô nghĩa.

---

## 📚 Mục Lục

1. [VPC Fundamentals cho Database](#1-vpc-fundamentals-cho-database)
2. [Subnet Design — Thiết Kế Subnet](#2-subnet-design--thiết-kế-subnet)
3. [Security Groups cho Database](#3-security-groups-cho-database)
4. [NACLs — Network ACLs](#4-nacls--network-acls)
5. [DB Subnet Groups — Nhóm Subnet Database](#5-db-subnet-groups--nhóm-subnet-database)
6. [VPC Endpoints — Điểm Cuối VPC](#6-vpc-endpoints--điểm-cuối-vpc)
7. [VPC Flow Logs — Nhật Ký Luồng VPC](#7-vpc-flow-logs--nhật-ký-luồng-vpc)
8. [Multi-Region & Cross-VPC Connectivity](#8-multi-region--cross-vpc-connectivity)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. VPC Fundamentals Cho Database

### VPC Là Gì?

**VPC** (Virtual Private Cloud — Đám Mây Riêng Ảo) là môi trường mạng ảo riêng biệt, cô lập hoàn toàn với các VPC khác và internet, trừ khi bạn cấu hình rõ ràng.

```
AWS Cloud
├── Region: ap-southeast-1 (Singapore)
│   ├── VPC: prod-vpc (10.0.0.0/16)
│   │   ├── AZ-1a
│   │   │   ├── Public Subnet:  10.0.1.0/24  (Load Balancers)
│   │   │   ├── Private Subnet: 10.0.2.0/24  (Application Servers)
│   │   │   └── DB Subnet:      10.0.3.0/24  (Databases)
│   │   ├── AZ-1b
│   │   │   ├── Public Subnet:  10.0.4.0/24
│   │   │   ├── Private Subnet: 10.0.5.0/24
│   │   │   └── DB Subnet:      10.0.6.0/24
│   │   └── AZ-1c (optional cho 3-AZ setup)
│   │
│   └── VPC: dev-vpc (10.1.0.0/16)
│       └── ... (môi trường dev riêng biệt)
```

### Tại Sao VPC Quan Trọng Với Database?

| Không Dùng VPC Đúng | Dùng VPC Đúng |
|---------------------|---------------|
| Database có thể accessible từ internet | Database chỉ accessible từ trong VPC |
| Bất kỳ IP nào cũng có thể thử connect | Chỉ application servers mới connect được |
| Khó audit ai đã connect | VPC Flow Logs ghi lại mọi kết nối |
| Blast radius (Phạm Vi Ảnh Hưởng) lớn khi bị tấn công | Tấn công bị giới hạn trong subnet |

---

## 2. Subnet Design — Thiết Kế Subnet

### 3-Tier Subnet Architecture (Kiến Trúc 3 Tầng)

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
┌─────────────────────────────────────────────┐
│  PUBLIC SUBNET (Mạng Con Công Khai)         │
│  CIDR: 10.0.1.0/24 (AZ-a), 10.0.4.0/24 (AZ-b) │
│                                              │
│  • Application Load Balancer (ALB)           │
│  • NAT Gateway (cho outbound internet)       │
│  • Bastion Host (jump server nếu cần)        │
│                                              │
│  Route table: 0.0.0.0/0 → Internet Gateway  │
└───────────────┬─────────────────────────────┘
                │ (chỉ traffic qua ALB/NAT)
                ▼
┌─────────────────────────────────────────────┐
│  PRIVATE SUBNET — App (Mạng Con Riêng)      │
│  CIDR: 10.0.2.0/24, 10.0.5.0/24            │
│                                              │
│  • EC2 instances / ECS Tasks / Lambda        │
│  • Không có public IP                        │
│  • Outbound internet qua NAT Gateway         │
│                                              │
│  Route table: 0.0.0.0/0 → NAT Gateway       │
└───────────────┬─────────────────────────────┘
                │ (chỉ traffic từ app SG)
                ▼
┌─────────────────────────────────────────────┐
│  PRIVATE SUBNET — DB (Subnet Database)      │
│  CIDR: 10.0.3.0/24, 10.0.6.0/24            │
│                                              │
│  • RDS, Aurora, ElastiCache                  │
│  • Không có public IP, không có NAT          │
│  • KHÔNG có route ra internet                │
│                                              │
│  Route table: local only (10.0.0.0/16 local) │
└─────────────────────────────────────────────┘
```

### Nguyên Tắc Thiết Kế Subnet Cho Database

```
✅ Subnet database phải ở AZ riêng biệt (tối thiểu 2 AZ cho HA)
✅ Route table của DB subnet: KHÔNG có route ra internet (không có 0.0.0.0/0)
✅ DB subnet nên tách biệt khỏi app subnet (khác CIDR range)
✅ Dùng /24 hoặc nhỏ hơn cho DB subnet — database ít instances hơn app
✅ Tên subnet rõ ràng: prod-db-private-1a, prod-db-private-1b
```

### CIDR Planning — Lập Kế Hoạch Dải IP

```
VPC: 10.0.0.0/16 → 65,536 IPs

Phân chia hợp lý:
├── Public:      10.0.0.0/20  → 4,096 IPs (overkill cho LB)
├── App Private: 10.0.16.0/20 → 4,096 IPs
├── DB Private:  10.0.32.0/20 → 4,096 IPs (ít hơn app)
└── Reserved:    10.0.48.0/20 → dự phòng cho tương lai
```

---

## 3. Security Groups Cho Database

### Security Group Là Gì?

**Security Group** (Nhóm Bảo Mật) là stateful firewall (tường lửa có trạng thái) ở cấp instance. Stateful nghĩa là: nếu cho phép inbound traffic, response tự động được phép ra (không cần rule outbound riêng).

### Cấu Trúc Security Group Điển Hình

#### SG-ALB — Security Group Cho Load Balancer

```
Inbound Rules (Quy Tắc Inbound):
┌─────────┬────────┬────────────────────────────────────┐
│ Port    │ Source │ Mô Tả                               │
├─────────┼────────┼────────────────────────────────────┤
│ 443     │ 0.0.0.0/0 | ::/0 │ HTTPS từ internet        │
│ 80      │ 0.0.0.0/0 | ::/0 │ HTTP (redirect về 443)   │
└─────────┴────────┴────────────────────────────────────┘

Outbound Rules (Quy Tắc Outbound):
┌─────────┬──────────────────┬──────────────────────────┐
│ Port    │ Destination      │ Mô Tả                    │
├─────────┼──────────────────┼──────────────────────────┤
│ 8080    │ SG-App           │ Forward về app servers   │
└─────────┴──────────────────┴──────────────────────────┘
```

#### SG-App — Security Group Cho Application Servers

```
Inbound Rules:
┌─────────┬──────────────────┬──────────────────────────┐
│ Port    │ Source           │ Mô Tả                    │
├─────────┼──────────────────┼──────────────────────────┤
│ 8080    │ SG-ALB           │ Chỉ từ Load Balancer     │
│ 22      │ SG-Bastion       │ SSH từ bastion host only  │
└─────────┴──────────────────┴──────────────────────────┘

Outbound Rules:
┌─────────┬──────────────────┬──────────────────────────┐
│ Port    │ Destination      │ Mô Tả                    │
├─────────┼──────────────────┼──────────────────────────┤
│ 3306    │ SG-DB-MySQL      │ MySQL/Aurora              │
│ 5432    │ SG-DB-PG         │ PostgreSQL                │
│ 6379    │ SG-Cache         │ Redis                    │
│ 443     │ 0.0.0.0/0        │ HTTPS ra ngoài (API calls)│
└─────────┴──────────────────┴──────────────────────────┘
```

#### SG-DB — Security Group Cho RDS/Aurora

```
Inbound Rules:
┌─────────┬──────────────────┬──────────────────────────┐
│ Port    │ Source           │ Mô Tả                    │
├─────────┼──────────────────┼──────────────────────────┤
│ 3306    │ SG-App           │ MySQL/Aurora từ app only  │
│ 3306    │ SG-Bastion       │ Admin access từ bastion   │
└─────────┴──────────────────┴──────────────────────────┘

Outbound Rules:
┌─────────┬──────────────────┬──────────────────────────┐
│ Mô Tả                                                  │
├────────────────────────────────────────────────────────┤
│ Không cần outbound rules (stateful — response tự động) │
│ Hoặc có thể để All traffic outbound (mặc định của AWS) │
└────────────────────────────────────────────────────────┘
```

### Security Group Reference — Tham Chiếu Security Group

Thay vì dùng IP range (CIDR), dùng **Security Group ID làm source** — đây là best practice:

```bash
# ❌ Không nên — hardcode IP range, fragile khi scale
aws ec2 authorize-security-group-ingress \
  --group-id sg-db123 \
  --protocol tcp --port 3306 \
  --cidr 10.0.2.0/24  # IP range của app subnet

# ✅ Nên dùng — reference Security Group ID
aws ec2 authorize-security-group-ingress \
  --group-id sg-db123 \
  --protocol tcp --port 3306 \
  --source-group sg-app456  # Security Group của app servers
```

**Tại sao dùng SG reference tốt hơn CIDR?**

- Khi app servers scale out, instances mới vẫn có SG-App → tự động được phép
- Không bị ảnh hưởng khi IP thay đổi (Auto Scaling thay đổi IP liên tục)
- Dễ audit: "ai được phép connect?" → xem SG membership

### Port Database Theo Engine

| Database Engine | Port Mặc Định | Notes |
|----------------|---------------|-------|
| MySQL | 3306 | RDS MySQL, Aurora MySQL |
| PostgreSQL | 5432 | RDS PostgreSQL, Aurora PostgreSQL |
| MariaDB | 3306 | Tương tự MySQL |
| Oracle | 1521 | |
| SQL Server | 1433 | |
| Redis (ElastiCache) | 6379 | Cluster mode: 6379-6384 |
| Memcached | 11211 | |

---

## 4. NACLs — Network ACLs

### NACL vs Security Group

| Đặc Điểm | Security Group | NACL |
|----------|---------------|------|
| Cấp độ | Instance | Subnet |
| Stateful / Stateless | Stateful | Stateless |
| Inbound và Outbound | Riêng biệt nhưng auto-linked | Phải define cả hai |
| Allow/Deny | Chỉ Allow | Cả Allow và Deny |
| Rule ordering | Tất cả rules được evaluate | Rules theo số thứ tự, stop at first match |
| Phù hợp cho | Primary control | Defense-in-depth, block known bad IPs |

### NACL Cho DB Subnet — Defense in Depth

```
NACL Inbound Rules cho DB Subnet:
┌──────┬──────┬───────────────────┬────────────────────────┐
│ Rule │ Type │ Source            │ Action                 │
├──────┼──────┼───────────────────┼────────────────────────┤
│  100 │ TCP  │ 10.0.2.0/24       │ ALLOW (app subnet)     │
│  110 │ TCP  │ 10.0.5.0/24       │ ALLOW (app subnet AZ-b)│
│  120 │ TCP  │ 10.0.8.0/24       │ ALLOW (bastion subnet) │
│  *   │ ALL  │ 0.0.0.0/0         │ DENY  (default deny)   │
└──────┴──────┴───────────────────┴────────────────────────┘

NACL Outbound Rules cho DB Subnet:
┌──────┬──────┬───────────────────────────────────────────┐
│ Rule │ Type │ Destination & Action                       │
├──────┼──────┼───────────────────────────────────────────┤
│  100 │ TCP  │ 10.0.2.0/24, port 1024-65535 ALLOW        │
│      │      │ (ephemeral ports — cổng tạm thời cho response) │
│  *   │ ALL  │ 0.0.0.0/0 DENY                            │
└──────┴──────┴───────────────────────────────────────────┘
```

> **Lưu ý Ephemeral Ports** (Cổng Tạm Thời): NACL stateless nên response traffic cần port ngẫu nhiên cao (1024-65535). Bắt buộc phải allow trong outbound NACL.

---

## 5. DB Subnet Groups — Nhóm Subnet Database

### DB Subnet Group Là Gì?

**DB Subnet Group** là tập hợp các subnet (ít nhất 2 subnet trong 2 AZ khác nhau) mà RDS/Aurora sẽ dùng để deploy. AWS cần điều này để biết nơi có thể tạo database và standby instance.

```bash
# Tạo DB Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name "prod-db-subnets" \
  --db-subnet-group-description "Production DB Subnets - Multi-AZ" \
  --subnet-ids subnet-db1a subnet-db1b subnet-db1c

# Kết quả:
# prod-db-subnets covers:
# - subnet-db1a: ap-southeast-1a (10.0.3.0/24)
# - subnet-db1b: ap-southeast-1b (10.0.6.0/24)
# - subnet-db1c: ap-southeast-1c (10.0.9.0/24)
```

### Dùng Khi Tạo RDS

```bash
aws rds create-db-instance \
  --db-instance-identifier prod-mysql \
  --db-instance-class db.r6g.large \
  --engine mysql \
  --db-subnet-group-name prod-db-subnets \   # ← chỉ định subnet group
  --vpc-security-group-ids sg-db-mysql \      # ← chỉ định security group
  --no-publicly-accessible \                  # ← KHÔNG public
  --multi-az                                  # ← Multi-AZ cho HA
```

---

## 6. VPC Endpoints — Điểm Cuối VPC

### Tại Sao Cần VPC Endpoints?

Khi database (đặt trong private subnet) cần gọi tới dịch vụ AWS khác (Secrets Manager, S3, KMS...), traffic **mặc định** phải đi ra internet qua NAT Gateway → tốn tiền và kém an toàn.

**VPC Endpoint** cho phép traffic đi thẳng trong mạng AWS, không qua internet.

### Loại VPC Endpoints

| Loại | Hoạt Động | Dùng Cho |
|------|-----------|---------|
| **Gateway Endpoint** | Thêm vào route table | S3, DynamoDB (miễn phí) |
| **Interface Endpoint** (PrivateLink) | ENI trong subnet | Hầu hết dịch vụ AWS khác |

### Các Endpoint Cần Cho Database Security

```bash
# Secrets Manager Endpoint — bắt buộc nếu app cần lấy credentials từ private subnet
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-prod \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.ap-southeast-1.secretsmanager \
  --subnet-ids subnet-app1a subnet-app1b \
  --security-group-ids sg-endpoint \
  --private-dns-enabled  # → secretsmanager.amazonaws.com resolve về private IP

# KMS Endpoint — bắt buộc cho encryption operations từ private resources
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-prod \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.ap-southeast-1.kms \
  --subnet-ids subnet-app1a subnet-app1b \
  --security-group-ids sg-endpoint

# RDS Endpoint — cho RDS Data API (nếu dùng)
# S3 Gateway Endpoint — miễn phí, nên luôn tạo
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-prod \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.ap-southeast-1.s3 \
  --route-table-ids rtb-private-app rtb-private-db
```

### Lợi Ích VPC Endpoints

```
Không có VPC Endpoint:
App (private) → NAT Gateway → Internet → Secrets Manager
                ↑ tốn tiền: $0.045/GB + $0.045/giờ cho NAT

Có VPC Endpoint:
App (private) → Interface Endpoint → Secrets Manager (private network)
                ↑ Interface Endpoint: $0.01/giờ/AZ + $0.01/GB (rẻ hơn và an toàn hơn)
```

---

## 7. VPC Flow Logs — Nhật Ký Luồng VPC

### VPC Flow Logs Là Gì?

**VPC Flow Logs** ghi lại metadata của mọi IP traffic đi qua VPC, subnet, hoặc ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi). Không capture nội dung packet, chỉ capture metadata.

### Format Flow Log Record

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status

Ví dụ (ACCEPT):
2 123456789012 eni-abc123 10.0.2.15 10.0.3.42 52341 3306 6 20 4200 1620000000 1620000060 ACCEPT OK

Ví dụ (REJECT — ai đó thử kết nối bị chặn):
2 123456789012 eni-abc123 203.0.113.100 10.0.3.42 54321 3306 6 5 300 1620000000 1620000010 REJECT OK
```

### Kích Hoạt Flow Logs

```bash
# Enable cho toàn bộ VPC (ghi vào CloudWatch Logs)
aws ec2 create-flow-logs \
  --resource-ids vpc-prod \
  --resource-type VPC \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-destination arn:aws:logs:ap-southeast-1:123456789012:log-group:/vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flow-logs-role

# Enable cho DB subnet cụ thể (ghi vào S3 — rẻ hơn cho long-term)
aws ec2 create-flow-logs \
  --resource-ids subnet-db1a subnet-db1b \
  --resource-type Subnet \
  --traffic-type REJECT \  # Chỉ ghi REJECT để tiết kiệm storage
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-logs-bucket/db-subnets/
```

### Use Cases Của Flow Logs

```
Security Monitoring:
- Phát hiện scan port (nhiều REJECT từ cùng IP)
- Phát hiện exfiltration (data transfer bất thường ra ngoài)
- Verify security group rules hoạt động đúng

Troubleshooting:
- Tại sao app không connect được DB? → xem REJECT trong flow logs
- Traffic đang đi qua route nào?

Compliance:
- Bằng chứng auditing mọi network access vào database
```

---

## 8. Multi-Region & Cross-VPC Connectivity

### VPC Peering — Kết Nối Ngang Hàng VPC

**VPC Peering** cho phép 2 VPC giao tiếp trực tiếp qua mạng AWS (không qua internet).

```
VPC A (prod): 10.0.0.0/16
VPC B (analytics): 10.1.0.0/16

VPC Peering Connection: pcx-123

Route table VPC A: 10.1.0.0/16 → pcx-123
Route table VPC B: 10.0.0.0/16 → pcx-123

→ Analytics app trong VPC B có thể query DB trong VPC A
   (với điều kiện Security Group cho phép)
```

**Hạn chế VPC Peering:**
- Không transitive: A↔B và B↔C, nhưng A KHÔNG thể nói chuyện với C qua B
- Không hỗ trợ overlapping CIDR (CIDR không được trùng nhau)

### AWS Transit Gateway — Cổng Quá Cảnh

Khi có nhiều VPC cần kết nối, dùng **Transit Gateway** thay vì tạo nhiều VPC Peering riêng lẻ:

```
VPC A (prod)       ─┐
VPC B (staging)    ─┤── Transit Gateway ──── On-premises (VPN/Direct Connect)
VPC C (dev)        ─┤
VPC D (analytics)  ─┘

→ Mọi VPC giao tiếp qua Transit Gateway (hub-and-spoke model)
→ Centralized routing control
→ Chi phí: $0.05/attachment/giờ + $0.02/GB
```

### AWS PrivateLink — Kết Nối Dịch Vụ Riêng Tư

Dùng khi muốn expose database service cho VPC khác mà không cần VPC Peering:

```
Service Provider VPC (có RDS):
  RDS → NLB → VPC Endpoint Service

Consumer VPC (muốn dùng RDS đó):
  Interface Endpoint → NLB → RDS

→ Consumer VPC không biết gì về network của Provider VPC
→ An toàn hơn VPC Peering (không expose toàn bộ VPC)
```

---

## 9. Câu Hỏi Phỏng Vấn

**Q: Database của bạn có nên đặt trong public subnet không?**

> Không bao giờ. Database phải trong private subnet không có route ra internet. Nếu cần admin access, dùng Bastion Host hoặc AWS Systems Manager Session Manager (SSM) — không cần mở SSH/RDP ra internet.

**Q: Phân biệt Security Group và NACL. Khi nào dùng loại nào?**

> Security Group là stateful firewall ở cấp instance (chỉ allow). NACL là stateless firewall ở cấp subnet (allow và deny). Trong thực tế: Security Group là primary control (luôn dùng), NACL là defense-in-depth bổ sung — thường dùng NACL để block known malicious IPs hoặc geographic block.

**Q: App server trong private subnet cần gọi Secrets Manager để lấy DB password. Làm thế nào mà không cần NAT Gateway?**

> Tạo Interface VPC Endpoint cho Secrets Manager trong app subnet. Traffic đi thẳng qua AWS backbone network, không qua internet, không cần NAT. An toàn hơn và rẻ hơn khi có lượng lớn API calls.

**Q: Security Group của RDS chỉ cho phép từ SG của app servers. Vậy DBA muốn connect từ máy tính để admin thì làm thế nào?**

> Hai cách an toàn: (1) Bastion Host trong public subnet, có Security Group cho phép SSH từ office IP, Security Group của RDS cho phép từ Bastion SG; (2) AWS Systems Manager Session Manager — không cần mở port 22, không cần Bastion Host, audit log qua CloudTrail tốt hơn. Không bao giờ add IP cá nhân trực tiếp vào SG của RDS.

---

## 📂 Điều Hướng

| ← Trước | Chủ Đề Hiện Tại | Tiếp → |
|---------|-----------------|--------|
| [README.md](./README.md) | **1-vpc-security-groups.md** | [2-iam-authentication.md](./2-iam-authentication.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-15 | **Trạng Thái:** ✅ Hoàn Thành
