# 1 — VPC Architecture (Kiến Trúc VPC)

> VPC (Virtual Private Cloud — Đám Mây Riêng Ảo) là nền tảng mạng của mọi kiến trúc AWS. Tài liệu này bao gồm thiết kế VPC, CIDR planning (Lập Kế Hoạch Dải IP), Region và Availability Zone.

---

## 📋 Mục Lục

1. [VPC là gì?](#1-vpc-là-gì)
2. [CIDR Notation và IP Planning](#2-cidr-notation-và-ip-planning)
3. [Region và Availability Zone](#3-region-và-availability-zone)
4. [Default VPC vs Custom VPC](#4-default-vpc-vs-custom-vpc)
5. [Thiết Kế VPC Thực Tế](#5-thiết-kế-vpc-thực-tế)
6. [Giới Hạn VPC](#6-giới-hạn-vpc)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. VPC là gì?

**VPC (Virtual Private Cloud — Đám Mây Riêng Ảo)** là một môi trường mạng ảo riêng biệt, được tách biệt về mặt logic khỏi các tài khoản AWS khác. Nó cho phép bạn:

- Kiểm soát hoàn toàn không gian địa chỉ IP (IP address space)
- Tạo Subnets (Mạng Con) trong nhiều Availability Zones
- Cấu hình Route Tables (Bảng Định Tuyến) và Network Gateways
- Áp dụng Security Groups (Nhóm Bảo Mật) và Network ACLs (Danh Sách Kiểm Soát Truy Cập)

### Thành Phần Cốt Lõi của VPC

```
VPC
├── CIDR Block (Dải Địa Chỉ IP)
│   ├── Primary CIDR: bắt buộc, ví dụ 10.0.0.0/16
│   └── Secondary CIDRs: tùy chọn, tối đa 4 CIDRs thêm
│
├── Subnets (Mạng Con)
│   ├── Mỗi Subnet nằm trong đúng một Availability Zone
│   └── Subnet có CIDR con của VPC CIDR
│
├── Route Tables (Bảng Định Tuyến)
│   ├── Main Route Table: mặc định cho mọi Subnet chưa được gán
│   └── Custom Route Tables: gán cho từng Subnet cụ thể
│
├── Internet Gateway — IGW (Cổng Internet)
│   └── Cho phép traffic ra/vào internet
│
├── DHCP Options Set (Tập Tùy Chọn DHCP)
│   └── Cấu hình DNS, NTP cho instances trong VPC
│
└── VPC Endpoints (Điểm Cuối VPC)
    └── Kết nối private đến các AWS services (S3, DynamoDB, v.v.)
```

---

## 2. CIDR Notation và IP Planning

### CIDR (Classless Inter-Domain Routing — Định Tuyến Liên Miền Không Phân Lớp)

CIDR là cách biểu diễn một dải địa chỉ IP. Ký hiệu: `địa_chỉ_IP/prefix_length`

**Prefix length** (độ dài tiền tố) xác định số bit dùng cho phần mạng, các bit còn lại dùng cho hosts.

### Bảng CIDR Nhanh

| CIDR | Số IP | Số Host Dùng Được | Phù Hợp Cho |
|------|-------|-------------------|-------------|
| /16  | 65.536 | 65.531 | VPC toàn bộ |
| /20  | 4.096  | 4.091  | VPC nhỏ / Subnet lớn |
| /24  | 256    | 251    | Subnet thông thường |
| /26  | 64     | 59     | Subnet nhỏ |
| /28  | 16     | 11     | Subnet rất nhỏ (Lambda, v.v.) |

> **Lưu ý AWS:** AWS giữ lại 5 địa chỉ IP trong mỗi Subnet:
> - `.0` — Network address (địa chỉ mạng)
> - `.1` — VPC router
> - `.2` — AWS DNS server
> - `.3` — Dự phòng cho tương lai
> - `.255` — Broadcast address (không dùng trong VPC nhưng vẫn bị giữ lại)

### Quy Tắc CIDR cho VPC

- **Kích thước VPC CIDR:** tối thiểu `/28` (16 IPs), tối đa `/16` (65.536 IPs)
- **Dải IP được hỗ trợ:** 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 (RFC 1918 — private IP ranges)
- **Không thể thay đổi** Primary CIDR sau khi tạo VPC
- **Có thể thêm** Secondary CIDRs (tối đa 5 CIDRs tổng cộng)

### CIDR Planning Thực Tế

**Chiến lược phân bổ IP theo môi trường:**

```
Production:   10.0.0.0/16   (65.536 IPs)
Staging:      10.1.0.0/16   (65.536 IPs)
Development:  10.2.0.0/16   (65.536 IPs)
```

**Tại sao không dùng dải IP trùng giữa các VPC?**

Nếu bạn sau này muốn kết nối các VPC qua VPC Peering (Kết Nối Ngang Hàng) hoặc Transit Gateway (Cổng Trung Chuyển), các CIDR blocks **không được trùng nhau**. Lập kế hoạch ngay từ đầu giúp tránh phải thiết kế lại sau này.

### Ví Dụ CIDR Cho Production VPC

```
VPC: 10.0.0.0/16
│
├── AZ-A (ap-southeast-1a)
│   ├── Public Subnet:    10.0.1.0/24   (251 hosts)
│   ├── Private Subnet:   10.0.11.0/24  (251 hosts)
│   └── Database Subnet:  10.0.21.0/24  (251 hosts)
│
├── AZ-B (ap-southeast-1b)
│   ├── Public Subnet:    10.0.2.0/24
│   ├── Private Subnet:   10.0.12.0/24
│   └── Database Subnet:  10.0.22.0/24
│
└── AZ-C (ap-southeast-1c)
    ├── Public Subnet:    10.0.3.0/24
    ├── Private Subnet:   10.0.13.0/24
    └── Database Subnet:  10.0.23.0/24
```

**Nguyên tắc đặt số Subnet:**
- Public: `10.0.X.0/24` với X = 1, 2, 3 (AZ A, B, C)
- Private: `10.0.1X.0/24` với X = 1, 2, 3
- Database/Isolated: `10.0.2X.0/24` với X = 1, 2, 3

Cách này giúp nhìn vào IP là biết ngay Subnet thuộc loại nào và AZ nào.

---

## 3. Region và Availability Zone

### AWS Region (Vùng AWS)

**Region** là một khu vực địa lý bao gồm nhiều Availability Zones. Mỗi Region hoàn toàn độc lập với các Region khác — tài nguyên không tự động replicate (nhân bản) giữa các Regions.

**VPC được tạo trong một Region cụ thể** và có thể trải rộng qua tất cả AZs trong Region đó.

### Availability Zone — AZ (Vùng Khả Dụng)

**AZ** là một hoặc nhiều trung tâm dữ liệu vật lý riêng biệt trong một Region, có:
- Nguồn điện độc lập
- Hệ thống làm lạnh độc lập
- Kết nối mạng riêng biệt
- Kết nối fiber (cáp quang) tốc độ cao, độ trễ thấp với các AZs khác trong cùng Region

```
Region: ap-southeast-1 (Singapore)
├── AZ: ap-southeast-1a  ──┐
├── AZ: ap-southeast-1b  ──┼── Kết nối nội bộ tốc độ cao, độ trễ < 1ms
└── AZ: ap-southeast-1c  ──┘
```

### Multi-AZ Design (Thiết Kế Đa Vùng Khả Dụng)

**Tại sao cần Multi-AZ?**

- Nếu một AZ bị sự cố (mất điện, thiên tai, v.v.), traffic tự động chuyển sang AZ khác
- Là yêu cầu bắt buộc cho mọi production system
- Không thêm chi phí đáng kể, nhưng tăng độ tin cậy lên đáng kể

**Quy tắc thiết kế:** Mỗi tầng (tier) của ứng dụng nên có ít nhất 2 AZs.

```
Single AZ (KHÔNG NÊN)        Multi-AZ (NÊN LÀM)
─────────────────────        ─────────────────────────────
AZ-A only:                   AZ-A:           AZ-B:
  Load Balancer    ──────►     Load Balancer   Load Balancer
  App Server                   App Server      App Server
  Database                     (Primary DB)    (Standby DB)
```

---

## 4. Default VPC vs Custom VPC

### Default VPC (VPC Mặc Định)

AWS tự động tạo một Default VPC trong mỗi Region cho mỗi tài khoản. Đặc điểm:

| Thuộc Tính | Giá Trị |
|-----------|---------|
| CIDR Block | 172.31.0.0/16 |
| Subnets | Một Public Subnet mỗi AZ |
| Internet Gateway | Đã được gắn sẵn |
| Route Table | Route mặc định ra internet |
| DNS Hostnames | Được bật |

**Vấn đề với Default VPC trong Production:**
- Tất cả instances mặc định có Public IP — không an toàn
- Không thể kiểm soát dải IP (có thể trùng với on-premises network)
- Không có Private Subnets — không phù hợp cho Database
- Mọi người trong team đều có thể vô tình deploy vào đây

### Custom VPC (VPC Tùy Chỉnh)

Luôn tạo Custom VPC cho môi trường production. Lợi ích:

- Kiểm soát hoàn toàn dải IP — tránh trùng với on-premises
- Tách biệt Public/Private/Isolated Subnets theo thiết kế
- Security mặc định mạnh hơn (không có Internet Gateway cho đến khi bạn chủ động thêm)
- Có thể đặt tên và tag có ý nghĩa

**Khi nào dùng Default VPC?**
- Thử nghiệm nhanh (prototyping)
- Lab learning
- **Không bao giờ** dùng cho production workload

---

## 5. Thiết Kế VPC Thực Tế

### Mẫu Thiết Kế 3-Tier (3 Lớp)

Đây là mẫu phổ biến nhất cho web applications:

```
Internet
    │
    ▼
Internet Gateway (IGW)
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ PUBLIC SUBNET (10.0.1.0/24, 10.0.2.0/24)           │
│   • Application Load Balancer (ALB)                 │
│   • NAT Gateway                                     │
│   • Bastion Host (nếu cần SSH vào instances)        │
└───────────────────┬─────────────────────────────────┘
                    │ Internal traffic only
                    ▼
┌─────────────────────────────────────────────────────┐
│ PRIVATE SUBNET (10.0.11.0/24, 10.0.12.0/24)        │
│   • EC2 Application Servers                         │
│   • ECS Tasks (Container workloads)                 │
│   • Lambda Functions (trong VPC)                    │
└───────────────────┬─────────────────────────────────┘
                    │ Database connections only
                    ▼
┌─────────────────────────────────────────────────────┐
│ ISOLATED/DATABASE SUBNET (10.0.21.0/24, 10.0.22.0) │
│   • RDS (Relational Database Service)               │
│   • ElastiCache (Redis, Memcached)                  │
│   • Amazon DocumentDB                               │
└─────────────────────────────────────────────────────┘
```

### Checklist Thiết Kế VPC

```
□ Chọn CIDR phù hợp, không trùng với các môi trường khác
□ Tạo Subnets trong ít nhất 2 AZs
□ Phân chia 3 loại Subnet: Public / Private / Isolated
□ Đặt tên rõ ràng (ví dụ: prod-public-1a, prod-private-1b)
□ Tag tất cả tài nguyên với Environment, Project, Owner
□ Bật VPC Flow Logs từ đầu để audit
□ Không dùng Default VPC cho workloads thực
□ Ghi lại CIDR allocation trong documentation
```

---

## 6. Giới Hạn VPC

| Giới Hạn | Mặc Định | Có Thể Tăng? |
|---------|---------|-------------|
| VPCs per Region | 5 | Có |
| Subnets per VPC | 200 | Có |
| Route Tables per VPC | 200 | Có |
| Security Groups per VPC | 2.500 | Có |
| CIDRs per VPC | 5 | Không |
| IGWs per Region | 5 | Có |

> Để tăng giới hạn, gửi yêu cầu qua AWS Service Quotas console.

---

## 7. Câu Hỏi Phỏng Vấn

### Q1: Giải thích VPC là gì và tại sao cần dùng?

**Trả lời mẫu:**

> VPC — Virtual Private Cloud — là môi trường mạng ảo riêng biệt trong AWS, giúp bạn kiểm soát hoàn toàn cấu hình mạng: địa chỉ IP, Subnets, Route Tables, và cách traffic vào ra. Tôi cần VPC vì nó cô lập workloads khỏi các tài khoản khác, cho phép thiết kế kiến trúc mạng phù hợp với yêu cầu bảo mật và compliance, và tích hợp được với môi trường on-premises qua VPN hoặc Direct Connect.

### Q2: CIDR /16 vs /24 — cái nào to hơn?

**Trả lời:**

> `/16` to hơn `/24`. CIDR `/16` có 2^(32-16) = 65.536 địa chỉ IP, còn `/24` có 2^(32-24) = 256 địa chỉ IP. Prefix length (số sau dấu /) càng nhỏ thì dải IP càng lớn. Tôi thường dùng `/16` cho VPC tổng thể và `/24` cho từng Subnet.

### Q3: Tại sao không nên dùng Default VPC cho production?

**Trả lời:**

> Default VPC có một số vấn đề: (1) Tất cả Subnets đều là Public — instances tự động có Public IP, tăng attack surface; (2) Không có Private/Isolated Subnets cho Database; (3) CIDR cố định 172.31.0.0/16, có thể xung đột với on-premises network khi muốn kết nối; (4) Không thể audit rõ ai đã deploy gì ở đó. Production cần Custom VPC với network segmentation (phân đoạn mạng) rõ ràng.

### Q4: Làm sao chọn kích thước CIDR cho VPC?

**Trả lời:**

> Tôi tính toán dựa trên: (1) Số lượng instances tối đa dự kiến; (2) Số AZs; (3) Số tầng (tiers) trong kiến trúc; (4) Kế hoạch tăng trưởng 3-5 năm tới. Thông thường `/16` cho production VPC là an toàn — 65.000 IPs đủ cho hầu hết workloads. Quan trọng hơn là phải chọn dải IP không trùng với các VPC khác hoặc on-premises, để sau này kết nối được.

---

## 📝 Tóm Tắt

| Khái Niệm | Định Nghĩa |
|-----------|-----------|
| VPC | Môi trường mạng ảo riêng biệt trong AWS |
| CIDR | Ký hiệu xác định dải địa chỉ IP |
| Region | Khu vực địa lý chứa nhiều AZs |
| Availability Zone | Trung tâm dữ liệu độc lập trong một Region |
| Default VPC | VPC mặc định AWS tạo sẵn — không dùng cho production |
| Custom VPC | VPC do bạn tạo và kiểm soát hoàn toàn |

---

## 🔗 Điều Hướng

- ← [README — Tổng Quan Section](./README.md)
- → [2. Subnets](./2-subnets.md)
- [Chỉ Mục Đầy Đủ](../INDEX.md)
