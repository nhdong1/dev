# Elastic IP (IP Đàn Hồi) & ENI — Elastic Network Interface (Giao Diện Mạng Đàn Hồi)

> **Thuộc topic:** 07-advanced-networking | **Mức độ:** Nâng cao

---

## Phần 1: Elastic IP (EIP) — Địa Chỉ IP Đàn Hồi

## 1. Elastic IP Là Gì?

**Elastic IP (EIP)** là địa chỉ **IPv4 tĩnh (static)** thuộc sở hữu của **AWS account**, không phải của một instance cụ thể. Bạn có thể gán/tháo EIP vào/ra bất kỳ EC2 instance hoặc ENI nào trong cùng region bất kỳ lúc nào.

### So sánh các loại Public IP trong AWS

| Loại IP | Tĩnh/Động | Thuộc về | Mất khi stop? | Chi phí |
|---------|-----------|----------|--------------|---------|
| **Auto-assigned Public IP** | Động (mỗi lần start/stop đổi) | Instance | Có | Miễn phí |
| **Elastic IP (EIP)** | Tĩnh (không bao giờ đổi) | Account | Không | Có phí khi không gán |
| **ALB/NLB DNS** | DNS (IP có thể đổi) | AWS | N/A | Miễn phí (service fee) |

### Vòng đời của Elastic IP

```
1. Allocate (Phân Bổ):
   Account ← EIP được cấp từ AWS public IP pool (hoặc BYOIP)
   
2. Associate (Gán vào):
   EIP → EC2 Instance (hoặc ENI)
   Instance nhận public IP tĩnh
   
3. Disassociate (Tháo ra):
   EIP ← tháo khỏi instance
   Instance mất public IP (hoặc nhận lại Auto-assigned IP tùy config)
   
4. Release (Trả lại):
   EIP → trả lại AWS IP pool
   IP có thể được cấp cho account khác
```

---

## 2. Giới Hạn và Chi Phí EIP

### Giới hạn mặc định

```
5 Elastic IP addresses per region per account (mặc định)
```

Có thể yêu cầu tăng thông qua AWS Service Quotas console.

### Billing (Tính phí)

```
EIP MIỄN PHÍ khi:
  ✓ Được gán vào running EC2 instance
  ✓ Instance đang ở trạng thái running

EIP TỐN PHÍ khi:
  ✗ Allocated nhưng KHÔNG gán vào instance nào
  ✗ Gán vào stopped instance
  ✗ Gán vào ENI nhưng ENI không gắn vào instance
  
Mức phí: $0.005/giờ/EIP không được dùng (~$3.6/tháng)
```

**Lý do tính phí khi không dùng:** AWS có giới hạn IP public pool — tính phí để discourage người dùng giữ IP mà không dùng.

### BYOIP — Bring Your Own IP (Mang IP Của Bạn)

Nếu tổ chức có IP public riêng (own IP block), có thể đăng ký dùng IP đó với AWS:

```bash
# Đăng ký BYOIP address pool
aws ec2 provision-byoip-cidr \
  --cidr 203.0.113.0/24 \
  --cidr-authorization-context Message=xxx,Signature=yyy

# Advertise (bắt đầu route)
aws ec2 advertise-byoip-cidr --cidr 203.0.113.0/24

# Allocate EIP từ BYOIP pool của bạn
aws ec2 allocate-address \
  --public-ipv4-pool ipv4pool-ec2-01234567890abcdef
```

---

## 3. Thao Tác Với Elastic IP

### AWS CLI

```bash
# Phân bổ EIP mới
aws ec2 allocate-address --domain vpc

# Output:
# {
#   "PublicIp": "54.123.45.67",
#   "AllocationId": "eipalloc-0a1b2c3d4e5f67890",
#   "Domain": "vpc"
# }

# Gán EIP vào EC2 instance
aws ec2 associate-address \
  --allocation-id eipalloc-0a1b2c3d4e5f67890 \
  --instance-id i-0a1b2c3d4e5f67890

# Gán EIP vào ENI (thay vì instance)
aws ec2 associate-address \
  --allocation-id eipalloc-0a1b2c3d4e5f67890 \
  --network-interface-id eni-0a1b2c3d4e5f67890

# Tháo EIP khỏi instance
aws ec2 disassociate-address \
  --association-id eipassoc-0a1b2c3d4e5f67890

# Trả EIP về AWS
aws ec2 release-address \
  --allocation-id eipalloc-0a1b2c3d4e5f67890

# Xem tất cả EIP trong account
aws ec2 describe-addresses \
  --query 'Addresses[*].{IP:PublicIp,AllocID:AllocationId,InstanceID:InstanceId,State:AssociationState}'
```

---

## 4. Use Cases Cho Elastic IP

### Use Case 1: High-Availability Failover thủ công

```
Bình thường:
  User → EIP: 54.123.45.67 → Primary EC2 (i-primary)

Khi primary fail:
  Admin/Script:
    1. Disassociate EIP từ i-primary
    2. Associate EIP vào i-standby
  
  User → EIP: 54.123.45.67 (vẫn cùng IP) → Standby EC2 (i-standby)
  (Không cần update DNS, không cần thông báo khách hàng)
```

```bash
# Script failover EIP
#!/bin/bash
PRIMARY_INSTANCE="i-0a1b2c3d4e5f67890"
STANDBY_INSTANCE="i-0fedcba987654321"
EIP_ALLOC_ID="eipalloc-0a1b2c3d4e5f67890"

# Lấy association ID hiện tại
ASSOC_ID=$(aws ec2 describe-addresses \
  --allocation-ids $EIP_ALLOC_ID \
  --query 'Addresses[0].AssociationId' \
  --output text)

# Tháo khỏi primary
aws ec2 disassociate-address --association-id $ASSOC_ID

# Gán vào standby
aws ec2 associate-address \
  --allocation-id $EIP_ALLOC_ID \
  --instance-id $STANDBY_INSTANCE

echo "Failover hoàn thành: EIP chuyển sang $STANDBY_INSTANCE"
```

### Use Case 2: NAT Instance

```
Private EC2 → NAT Instance (có EIP) → Internet
              (Elastic IP là IP nguồn cho traffic ra ngoài)
```

**Lưu ý:** Ngày nay, NAT Gateway được ưa chuộng hơn NAT Instance vì:
- Managed service (không cần maintain OS)
- Scale tự động
- High availability built-in

NAT Instance với EIP chỉ còn phù hợp khi cần tiết kiệm chi phí ở scale nhỏ (NAT Gateway tốn $0.059/giờ).

### Use Case 3: Bastion Host (Máy Chủ Pháo Đài)

```
Admin → EIP (IP cố định) → Bastion Host → SSH vào Private EC2
       (Whitelist EIP trong Security Group của private instances)
```

EIP đảm bảo bastion host luôn có cùng IP để whitelist trong security groups và on-premises firewall rules.

### Use Case 4: DNS A Record cho EC2

```
Nếu dùng Auto-assigned Public IP:
  → IP thay đổi mỗi lần stop/start
  → DNS A record phải update thủ công hoặc tự động (phức tạp)

Nếu dùng Elastic IP:
  → IP không bao giờ thay đổi
  → DNS A record ổn định
  → Không cần automation để update DNS
```

---

## 5. Best Practices và Anti-Patterns

### Best Practices

1. **Gắn EIP vào ENI thay vì instance trực tiếp**
   - Khi cần failover, chỉ cần detach ENI từ instance cũ và attach vào instance mới
   - EIP, private IP, Security Groups đều di chuyển cùng ENI

2. **Sử dụng ALB/NLB thay vì EIP khi có thể**
   - ALB/NLB cung cấp HA, auto-scaling, health checks tốt hơn
   - EIP → Load Balancer → nhiều EC2 instances (tốt hơn EIP → 1 EC2)

3. **Giám sát EIP chưa được gán**
   - Tạo AWS Config rule để alert khi có EIP không được gán (tốn tiền vô ích)
   - Hoặc dùng Lambda + EventBridge để auto-release EIP không dùng

4. **Tag EIP rõ ràng**
   ```
   Name: bastion-host-eip
   Environment: production
   Owner: team-infrastructure
   ```

### Anti-Patterns

- **Không dùng EIP cho webservers trong production** nếu có nhiều instances — dùng ALB thay thế
- **Không giữ EIP "phòng khi cần"** — sẽ tốn $0.005/giờ vô ích
- **Không release EIP của production** mà không kiểm tra — IP sẽ trả về pool và có thể cấp cho account khác

---

---

## Phần 2: ENI — Elastic Network Interface (Giao Diện Mạng Đàn Hồi)

## 6. ENI Là Gì?

**ENI (Elastic Network Interface)** là **virtual NIC** (card mạng ảo) trong VPC. Mỗi EC2 instance có ít nhất một ENI (**primary ENI** — `eth0`), và có thể attach thêm **secondary ENIs**.

ENI là một AWS resource độc lập — có thể tồn tại độc lập với instance, và được attach/detach giữa các instances.

---

## 7. Các Thuộc Tính (Attributes) Của ENI

| Thuộc tính | Mô tả |
|-----------|-------|
| **Primary Private IP** | IP private chính, gán từ subnet CIDR |
| **Secondary Private IPs** | Nhiều IP private phụ từ cùng subnet |
| **Public IP** | Gán tự động khi tạo (tùy subnet config) |
| **Elastic IP** | Có thể gán EIP vào primary hoặc secondary private IP |
| **MAC Address** | Địa chỉ MAC cố định, không thay đổi khi detach/attach |
| **Security Groups** | ENI có SG riêng (tối đa 5 SG per ENI) |
| **Source/Dest Check** | Bật/tắt kiểm tra nguồn/đích (tắt khi làm NAT/VPN) |
| **Description** | Mô tả tùy chọn |
| **Subnet** | ENI thuộc về một subnet cố định |
| **Availability Zone** | Cùng AZ với subnet |

```bash
# Xem chi tiết ENI
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-0a1b2c3d4e5f67890 \
  --query 'NetworkInterfaces[0].{
    ID:NetworkInterfaceId,
    PrivateIP:PrivateIpAddress,
    SecondaryIPs:PrivateIpAddresses[*].PrivateIpAddress,
    PublicIP:Association.PublicIp,
    MAC:MacAddress,
    State:Status,
    Subnet:SubnetId,
    SGs:Groups[*].GroupId
  }'
```

---

## 8. Primary ENI vs Secondary ENI

### Primary ENI (eth0)

- Được tạo tự động khi launch EC2 instance
- **Không thể detach** khi instance đang chạy
- Có thể detach sau khi instance bị terminate (nếu cấu hình preserve on termination)
- Instance public IP và hostname (nếu có) gắn với primary ENI

### Secondary ENI (eth1, eth2...)

- Được tạo độc lập và attach vào instance
- **Có thể detach/attach** khi instance đang chạy (hot-attach)
- Được giữ lại sau khi instance terminate (ENI không bị xóa)
- Mỗi instance type có giới hạn số ENI khác nhau

**Số ENI tối đa theo instance type:**

| Instance Type | Max ENIs | Max Private IPs per ENI |
|--------------|---------|------------------------|
| t3.micro | 2 | 2 |
| t3.medium | 3 | 6 |
| m5.large | 3 | 10 |
| m5.xlarge | 4 | 15 |
| c5.4xlarge | 8 | 30 |
| r5.metal | 15 | 50 |

```bash
# Xem giới hạn ENI của instance type
aws ec2 describe-instance-types \
  --instance-types m5.large \
  --query 'InstanceTypes[0].NetworkInfo.{MaxENIs:MaximumNetworkInterfaces,MaxIPsPerENI:Ipv4AddressesPerInterface}'
```

---

## 9. ENI Use Cases

### Use Case 1: Dual-Home Instance (Instance Hai Giao Diện Mạng)

```
                    Public Subnet 10.0.1.0/24
                            │
                     eth0 (ENI-1)
                     IP: 10.0.1.50
                            │
                    ┌───────────────┐
                    │  EC2 Instance │
                    └───────────────┘
                            │
                     eth1 (ENI-2)
                     IP: 10.0.2.60
                            │
                    Private Subnet 10.0.2.0/24
                            │
                    Database / Internal services
```

Ứng dụng: Web server với một interface nhận traffic công khai (port 80/443) và một interface kết nối database nội bộ — với Security Groups khác nhau trên từng ENI.

### Use Case 2: Low-Budget HA Failover (Dự Phòng Cao Tải Chi Phí Thấp)

```
Primary EC2      Secondary EC2
     │                │
     └──── ENI ───────┘
          (floating)

Bình thường: ENI attach vào Primary EC2
Khi Primary fail: Detach ENI từ Primary, Attach vào Secondary
→ IP, MAC, SG di chuyển nguyên vẹn sang Secondary
→ Ứng dụng tiếp tục hoạt động với cùng IP private
```

**So sánh với EIP failover:**

| Phương pháp | Di chuyển gì | Thời gian failover |
|------------|-------------|-------------------|
| EIP failover | Public IP | 10-30 giây (thay đổi cần time) |
| ENI failover | Private IP + MAC + SG | 10-30 giây |
| ENI + EIP | Cả public và private IP | 10-30 giây |

### Use Case 3: Network Appliances (Thiết Bị Mạng Ảo)

Firewall ảo, VPN concentrator, IDS/IPS cần nhiều ENIs để:
- ENI quản lý (management interface)
- ENI phía ngoài (external/WAN facing)
- ENI phía trong (internal/LAN facing)

```bash
# Tạo ENI cho network appliance
aws ec2 create-network-interface \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --description "External interface for firewall" \
  --groups sg-external-firewall \
  --no-source-dest-check   # Tắt source/dest check cho network appliance

# Attach ENI vào instance
aws ec2 attach-network-interface \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --instance-id i-0fedcba987654321 \
  --device-index 1   # eth1
```

### Use Case 4: ENI Warm Pool cho Lambda trong VPC

**Vấn đề cold start:** Lambda trong VPC phải tạo ENI mỗi lần cold start, mất 10-15 giây trước 2019. Từ 2019, AWS giải quyết bằng cách sử dụng **shared ENIs** (Hyperplane ENIs):

```
Lambda service (internal)
  → Tạo sẵn pool của Hyperplane ENIs trong subnet của bạn
  → Lambda function execution containers dùng chung ENIs này
  → Cold start chỉ mất ~100ms thay vì 10-15 giây

Điều này có nghĩa: Lambda trong VPC không còn là vấn đề hiệu suất lớn nữa
(Từ Lambda Improvements 2019 — AWS bỏ VPC-attached ENI per invocation)
```

---

## 10. Detach và Reattach ENI

```bash
# Tạo ENI độc lập (không gắn vào instance nào)
aws ec2 create-network-interface \
  --subnet-id subnet-0a1b2c3d4e5f67890 \
  --private-ip-address 10.0.1.100 \
  --description "Floating ENI for HA" \
  --groups sg-0a1b2c3d4e5f67890

# Attach ENI vào instance đang chạy
aws ec2 attach-network-interface \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --instance-id i-0a1b2c3d4e5f67890 \
  --device-index 1

# Detach ENI từ instance (instance vẫn chạy)
ATTACHMENT_ID=$(aws ec2 describe-network-interfaces \
  --network-interface-ids eni-0a1b2c3d4e5f67890 \
  --query 'NetworkInterfaces[0].Attachment.AttachmentId' \
  --output text)

aws ec2 detach-network-interface \
  --attachment-id $ATTACHMENT_ID \
  --force   # Force detach ngay cả khi instance đang busy

# Attach ENI vào instance khác (failover)
aws ec2 attach-network-interface \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --instance-id i-standby123456789 \
  --device-index 1
```

---

## 11. Secondary Private IPs — IP Private Thứ Cấp

Một ENI có thể có **nhiều private IP addresses** từ cùng subnet CIDR. Điều này có nhiều ứng dụng quan trọng.

### Use Case: Container Networking trên EKS/ECS

AWS VPC CNI (Container Network Interface — Giao Diện Mạng Container) plugin cho EKS dùng secondary IPs:

```
Node EC2 (m5.large) → max 3 ENIs × 10 IPs = 30 IPs tổng
                    → 1 IP cho node (primary)
                    → 29 IPs cho Pods

Mỗi Pod nhận một secondary private IP thật trong VPC
→ Pods có thể kết nối trực tiếp với các service trong VPC
→ Không cần overlay network (như Flannel, Calico)
→ Không có NAT ở trong cluster
→ Security Groups có thể apply trực tiếp cho từng Pod
```

```bash
# Thêm secondary private IP vào ENI
aws ec2 assign-private-ip-addresses \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --secondary-private-ip-address-count 5   # Thêm 5 IP phụ

# Hoặc chỉ định IP cụ thể
aws ec2 assign-private-ip-addresses \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --private-ip-addresses 10.0.1.101 10.0.1.102 10.0.1.103

# Xóa secondary IP
aws ec2 unassign-private-ip-addresses \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --private-ip-addresses 10.0.1.101
```

### Use Case: Multiple SSL/TLS Certificates trên cùng server

Trước SNI (Server Name Indication — Chỉ Thị Tên Máy Chủ), mỗi SSL certificate cần một IP riêng:

```
Nginx server với 3 virtual hosts:
  10.0.1.10 → site1.example.com (cert 1)
  10.0.1.11 → site2.example.com (cert 2)
  10.0.1.12 → site3.example.com (cert 3)

Tất cả 3 IP là secondary IPs trên cùng một ENI của cùng EC2 instance
```

(Ngày nay SNI được hỗ trợ rộng rãi nên cách này ít cần thiết hơn.)

---

## 12. Enhanced Networking — Mạng Tăng Cường

Enhanced Networking cải thiện throughput, latency và jitter cho EC2 instances thông qua **SR-IOV (Single Root I/O Virtualization — Ảo Hóa I/O Gốc Đơn)**.

### ENA — Elastic Network Adapter

**ENA** (Elastic Network Adapter — Bộ Thích Ứng Mạng Đàn Hồi) là driver enhanced networking mới nhất của AWS:
- Hỗ trợ tốc độ lên đến **100 Gbps**
- Packet per second (PPS) cao hơn
- Latency thấp hơn, jitter thấp hơn
- Được hỗ trợ bởi hầu hết instance types mới

```bash
# Kiểm tra ENA support
aws ec2 describe-instances \
  --instance-ids i-0a1b2c3d4e5f67890 \
  --query 'Reservations[0].Instances[0].EnaSupport'
# Output: true

# Bật ENA cho instance (instance phải stopped)
aws ec2 modify-instance-attribute \
  --instance-id i-0a1b2c3d4e5f67890 \
  --ena-support

# Kiểm tra từ trong instance (Linux)
ethtool -i eth0 | grep driver
# driver: ena   ← ENA đang active
```

### Intel 82599 VF Interface (VF — Virtual Function)

Loại Enhanced Networking cũ, tốc độ tối đa 10 Gbps. Chỉ còn trên một số instance types cũ.

### Placement Groups (Nhóm Đặt) + Enhanced Networking

Để tối đa băng thông giữa các instances:

```
Cluster Placement Group (Nhóm Đặt Cụm):
  → Các instances nằm trong cùng availability zone, cùng rack
  → Băng thông giữa instances: lên đến 25 Gbps (single flow)
  → Latency inter-instance thấp nhất
  → Phù hợp: HPC, Big Data, tightly coupled workloads

Spread Placement Group (Nhóm Đặt Phân Tán):
  → Mỗi instance ở rack khác nhau
  → Tối đa isolation, tránh hardware failure ảnh hưởng nhiều instances
  → Phù hợp: critical instances cần fault isolation
```

---

## 13. Bảng So Sánh: EIP vs Public IP Tự Động vs ALB DNS

| Khía cạnh | Auto-assigned Public IP | Elastic IP (EIP) | ALB/NLB DNS |
|-----------|------------------------|-----------------|-------------|
| **Tính thay đổi** | Thay đổi khi stop/start | Tĩnh, không đổi | DNS có thể thay đổi |
| **Chi phí** | Miễn phí | Miễn phí khi gán + $0.005/giờ khi không dùng | Phí theo service |
| **High Availability** | Không | Thủ công (script failover) | Tự động (built-in) |
| **Scale** | 1 instance | 1 IP → 1 instance tại một thời điểm | Nhiều instances |
| **DNS record** | Phải update thường xuyên | Ổn định (A record cố định) | CNAME/Alias (thay đổi) |
| **Firewall whitelist** | Khó (IP thay đổi) | Dễ (IP cố định) | Khó (DNS thay đổi) |
| **Use case phù hợp** | Dev/Test | Bastion, NAT Instance | Production web |

---

## 14. Terraform Examples

```hcl
# ========================================
# Elastic IP cho Bastion Host
# ========================================
resource "aws_eip" "bastion" {
  domain = "vpc"

  tags = {
    Name        = "bastion-eip"
    Environment = "production"
  }

  # Không tự release khi terraform destroy nếu được associate với instance
  lifecycle {
    prevent_destroy = true   # Bảo vệ EIP quan trọng không bị xóa nhầm
  }
}

resource "aws_eip_association" "bastion" {
  instance_id   = aws_instance.bastion.id
  allocation_id = aws_eip.bastion.id
}

# ========================================
# ENI cho High-Availability (Floating IP)
# ========================================
resource "aws_network_interface" "floating" {
  subnet_id       = aws_subnet.private_a.id
  private_ips     = ["10.0.1.100"]
  description     = "Floating ENI for HA failover"
  security_groups = [aws_security_group.app_sg.id]

  # Giữ ENI khi instance bị terminate
  attachment {
    instance     = aws_instance.primary.id
    device_index = 1
  }

  tags = {
    Name = "floating-eni-ha"
  }
}

# EIP gắn vào ENI (không gắn vào instance)
resource "aws_eip" "floating" {
  domain            = "vpc"
  network_interface = aws_network_interface.floating.id

  tags = {
    Name = "floating-eip"
  }
}

# ========================================
# ENI với Secondary Private IPs cho EKS
# ========================================
resource "aws_network_interface" "eks_node" {
  subnet_id   = aws_subnet.private_a.id
  description = "ENI for EKS node with pod IPs"

  # Thêm secondary IPs cho pods
  private_ips_count = 14  # 1 primary + 14 secondary = 15 total

  security_groups = [aws_security_group.eks_node_sg.id]

  tags = {
    Name = "eks-node-eni"
  }
}

# ========================================
# EC2 Instance với Multiple ENIs
# ========================================
resource "aws_instance" "network_appliance" {
  ami           = var.firewall_ami
  instance_type = "c5.xlarge"

  # Primary ENI (management)
  network_interface {
    device_index         = 0
    network_interface_id = aws_network_interface.mgmt.id
  }

  # Secondary ENI (external)
  network_interface {
    device_index         = 1
    network_interface_id = aws_network_interface.external.id
  }

  # Tertiary ENI (internal)
  network_interface {
    device_index         = 2
    network_interface_id = aws_network_interface.internal.id
  }

  tags = { Name = "virtual-firewall" }
}

resource "aws_network_interface" "mgmt" {
  subnet_id       = aws_subnet.management.id
  security_groups = [aws_security_group.mgmt_sg.id]
  description     = "Management interface"
}

resource "aws_network_interface" "external" {
  subnet_id         = aws_subnet.public.id
  security_groups   = [aws_security_group.external_sg.id]
  source_dest_check = false   # Bắt buộc tắt cho network appliance!
  description       = "External (WAN) interface"
}

resource "aws_network_interface" "internal" {
  subnet_id         = aws_subnet.private.id
  security_groups   = [aws_security_group.internal_sg.id]
  source_dest_check = false   # Bắt buộc tắt cho network appliance!
  description       = "Internal (LAN) interface"
}
```

---

## 15. Monitoring và Troubleshooting

```bash
# Kiểm tra tất cả ENI trong VPC
aws ec2 describe-network-interfaces \
  --filters "Name=vpc-id,Values=vpc-0123456789abcdef0" \
  --query 'NetworkInterfaces[*].{
    ID:NetworkInterfaceId,
    Status:Status,
    PrivateIP:PrivateIpAddress,
    PublicIP:Association.PublicIp,
    Instance:Attachment.InstanceId,
    Description:Description
  }' \
  --output table

# Tìm ENI không được gán (Available)
aws ec2 describe-network-interfaces \
  --filters "Name=status,Values=available" \
  --query 'NetworkInterfaces[*].{ID:NetworkInterfaceId,SubnetId:SubnetId,Created:TagSet[?Key==`Name`]|[0].Value}'

# Kiểm tra source/dest check
aws ec2 describe-network-interface-attribute \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --attribute sourceDestCheck

# Tắt source/dest check (cần cho NAT/VPN/router)
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-0a1b2c3d4e5f67890 \
  --no-source-dest-check

# Xem EIP không được gán (tốn tiền vô ích)
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocID:AllocationId}'
```

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Sự khác biệt giữa Elastic IP và Auto-assigned Public IP là gì?

**Trả lời:**
**Auto-assigned Public IP** là địa chỉ IPv4 tạm thời, tự động gán khi instance launch (nếu subnet cấu hình auto-assign public IP). Địa chỉ này **mất đi khi instance stop** — khi start lại, instance nhận địa chỉ khác. Miễn phí nhưng không ổn định.

**Elastic IP** là địa chỉ IPv4 tĩnh, thuộc về AWS account. Không bao giờ thay đổi trừ khi bạn chủ động release. Vẫn tồn tại khi instance stop. Tốn phí khi không được gán ($0.005/giờ). Phù hợp cho bastion host, NAT instance, các server cần IP cố định.

Nguyên tắc: Nếu cần IP ổn định cho DNS A record hoặc whitelist → dùng EIP. Nếu chỉ cần truy cập internet tạm thời → Auto-assigned hoặc dùng ALB/NLB.

---

### Q2: Tại sao phải tắt Source/Destination Check (Kiểm Tra Nguồn/Đích) khi dùng instance làm NAT hoặc router?

**Trả lời:**
Mặc định, mỗi ENI trong AWS thực hiện **Source/Destination Check**: chỉ chấp nhận traffic có IP nguồn hoặc IP đích khớp với private IP của ENI đó. Nếu không khớp, packet bị drop.

Khi instance làm NAT hay router, nó phải **forward traffic** từ source IP khác đến destination IP khác (không phải IP của chính nó). Nếu giữ Source/Dest Check, mọi forwarded packet đều bị drop.

Giải pháp: tắt Source/Dest Check trên ENI của instance NAT/router để cho phép forward packet tùy ý.

```bash
aws ec2 modify-network-interface-attribute \
  --network-interface-id eni-xxxx \
  --no-source-dest-check
```

---

### Q3: ENI và instance có mối quan hệ như thế nào? Điều gì xảy ra với ENI khi instance bị terminate?

**Trả lời:**
**Primary ENI (eth0):** Được tạo cùng với instance, terminate cùng với instance (theo mặc định). Có thể cấu hình để giữ lại primary ENI sau khi terminate nhưng ít được dùng.

**Secondary ENIs:** Được tạo riêng, khi attach có thể cấu hình `deleteOnTermination`:
- `deleteOnTermination=false` (mặc định với manually-attached ENIs): ENI tồn tại sau khi instance terminate, có thể attach lại instance khác
- `deleteOnTermination=true`: ENI bị xóa cùng với instance

Đây là cơ sở cho pattern HA: tạo "floating ENI" với private IP và EIP, attach vào primary instance. Khi primary fail, detach ENI và attach vào standby — IP, MAC, Security Groups, EIP di chuyển cùng.

---

### Q4: Tại sao EKS dùng Secondary Private IPs của ENI cho Pods?

**Trả lời:**
AWS VPC CNI plugin cho EKS sử dụng **Secondary Private IPs** thay vì overlay network (mạng phủ) vì:

1. **Hiệu suất:** Không có encapsulation overhead (VXLAN, IP-in-IP) của overlay. Pods giao tiếp trực tiếp qua native VPC routing.
2. **Khả năng quan sát:** Mỗi Pod có IP thật trong VPC — có thể trace traffic qua VPC Flow Logs, Security Groups áp dụng trực tiếp cho Pod IP.
3. **Tích hợp VPC:** Pods có thể dùng VPC endpoints, truy cập RDS, ElastiCache với Security Groups như EC2 thông thường.
4. **Security Groups for Pods:** Từ 2021, có thể gán Security Group trực tiếp cho Pod (không chỉ cho node).

Giới hạn: số Pods tối đa trên một node phụ thuộc vào số IP tối đa của ENIs (ví dụ m5.large: 3 ENIs × 10 IPs = 29 Pods).

---

### Q5: Khi nào nên dùng ENI failover thay vì EIP failover?

**Trả lời:**
**ENI failover** di chuyển **private IP address** (và kéo theo MAC address, Security Groups) từ instance này sang instance khác trong cùng AZ. Phù hợp khi:
- Cần failover **IP private** (cho internal communication)
- Cần di chuyển cả **Security Groups** theo instance mới
- Dùng **MAC address** làm license key cho phần mềm commercial
- Cần chuyển **cả public lẫn private IP** (gán EIP vào ENI, ENI failover mang cả hai)

**EIP failover** chỉ di chuyển **public IP** (EIP), không ảnh hưởng private IP. Phù hợp khi:
- Chỉ cần thay đổi public IP (external traffic)
- Instances ở các AZ khác nhau (ENI không thể cross-AZ)
- Public IP là điểm quan trọng cần failover

---

### Q6: Enhanced Networking (ENA) cải thiện gì so với driver mạng thông thường?

**Trả lời:**
ENA (Elastic Network Adapter) sử dụng **SR-IOV** (Single Root I/O Virtualization) để bypass hypervisor layer trong data path:

**Không có ENA (emulated NIC):**
```
Guest OS Network Stack → Hypervisor → Physical NIC
(mỗi packet phải qua hypervisor overhead)
```

**Có ENA (SR-IOV):**
```
Guest OS Network Stack → ENA Driver → Physical NIC (trực tiếp)
(bypasses hypervisor, giảm CPU overhead)
```

Kết quả:
- Throughput cao hơn (lên đến 100 Gbps với p4/hpc instances)
- Latency thấp hơn (microseconds thay vì milliseconds)
- CPU utilization thấp hơn cho network processing
- Jitter (biến động latency) thấp hơn — quan trọng cho HPC và real-time apps

---

## Điều Hướng

- [← 3-global-accelerator.md](./3-global-accelerator.md) — Global Accelerator
- [→ 5-ipv6.md](./5-ipv6.md) — IPv6 & Dual-Stack
- [1-vpc-endpoints.md](./1-vpc-endpoints.md) — VPC Endpoints
- [2-privatelink.md](./2-privatelink.md) — AWS PrivateLink
- [README.md](./README.md) — Tổng quan 07-advanced-networking
- [← 06-connectivity](../06-connectivity/) — VPN, Direct Connect, Transit Gateway
