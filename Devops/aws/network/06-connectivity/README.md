# 06 — Connectivity — Kết Nối Hybrid & On-Premises

> Tổng quan về các giải pháp kết nối AWS với môi trường on-premises (tại chỗ) và multi-cloud (đa đám mây).

---

## 📚 Mục Lục Section Này

| File | Chủ Đề | Mức Độ |
|------|---------|--------|
| [1-site-to-site-vpn.md](./1-site-to-site-vpn.md) | Site-to-Site VPN — VPN Địa Điểm-đến-Địa Điểm | ⭐⭐ |
| [2-client-vpn.md](./2-client-vpn.md) | Client VPN — VPN Truy Cập Từ Xa | ⭐⭐ |
| [3-direct-connect.md](./3-direct-connect.md) | Direct Connect — Kết Nối Chuyên Dụng | ⭐⭐⭐ |
| [4-direct-connect-gateway.md](./4-direct-connect-gateway.md) | Direct Connect Gateway — Cổng Kết Nối Chuyên Dụng | ⭐⭐⭐ |
| [5-transit-gateway.md](./5-transit-gateway.md) | Transit Gateway — Cổng Trung Chuyển Hub-and-Spoke | ⭐⭐⭐ |

---

## 🎯 Tại Sao Connectivity Quan Trọng?

### Thực Tế Doanh Nghiệp

Hầu hết các doanh nghiệp lớn **không di chuyển 100% lên cloud** ngay lập tức. Thay vào đó, họ vận hành mô hình **Hybrid Cloud** (Đám Mây Lai) trong đó:

- **On-premises datacenter** vẫn chứa hệ thống legacy (di sản), database lịch sử, ứng dụng không thể cloud-native
- **AWS Cloud** chứa workloads (khối lượng công việc) mới, ứng dụng web, API, analytics
- **Cả hai môi trường phải giao tiếp** một cách bảo mật, đáng tin cậy, hiệu suất cao

### Hai Câu Hỏi Cốt Lõi

```
1. Làm thế nào để on-premises kết nối vào AWS VPC một cách bảo mật?
   → VPN Site-to-Site hoặc Direct Connect

2. Làm thế nào để quản lý kết nối khi có hàng chục VPCs và locations?
   → Transit Gateway
```

---

## 🗺️ Bản Đồ Quyết Định Connectivity

```
Cần kết nối on-premises ↔ AWS?
│
├─ Ngân sách thấp / Thiết lập nhanh / Backup link?
│  └─→ Site-to-Site VPN
│       • Mã hóa qua internet công cộng (IPsec)
│       • Thiết lập trong giờ, không cần phần cứng đặc biệt
│       • Băng thông: tối đa 1.25 Gbps/tunnel
│
├─ Yêu cầu băng thông cao / Độ trễ thấp / SLA mạnh?
│  └─→ Direct Connect
│       • Đường truyền riêng, không qua internet
│       • Băng thông: 1 Gbps đến 100 Gbps
│       • Thời gian thiết lập: tuần đến tháng
│
└─ Cả hai (redundancy / failover)?
   └─→ Direct Connect + VPN backup
        • DX làm primary, VPN làm failover tự động

Cần kết nối nhiều VPCs với nhau?
│
├─ 2-3 VPCs, traffic đơn giản?
│  └─→ VPC Peering (Kết Nối Ngang Hàng)
│
└─ Nhiều VPCs (5+), cần centralized routing?
   └─→ Transit Gateway (TGW)
        • Hub-and-spoke model
        • Kết nối VPCs, VPN, Direct Connect qua 1 điểm
```

---

## 🏗️ Kiến Trúc Hybrid Cloud Điển Hình

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS Cloud                                    │
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │   VPC Dev   │    │  VPC Prod   │    │  VPC Shared │        │
│  │ 10.1.0.0/16 │    │ 10.2.0.0/16 │    │ 10.3.0.0/16 │        │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│         │                  │                  │                │
│         └──────────────────┼──────────────────┘                │
│                            │                                   │
│                   ┌────────▼────────┐                          │
│                   │ Transit Gateway │                          │
│                   │   (TGW Hub)     │                          │
│                   └────────┬────────┘                          │
│                            │                                   │
│          ┌─────────────────┼─────────────────┐                │
│          │                 │                 │                │
│    ┌─────▼──────┐   ┌──────▼─────┐   ┌──────▼─────┐         │
│    │  VPN Conn  │   │  DX Attach │   │  VPN Conn  │         │
│    │  (IPsec)   │   │ (Private)  │   │  (Backup)  │         │
│    └─────┬──────┘   └──────┬─────┘   └────────────┘         │
└──────────┼─────────────────┼──────────────────────────────────┘
           │                 │
           │      Internet   │   AWS Direct Connect Location
           │                 │   (Co-location facility)
           │                 │
┌──────────▼─────────────────▼──────────────────────────────────┐
│                    On-Premises Datacenter                       │
│                                                                 │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  Customer   │    │   Router /   │    │   Servers,       │  │
│  │  Gateway    │    │   Firewall   │    │   Databases,     │  │
│  │  (CGW)      │    │              │    │   Legacy Apps    │  │
│  └─────────────┘    └──────────────┘    └──────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## 📡 Tổng Quan Các Dịch Vụ

### 1. Site-to-Site VPN

| Thông Số | Giá Trị |
|----------|---------|
| **Giao thức** | IPsec (Internet Protocol Security) |
| **Mã hóa** | AES-256, SHA-2 |
| **Băng thông tối đa** | 1.25 Gbps/tunnel (2 tunnels/connection) |
| **Độ trễ** | Phụ thuộc internet (20-100ms+) |
| **Chi phí** | ~$0.05/giờ/connection + data transfer |
| **Thời gian setup** | Vài giờ |
| **SLA** | Không đảm bảo (qua internet) |

### 2. Client VPN

| Thông Số | Giá Trị |
|----------|---------|
| **Giao thức** | OpenVPN (TLS 1.2+) |
| **Xác thực** | Active Directory, Mutual TLS, SAML 2.0 |
| **Client OS** | Windows, macOS, Linux, iOS, Android |
| **Chi phí** | ~$0.10/giờ endpoint + $0.05/giờ/association |
| **Phù hợp** | Remote workers, developers |

### 3. AWS Direct Connect

| Thông Số | Giá Trị |
|----------|---------|
| **Loại kết nối** | Dedicated (1/10/100 Gbps) hoặc Hosted (50 Mbps–10 Gbps) |
| **Giao thức** | BGP (Border Gateway Protocol) |
| **Mã hóa** | Không mặc định (cần MACsec hoặc VPN over DX) |
| **Độ trễ** | Thấp và ổn định (single-digit ms trong cùng region) |
| **Chi phí** | Cao — port fee + data transfer |
| **Thời gian setup** | Vài tuần đến vài tháng |
| **SLA** | 99.99% với redundant connections |

### 4. Direct Connect Gateway

| Thông Số | Giá Trị |
|----------|---------|
| **Mục đích** | Kết nối 1 DX connection đến nhiều VPCs hoặc nhiều Regions |
| **Phạm vi** | Global (không bị giới hạn Region) |
| **VGW limit** | Tối đa 10 VGWs (Virtual Private Gateways) |
| **TGW integration** | Kết nối được với Transit Gateway |
| **Chi phí** | Không phát sinh thêm (trả theo DX port) |

### 5. Transit Gateway

| Thông Số | Giá Trị |
|----------|---------|
| **Model** | Hub-and-spoke (nan hoa) |
| **VPC attachments** | Tối đa 5,000 VPCs/TGW |
| **Bandwidth** | Tối đa 50 Gbps/VPC attachment |
| **Route tables** | Multiple — hỗ trợ segmentation |
| **Chi phí** | ~$0.05/giờ/attachment + $0.02/GB data |
| **Cross-region** | Có (TGW Peering) |

---

## 🔐 So Sánh Bảo Mật

| Tiêu Chí | Site-to-Site VPN | Direct Connect | Direct Connect + MACsec |
|---------|-----------------|----------------|------------------------|
| **Mã hóa** | IPsec AES-256 ✅ | Không có ❌ | MACsec (Layer 2) ✅ |
| **Đường truyền** | Internet công cộng | Riêng tư | Riêng tư + mã hóa |
| **MITM risk** | Thấp (mã hóa) | Trung bình | Rất thấp |
| **Compliance** | PCI-DSS, HIPAA | Cần thêm mã hóa | PCI-DSS, HIPAA ✅ |

> **Best Practice:** Với Direct Connect trong môi trường regulated (tuân thủ quy định), luôn bật **MACsec** hoặc chạy **VPN tunnel over Direct Connect**.

---

## 💡 Quyết Định Kiến Trúc Phổ Biến

### Câu Hỏi Phỏng Vấn Hay Gặp

**"Direct Connect hay VPN — khi nào dùng cái nào?"**

```
VPN Site-to-Site phù hợp khi:
✅ Ngân sách hạn chế (chi phí thấp hơn nhiều)
✅ Cần triển khai nhanh (vài giờ vs vài tuần)
✅ Backup/failover cho Direct Connect
✅ Băng thông <1 Gbps là đủ
✅ Không có yêu cầu SLA nghiêm ngặt

Direct Connect phù hợp khi:
✅ Băng thông lớn, ổn định (1–100 Gbps)
✅ Độ trễ thấp, nhất quán (database replication, real-time)
✅ Chi phí data transfer thấp hơn (với khối lượng lớn)
✅ Môi trường regulated cần đường truyền riêng
✅ SLA yêu cầu 99.99% uptime
```

**"VPC Peering hay Transit Gateway?"**

```
VPC Peering phù hợp khi:
✅ Ít VPCs (2-4)
✅ Traffic đơn giản, không cần routing phức tạp
✅ Chi phí quan trọng (Peering không phí attachment)

Transit Gateway phù hợp khi:
✅ Nhiều VPCs (5+)
✅ Cần centralized routing/firewall
✅ Kết nối VPN, Direct Connect vào nhiều VPCs
✅ Network segmentation với route tables khác nhau
✅ Cross-region connectivity
```

---

## 🧪 Hands-On Labs Gợi Ý

### Lab 1 — Tạo Site-to-Site VPN với Simulated CGW

```bash
# 1. Tạo Customer Gateway (CGW)
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip <YOUR_PUBLIC_IP> \
  --bgp-asn 65000

# 2. Tạo Virtual Private Gateway (VGW)
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512

# 3. Attach VGW vào VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id <VGW_ID> \
  --vpc-id <VPC_ID>

# 4. Tạo VPN Connection
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id <CGW_ID> \
  --vpn-gateway-id <VGW_ID>
```

### Lab 2 — Tạo Transit Gateway và Kết Nối VPCs

```bash
# 1. Tạo Transit Gateway
aws ec2 create-transit-gateway \
  --description "Main TGW" \
  --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable

# 2. Attach VPC vào TGW
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id <TGW_ID> \
  --vpc-id <VPC_ID> \
  --subnet-ids <SUBNET_ID_1> <SUBNET_ID_2>

# 3. Xem route tables của TGW
aws ec2 describe-transit-gateway-route-tables \
  --filters Name=transit-gateway-id,Values=<TGW_ID>
```

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [05-cdn-cloudfront/](../05-cdn-cloudfront/README.md) |
| → Tiếp theo | [07-advanced-networking/](../07-advanced-networking/README.md) |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
