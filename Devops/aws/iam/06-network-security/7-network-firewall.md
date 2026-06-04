# AWS Network Firewall — Tường Lửa Mạng Có Thể Lập Trình

> AWS Network Firewall (Tường Lửa Mạng AWS) là dịch vụ tường lửa stateful (có trạng thái) được quản lý hoàn toàn, hỗ trợ DPI (Deep Packet Inspection — Kiểm Tra Gói Tin Sâu), IPS (Intrusion Prevention System — Hệ Thống Ngăn Chặn Xâm Nhập), và lọc theo domain name, triển khai bên trong VPC.

## 📚 Mục Lục

1. [Network Firewall là gì?](#network-firewall-là-gì)
2. [Kiến Trúc Deployment](#kiến-trúc-deployment)
3. [Thành Phần Chính](#thành-phần-chính)
4. [Rule Types — Loại Rule](#rule-types--loại-rule)
5. [Stateful vs Stateless Rules](#stateful-vs-stateless-rules)
6. [Thiết Lập Network Firewall](#thiết-lập-network-firewall)
7. [Use Cases Thực Tế](#use-cases-thực-tế)
8. [So Sánh với Các Cơ Chế Khác](#so-sánh-với-các-cơ-chế-khác)
9. [Best Practices](#best-practices)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Network Firewall là gì?

AWS Network Firewall là managed firewall service (dịch vụ tường lửa được quản lý) triển khai trong VPC, cung cấp khả năng lọc traffic nâng cao hơn Security Groups và NACLs:

| Khả Năng | Security Group | NACL | Network Firewall |
|---|---|---|---|
| Lọc theo IP/Port | ✅ | ✅ | ✅ |
| Stateful tracking | ✅ | ❌ | ✅ |
| Domain name filtering | ❌ | ❌ | ✅ |
| TLS inspection | ❌ | ❌ | ✅ |
| IPS/IDS (Suricata rules) | ❌ | ❌ | ✅ |
| Protocol-aware rules | ❌ | ❌ | ✅ |
| Centralized logging | ❌ | ❌ | ✅ |

### Khi Nào Cần Network Firewall?

Dùng Network Firewall khi cần:
- **Egress filtering** — kiểm soát ứng dụng được phép gọi domain nào ra internet
- **East-West traffic inspection** — kiểm tra traffic giữa các subnet/VPC
- **IPS/IDS** — phát hiện và ngăn chặn intrusion dựa trên Suricata signatures
- **Centralized firewall** — một điểm kiểm soát cho toàn VPC thay vì Security Group phân tán

---

## Kiến Trúc Deployment

### Traffic Flow qua Network Firewall

Network Firewall yêu cầu thay đổi route table để buộc traffic đi qua firewall endpoint:

```
Internet
    │
    ▼
Internet Gateway
    │ (route table của IGW redirect về Firewall)
    ▼
Firewall Subnet (dedicated subnet)
    │  AWS Network Firewall Endpoint
    │  (ENI của Firewall trong AZ)
    ▼
Application Subnet
    │
    ▼
EC2 / ECS Application
```

### Route Table Configuration

```
Internet Gateway Route Table (ingress routing):
  Destination: 10.0.1.0/24 (App Subnet) → Firewall Endpoint ENI

Firewall Subnet Route Table:
  Destination: 0.0.0.0/0 → Internet Gateway

Application Subnet Route Table:
  Destination: 0.0.0.0/0 → Firewall Endpoint ENI
```

### Multi-AZ Deployment

```
VPC (10.0.0.0/16)
├── AZ-a
│   ├── Firewall Subnet (10.0.0.0/28)    ← Network Firewall Endpoint
│   └── App Subnet (10.0.1.0/24)          ← EC2 instances
│
└── AZ-b
    ├── Firewall Subnet (10.0.2.0/28)    ← Network Firewall Endpoint
    └── App Subnet (10.0.3.0/24)          ← EC2 instances
```

Mỗi AZ cần một Firewall Endpoint riêng — traffic không cross-AZ qua firewall.

---

## Thành Phần Chính

### Firewall Policy (Chính Sách Tường Lửa)

Container chứa các rule groups, quy định thứ tự đánh giá và hành động mặc định:

```
Firewall Policy
├── Stateless Rule Groups (đánh giá trước, nhanh)
│   ├── Priority 1: Allow-established-tcp (pass known connections)
│   └── Priority 2: Drop-invalid-packets
│
└── Stateful Rule Groups (đánh giá sau, có context)
    ├── Priority 1: Block-malware-domains
    ├── Priority 2: Allow-approved-domains-only
    └── Priority 3: IPS-suricata-rules
```

### Rule Groups (Nhóm Rule)

Có thể dùng chung (shared) một rule group cho nhiều firewall policy — hữu ích trong môi trường multi-account với AWS Firewall Manager.

---

## Rule Types — Loại Rule

### 1. Stateless Rules (Rule Không Trạng Thái)

Tương tự NACL — đánh giá từng packet độc lập theo header (IP, port, protocol):

```json
{
  "Priority": 100,
  "RuleDefinition": {
    "MatchAttributes": {
      "Sources": [{"AddressDefinition": "0.0.0.0/0"}],
      "Destinations": [{"AddressDefinition": "10.0.0.0/8"}],
      "Protocols": [6],
      "DestinationPorts": [{"FromPort": 443, "ToPort": 443}]
    },
    "Actions": ["aws:pass"]
  }
}
```

Hành động có thể là: `aws:pass`, `aws:drop`, `aws:forward_to_sfe` (chuyển đến Stateful Engine)

### 2. Stateful Rules — Domain List (Danh Sách Domain)

Lọc theo tên miền — tính năng đặc trưng của Network Firewall:

```json
{
  "RulesSourceList": {
    "Targets": [
      ".amazonaws.com",
      ".github.com",
      ".npmjs.com",
      ".docker.io"
    ],
    "TargetTypes": ["HTTP_HOST", "TLS_SNI"],
    "GeneratedRulesType": "ALLOWLIST"
  }
}
```

**ALLOWLIST:** Chỉ cho phép domain trong danh sách, block tất cả còn lại
**DENYLIST:** Block domain trong danh sách, cho phép tất cả còn lại

### 3. Stateful Rules — Suricata Compatible Rules (IPS/IDS)

Network Firewall hỗ trợ cú pháp Suricata — công cụ IPS/IDS mã nguồn mở phổ biến:

```suricata
# Block kết nối đến IP đã biết là C2 (Command and Control) server
drop ip any any -> 198.51.100.5 any (
  msg:"Known C2 Server connection blocked";
  sid:1000001;
  rev:1;
)

# Phát hiện SQL Injection trong HTTP requests
alert http any any -> any any (
  msg:"SQL Injection attempt detected";
  content:"UNION SELECT";
  http_uri;
  nocase;
  sid:1000002;
  rev:1;
)

# Block SMB traffic (ngăn chặn lây lan ransomware)
drop tcp any any -> any 445 (
  msg:"Block SMB - Ransomware prevention";
  sid:1000003;
  rev:1;
)

# Alert khi có data exfiltration qua DNS
alert dns any any -> any any (
  msg:"Suspicious long DNS query - possible DNS tunneling";
  dns.query;
  pcre:"/^.{100,}/";
  sid:1000004;
  rev:1;
)
```

---

## Stateful vs Stateless Rules

### Thứ Tự Đánh Giá

```
Packet đến
    │
    ▼
[Stateless Rules] — đánh giá nhanh theo header
    │
    ├── aws:drop → DROP (packet bị loại bỏ)
    ├── aws:pass → PASS (packet được phép, không qua stateful)
    └── aws:forward_to_sfe → tiếp tục đến Stateful Engine
            │
            ▼
    [Stateful Rules] — đánh giá theo context, flow, payload
            │
            ├── PASS → Cho phép
            ├── DROP → Từ chối
            └── ALERT → Cho phép + ghi log cảnh báo
```

**Quy tắc sử dụng:**
- **Stateless:** Chặn nhanh traffic rõ ràng không hợp lệ (ví dụ: IP known-bad)
- **Stateful:** Kiểm tra payload, domain, application protocol

---

## Thiết Lập Network Firewall

### Terraform: Network Firewall Hoàn Chỉnh

```terraform
# Domain Allowlist Rule Group
resource "aws_networkfirewall_rule_group" "domain_allowlist" {
  capacity = 100
  name     = "approved-domains-allowlist"
  type     = "STATEFUL"

  rule_group {
    rules_source {
      rules_source_list {
        generated_rules_type = "ALLOWLIST"
        target_types         = ["HTTP_HOST", "TLS_SNI"]
        targets = [
          ".amazonaws.com",
          ".github.com",
          ".npmjs.com",
          ".pypi.org",
          ".docker.io",
          ".ghcr.io",
        ]
      }
    }
  }
}

# Suricata IPS Rules
resource "aws_networkfirewall_rule_group" "ips_rules" {
  capacity = 1000
  name     = "custom-ips-rules"
  type     = "STATEFUL"

  rule_group {
    rules_source {
      rules_string = <<EOF
# Block known malicious IPs (threat intelligence feed)
drop ip [198.51.100.0/24,203.0.113.0/24] any -> $HOME_NET any (msg:"Block threat intel IPs"; sid:2000001; rev:1;)

# Detect port scanning
alert tcp any any -> $HOME_NET any (msg:"Port scan detected"; flags:S; threshold:type both, track by_src, count 20, seconds 10; sid:2000002; rev:1;)

# Block DNS tunneling
alert dns any any -> any any (msg:"Long DNS query - possible tunneling"; dns.query; pcre:"/^.{150,}/"; sid:2000003; rev:1;)
EOF
    }
  }
}

# Firewall Policy
resource "aws_networkfirewall_firewall_policy" "main" {
  name = "production-firewall-policy"

  firewall_policy {
    stateless_default_actions          = ["aws:forward_to_sfe"]
    stateless_fragment_default_actions = ["aws:forward_to_sfe"]

    stateful_rule_group_reference {
      resource_arn = aws_networkfirewall_rule_group.domain_allowlist.arn
      priority     = 100
    }

    stateful_rule_group_reference {
      resource_arn = aws_networkfirewall_rule_group.ips_rules.arn
      priority     = 200
    }

    stateful_default_actions = ["aws:drop_strict"]
  }
}

# Network Firewall
resource "aws_networkfirewall_firewall" "main" {
  name                = "production-network-firewall"
  firewall_policy_arn = aws_networkfirewall_firewall_policy.main.arn
  vpc_id              = aws_vpc.main.id

  dynamic "subnet_mapping" {
    for_each = aws_subnet.firewall
    content {
      subnet_id = subnet_mapping.value.id
    }
  }

  tags = { Environment = "production" }
}

# Logging Configuration
resource "aws_networkfirewall_logging_configuration" "main" {
  firewall_arn = aws_networkfirewall_firewall.main.arn

  logging_configuration {
    log_destination_config {
      log_destination = {
        logGroup = "/aws/network-firewall/flow-logs"
      }
      log_destination_type = "CloudWatchLogs"
      log_type             = "FLOW"
    }

    log_destination_config {
      log_destination = {
        logGroup = "/aws/network-firewall/alert-logs"
      }
      log_destination_type = "CloudWatchLogs"
      log_type             = "ALERT"
    }
  }
}
```

### Cập Nhật Route Tables

```terraform
# Route table của Application Subnet
# Traffic ra ngoài → qua Firewall Endpoint
resource "aws_route" "app_to_firewall" {
  route_table_id         = aws_route_table.app.id
  destination_cidr_block = "0.0.0.0/0"
  vpc_endpoint_id        = tolist(tolist(aws_networkfirewall_firewall.main.firewall_status[0].sync_states)[0].attachment)[0].endpoint_id
}

# Route table của Internet Gateway (Ingress Routing)
# Traffic vào App Subnet → qua Firewall Endpoint
resource "aws_route" "igw_to_firewall" {
  route_table_id         = aws_route_table.igw.id
  destination_cidr_block = aws_subnet.app.cidr_block
  vpc_endpoint_id        = tolist(tolist(aws_networkfirewall_firewall.main.firewall_status[0].sync_states)[0].attachment)[0].endpoint_id
}
```

---

## Use Cases Thực Tế

### 1. Centralized Egress Filtering

Thay vì dùng Security Group cho từng instance, kiểm soát tất cả traffic ra internet qua một điểm:

```
Private Subnets (nhiều team, nhiều ứng dụng)
    │
    ▼
Centralized NAT Gateway + Network Firewall
  Rule: Allow domain .githubusercontent.com (GitHub artifacts)
  Rule: Allow domain .amazonaws.com (AWS services)
  Rule: Block everything else
    │
    ▼
Internet
```

### 2. East-West Traffic Inspection (Kiểm Tra Traffic Nội Bộ)

Kiểm tra traffic giữa các VPC trong Transit Gateway:

```
VPC-A (App) ──┐
              ├──► Transit Gateway → Network Firewall VPC → Transit Gateway
VPC-B (DB)  ──┘
```

### 3. Compliance — Chặn Data Exfiltration

```suricata
# Block upload đến file sharing sites không được phép
drop http $HOME_NET any -> any any (
  msg:"Block unauthorized file upload";
  http.uri;
  content:"upload";
  http.host;
  content:"wetransfer.com";
  sid:3000001; rev:1;
)
```

---

## So Sánh với Các Cơ Chế Khác

| Tính Năng | Security Group | NACL | WAF | Network Firewall |
|---|---|---|---|---|
| Cấp độ | Instance | Subnet | Application | VPC |
| Tầng OSI | L3/L4 | L3/L4 | L7 (HTTP) | L3-L7 |
| Domain filtering | ❌ | ❌ | Không native | ✅ |
| IPS/IDS | ❌ | ❌ | ❌ | ✅ (Suricata) |
| TLS inspection | ❌ | ❌ | ❌ | ✅ |
| East-West | Per-SG | Subnet | ❌ | ✅ |
| Chi phí | Thấp | Thấp | Vừa | Cao |

---

## Best Practices

1. **Dùng Dedicated Firewall Subnet** — subnet riêng cho firewall endpoint, không chia sẻ với application
2. **Multi-AZ deployment** — một endpoint per AZ để đảm bảo HA (High Availability — Tính Sẵn Sàng Cao)
3. **Bắt đầu với ALERT trước BLOCK** — tương tự WAF, quan sát log trước khi block
4. **Domain Allowlist thay Denylist** — whitelist cụ thể an toàn hơn blacklist vô hạn
5. **Tích hợp Threat Intelligence** — cập nhật Suricata rules từ các feed như Emerging Threats
6. **Bật FLOW và ALERT logging** — gửi về CloudWatch Logs hoặc S3 để phân tích
7. **Dùng Firewall Manager** — quản lý tập trung cho nhiều tài khoản và VPC

---

## Câu Hỏi Phỏng Vấn

### Q: Khi nào nên dùng Network Firewall thay vì chỉ dùng Security Groups và WAF?

**Trả lời:** Network Firewall phù hợp khi:
1. Cần **domain-based filtering** (egress control theo tên miền) — Security Group không thể lọc theo domain
2. Cần **IPS/IDS** dựa trên Suricata signatures — phát hiện ransomware, C2, data exfiltration patterns
3. Cần **east-west inspection** — kiểm tra traffic giữa các VPC trong Transit Gateway
4. Tổ chức yêu cầu **centralized firewall** thay vì Security Group phân tán

Với ứng dụng web đơn giản, Security Groups + WAF thường đủ. Network Firewall thêm chi phí đáng kể ($0.395/hour + $0.065/GB).

### Q: Network Firewall stateful rule và stateless rule khác nhau thế nào?

**Trả lời:**
- **Stateless rules:** Đánh giá từng packet độc lập theo IP/port header; rất nhanh; không biết ngữ cảnh connection; tương tự NACL
- **Stateful rules:** Theo dõi connection state; có thể inspect payload và application protocol; hỗ trợ domain filtering và Suricata IPS; chậm hơn nhưng thông minh hơn

Best practice: dùng stateless rules để drop/pass traffic rõ ràng nhanh chóng, forward phần còn lại cho stateful engine xử lý.

### Q: Làm thế nào deploy Network Firewall trong kiến trúc high-availability?

**Trả lời:** Mỗi AZ cần một Firewall Endpoint riêng (Network Firewall là zonal resource). Cấu hình route table trong từng AZ để traffic chỉ đi qua endpoint cùng AZ. Nếu một AZ bị down, traffic của AZ đó bị ảnh hưởng nhưng các AZ khác vẫn hoạt động. AWS tự quản lý scaling của firewall endpoint, không cần lo về capacity.

---

## 🔗 Xem Thêm

- [README.md](README.md) — Tổng quan Network Security và Defense-in-Depth
- [1-security-groups.md](1-security-groups.md) — Security Groups cơ bản
- [5-waf-setup.md](5-waf-setup.md) — WAF cho Layer 7

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
