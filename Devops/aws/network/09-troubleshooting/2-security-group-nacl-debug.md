# 🛡️ Security Group & NACL Debug — Gỡ Lỗi Tường Lửa AWS

> Hướng dẫn chi tiết để chẩn đoán và sửa lỗi Security Group (Nhóm Bảo Mật — tường lửa stateful) và Network ACL — NACL (Danh Sách Kiểm Soát Truy Cập Mạng — tường lửa stateless) trong AWS VPC.

---

## 📚 Mục Lục

1. [Ôn Lại Sự Khác Biệt Quan Trọng](#ôn-lại-sự-khác-biệt-quan-trọng)
2. [Debug Security Group](#debug-security-group)
3. [Debug Network ACL](#debug-network-acl)
4. [Lỗi Thường Gặp Và Cách Sửa](#lỗi-thường-gặp-và-cách-sửa)
5. [Kịch Bản Phức Tạp](#kịch-bản-phức-tạp)
6. [Công Cụ Và Lệnh CLI Hỗ Trợ](#công-cụ-và-lệnh-cli-hỗ-trợ)
7. [Checklist Kiểm Tra](#checklist-kiểm-tra)

---

## ⚡ Ôn Lại Sự Khác Biệt Quan Trọng

Hiểu rõ sự khác biệt này là chìa khóa để debug nhanh:

| Đặc Điểm | Security Group (SG) | Network ACL (NACL) |
|----------|--------------------|--------------------|
| **Loại firewall** | Stateful (Có Trạng Thái) | Stateless (Phi Trạng Thái) |
| **Áp dụng cho** | EC2 instance (ENI) | Toàn bộ Subnet |
| **Chiều traffic** | Chỉ cần define inbound | Phải define cả inbound VÀ outbound |
| **Mặc định outbound** | ALLOW tất cả | Phụ thuộc vào rule |
| **Mặc định inbound** | DENY tất cả (không có rule) | ALLOW tất cả (NACL mặc định) |
| **Return traffic** | Tự động cho phép (stateful) | PHẢI có outbound rule tương ứng |
| **Rule order** | Tất cả rules được đánh giá | Đánh giá theo số thứ tự, dừng khi match |
| **Deny rule** | Không thể tạo explicit DENY | Có thể tạo explicit DENY |

### Điểm Quan Trọng Nhất

```
Security Group: Return traffic TỰ ĐỘNG được phép
               → Chỉ cần ALLOW inbound là đủ

NACL:           Return traffic PHẢI được ALLOW tường minh
               → Cần ALLOW cả inbound VÀ outbound
               → Ephemeral ports (1024-65535) thường hay bị quên
```

---

## 🔎 Debug Security Group

### Tìm Security Group Của EC2

```bash
# Xem tất cả SG gán cho EC2
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[].Instances[].[InstanceId, SecurityGroups]'

# Xem chi tiết rules của SG
aws ec2 describe-security-groups \
  --group-ids <sg-id>
```

### Đọc Kết Quả Security Group Rules

```json
{
  "IpPermissions": [
    {
      "FromPort": 443,
      "ToPort": 443,
      "IpProtocol": "tcp",
      "IpRanges": [{"CidrIp": "0.0.0.0/0"}]
    },
    {
      "FromPort": 22,
      "ToPort": 22,
      "IpProtocol": "tcp",
      "UserIdGroupPairs": [{"GroupId": "sg-abc123"}]
    }
  ]
}
```

Diễn giải:
- Rule 1: ALLOW TCP 443 từ mọi IP (`0.0.0.0/0`)
- Rule 2: ALLOW TCP 22 từ instances trong `sg-abc123` (Security Group reference — tham chiếu SG)

### Kiểm Tra Inbound Rules

**Câu hỏi cần trả lời:**
1. Port ứng dụng có được mở không?
2. Source IP hoặc CIDR có chính xác không?
3. Nếu dùng SG reference — SG đó có đúng không?

```bash
# Kiểm tra traffic từ IP cụ thể có được phép vào port 8080
aws ec2 describe-security-groups \
  --group-ids sg-abc123 \
  --query 'SecurityGroups[].IpPermissions[?FromPort==`8080`]'
```

### Kiểm Tra Outbound Rules

Security Group mặc định: **ALLOW tất cả outbound**.

Nếu đã tạo custom outbound rules, kiểm tra:

```bash
aws ec2 describe-security-groups \
  --group-ids <sg-id> \
  --query 'SecurityGroups[].IpPermissionsEgress'
```

**Lỗi hay gặp:** Custom outbound rules chặn response traffic về phía database hoặc API.

### Lỗi "Security Group Reference" Không Hoạt Động

Khi dùng SG-to-SG reference trong cùng VPC:
```
ALB Security Group (sg-alb): outbound ALLOW tất cả
EC2 Security Group (sg-ec2): inbound ALLOW từ sg-alb port 8080
```

**Lỗi thường gặp:**
- EC2 đang dùng `sg-ec2-old` thay vì `sg-ec2` mới
- SG reference trỏ sai SG ID
- EC2 ở VPC khác với ALB — SG cross-VPC reference không hoạt động

---

## 🔎 Debug Network ACL

### Tìm NACL Của Subnet

```bash
# Tìm NACL liên kết với subnet
aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=<subnet-id>" \
  --query 'NetworkAcls[].[NetworkAclId, Associations, Entries]'
```

### Đọc NACL Entries (Bản Ghi NACL)

```json
{
  "Entries": [
    {
      "RuleNumber": 100,
      "Protocol": "6",
      "RuleAction": "allow",
      "Egress": false,
      "CidrBlock": "0.0.0.0/0",
      "PortRange": {"From": 443, "To": 443}
    },
    {
      "RuleNumber": 32767,
      "Protocol": "-1",
      "RuleAction": "deny",
      "Egress": false,
      "CidrBlock": "0.0.0.0/0"
    }
  ]
}
```

Diễn giải:
- Rule 100: ALLOW TCP 443 inbound từ mọi IP
- Rule 32767: DENY tất cả (default rule — rule mặc định, không thể xóa)

**Protocol codes:**
- `6` = TCP
- `17` = UDP
- `-1` = Tất cả protocols

### Debug NACL Stateless — Vấn Đề Ephemeral Ports

Đây là lỗi phổ biến nhất với NACL.

**Ví dụ tình huống:**
```
Client (IP: 203.0.113.1, port nguồn: 54321) → EC2 (port đích: 443)
```

**NACL cần có:**
```
Inbound Rule:  ALLOW TCP 443 từ 0.0.0.0/0      ← Request vào
Outbound Rule: ALLOW TCP 1024-65535 đến 0.0.0.0/0 ← Response về phía client (port 54321)
```

Nếu thiếu outbound rule cho ephemeral ports, **client không nhận được response** dù request vào được.

```bash
# Kiểm tra outbound rule cho ephemeral ports
aws ec2 describe-network-acls \
  --network-acl-ids <nacl-id> \
  --query 'NetworkAcls[].Entries[?Egress==`true`]'
```

### NACL Rule Order — Thứ Tự Ưu Tiên

NACL đánh giá rules **từ số nhỏ nhất lên lớn nhất** và **dừng khi match**.

**Ví dụ lỗi phổ biến:**
```
Rule 100: DENY TCP port 22 từ 0.0.0.0/0    ← Chặn SSH toàn bộ
Rule 200: ALLOW TCP port 22 từ 10.0.0.0/8  ← Không bao giờ được đánh giá!
```

Rule 100 match trước cho mọi IP, nên rule 200 không bao giờ được áp dụng.

**Cách sửa:** Đặt ALLOW rule có số nhỏ hơn DENY rule:
```
Rule 100: ALLOW TCP port 22 từ 10.0.0.0/8  ← Admin IP range
Rule 200: DENY TCP port 22 từ 0.0.0.0/0    ← Chặn tất cả còn lại
```

---

## ⚠️ Lỗi Thường Gặp Và Cách Sửa

### Lỗi 1: SSH Timeout Nhưng Security Group Trông Có Vẻ Đúng

**Kiểm tra:**
1. NACL inbound: có ALLOW port 22 không?
2. NACL outbound: có ALLOW ephemeral ports (1024-65535) không?

```bash
# Thêm NACL outbound rule cho ephemeral ports nếu thiếu
aws ec2 create-network-acl-entry \
  --network-acl-id <nacl-id> \
  --rule-number 900 \
  --protocol 6 \
  --rule-action allow \
  --egress \
  --cidr-block 0.0.0.0/0 \
  --port-range From=1024,To=65535
```

### Lỗi 2: Ứng Dụng Trả Lời Được Nhưng Chậm/Intermittent

**Nguyên nhân thường gặp:** NACL chặn một số ephemeral port range, không phải tất cả.

**Kiểm tra Flow Logs:**
```sql
-- Tìm packets bị REJECT từ server về client
fields @timestamp, srcAddr, srcPort, dstAddr, dstPort, action
| filter srcAddr = "10.0.1.50" and action = "REJECT"
| sort @timestamp desc
```

### Lỗi 3: EC2 Mới Trong Auto Scaling Group Không Nhận Traffic Từ ALB

**Nguyên nhân:** Security Group của EC2 không cho phép traffic từ ALB Security Group.

**Kiểm tra và sửa:**
```bash
# Thêm inbound rule vào SG của EC2: ALLOW từ SG của ALB
aws ec2 authorize-security-group-ingress \
  --group-id <ec2-sg-id> \
  --protocol tcp \
  --port 8080 \
  --source-group <alb-sg-id>
```

### Lỗi 4: Database Từ Chối Kết Nối Từ EC2

**Checklist:**
```
□ Security Group của RDS: inbound port 3306/5432 từ SG của EC2?
□ EC2 và RDS ở cùng VPC không?
□ Nếu khác VPC: VPC Peering và route tables đã cấu hình?
□ RDS ở private subnet nhưng route table trỏ sai?
```

### Lỗi 5: Lambda Không Kết Nối Được RDS Trong VPC

Lambda cần:
- **VPC configuration** được cấu hình
- **Security Group** riêng cho Lambda được gán
- RDS Security Group: ALLOW inbound từ **Lambda Security Group**

```bash
# Kiểm tra VPC config của Lambda function
aws lambda get-function-configuration \
  --function-name <function-name> \
  --query 'VpcConfig'
```

---

## 🏗️ Kịch Bản Phức Tạp

### Kịch Bản: 3-Tier App (Internet → ALB → EC2 App → RDS)

```
Internet
  │
  ▼
ALB (sg-alb)
  │ Security Group: inbound 443 từ 0.0.0.0/0
  ▼
EC2 App (sg-app)
  │ Security Group: inbound 8080 từ sg-alb
  ▼
RDS (sg-rds)
  Security Group: inbound 5432 từ sg-app
```

**Chuỗi kiểm tra khi có lỗi:**

```
Step 1: curl -v https://<alb-dns>
        → Lỗi? → Kiểm tra SG của ALB và certificate
        → OK? → Tiếp Step 2

Step 2: Từ EC2 App: curl -v http://localhost:8080/health
        → Lỗi? → App không chạy, kiểm tra app logs
        → OK? → Tiếp Step 3

Step 3: ALB Health Check → Target Unhealthy?
        → Kiểm tra SG của EC2 App có ALLOW từ sg-alb port 8080?

Step 4: Từ EC2 App: nc -zv <rds-endpoint> 5432
        → Timeout? → Kiểm tra SG của RDS có ALLOW từ sg-app?
```

### Kịch Bản: Microservices Trong Kubernetes/ECS

Service A gọi Service B qua internal DNS:

```
Service A (sg-svc-a, subnet: 10.0.1.0/24)
  │
  ▼
Service B (sg-svc-b, subnet: 10.0.2.0/24)
```

**Security Group của Service B cần:**
```
Inbound: TCP <port-b> từ sg-svc-a
```

**NACL của subnet Service B cần:**
```
Inbound:  TCP <port-b> từ 10.0.1.0/24
Outbound: TCP 1024-65535 đến 10.0.1.0/24
```

---

## 🔧 Công Cụ Và Lệnh CLI Hỗ Trợ

### Xem Tất Cả EC2 Và Security Groups

```bash
# Danh sách EC2 và SG tương ứng
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,PrivateIpAddress,PublicIpAddress,State.Name,SecurityGroups[*].GroupId]' \
  --output table
```

### Tìm Security Groups Đang Mở Port Nguy Hiểm

```bash
# Tìm SG nào đang cho phép SSH từ mọi IP (0.0.0.0/0)
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?FromPort==`22` && IpRanges[?CidrIp==`0.0.0.0/0`]]].[GroupId,GroupName,VpcId]'
```

### Xem Tất Cả NACL Trong VPC

```bash
# Liệt kê NACL và subnet liên kết
aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'NetworkAcls[].[NetworkAclId, IsDefault, Associations[*].SubnetId]' \
  --output table
```

### Script Kiểm Tra Nhanh SG Cho EC2

```bash
#!/bin/bash
INSTANCE_ID=$1
echo "=== Security Groups for $INSTANCE_ID ==="
SG_IDS=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[].Instances[].SecurityGroups[].GroupId' \
  --output text)

for SG in $SG_IDS; do
  echo ""
  echo "--- SG: $SG ---"
  echo "INBOUND:"
  aws ec2 describe-security-groups --group-ids $SG \
    --query 'SecurityGroups[].IpPermissions[].[FromPort,ToPort,IpProtocol,IpRanges[*].CidrIp,UserIdGroupPairs[*].GroupId]' \
    --output table
  echo "OUTBOUND:"
  aws ec2 describe-security-groups --group-ids $SG \
    --query 'SecurityGroups[].IpPermissionsEgress[].[FromPort,ToPort,IpProtocol,IpRanges[*].CidrIp]' \
    --output table
done
```

---

## ✅ Checklist Kiểm Tra

### Security Group Checklist

```
□ EC2 có đúng Security Group được gán không?
□ Inbound rule có đúng port không?
□ Inbound source có đúng IP/CIDR/SG-reference không?
□ Nếu dùng SG-reference: hai resources có cùng VPC không?
□ Outbound rule: mặc định ALLOW all — có bị customize giới hạn không?
□ Không nhầm lẫn inbound/outbound direction?
```

### NACL Checklist

```
□ NACL đúng subnet đang gặp vấn đề?
□ Inbound ALLOW cho port cần thiết?
□ Outbound ALLOW cho ephemeral ports (1024-65535)?
□ Kiểm tra rule number order — ALLOW trước DENY?
□ CIDR range có đúng không?
□ Protocol có đúng không (TCP=6, UDP=17, ALL=-1)?
□ Nếu có DENY rule — có ảnh hưởng return traffic không?
```

### So Sánh SG vs NACL Khi Debug

```
Vấn đề:                  Kiểm tra:
Connection timeout    →  NACL đang block (stateless, im lặng drop)
Connection refused    →  SG đang block HOẶC app không chạy
Intermittent timeout  →  NACL block return/ephemeral traffic
Works only sometimes  →  Multi-AZ: NACL khác nhau giữa các AZ
```

---

## 📚 Tài Liệu Liên Quan

- [1-connectivity-debug.md](./1-connectivity-debug.md) — Debug kết nối tổng quát
- [../02-security/1-security-groups.md](../02-security/1-security-groups.md) — Security Groups chi tiết
- [../02-security/2-network-acls.md](../02-security/2-network-acls.md) — Network ACLs chi tiết
- [../08-monitoring/1-vpc-flow-logs.md](../08-monitoring/1-vpc-flow-logs.md) — VPC Flow Logs phân tích

---

**Cập Nhật Lần Cuối:** 2026-05-14
