# 5 — Transit Gateway — Cổng Trung Chuyển Hub-and-Spoke

> AWS Transit Gateway (TGW) — dịch vụ trung tâm kết nối (network hub) cho phép kết nối hàng nghìn VPCs, VPN connections, và Direct Connect gateways qua một điểm trung tâm duy nhất, theo mô hình hub-and-spoke (nan hoa — trung tâm và các nhánh).

---

## 📋 Mục Lục

1. [Tổng Quan & Vấn Đề Giải Quyết](#tổng-quan--vấn-đề-giải-quyết)
2. [Kiến Trúc Hub-and-Spoke](#kiến-trúc-hub-and-spoke)
3. [Các Loại Attachment — Gắn Kết](#các-loại-attachment--gắn-kết)
4. [Transit Gateway Route Tables](#transit-gateway-route-tables)
5. [Network Segmentation — Phân Vùng Mạng](#network-segmentation--phân-vùng-mạng)
6. [Transit Gateway Peering — Liên Kết Đa Vùng](#transit-gateway-peering--liên-kết-đa-vùng)
7. [Cấu Hình & Triển Khai](#cấu-hình--triển-khai)
8. [Transit Gateway vs VPC Peering](#transit-gateway-vs-vpc-peering)
9. [Chi Phí](#chi-phí)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## Tổng Quan & Vấn Đề Giải Quyết

### Vấn Đề Với VPC Peering

Khi có nhiều VPCs cần kết nối với nhau, VPC Peering tạo ra mạng lưới kết nối phức tạp:

```
VPC Peering — Full Mesh (Lưới Đầy Đủ) với 4 VPCs:
Cần 6 peering connections riêng lẻ

VPC-A ─────────────── VPC-B
  │  \               /  │
  │    \           /    │
  │      \       /      │
  │        \   /        │
  │          X          │
  │        /   \        │
  │      /       \      │
VPC-C ─────────────── VPC-D

Số connections cần thiết = n×(n-1)/2
- 4 VPCs → 6 peering connections
- 10 VPCs → 45 peering connections
- 50 VPCs → 1,225 peering connections 😱

Vấn đề:
❌ Không scalable
❌ Không có transitive routing (VPC-A không thể reach VPC-D qua VPC-B)
❌ Routing tables phức tạp
❌ Quản lý rất khó khi scale lớn
```

### Giải Pháp Với Transit Gateway

```
Transit Gateway — Hub-and-Spoke (Nan Hoa):

         VPC-A
           │
VPC-D ─────┼───── VPC-B
           │
         VPC-C

Tất cả kết nối qua TGW ở trung tâm.
Số attachments = n (tuyến tính, không phải n²)

- 4 VPCs → 4 attachments
- 10 VPCs → 10 attachments
- 50 VPCs → 50 attachments ✅

Ưu điểm:
✅ Scalable: tối đa 5,000 VPC attachments
✅ Transitive routing: VPC-A → TGW → VPC-C
✅ Centralized routing management
✅ Kết nối được VPN, Direct Connect, TGW Peering
✅ Network segmentation với multiple route tables
```

---

## Kiến Trúc Hub-and-Spoke

### Sơ Đồ Toàn Diện

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Region (us-east-1)                     │
│                                                                     │
│  VPC-Production   VPC-Dev   VPC-Staging    VPC-Shared-Services     │
│  10.0.0.0/16    10.1.0.0/16 10.2.0.0/16    10.3.0.0/16           │
│       │               │          │               │                 │
│       └───────────────┼──────────┼───────────────┘                 │
│                       │          │                                 │
│              ┌────────▼──────────▼──────────────────────────────┐  │
│              │                                                   │  │
│              │              Transit Gateway                      │  │
│              │           (TGW — Hub Center)                     │  │
│              │                                                   │  │
│              │  Route Tables:                                    │  │
│              │  ┌────────────────┐  ┌────────────────────────┐  │  │
│              │  │ Production RT  │  │  Non-Prod RT           │  │  │
│              │  │ Routes: Shared │  │  Routes: Shared only   │  │  │
│              │  │         VPN    │  │  (Dev/Staging isolated)│  │  │
│              │  └────────────────┘  └────────────────────────┘  │  │
│              │                                                   │  │
│              └────────┬──────────────────────┬───────────────────┘  │
│                       │                      │                      │
│              VPN Attachment          DX Gateway Attachment           │
└───────────────────────┼──────────────────────┼──────────────────────┘
                        │                      │
               Site-to-Site VPN       Direct Connect
               (IPsec Tunnel)         (Private/Transit VIF)
                        │                      │
┌───────────────────────▼──────────────────────▼──────────────────────┐
│                      On-Premises Datacenter                         │
│                      192.168.0.0/16                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Các Loại Attachment — Gắn Kết

### 1. VPC Attachment

```bash
# Tạo VPC attachment
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-xxxxxxxxxx \
  --vpc-id vpc-xxxxxxxxxx \
  --subnet-ids subnet-aaa subnet-bbb \  # 1 subnet/AZ — khuyến nghị
  --options '{
    "DnsSupport": "enable",
    "Ipv6Support": "disable",
    "ApplianceModeSupport": "disable"
  }' \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=VPC-Prod-Attachment}]'
```

**Lưu ý:** Mỗi VPC attachment cần ít nhất 1 subnet per AZ. TGW tự động tạo ENI (Elastic Network Interface — Giao Diện Mạng Đàn Hồi) trong subnet đó.

### 2. VPN Attachment

```bash
# Tạo Customer Gateway trước
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.10 \
  --bgp-asn 65000

# Tạo VPN connection gắn vào TGW (không phải VGW)
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxxxxxxxxx \
  --transit-gateway-id tgw-xxxxxxxxxx \   # Chú ý: transit-gateway-id, không phải vpn-gateway-id
  --options StaticRoutesOnly=false
```

### 3. Direct Connect Gateway Attachment

```bash
# Associate DX GW với TGW
aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id dxgw-xxxxxxxxxx \
  --gateway-id tgw-xxxxxxxxxx \
  --add-allowed-prefixes-to-direct-connect-gateway \
    '[{"cidr": "10.0.0.0/8"}]'
```

### 4. TGW Peering Attachment — Kết Nối Liên Region

```bash
# Tạo peering request từ us-east-1 (requester) đến eu-west-1 (accepter)
aws ec2 create-transit-gateway-peering-attachment \
  --transit-gateway-id tgw-us-east-1-id \
  --peer-transit-gateway-id tgw-eu-west-1-id \
  --peer-account-id 123456789012 \
  --peer-region eu-west-1 \
  --region us-east-1

# Accept từ eu-west-1
aws ec2 accept-transit-gateway-peering-attachment \
  --transit-gateway-attachment-id tgw-attach-xxxxxxxxxx \
  --region eu-west-1
```

---

## Transit Gateway Route Tables

### Khái Niệm

Transit Gateway sử dụng **route tables riêng** (khác với VPC route tables):

```
TGW Route Table:
┌──────────────────────────────────────────────────────────┐
│ Destination CIDR    │ Attachment           │ Type        │
├──────────────────────────────────────────────────────────┤
│ 10.0.0.0/16         │ vpc-prod-attachment  │ Propagated  │
│ 10.1.0.0/16         │ vpc-dev-attachment   │ Propagated  │
│ 192.168.0.0/16      │ vpn-attachment       │ Propagated  │
│ 0.0.0.0/0           │ vpc-shared-attachment│ Static      │
└──────────────────────────────────────────────────────────┘
```

### Route Propagation vs Static Routes

**Route Propagation — Lan Truyền Route Tự Động:**
- VPC attachments tự động quảng bá CIDR của VPC vào TGW route table
- VPN/DX attachments quảng bá routes qua BGP
- Cần bật propagation từ attachment vào route table

**Static Routes — Route Tĩnh:**
- Thêm thủ công: "Gửi traffic 0.0.0.0/0 qua VPC Shared"
- Phù hợp: Default route, blackhole route (chặn traffic)

```bash
# Tạo static route trong TGW route table
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-xxxxxxxxxx \
  --destination-cidr-block "0.0.0.0/0" \
  --transit-gateway-attachment-id tgw-attach-shared-vpc

# Tạo blackhole route (chặn traffic đến subnet này)
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id tgw-rtb-xxxxxxxxxx \
  --destination-cidr-block "10.99.0.0/16" \
  --blackhole

# Bật route propagation
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id tgw-rtb-xxxxxxxxxx \
  --transit-gateway-attachment-id tgw-attach-vpc-prod
```

---

## Network Segmentation — Phân Vùng Mạng

### Pattern: Isolated VPCs (Cô Lập VPC)

Kịch bản: Dev và Prod không được nói chuyện với nhau, nhưng cả hai đều kết nối được vào Shared Services (DNS, Monitoring, Bastion).

```
┌─────────────────────────────────────────────────────────────────┐
│                     Transit Gateway                             │
│                                                                 │
│   ┌─────────────────────────┐  ┌─────────────────────────────┐ │
│   │   Production RT         │  │      Dev/Test RT            │ │
│   │                         │  │                             │ │
│   │ Propagations:           │  │ Propagations:               │ │
│   │   - vpc-prod (10.0.0/16)│  │   - vpc-dev (10.1.0.0/16)  │ │
│   │   - vpc-shared (10.3/16)│  │   - vpc-shared (10.3.0.0/16)│ │
│   │   - vpn-on-prem         │  │                             │ │
│   │                         │  │ (VPN, Prod KHÔNG có ở đây) │ │
│   │ Associations:           │  │                             │ │
│   │   - vpc-prod-attachment │  │ Associations:               │ │
│   │   - vpc-shared-attach   │  │   - vpc-dev-attachment      │ │
│   │   - vpn-attachment      │  │   - vpc-shared-attachment   │ │
│   └─────────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

Kết quả:
✅ VPC-Prod → VPC-Shared: ✅ (cùng Route Table Prod, Shared có route propagation)
✅ VPC-Dev → VPC-Shared: ✅ (cùng Route Table Dev)
❌ VPC-Prod → VPC-Dev: ❌ (Prod RT không có route đến 10.1.0.0/16)
❌ VPC-Dev → VPC-Prod: ❌ (Dev RT không có route đến 10.0.0.0/16)
❌ VPC-Dev → VPN (on-premises): ❌ (Dev RT không có VPN attachment)
```

### Pattern: Centralized Egress — Luồng Ra Tập Trung

Tất cả traffic internet (0.0.0.0/0) đi qua VPC Shared có NAT Gateway:

```
VPC-A, VPC-B, VPC-C ─────► TGW ────► VPC-Egress (NAT GW) ────► Internet
                                      (Centralized NAT)

TGW Route Table (tất cả VPCs):
- 0.0.0.0/0 → tgw-attach-egress-vpc

VPC-Egress Route Table:
- 0.0.0.0/0 → NAT Gateway
- 10.0.0.0/8 → TGW (traffic ngược về VPCs)

Lợi ích:
✅ Tiết kiệm chi phí NAT Gateway (chỉ cần 1 thay vì N)
✅ Centralized internet egress logging
✅ Dễ apply firewall rules tại điểm tập trung
```

### Pattern: Centralized Inspection — Kiểm Tra Tập Trung

Tất cả traffic đi qua AWS Network Firewall hoặc 3rd party appliance:

```
VPC-A ─► TGW ─► VPC-Inspection (Network Firewall) ─► TGW ─► VPC-B

TGW Route Table (phía sender):
- 10.2.0.0/16 (VPC-B) → tgw-attach-inspection-vpc

VPC-Inspection Route Table:
- 10.2.0.0/16 → TGW (sau khi inspect xong, forward tiếp)

Lưu ý: Cần bật Appliance Mode trên TGW attachment của VPC-Inspection
để đảm bảo traffic đi và về qua cùng một AZ (symmetric routing)
```

---

## Transit Gateway Peering — Liên Kết Đa Vùng

### Kiến Trúc Multi-Region

```
┌────────────────────────────┐     ┌────────────────────────────┐
│       us-east-1            │     │        eu-west-1           │
│                            │     │                            │
│  VPC-A    VPC-B    VPC-C   │     │  VPC-D    VPC-E    VPC-F  │
│    │        │        │     │     │    │        │        │    │
│    └────────┼────────┘     │     │    └────────┼────────┘    │
│             │              │     │             │             │
│     ┌───────▼──────┐       │     │     ┌───────▼──────┐      │
│     │    TGW-US    │◄──────┼─────┼────►│    TGW-EU    │      │
│     │  (us-east-1) │       │     │     │  (eu-west-1) │      │
│     └──────────────┘       │     │     └──────────────┘      │
│              │             │     │                            │
│    VPN/DX On-prem US       │     │    VPN/DX On-prem EU      │
└────────────────────────────┘     └────────────────────────────┘

Routing:
us-east-1 TGW Route Table:
  10.4.0.0/16 (VPC-D) → TGW Peering attachment → TGW-EU
  10.5.0.0/16 (VPC-E) → TGW Peering attachment → TGW-EU

eu-west-1 TGW Route Table:
  10.0.0.0/16 (VPC-A) → TGW Peering attachment → TGW-US
  10.1.0.0/16 (VPC-B) → TGW Peering attachment → TGW-US
```

**Lưu ý:** TGW Peering routes phải được thêm **thủ công** (static) — không có propagation tự động qua peering.

---

## Cấu Hình & Triển Khai

### Tạo Transit Gateway

```bash
# Tạo Transit Gateway
aws ec2 create-transit-gateway \
  --description "Main Enterprise TGW" \
  --options '{
    "AmazonSideAsn": 64512,
    "AutoAcceptSharedAttachments": "disable",
    "DefaultRouteTableAssociation": "disable",
    "DefaultRouteTablePropagation": "disable",
    "DnsSupport": "enable",
    "VpnEcmpSupport": "enable",
    "MulticastSupport": "disable"
  }' \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=Main-TGW}]'
```

**Tùy chọn quan trọng:**
- `DefaultRouteTableAssociation: disable` — Không tự động associate attachment vào default RT (khuyến nghị để kiểm soát tốt hơn)
- `DefaultRouteTablePropagation: disable` — Không tự động propagate routes (khuyến nghị)
- `VpnEcmpSupport: enable` — ECMP (Equal-Cost Multi-Path — Đa Đường Chi Phí Bằng Nhau) cho VPN, tăng throughput

### Tạo Route Tables Tùy Chỉnh

```bash
# Tạo Production Route Table
PROD_RT=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id tgw-xxxxxxxxxx \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Prod-RT}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
  --output text)

# Tạo Dev Route Table
DEV_RT=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id tgw-xxxxxxxxxx \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Dev-RT}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
  --output text)

# Associate VPC-Prod attachment với Prod RT
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id $PROD_RT \
  --transit-gateway-attachment-id tgw-attach-vpc-prod

# Enable propagation: VPC-Prod quảng bá route vào Prod RT
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $PROD_RT \
  --transit-gateway-attachment-id tgw-attach-vpc-prod

# Enable propagation: VPC-Shared quảng bá route vào Prod RT
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $PROD_RT \
  --transit-gateway-attachment-id tgw-attach-vpc-shared
```

### Cập Nhật VPC Route Table Để Dùng TGW

```bash
# Trong VPC-Prod, thêm route đến TGW cho các dải đích
aws ec2 create-route \
  --route-table-id rtb-prod-private \
  --destination-cidr-block "10.0.0.0/8" \   # Tất cả internal traffic qua TGW
  --transit-gateway-id tgw-xxxxxxxxxx
```

---

## Transit Gateway vs VPC Peering

### So Sánh Chi Tiết

| Tiêu Chí | Transit Gateway | VPC Peering |
|---------|----------------|------------|
| **Mô hình** | Hub-and-spoke | Point-to-point |
| **Transitive routing** | ✅ Có | ❌ Không |
| **Số kết nối** | O(n) attachments | O(n²) peering connections |
| **Cross-region** | Có (TGW Peering) | Có (inter-region peering) |
| **Cross-account** | Có (RAM sharing) | Có |
| **Chi phí** | Có (attachment fee + data) | Có (data transfer across AZ) |
| **Bandwidth** | 50 Gbps/attachment | 10 Gbps/flow, no limit tổng |
| **Routing control** | Route tables linh hoạt | Route tables riêng mỗi VPC |
| **Network Firewall** | Centralized inspection ✅ | Không thể centralize |
| **VPN/DX integration** | ✅ Tích hợp tốt | ❌ Không kết nối được |
| **Management** | Centralized | Phân tán, khó manage |

### Khi Nào Dùng Cái Nào?

```
Dùng VPC Peering khi:
✅ Chỉ 2-4 VPCs cần kết nối
✅ Không cần transitive routing
✅ Muốn tiết kiệm chi phí (không có attachment fee)
✅ Performance cực cao (không qua TGW hop)
✅ Kết nối đơn giản, ít thay đổi

Dùng Transit Gateway khi:
✅ Nhiều VPCs (5+ và growing)
✅ Cần centralized routing/firewall
✅ Kết nối VPN hoặc Direct Connect vào nhiều VPCs
✅ Cần network segmentation (isolated environments)
✅ Multi-account, enterprise environment
✅ Cần visibility tập trung (Flow Logs trên TGW)
```

---

## Chi Phí

### Bảng Giá (US East — N. Virginia)

| Thành Phần | Đơn Giá | Ghi Chú |
|-----------|---------|---------|
| **TGW Attachment** | $0.05/giờ/attachment | ~$36/tháng/attachment |
| **Data processed** | $0.02/GB | Mỗi GB qua TGW |

### Ước Tính Chi Phí

```
Kịch bản: 10 VPCs + 2 VPN connections + 1 DX GW attachment
          5 TB/tháng data qua TGW

Attachments:  (10 + 2 + 1) × $0.05 × 24h × 30 ngày = $468/tháng
Data:         5 TB × 1024 GB × $0.02               = $102.40/tháng
────────────────────────────────────────────────────────────────────
Tổng:                                               ~$570/tháng

So với VPC Peering (10 VPCs, 45 connections):
  Peering không có attachment fee, nhưng inter-AZ data transfer
  10 VPCs × $0.02/GB × 5TB chia đều ≈ khó tính chính xác
  Thường rẻ hơn TGW nếu data transfer ít
```

### Tối Ưu Chi Phí

```
1. Consolidate attachments: Dùng TGW Peering thay vì nhiều DX connections
2. Centralized egress: 1 NAT GW qua TGW thay vì N NAT GWs riêng
3. Xóa unused attachments: Mỗi attachment = $36/tháng dù không dùng
4. Placement: Subnet selection cho TGW attachment — chỉ 1 subnet/AZ cần thiết
```

---

## Câu Hỏi Phỏng Vấn

### Q1: Giải thích mô hình hub-and-spoke của Transit Gateway

**Trả lời mẫu:**

> Transit Gateway hoạt động như một router trung tâm (hub) trong mô hình hub-and-spoke. Thay vì tạo kết nối point-to-point giữa từng cặp VPCs, mỗi VPC chỉ cần một attachment vào TGW. TGW sau đó route traffic giữa các VPCs dựa trên route tables của nó. Lợi ích lớn nhất là transitive routing — VPC-A có thể reach VPC-C thông qua TGW mà không cần kết nối trực tiếp, điều này VPC Peering không làm được. Ngoài ra, TGW còn kết nối được VPN connections và Direct Connect, trở thành điểm trung tâm cho toàn bộ hybrid connectivity của doanh nghiệp.

### Q2: Làm thế nào để cô lập Dev và Prod trong Transit Gateway?

**Trả lời mẫu:**

> Dùng multiple route tables trong TGW. Tạo Prod Route Table và Dev Route Table riêng. VPC-Prod attachment được associate với Prod RT và propagate routes vào đó. VPC-Dev attachment được associate với Dev RT và propagate routes vào đó. VPC Shared Services được propagate vào cả hai RT. Kết quả: Dev có thể reach Shared Services nhưng không thể reach Prod (vì Prod RT không có route đến Dev CIDR và ngược lại). Đây là network segmentation mà không cần Security Groups hay NACLs riêng.

### Q3: Transit Gateway vs VPC Peering — khi nào dùng cái nào?

**Trả lời mẫu:**

> VPC Peering phù hợp cho kết nối đơn giản giữa 2-4 VPCs — không có overhead của TGW, không có attachment fee, và bandwidth không bị giới hạn bởi TGW hop. Nhưng VPC Peering không hỗ trợ transitive routing, nghĩa là VPC-A không thể reach VPC-C qua VPC-B. Khi có nhiều VPCs (5+), cần centralized routing, muốn kết nối VPN hoặc Direct Connect vào nhiều VPCs cùng lúc, hoặc cần network segmentation linh hoạt — Transit Gateway là lựa chọn đúng. Trong enterprise production environment với nhiều accounts và regions, Transit Gateway gần như là bắt buộc.

### Q4: Thiết kế kiến trúc TGW cho công ty với 30 VPCs, on-premises datacenter, và DR region

**Đáp án kiến trúc:**

```
Kiến trúc:

Primary Region (us-east-1):
  TGW-Primary ─────► 20 VPCs (Production)
       │          ─► 10 VPCs (Dev/Staging)
       │          ─► VPN (backup connectivity)
       └──────────► DX GW → Direct Connect (primary to on-prem)

DR Region (us-west-2):
  TGW-DR ──────────► 5 VPCs (DR standby VPCs)
       └──────────► VPN (backup to on-prem)

TGW Peering:
  TGW-Primary ◄──── TGW Peering ────► TGW-DR

On-Premises:
  DX (10 Gbps) → DX GW → TGW-Primary (primary)
  VPN tunnels → TGW-Primary và TGW-DR (failover)

Network Segmentation:
  Prod RT: Production VPCs + On-Prem access
  Dev RT: Dev/Staging VPCs (isolated from Prod)
  Shared RT: Shared Services VPCs (accessible from all)

Failover:
  DX fails → BGP chọn VPN path tự động (LOCAL_PREF setting)
  Primary region issues → Route 53 failover → DR region
```

### Q5: Appliance Mode trong TGW là gì?

**Trả lời mẫu:**

> Appliance Mode là tùy chọn trên TGW VPC attachment, cần bật khi VPC đó chứa stateful network appliance (như AWS Network Firewall, hoặc 3rd party firewall). Vấn đề: TGW mặc định route traffic theo AZ (traffic vào AZ-a từ VPC khác sẽ đi qua TGW ENI ở AZ-a của VPC Inspection). Nhưng traffic trả về có thể đi qua AZ khác, phá vỡ stateful inspection. Appliance Mode giải quyết bằng cách đảm bảo cả hai chiều của một flow đều đi qua cùng một AZ, cho phép stateful appliance track connection state đúng cách.

---

## 🔗 Điều Hướng

| | |
|--|--|
| ← Trước | [4-direct-connect-gateway.md](./4-direct-connect-gateway.md) |
| → Tiếp theo | [07-advanced-networking/README.md](../07-advanced-networking/README.md) |
| ↑ Section | [06-connectivity/README.md](./README.md) |
| ↑ Chỉ mục | [INDEX.md](../INDEX.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
