# 4 — Direct Connect Gateway — Cổng Kết Nối Chuyên Dụng

> AWS Direct Connect Gateway (DX Gateway) — dịch vụ globally distributed (phân tán toàn cầu) cho phép kết nối một AWS Direct Connect connection với nhiều VPCs ở nhiều AWS Regions, giải quyết bài toán nhiều-đến-nhiều trong kiến trúc hybrid cloud.

---

## 📋 Mục Lục

1. [Tổng Quan & Vấn Đề Giải Quyết](#tổng-quan--vấn-đề-giải-quyết)
2. [Kiến Trúc & Thành Phần](#kiến-trúc--thành-phần)
3. [Loại Kết Nối Với DX Gateway](#loại-kết-nối-với-dx-gateway)
4. [Cấu Hình & Triển Khai](#cấu-hình--triển-khai)
5. [DX Gateway + Transit Gateway](#dx-gateway--transit-gateway)
6. [Routing & BGP với DX Gateway](#routing--bgp-với-dx-gateway)
7. [Giới Hạn & Quotas](#giới-hạn--quotas)
8. [Chi Phí](#chi-phí)
9. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan & Vấn Đề Giải Quyết

### Vấn Đề Không Có DX Gateway

Trước khi DX Gateway ra đời, mỗi VPC trong mỗi Region cần **Private VIF riêng** cho cùng một DX connection:

```
Trước DX Gateway (Không Tối Ưu):
                          ┌── Private VIF 1 ──► VGW-A ──► VPC-A (us-east-1)
                          ├── Private VIF 2 ──► VGW-B ──► VPC-B (us-east-1)
DX Connection ────────────┤
                          ├── Private VIF 3 ──► VGW-C ──► VPC-C (eu-west-1)
                          └── Private VIF 4 ──► VGW-D ──► VPC-D (ap-southeast-1)

Vấn đề:
❌ Tốn nhiều VIFs (limited slot)
❌ Phức tạp, khó quản lý
❌ Không scalable khi thêm VPCs
❌ Chi phí quản lý cao
```

### Giải Pháp Với DX Gateway

```
Sau DX Gateway (Tối Ưu):
                            ┌──► VGW-A ──► VPC-A (us-east-1)
                            ├──► VGW-B ──► VPC-B (us-east-1)
DX Connection ─[1 Private VIF]─► DX Gateway ─┤
                            ├──► VGW-C ──► VPC-C (eu-west-1)
                            └──► VGW-D ──► VPC-D (ap-southeast-1)

Lợi ích:
✅ 1 VIF duy nhất kết nối nhiều VPCs
✅ Kết nối được nhiều Regions (global resource)
✅ Dễ quản lý, scalable
✅ Không phí thêm cho DX Gateway
```

---

## Kiến Trúc & Thành Phần

### Sơ Đồ Kiến Trúc Đầy Đủ

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Global Infrastructure                    │
│                                                                     │
│  ┌──────────────────┐      ┌──────────────────────────────────────┐ │
│  │   us-east-1      │      │          eu-west-1                   │ │
│  │                  │      │                                      │ │
│  │  ┌──────────┐    │      │   ┌──────────┐   ┌──────────┐       │ │
│  │  │  VPC-A   │    │      │   │  VPC-C   │   │  VPC-D   │       │ │
│  │  │10.0.0/16 │    │      │   │10.2.0/16 │   │10.3.0/16 │       │ │
│  │  └────┬─────┘    │      │   └────┬─────┘   └────┬─────┘       │ │
│  │       │          │      │        │               │             │ │
│  │  ┌────▼─────┐    │      │   ┌────▼─────────────▼──────────┐   │ │
│  │  │  VGW-A   │    │      │   │        Transit Gateway        │   │ │
│  │  └────┬─────┘    │      │   │        (eu-west-1 TGW)        │   │ │
│  │       │          │      │   └───────────────┬───────────────┘   │ │
│  └───────┼──────────┘      └───────────────────┼───────────────────┘ │
│          │                                     │                   │
│          └──────────────────┬──────────────────┘                   │
│                             │                                       │
│              ┌──────────────▼──────────────────┐                   │
│              │         Direct Connect Gateway   │                   │
│              │         (DX GW — Global)         │                   │
│              └──────────────┬───────────────────┘                   │
└─────────────────────────────┼───────────────────────────────────────┘
                              │
                    Private VIF (1 VIF duy nhất)
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                     Direct Connect Location                         │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │  DX Connection (1 Gbps hoặc 10 Gbps dedicated)             │  │
│   └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                      On-Premises Datacenter                         │
│                     192.168.0.0/16                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Thành Phần Chi Tiết

**Direct Connect Gateway (DX GW)**
- **Global resource** — không thuộc về một Region cụ thể
- Được tạo trong một AWS Account, có thể chia sẻ qua **AWS RAM** (Resource Access Manager)
- Không phải gateway vật lý — là control plane resource logic
- 1 DX GW có thể associated với tối đa **10 VGWs** hoặc **3 Transit Gateways**

**Private VIF ↔ DX Gateway Association**
- Chỉ cần 1 Private VIF cho toàn bộ DX GW
- BGP session được thiết lập giữa on-premises router và DX GW
- DX GW route traffic đến đúng VGW/TGW

---

## Loại Kết Nối Với DX Gateway

### Model 1: DX GW + Multiple VGWs

```
Use Case: Nhiều VPCs riêng lẻ trong nhiều Regions

DX Connection → Private VIF → DX GW ──► VGW-1 → VPC-A (us-east-1)
                                      ├──► VGW-2 → VPC-B (us-east-1)
                                      ├──► VGW-3 → VPC-C (eu-west-1)
                                      └──► ... (tối đa 10 VGWs)

Giới hạn:
- Tối đa 10 VGW associations per DX GW
- VPCs không thể kết nối với nhau qua DX GW (chỉ on-premises ↔ VPC)
- Cần VPC Peering hoặc Transit Gateway nếu VPCs cần giao tiếp với nhau
```

### Model 2: DX GW + Transit Gateway (Khuyến Nghị Cho Scale Lớn)

```
Use Case: Nhiều VPCs, cần VPC-to-VPC routing, multi-region

DX Connection → Transit VIF → DX GW ──► TGW-us-east → VPC-A, VPC-B
                                      └──► TGW-eu-west → VPC-C, VPC-D

Ưu điểm:
✅ TGW kết nối không giới hạn VPCs (5000+)
✅ VPCs có thể giao tiếp với nhau qua TGW
✅ Network segmentation với TGW route tables
✅ Scalable hơn nhiều
```

---

## Cấu Hình & Triển Khai

### Tạo DX Gateway và Associate VGW

```bash
# Bước 1: Tạo Direct Connect Gateway
aws directconnect create-direct-connect-gateway \
  --direct-connect-gateway-name "Enterprise-DXGW" \
  --amazon-side-asn 64512

# Output: directConnectGatewayId = "dxgw-xxxxxxxxxx"

# Bước 2: Tạo VGW và Attach vào VPC
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64513

aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-xxxxxxxxx \
  --vpc-id vpc-xxxxxxxxx

# Bước 3: Associate VGW với DX Gateway
aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id dxgw-xxxxxxxxxx \
  --gateway-id vgw-xxxxxxxxx \
  --add-allowed-prefixes-to-direct-connect-gateway \
    '[{"cidr": "10.0.0.0/16"}]'

# Bước 4: Tạo Private VIF và Associate với DX GW
aws directconnect create-private-virtual-interface \
  --connection-id dxcon-xxxxxxxxxx \
  --new-private-virtual-interface '{
    "virtualInterfaceName": "Private-VIF-to-DXGW",
    "vlan": 100,
    "asn": 65000,
    "directConnectGatewayId": "dxgw-xxxxxxxxxx",
    "authKey": "bgp-auth-key-optional",
    "amazonAddress": "169.254.255.2/30",
    "customerAddress": "169.254.255.1/30",
    "addressFamily": "ipv4"
  }'

# Kiểm tra trạng thái
aws directconnect describe-direct-connect-gateways \
  --direct-connect-gateway-id dxgw-xxxxxxxxxx
```

### Cấu Hình Router Phía On-Premises (Cisco IOS Ví Dụ)

```
interface GigabitEthernet0/1.100
  description "DX Connection to AWS"
  encapsulation dot1Q 100
  ip address 169.254.255.1 255.255.255.252
!
router bgp 65000
  neighbor 169.254.255.2 remote-as 64512
  neighbor 169.254.255.2 description "AWS Direct Connect Gateway"
  neighbor 169.254.255.2 password bgp-auth-key-optional
  !
  address-family ipv4
    neighbor 169.254.255.2 activate
    network 192.168.0.0 mask 255.255.0.0
    neighbor 169.254.255.2 soft-reconfiguration inbound
  exit-address-family
```

---

## DX Gateway + Transit Gateway

### Kiến Trúc Kết Hợp

```
On-Premises ──[DX]──► [Private VIF / Transit VIF] ──► DX GW ──► TGW
                                                                   │
                                                               ┌───┼───┐
                                                               │   │   │
                                                           VPC-A VPC-B VPC-C
```

### Association DX GW với Transit Gateway

```bash
# Associate DX GW với Transit Gateway
aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id dxgw-xxxxxxxxxx \
  --gateway-id tgw-xxxxxxxxxx \
  --add-allowed-prefixes-to-direct-connect-gateway \
    '[
      {"cidr": "10.0.0.0/8"},
      {"cidr": "172.16.0.0/12"}
    ]'
```

### Routing Với TGW Integration

```
BGP Route Advertisement Flow:

On-Premises → DX GW → TGW Route Table → VPCs
(192.168.0.0/16) → (advertised) → (learned) → (reachable)

VPCs → TGW Route Table → DX GW → On-Premises
(10.x.x.x/16 từ mỗi VPC) → (propagated) → (advertised) → (reachable)

Allowed Prefixes (Tiền Tố Được Phép):
- Danh sách CIDR ranges mà DX GW sẽ advertise vào on-premises
- Phải khai báo tường minh khi tạo association
- Security control: on-premises chỉ biết những prefix bạn cho phép
```

---

## Routing & BGP với DX Gateway

### Allowed Prefixes — Tiền Tố Được Phép

Đây là tính năng quan trọng của DX Gateway:

```
Khái niệm: Kiểm soát CIDR nào được quảng bá từ AWS → on-premises

Ví dụ:
VPC-A: 10.0.0.0/16
VPC-B: 10.1.0.0/16
VPC-C: 10.2.0.0/16

Allowed Prefixes trong DX GW association:
- Nếu set "10.0.0.0/8" → on-premises thấy 10.0.0.0/8 (summarized)
- Nếu set từng /16 → on-premises thấy 3 routes riêng lẻ
- Nếu không set → BGP không advertise routes (kết nối nhưng không route được)
```

### BGP Route Selection — Chọn Đường Đi

```
Khi có nhiều đường (DX + VPN Backup):

DX path: Ưu tiên bằng BGP LOCAL_PREF cao
VPN path: LOCAL_PREF thấp hơn → dự phòng

Cấu hình (Cisco):
route-map SET_DX_PREF permit 10
  set local-preference 200
!
route-map SET_VPN_PREF permit 10
  set local-preference 100
!
router bgp 65000
  neighbor 169.254.255.2 route-map SET_DX_PREF in   ! DX neighbor
  neighbor 169.254.100.2 route-map SET_VPN_PREF in  ! VPN neighbor
```

---

## Giới Hạn & Quotas

| Tham Số | Giới Hạn Mặc Định |
|---------|------------------|
| VGW per DX GW | 10 |
| Transit Gateway per DX GW | 3 |
| DX GW per AWS account | 200 |
| Prefixes from on-premises per VIF | 100 |
| Prefixes advertised from AWS per VIF | 1,000 |
| VIF per DX dedicated connection | 50 |
| VIF per DX hosted connection | 1 |

> **Quan trọng:** VGW limit 10 là giới hạn hay gặp trong thực tế. Khi cần nhiều hơn 10 VPCs, chuyển sang mô hình DX GW + Transit Gateway.

---

## Chi Phí

### DX Gateway Không Tính Phí Riêng

DX Gateway là **free** — không có phí cho bản thân dịch vụ. Chi phí phát sinh từ:

```
1. DX Connection port fee: $0.30/giờ (1 Gbps dedicated)
2. Data transfer out (AWS → on-premises): $0.02/GB
3. Transit Gateway attachment fee (nếu dùng TGW): $0.05/giờ/attachment
```

### Multi-Region Traffic Cost

```
Lưu ý: Traffic đi qua DX GW từ Region này sang Region khác có thể phát sinh inter-region data transfer charge (phí chuyển vùng)

Ví dụ:
On-premises → DX → DX GW → VPC (us-east-1): $0.02/GB (standard DX rate)
On-premises → DX → DX GW → VPC (eu-west-1): $0.02/GB + inter-region fee
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Tại sao cần Direct Connect Gateway, Direct Connect thôi chưa đủ sao?

**Trả lời mẫu:**

> Direct Connect đơn thuần chỉ kết nối được với một Region và cần Private VIF riêng cho mỗi VPC. Khi doanh nghiệp có nhiều VPCs ở nhiều Regions, mỗi VPC sẽ cần VIF riêng — vừa tốn VIF slots (giới hạn 50/connection), vừa phức tạp để quản lý. Direct Connect Gateway giải quyết bằng cách tập trung: chỉ cần 1 Private VIF duy nhất kết nối đến DX GW, rồi DX GW route traffic đến tối đa 10 VGWs ở bất kỳ Region nào. Đây là global resource, không bị giới hạn bởi Region. Với scale lớn hơn, DX GW kết hợp với Transit Gateway cho phép kết nối không giới hạn VPCs.

### Q2: Sự khác biệt giữa Private VIF và Transit VIF khi dùng với DX Gateway

**Trả lời mẫu:**

> Private VIF được dùng khi muốn kết nối DX GW với VGW (Virtual Private Gateway) — mỗi VGW đại diện cho một VPC. Giới hạn là 10 VGWs per DX GW. Transit VIF được dùng khi muốn kết nối DX GW với Transit Gateway — một TGW có thể kết nối đến hàng nghìn VPCs. Transit VIF là lựa chọn cho môi trường enterprise quy mô lớn. Điểm khác biệt quan trọng: Transit VIF không thể associate với VGW trực tiếp, chỉ được dùng với Transit Gateway.

### Q3: Allowed Prefixes trong DX Gateway Association là gì?

**Trả lời mẫu:**

> Allowed Prefixes là danh sách CIDR ranges mà Direct Connect Gateway được phép quảng bá từ AWS về phía on-premises qua BGP. Khi bạn associate VGW hoặc TGW với DX GW, bạn phải khai báo tường minh những prefixes này. Ví dụ, nếu VPC có CIDR 10.0.0.0/16 nhưng bạn chỉ khai báo 10.0.1.0/24 trong allowed prefixes, thì on-premises sẽ chỉ biết route đến subnet đó, không phải toàn bộ VPC. Đây là cơ chế bảo mật quan trọng — kiểm soát chính xác những gì on-premises có thể reach trong AWS.

### Q4: Thiết kế kết nối cho 50 VPCs ở 3 Regions qua 1 DX Connection

**Đáp án kiến trúc:**

```
1 DX Connection (10 Gbps dedicated)
  └──► 1 Transit VIF
         └──► Direct Connect Gateway (DX GW)
               ├──► Transit Gateway (us-east-1) ──► 20 VPCs
               ├──► Transit Gateway (eu-west-1) ──► 15 VPCs
               └──► Transit Gateway (ap-southeast-1) ──► 15 VPCs

Tại sao không dùng Private VIF + VGW?
- 50 VPCs cần 50 VGWs → vượt giới hạn 10 VGWs/DX GW
- Không có inter-VPC routing nếu cần

Với Transit Gateway:
- TGW route table kiểm soát VPC nào kết nối được với nhau
- On-premises kết nối qua DX GW → TGW → tất cả VPCs
- Có thể dùng TGW Peering để kết nối 3 TGWs với nhau
```

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [3-direct-connect.md](./3-direct-connect.md) |
| → Tiếp theo | [5-transit-gateway.md](./5-transit-gateway.md) |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
