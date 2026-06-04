# Hosted Zones — Vùng Lưu Trữ DNS

> Hosted Zone (Vùng Lưu Trữ DNS) là container chứa các DNS records cho một domain cụ thể. Route 53 hỗ trợ hai loại: Public (công khai) và Private (nội bộ).

## 📚 Mục Lục

1. [Hosted Zone Là Gì?](#hosted-zone-là-gì)
2. [Public Hosted Zone](#public-hosted-zone)
3. [Private Hosted Zone](#private-hosted-zone)
4. [Split-Horizon DNS](#split-horizon-dns)
5. [Quản Lý Hosted Zone](#quản-lý-hosted-zone)
6. [Delegated Subdomains](#delegated-subdomains)
7. [Migration DNS Vào Route 53](#migration-dns-vào-route-53)

---

## Hosted Zone Là Gì?

Hosted Zone là một **namespace DNS** — tập hợp các DNS records cho một domain và các subdomain của nó.

```
Hosted Zone: example.com
├── example.com           A     93.184.216.34
├── www.example.com       CNAME example.com
├── api.example.com       A     10.0.1.100
├── mail.example.com      MX    10 mail1.example.com
└── _dkim.example.com     TXT   "v=DKIM1; k=rsa; p=..."
```

Mỗi Hosted Zone có:
- **Zone ID** (ID duy nhất, dạng `Z1234567890`)
- **4 NS records** tự động (AWS nameservers)
- **1 SOA record** tự động (Start of Authority)

---

## Public Hosted Zone

### Đặc Điểm

- Phục vụ DNS queries **từ internet**
- Ai cũng có thể query
- Phải đăng ký domain và trỏ NS về Route 53
- Chi phí: **$0.50/tháng** + chi phí queries

### Khi Tạo Public Hosted Zone

1. Tạo Hosted Zone cho `example.com` trong Route 53
2. Route 53 cấp **4 NS records** (nameservers):
   ```
   ns-123.awsdns-45.com
   ns-456.awsdns-67.net
   ns-789.awsdns-89.org
   ns-012.awsdns-01.co.uk
   ```
3. Cập nhật NS records tại domain registrar (GoDaddy, Namecheap, hoặc Route 53 Registrar)
4. DNS propagation (lan truyền) mất **24-48 giờ** để toàn cầu nhận biết

### Kiến Trúc Public Hosted Zone

```
Internet
    ↓
Query: www.example.com?
    ↓
Root Server → TLD Server (.com)
    ↓
TLD Server biết NS của example.com là ns-XXX.awsdns-YY.com
    ↓
Route 53 Authoritative Server (NS record trỏ về đây)
    ↓
Trả về: 93.184.216.34
```

### Ví Dụ Terraform

```hcl
resource "aws_route53_zone" "public" {
  name = "example.com"

  tags = {
    Environment = "production"
  }
}

resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "www.example.com"
  type    = "A"
  ttl     = 300
  records = ["93.184.216.34"]
}

# Alias record trỏ đến ALB
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "app.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.app.dns_name
    zone_id                = aws_lb.app.zone_id
    evaluate_target_health = true
  }
}
```

---

## Private Hosted Zone

### Đặc Điểm

- Phục vụ DNS queries **chỉ từ bên trong VPC** (Virtual Private Cloud)
- Không thể truy cập từ internet
- Phải liên kết (associate) với một hoặc nhiều VPC
- Chi phí: **$0.50/tháng/VPC liên kết** + queries

### Khi Nào Dùng Private Hosted Zone

```
✅ Internal service discovery:
   db.internal → 10.0.2.100 (RDS instance)
   cache.internal → 10.0.3.50 (ElastiCache)
   api.internal → 10.0.1.0/24 (internal ALB)

✅ Human-readable names cho VPC resources

✅ Microservices trong VPC giao tiếp với nhau

✅ On-premises servers cần resolve tên AWS nội bộ
   (qua Route 53 Resolver Inbound Endpoint)
```

### Yêu Cầu Bật Private Hosted Zone

VPC phải bật hai cài đặt DNS:

```
enableDnsHostnames = true
enableDnsSupport   = true
```

Kiểm tra qua AWS CLI:
```bash
aws ec2 describe-vpc-attribute \
  --vpc-id vpc-123456 \
  --attribute enableDnsHostnames

aws ec2 describe-vpc-attribute \
  --vpc-id vpc-123456 \
  --attribute enableDnsSupport
```

### Ví Dụ: Internal Service Discovery

```
Private Hosted Zone: internal.company.com
Liên kết với: VPC prod-vpc (10.0.0.0/16)

Records:
├── database.internal.company.com   A    10.0.2.100
├── redis.internal.company.com      A    10.0.3.50
├── auth.internal.company.com       A    10.0.1.200
└── payments.internal.company.com   CNAME payments-alb.us-east-1.elb.amazonaws.com
```

```hcl
# Terraform: Private Hosted Zone
resource "aws_route53_zone" "private" {
  name = "internal.company.com"

  vpc {
    vpc_id = aws_vpc.main.id
  }

  tags = {
    Environment = "production"
    Type        = "private"
  }
}

# Associate thêm VPC khác vào cùng zone
resource "aws_route53_zone_association" "secondary_vpc" {
  zone_id = aws_route53_zone.private.zone_id
  vpc_id  = aws_vpc.secondary.id
}

resource "aws_route53_record" "database" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "database.internal.company.com"
  type    = "A"
  ttl     = 60
  records = ["10.0.2.100"]
}
```

### Private Hosted Zone — Cross-Account

Khi cần share Private Hosted Zone với VPC ở **AWS account khác**:

```
Account A: Sở hữu Private Hosted Zone
Account B: Sở hữu VPC cần dùng zone đó

Bước 1 (Account A): Tạo VPC association authorization
aws route53 create-vpc-association-authorization \
  --hosted-zone-id Z123456 \
  --vpc VPCRegion=us-east-1,VPCId=vpc-789012

Bước 2 (Account B): Associate VPC vào zone
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z123456 \
  --vpc VPCRegion=us-east-1,VPCId=vpc-789012
```

---

## Split-Horizon DNS

### Khái Niệm

Split-horizon DNS (còn gọi là split-view DNS) là kỹ thuật **trả về kết quả khác nhau** cho cùng một tên miền, tùy thuộc vào nguồn gốc của query.

```
Cùng domain: api.example.com

Từ internet:          → 203.0.113.10 (Public IP của ALB)
Từ bên trong VPC:     → 10.0.1.100  (Private IP của internal ALB)
```

### Lợi Ích

- **Tiết kiệm băng thông:** Internal traffic không cần ra internet
- **Giảm latency:** Internal traffic đi thẳng, không qua internet
- **Bảo mật:** Internal endpoints không bị lộ ra ngoài
- **Đơn giản hóa:** Developers dùng cùng URL, không cần biết internal/external

### Cách Thiết Lập Trong Route 53

```
Public Hosted Zone: example.com
  api.example.com → A → 203.0.113.10 (Public ALB)
  TTL: 300

Private Hosted Zone: example.com (cùng tên domain!)
  api.example.com → A → 10.0.1.100 (Internal ALB)
  Liên kết với: VPC prod-vpc
  TTL: 60
```

**Cách hoạt động:**
- Instance trong VPC query `api.example.com` → Route 53 Resolver trong VPC → Private Zone có ưu tiên cao hơn → trả về `10.0.1.100`
- Client ngoài internet query `api.example.com` → Authoritative server → Public Zone → trả về `203.0.113.10`

### Ví Dụ Kiến Trúc Thực Tế

```
┌─────────────────────────────────────────────────────┐
│                        VPC                          │
│                                                     │
│  Private Hosted Zone: api.example.com               │
│      → 10.0.1.100 (Internal ALB)                   │
│                    ↓                               │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐       │
│  │ Service A│   │ Service B│   │ Service C│       │
│  └──────────┘   └──────────┘   └──────────┘       │
└─────────────────────────────────────────────────────┘
                         ↑ Public Internet
Public Hosted Zone: api.example.com
    → 203.0.113.10 (Public ALB)
         ↑
    External Users / Mobile Apps
```

---

## Quản Lý Hosted Zone

### Xem Hosted Zones

```bash
# List tất cả hosted zones
aws route53 list-hosted-zones

# Xem records trong một zone
aws route53 list-resource-record-sets \
  --hosted-zone-id Z123456789
```

### Tạo/Xóa Records

```bash
# Tạo record bằng change batch
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "93.184.216.34"}]
      }
    }]
  }'

# Xóa record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456789 \
  --change-batch '{
    "Changes": [{
      "Action": "DELETE",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "93.184.216.34"}]
      }
    }]
  }'
```

### Kiểm Tra Trạng Thái Thay Đổi

```bash
# DNS changes trong Route 53 là async — kiểm tra trạng thái
aws route53 get-change --id CHANGE_ID

# Kết quả có thể là: PENDING hoặc INSYNC
# INSYNC nghĩa là đã áp dụng trên tất cả Route 53 name servers
```

---

## Delegated Subdomains

### Khái Niệm

Delegation (ủy quyền) cho phép **một subdomain được quản lý bởi một hosted zone riêng**, thường ở team hoặc account khác.

```
Hosted Zone chính: example.com (quản lý bởi Platform Team)

Delegation sang:
├── dev.example.com  → Hosted Zone riêng (Dev Team)
├── staging.example.com → Hosted Zone riêng (QA Team)
└── api.example.com → Hosted Zone riêng (API Team)
```

### Cách Thiết Lập Delegation

```
Bước 1: Tạo Hosted Zone mới cho subdomain
         dev.example.com (Zone ID: Z999)
         → Nhận 4 NS records của zone mới

Bước 2: Trong zone CHA (example.com), thêm NS records:
         dev.example.com NS ns-111.awsdns-22.com
         dev.example.com NS ns-333.awsdns-44.net
         dev.example.com NS ns-555.awsdns-66.org
         dev.example.com NS ns-777.awsdns-88.co.uk

Bước 3: Dev Team quản lý records trong zone dev.example.com
         → app.dev.example.com A 10.0.1.50
         → db.dev.example.com  A 10.0.2.50
```

### Lợi Ích của Delegation

- **Phân quyền:** Mỗi team quản lý subdomain của mình
- **Độc lập:** Thay đổi trong subdomain không ảnh hưởng zone chính
- **Bảo mật:** Giới hạn quyền truy cập qua IAM per hosted zone
- **Multi-account:** Subdomain ở account khác nhau

---

## Migration DNS Vào Route 53

### Chiến Lược Migration Không Downtime

```
Bước 1: Kiểm tra TTL hiện tại
  → Nếu TTL cao (86400), cần đợi cache hết hạn

Bước 2: Tạo Hosted Zone trong Route 53 (CHƯA cập nhật NS)
  → Export tất cả DNS records từ DNS cũ
  → Import vào Route 53

Bước 3: Giảm TTL ở DNS cũ
  → Giảm tất cả records xuống 60-300 giây
  → Đợi ít nhất bằng TTL cũ (ví dụ: 24 giờ)

Bước 4: Test DNS với NS của Route 53
  → dig @ns-XXX.awsdns-YY.com www.example.com
  → Xác nhận kết quả đúng

Bước 5: Cập nhật NS tại domain registrar
  → Trỏ sang Route 53 NS records
  → Propagation mất 24-48 giờ

Bước 6: Theo dõi và tăng TTL
  → Sau 48 giờ, tăng TTL trở lại (3600 hoặc 86400)
```

### Tools Hỗ Trợ Migration

```bash
# Export records từ BIND format
# Nhiều DNS providers cho phép export zone file

# Import vào Route 53 dùng CLI
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch file://records.json

# Verify sau migration
dig +trace www.example.com
nslookup www.example.com
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Sự khác biệt giữa Public và Private Hosted Zone?

**Trả lời:**
- **Public:** Giải quyết queries từ internet, cần đăng ký domain và cập nhật NS tại registrar
- **Private:** Chỉ giải quyết queries từ VPC được liên kết, không truy cập được từ internet, không cần domain registrar

### Câu 2: Split-horizon DNS là gì và khi nào dùng?

**Trả lời:** Split-horizon DNS trả về kết quả khác nhau cho cùng domain tùy nguồn gốc query. Dùng khi:
- Internal traffic cần resolve về private IP để tránh đi ra internet
- Muốn dùng cùng URL cho internal và external nhưng trỏ đến endpoint khác
- Tạo bằng cách tạo Public và Private Hosted Zone cho cùng domain name

### Câu 3: Cần làm gì để Private Hosted Zone hoạt động?

**Trả lời:**
1. Bật `enableDnsSupport = true` trong VPC
2. Bật `enableDnsHostnames = true` trong VPC
3. Associate VPC với Private Hosted Zone
4. VPC instances dùng `VPC_CIDR + 2` làm DNS server → tự động được route đến Route 53

### Câu 4: Làm sao migrate DNS vào Route 53 mà không có downtime?

**Trả lời:**
1. Tạo Hosted Zone trong Route 53 và import tất cả records
2. Giảm TTL ở DNS cũ xuống thấp (60-300s) và đợi cache cũ hết hạn
3. Test kỹ với NS của Route 53 trước khi switch
4. Cập nhật NS tại domain registrar sang Route 53
5. Theo dõi 48 giờ rồi tăng TTL trở lại

---

## 🔗 Điều Hướng

- ← [1-dns-fundamentals.md](./1-dns-fundamentals.md) — DNS Fundamentals
- → [3-routing-policies.md](./3-routing-policies.md) — Routing Policies
- [README.md](./README.md) — Tổng quan section

---

**Cập Nhật Lần Cuối:** 2026-05-14
