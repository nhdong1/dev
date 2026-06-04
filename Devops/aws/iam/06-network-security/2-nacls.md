# NACLs — Network Access Control Lists (Danh Sách Kiểm Soát Truy Cập Mạng)

> NACL (Network Access Control List — Danh Sách Kiểm Soát Truy Cập Mạng) là tường lửa stateless (không có trạng thái) hoạt động ở cấp subnet trong VPC, là lớp bảo vệ bổ sung bên ngoài Security Groups.

## 📚 Mục Lục

1. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
2. [Cấu Trúc Rule](#cấu-trúc-rule)
3. [Stateless — Điểm Khác Biệt Quan Trọng](#stateless--điểm-khác-biệt-quan-trọng)
4. [Default NACL vs Custom NACL](#default-nacl-vs-custom-nacl)
5. [Thiết Kế NACL Thực Tế](#thiết-kế-nacl-thực-tế)
6. [Best Practices](#best-practices)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm Cốt Lõi

### NACL là gì?

NACL được gắn với **subnet** (không phải instance), áp dụng cho tất cả traffic đi vào và đi ra khỏi subnet đó. Mỗi subnet chỉ có thể liên kết với một NACL tại một thời điểm; một NACL có thể liên kết với nhiều subnet.

**Đặc điểm chính:**
- **Stateless** — mỗi packet được đánh giá độc lập, không theo dõi connection state
- **Allow và Deny** — có thể tạo rule cả Allow lẫn Deny tường minh
- **Numbered rules** — rule được đánh số, đánh giá từ số nhỏ đến lớn, rule đầu tiên khớp thì dừng
- **Implicit deny** — nếu không khớp rule nào, traffic bị từ chối (rule \* ở cuối)
- **Subnet-level** — áp dụng cho tất cả ENI trong subnet

### Vị Trí trong Kiến Trúc

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
NACL (kiểm tra inbound vào subnet)
    │
    ▼
Subnet
    │
    ▼
Security Group (kiểm tra trước khi vào instance)
    │
    ▼
EC2 / RDS / Lambda
    │
    ▼ (return traffic)
Security Group (auto-allow vì stateful)
    │
    ▼
NACL (kiểm tra outbound ra khỏi subnet — phải có rule)
    │
    ▼
Internet
```

---

## Cấu Trúc Rule

### Các Trường của Rule

| Trường | Mô Tả | Ví Dụ |
|---|---|---|
| **Rule number** | Số thứ tự ưu tiên (1–32766) | 100, 200, 300 |
| **Type** | Loại traffic | HTTP (80), HTTPS (443), Custom TCP |
| **Protocol** | Giao thức | TCP, UDP, ICMP, All |
| **Port range** | Dải cổng | 443, 1024-65535 |
| **Source/Destination** | CIDR block | 203.0.113.0/24, 0.0.0.0/0 |
| **Allow/Deny** | Hành động | ALLOW, DENY |

### Ví Dụ NACL cho Public Subnet

**Inbound Rules (Rule Chiều Vào):**

| Rule # | Type | Protocol | Port | Source | Action |
|---|---|---|---|---|---|
| 100 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 110 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 120 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| \* | All traffic | All | All | 0.0.0.0/0 | DENY |

**Outbound Rules (Rule Chiều Ra):**

| Rule # | Type | Protocol | Port | Destination | Action |
|---|---|---|---|---|---|
| 100 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 110 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| \* | All traffic | All | All | 0.0.0.0/0 | DENY |

---

## Stateless — Điểm Khác Biệt Quan Trọng

### Vấn Đề Ephemeral Ports (Cổng Tạm Thời)

Khi client kết nối đến server:
- Client chọn một ephemeral port ngẫu nhiên (1024–65535) làm source port
- Server response về ephemeral port đó

Vì NACL là stateless, response packet phải được cho phép tường minh:

```
Kịch bản: Browser (203.0.113.5) truy cập EC2 (10.0.1.10) qua HTTPS

Request:  src=203.0.113.5:52341,  dst=10.0.1.10:443
Response: src=10.0.1.10:443,      dst=203.0.113.5:52341

NACL Inbound cần:  Allow TCP 443 from 0.0.0.0/0        ✅
NACL Outbound cần: Allow TCP 1024-65535 to 0.0.0.0/0   ✅ (ephemeral port!)

Nếu thiếu outbound rule cho ephemeral ports:
→ Response bị block tại NACL
→ Timeout ở phía client
→ Security Group vẫn OK (stateful) nhưng NACL chặn!
```

### Ephemeral Port Ranges Theo Hệ Điều Hành

| Hệ Điều Hành | Dải Ephemeral Port |
|---|---|
| Linux (kernel 4.x+) | 32768–60999 |
| Windows Server | 49152–65535 |
| AWS NAT Gateway | 1024–65535 |
| ELB / ALB | 1024–65535 |

**Khuyến nghị:** Dùng 1024–65535 cho outbound rule để bao hết tất cả OS.

---

## Default NACL vs Custom NACL

### Default NACL (NACL Mặc Định)

```
Mỗi VPC mới được tạo với một Default NACL:

Inbound:
  Rule 100: Allow All traffic (0.0.0.0/0)
  Rule *:   Deny All

Outbound:
  Rule 100: Allow All traffic (0.0.0.0/0)
  Rule *:   Deny All

→ Mặc định cho phép tất cả traffic (kém an toàn)
→ Nên tạo Custom NACL và liên kết với từng subnet
```

### Custom NACL (NACL Tùy Chỉnh)

```
Custom NACL mới tạo:

Inbound:
  Rule *: Deny All    ← chỉ có rule deny, không có allow

Outbound:
  Rule *: Deny All    ← chỉ có rule deny, không có allow

→ Block tất cả cho đến khi bạn thêm rule Allow
```

---

## Thiết Kế NACL Thực Tế

### NACL cho Public Subnet (Chứa ALB/Bastion)

```bash
# Tạo NACL
aws ec2 create-network-acl --vpc-id vpc-12345

# Inbound Rules
aws ec2 create-network-acl-entry \
  --network-acl-id acl-public \
  --rule-number 100 \
  --protocol tcp \
  --rule-action allow \
  --ingress \
  --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0

aws ec2 create-network-acl-entry \
  --network-acl-id acl-public \
  --rule-number 110 \
  --protocol tcp \
  --rule-action allow \
  --ingress \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0

# Cho phép return traffic (ephemeral ports)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-public \
  --rule-number 120 \
  --protocol tcp \
  --rule-action allow \
  --ingress \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0

# Outbound Rules
aws ec2 create-network-acl-entry \
  --network-acl-id acl-public \
  --rule-number 100 \
  --protocol tcp \
  --rule-action allow \
  --egress \
  --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0

aws ec2 create-network-acl-entry \
  --network-acl-id acl-public \
  --rule-number 110 \
  --protocol tcp \
  --rule-action allow \
  --egress \
  --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0
```

### NACL cho Private App Subnet

```
Inbound Rules:
  Rule 100: Allow TCP 8080 from 10.0.1.0/24 (public subnet range)  ← Từ ALB
  Rule 110: Allow TCP 1024-65535 from 10.0.3.0/24 (DB subnet)     ← Return từ DB
  Rule *:   Deny All

Outbound Rules:
  Rule 100: Allow TCP 5432 to 10.0.3.0/24 (DB subnet range)
  Rule 110: Allow TCP 443 to 0.0.0.0/0                             ← Đến S3, AWS services
  Rule 120: Allow TCP 1024-65535 to 10.0.1.0/24                    ← Return về ALB
  Rule *:   Deny All
```

### NACL cho Private Data Subnet (Database)

```
Inbound Rules:
  Rule 100: Allow TCP 5432 from 10.0.2.0/24 (app subnet range)
  Rule *:   Deny All

Outbound Rules:
  Rule 100: Allow TCP 1024-65535 to 10.0.2.0/24                    ← Return về App
  Rule *:   Deny All
```

### Terraform Example

```terraform
resource "aws_network_acl" "private_app" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = [aws_subnet.private_app.id]

  # Nhận từ ALB (public subnet)
  ingress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_subnet.public.cidr_block
    from_port  = 8080
    to_port    = 8080
  }

  # Return traffic từ DB
  ingress {
    rule_no    = 110
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_subnet.private_db.cidr_block
    from_port  = 1024
    to_port    = 65535
  }

  # Đến DB
  egress {
    rule_no    = 100
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_subnet.private_db.cidr_block
    from_port  = 5432
    to_port    = 5432
  }

  # Return về ALB
  egress {
    rule_no    = 110
    protocol   = "tcp"
    action     = "allow"
    cidr_block = aws_subnet.public.cidr_block
    from_port  = 1024
    to_port    = 65535
  }

  # HTTPS ra ngoài
  egress {
    rule_no    = 120
    protocol   = "tcp"
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  tags = { Name = "nacl-private-app" }
}
```

---

## Use Case: Chặn IP Tấn Công Khẩn Cấp

NACL rất hữu ích khi cần **block nhanh một dải IP** mà không cần thay đổi Security Group từng instance:

```bash
# Phát hiện IP 198.51.100.0/24 đang tấn công

# Thêm Deny rule với số nhỏ hơn các Allow rule hiện tại
aws ec2 create-network-acl-entry \
  --network-acl-id acl-public \
  --rule-number 50 \           # Nhỏ hơn 100 → đánh giá trước
  --protocol -1 \              # Tất cả protocol
  --rule-action deny \
  --ingress \
  --cidr-block 198.51.100.0/24

# → Tất cả traffic từ dải IP này bị block ngay lập tức
# → Không cần sửa Security Group từng instance
```

---

## Best Practices

1. **Không dùng Default NACL** — Tạo Custom NACL cho từng loại subnet
2. **Đánh số rule thưa thớt** — Dùng bội số 100 (100, 200, 300) để chèn rule sau này
3. **Luôn nhớ ephemeral ports** — Rule 1024-65535 cho return traffic
4. **Kết hợp với Security Groups** — NACL là lớp subnet, SG là lớp instance
5. **Dùng NACL để block dải IP khẩn cấp** — Nhanh và áp dụng toàn subnet
6. **Ghi chép mục đích mỗi rule** — Dùng comment (AWS hỗ trợ từ 2023)

---

## Câu Hỏi Phỏng Vấn

### Q: Tại sao cần NACL nếu đã có Security Group?

**Trả lời:** Defense-in-depth (phòng thủ theo chiều sâu). NACL bổ sung 2 khả năng Security Group không có:
1. **Rule Deny tường minh** — có thể block một IP cụ thể mà không cần xóa allow rule
2. **Bảo vệ cấp subnet** — một rule NACL áp dụng cho toàn subnet, không cần cập nhật từng instance

Tình huống điển hình: block ngay một dải IP đang tấn công trong khi điều tra.

### Q: Điều gì xảy ra nếu quên ephemeral ports trong NACL?

**Trả lời:** Kết nối TCP sẽ timeout. Request đi vào OK (nếu có rule Allow inbound), nhưng response packet bị drop tại NACL vì không có outbound rule cho port ngẫu nhiên (1024–65535) mà client chọn. Kết quả: browser thấy timeout, không phải connection refused. Đây là lỗi phổ biến nhất khi cấu hình NACL.

### Q: NACL rule được đánh giá theo thứ tự nào?

**Trả lời:** Từ rule number nhỏ nhất đến lớn nhất. Rule đầu tiên khớp với packet sẽ được áp dụng và dừng đánh giá (first-match, stop processing). Rule `*` ở cuối là implicit deny, áp dụng nếu không rule nào khớp. Đây là lý do đặt Deny rule với số nhỏ hơn Allow rule khi muốn block một IP cụ thể.

---

## 🔗 Xem Thêm

- [1-security-groups.md](1-security-groups.md) — Stateful firewall cấp instance
- [7-network-firewall.md](7-network-firewall.md) — Deep packet inspection nâng cao
- [README.md](README.md) — Tổng quan Network Security

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
