# 1 — Site-to-Site VPN — VPN Địa Điểm-đến-Địa Điểm

> AWS Site-to-Site VPN — Virtual Private Network (Mạng Riêng Ảo) — cho phép kết nối bảo mật giữa mạng on-premises (tại chỗ) và AWS VPC thông qua đường truyền internet được mã hóa bằng giao thức IPsec (Internet Protocol Security).

---

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc & Thành Phần](#kiến-trúc--thành-phần)
3. [Cơ Chế Hoạt Động](#cơ-chế-hoạt-động)
4. [Cấu Hình Chi Tiết](#cấu-hình-chi-tiết)
5. [Routing — Định Tuyến](#routing--định-tuyến)
6. [High Availability — Tính Sẵn Sàng Cao](#high-availability--tính-sẵn-sàng-cao)
7. [Bảo Mật & Mã Hóa](#bảo-mật--mã-hóa)
8. [Chi Phí](#chi-phí)
9. [So Sánh & Trade-offs](#so-sánh--trade-offs)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan

### Site-to-Site VPN Là Gì?

**Site-to-Site VPN** (VPN Địa Điểm-đến-Địa Điểm) kết nối toàn bộ mạng nội bộ của doanh nghiệp với AWS VPC. Đây là giải pháp "nhanh, rẻ" để thiết lập hybrid connectivity (kết nối lai).

```
On-Premises Network ←──IPsec Tunnel──→ AWS VPC
(Mạng nội bộ)          (Đường hầm mã hóa)  (Đám mây AWS)
```

### Khi Nào Dùng Site-to-Site VPN?

| Tình Huống | Phù Hợp? |
|-----------|----------|
| Cần kết nối nhanh (hours) với on-premises | ✅ Rất phù hợp |
| Băng thông < 1.25 Gbps là đủ | ✅ Phù hợp |
| Backup/failover cho Direct Connect | ✅ Rất phù hợp |
| Ngân sách hạn chế | ✅ Phù hợp |
| Cần băng thông lớn (10+ Gbps) | ❌ Không phù hợp |
| Yêu cầu độ trễ thấp, ổn định | ❌ Không phù hợp |
| Cần SLA 99.99% | ❌ Không đảm bảo |

---

## Kiến Trúc & Thành Phần

### Các Thành Phần Chính

```
┌──────────────────────────────────────────────────────────────────┐
│                         AWS Cloud                                │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                      VPC                                │   │
│   │                                                         │   │
│   │   ┌──────────────┐        ┌──────────────────────────┐ │   │
│   │   │  Private     │        │  Virtual Private Gateway  │ │   │
│   │   │  Subnet      │◄──────►│  (VGW) — Cổng Riêng Tư  │ │   │
│   │   │  10.0.1.0/24 │        │  AWS-managed endpoint     │ │   │
│   │   └──────────────┘        └────────────┬─────────────┘ │   │
│   └────────────────────────────────────────┼────────────────┘   │
│                                            │                     │
│                          ┌─────────────────▼──────────────────┐ │
│                          │         VPN Connection              │ │
│                          │  Tunnel 1: IP 52.x.x.x (Active)   │ │
│                          │  Tunnel 2: IP 52.y.y.y (Standby)  │ │
│                          └─────────────────┬──────────────────┘ │
└────────────────────────────────────────────┼────────────────────┘
                                             │
                                      Internet (IPsec)
                                      Mã hóa AES-256
                                             │
┌────────────────────────────────────────────┼────────────────────┐
│                On-Premises                 │                    │
│                                            │                    │
│   ┌──────────────────────────────────────▼──────────────────┐  │
│   │              Customer Gateway (CGW)                      │  │
│   │      Cổng Khách Hàng — Router/Firewall vật lý            │  │
│   │      Public IP: 203.x.x.x                                │  │
│   └──────────────────┬───────────────────────────────────────┘  │
│                      │                                           │
│   ┌──────────────────▼───────────────────────────────────────┐  │
│   │           On-Premises Network                             │  │
│   │           192.168.0.0/16                                  │  │
│   └──────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Giải Thích Từng Thành Phần

**1. Customer Gateway (CGW) — Cổng Khách Hàng**
- Là thiết bị vật lý hoặc phần mềm phía on-premises (router, firewall)
- Trong AWS Console, CGW là **tài nguyên logic** đại diện cho thiết bị đó
- Cần có: Static Public IP của thiết bị on-premises
- Hỗ trợ BGP (Border Gateway Protocol — Giao Thức Cổng Biên) hoặc Static routing

**2. Virtual Private Gateway (VGW) — Cổng Riêng Ảo**
- Là VPN concentrator phía AWS
- Được AWS quản lý hoàn toàn (highly available theo thiết kế)
- Attach vào 1 VPC tại một thời điểm
- Có thể kết nối nhiều VPN connections và Direct Connect connections
- Amazon side ASN (Autonomous System Number — Số Hệ Thống Tự Trị): mặc định 64512, có thể tùy chỉnh

**3. VPN Connection — Kết Nối VPN**
- Liên kết CGW và VGW
- Luôn có **2 tunnels** (đường hầm) cho high availability
- Mỗi tunnel có IP endpoint riêng phía AWS
- Pre-Shared Key (PSK — Khóa Chia Sẻ Trước) tự động tạo hoặc tùy chỉnh

**4. VPN Tunnel — Đường Hầm VPN**
- Mỗi connection có 2 tunnels (1 active, 1 standby theo mặc định)
- Mỗi tunnel hỗ trợ tối đa **1.25 Gbps**
- Sử dụng IKE (Internet Key Exchange — Trao Đổi Khóa Internet) để thỏa thuận mã hóa

---

## Cơ Chế Hoạt Động

### Quy Trình Thiết Lập Tunnel (IKE Handshake)

```
Phase 1 — IKE SA (Security Association — Liên Kết Bảo Mật)
────────────────────────────────────────────────────────────
CGW                          VGW (AWS)
 │                                │
 │── IKE_INIT (propose) ─────────►│
 │◄─ IKE_AUTH (agree) ───────────│
 │── Diffie-Hellman key exchange ─►│
 │◄─ IKE_SA established ─────────│
 │                                │
 ↓ Phase 1 hoàn thành            ↓
 
Phase 2 — IPsec SA (Đàm Phán Mã Hóa Thực Tế)
──────────────────────────────────────────────
 │── Quick Mode (propose ESP) ────►│
 │◄─ Agree on AES-256, SHA-256 ───│
 │── IPsec SA established ────────►│
 │                                │
 ↓ Tunnel sẵn sàng               ↓

Data Transfer — Truyền Dữ Liệu
────────────────────────────────
 │═══ Encrypted traffic (IPsec ESP) ══►│
 │◄══ Encrypted traffic (IPsec ESP) ══│
```

### Dead Peer Detection (DPD) — Phát Hiện Đường Hầm Chết

AWS sử dụng **DPD** để giám sát trạng thái tunnel:
- Gửi keepalive mỗi 10 giây
- Nếu không nhận response sau 30 giây → tunnel được đánh dấu DOWN
- AWS tự động chuyển traffic sang tunnel còn lại

---

## Cấu Hình Chi Tiết

### Tạo VPN Connection (AWS Console / CLI)

```bash
# Bước 1: Tạo Customer Gateway
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.10 \    # IP công khai của on-premises router
  --bgp-asn 65000               # ASN của on-premises (nếu dùng BGP)
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=OnPrem-CGW}]'

# Bước 2: Tạo Virtual Private Gateway
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512 \     # ASN của AWS side
  --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=Main-VGW}]'

# Bước 3: Attach VGW vào VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-xxxxxxxxx \
  --vpc-id vpc-xxxxxxxxx

# Bước 4: Tạo VPN Connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxxxxxxxx \
  --vpn-gateway-id vgw-xxxxxxxxx \
  --options '{
    "StaticRoutesOnly": false,
    "TunnelOptions": [
      {
        "TunnelInsideCidr": "169.254.10.0/30",
        "PreSharedKey": "MySecureKey123!",
        "Phase1EncryptionAlgorithms": [{"Value": "AES256"}],
        "Phase2EncryptionAlgorithms": [{"Value": "AES256"}],
        "Phase1IntegrityAlgorithms": [{"Value": "SHA2-256"}],
        "Phase2IntegrityAlgorithms": [{"Value": "SHA2-256"}],
        "Phase1DHGroupNumbers": [{"Value": 14}],
        "Phase2DHGroupNumbers": [{"Value": 14}]
      }
    ]
  }'
```

### Tham Số Mã Hóa Được Hỗ Trợ

| Tham Số | Tùy Chọn AWS | Khuyến Nghị |
|---------|-------------|-------------|
| **IKE version** | IKEv1, IKEv2 | IKEv2 |
| **Encryption** | AES128, AES256, AES128-GCM-16, AES256-GCM-16 | AES256-GCM-16 |
| **Integrity** | SHA1, SHA2-256, SHA2-384, SHA2-512 | SHA2-256+ |
| **DH Group** | 2, 14, 15, 16, 17, 18, 19, 20, 21 | 14+ |
| **Rekey interval** | 900–28800 seconds | 3600s (1 giờ) |

---

## Routing — Định Tuyến

### Static Routing (Định Tuyến Tĩnh)

Dùng khi on-premises router không hỗ trợ BGP:

```
Thêm static route thủ công:
- Phía AWS: Trong VPN Connection, thêm route 192.168.0.0/16 → VGW
- Phía on-premises: Thêm route 10.0.0.0/16 → VPN tunnel interface
```

**Nhược điểm:** Không tự động phát hiện route mới, phải update thủ công khi thay đổi mạng.

### Dynamic Routing với BGP (Giao Thức Cổng Biên)

Phương pháp được khuyến nghị:

```
BGP Session:
- CGW ASN (on-premises): 65000 (private ASN)
- VGW ASN (AWS): 64512 (private ASN)
- Tunnel inside CIDR: 169.254.x.x/30 (link-local)

BGP Advertisement:
- On-premises quảng cáo: 192.168.0.0/16 → AWS
- AWS quảng cáo: 10.0.0.0/16 → on-premises
- Tự động học route mới
```

### Route Propagation (Lan Truyền Route)

Bật route propagation để VGW tự động update route table của VPC:

```bash
# Bật route propagation cho VPC route table
aws ec2 enable-vgw-route-propagation \
  --route-table-id rtb-xxxxxxxxx \
  --gateway-id vgw-xxxxxxxxx
```

Khi bật, routes từ on-premises sẽ tự động xuất hiện trong route table của VPC.

---

## High Availability — Tính Sẵn Sàng Cao

### Cấu Hình 2 Tunnels (Mặc Định)

Mỗi VPN Connection có **2 tunnels** ở 2 AZ (Availability Zone — Vùng Sẵn Sàng) khác nhau:

```
On-Premises                    AWS
                               ┌─────────┐
                    Tunnel 1   │  AZ-a   │
CGW ──────────────────────────►│  VGW    │
 │              (Primary)      │Endpoint │
 │                             └─────────┘
 │                             ┌─────────┐
 │              Tunnel 2       │  AZ-b   │
 └────────────────────────────►│  VGW    │
                (Standby)      │Endpoint │
                               └─────────┘
```

**Lưu ý quan trọng:** AWS khuyến nghị cấu hình on-premises router để dùng **cả 2 tunnels active-active** thay vì active-passive để tận dụng tối đa băng thông.

### HA Architecture — Kiến Trúc Sẵn Sàng Cao

**Option 1: Single CGW với 2 Tunnels** (cơ bản)
```
1 CGW → 1 VGW → VPC
Downtime nếu CGW fails
```

**Option 2: 2 CGWs + 2 VPN Connections** (khuyến nghị)
```
CGW-1 ─┐
       ├─→ VGW → VPC
CGW-2 ─┘
Không downtime ngay cả khi 1 CGW fails
```

**Option 3: VPN + Direct Connect Backup** (enterprise)
```
Direct Connect (primary) ─┐
                          ├─→ VGW → VPC
VPN (failover) ───────────┘
```

---

## Bảo Mật & Mã Hóa

### Lớp Mã Hóa

```
Data: Application payload
  ↓ Encrypted by
ESP (Encapsulating Security Payload — Tải Trọng Bảo Mật Bọc)
  ↓ Authentication by
AH (Authentication Header) hoặc ESP-AUTH
  ↓ Tunneled inside
IP packet → Internet → AWS VGW
```

### Tunnel Options Tùy Chỉnh

```json
{
  "Phase1EncryptionAlgorithms": ["AES256-GCM-16"],
  "Phase2EncryptionAlgorithms": ["AES256-GCM-16"],
  "Phase1IntegrityAlgorithms": ["SHA2-384"],
  "Phase2IntegrityAlgorithms": ["SHA2-384"],
  "Phase1DHGroupNumbers": [20],
  "Phase2DHGroupNumbers": [20],
  "IKEVersions": ["ikev2"],
  "RekeyMarginTime": 540,
  "RekeyFuzzPercentage": 100,
  "ReplayWindowSize": 1024,
  "DPDTimeoutSeconds": 30,
  "DPDTimeoutAction": "restart"
}
```

### Best Practices Bảo Mật

```
✅ Luôn dùng IKEv2 thay vì IKEv1
✅ Dùng AES-256-GCM thay vì AES-256-CBC
✅ DH Group 14 trở lên (tránh Group 1, 2, 5)
✅ Pre-Shared Key tối thiểu 20 ký tự, ngẫu nhiên
✅ Lưu PSK trong AWS Secrets Manager
✅ Bật CloudWatch Logs cho VPN monitoring
✅ Dùng BGP thay vì static routing (dynamic, không lỗi thủ công)
✅ Hạn chế Security Group — chỉ cho phép traffic cần thiết từ on-premises
```

---

## Chi Phí

### Bảng Giá (US East — N. Virginia)

| Thành Phần | Đơn Giá | Ghi Chú |
|-----------|---------|---------|
| **VPN Connection** | $0.05/giờ | ~$36/tháng/connection |
| **Data Transfer Out** | $0.09/GB | Từ AWS → Internet |
| **Data Transfer In** | Miễn phí | Từ Internet → AWS |

### Ước Tính Chi Phí

```
Ví dụ: 1 VPN Connection, 1 TB/tháng data transfer out

VPN Connection:    $0.05 × 24h × 30 ngày = $36/tháng
Data Transfer:     $0.09 × 1024 GB       = $92.16/tháng
─────────────────────────────────────────────────────
Tổng:                                    ~$128/tháng

So với Direct Connect 1 Gbps:            ~$216/tháng (port) + data
→ VPN rẻ hơn đáng kể cho khối lượng nhỏ
```

---

## So Sánh & Trade-offs

### Site-to-Site VPN vs Direct Connect

| Tiêu Chí | Site-to-Site VPN | Direct Connect |
|---------|-----------------|----------------|
| **Băng thông** | 1.25 Gbps/tunnel | 1–100 Gbps |
| **Độ trễ** | Không đảm bảo (internet) | Thấp, ổn định |
| **Mã hóa** | IPsec (tích hợp sẵn) | Không có (cần thêm) |
| **Chi phí setup** | Không (pay-as-you-go) | Cao (NRC + port fee) |
| **Thời gian triển khai** | Vài giờ | Vài tuần đến tháng |
| **SLA** | Không đảm bảo | 99.99% (redundant) |
| **Tốt cho** | Băng thông nhỏ, DR, dev | Production enterprise |

### Site-to-Site VPN vs Client VPN

| Tiêu Chí | Site-to-Site VPN | Client VPN |
|---------|-----------------|------------|
| **Mục đích** | Kết nối toàn bộ mạng | Kết nối từng máy tính |
| **Phía on-premises** | Router/Firewall | Phần mềm VPN client |
| **Xác thực** | PSK hoặc Certificate | AD, MutualTLS, SAML |
| **Phù hợp** | Datacenter ↔ AWS | Remote worker ↔ AWS |

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích kiến trúc Site-to-Site VPN trên AWS

**Trả lời mẫu:**

> Site-to-Site VPN kết nối mạng on-premises với AWS VPC qua internet được mã hóa. Kiến trúc gồm 3 thành phần chính: **Customer Gateway (CGW)** đại diện cho thiết bị phía on-premises trong AWS Console, **Virtual Private Gateway (VGW)** là VPN concentrator phía AWS được attach vào VPC, và **VPN Connection** là tunnel IPsec nối hai bên. Mỗi connection luôn có 2 tunnels ở 2 Availability Zones khác nhau để đảm bảo high availability. Có thể dùng static routing hoặc BGP dynamic routing để trao đổi route giữa hai bên.

### Q2: Tại sao Site-to-Site VPN có 2 tunnels?

**Trả lời mẫu:**

> AWS cung cấp 2 tunnels cho mỗi VPN connection để đảm bảo high availability. Hai tunnels này terminate ở 2 Availability Zones khác nhau phía AWS, nên nếu một AZ có sự cố, traffic tự động failover sang tunnel còn lại. AWS khuyến nghị cấu hình on-premises router để dùng cả 2 tunnels ở chế độ active-active thay vì active-passive để tận dụng tối đa băng thông (tổng cộng 2.5 Gbps thay vì 1.25 Gbps) và giảm thời gian failover.

### Q3: Khi nào nên dùng VPN thay vì Direct Connect?

**Trả lời mẫu:**

> Tôi chọn VPN khi: cần triển khai nhanh (vài giờ so với vài tuần của Direct Connect), ngân sách hạn chế, băng thông yêu cầu dưới 1 Gbps, hoặc làm backup/failover cho Direct Connect. Direct Connect phù hợp hơn khi cần băng thông lớn ổn định (1–100 Gbps), độ trễ thấp nhất quán (cho database replication hoặc real-time workloads), hoặc khi môi trường yêu cầu đường truyền riêng (không qua internet công cộng) cho compliance.

### Q4: Troubleshooting khi VPN tunnel ở trạng thái DOWN

**Checklist chẩn đoán:**

```
1. Kiểm tra IKE negotiation:
   - Firewall on-premises có chặn UDP 500 và 4500 không?
   - Pre-Shared Key có khớp giữa 2 bên không?
   - IKE version có nhất quán (IKEv1 vs IKEv2)?

2. Kiểm tra cấu hình mã hóa:
   - Encryption algorithms có khớp không?
   - DH Group có tương thích không?

3. Kiểm tra routing:
   - BGP session có UP không?
   - Routes có được advertise đúng không?
   - Route propagation đã bật trong VPC route table chưa?

4. Kiểm tra Security Groups và NACL:
   - Có cho phép traffic từ on-premises IP range không?

5. Dùng CloudWatch metrics:
   - TunnelState: 0 = DOWN, 1 = UP
   - TunnelDataIn/Out: 0 → không có traffic
```

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Section | [06-connectivity/README.md](./README.md) |
| → Tiếp theo | [2-client-vpn.md](./2-client-vpn.md) |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
