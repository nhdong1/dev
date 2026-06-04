# Route 53 Resolver — Bộ Phân Giải DNS Hybrid

> Route 53 Resolver (Bộ Phân Giải Route 53) cho phép DNS resolution (phân giải DNS) liền mạch giữa môi trường **AWS VPC** và **on-premises** (tại chỗ) — giải quyết bài toán hybrid DNS mà các doanh nghiệp lớn thường gặp phải.

## 📚 Mục Lục

1. [Vấn Đề Hybrid DNS](#vấn-đề-hybrid-dns)
2. [AmazonProvidedDNS](#amazonprovideddns)
3. [Route 53 Resolver Endpoints](#route-53-resolver-endpoints)
4. [Inbound Endpoints](#inbound-endpoints)
5. [Outbound Endpoints](#outbound-endpoints)
6. [Resolver Rules — Quy Tắc Phân Giải](#resolver-rules--quy-tắc-phân-giải)
7. [Kiến Trúc Hybrid DNS Phổ Biến](#kiến-trúc-hybrid-dns-phổ-biến)
8. [DNS Firewall](#dns-firewall)

---

## Vấn Đề Hybrid DNS

### Kịch Bản Thực Tế

Doanh nghiệp thường có:
- **On-premises:** Active Directory, internal services tại `corp.internal`
- **AWS VPC:** Microservices, databases tại `aws.internal`

```
Vấn đề 1: EC2 trong VPC muốn resolve db.corp.internal
→ AmazonProvidedDNS không biết corp.internal
→ Resolution fails!

Vấn đề 2: On-premises server muốn resolve redis.aws.internal
→ On-premises DNS server không biết aws.internal
→ Resolution fails!

Vấn đề 3: Developer muốn dùng cùng domain
  On-premises: hr.company.com → 192.168.1.100 (nội bộ)
  AWS: hr.company.com → 10.0.1.50 (ECS service)
```

### Giải Pháp: Route 53 Resolver Endpoints

```
AWS VPC ←→ Route 53 Resolver Endpoints ←→ On-premises DNS

Inbound Endpoint:  On-premises → VPC (AWS nhận query)
Outbound Endpoint: VPC → On-premises (AWS gửi query ra ngoài)
```

---

## AmazonProvidedDNS

### Mặc Định Trong Mọi VPC

Khi tạo VPC, AWS tự động cung cấp DNS resolver tại địa chỉ:
```
IP = VPC_CIDR_base + 2

Ví dụ:
  VPC CIDR: 10.0.0.0/16 → DNS tại 10.0.0.2
  VPC CIDR: 172.16.0.0/12 → DNS tại 172.16.0.2
  VPC CIDR: 192.168.0.0/16 → DNS tại 192.168.0.2

Cũng accessible tại: 169.254.169.253 (link-local address)
```

### Những Gì AmazonProvidedDNS Có Thể Làm

```
✅ Resolve public domains (google.com, s3.amazonaws.com)
✅ Resolve Public Hosted Zones của Route 53
✅ Resolve Private Hosted Zones liên kết với VPC
✅ Resolve AWS internal hostnames (ec2-12-34-56-78.compute-1.amazonaws.com)

❌ Không thể resolve on-premises domains (corp.internal)
❌ Không thể resolve custom DNS zones không liên kết với VPC
```

### VPC DNS Settings

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  # Bắt buộc bật để dùng Route 53 Private Hosted Zones
  enable_dns_support   = true  # Mặc định: true
  enable_dns_hostnames = true  # Mặc định: false, cần bật!

  tags = {
    Name = "main-vpc"
  }
}
```

---

## Route 53 Resolver Endpoints

### Khái Niệm

Resolver Endpoints là các **ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi)** được tạo trong VPC của bạn, hoạt động như bridge (cầu nối) giữa AWS DNS và on-premises DNS.

```
Kiến trúc:
        On-premises DNS Server (192.168.1.53)
                    ↕ (qua Direct Connect hoặc VPN)
        Route 53 Resolver Endpoints
        (ENIs trong subnet của VPC, ví dụ: 10.0.1.10, 10.0.1.11)
                    ↕
        AmazonProvidedDNS (10.0.0.2)
                    ↕
        Route 53 (Public + Private Hosted Zones)
```

### Yêu Cầu Cơ Sở Hạ Tầng

```
Kết nối mạng: Direct Connect HOẶC VPN Site-to-Site
  → Không có kết nối mạng → Resolver Endpoints không hoạt động được

Số lượng ENIs tối thiểu: 2 (một per Availability Zone để HA)
  → Khuyến nghị: 2-3 AZ

IP Addresses: Mỗi ENI cần 1 IP từ subnet
  → Endpoints có IP cố định (không thay đổi)
  → On-premises có thể cấu hình IP này vào DNS forwarder

Security Group: Phải cho phép DNS traffic (UDP/TCP port 53)
```

---

## Inbound Endpoints

### Chức Năng

Cho phép **on-premises DNS servers** gửi DNS queries đến Route 53 để resolve:
- Private Hosted Zone records (nội bộ AWS)
- AWS internal hostnames

```
Flow:
On-premises server query: redis.aws.internal
    ↓
On-premises DNS server không biết → Forward đến Inbound Endpoint
    ↓
Inbound Endpoint IP: 10.0.1.10 (qua Direct Connect/VPN)
    ↓
Route 53 Resolver xử lý
    ↓
Private Hosted Zone: aws.internal
  redis.aws.internal → 10.0.3.50
    ↓
Trả về 10.0.3.50 → On-premises server → Application
```

### Cấu Hình Terraform

```hcl
# Security Group cho Inbound Endpoint
resource "aws_security_group" "resolver_inbound" {
  name   = "resolver-inbound"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = ["192.168.0.0/16"]  # On-premises CIDR
  }

  ingress {
    from_port   = 53
    to_port     = 53
    protocol    = "tcp"
    cidr_blocks = ["192.168.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Inbound Resolver Endpoint
resource "aws_route53_resolver_endpoint" "inbound" {
  name      = "inbound-resolver"
  direction = "INBOUND"

  security_group_ids = [aws_security_group.resolver_inbound.id]

  # Tạo ENI trong 2 AZ để đảm bảo HA
  ip_address {
    subnet_id  = aws_subnet.private_az1.id
    ip         = "10.0.1.10"  # IP cố định
  }

  ip_address {
    subnet_id  = aws_subnet.private_az2.id
    ip         = "10.0.2.10"  # IP cố định
  }

  tags = {
    Name = "inbound-resolver"
  }
}
```

### Cấu Hình On-premises DNS

```
Trên Windows Server DNS / BIND / Unbound:

Tạo conditional forwarder cho zone aws.internal:
  Forward đến: 10.0.1.10, 10.0.2.10 (Inbound Endpoint IPs)

Ví dụ BIND:
zone "aws.internal" {
    type forward;
    forward only;
    forwarders { 10.0.1.10; 10.0.2.10; };
};
```

---

## Outbound Endpoints

### Chức Năng

Cho phép **EC2 instances và services trong VPC** gửi DNS queries ra **on-premises DNS server** để resolve corporate domains.

```
Flow:
EC2 instance query: db.corp.internal
    ↓
AmazonProvidedDNS (10.0.0.2) nhận query
    ↓
Kiểm tra Resolver Rules:
  corp.internal → Forward đến 192.168.1.53 (on-premises DNS)
    ↓
Outbound Endpoint ENI
    ↓ (qua Direct Connect/VPN)
On-premises DNS Server: 192.168.1.53
    ↓
Trả về: 192.168.1.100
    ↓ (qua Direct Connect/VPN)
Outbound Endpoint → AmazonProvidedDNS → EC2 instance
```

### Cấu Hình Terraform

```hcl
# Security Group cho Outbound Endpoint
resource "aws_security_group" "resolver_outbound" {
  name   = "resolver-outbound"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = ["192.168.0.0/16"]  # On-premises CIDR
  }

  egress {
    from_port   = 53
    to_port     = 53
    protocol    = "tcp"
    cidr_blocks = ["192.168.0.0/16"]
  }
}

# Outbound Resolver Endpoint
resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "outbound-resolver"
  direction = "OUTBOUND"

  security_group_ids = [aws_security_group.resolver_outbound.id]

  ip_address {
    subnet_id = aws_subnet.private_az1.id
  }

  ip_address {
    subnet_id = aws_subnet.private_az2.id
  }

  tags = {
    Name = "outbound-resolver"
  }
}
```

---

## Resolver Rules — Quy Tắc Phân Giải

### Khái Niệm

Resolver Rules (Quy Tắc Phân Giải) xác định **khi nào và đến đâu** để forward DNS queries từ Outbound Endpoint.

### Các Loại Rules

```
1. FORWARD Rule (Quy Tắc Chuyển Tiếp):
   Chỉ định domain → Forward đến IP cụ thể
   Ví dụ: corp.internal → 192.168.1.53

2. SYSTEM Rule (Quy Tắc Hệ Thống):
   Route 53 tự tạo, override FORWARD rules
   Dùng cho: amazonaws.com, aws-dns.com
   → Đảm bảo AWS internal domains không bị forward ra ngoài

3. RECURSIVE Rule (Quy Tắc Đệ Quy):
   Dùng Route 53 là recursive resolver cho mọi queries không khớp rule nào
   → Mặc định cho tất cả domains
```

### Cấu Hình Resolver Rules

```hcl
# Forward Rule: corp.internal → On-premises DNS
resource "aws_route53_resolver_rule" "corp_internal" {
  domain_name          = "corp.internal"
  name                 = "forward-corp-internal"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id

  target_ip {
    ip   = "192.168.1.53"  # Primary on-premises DNS
    port = 53
  }

  target_ip {
    ip   = "192.168.1.54"  # Secondary on-premises DNS
    port = 53
  }
}

# Associate rule với VPC
resource "aws_route53_resolver_rule_association" "corp_internal" {
  resolver_rule_id = aws_route53_resolver_rule.corp_internal.id
  vpc_id           = aws_vpc.main.id
}

# Share rule với accounts khác qua RAM (Resource Access Manager)
resource "aws_ram_resource_share" "resolver_rules" {
  name = "resolver-rules-share"
}

resource "aws_ram_resource_association" "resolver_rule" {
  resource_arn       = aws_route53_resolver_rule.corp_internal.arn
  resource_share_arn = aws_ram_resource_share.resolver_rules.arn
}
```

### Rule Priority

```
Thứ tự ưu tiên khi matching (từ cao xuống thấp):
1. SYSTEM rules (AWS tự tạo, không thể override)
2. FORWARD rules có domain cụ thể nhất (longest match)
   Ví dụ: hr.corp.internal khớp trước corp.internal
3. RECURSIVE rule (mặc định cho tất cả)
```

---

## Kiến Trúc Hybrid DNS Phổ Biến

### Kiến Trúc 1: Cơ Bản — Một VPC + On-premises

```
On-premises Network (192.168.0.0/16)
├── DNS Server: 192.168.1.53
├── Corp domain: corp.internal
└── Direct Connect / VPN tunnel

                    ↕ Network connectivity

AWS VPC (10.0.0.0/16)
├── Inbound Endpoint: 10.0.1.10, 10.0.2.10
│   → Cho phép on-premises resolve aws.internal
├── Outbound Endpoint
│   → EC2 có thể resolve corp.internal
├── Resolver Rule: corp.internal → 192.168.1.53
├── Private Hosted Zone: aws.internal
│   ├── db.aws.internal → 10.0.3.100 (RDS)
│   └── cache.aws.internal → 10.0.4.50 (ElastiCache)
└── Private Hosted Zone: company.com (split-horizon)
    └── app.company.com → 10.0.1.80 (Internal ALB)
```

### Kiến Trúc 2: Hub-and-Spoke — Multiple VPCs

```
Vấn đề: Nhiều VPC, mỗi VPC đều cần forward DNS ra on-premises
→ Không muốn tạo Outbound Endpoint trong mỗi VPC (tốn tiền, phức tạp)

Giải pháp: Centralized DNS VPC (VPC trung tâm)

Spoke VPC 1  \
Spoke VPC 2   ── Transit Gateway ── Hub DNS VPC
Spoke VPC 3  /                         ├── Outbound Endpoint
                                        └── Resolver Rules (shared via RAM)

Cấu hình:
1. Tạo Hub DNS VPC với Outbound Endpoint
2. Tạo Resolver Rules trong Hub VPC
3. Share rules via AWS RAM (Resource Access Manager) đến Spoke VPCs
4. Associate rules với tất cả Spoke VPCs
5. Route DNS traffic từ Spoke VPCs qua Transit Gateway đến Hub DNS VPC
```

### Kiến Trúc 3: Multi-Account với AWS Organizations

```
Management Account
└── Route 53 Resolver Rules (Central)
    └── Shared via RAM to all accounts

Network Account (Hub)
└── Outbound Endpoint → On-premises DNS

Workload Account A
└── Associate shared rules → DNS resolution works!

Workload Account B
└── Associate shared rules → DNS resolution works!
```

---

## DNS Firewall

### Route 53 Resolver DNS Firewall

DNS Firewall (Tường Lửa DNS) lọc **outbound DNS queries** từ VPC — ngăn DNS-based attacks và data exfiltration (đánh cắp dữ liệu qua DNS).

### Vấn Đề DNS Firewall Giải Quyết

```
DNS Tunneling (Đường Hầm DNS):
  Attacker encode data trong DNS queries
  example: aGVsbG8gd29ybGQ.evil-c2.com
  → Bypass firewalls vì port 53 thường được cho phép

DNS Exfiltration (Rò Rỉ Dữ Liệu Qua DNS):
  Malware gửi sensitive data trong DNS queries
  → Khó phát hiện hơn HTTP/HTTPS

Phishing via DNS:
  Malware resolve malicious domains để download payload
```

### Cấu Hình DNS Firewall

```hcl
# Domain list: Block known malicious domains
resource "aws_route53_resolver_firewall_domain_list" "blocklist" {
  name    = "malicious-domains"
  domains = ["malware.example.com", "phishing.badsite.com"]

  tags = {
    Name = "blocklist"
  }
}

# Rule group
resource "aws_route53_resolver_firewall_rule_group" "main" {
  name = "main-firewall-rules"
}

# Block rule
resource "aws_route53_resolver_firewall_rule" "block" {
  name                    = "block-malicious"
  action                  = "BLOCK"
  block_response          = "NXDOMAIN"  # Trả về "domain not found"
  firewall_domain_list_id = aws_route53_resolver_firewall_domain_list.blocklist.id
  firewall_rule_group_id  = aws_route53_resolver_firewall_rule_group.main.id
  priority                = 100
}

# Associate với VPC
resource "aws_route53_resolver_firewall_rule_group_association" "main" {
  name                   = "main-firewall"
  firewall_rule_group_id = aws_route53_resolver_firewall_rule_group.main.id
  vpc_id                 = aws_vpc.main.id
  priority               = 100
}
```

### Managed Domain Lists

AWS cung cấp **Managed Domain Lists** (Danh Sách Domain Được Quản Lý) tự động cập nhật:
- `AWSManagedDomainsBotnetCommandandControl` — C2 servers của botnet
- `AWSManagedDomainsMalwareDomainList` — Malware distribution
- `AWSManagedDomainsAggregatedList` — Tổng hợp tất cả

```hcl
data "aws_route53_resolver_firewall_domain_list" "managed_botnet" {
  name = "AWSManagedDomainsBotnetCommandandControl"
}
```

---

## Debugging DNS Issues trong Hybrid Setup

### Công Cụ

```bash
# Từ EC2 trong VPC: kiểm tra resolution
dig @10.0.0.2 db.corp.internal  # Hỏi AmazonProvidedDNS trực tiếp
dig db.corp.internal              # Dùng system DNS

# Kiểm tra Resolver Rules có hiệu lực không
aws route53resolver list-resolver-rule-associations \
  --filters Name=VPCId,Values=vpc-123456

# Xem logs nếu bật Query Logging
aws route53resolver list-resolver-query-log-configs
```

### Query Logging (Ghi Nhật Ký Query)

```hcl
resource "aws_route53_resolver_query_log_config" "main" {
  name            = "vpc-dns-logs"
  destination_arn = aws_cloudwatch_log_group.dns.arn
}

resource "aws_route53_resolver_query_log_config_association" "main" {
  resolver_query_log_config_id = aws_route53_resolver_query_log_config.main.id
  resource_id                  = aws_vpc.main.id
}
```

Log entry mẫu:
```json
{
  "version": "1.100000",
  "account_id": "123456789012",
  "region": "us-east-1",
  "vpc_id": "vpc-123456",
  "query_timestamp": "2026-05-14T10:00:00Z",
  "query_name": "db.corp.internal.",
  "query_type": "A",
  "query_class": "IN",
  "rcode": "NOERROR",
  "answers": [{"Rdata": "192.168.1.100", "Type": "A", "Class": "IN"}]
}
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Giải thích Inbound và Outbound Resolver Endpoints

**Trả lời:**
- **Inbound Endpoint:** Cho phép on-premises DNS server forward queries VÀO AWS để resolve các domain nội bộ AWS (Private Hosted Zones). On-premises server cấu hình conditional forwarder trỏ đến IP của Inbound Endpoint.
- **Outbound Endpoint:** Cho phép EC2 trong VPC forward queries RA NGOÀI on-premises DNS server để resolve corporate domains. Cần kết hợp với Resolver Rules chỉ định domain nào cần forward.

### Câu 2: Khi nào cần Route 53 Resolver Endpoints?

**Trả lời:** Khi có môi trường hybrid — AWS VPC kết nối với on-premises qua Direct Connect hoặc VPN, và cần:
1. EC2 resolve được tên miền on-premises (corp.internal) → Outbound Endpoint
2. On-premises servers resolve được tên miền AWS nội bộ (aws.internal) → Inbound Endpoint

### Câu 3: Làm sao share Resolver Rules giữa nhiều VPCs/accounts?

**Trả lời:** Dùng **AWS RAM (Resource Access Manager)**. Tạo Resolver Rules trong một central account, share qua RAM với các accounts khác, sau đó associate rules với VPCs trong mỗi account. Với Organizations, có thể share toàn bộ OU.

### Câu 4: DNS Firewall trong Route 53 bảo vệ điều gì?

**Trả lời:** Lọc outbound DNS queries từ VPC — ngăn chặn resolution của malicious domains (malware, botnet C2 servers), chống DNS tunneling và DNS-based data exfiltration. AWS cung cấp managed domain lists tự động cập nhật các domains độc hại đã biết.

---

## 🔗 Điều Hướng

- ← [4-health-checks.md](./4-health-checks.md) — Health Checks
- → [05-cdn-cloudfront/README.md](../05-cdn-cloudfront/README.md) — CloudFront CDN
- [README.md](./README.md) — Tổng quan section
- [INDEX.md](../INDEX.md) — Chỉ mục đầy đủ

---

**Cập Nhật Lần Cuối:** 2026-05-14
