# 5 — VPC Peering & Resource Sharing (Kết Nối Ngang Hàng VPC & Chia Sẻ Tài Nguyên)

> VPC Peering (Kết Nối Ngang Hàng VPC) cho phép hai VPC kết nối với nhau qua mạng riêng của AWS, như thể chúng nằm trong cùng một mạng. Traffic không đi qua internet.

---

## 📋 Mục Lục

1. [VPC Peering là gì?](#1-vpc-peering-là-gì)
2. [Cách VPC Peering Hoạt Động](#2-cách-vpc-peering-hoạt-động)
3. [Thiết Lập VPC Peering](#3-thiết-lập-vpc-peering)
4. [Giới Hạn VPC Peering](#4-giới-hạn-vpc-peering)
5. [VPC Peering vs Transit Gateway](#5-vpc-peering-vs-transit-gateway)
6. [AWS Resource Access Manager — RAM](#6-aws-resource-access-manager--ram)
7. [Use Cases Thực Tế](#7-use-cases-thực-tế)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. VPC Peering là gì?

**VPC Peering** là kết nối mạng giữa hai VPC cho phép traffic đi lại giữa chúng bằng địa chỉ IP riêng. Traffic đi qua **infrastructure riêng của AWS** — không qua internet, không qua VPN gateway, không qua NAT.

### Đặc Điểm Chính

| Thuộc Tính | Mô Tả |
|-----------|-------|
| Loại kết nối | Point-to-point (điểm đến điểm), 1-1 |
| Phạm vi | Cùng Region hoặc Cross-Region (Inter-Region Peering) |
| Cross-Account | Hỗ trợ (kết nối VPC ở tài khoản khác) |
| Phí | Không phí cho Peering connection; tính phí data transfer |
| Mã hóa | Traffic được mã hóa khi Cross-Region |
| Bandwidth | Không giới hạn |

### Khi Nào Dùng VPC Peering?

```
Phù hợp:
✓ Kết nối 2-5 VPCs với nhau
✓ Kết nối VPC development với VPC production (đọc shared services)
✓ Kết nối VPC ở tài khoản khác (vendor, partner)
✓ Latency thấp, bandwidth cao quan trọng

Không phù hợp:
✗ Kết nối nhiều hơn 5-10 VPCs (quản lý phức tạp)
✗ Cần transitive routing (định tuyến bắc cầu) giữa các VPC
✗ Hub-and-spoke architecture với nhiều VPCs
→ Dùng Transit Gateway thay thế
```

---

## 2. Cách VPC Peering Hoạt Động

### Sơ Đồ Kết Nối

```
VPC A (10.0.0.0/16)  ←──── Peering Connection ────►  VPC B (10.1.0.0/16)
   Subnet: 10.0.1.0/24                                  Subnet: 10.1.1.0/24
   Instance A: 10.0.1.10  ◄── Private Traffic ──►       Instance B: 10.1.1.20
```

Traffic từ Instance A đến Instance B:
1. Instance A (10.0.1.10) gửi packet đến 10.1.1.20
2. Route Table của VPC A có entry: `10.1.0.0/16 → pcx-xxxxxxxx`
3. Packet đi qua Peering Connection, đến VPC B
4. Route Table của VPC B có entry: `10.0.0.0/16 → pcx-xxxxxxxx`
5. Instance B (10.1.1.20) nhận packet

### Yêu Cầu Bắt Buộc: Non-overlapping CIDRs

**CIDRs của hai VPC KHÔNG ĐƯỢC trùng nhau.** Đây là giới hạn cứng không thể workaround.

```
✅ Cho phép:
VPC A: 10.0.0.0/16
VPC B: 10.1.0.0/16  ← Không trùng → Peering được

❌ Không cho phép:
VPC A: 10.0.0.0/16
VPC B: 10.0.0.0/24  ← CIDR B nằm trong CIDR A → Không thể Peering
```

### Không Có Transitive Routing (Định Tuyến Bắc Cầu)

Đây là giới hạn quan trọng nhất cần nhớ:

```
VPC A ←──peering──► VPC B ←──peering──► VPC C

VPC A KHÔNG thể kết nối VPC C qua VPC B!
Traffic từ A đến C phải có Peering trực tiếp A ↔ C
```

Điều này có nghĩa là với N VPCs kết nối full mesh, cần N×(N-1)/2 peering connections:
- 3 VPCs: 3 peerings
- 5 VPCs: 10 peerings
- 10 VPCs: 45 peerings ← Quản lý cực kỳ phức tạp

---

## 3. Thiết Lập VPC Peering

### Bước 1: Tạo Peering Request

```
AWS Console:
VPC → Peering Connections → Create Peering Connection

Điền thông tin:
- Peering Connection name tag: prod-to-shared-services
- VPC (Requester — Bên Yêu Cầu): vpc-aaaa (VPC của bạn)
- Account: Same account / Another account (nhập Account ID)
- Region: Same region / Another region
- VPC (Accepter — Bên Chấp Nhận): vpc-bbbb
```

### Bước 2: Chấp Nhận Peering Request

Nếu cùng tài khoản, bạn phải chủ động chấp nhận:
```
VPC → Peering Connections → Chọn connection đang Pending Acceptance
→ Actions → Accept Request
```

Nếu khác tài khoản, chủ tài khoản của VPC đích phải login và Accept.

### Bước 3: Cập Nhật Route Tables (Cả Hai Phía)

**Quan trọng:** Cả hai VPC đều phải cập nhật Route Table — thiếu một bên sẽ không thông.

```
VPC A — Route Table (Private Subnet):
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │
│ 10.1.0.0/16         │ pcx-0abc123def      │  ← Thêm route đến VPC B
│ 0.0.0.0/0           │ nat-xxxxxxxx        │
└─────────────────────┴─────────────────────┘

VPC B — Route Table (Private Subnet):
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.1.0.0/16         │ local               │
│ 10.0.0.0/16         │ pcx-0abc123def      │  ← Thêm route đến VPC A
└─────────────────────┴─────────────────────┘
```

### Bước 4: Cập Nhật Security Groups

Security Group của instances đích phải cho phép traffic từ CIDR của VPC nguồn:

```
VPC B — Security Group của Database:
Inbound Rules:
┌──────────┬──────────┬─────────────────────────────┐
│ Protocol │ Port     │ Source                      │
├──────────┼──────────┼─────────────────────────────┤
│ TCP      │ 5432     │ 10.0.0.0/16  (CIDR VPC A)   │  ← PostgreSQL
└──────────┴──────────┴─────────────────────────────┘
```

### DNS Resolution Across Peering

Để resolve DNS hostname của instances trong VPC đối diện, phải bật:
- **Enable DNS resolution** trên Peering Connection
- **Enable DNS hostnames** trên cả hai VPCs

---

## 4. Giới Hạn VPC Peering

| Giới Hạn | Mặc Định |
|---------|---------|
| Peering connections per VPC | 50 (có thể tăng lên 125) |
| Outstanding peering requests | 25 |
| Expiry time của Peering Request chưa Accept | 7 ngày |

### Giới Hạn Không Thể Thay Đổi

- **Không có transitive routing** (bắc cầu qua VPC trung gian)
- **CIDRs không được trùng**
- **Không thể route qua Internet Gateway, VPN, Direct Connect của VPC đối diện**

---

## 5. VPC Peering vs Transit Gateway

### Khi Nào Dùng VPC Peering?

```
Tốt cho VPC Peering:
✓ Số lượng VPC nhỏ (≤ 5)
✓ Mỗi cặp VPC cần bandwidth cao, latency cực thấp
✓ Kiến trúc đơn giản, không cần routing phức tạp
✓ Chi phí thấp hơn (không có Transit Gateway fee)
✓ Kết nối cross-account với vendor/partner
```

### Khi Nào Dùng Transit Gateway?

```
Tốt cho Transit Gateway:
✓ Nhiều VPCs (≥ 5-10) cần kết nối với nhau
✓ Hub-and-spoke architecture (kiến trúc nan hoa)
✓ Kết nối VPC với on-premises (VPN/Direct Connect) tập trung
✓ Cần phân vùng routing (Route Domains)
✓ Multi-account, multi-region networking
```

### So Sánh Chi Phí

| Phương Án | Chi Phí |
|---------|--------|
| VPC Peering | Miễn phí cho connection; tính phí data transfer |
| Transit Gateway | ~$0.05/giờ/attachment + $0.02/GB data processed |

**Ví dụ:** 10 VPCs full mesh
- VPC Peering: 45 peering connections, quản lý 90 Route Table entries
- Transit Gateway: 10 attachments × $0.05 = $0.5/giờ = ~$360/tháng, nhưng chỉ 10 Route Table entries

---

## 6. AWS Resource Access Manager — RAM

**AWS RAM (Resource Access Manager — Trình Quản Lý Truy Cập Tài Nguyên)** là dịch vụ cho phép chia sẻ tài nguyên AWS giữa các tài khoản trong cùng AWS Organization (Tổ Chức AWS), thay vì phải tạo tài nguyên duplicate ở mỗi tài khoản.

### Tài Nguyên Có Thể Chia Sẻ Qua RAM

| Tài Nguyên | Mô Tả |
|-----------|-------|
| **Subnets** | Share Subnet sang tài khoản khác, cho phép deploy resources vào Subnet chung |
| **Transit Gateway** | Chia sẻ một Transit Gateway cho nhiều tài khoản |
| **Route 53 Resolver Rules** | Chia sẻ DNS resolver rules |
| **License Manager Configurations** | Quản lý license tập trung |
| **AWS Glue** Catalogs | Chia sẻ data catalog |

### Shared Subnets (Subnet Chia Sẻ) — Use Case Quan Trọng

Thay vì dùng VPC Peering, bạn có thể chia sẻ Subnets từ tài khoản trung tâm (networking account) sang các tài khoản ứng dụng:

```
Networking Account (VPC Owner):
└── VPC: 10.0.0.0/16
    ├── Subnet: 10.0.1.0/24 → Share qua RAM cho Account A
    └── Subnet: 10.0.2.0/24 → Share qua RAM cho Account B

Account A (App Team A):
└── Deploy EC2 instances vào 10.0.1.0/24 của Networking Account

Account B (App Team B):
└── Deploy EC2 instances vào 10.0.2.0/24 của Networking Account
```

**Lợi ích:**
- Quản lý network tập trung tại một tài khoản
- Các account ứng dụng không cần biết về networking details
- Không cần VPC Peering giữa các account
- Đây là pattern được AWS Well-Architected Framework khuyến nghị

---

## 7. Use Cases Thực Tế

### Use Case 1: Shared Services VPC

```
┌─────────────────────────────────────────────────────────────┐
│ Shared Services VPC (10.2.0.0/16)                          │
│   - Internal ALB                                            │
│   - Container Registry (ECR Pull Through Cache)             │
│   - Monitoring (Prometheus, Grafana)                        │
│   - Centralized Logging                                     │
└──────────────────────────────────────────────────────────────┘
          │                              │
          │ Peering                      │ Peering
          ▼                              ▼
┌─────────────────────┐       ┌─────────────────────┐
│ Production VPC      │       │ Staging VPC         │
│ (10.0.0.0/16)       │       │ (10.1.0.0/16)       │
│ App Servers         │       │ App Servers         │
└─────────────────────┘       └─────────────────────┘
```

Cả Production và Staging đều có thể dùng Shared Services mà không cần duplicate.

### Use Case 2: Database VPC Tách Biệt

```
App VPC (10.0.0.0/16) ←── Peering ──► Database VPC (10.10.0.0/16)
   App Servers                              RDS Cluster
   Chỉ có quyền kết nối port 5432           Aurora PostgreSQL
   đến DB Subnet
```

Tách Database VPC riêng giúp kiểm soát truy cập tốt hơn — chỉ App VPC được kết nối đến DB, không phải tất cả VPCs.

---

## 8. Câu Hỏi Phỏng Vấn

### Q1: VPC Peering là gì và có những giới hạn nào?

**Trả lời mẫu:**

> VPC Peering là kết nối mạng point-to-point giữa hai VPC, cho phép traffic đi qua mạng nội bộ AWS — không qua internet. Các giới hạn chính: (1) Không có transitive routing — A kết nối B và B kết nối C không có nghĩa A kết nối được C, phải có peering trực tiếp; (2) CIDRs của hai VPC không được trùng nhau; (3) Phải cập nhật Route Tables cả hai phía; (4) Số peering connections tăng theo bình phương số VPCs — 10 VPCs cần 45 peerings.

### Q2: Khi nào dùng VPC Peering và khi nào dùng Transit Gateway?

**Trả lời:**

> VPC Peering phù hợp khi kết nối ít VPCs (dưới 5-6) và kiến trúc đơn giản, vì không tốn phí cho connection itself và latency thấp hơn. Transit Gateway phù hợp khi có nhiều VPCs, cần hub-and-spoke architecture, hoặc cần kết nối tập trung với on-premises qua VPN/Direct Connect. Transit Gateway tốn khoảng $0.05/giờ/attachment nhưng đơn giản hóa quản lý routing đáng kể khi có nhiều VPCs.

### Q3: Giải thích tại sao Transitive Routing không hoạt động với VPC Peering.

**Trả lời:**

> VPC Peering là kết nối điểm-điểm, không phải một router. Khi A peering với B và B peering với C, không có cơ chế nào trong B tự động forward traffic từ A đến C — đó không phải là chức năng của Peering Connection. Để A kết nối C, phải có Peering Connection trực tiếp A-C. Đây là lý do tại sao khi có nhiều VPCs, Transit Gateway là lựa chọn tốt hơn — nó là một router thực sự, xử lý việc forward traffic giữa các VPCs được attach.

### Q4: Tại sao CIDRs hai VPC không được trùng nhau khi Peering?

**Trả lời:**

> Khi CIDRs trùng nhau, router không biết traffic đến một địa chỉ IP cụ thể nên đi về VPC nào. Ví dụ: VPC A và VPC B đều có range 10.0.1.0/24. Khi Instance trong VPC A gửi packet đến 10.0.1.50, đó là Instance trong VPC A hay VPC B? Không thể xác định được. Vì vậy, AWS yêu cầu bắt buộc các VPC muốn Peering phải có CIDRs không trùng nhau. Đây là lý do quan trọng phải lập kế hoạch IP ngay từ đầu cho toàn bộ môi trường.

---

## 📝 Tóm Tắt

| Khái Niệm | Điểm Quan Trọng |
|----------|----------------|
| VPC Peering | Kết nối point-to-point, không qua internet |
| Transitive Routing | **Không hỗ trợ** — A↔B và B↔C không có nghĩa A↔C |
| CIDRs | **Không được trùng** giữa hai VPC Peering |
| Route Tables | **Cả hai phía** phải cập nhật routes |
| VPC Peering vs Transit GW | Peering: ít VPCs, đơn giản; TGW: nhiều VPCs, hub-spoke |
| AWS RAM | Chia sẻ tài nguyên (Subnets, TGW) giữa tài khoản |

---

## 🔗 Điều Hướng

- ← [4. NAT Gateway](./4-nat-gateway.md)
- → [02 Security — Bảo Mật Mạng](../02-security/README.md)
- [README — Tổng Quan Section](./README.md)
- [Chỉ Mục Đầy Đủ](../INDEX.md)
