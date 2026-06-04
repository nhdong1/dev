# Security Groups — Tường Lửa Trạng Thái AWS

> Security Group (SG — Nhóm Bảo Mật) là tường lửa ảo **stateful** (có trạng thái) hoạt động ở cấp độ ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi). Đây là lớp bảo vệ đầu tiên và quan trọng nhất cho mọi resource trong VPC.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#1-khái-niệm-cơ-bản)
2. [Cách Hoạt Động Stateful](#2-cách-hoạt-động-stateful)
3. [Cấu Trúc Rules](#3-cấu-trúc-rules)
4. [Tham Chiếu Security Group Khác](#4-tham-chiếu-security-group-khác)
5. [Giới Hạn & Quota](#5-giới-hạn--quota)
6. [Design Patterns — Mẫu Thiết Kế](#6-design-patterns--mẫu-thiết-kế)
7. [Best Practices — Thực Hành Tốt Nhất](#7-best-practices--thực-hành-tốt-nhất)
8. [Ví Dụ Thực Tế](#8-ví-dụ-thực-tế)
9. [Câu Hỏi Phỏng Vấn](#9-câu-hỏi-phỏng-vấn)

---

## 1. Khái Niệm Cơ Bản

### Security Group Là Gì?

Security Group là **virtual firewall** (tường lửa ảo) kiểm soát traffic **inbound** (vào) và **outbound** (ra) cho các AWS resources như EC2, RDS, ECS, Lambda (khi có VPC config).

```
┌─────────────────────────────────────────────────┐
│                    VPC                          │
│                                                 │
│   ┌─────────────────────────────────────────┐  │
│   │              Subnet                     │  │
│   │                                         │  │
│   │   ┌─────────────────────────────────┐   │  │
│   │   │         Security Group          │   │  │
│   │   │  ┌──────────────────────────┐   │   │  │
│   │   │  │       EC2 Instance       │   │   │  │
│   │   │  │  (gắn với ENI)           │   │   │  │
│   │   │  └──────────────────────────┘   │   │  │
│   │   └─────────────────────────────────┘   │  │
│   └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Đặc Điểm Quan Trọng

| Đặc Điểm | Chi Tiết |
|----------|----------|
| **Stateful** | Return traffic (traffic phản hồi) tự động được cho phép |
| **Instance-level** | Áp dụng cho ENI, không phải subnet |
| **Allow-only** | Chỉ có Allow rules, không có Deny rules |
| **Default deny** | Traffic không khớp rule nào → bị chặn |
| **Multiple SGs** | Một ENI có thể gắn nhiều SGs (tối đa 5) |
| **Immediate effect** | Thay đổi rule có hiệu lực ngay lập tức |

---

## 2. Cách Hoạt Động Stateful

### Stateful — Nhớ Trạng Thái Kết Nối

```
Client (bên ngoài)          Security Group          EC2 Instance
─────────────────           ──────────────          ────────────

  SYN  ──────────────────►  [Inbound Rule?]
                              Port 443: ALLOW ──►    [Nhận request]
                                                     [Xử lý...]

  ◄──────────────────────  [Return traffic]
  SYN-ACK                   Tự động ALLOW            [Gửi phản hồi]
  (không cần outbound rule)

  ACK  ──────────────────►  [Kết nối đã được nhớ]
                              Tự động ALLOW  ──►
```

**Ý nghĩa thực tế:** Nếu bạn tạo inbound rule cho port 443, bạn **không cần** tạo outbound rule cho phản hồi — Security Group tự động cho phép return traffic.

### So Sánh Với Stateless (Network ACL)

```
Stateful (Security Group):
  Request vào  → Kiểm tra inbound rule → Allow
  Response ra  → TỰ ĐỘNG Allow (không cần rule)

Stateless (Network ACL):
  Request vào  → Kiểm tra inbound rule → Allow
  Response ra  → Kiểm tra outbound rule → PHẢI CÓ RULE
```

---

## 3. Cấu Trúc Rules

### Inbound Rules (Quy Tắc Vào)

| Trường | Mô Tả | Ví Dụ |
|--------|-------|-------|
| **Type** | Loại giao thức | HTTP, HTTPS, SSH, Custom TCP |
| **Protocol** | TCP, UDP, ICMP, All | TCP |
| **Port range** | Cổng hoặc dải cổng | 443 hoặc 8000-8080 |
| **Source** | Nguồn traffic | 0.0.0.0/0, 10.0.0.0/8, sg-xxxx |
| **Description** | Mô tả (tùy chọn nhưng nên có) | "HTTPS from internet" |

### Outbound Rules (Quy Tắc Ra)

Tương tự inbound nhưng thay **Source** bằng **Destination** (Đích).

Mặc định: Outbound rule cho phép **tất cả traffic ra** (`0.0.0.0/0 All Traffic`).

### Default Security Group (Nhóm Bảo Mật Mặc Định)

Mỗi VPC có một Default Security Group với:
- **Inbound:** Cho phép traffic từ cùng Security Group (self-referencing)
- **Outbound:** Cho phép tất cả traffic ra

```
Default SG — Inbound:
  Source: sg-default (chính nó)  Protocol: All  Port: All  → ALLOW

Default SG — Outbound:
  Destination: 0.0.0.0/0  Protocol: All  Port: All  → ALLOW
```

**Thực hành tốt nhất:** Không dùng Default Security Group cho resources trong production. Tạo SG riêng cho từng tier.

---

## 4. Tham Chiếu Security Group Khác

Thay vì dùng IP address (địa chỉ IP), bạn có thể tham chiếu một Security Group khác làm Source/Destination. Đây là tính năng rất mạnh cho kiến trúc micro-segmentation (phân đoạn vi mô).

### Ví Dụ 3-Tier Architecture

```
┌──────────────────────────────────────────────────┐
│  sg-web (Web Tier Security Group)                │
│  Inbound: 0.0.0.0/0:443 ALLOW                   │
│  Inbound: 0.0.0.0/0:80  ALLOW                   │
│                                                  │
│  ┌─────────────────────────────────────────┐     │
│  │  Web Server EC2                         │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
         │ Gọi App Tier trên port 8080
         ▼
┌──────────────────────────────────────────────────┐
│  sg-app (App Tier Security Group)                │
│  Inbound: Source sg-web Port 8080 ALLOW         │
│  (Chỉ cho phép traffic từ sg-web)               │
│                                                  │
│  ┌─────────────────────────────────────────┐     │
│  │  App Server EC2                         │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
         │ Kết nối DB trên port 5432
         ▼
┌──────────────────────────────────────────────────┐
│  sg-db (Database Tier Security Group)            │
│  Inbound: Source sg-app Port 5432 ALLOW         │
│  (Chỉ cho phép traffic từ sg-app)               │
│                                                  │
│  ┌─────────────────────────────────────────┐     │
│  │  RDS PostgreSQL                         │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
```

**Lợi ích:** Khi scale thêm App Servers, không cần cập nhật DB Security Group — tất cả instances với `sg-app` đều được phép tự động.

### Self-Referencing (Tự Tham Chiếu)

Cho phép traffic giữa các instances trong cùng Security Group:

```
sg-cluster:
  Inbound: Source sg-cluster  Port: All  → ALLOW
  (Các nodes trong cluster có thể giao tiếp với nhau)
```

Dùng cho: Elasticsearch clusters, Redis clusters, Kubernetes nodes.

---

## 5. Giới Hạn & Quota

| Giới Hạn | Mặc Định | Tối Đa (Có Thể Tăng) |
|---------|----------|----------------------|
| Security Groups per VPC | 2,500 | 10,000 |
| Rules per Security Group (inbound + outbound) | 60 + 60 = 120 | 1,000 |
| Security Groups per ENI | 5 | 16 |
| ENIs per instance | Tùy instance type | — |

**Lưu ý:** Mỗi rule tham chiếu một SG khác tính là 1 rule cho mỗi IP address trong SG đó (với cơ chế tracking). Hiểu điều này quan trọng khi SG có nhiều members.

---

## 6. Design Patterns — Mẫu Thiết Kế

### Pattern 1: Bastion Host (Máy Chủ Nhảy)

```
Internet → sg-bastion (Port 22 từ IP cụ thể) → Bastion EC2
                                                      │
                                                      ▼
                                               sg-private
                                               (Port 22 từ sg-bastion)
                                                      │
                                                      ▼
                                               Private EC2 Instances
```

```
# sg-bastion
Inbound: TCP 22 Source: YOUR_OFFICE_IP/32  → ALLOW

# sg-private
Inbound: TCP 22 Source: sg-bastion         → ALLOW
```

### Pattern 2: ALB → EC2 Pattern

```
Internet → ALB (sg-alb) → EC2 instances (sg-ec2)
```

```
# sg-alb
Inbound:  TCP 80  0.0.0.0/0  ALLOW
Inbound:  TCP 443 0.0.0.0/0  ALLOW
Outbound: TCP 8080 sg-ec2    ALLOW (chỉ gửi đến EC2)

# sg-ec2
Inbound: TCP 8080 sg-alb ALLOW  (chỉ nhận từ ALB)
Outbound: All traffic → ALLOW   (EC2 gọi ra ngoài: S3, APIs)
```

**Lợi ích:** EC2 instances không bao giờ nhận direct traffic từ internet — bắt buộc qua ALB.

### Pattern 3: VPC Endpoint Access (Truy Cập Điểm Cuối VPC)

```
EC2 (sg-app) → VPC Endpoint (sg-endpoint) → AWS Service (S3, RDS...)
```

```
# sg-endpoint
Inbound: TCP 443 sg-app ALLOW  (chỉ cho phép app tier gọi vào endpoint)
```

### Pattern 4: Lambda → RDS Pattern

```
# sg-lambda
Outbound: TCP 5432 sg-rds ALLOW

# sg-rds
Inbound: TCP 5432 sg-lambda ALLOW
```

---

## 7. Best Practices — Thực Hành Tốt Nhất

### ✅ Nên Làm

1. **Đặt tên có ý nghĩa:** `prod-web-sg`, `staging-db-sg`, `corp-vpn-bastion-sg`
2. **Luôn thêm Description:** Mô tả rõ mục đích của từng rule
3. **Tham chiếu SG thay vì IP:** Dùng `sg-xxxx` thay vì CIDR block nội bộ
4. **Least Privilege:** Chỉ mở đúng port cần thiết, đúng source cần thiết
5. **Tách biệt theo môi trường:** `prod-*`, `staging-*`, `dev-*`
6. **Review định kỳ:** Xóa rules không còn dùng
7. **Dùng prefix lists:** Quản lý tập IP tập trung cho corporate networks

### ❌ Không Nên Làm

1. **Không mở SSH/RDP từ `0.0.0.0/0`** trong production — dùng Bastion Host hoặc Systems Manager Session Manager
2. **Không dùng Default Security Group** cho workloads production
3. **Không tạo "super SG"** với quá nhiều rules cho nhiều purposes khác nhau
4. **Không hardcode IP** cho internal services — dùng SG reference
5. **Không để orphaned rules** — rules tham chiếu SG đã bị xóa
6. **Không mở port range rộng** như 1024-65535 inbound

### Quy Tắc Đặt Tên Gợi Ý

```
{environment}-{tier}-{purpose}-sg

Ví dụ:
  prod-web-alb-sg        → Production ALB Security Group
  prod-app-ec2-sg        → Production App EC2 Security Group
  prod-db-rds-sg         → Production RDS Security Group
  shared-bastion-sg      → Bastion Host (dùng chung)
  dev-all-sg             → Development (mở rộng hơn)
```

---

## 8. Ví Dụ Thực Tế

### Ví Dụ 1: Web Application 3-Tier Production

```
# Load Balancer Security Group
sg-prod-alb:
  Inbound:
    HTTPS 443 0.0.0.0/0       → ALLOW  (traffic HTTPS từ internet)
    HTTP  80  0.0.0.0/0       → ALLOW  (redirect về HTTPS)
  Outbound:
    TCP 8443 sg-prod-app       → ALLOW  (forward tới app tier)

# Application Security Group
sg-prod-app:
  Inbound:
    TCP 8443 sg-prod-alb       → ALLOW  (chỉ nhận từ ALB)
    TCP 22   sg-prod-bastion   → ALLOW  (SSH qua bastion)
  Outbound:
    TCP 5432 sg-prod-db        → ALLOW  (kết nối PostgreSQL)
    TCP 6379 sg-prod-redis     → ALLOW  (kết nối Redis cache)
    TCP 443  0.0.0.0/0         → ALLOW  (gọi external APIs: payment, email)

# Database Security Group
sg-prod-db:
  Inbound:
    TCP 5432 sg-prod-app       → ALLOW  (chỉ nhận từ app tier)
  Outbound:  (Không cần — DB không chủ động kết nối ra ngoài)

# Redis Cache Security Group
sg-prod-redis:
  Inbound:
    TCP 6379 sg-prod-app       → ALLOW
  Outbound:  (Không cần)

# Bastion Host Security Group
sg-prod-bastion:
  Inbound:
    TCP 22  203.0.113.10/32    → ALLOW  (chỉ từ IP văn phòng)
  Outbound:
    TCP 22  10.0.0.0/8         → ALLOW  (SSH tới internal resources)
```

### Ví Dụ 2: Microservices với ECS

```
# ALB facing internet
sg-ecs-alb:
  Inbound: TCP 443 0.0.0.0/0 ALLOW

# Service A (User Service)
sg-svc-user:
  Inbound: TCP 3000 sg-ecs-alb ALLOW
  Outbound: TCP 5432 sg-db-user ALLOW

# Service B (Order Service)
sg-svc-order:
  Inbound: TCP 3001 sg-ecs-alb ALLOW
  Outbound:
    TCP 5432   sg-db-order   ALLOW
    TCP 3000   sg-svc-user   ALLOW  (Order gọi User Service)
    TCP 6379   sg-cache      ALLOW
```

---

## 9. Câu Hỏi Phỏng Vấn

### Câu 1: Security Group và Network ACL khác nhau thế nào?

**Trả lời:**
- **Security Group:** Stateful (trạng thái), hoạt động ở cấp ENI/instance, chỉ có Allow rules, tự động cho phép return traffic.
- **Network ACL:** Stateless (phi trạng thái), hoạt động ở cấp subnet, có cả Allow và Deny rules, phải tạo rules cho cả 2 chiều.

**Dùng khi nào:**
- SG: Kiểm soát traffic cho từng instance/service cụ thể
- NACL: Thêm lớp bảo vệ ở cấp subnet, chặn dải IP độc hại

---

### Câu 2: Tại sao dùng SG reference thay vì IP CIDR?

**Trả lời:** Khi dùng SG reference:
- Không cần cập nhật rule khi scale (thêm/bớt instances)
- Tránh lỗi do IP thay đổi (EC2 restart → IP mới)
- Rõ ràng về intent: "Cho phép traffic từ web tier" thay vì một dải IP cụ thể
- Auto-tracking: AWS theo dõi instances trong SG tự động

---

### Câu 3: Một EC2 có 2 Security Groups — cách hoạt động?

**Trả lời:** Rules từ tất cả Security Groups được **hợp nhất (merge)** lại. Traffic được cho phép nếu **bất kỳ** Security Group nào có matching rule (logic OR). Không có SG nào có thể "deny" traffic mà SG kia đã "allow".

```
SG-1 Inbound: TCP 22 0.0.0.0/0 ALLOW
SG-2 Inbound: TCP 443 0.0.0.0/0 ALLOW
→ Kết quả: Cả port 22 và 443 đều được mở
```

---

### Câu 4: Cách debug khi EC2 không nhận được traffic?

**Checklist (theo thứ tự):**

```
1. Security Group inbound rule có mở đúng port không?
2. Outbound rule của source (ALB, EC2 khác) có allow port đó không?
3. Network ACL của subnet có block traffic không?
4. Route Table có route đến destination không?
5. Instance có đang chạy không? Application có listen đúng port không?
6. VPC Flow Logs: Kiểm tra ACCEPT/REJECT để xác định lớp nào chặn
```

---

### Câu 5: Security Group có thể áp dụng cho những resource nào?

**Trả lời:** Security Group áp dụng cho bất kỳ resource nào có ENI (Elastic Network Interface):
- EC2 instances
- RDS databases
- ECS tasks (với awsvpc network mode)
- Lambda functions (khi cấu hình VPC)
- ELB (Elastic Load Balancer)
- ElastiCache clusters
- OpenSearch domains
- EFS (Elastic File System) mount targets

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Tiếp Theo |
|-------|----------|-----------|
| [README.md](./README.md) | **1-security-groups.md** | [2-network-acls.md](./2-network-acls.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
