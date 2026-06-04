# 2 — Subnets (Mạng Con)

> Subnet (Mạng Con) là phân đoạn của một VPC nằm trong một Availability Zone cụ thể. Thiết kế Subnets đúng cách là yếu tố quyết định bảo mật và tính sẵn sàng của hệ thống.

---

## 📋 Mục Lục

1. [Subnet là gì?](#1-subnet-là-gì)
2. [Ba Loại Subnet Chính](#2-ba-loại-subnet-chính)
3. [Public Subnet — Chi Tiết](#3-public-subnet--chi-tiết)
4. [Private Subnet — Chi Tiết](#4-private-subnet--chi-tiết)
5. [Isolated Subnet — Chi Tiết](#5-isolated-subnet--chi-tiết)
6. [Thiết Kế Multi-AZ Subnet](#6-thiết-kế-multi-az-subnet)
7. [Auto-assign Public IP](#7-auto-assign-public-ip)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Subnet là gì?

**Subnet (Mạng Con)** là phân đoạn nhỏ hơn trong một VPC với CIDR block riêng. Mỗi Subnet:

- Nằm trong **đúng một** Availability Zone (không thể trải qua nhiều AZs)
- Có CIDR là tập con của VPC CIDR
- Được liên kết với một Route Table
- Có thể bật hoặc tắt tính năng tự động gán Public IP cho instances

```
VPC: 10.0.0.0/16
├── Subnet A (AZ-a): 10.0.1.0/24   ── 256 IPs trong AZ ap-southeast-1a
├── Subnet B (AZ-b): 10.0.2.0/24   ── 256 IPs trong AZ ap-southeast-1b
└── Subnet C (AZ-a): 10.0.3.0/24   ── 256 IPs trong AZ ap-southeast-1a
```

**Lưu ý:** Nhiều Subnets có thể nằm trong cùng một AZ, nhưng một Subnet chỉ nằm trong một AZ duy nhất.

---

## 2. Ba Loại Subnet Chính

Về mặt kỹ thuật, AWS không có khái niệm "loại Subnet" — tất cả đều là Subnet thông thường. Sự khác biệt nằm ở **cấu hình Route Table** liên kết với Subnet đó.

| Loại | Route Table | Internet? | Dùng Cho |
|------|------------|-----------|---------|
| **Public Subnet** | Có route `0.0.0.0/0 → IGW` | Vào/Ra | ALB, NAT Gateway, Bastion |
| **Private Subnet** | Có route `0.0.0.0/0 → NAT Gateway` | Ra (không vào) | App servers, ECS, Lambda |
| **Isolated Subnet** | Không có route ra ngoài VPC | Không | Database, Cache, Secrets |

---

## 3. Public Subnet — Chi Tiết

### Đặc Điểm

- Route Table có entry: `0.0.0.0/0 → Internet Gateway (IGW)`
- Instances **có thể** nhận Public IP hoặc Elastic IP (IP Đàn Hồi)
- Traffic từ internet có thể vào trực tiếp (nếu Security Group cho phép)

### Dùng Cho

```
Public Subnet chứa:
├── Application Load Balancer (ALB) — nhận traffic HTTP/HTTPS từ internet
├── NAT Gateway — cho Private Subnet ra internet
├── Bastion Host (Jump Box) — SSH proxy để vào Private instances
├── Elastic IP (IP Đàn Hồi) — IP cố định công khai
└── Network Load Balancer (NLB) — nếu cần Layer 4 load balancing
```

### Cấu Hình Route Table

```
Public Route Table:
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │  ← Traffic nội bộ VPC
│ 0.0.0.0/0           │ igw-xxxxxxxx        │  ← Traffic ra internet
└─────────────────────┴─────────────────────┘
```

### Bảo Mật Public Subnet

Mặc dù có thể truy cập từ internet, Public Subnet **không phải là không bảo mật** nếu cấu hình đúng:

- **Security Groups** kiểm soát port nào được phép (ví dụ: chỉ port 80, 443 cho ALB)
- **Network ACLs** là lớp bảo vệ thứ hai ở mức Subnet
- **Không để Database** trong Public Subnet dù Security Group có restrict

---

## 4. Private Subnet — Chi Tiết

### Đặc Điểm

- Route Table có entry: `0.0.0.0/0 → NAT Gateway`
- NAT Gateway nằm trong **Public Subnet** cùng AZ
- Instances **không có** Public IP — không thể truy cập trực tiếp từ internet
- Vẫn ra được internet (để download packages, gọi API ngoài) qua NAT Gateway

### Dùng Cho

```
Private Subnet chứa:
├── EC2 Application Servers — web app, API servers
├── ECS (Elastic Container Service) Tasks — containers
├── Lambda Functions (trong VPC)
├── Elasticsearch / OpenSearch domains
└── Internal ALB (Application Load Balancer nội bộ)
```

### Cấu Hình Route Table

```
Private Route Table (AZ-A):
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │  ← Traffic nội bộ VPC
│ 0.0.0.0/0           │ nat-xxxxxxxx        │  ← Ra internet qua NAT
└─────────────────────┴─────────────────────┘
```

### Tại Sao Cần NAT Gateway ở Public Subnet?

NAT Gateway cần một Public IP để thực hiện địa chỉ dịch (Network Address Translation). Do đó, NAT Gateway phải đặt ở Public Subnet (có route ra IGW). Instances trong Private Subnet gửi traffic đến NAT Gateway, NAT Gateway thay thế IP nguồn bằng IP của mình rồi gửi ra internet.

```
Private Instance (10.0.11.5) 
    → NAT Gateway (Public IP: 54.1.2.3)
        → Internet Server
            → Phản hồi về NAT Gateway
                → NAT Gateway chuyển về Private Instance
```

---

## 5. Isolated Subnet — Chi Tiết

### Đặc Điểm

- Route Table **không có** route `0.0.0.0/0` — tức là không ra được internet theo bất kỳ hướng nào
- Chỉ có traffic nội bộ VPC (`local` route)
- Bảo mật cao nhất trong ba loại
- Một số tài liệu AWS gọi là "Database Subnet" hoặc "Intranet Subnet"

### Dùng Cho

```
Isolated Subnet chứa:
├── RDS (Relational Database Service) — MySQL, PostgreSQL, Oracle
├── Amazon Aurora clusters
├── ElastiCache — Redis, Memcached
├── Amazon DocumentDB
├── Amazon Keyspaces (Cassandra-compatible)
└── AWS Secrets Manager Endpoints (tùy thiết kế)
```

### Cấu Hình Route Table

```
Isolated Route Table:
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │  ← Chỉ traffic nội bộ VPC
└─────────────────────┴─────────────────────┘
```

### Truy Cập AWS Services từ Isolated Subnet

Vì không có internet, để Database trong Isolated Subnet gọi được AWS services (S3 để backup, CloudWatch để monitoring), bạn cần dùng **VPC Endpoints** (Điểm Cuối VPC):

- **Gateway Endpoint** cho S3 và DynamoDB — miễn phí
- **Interface Endpoint** (PrivateLink) cho hầu hết các AWS services khác — có phí

---

## 6. Thiết Kế Multi-AZ Subnet

### Tại Sao Cần Multi-AZ?

Nếu chỉ có Subnets trong một AZ và AZ đó gặp sự cố, toàn bộ hệ thống sẽ ngừng hoạt động. Multi-AZ đảm bảo High Availability (Tính Sẵn Sàng Cao).

### Mẫu Chuẩn Cho Production (3 AZs)

```
VPC: 10.0.0.0/16
│
├── AZ-A (ap-southeast-1a)
│   ├── public-1a:    10.0.1.0/24    → Route: 0.0.0.0/0 → IGW
│   ├── private-1a:   10.0.11.0/24   → Route: 0.0.0.0/0 → nat-1a
│   └── database-1a:  10.0.21.0/24   → Route: local only
│
├── AZ-B (ap-southeast-1b)
│   ├── public-1b:    10.0.2.0/24
│   ├── private-1b:   10.0.12.0/24   → Route: 0.0.0.0/0 → nat-1b
│   └── database-1b:  10.0.22.0/24
│
└── AZ-C (ap-southeast-1c)
    ├── public-1c:    10.0.3.0/24
    ├── private-1c:   10.0.13.0/24   → Route: 0.0.0.0/0 → nat-1c
    └── database-1c:  10.0.23.0/24
```

**Lưu ý:** Mỗi Private Subnet cần một NAT Gateway riêng trong cùng AZ để đảm bảo HA. Nếu dùng chung một NAT Gateway cho tất cả AZs, khi AZ chứa NAT Gateway gặp sự cố, Private Subnets ở AZ khác cũng mất internet.

### Tối Ưu Chi Phí (Non-Production)

Với môi trường dev/staging, bạn có thể dùng **một NAT Gateway duy nhất** để giảm chi phí, chấp nhận single point of failure:

```
Development VPC:
├── AZ-A: public-1a (có NAT Gateway duy nhất)
├── AZ-A: private-1a → nat-1a (trong public-1a)
└── AZ-B: private-1b → nat-1a (CÙNG NAT Gateway! Rủi ro nhưng tiết kiệm chi phí)
```

---

## 7. Auto-assign Public IP

**Auto-assign Public IP** — Tự Động Gán IP Công Cộng là cài đặt ở mức Subnet, quyết định instances mới trong Subnet có tự động nhận Public IP hay không.

| Subnet Type | Cài Đặt Khuyến Nghị |
|------------|---------------------|
| Public Subnet | **Bật** — instances cần Public IP để hoạt động |
| Private Subnet | **Tắt** — instances không nên có Public IP |
| Isolated Subnet | **Tắt** — tuyệt đối không có Public IP |

**Lưu ý:** Bật Auto-assign Public IP không đồng nghĩa với có thể truy cập từ internet — Security Group vẫn kiểm soát điều này. Nhưng theo Principle of Least Privilege (Nguyên Tắc Đặc Quyền Tối Thiểu), không nên cấp Public IP nếu không cần thiết.

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: Sự khác biệt giữa Public và Private Subnet là gì?

**Trả lời mẫu:**

> Sự khác biệt chính nằm ở Route Table. Public Subnet có route `0.0.0.0/0 → Internet Gateway`, cho phép traffic ra/vào internet trực tiếp. Private Subnet có route `0.0.0.0/0 → NAT Gateway` — instances chỉ ra được internet nhưng không nhận được traffic từ ngoài vào. Isolated Subnet không có route nào ra ngoài VPC. Tôi đặt Load Balancer ở Public, Application Server ở Private, và Database ở Isolated.

### Q2: Tại sao NAT Gateway phải đặt ở Public Subnet?

**Trả lời:**

> NAT Gateway cần một Elastic IP (IP công khai cố định) để thực hiện Network Address Translation — dịch địa chỉ IP riêng của instances sang IP công khai khi ra internet. Để có IP công khai và route ra internet, NAT Gateway phải nằm trong Public Subnet (có kết nối với Internet Gateway). Nếu đặt NAT Gateway ở Private Subnet, nó không có đường ra internet để forward traffic.

### Q3: Có thể có nhiều Subnets trong cùng một AZ không?

**Trả lời:**

> Có, và đây là cách thiết kế bình thường. Trong mỗi AZ thường có ít nhất 3 Subnets: Public, Private, và Isolated. Mỗi Subnet có CIDR riêng không trùng nhau. Ví dụ trong AZ-A: `10.0.1.0/24` (Public), `10.0.11.0/24` (Private), `10.0.21.0/24` (Isolated).

### Q4: Tại sao nên có NAT Gateway riêng mỗi AZ?

**Trả lời:**

> Để đảm bảo High Availability. Nếu dùng chung một NAT Gateway và AZ chứa NAT Gateway đó gặp sự cố, tất cả Private Subnets ở các AZ khác cũng mất khả năng ra internet, dù bản thân các AZ kia vẫn hoạt động bình thường. Với production, tôi luôn tạo NAT Gateway riêng trong Public Subnet của mỗi AZ và cấu hình Private Route Table của từng AZ chỉ đến NAT Gateway trong AZ đó.

---

## 📝 Tóm Tắt

| Loại Subnet | Route ra Internet | Public IP | Dùng Cho |
|------------|-------------------|-----------|---------|
| Public | Qua Internet Gateway | Có (tùy chọn) | ALB, NAT GW, Bastion |
| Private | Qua NAT Gateway | Không | App servers, ECS |
| Isolated | Không có | Không | DB, Cache |

---

## 🔗 Điều Hướng

- ← [1. VPC Architecture](./1-vpc-architecture.md)
- → [3. Route Tables & Internet Gateway](./3-route-tables.md)
- [Chỉ Mục Đầy Đủ](../INDEX.md)
