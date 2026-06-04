# IPv6 trong VPC — Dual-Stack Architecture (Kiến Trúc Hai Ngăn Xếp)

> **Thuộc topic:** 07-advanced-networking | **Mức độ:** Nâng cao

---

## 1. Tại Sao Cần IPv6?

### Vấn đề cạn kiệt IPv4

IPv4 có **4,294,967,296** địa chỉ (~4.3 tỷ). Thoạt nghe có vẻ nhiều, nhưng:
- Nhiều địa chỉ được dành riêng (private ranges, loopback, multicast)
- Số thiết bị kết nối internet tăng theo hàm mũ (IoT, mobile, cloud)
- **IANA (Internet Assigned Numbers Authority — Cơ Quan Quản Lý Số Internet Được Giao)** phân bổ IPv4 cuối cùng cho các RIR vào **tháng 2/2011**
- Châu Á-Thái Bình Dương (APNIC) hết IPv4 pool từ **tháng 4/2011**

**IPv6 giải quyết:** Không gian địa chỉ **2^128 = 340 undecillion** địa chỉ (~340 × 10^36) — thực tế là vô hạn.

### Các lý do triển khai IPv6 trong AWS

| Lý do | Chi tiết |
|-------|---------|
| **Hết IPv4** | Public IPv4 khan hiếm, chi phí tăng (AWS thu phí $0.005/giờ/IP từ 2024) |
| **IoT** | Hàng tỷ thiết bị cần IP riêng, không thể dùng NAT mãi |
| **Yêu cầu pháp lý** | Nhiều chính phủ (Mỹ, EU) yêu cầu hạ tầng công hỗ trợ IPv6 |
| **Hiệu suất** | Không cần NAT → routing đơn giản hơn, latency thấp hơn |
| **Modern networking** | Cloud-native architecture khuyến khích native connectivity |
| **Chi phí** | Từ 2024, AWS tính phí public IPv4 → IPv6 miễn phí (không tính phí tương tự) |

---

## 2. IPv6 Basics trong Ngữ Cảnh AWS

### Định dạng địa chỉ IPv6

```
IPv4: 192.168.1.1           (32 bit, ký hiệu thập phân)
IPv6: 2406:da18:xxx:xxxx::/56  (128 bit, ký hiệu thập lục phân)

Đầy đủ:  2406:da18:0000:0000:0000:0000:0000:0001
Rút gọn: 2406:da18::1   (:: thay thế chuỗi 0000 liên tiếp)
```

### AWS cấp IPv6 như thế nào?

**Quan trọng:** AWS không cho phép bạn chọn IPv6 CIDR tùy ý. AWS cấp từ **Amazon-owned IPv6 prefix pool**:

```
VPC: /56 IPv6 CIDR  (cố định bởi AWS)
  Ví dụ: 2406:da18:abc:1234::/56
  
Mỗi Subnet: /64 IPv6 CIDR (chọn trong phạm vi /56 của VPC)
  Subnet-A: 2406:da18:abc:1234::/64
  Subnet-B: 2406:da18:abc:1235::/64
  Subnet-C: 2406:da18:abc:1236::/64
  ... (tối đa 256 subnets /64 trong một /56)
```

**Lý do cố định /56 cho VPC và /64 cho subnet:**
- /64 là đơn vị cơ bản trong IPv6 (SLAAC — Stateless Address Autoconfiguration yêu cầu /64)
- /56 cho VPC đủ để tạo 256 subnets /64
- AWS quản lý pool toàn cầu, đảm bảo không overlap

### IPv6 Address Types trong AWS

| Loại | Mô tả | AWS dùng |
|------|-------|----------|
| **GUA** — Global Unicast Address (Địa Chỉ Đơn Hướng Toàn Cầu) | 2000::/3 — Routable toàn cầu | Có — đây là loại AWS cấp |
| **ULA** — Unique Local Address (Địa Chỉ Cục Bộ Duy Nhất) | fc00::/7 — Tương đương private IPv4 | **Có** từ 2023 (IPAM-managed) |
| **Link-local** | fe80::/10 — Chỉ trong local link | Internal use |
| **Loopback** | ::1 | Localhost |

**AWS GUA luôn public routable** — không có khái niệm "private IPv6 GUA" như IPv4 private ranges. Đây là lý do quan trọng cần Egress-Only Internet Gateway.

---

## 3. Dual-Stack VPC — Kiến Trúc Hai Ngăn Xếp

**Dual-Stack** nghĩa là VPC/subnet/instance hỗ trợ **cả IPv4 và IPv6 đồng thời**.

```
Dual-Stack VPC
┌────────────────────────────────────────────────────────────────┐
│  IPv4 CIDR: 10.0.0.0/16                                        │
│  IPv6 CIDR: 2406:da18:abc:1234::/56                            │
│                                                                  │
│  Public Subnet                                                   │
│  IPv4: 10.0.1.0/24                                              │
│  IPv6: 2406:da18:abc:1234::/64                                  │
│  ┌──────────────────────────────────┐                           │
│  │  EC2 Instance                    │                           │
│  │  Private IPv4: 10.0.1.50         │                           │
│  │  Public IPv4: 54.xxx.xxx.xxx     │ ←→ IPv4 Internet (IGW)   │
│  │  IPv6 GUA: 2406:da18:abc:1234::50│ ←→ IPv6 Internet (IGW)   │
│  └──────────────────────────────────┘                           │
│                                                                  │
│  Private Subnet                                                  │
│  IPv4: 10.0.2.0/24                                              │
│  IPv6: 2406:da18:abc:1235::/64                                  │
│  ┌──────────────────────────────────┐                           │
│  │  EC2 Instance                    │                           │
│  │  Private IPv4: 10.0.2.60         │ → IPv4 Internet (NAT GW) │
│  │  IPv6 GUA: 2406:da18:abc:1235::60│ → IPv6 Internet (EIGW)   │
│  └──────────────────────────────────┘                           │
└────────────────────────────────────────────────────────────────┘
```

---

## 4. Egress-Only Internet Gateway (EIGW — Cổng Internet Chỉ Ra)

**EIGW** là Internet Gateway chỉ cho phép traffic **ra** (egress) từ IPv6, không cho phép traffic **vào** (ingress) từ internet — tương đương NAT Gateway nhưng cho IPv6.

### Tại sao cần EIGW?

Vì IPv6 GUA luôn là public routable, nếu chỉ dùng Internet Gateway thông thường:

```
Instance với IPv6 GUA → Internet Gateway → Có thể bị kết nối từ bên ngoài!
(Không an toàn cho servers trong private subnet)
```

Với EIGW:

```
Instance với IPv6 GUA → EIGW → Ra được internet (để update packages, gọi API)
Internet               → EIGW → BỊ BLOCK, không vào được instance
```

### So sánh: NAT Gateway vs Egress-Only IGW

| Khía cạnh | NAT Gateway | Egress-Only IGW |
|-----------|-------------|----------------|
| **IP version** | IPv4 | IPv6 |
| **Hướng traffic** | Outbound only (stateful) | Outbound only (stateful) |
| **Masquerade IP** | Có — che IP private nguồn | Không — GUA của instance vẫn là nguồn |
| **Chi phí** | $0.059/giờ + $0.059/GB | **Miễn phí** (không tính phí) |
| **Managed service** | Có | Có |
| **Highly available** | Trong AZ (dùng nhiều để HA) | Regional (HA tự động) |

**Quan trọng:** EIGW miễn phí và là regional resource (1 EIGW cho toàn VPC, tất cả AZs).

---

## 5. Route Tables với IPv6

Mỗi route table cần thêm routes cho IPv6:

### Public Subnet Route Table

```
Destination          Target
10.0.0.0/16          local
0.0.0.0/0            igw-xxxxxxxx      ← IPv4 ra internet
::/0                 igw-xxxxxxxx      ← IPv6 ra internet (thêm dòng này)
```

### Private Subnet Route Table

```
Destination          Target
10.0.0.0/16          local
0.0.0.0/0            nat-xxxxxxxx      ← IPv4 ra internet qua NAT
::/0                 eigw-xxxxxxxx     ← IPv6 ra internet qua EIGW (thêm dòng này)
```

```bash
# Thêm IPv6 route cho public subnet
aws ec2 create-route \
  --route-table-id rtb-public-0a1b2c3d \
  --destination-ipv6-cidr-block ::/0 \
  --gateway-id igw-0a1b2c3d4e5f67890

# Thêm IPv6 route cho private subnet (qua EIGW)
aws ec2 create-route \
  --route-table-id rtb-private-0a1b2c3d \
  --destination-ipv6-cidr-block ::/0 \
  --egress-only-internet-gateway-id eigw-0a1b2c3d4e5f67890
```

---

## 6. Cấu Hình IPv6 Trên VPC và Subnet

### Bật IPv6 trên VPC hiện tại

```bash
# Bước 1: Gán IPv6 CIDR cho VPC
aws ec2 associate-vpc-cidr-block \
  --vpc-id vpc-0123456789abcdef0 \
  --amazon-provided-ipv6-cidr-block

# Đợi CIDR được gán (state: associated)
aws ec2 describe-vpcs \
  --vpc-ids vpc-0123456789abcdef0 \
  --query 'Vpcs[0].Ipv6CidrBlockAssociationSet'

# Output:
# [{
#   "AssociationId": "vpc-cidr-assoc-xxxx",
#   "Ipv6CidrBlock": "2406:da18:abc:1234::/56",
#   "Ipv6CidrBlockState": {"State": "associated"}
# }]

# Bước 2: Gán /64 prefix cho subnet
aws ec2 associate-subnet-cidr-block \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --ipv6-cidr-block 2406:da18:abc:1234::/64

# Bước 3: Bật auto-assign IPv6 cho subnet
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --assign-ipv6-address-on-creation

# Bước 4: Tạo Egress-Only Internet Gateway
aws ec2 create-egress-only-internet-gateway \
  --vpc-id vpc-0123456789abcdef0

# Bước 5: Thêm routes (xem section 5 bên trên)
```

### Instance nhận IPv6 như thế nào?

**Tự động (SLAAC — Stateless Address Autoconfiguration):**
```
Instance boot → Gửi Router Solicitation → 
Nhận Router Advertisement từ VPC router →
Tự tạo IPv6 address từ /64 prefix + EUI-64 (từ MAC address)
```

**Hoặc gán thủ công:**
```bash
# Gán IPv6 cụ thể cho instance
aws ec2 assign-ipv6-addresses \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --ipv6-addresses 2406:da18:abc:1234::100
```

---

## 7. Security Groups với IPv6

Security Groups hoạt động **stateful** với cả IPv4 và IPv6, nhưng **phải thêm rules riêng cho IPv6**. Rules IPv4 không tự động áp dụng cho IPv6.

```bash
# Cho phép HTTP từ IPv4
aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# PHẢI THÊM riêng cho IPv6
aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --protocol tcp --port 80 --ipv6-cidr ::/0

# Cho phép HTTPS (cả IPv4 và IPv6)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --ip-permissions \
    '[{"IpProtocol":"tcp","FromPort":443,"ToPort":443,"IpRanges":[{"CidrIp":"0.0.0.0/0"}],"Ipv6Ranges":[{"CidrIpv6":"::/0"}]}]'

# SSH chỉ từ office (IPv4 và IPv6)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0a1b2c3d4e5f67890 \
  --ip-permissions \
    '[{"IpProtocol":"tcp","FromPort":22,"ToPort":22,
       "IpRanges":[{"CidrIp":"203.0.113.0/24","Description":"Office IPv4"}],
       "Ipv6Ranges":[{"CidrIpv6":"2001:db8::/32","Description":"Office IPv6"}]}]'
```

**Common Pitfall:** Quên thêm IPv6 rules → website không accessible từ IPv6-only clients.

---

## 8. Network ACLs với IPv6

Network ACLs (NACLs — Danh Sách Kiểm Soát Truy Cập Mạng) là **stateless** — phải thêm rules cho cả inbound và outbound, và phải thêm rules riêng cho IPv6.

### Ví dụ NACL cho Dual-Stack Public Subnet

**Inbound Rules:**

| Rule # | Type | Protocol | Port | Source | Action |
|--------|------|----------|------|--------|--------|
| 100 | IPv4 HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 101 | IPv6 HTTP | TCP | 80 | ::/0 | ALLOW |
| 110 | IPv4 HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 111 | IPv6 HTTPS | TCP | 443 | ::/0 | ALLOW |
| 120 | IPv4 Ephemeral | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| 121 | IPv6 Ephemeral | TCP | 1024-65535 | ::/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |
| * | All | All | All | ::/0 | DENY |

**Outbound Rules:**

| Rule # | Type | Protocol | Port | Destination | Action |
|--------|------|----------|------|-------------|--------|
| 100 | IPv4 HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 101 | IPv6 HTTP | TCP | 80 | ::/0 | ALLOW |
| 110 | IPv4 HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 111 | IPv6 HTTPS | TCP | 443 | ::/0 | ALLOW |
| 120 | IPv4 Ephemeral | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| 121 | IPv6 Ephemeral | TCP | 1024-65535 | ::/0 | ALLOW |

**Lưu ý:** Ephemeral ports (1024-65535) cần được allow cho response traffic vì NACL stateless — response từ server ra ngoài dùng ephemeral port.

---

## 9. IPv6 với Load Balancers

### Application Load Balancer (ALB) — Dual-Stack

ALB hỗ trợ **dualstack** — nhận cả IPv4 và IPv6 requests:

```bash
# Tạo ALB dual-stack
aws elbv2 create-load-balancer \
  --name my-dualstack-alb \
  --type application \
  --scheme internet-facing \
  --ip-address-type dualstack \   # IPv4 và IPv6
  --subnets subnet-ipv4-a subnet-ipv4-b

# Dualstack ALB có DNS name dạng:
# dualstack.my-dualstack-alb-xxxxx.ap-southeast-1.elb.amazonaws.com
# DNS này resolve ra cả A record (IPv4) và AAAA record (IPv6)
```

Khi client IPv6 kết nối:
```
IPv6 Client → ALB (nhận IPv6 traffic) → Targets (EC2/ECS) qua IPv4
(ALB làm dual-stack termination, targets có thể chỉ dùng IPv4)
```

### Network Load Balancer (NLB) — Dual-Stack

```bash
aws elbv2 create-load-balancer \
  --name my-dualstack-nlb \
  --type network \
  --scheme internet-facing \
  --ip-address-type dualstack \
  --subnets subnet-a subnet-b
```

**NLB Dual-Stack với UDP:**
NLB dual-stack hỗ trợ TCP, UDP, và TCP_UDP protocols — phù hợp cho gaming servers, DNS, VoIP cần IPv6.

### Gateway Load Balancer (GWLB) với IPv6

GWLB (phiên bản mới) hỗ trợ IPv6 cho transparent network appliances:

```
IPv6 Traffic → GWLB (IPv6) → Network Appliance → GWLB → Đích
```

---

## 10. IPv6 với CloudFront

CloudFront hỗ trợ nhận **IPv6 requests** từ client:

```bash
# Bật IPv6 cho CloudFront distribution
aws cloudfront create-distribution --distribution-config '{
  "IsIPV6Enabled": true,
  ...
}'
```

**Flow với CloudFront dual-stack:**
```
IPv6 Client → CloudFront Edge (IPv6) → Origin (IPv4 hoặc IPv6)
```

**Lưu ý quan trọng:** Nếu dùng **Lambda@Edge** hoặc **CloudFront Functions** để check `X-Forwarded-For` hoặc client IP, IPv6 address có thể xuất hiện ở đó — cần code xử lý đúng format IPv6.

**WAF và IPv6:**
```bash
# AWS WAF rule cho IPv6 range
{
  "Name": "BlockSpecificIPv6",
  "Priority": 1,
  "Action": {"Block": {}},
  "Statement": {
    "IPSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:...:ipset/ipv6-blocked-ranges/..."
    }
  }
}
```

---

## 11. Migration Path — Lộ Trình Chuyển Đổi

### Giai đoạn 1: IPv4-only (Hiện tại của nhiều tổ chức)

```
Tất cả resources chỉ dùng IPv4
Private IPs: RFC 1918 (10.x, 172.16.x, 192.168.x)
Internet access: qua NAT Gateway
```

### Giai đoạn 2: Dual-Stack (Khuyến nghị ngay bây giờ)

```
Bước 1: Enable IPv6 CIDR trên VPC
Bước 2: Assign /64 cho từng subnet
Bước 3: Update route tables (thêm ::/0 routes)
Bước 4: Update Security Groups (thêm IPv6 rules)
Bước 5: Update NACLs (thêm IPv6 rules)
Bước 6: Enable IPv6 trên Load Balancers
Bước 7: Update DNS (thêm AAAA records)
Bước 8: Test với IPv6 clients
```

### Giai đoạn 3: IPv6-Preferred (Tương lai)

```
Mọi traffic ưu tiên IPv6 khi có thể
IPv4 chỉ là fallback
Giảm dần phụ thuộc vào NAT Gateway
```

### Giai đoạn 4: IPv6-only (Dài hạn)

```
Loại bỏ hoàn toàn IPv4
Chỉ dùng IPv6 GUA hoặc ULA
Không cần NAT Gateway
```

**AWS IPv6-only Subnets (từ 2021):**

```bash
# Tạo subnet chỉ dùng IPv6
aws ec2 create-subnet \
  --vpc-id vpc-0123456789abcdef0 \
  --ipv6-native \
  --ipv6-cidr-block 2406:da18:abc:1234::/64 \
  --no-map-public-ip-on-launch
```

Instances trong IPv6-only subnet:
- Chỉ có IPv6 address, không có IPv4
- Vẫn cần DNS64 và NAT64 để giao tiếp với IPv4-only services
- AWS cung cấp **DNS64** (64:ff9b::/96) và **NAT64** tích hợp trong VPC

---

## 12. DNS64 và NAT64 cho IPv6-only Subnets

Khi instance IPv6-only cần gọi service IPv4-only (ví dụ: dịch vụ bên thứ ba chưa hỗ trợ IPv6):

```
IPv6 Instance → DNS64 (synthesize AAAA từ A record)
              → NAT64 (translate IPv6 → IPv4 tại biên)
              → IPv4 Service
```

**Cấu hình trong VPC:**

```bash
# Bật DNS64 cho subnet
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --enable-dns64

# Route traffic qua NAT Gateway (NAT Gateway tự xử lý NAT64)
aws ec2 create-route \
  --route-table-id rtb-0a1b2c3d \
  --destination-ipv6-cidr-block 64:ff9b::/96 \
  --nat-gateway-id nat-0a1b2c3d4e5f67890
```

**Flow với DNS64/NAT64:**
```
IPv6 App gọi: api.example.com (chỉ có A record IPv4: 203.0.113.10)
DNS64: synthesize AAAA → 64:ff9b::203.0.113.10
IPv6 App gửi packet đến 64:ff9b::203.0.113.10
NAT Gateway: nhận packet, translate IPv6 → IPv4, gửi đến 203.0.113.10
Response: NAT Gateway dịch IPv4 → IPv6, trả về App
```

---

## 13. Terraform Example — Dual-Stack VPC Đầy Đủ

```hcl
# ========================================
# VPC với Dual-Stack (IPv4 + IPv6)
# ========================================
resource "aws_vpc" "dualstack" {
  cidr_block                       = "10.0.0.0/16"
  assign_generated_ipv6_cidr_block = true   # AWS tự gán /56 IPv6
  enable_dns_hostnames             = true
  enable_dns_support               = true

  tags = { Name = "dualstack-vpc" }
}

# ========================================
# Internet Gateway (cho cả IPv4 và IPv6)
# ========================================
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.dualstack.id
  tags   = { Name = "dualstack-igw" }
}

# ========================================
# Egress-Only Internet Gateway (chỉ cho IPv6)
# ========================================
resource "aws_egress_only_internet_gateway" "ipv6" {
  vpc_id = aws_vpc.dualstack.id
  tags   = { Name = "eigw-ipv6" }
}

# ========================================
# Public Subnets — Dual-Stack
# ========================================
resource "aws_subnet" "public_a" {
  vpc_id                          = aws_vpc.dualstack.id
  cidr_block                      = "10.0.1.0/24"
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.dualstack.ipv6_cidr_block, 8, 1)
  # cidrsubnet tính /64 từ /56: VPC /56 + 8 bits = /64
  availability_zone               = "ap-southeast-1a"
  map_public_ip_on_launch         = true
  assign_ipv6_address_on_creation = true   # Auto-assign IPv6 cho instances

  tags = { Name = "public-a-dualstack" }
}

resource "aws_subnet" "public_b" {
  vpc_id                          = aws_vpc.dualstack.id
  cidr_block                      = "10.0.2.0/24"
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.dualstack.ipv6_cidr_block, 8, 2)
  availability_zone               = "ap-southeast-1b"
  map_public_ip_on_launch         = true
  assign_ipv6_address_on_creation = true

  tags = { Name = "public-b-dualstack" }
}

# ========================================
# Private Subnets — Dual-Stack
# ========================================
resource "aws_subnet" "private_a" {
  vpc_id                          = aws_vpc.dualstack.id
  cidr_block                      = "10.0.11.0/24"
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.dualstack.ipv6_cidr_block, 8, 11)
  availability_zone               = "ap-southeast-1a"
  assign_ipv6_address_on_creation = true

  tags = { Name = "private-a-dualstack" }
}

resource "aws_subnet" "private_b" {
  vpc_id                          = aws_vpc.dualstack.id
  cidr_block                      = "10.0.12.0/24"
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.dualstack.ipv6_cidr_block, 8, 12)
  availability_zone               = "ap-southeast-1b"
  assign_ipv6_address_on_creation = true

  tags = { Name = "private-b-dualstack" }
}

# ========================================
# NAT Gateways cho IPv4 Private Subnets
# ========================================
resource "aws_eip" "nat_a" {
  domain = "vpc"
}

resource "aws_nat_gateway" "nat_a" {
  allocation_id = aws_eip.nat_a.id
  subnet_id     = aws_subnet.public_a.id
  tags          = { Name = "nat-gw-a" }
  depends_on    = [aws_internet_gateway.main]
}

# ========================================
# Route Tables
# ========================================

# Public Route Table — IPv4 + IPv6 ra internet qua IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.dualstack.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  route {
    ipv6_cidr_block = "::/0"
    gateway_id      = aws_internet_gateway.main.id
  }

  tags = { Name = "public-rt-dualstack" }
}

resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

# Private Route Table AZ-A — IPv4 qua NAT, IPv6 qua EIGW
resource "aws_route_table" "private_a" {
  vpc_id = aws_vpc.dualstack.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat_a.id
  }

  route {
    ipv6_cidr_block        = "::/0"
    egress_only_gateway_id = aws_egress_only_internet_gateway.ipv6.id
  }

  tags = { Name = "private-rt-a-dualstack" }
}

resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private_a.id
}

# ========================================
# Security Group — Dual-Stack
# ========================================
resource "aws_security_group" "web_dualstack" {
  name   = "web-dualstack-sg"
  vpc_id = aws_vpc.dualstack.id

  # HTTP từ IPv4
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP từ IPv6 — PHẢI thêm riêng!
  ingress {
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    ipv6_cidr_blocks = ["::/0"]
  }

  # HTTPS từ IPv4
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS từ IPv6
  ingress {
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    ipv6_cidr_blocks = ["::/0"]
  }
}

# ========================================
# ALB Dual-Stack
# ========================================
resource "aws_lb" "dualstack_alb" {
  name               = "dualstack-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.web_dualstack.id]
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]
  ip_address_type    = "dualstack"   # Bật IPv6 cho ALB

  tags = { Name = "dualstack-alb" }
}

# Output IPv6 CIDR của VPC
output "vpc_ipv6_cidr" {
  value = aws_vpc.dualstack.ipv6_cidr_block
}
```

---

## 14. Kiểm Tra IPv6 Connectivity

```bash
# Từ EC2 instance, kiểm tra địa chỉ IPv6
ip -6 addr show eth0
# Kết quả: inet6 2406:da18:abc:1234::50/128 scope global dynamic

# Ping IPv6 ra internet
ping6 ipv6.google.com -c 4

# Curl qua IPv6
curl -6 https://ipv6.google.com

# Kiểm tra connectivity qua IPv6
curl -v --ipv6 https://api.example.com

# Từ local máy, kiểm tra DNS AAAA record
dig AAAA api.example.com

# Kiểm tra ALB dual-stack DNS
nslookup dualstack.my-alb-xxxxx.ap-southeast-1.elb.amazonaws.com
# Phải trả về cả A (IPv4) và AAAA (IPv6) records

# Test kết nối đến ALB qua IPv6
curl -6 https://dualstack.my-alb-xxxxx.ap-southeast-1.elb.amazonaws.com
```

---

## 15. Chi Phí IPv6 trong AWS

**IPv6 addresses không tính phí** — AWS không charge per IPv6 address (khác với IPv4 public từ 2024 tính $0.005/giờ).

| Resource | Chi phí IPv6 |
|---------|--------------|
| IPv6 address (GUA) trên EC2 | Miễn phí |
| IPv6 CIDR trên VPC/Subnet | Miễn phí |
| Egress-Only Internet Gateway | Miễn phí (không phí theo giờ) |
| Data transfer IPv6 | Tương tự IPv4 data transfer rates |
| ALB dualstack | Không thêm phí (phí theo LCU như thường) |

**Tiết kiệm khi dùng IPv6:**
```
Mỗi EC2 instance IPv4 public (từ 2024):
  $0.005/giờ × 24 × 30 = $3.60/tháng/instance

Nếu 100 instances:
  100 × $3.60 = $360/tháng chỉ cho IPv4 public IPs

Chuyển sang IPv6 dual-stack:
  IPv6 không tính phí → Tiết kiệm $360/tháng
  (Giữ IPv4 private, dùng IPv6 cho public traffic qua ALB)
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Sự khác biệt giữa Egress-Only Internet Gateway và NAT Gateway là gì?

**Trả lời:**
Cả hai đều cho phép instances trong private subnet ra internet nhưng chặn inbound connections từ internet:

**NAT Gateway:**
- Dành cho **IPv4**
- Thực hiện **Network Address Translation (NAT — Dịch Địa Chỉ Mạng)**: thay thế IP private nguồn bằng IP public của NAT GW
- **Có phí**: $0.059/giờ + $0.059/GB
- Cần tạo ở từng AZ để HA

**Egress-Only Internet Gateway:**
- Dành cho **IPv6**
- Không cần NAT (IPv6 GUA là routable, nhưng EIGW chỉ cho outbound)
- **Miễn phí**
- Là regional resource (không cần tạo per-AZ)
- Stateful như NAT GW: response traffic được tự động cho phép

---

### Q2: Tại sao IPv6 GUA luôn là public routable? Làm sao protect private instances?

**Trả lời:**
IPv6 GUA (2000::/3) được thiết kế để globally routable — không có khái niệm "private IPv6 range" như RFC 1918 của IPv4 (10.x, 172.16.x, 192.168.x). Đây là thiết kế có chủ đích: IPv6 đủ lớn để mỗi thiết bị có IP thật, không cần NAT.

Để protect private instances với IPv6:
1. **Egress-Only IGW:** Block inbound từ internet, cho phép outbound
2. **Security Groups:** Chỉ allow traffic cần thiết (SG stateful — response tự động được phép)
3. **NACLs:** Thêm lớp kiểm soát stateless
4. **ULA (Unique Local Address — fc00::/7):** Dùng cho internal-only communication (từ 2023 AWS hỗ trợ qua IPAM)

---

### Q3: Dual-Stack ALB hoạt động như thế nào khi targets chỉ có IPv4?

**Trả lời:**
ALB dual-stack là **dual-stack termination point** — ALB nhận cả IPv4 và IPv6 requests từ internet, nhưng kết nối về phía targets luôn là **IPv4**.

```
IPv6 Client → ALB (accept IPv6) → Targets (IPv4 private IPs)
IPv4 Client → ALB (accept IPv4) → Targets (IPv4 private IPs)
```

Targets (EC2, ECS) không cần có IPv6. ALB làm protocol translation. Đây là lý do tại sao dual-stack ALB là bước đơn giản nhất để bắt đầu hỗ trợ IPv6 mà không cần thay đổi backend.

---

### Q4: AWS cấp IPv6 CIDR cho VPC như thế nào? Có thể chọn CIDR không?

**Trả lời:**
AWS cấp IPv6 CIDR theo 2 cách:

1. **Amazon-provided IPv6 CIDR (phổ biến):** AWS cấp /56 từ pool IPv6 của Amazon. Bạn **không thể chọn** CIDR cụ thể — AWS tự chọn. Mỗi VPC nhận /56 khác nhau.

2. **BYOIP (Bring Your Own IP — Mang IP Của Bạn):** Nếu tổ chức có IPv6 block riêng, có thể đăng ký với AWS và dùng prefix đó.

3. **IPAM-allocated IPv6 (từ 2021):** Dùng AWS IPAM (IP Address Manager — Quản Lý Địa Chỉ IP) để phân bổ IPv6 từ pool nội bộ tổ chức — hỗ trợ ULA.

Sau khi VPC có /56, bạn chọn /64 cho từng subnet (256 subnet /64 có thể trong /56).

---

### Q5: Security Groups có cần thêm IPv6 rules riêng không?

**Trả lời:**
**Có**, bắt buộc. Security Group rules cho IPv4 (`0.0.0.0/0`) và IPv6 (`::/0`) là **hoàn toàn tách biệt**. Một rule chỉ có `0.0.0.0/0` sẽ không apply cho IPv6 traffic.

Ví dụ: Nếu chỉ có rule `inbound TCP 443 from 0.0.0.0/0`:
- IPv4 client → HTTPS → Được phép ✓
- IPv6 client → HTTPS → Bị block ✗

Phải thêm rule riêng: `inbound TCP 443 from ::/0` để IPv6 client vào được.

**Lưu ý NACLs:** Tương tự, NACL cũng cần rules riêng cho IPv6, với đặc điểm là stateless (phải có cả inbound và outbound rules cho mỗi chiều traffic).

---

### Q6: Làm thế nào để EC2 instance trong IPv6-only subnet giao tiếp với service chỉ có IPv4?

**Trả lời:**
Dùng **DNS64 + NAT64**:

1. **DNS64** (bật trên subnet): Khi app query DNS cho domain chỉ có A record (IPv4), Route 53 Resolver synthesize (tổng hợp) một AAAA record trong prefix `64:ff9b::/96`. Ví dụ: IPv4 `1.2.3.4` → AAAA `64:ff9b::1.2.3.4`.

2. **NAT64** (tích hợp trong NAT Gateway): Traffic đến `64:ff9b::/96` được NAT Gateway dịch từ IPv6 sang IPv4 trước khi gửi ra internet.

Setup:
```bash
# Bật DNS64 cho subnet
aws ec2 modify-subnet-attribute \
  --subnet-id subnet-xxx \
  --enable-dns64

# Route traffic NAT64 qua NAT Gateway
aws ec2 create-route \
  --route-table-id rtb-xxx \
  --destination-ipv6-cidr-block 64:ff9b::/96 \
  --nat-gateway-id nat-xxx
```

Điều này cho phép IPv6-only instances giao tiếp với bất kỳ service IPv4 nào mà không cần thay đổi ứng dụng.

---

## Điều Hướng

- [← 4-elastic-ip-eni.md](./4-elastic-ip-eni.md) — Elastic IP & ENI
- [1-vpc-endpoints.md](./1-vpc-endpoints.md) — VPC Endpoints
- [2-privatelink.md](./2-privatelink.md) — AWS PrivateLink
- [3-global-accelerator.md](./3-global-accelerator.md) — Global Accelerator
- [README.md](./README.md) — Tổng quan 07-advanced-networking
- [← 06-connectivity](../06-connectivity/) — VPN, Direct Connect, Transit Gateway
- [→ INDEX.md](../INDEX.md) — Chỉ mục toàn bộ AWS Networking
