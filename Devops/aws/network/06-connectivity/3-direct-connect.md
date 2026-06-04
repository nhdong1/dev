# 3 — AWS Direct Connect — Kết Nối Chuyên Dụng

> AWS Direct Connect — đường truyền mạng riêng tư, chuyên dụng từ cơ sở hạ tầng on-premises (tại chỗ) của doanh nghiệp đến AWS, không đi qua internet công cộng, cung cấp băng thông cao, độ trễ thấp và ổn định.

---

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc & Thành Phần](#kiến-trúc--thành-phần)
3. [Loại Kết Nối](#loại-kết-nối)
4. [Virtual Interfaces — Giao Diện Ảo](#virtual-interfaces--giao-diện-ảo)
5. [BGP Routing](#bgp-routing)
6. [High Availability Patterns](#high-availability-patterns)
7. [Bảo Mật — MACsec & Encryption](#bảo-mật--macsec--encryption)
8. [Chi Phí](#chi-phí)
9. [Direct Connect vs VPN](#direct-connect-vs-vpn)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Direct Connect Là Gì?

**AWS Direct Connect** (DX) cung cấp **đường truyền vật lý riêng** từ datacenter của doanh nghiệp đến AWS thông qua **Direct Connect Location** — một cơ sở co-location (thuê chỗ đặt thiết bị) được AWS ủy quyền.

```
On-Premises                        AWS
Datacenter ──── Leased Line ──── DX Location ──── AWS Region
               (Đường thuê riêng)  (Co-location)   (US-East-1, etc.)
```

### Tại Sao Dùng Direct Connect?

| Vấn Đề Với Internet | Direct Connect Giải Quyết |
|--------------------|--------------------------|
| Băng thông không ổn định | Dedicated bandwidth (băng thông riêng, không chia sẻ) |
| Độ trễ cao và biến động | Latency thấp, nhất quán (single-digit ms) |
| Chi phí data transfer cao | Giảm 40–60% chi phí data transfer |
| Rủi ro bảo mật (public internet) | Đường truyền riêng, không qua internet |
| SLA không đảm bảo | 99.99% uptime với redundant connections |

---

## Kiến Trúc & Thành Phần

### Sơ Đồ Toàn Diện

```
┌─────────────────────────────────────────────────────────────────┐
│                         AWS Region                              │
│                                                                 │
│   ┌──────────────────────┐    ┌──────────────────────────────┐ │
│   │        VPC           │    │   Direct Connect Gateway     │ │
│   │   10.0.0.0/16        │◄───┤   (DX GW) — optional        │ │
│   │                      │    │   Cho multi-VPC/multi-region │ │
│   │  ┌───────────────┐   │    └──────────────────────────────┘ │
│   │  │ Private Subnet│   │                    │                 │
│   │  │ 10.0.1.0/24   │   │                    │                 │
│   │  └───────────────┘   │                    │                 │
│   └──────────────────────┘                    │                 │
│            │                                  │                 │
│   Virtual Private Gateway (VGW)               │                 │
│   Hoặc Transit Gateway (TGW)                  │                 │
│            │                                  │                 │
└────────────┼──────────────────────────────────┼─────────────────┘
             │                                  │
             └────────────────┬─────────────────┘
                              │ Private VIF hoặc Transit VIF
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│                    AWS Direct Connect Location                   │
│           (Equinix, CyberOne, Telx... — Co-location facility)   │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐   │
│   │                 Meet Me Room (MMR)                     │   │
│   │                                                        │   │
│   │  ┌──────────────────┐    ┌──────────────────────────┐ │   │
│   │  │  AWS Cage        │    │  Customer/Partner Cage   │ │   │
│   │  │  AWS DX Router   │◄──►│  Customer Router OR      │ │   │
│   │  │                  │    │  Partner/Carrier Router   │ │   │
│   │  └──────────────────┘    └──────────────────────────┘ │   │
│   └────────────────────────────────────────────────────────┘   │
│                              │ Cross Connect (Đấu Nối Chéo)    │
└──────────────────────────────┼──────────────────────────────────┘
                               │
                   Last Mile (Đường Cuối Dặm)
                   (Leased line / Dark fiber)
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│               On-Premises Datacenter                            │
│                                                                 │
│   ┌────────────────┐    ┌──────────────────────────────────┐   │
│   │ Customer Edge  │    │  Internal Network                │   │
│   │ Router / DX   │◄───┤  192.168.0.0/16                  │   │
│   │ Termination   │    │  Servers, Databases, Apps         │   │
│   └────────────────┘    └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Giải Thích Các Thành Phần

**1. Direct Connect Location — Vị Trí Kết Nối**
- Cơ sở co-location (Equinix, Coresite, Telx, CyberOne...)
- Nơi AWS đặt DX routers, khách hàng có thể thuê chỗ đặt router
- Danh sách đầy đủ: `aws directconnect describe-locations`

**2. Cross Connect — Đấu Nối Chéo**
- Cáp vật lý nối router của AWS với router của khách hàng/partner
- Trong cùng một co-location facility
- Yêu cầu Letter of Authorization (LOA — Thư Ủy Quyền) từ AWS

**3. Virtual Interface (VIF) — Giao Diện Ảo**
- Logical interface được tạo trên DX Connection
- Cho phép phân chia 1 physical connection thành nhiều logical connections
- 3 loại: Private VIF, Public VIF, Transit VIF (xem bên dưới)

**4. BGP Session — Phiên BGP**
- Border Gateway Protocol — giao thức trao đổi route giữa AWS và on-premises
- Cần BGP ASN (Autonomous System Number — Số Hệ Thống Tự Trị) từ cả 2 phía

---

## Loại Kết Nối

### Dedicated Connection — Kết Nối Chuyên Dụng

```
Đặc điểm:
- Đường cáp vật lý riêng từ DX Location đến AWS
- Yêu cầu khách hàng hoặc partner phải có thiết bị tại DX Location
- Băng thông: 1 Gbps, 10 Gbps, 100 Gbps (fixed)
- Thời gian cấp phát: Vài tuần (paperwork + physical installation)
- Phù hợp: Enterprise với lưu lượng lớn, cần SLA cao nhất

Quy trình:
1. Yêu cầu connection trong AWS Console
2. AWS gửi LOA-CFA (Letter of Authorization - Connecting Facility Assignment)
3. Đặt Cross Connect tại DX Location
4. Cấu hình BGP trên customer router
5. Tạo VIF trong AWS Console
```

### Hosted Connection — Kết Nối Được Lưu Trữ

```
Đặc điểm:
- APN Partner (đối tác AWS) cung cấp kết nối đến DX Location
- Không cần khách hàng có thiết bị tại DX Location
- Băng thông: 50 Mbps, 100 Mbps, 200 Mbps, ... 10 Gbps (flexible)
- Thời gian cấp phát: Nhanh hơn (partner đã có hạ tầng sẵn)
- Phù hợp: SMB, muốn băng thông nhỏ hơn hoặc không có cơ sở vật chất

Quy trình:
1. Liên hệ APN Partner (Verizon, Comcast, Equinix...)
2. Partner tạo hosted connection, share với AWS account của bạn
3. Bạn accept invitation trong AWS Console
4. Tạo VIF
```

### So Sánh

| Tiêu Chí | Dedicated | Hosted |
|---------|----------|--------|
| **Băng thông** | 1/10/100 Gbps | 50 Mbps – 10 Gbps |
| **Kiểm soát** | Đầy đủ | Phụ thuộc partner |
| **Thời gian** | Lâu hơn | Nhanh hơn |
| **VIF/connection** | Nhiều VIF trên 1 connection | 1 VIF/connection |
| **Chi phí setup** | Cao | Thấp |
| **SLA** | AWS SLA | Partner SLA + AWS |

---

## Virtual Interfaces — Giao Diện Ảo

### Private VIF — Giao Diện Ảo Riêng Tư

```
Dùng để kết nối với VPC qua VGW (Virtual Private Gateway) hoặc DX Gateway
- Truy cập private IP addresses trong VPC
- VLAN tag riêng cho mỗi VIF
- BGP peering trên 169.254.x.x/30 (link-local)

Ví dụ:
On-Premises ──[Private VIF]──► VGW ──► VPC (10.0.0.0/16)
BGP route: AWS quảng cáo 10.0.0.0/16 → on-premises
```

### Public VIF — Giao Diện Ảo Công Khai

```
Dùng để kết nối với AWS Public Services (S3, DynamoDB, SQS...) mà không qua internet
- Truy cập public IP ranges của AWS
- Không cần qua VPC
- Phù hợp: Database migration, backup lớn, direct AWS API access

Ví dụ:
On-Premises ──[Public VIF]──► AWS Public Endpoints (S3, DynamoDB...)
BGP route: AWS quảng cáo tất cả AWS public prefixes → on-premises
```

### Transit VIF — Giao Diện Ảo Trung Chuyển

```
Dùng để kết nối với Transit Gateway (TGW)
- 1 VIF kết nối được nhiều VPCs (thông qua TGW)
- Thay thế nhiều Private VIFs riêng lẻ
- Phù hợp: Môi trường với nhiều VPCs, multi-account

Ví dụ:
On-Premises ──[Transit VIF]──► TGW ──► VPC-A
                                    ├──► VPC-B
                                    └──► VPC-C
```

---

## BGP Routing

### Cấu Hình BGP Cơ Bản

```
Private VIF BGP Session:
- Customer side IP: 169.254.255.1/30
- AWS side IP: 169.254.255.2/30
- Customer ASN: 65001 (private ASN range: 64512-65534)
- AWS ASN: 64512 (hoặc ASN khác do customer chọn lúc tạo VGW)
- BGP Authentication Key: Optional nhưng khuyến nghị (MD5)
```

### Route Advertisement — Quảng Bá Route

```
On-premises quảng bá vào AWS:
  - 192.168.0.0/16 (mạng nội bộ)
  - Tối đa 100 prefixes/VIF

AWS quảng bá vào on-premises:
  - 10.0.0.0/16 (VPC CIDR)
  - Nếu dùng DX Gateway: tất cả VPCs được associate
```

### BGP Communities — Cộng Đồng BGP

AWS hỗ trợ BGP communities để kiểm soát routing:

```
7224:9100 — Local AWS Region
7224:9200 — Tất cả Regions cùng continent (US, EU, APAC)
7224:9300 — Global (tất cả Regions)

Ví dụ: Chỉ nhận routes từ US East:
route-map FILTER permit
  match community 7224:9100

Ví dụ: Ưu tiên DX hơn VPN (dùng LOCAL_PREF):
Trên DX: LOCAL_PREF 200
Trên VPN: LOCAL_PREF 100
→ BGP chọn đường có LOCAL_PREF cao hơn
```

---

## High Availability Patterns

### Pattern 1: Single Connection (Không Khuyến Nghị)

```
On-Premises ──── DX Connection ──── AWS VPC
                (Single point of failure — Điểm lỗi duy nhất)
```

**SLA:** 99.9% (một số downtime trong năm)

### Pattern 2: Dual Connection — Same Location

```
                ┌─── DX Connection 1 ───►
On-Premises ────┤                          AWS VPC
                └─── DX Connection 2 ───►
                (Same DX Location)
```

**SLA:** 99.9% (DX Location vẫn là single point)

### Pattern 3: Dual Connection — Different Locations (Khuyến Nghị)

```
                ┌─── DX Location A ── Connection 1 ───►
On-Premises ────┤                                        AWS VPC
                └─── DX Location B ── Connection 2 ───►
```

**SLA:** 99.99% (resilient — đàn hồi, bền vững)

### Pattern 4: Direct Connect + VPN Backup (Phổ Biến Nhất)

```
                ┌─── DX Connection (Primary — Chính) ───►
On-Premises ────┤                                          AWS VPC
                └─── VPN Tunnel (Failover — Dự Phòng) ──►
```

**Cơ chế failover tự động với BGP:**
```
DX path: LOCAL_PREF = 200 (ưu tiên cao hơn)
VPN path: LOCAL_PREF = 100 (dự phòng)
→ Khi DX down, BGP tự chọn VPN path
```

---

## Bảo Mật — MACsec & Encryption

### Vấn Đề Bảo Mật Của Direct Connect

Direct Connect mặc định **không mã hóa**. Dữ liệu truyền qua đường vật lý không được encrypt.

```
Rủi ro: Nếu ai đó có thể tap (nghe lén) đường cáp vật lý → có thể đọc dữ liệu
Thực tế: Rất khó trong co-location facility nhưng vẫn là compliance requirement
```

### Giải Pháp 1: MACsec (IEEE 802.1AE)

```
MACsec — Media Access Control Security — Bảo Mật Lớp MAC:
- Mã hóa ở Layer 2 (Data Link)
- Mã hóa toàn bộ frame Ethernet
- AES-256-GCM
- Không ảnh hưởng đến latency (hardware offload)

Yêu cầu:
- Dedicated connection 10/100 Gbps (không phải tất cả locations)
- Customer router phải hỗ trợ MACsec
- Chỉ bảo vệ đoạn DX Location → AWS (không bảo vệ last mile)
```

### Giải Pháp 2: IPsec VPN over Direct Connect

```
Cấu trúc: IPsec tunnel chạy bên trong DX connection
- Mã hóa end-to-end (on-premises → AWS VPC)
- Kết hợp privacy của DX với security của VPN
- Nhược điểm: Giảm băng thông hiệu quả (overhead IPsec)
- Phù hợp: Compliance requirements (PCI-DSS, HIPAA)

Cấu hình:
On-Premises ──[DX]──► Public VIF ──► VGW ──► IPsec Tunnel ──► VPC
                    (Physical)     (Logical)  (Encrypted)
```

---

## Chi Phí

### Cấu Trúc Chi Phí

| Thành Phần | Chi Tiết | Ví Dụ Giá |
|-----------|---------|-----------|
| **Port fee** | Phí giờ cho DX port | 1 Gbps: $0.30/giờ ≈ $216/tháng |
| **Data transfer out** | AWS → on-premises | $0.02/GB (rẻ hơn internet $0.09/GB) |
| **Data transfer in** | On-premises → AWS | Miễn phí |
| **VIF** | Không phí thêm | - |

### So Sánh Chi Phí Với VPN

```
Kịch bản: 10 TB/tháng data transfer out

Direct Connect 1 Gbps:
  Port fee:      $0.30 × 24 × 30 = $216/tháng
  Data out:      $0.02 × 10240 GB = $204.80/tháng
  Tổng:          ~$421/tháng

Site-to-Site VPN:
  Connection:    $0.05 × 24 × 30 = $36/tháng
  Data out:      $0.09 × 10240 GB = $921.60/tháng
  Tổng:          ~$958/tháng

→ Với 10 TB+/tháng, DX rẻ hơn VPN
→ Break-even point: ~3-4 TB/tháng
```

---

## Direct Connect vs VPN

### Bảng So Sánh Toàn Diện

| Tiêu Chí | Direct Connect | Site-to-Site VPN |
|---------|---------------|-----------------|
| **Đường truyền** | Riêng tư, vật lý | Internet công cộng |
| **Mã hóa** | Không có (cần thêm) | IPsec tích hợp |
| **Băng thông** | 50 Mbps – 100 Gbps | Max 1.25 Gbps/tunnel |
| **Độ trễ** | Thấp, nhất quán | Không ổn định |
| **SLA** | 99.9% – 99.99% | Không đảm bảo |
| **Chi phí setup** | Cao (months, NRC) | Không (pay-per-use) |
| **Thời gian setup** | Tuần – tháng | Vài giờ |
| **Chi phí data** | Thấp (cao volume) | Cao |
| **Phù hợp cho** | Enterprise, production | Dev, backup, SMB |

### Ma Trận Quyết Định

```
Cần kết nối on-premises ↔ AWS?
│
├─ Băng thông > 1 Gbps? ──────────────────────────────────► Direct Connect
│
├─ Latency < 5ms quan trọng? ──────────────────────────────► Direct Connect
│
├─ Compliance yêu cầu private link? ───────────────────────► Direct Connect
│
├─ Chỉ cần backup/DR connectivity? ────────────────────────► VPN
│
├─ Setup < 1 tuần? ─────────────────────────────────────────► VPN
│
└─ Ngân sách thấp + volume nhỏ? ───────────────────────────► VPN
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích kiến trúc Direct Connect từ on-premises đến VPC

**Trả lời mẫu:**

> Direct Connect hoạt động qua 3 lớp. Lớp vật lý: đường cáp từ datacenter doanh nghiệp đến Direct Connect Location (co-location như Equinix), tại đây có Cross Connect nối router của khách hàng với router của AWS trong Meet Me Room. Lớp logic: Virtual Interface (VIF) được tạo trên connection — Private VIF để reach VPC private IPs, Public VIF để reach AWS public services, Transit VIF để kết nối qua Transit Gateway. Lớp routing: BGP session được thiết lập để trao đổi routes giữa on-premises ASN và AWS ASN, cho phép on-premises biết prefix của VPC và ngược lại.

### Q2: Direct Connect có mã hóa không, và làm thế nào để thêm mã hóa?

**Trả lời mẫu:**

> Direct Connect mặc định không mã hóa — đây là điểm nhiều người bỏ qua. Dữ liệu truyền trên đường vật lý giữa on-premises và DX Location không được encrypt. Có 2 cách thêm mã hóa: Đầu tiên là MACsec (IEEE 802.1AE) — mã hóa Layer 2, AES-256-GCM, không ảnh hưởng latency, chỉ hỗ trợ trên dedicated connection 10/100 Gbps tại một số locations. Cách thứ hai là IPsec VPN over Direct Connect — chạy VPN tunnel bên trong DX, mã hóa end-to-end nhưng giảm effective bandwidth do overhead. Với môi trường PCI-DSS hoặc HIPAA, luôn cần một trong hai giải pháp này.

### Q3: Thiết kế high availability cho Direct Connect

**Trả lời mẫu:**

> Mức HA cao nhất là dùng 2 dedicated connections ở 2 Direct Connect Locations khác nhau (ví dụ Equinix NY và Equinix Chicago) — đây là kiến trúc AWS khuyến nghị cho 99.99% SLA. Ngoài ra, cần 2 router vật lý ở on-premises (không có single point of failure nào từ on-premises đến AWS). Trong thực tế production, tôi hay thêm Site-to-Site VPN làm failover thứ ba — khi cả 2 DX locations đều có sự cố (rất hiếm), VPN qua internet vẫn duy trì kết nối. BGP LOCAL_PREF được dùng để ưu tiên DX > VPN trong routing.

### Q4: Private VIF vs Public VIF vs Transit VIF

| VIF Type | Kết Nối Đến | Use Case |
|---------|-------------|---------|
| Private VIF | VGW → VPC | Truy cập private IPs trong VPC |
| Public VIF | AWS Public Endpoints | S3, DynamoDB qua DX (không qua internet) |
| Transit VIF | Transit Gateway | Nhiều VPCs qua 1 VIF |

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [2-client-vpn.md](./2-client-vpn.md) |
| → Tiếp theo | [4-direct-connect-gateway.md](./4-direct-connect-gateway.md) |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
