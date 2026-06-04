# Security Groups — Stateful Firewall (Tường Lửa Có Trạng Thái) ở Cấp Instance

> Security Group (Nhóm Bảo Mật) là tường lửa ảo stateful kiểm soát lưu lượng inbound (vào) và outbound (ra) cho các tài nguyên AWS như EC2 instance, RDS, Lambda trong VPC.

## 📚 Mục Lục

1. [Khái Niệm Cốt Lõi](#khái-niệm-cốt-lõi)
2. [Cấu Trúc Rule](#cấu-trúc-rule)
3. [Stateful vs Stateless](#stateful-vs-stateless)
4. [Security Group Chaining](#security-group-chaining)
5. [Best Practices](#best-practices)
6. [Ví Dụ Thực Tế](#ví-dụ-thực-tế)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Khái Niệm Cốt Lõi

### Security Group là gì?

Security Group hoạt động như một **virtual firewall** (tường lửa ảo) gắn trực tiếp với ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi) của tài nguyên, không phải gắn với subnet.

**Đặc điểm chính:**
- **Stateful** — tự động theo dõi kết nối, return traffic được cho phép tự động
- **Allow-only** — chỉ có rule Allow, không có rule Deny; traffic không khớp rule nào sẽ bị từ chối
- **Multiple SGs per resource** — một ENI có thể gắn tối đa 5 Security Groups
- **Scope** — phạm vi hoạt động trong một VPC; không cross-VPC mặc định

### So Sánh với NACL

| Đặc Điểm | Security Group | NACL |
|---|---|---|
| Cấp độ áp dụng | Instance (ENI) | Subnet |
| Stateful? | ✅ Có | ❌ Không |
| Rule Allow/Deny | Chỉ Allow | Allow và Deny |
| Đánh giá rule | Tất cả cùng lúc | Theo số thứ tự |
| Return traffic | Tự động cho phép | Phải tạo rule ngược |

---

## Cấu Trúc Rule

### Inbound Rule (Rule Chiều Vào)

```json
{
  "IpProtocol": "tcp",
  "FromPort": 443,
  "ToPort": 443,
  "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "HTTPS from internet"}]
}
```

Các trường cần cấu hình:
- **Protocol** — `tcp`, `udp`, `icmp`, `-1` (tất cả protocol)
- **Port Range** — FromPort đến ToPort; `-1` cho tất cả port
- **Source** — CIDR block, Security Group ID, Prefix List ID

### Outbound Rule (Rule Chiều Ra)

Mặc định, Security Group mới có rule outbound `Allow All` (`0.0.0.0/0`). Có thể giới hạn để kiểm soát egress (lưu lượng ra).

```json
{
  "IpProtocol": "tcp",
  "FromPort": 5432,
  "ToPort": 5432,
  "UserIdGroupPairs": [{"GroupId": "sg-database-id", "Description": "To RDS PostgreSQL"}]
}
```

---

## Stateful vs Stateless

### Cách Stateful Hoạt Động

```
Client (IP: 203.0.113.5)  ──→  EC2 Instance
Request: src=203.0.113.5:54321, dst=10.0.1.10:443

Security Group đánh giá:
  Inbound rule: Allow TCP 443 from 0.0.0.0/0 ✅ MATCH → CHO PHÉP

Connection tracking ghi nhận:
  State: 203.0.113.5:54321 ↔ 10.0.1.10:443 = ESTABLISHED

Response: src=10.0.1.10:443, dst=203.0.113.5:54321

Security Group kiểm tra connection tracking:
  → Tìm thấy established connection
  → TỰ ĐỘNG CHO PHÉP (không cần outbound rule cho port 54321)
```

**Lợi ích:** Không cần viết rule cho ephemeral ports (cổng tạm thời 1024–65535) trong response.

---

## Security Group Chaining

### Khái Niệm

Security Group Chaining (Nối Chuỗi Nhóm Bảo Mật) là kỹ thuật dùng Security Group ID làm source/destination thay vì CIDR IP, giúp tạo luồng traffic có kiểm soát giữa các tầng ứng dụng.

### Ví Dụ Kiến Trúc 3-Tier

```
sg-alb (ALB Security Group)
  Inbound:  Allow TCP 443 from 0.0.0.0/0
  Outbound: Allow TCP 8080 to sg-app

sg-app (App Server Security Group)
  Inbound:  Allow TCP 8080 from sg-alb     ← Chỉ ALB mới vào được
  Outbound: Allow TCP 5432 to sg-db

sg-db (Database Security Group)
  Inbound:  Allow TCP 5432 from sg-app     ← Chỉ App Server mới vào được
  Outbound: (không cần, stateful)
```

**Lợi ích so với CIDR:**
- Không cần biết IP cụ thể của instance
- Tự động áp dụng khi scale in/out với Auto Scaling
- Dễ quản lý và audit hơn

### Ví Dụ với AWS CLI

```bash
# Tạo Security Groups
aws ec2 create-security-group \
  --group-name sg-alb \
  --description "ALB Security Group" \
  --vpc-id vpc-12345

aws ec2 create-security-group \
  --group-name sg-app \
  --description "App Server Security Group" \
  --vpc-id vpc-12345

# Cho phép App nhận traffic từ ALB (dùng SG ID, không dùng IP)
aws ec2 authorize-security-group-ingress \
  --group-id sg-0app123 \
  --protocol tcp \
  --port 8080 \
  --source-group sg-0alb456
```

---

## Best Practices

### 1. Nguyên Tắc Least Privilege (Đặc Quyền Tối Thiểu)

```bash
# ❌ Xấu — mở quá rộng
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol -1 \        # Tất cả protocol
  --port -1 \            # Tất cả port
  --cidr 0.0.0.0/0       # Từ mọi nơi

# ✅ Tốt — chỉ mở cần thiết
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

### 2. Không Dùng Default Security Group

Default Security Group có rule cho phép tất cả traffic giữa các resource cùng SG. Tạo SG riêng cho từng tầng ứng dụng.

```bash
# Xóa tất cả rule của default SG (không thể xóa bản thân SG)
aws ec2 revoke-security-group-ingress \
  --group-id sg-default-id \
  --protocol -1 --port -1 --source-group sg-default-id

aws ec2 revoke-security-group-egress \
  --group-id sg-default-id \
  --protocol -1 --port -1 --cidr 0.0.0.0/0
```

### 3. Không Mở SSH/RDP Trực Tiếp

```bash
# ❌ Tuyệt đối tránh
# Allow SSH from 0.0.0.0/0

# ✅ Dùng AWS Systems Manager Session Manager
# Không cần mở port 22 — kết nối qua SSM Agent
aws ssm start-session --target i-1234567890abcdef0

# ✅ Hoặc dùng Bastion Host với MFA
# Bastion SG: Allow SSH from corp-ip-range only
# App SG: Allow SSH from sg-bastion only
```

### 4. Đặt Description Có Ý Nghĩa

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --ip-permissions '[{
    "IpProtocol": "tcp",
    "FromPort": 8080,
    "ToPort": 8080,
    "UserIdGroupPairs": [{
      "GroupId": "sg-alb-id",
      "Description": "HTTP from ALB to App — added 2026-05-16"
    }]
  }]'
```

### 5. Giới Hạn Egress (Lưu Lượng Ra)

Mặc định SG cho phép tất cả outbound. Giới hạn lại để ngăn data exfiltration:

```json
// Outbound rules giới hạn cho App Server
[
  {"protocol": "tcp", "port": 5432, "destination": "sg-db", "desc": "To RDS"},
  {"protocol": "tcp", "port": 443, "destination": "pl-s3-prefix-list", "desc": "To S3"},
  {"protocol": "tcp", "port": 443, "destination": "pl-secretsmanager", "desc": "To Secrets Manager"}
]
```

---

## Ví Dụ Thực Tế

### Kiến Trúc Web Application hoàn chỉnh

```terraform
# Security Group cho ALB (Application Load Balancer)
resource "aws_security_group" "alb" {
  name        = "sg-alb-prod"
  description = "ALB — nhận HTTPS từ internet"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from internet"
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP redirect to HTTPS"
  }

  egress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "To App Server"
  }
}

# Security Group cho App Server
resource "aws_security_group" "app" {
  name        = "sg-app-prod"
  description = "App Server — chỉ nhận từ ALB"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "From ALB"
  }

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.db.id]
    description     = "To PostgreSQL"
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS to AWS services (S3, Secrets Manager)"
  }
}

# Security Group cho Database
resource "aws_security_group" "db" {
  name        = "sg-db-prod"
  description = "Database — chỉ nhận từ App Server"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "PostgreSQL from App Server"
  }
}
```

### Monitoring Security Groups với AWS Config

```bash
# Phát hiện Security Group mở port nguy hiểm
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "restricted-ssh",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "INCOMING_SSH_DISABLED"
  }
}'

# Kiểm tra compliance
aws configservice get-compliance-details-by-config-rule \
  --config-rule-name restricted-ssh
```

---

## Câu Hỏi Phỏng Vấn

### Q: Security Group có rule Deny không?

**Trả lời:** Không. Security Group chỉ có rule Allow. Traffic không khớp với bất kỳ rule Allow nào sẽ bị từ chối ngầm (implicit deny). Đây là sự khác biệt quan trọng với NACL — NACL có thể tạo rule Deny tường minh.

### Q: Một EC2 instance có thể gắn bao nhiêu Security Group?

**Trả lời:** Tối đa 5 Security Group per ENI (mặc định). Có thể tăng lên tối đa 16 bằng cách liên hệ AWS Support. Khi nhiều SG cùng áp dụng, rule được hợp nhất — nếu bất kỳ SG nào cho phép traffic, traffic đó được phép.

### Q: Tại sao nên dùng Security Group ID thay vì IP trong rule?

**Trả lời:** Khi dùng SG ID làm source, rule tự động áp dụng cho tất cả instance trong SG đó, bất kể IP. Điều này đặc biệt quan trọng trong môi trường Auto Scaling Group khi IP instance thay đổi liên tục. Cách tiếp cận này giảm lỗi cấu hình và dễ audit hơn.

### Q: Sự khác biệt giữa Security Group và NACL khi nào dùng cái nào?

**Trả lời:**
- **Security Group:** Kiểm soát chi tiết cấp instance; luôn dùng như lớp bảo vệ cơ bản
- **NACL:** Bảo vệ toàn subnet; hữu ích khi cần block một dải IP ngay lập tức (ví dụ: chặn IP tấn công) hoặc kiểm soát cấp subnet độc lập với Security Group

Dùng cả hai để có defense-in-depth.

### Q: Return traffic hoạt động thế nào với Security Group?

**Trả lời:** Security Group là stateful — nó theo dõi connection state (trạng thái kết nối). Khi một kết nối TCP được phép inbound, response packet được tự động cho phép outbound mà không cần rule outbound tương ứng. Điều này khác với NACL nơi phải tạo rule rõ ràng cho cả 2 chiều, bao gồm cả ephemeral ports (1024–65535).

---

## 🔗 Xem Thêm

- [2-nacls.md](2-nacls.md) — Bảo vệ cấp subnet với NACL
- [3-vpc-endpoints.md](3-vpc-endpoints.md) — Kết nối private đến dịch vụ AWS
- [README.md](README.md) — Tổng quan Network Security

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
