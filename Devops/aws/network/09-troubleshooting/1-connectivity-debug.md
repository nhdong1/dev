# 🔌 EC2 Connectivity Debug — Chẩn Đoán Kết Nối EC2 Từng Bước

> Hướng dẫn có hệ thống để chẩn đoán khi EC2 (Elastic Compute Cloud — Máy Chủ Ảo) không kết nối được từ internet, từ VPC khác, hoặc không ra được internet.

---

## 📚 Mục Lục

1. [Ba Kịch Bản Kết Nối Phổ Biến](#ba-kịch-bản-kết-nối-phổ-biến)
2. [Kịch Bản 1: Không SSH/RDP Được Vào EC2](#kịch-bản-1-không-sshrdp-được-vào-ec2)
3. [Kịch Bản 2: EC2 Không Ra Được Internet](#kịch-bản-2-ec2-không-ra-được-internet)
4. [Kịch Bản 3: Hai EC2 Không Nói Chuyện Được Với Nhau](#kịch-bản-3-hai-ec2-không-nói-chuyện-được-với-nhau)
5. [Dùng VPC Reachability Analyzer](#dùng-vpc-reachability-analyzer)
6. [Phân Tích VPC Flow Logs](#phân-tích-vpc-flow-logs)
7. [Checklist Nhanh](#checklist-nhanh)

---

## 🗺️ Ba Kịch Bản Kết Nối Phổ Biến

```
Kịch Bản 1: Internet → EC2 (SSH/HTTP/HTTPS từ ngoài vào)
Kịch Bản 2: EC2 → Internet (EC2 cần gọi API bên ngoài, update package)
Kịch Bản 3: EC2-A → EC2-B (giao tiếp nội bộ trong VPC hoặc giữa các VPC)
```

---

## 🔴 Kịch Bản 1: Không SSH/RDP Được Vào EC2

### Sơ Đồ Kiểm Tra

```
Internet
   │
   ▼
[Route 53 DNS] ──→ phân giải IP Public EC2 hoặc EIP
   │
   ▼
[Internet Gateway — IGW] ──→ có tồn tại và attach vào VPC?
   │
   ▼
[Route Table] ──→ có route 0.0.0.0/0 → IGW không?
   │
   ▼
[Network ACL — NACL] ──→ có ALLOW inbound port 22 không?
   │
   ▼
[Security Group] ──→ có ALLOW inbound port 22 từ IP bạn không?
   │
   ▼
[EC2 Instance] ──→ có đang chạy? SSH daemon có hoạt động không?
```

### Bước 1: Xác Nhận EC2 Có IP Public

```bash
# Kiểm tra EC2 có Public IP hay Elastic IP (IP Đàn Hồi) không
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[].Instances[].[PublicIpAddress, PublicDnsName, State.Name]'
```

**Vấn đề thường gặp:**
- EC2 được launch trong **private subnet** — không có Public IP
- Subnet không bật **Auto-assign Public IP** (Tự Động Gán IP Công Khai)
- EC2 ở trạng thái `stopped` hoặc `terminated`

### Bước 2: Kiểm Tra EC2 Ở Đúng Subnet (Mạng Con)

```bash
# Xem subnet của EC2
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[].Instances[].[SubnetId, VpcId]'

# Kiểm tra subnet có route ra Internet Gateway không
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<subnet-id>"
```

**Subnet công khai (Public Subnet) phải có:**
- Route `0.0.0.0/0` → Internet Gateway (IGW)

### Bước 3: Kiểm Tra Security Group

```bash
# Xem Security Group của EC2
aws ec2 describe-security-groups \
  --group-ids <sg-id> \
  --query 'SecurityGroups[].IpPermissions'
```

**Cần có inbound rule:**
```
Type: SSH (hoặc RDP)
Protocol: TCP
Port: 22 (hoặc 3389)
Source: <your-ip>/32 hoặc 0.0.0.0/0 (kém bảo mật)
```

> ⚠️ **Lưu ý bảo mật:** Không mở port 22 cho `0.0.0.0/0` trên production. Dùng AWS Systems Manager Session Manager thay thế.

### Bước 4: Kiểm Tra Network ACL (NACL)

NACL là **stateless** (Phi Trạng Thái) — phải có cả inbound VÀ outbound rules.

```bash
# Tìm NACL liên kết với subnet
aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=<subnet-id>"
```

**Checklist NACL cho SSH:**
```
Inbound:  ALLOW TCP port 22 từ source IP
Outbound: ALLOW TCP port 1024-65535 (ephemeral ports — cổng tạm thời) về phía client
```

### Bước 5: Kiểm Tra SSH Daemon Trong EC2

Nếu network hoàn toàn ổn nhưng vẫn không SSH được:

```bash
# Dùng EC2 Serial Console (Giao Diện Cổng Nối Tiếp) hoặc EC2 Instance Connect
# Kiểm tra SSH service
sudo systemctl status sshd

# Xem SSH logs
sudo tail -f /var/log/auth.log      # Ubuntu/Debian
sudo tail -f /var/log/secure        # Amazon Linux/RHEL
```

**Lỗi thường gặp:**
- Sai SSH key pair (Cặp Khóa SSH) — key bạn dùng không match
- `sshd_config` bị sửa để chặn root login hoặc password auth
- Disk full khiến sshd không khởi động được

---

## 🟡 Kịch Bản 2: EC2 Không Ra Được Internet

### Sơ Đồ Kiểm Tra

```
EC2 (Private Subnet)
   │
   ▼
[Route Table] ──→ có route 0.0.0.0/0 → NAT Gateway không?
   │
   ▼
[NAT Gateway] ──→ có tồn tại, ở đúng PUBLIC subnet, ở trạng thái Available?
   │
   ▼
[Public Subnet Route Table] ──→ có route 0.0.0.0/0 → IGW không?
   │
   ▼
[Internet Gateway] ──→ có attach vào VPC không?
   │
   ▼
Internet
```

### Bước 1: Kiểm Tra Route Table Của EC2

```bash
# EC2 ở private subnet — route table phải có NAT Gateway
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<private-subnet-id>" \
  --query 'RouteTables[].Routes'
```

**Cần có:**
```
Destination: 0.0.0.0/0
Target: nat-xxxxxxxxx (NAT Gateway ID)
```

### Bước 2: Kiểm Tra NAT Gateway

```bash
# Xem trạng thái NAT Gateway
aws ec2 describe-nat-gateways \
  --filter "Name=state,Values=available" \
  --query 'NatGateways[].[NatGatewayId, State, SubnetId, NatGatewayAddresses]'
```

**NAT Gateway phải:**
- Ở trạng thái `available`
- Nằm trong **public subnet** (không phải private subnet)
- Có Elastic IP (EIP) được gán

### Bước 3: Kiểm Tra Public Subnet Của NAT Gateway

NAT Gateway nằm trong public subnet — subnet đó cần route `0.0.0.0/0 → IGW`.

```bash
# Kiểm tra route table của public subnet chứa NAT Gateway
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<public-subnet-id>"
```

### Bước 4: Test Kết Nối Từ EC2

```bash
# Từ EC2, test ping DNS Google
ping 8.8.8.8

# Test TCP đến cổng HTTPS
curl -v https://aws.amazon.com --connect-timeout 5

# Kiểm tra default route
ip route show default
```

**Kết quả mong đợi:**
```
default via 10.0.1.1 dev eth0    # 10.0.1.1 là router của subnet
```

---

## 🔵 Kịch Bản 3: Hai EC2 Không Nói Chuyện Được Với Nhau

### Trường Hợp 3a: Cùng VPC, Cùng Subnet

```bash
# Kiểm tra Security Group của EC2-B có cho phép inbound từ EC2-A không
# Source có thể là Security Group ID của EC2-A (tốt hơn dùng IP)
```

**Cách kiểm tra tốt nhất:**
1. Vào EC2-B Security Group
2. Thêm inbound rule: source = `Security Group của EC2-A`
3. Test lại với `nc -zv <ip-ec2-b> <port>`

### Trường Hợp 3b: Cùng VPC, Khác Subnet

Bổ sung kiểm tra:
- Route Table của **cả hai subnet** — thường không cần route đặc biệt trong cùng VPC
- NACL của **cả hai subnet** — nhớ NACL stateless, cần cả inbound lẫn outbound

### Trường Hợp 3c: Khác VPC (VPC Peering — Kết Nối Ngang Hàng VPC)

```bash
# Kiểm tra VPC Peering connection status
aws ec2 describe-vpc-peering-connections \
  --query 'VpcPeeringConnections[].[VpcPeeringConnectionId, Status.Code, RequesterVpcInfo, AccepterVpcInfo]'
```

**Checklist VPC Peering:**
```
□ Peering connection ở trạng thái "active"
□ Route Table VPC-A: <CIDR-VPC-B> → pcx-xxxxxx
□ Route Table VPC-B: <CIDR-VPC-A> → pcx-xxxxxx
□ Security Group EC2-B: ALLOW inbound từ CIDR của VPC-A
□ NACL: ALLOW inbound VÀ outbound cho CIDR tương ứng
□ Không có CIDR overlap (chồng lấn) giữa hai VPC
```

---

## 🔬 Dùng VPC Reachability Analyzer

VPC Reachability Analyzer (Bộ Phân Tích Khả Năng Tiếp Cận VPC) kiểm tra đường đi của packet mà **không cần gửi traffic thật** — rất hữu ích để debug mà không ảnh hưởng production.

```bash
# Bước 1: Tạo Network Insights Path (Đường Đi Network Insights)
aws ec2 create-network-insights-path \
  --source <source-instance-id> \
  --destination <destination-instance-id> \
  --protocol tcp \
  --destination-port 443 \
  --tag-specifications 'ResourceType=network-insights-path,Tags=[{Key=Name,Value=debug-path}]'

# Bước 2: Chạy phân tích
aws ec2 start-network-insights-analysis \
  --network-insights-path-id <path-id>

# Bước 3: Xem kết quả
aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids <analysis-id> \
  --query 'NetworkInsightsAnalyses[].[NetworkInsightsAnalysisId,Status,NetworkPathFound,ExplanationCodes]'
```

**Kết quả trả về:**
- `NetworkPathFound: true` — kết nối có thể thiết lập
- `NetworkPathFound: false` + `ExplanationCodes` — chỉ ra đúng chỗ bị chặn

---

## 📋 Phân Tích VPC Flow Logs

### Bật Flow Logs Cho VPC

```bash
# Tạo Flow Log ghi vào CloudWatch Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids <vpc-id> \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn <iam-role-arn>
```

### Đọc Flow Log Record

Format mặc định của một dòng log:
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

Ví dụ thực tế:
```
2 123456789 eni-abc123 203.0.113.1 10.0.1.50 54321 443 6 10 4000 1620000000 1620000060 ACCEPT OK
2 123456789 eni-abc123 10.0.1.50 203.0.113.1 443 54321 6 10 8000 1620000000 1620000060 ACCEPT OK
```

### Query Bằng CloudWatch Logs Insights

```sql
-- Tìm traffic bị REJECT đến EC2 cụ thể trong 1 giờ qua
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, action
| filter dstAddr = "10.0.1.50" and action = "REJECT"
| sort @timestamp desc
| limit 50

-- Tìm nguồn IP nào đang kết nối nhiều nhất
stats count(*) as connections by srcAddr
| filter dstAddr = "10.0.1.50" and action = "ACCEPT"
| sort connections desc
| limit 20
```

---

## ✅ Checklist Nhanh

### Inbound (Internet vào EC2)

```
□ EC2 đang chạy (state: running)?
□ EC2 có Public IP hoặc Elastic IP?
□ EC2 nằm trong Public Subnet?
□ Route Table: 0.0.0.0/0 → IGW?
□ Security Group: ALLOW inbound port cần thiết?
□ NACL: ALLOW inbound port cần thiết?
□ NACL: ALLOW outbound ephemeral ports (1024-65535)?
□ SSH/App service đang chạy trong EC2?
□ Đúng key pair đang dùng?
```

### Outbound (EC2 ra Internet)

```
□ EC2 ở private subnet?
□ Route Table: 0.0.0.0/0 → NAT Gateway?
□ NAT Gateway ở trạng thái "available"?
□ NAT Gateway nằm trong public subnet?
□ NAT Gateway có Elastic IP?
□ Public subnet của NAT Gateway: 0.0.0.0/0 → IGW?
□ Security Group: ALLOW outbound (mặc định tất cả outbound đều được phép)?
□ NACL: ALLOW outbound và inbound return traffic?
```

### EC2 sang EC2 (Nội Bộ)

```
□ Security Group đích: ALLOW inbound từ SG nguồn hoặc CIDR nguồn?
□ NACL cả hai subnet: ALLOW cả hai chiều?
□ Nếu khác VPC: Peering connection active?
□ Nếu khác VPC: Route tables cả hai VPC đã cập nhật?
□ Không có CIDR overlap giữa các VPC?
```

---

## 📚 Tài Liệu Liên Quan

- [2-security-group-nacl-debug.md](./2-security-group-nacl-debug.md) — Debug chi tiết Security Group & NACL
- [../02-security/1-security-groups.md](../02-security/1-security-groups.md) — Security Groups toàn diện
- [../08-monitoring/4-reachability-analyzer.md](../08-monitoring/4-reachability-analyzer.md) — Reachability Analyzer chi tiết

---

**Cập Nhật Lần Cuối:** 2026-05-14
