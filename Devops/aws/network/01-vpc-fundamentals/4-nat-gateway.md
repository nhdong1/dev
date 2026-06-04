# 4 — NAT Gateway vs NAT Instance (Cổng NAT và Máy Chủ NAT)

> NAT (Network Address Translation — Dịch Địa Chỉ Mạng) cho phép instances trong Private Subnet khởi tạo kết nối ra internet trong khi ngăn internet khởi tạo kết nối vào. Tài liệu này so sánh NAT Gateway (dịch vụ managed của AWS) và NAT Instance (tự quản lý trên EC2).

---

## 📋 Mục Lục

1. [Tại Sao Cần NAT?](#1-tại-sao-cần-nat)
2. [NAT Gateway — Chi Tiết](#2-nat-gateway--chi-tiết)
3. [NAT Instance — Chi Tiết](#3-nat-instance--chi-tiết)
4. [So Sánh NAT Gateway vs NAT Instance](#4-so-sánh-nat-gateway-vs-nat-instance)
5. [Public NAT vs Private NAT](#5-public-nat-vs-private-nat)
6. [Thiết Kế Multi-AZ NAT](#6-thiết-kế-multi-az-nat)
7. [Chi Phí NAT Gateway](#7-chi-phí-nat-gateway)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Tại Sao Cần NAT?

Instances trong Private Subnet không có Public IP và không có route trực tiếp ra internet. Nhưng chúng vẫn cần ra internet cho nhiều tác vụ:

- **Tải packages/dependencies:** `yum update`, `apt-get install`, `npm install`
- **Gọi external APIs:** Payment gateway, SMS service, third-party APIs
- **Kết nối AWS services** (nếu không dùng VPC Endpoints): S3, CloudWatch, ECR
- **Kiểm tra license:** Một số phần mềm cần gọi về server để verify license

**Giải pháp:** Dùng NAT (Network Address Translation) để "đại diện" cho Private Instances khi ra internet.

```
Private Instance (10.0.11.5)           Internet
│                                         │
│ Muốn kết nối đến 8.8.8.8               │
│                                         │
▼                                         │
NAT Gateway (Public IP: 13.214.1.100)     │
│                                         │
│ Đổi IP nguồn: 10.0.11.5 → 13.214.1.100 │
│────────────────────────────────────────►│
│                                         │
│◄────────────────────────────────────────│
│ Nhận phản hồi, forward về 10.0.11.5     │
```

---

## 2. NAT Gateway — Chi Tiết

### NAT Gateway là gì?

**NAT Gateway** là dịch vụ managed (được quản lý) của AWS thực hiện Network Address Translation. AWS chịu trách nhiệm vận hành, bảo trì, và đảm bảo tính sẵn sàng.

### Đặc Điểm Kỹ Thuật

| Thuộc Tính | Giá Trị |
|-----------|---------|
| Bandwidth tối đa | 100 Gbps |
| Concurrent connections (Kết nối đồng thời) | 55.000 per destination IP:Port |
| Độ sẵn sàng | Managed HA trong một AZ |
| Protocols hỗ trợ | TCP, UDP, ICMP |
| Ports hỗ trợ | 1024–65535 (ephemeral ports) |
| IPv6 | Egress-only Internet Gateway cho IPv6 (khác với NAT) |

### Bước Triển Khai NAT Gateway

```
1. Tạo Elastic IP (IP Đàn Hồi) trong Region
   → EC2 → Elastic IPs → Allocate Elastic IP address

2. Tạo NAT Gateway trong PUBLIC Subnet
   → VPC → NAT Gateways → Create NAT Gateway
   → Chọn Public Subnet, gán Elastic IP vừa tạo

3. Cập nhật Route Table của Private Subnet
   → VPC → Route Tables → Edit routes
   → Thêm: Destination: 0.0.0.0/0, Target: nat-xxxxxxxxx
```

### Managed Service Advantages (Lợi Ích Dịch Vụ Managed)

- **Không cần patch (vá lỗi):** AWS tự cập nhật security patches
- **Không cần scale:** Tự động xử lý traffic tăng đột biến
- **Không cần monitor instance health:** AWS đảm bảo availability
- **Không single point of failure trong AZ:** Được built với redundancy

---

## 3. NAT Instance — Chi Tiết

### NAT Instance là gì?

**NAT Instance** là EC2 instance được cấu hình để thực hiện NAT. Đây là giải pháp cũ hơn, trước khi NAT Gateway ra đời. Bạn phải tự quản lý instance này.

### Cấu Hình NAT Instance

```bash
# 1. Tắt Source/Destination Check trên EC2 instance
# (EC2 mặc định drop packets không phải từ/đến chính nó)
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxxxxxx \
  --no-source-dest-check

# 2. Bật IP forwarding trong OS
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# 3. Cấu hình iptables để NAT
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### AMI (Amazon Machine Image — Ảnh Máy Ảo) Dùng Cho NAT Instance

AWS cung cấp sẵn AMI: `amzn-ami-vpc-nat-*` — Amazon Linux với cấu hình NAT sẵn. Tuy nhiên, AMI này không còn được cập nhật thường xuyên.

---

## 4. So Sánh NAT Gateway vs NAT Instance

### Bảng So Sánh Đầy Đủ

| Tiêu Chí | NAT Gateway | NAT Instance |
|---------|-------------|-------------|
| **Quản lý** | AWS managed (không cần quản lý) | Tự quản lý (patch, monitor) |
| **Tính sẵn sàng** | HA trong một AZ, tự động | Phải tự cấu hình (Auto Scaling Group) |
| **Bandwidth** | Đến 100 Gbps tự động scale | Phụ thuộc loại EC2 instance |
| **Security Groups** | Không áp dụng được | Có thể gán Security Group |
| **Bastion Host** | Không thể dùng làm Bastion | Có thể dùng làm Bastion + NAT |
| **Chi phí** | Cao hơn ($0.045/giờ + data transfer) | Thấp hơn (chỉ trả tiền EC2) |
| **Port Forwarding** | Không hỗ trợ | Hỗ trợ qua iptables |
| **Protocol** | TCP, UDP, ICMP | Mọi protocol (linh hoạt hơn) |
| **Source/Dest Check** | Không cần tắt | Phải tắt thủ công |
| **Logging** | Không có built-in flow logs riêng | Có thể log thủ công |
| **Khuyến nghị** | **Production** | Lab/Dev cần tiết kiệm chi phí |

### Quyết Định Nhanh

```
Bạn cần gì?
│
├── Production workload → NAT Gateway (luôn chọn này)
│
├── Tiết kiệm chi phí tuyệt đối cho dev/lab → NAT Instance (t3.nano ~$4/tháng)
│
├── Cần Port Forwarding từ internet vào Private Instance → NAT Instance
│
└── Cần Bastion Host + NAT kết hợp → NAT Instance (nhưng không khuyến nghị)
```

---

## 5. Public NAT vs Private NAT

AWS NAT Gateway có hai loại:

### Public NAT Gateway (Mặc Định — Phổ Biến)

- Đặt trong **Public Subnet**
- Có **Elastic IP** (IP công khai)
- Instances trong Private Subnet → NAT GW → Internet qua IGW

```
Private Instance → [Private Route Table] → NAT GW (Public Subnet) → IGW → Internet
```

### Private NAT Gateway

- Đặt trong **Private Subnet**
- **Không có** Elastic IP
- Dùng để kết nối giữa hai VPCs hoặc on-premises, **không phải** ra internet
- Traffic đi qua Transit Gateway hoặc VGW (Virtual Private Gateway)

```
VPC A (Private Instance) → NAT GW (Private) → Transit Gateway → VPC B
```

**Use case (Trường Hợp Sử Dụng) Private NAT Gateway:**
- Kết nối hai VPC có overlapping CIDRs (CIDR trùng nhau) — NAT che giấu IP thực
- Kết nối qua Transit Gateway mà muốn ẩn IP nguồn

---

## 6. Thiết Kế Multi-AZ NAT

### Vấn Đề Single NAT Gateway

```
❌ KHÔNG NÊN (Single NAT Gateway):

AZ-A:
  private-1a → [Route: 0.0.0.0/0 → nat-1a] → nat-1a (Public Subnet-A) → IGW

AZ-B:
  private-1b → [Route: 0.0.0.0/0 → nat-1a] → ← CROSS-AZ TRAFFIC!
                                                   (phí cross-AZ data transfer)
                                                   (AZ-A fail → cả 2 AZ mất internet)
```

### Giải Pháp: NAT Gateway Per-AZ

```
✅ NÊN LÀM (NAT Gateway per AZ):

AZ-A:
  public-1a: NAT GW A (Elastic IP: 54.1.1.1)
  private-1a → [Route: 0.0.0.0/0 → nat-A] → nat-A → IGW

AZ-B:
  public-1b: NAT GW B (Elastic IP: 54.1.1.2)
  private-1b → [Route: 0.0.0.0/0 → nat-B] → nat-B → IGW
```

**Kết quả:**
- Không có cross-AZ traffic (tiết kiệm chi phí data transfer)
- Khi AZ-A fail: AZ-B vẫn ra internet bình thường
- Khi AZ-B fail: AZ-A vẫn ra internet bình thường

---

## 7. Chi Phí NAT Gateway

Chi phí NAT Gateway gồm hai phần:

| Loại Chi Phí | Giá (ap-southeast-1) |
|-------------|---------------------|
| **Hourly charge** (Phí theo giờ) | ~$0.045/giờ = ~$32.4/tháng mỗi NAT GW |
| **Data processing** (Phí xử lý data) | ~$0.045/GB |

**Ví dụ chi phí:**
- 3 AZs, mỗi AZ một NAT Gateway: 3 × $32.4 = **~$97/tháng** chỉ cho NAT Gateway
- Thêm data processing: nếu 100GB/tháng × $0.045 = $4.5

**Giảm chi phí NAT Gateway:**
- Dùng **VPC Endpoints** cho các AWS services (S3, DynamoDB, ECR, CloudWatch) — traffic không qua NAT Gateway
- Với dev/staging: có thể dùng 1 NAT Gateway, chấp nhận rủi ro single point of failure
- Dùng **NAT Instance** (t3.nano ~$4/tháng) cho môi trường không quan trọng

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: Tại sao nên dùng NAT Gateway thay vì NAT Instance?

**Trả lời mẫu:**

> NAT Gateway là managed service nên tôi không phải lo về patching, scaling, hay monitoring instance health. Nó tự động scale đến 100 Gbps và có high availability trong AZ. NAT Instance đòi hỏi tôi phải tắt Source/Destination Check, cấu hình iptables, tự setup Auto Scaling nếu muốn HA, và chịu trách nhiệm patch OS. Trong production, thời gian vận hành quan trọng hơn tiết kiệm vài chục đô mỗi tháng. Tuy nhiên với dev/lab, NAT Instance trên t3.nano có thể tiết kiệm chi phí đáng kể.

### Q2: NAT Gateway phải đặt ở Public hay Private Subnet?

**Trả lời:**

> Public NAT Gateway — loại phổ biến nhất — phải đặt ở **Public Subnet** vì nó cần Elastic IP (Public IP) để thực hiện NAT ra internet, và Public Subnet mới có route đến Internet Gateway. Nếu đặt ở Private Subnet thì NAT Gateway không có đường ra internet và không hoạt động. Còn Private NAT Gateway thì đặt ở Private Subnet nhưng nó không dùng để ra internet mà dùng để kết nối giữa các VPC.

### Q3: Làm sao giảm chi phí NAT Gateway?

**Trả lời:**

> Có ba cách chính: (1) Dùng VPC Endpoints cho các AWS services như S3, DynamoDB, ECR, CloudWatch Logs — traffic này sẽ không đi qua NAT Gateway nên không mất phí data processing; (2) Kiểm tra xem ứng dụng có transfer lượng lớn data qua NAT không, nếu có thì tối ưu code để giảm; (3) Với môi trường dev/staging, dùng 1 NAT Gateway thay vì 1 per AZ. Trong dự án trước, tôi đã giảm chi phí NAT ~40% chỉ bằng cách thêm S3 và ECR VPC Endpoints.

### Q4: Điều gì xảy ra nếu NAT Gateway bị fail?

**Trả lời:**

> NAT Gateway được AWS quản lý và có redundancy nội bộ trong AZ, nên fail tự phát là rất hiếm. Tuy nhiên, nếu toàn bộ AZ gặp sự cố, NAT Gateway trong AZ đó cũng không hoạt động được. Vì vậy, với production, tôi luôn tạo NAT Gateway riêng trong mỗi AZ và cấu hình Route Table Private Subnet của từng AZ chỉ đến NAT Gateway trong AZ đó. Khi AZ bị sự cố, chỉ Subnets trong AZ đó bị ảnh hưởng, AZ khác hoạt động bình thường.

---

## 📝 Tóm Tắt

| Tiêu Chí | NAT Gateway | NAT Instance |
|---------|-------------|-------------|
| Quản lý | AWS managed | Tự quản lý |
| HA | Có (per AZ) | Cần tự cấu hình |
| Bandwidth | Đến 100 Gbps | Phụ thuộc EC2 type |
| Chi phí | ~$32/tháng/AZ | ~$4/tháng (t3.nano) |
| Khuyến nghị | Production | Dev/Lab/Cost saving |

---

## 🔗 Điều Hướng

- ← [3. Route Tables & Internet Gateway](./3-route-tables.md)
- → [5. VPC Peering](./5-vpc-peering.md)
- [Chỉ Mục Đầy Đủ](../INDEX.md)
