# 3 — Route Tables & Internet Gateway (Bảng Định Tuyến & Cổng Internet)

> Route Table (Bảng Định Tuyến) xác định traffic (lưu lượng mạng) trong VPC đi đến đâu. Internet Gateway (IGW — Cổng Internet) là thành phần cho phép VPC giao tiếp với internet.

---

## 📋 Mục Lục

1. [Route Table là gì?](#1-route-table-là-gì)
2. [Internet Gateway — IGW](#2-internet-gateway--igw)
3. [Cách Route Table Hoạt Động](#3-cách-route-table-hoạt-động)
4. [Các Loại Route Table](#4-các-loại-route-table)
5. [Cấu Hình Route Table Thực Tế](#5-cấu-hình-route-table-thực-tế)
6. [Troubleshooting Route Issues](#6-troubleshooting-route-issues)
7. [Câu Hỏi Phỏng Vấn](#7-câu-hỏi-phỏng-vấn)

---

## 1. Route Table là gì?

**Route Table (Bảng Định Tuyến)** là tập hợp các rules (quy tắc) gọi là routes (tuyến đường), xác định traffic từ Subnet hoặc Internet Gateway sẽ được chuyển tiếp đến đâu.

### Thành Phần của Một Route

Mỗi route có hai thành phần:

| Thành Phần | Ý Nghĩa | Ví Dụ |
|-----------|---------|-------|
| **Destination** (Đích) | Dải IP của traffic cần định tuyến | `10.0.0.0/16`, `0.0.0.0/0` |
| **Target** (Mục Tiêu) | Nơi traffic sẽ được chuyển đến | `local`, `igw-xxx`, `nat-xxx` |

### Route Mặc Định "local"

Mỗi Route Table luôn có một route không thể xóa:

```
Destination: 10.0.0.0/16    Target: local
```

Route này đảm bảo tất cả instances trong VPC có thể giao tiếp với nhau trực tiếp, không cần qua bất kỳ gateway nào.

---

## 2. Internet Gateway — IGW

### IGW là gì?

**Internet Gateway (IGW — Cổng Internet)** là thành phần mạng cho phép giao tiếp giữa VPC và internet. Đặc điểm:

- **Horizontally scaled** (Mở Rộng Theo Chiều Ngang) — tự động xử lý mọi lượng traffic
- **Highly Available** (Tính Sẵn Sàng Cao) — không có điểm lỗi duy nhất
- **Không tính phí** cho bản thân IGW (chỉ tính phí data transfer)
- Mỗi VPC chỉ có thể gắn **một** IGW tại một thời điểm

### Cách IGW Hoạt Động

IGW thực hiện hai việc:

1. **Cung cấp route** cho traffic ra internet (outbound) và từ internet vào (inbound)
2. **Dịch địa chỉ IP** (NAT — Network Address Translation): chuyển đổi giữa Public IP của instance và IP nội bộ khi traffic ra/vào internet

```
Instance (10.0.1.5) có Public IP: 54.1.2.3
│
│ Gửi request đến google.com
▼
IGW: Thay IP nguồn từ 10.0.1.5 → 54.1.2.3
│
▼
Internet: Google nhìn thấy request từ 54.1.2.3
│
▼
IGW nhận phản hồi, chuyển về 10.0.1.5
```

### Điều Kiện Để Instance Trong Public Subnet Truy Cập Internet

Phải có đủ **tất cả** bốn điều kiện sau:

```
1. VPC phải có Internet Gateway (IGW) được gắn vào
2. Route Table của Subnet phải có route: 0.0.0.0/0 → IGW
3. Instance phải có Public IP hoặc Elastic IP
4. Security Group phải cho phép traffic (outbound ít nhất)
```

Thiếu bất kỳ điều kiện nào cũng sẽ không kết nối được.

---

## 3. Cách Route Table Hoạt Động

### Quy Tắc Longest Prefix Match (Khớp Tiền Tố Dài Nhất)

Khi có nhiều routes phù hợp với một địa chỉ IP đích, Route Table chọn route có **prefix dài nhất** (specific nhất):

```
Route Table:
┌─────────────────────┬─────────────┐
│ Destination         │ Target      │
├─────────────────────┼─────────────┤
│ 10.0.0.0/16         │ local       │
│ 10.0.5.0/24         │ pcx-xxxxxx  │  ← VPC Peering
│ 0.0.0.0/0           │ igw-xxxxxx  │  ← Internet
└─────────────────────┴─────────────┘
```

Ví dụ với traffic đến `10.0.5.15`:
- `/24` (10.0.5.0/24) cụ thể hơn `/16` (10.0.0.0/16)
- Traffic sẽ đi qua VPC Peering, **không phải** `local` route

### Static Routes vs Propagated Routes

| Loại Route | Mô Tả |
|-----------|-------|
| **Static Route** (Tuyến Tĩnh) | Do bạn tạo thủ công |
| **Propagated Route** (Tuyến Được Truyền Bá) | Tự động từ VPN hoặc Direct Connect qua Virtual Private Gateway (VGW) |

---

## 4. Các Loại Route Table

### Main Route Table (Bảng Định Tuyến Chính)

- Tự động tạo khi tạo VPC
- Áp dụng cho mọi Subnet **chưa** được gán Route Table tùy chỉnh
- Nên giữ minimal (tối giản) — chỉ có route `local`
- **Không nên thêm** route ra internet vào Main Route Table để tránh vô tình expose Subnets

### Custom Route Table (Bảng Định Tuyến Tùy Chỉnh)

- Tạo và quản lý thủ công
- Gán cho Subnet cụ thể
- Cho phép kiểm soát routing per-Subnet (định tuyến theo từng Subnet)

### Subnet Association (Liên Kết Subnet)

Mỗi Subnet phải được liên kết với **đúng một** Route Table:
- Nếu không liên kết tường minh → dùng Main Route Table
- Một Route Table có thể liên kết với nhiều Subnets

---

## 5. Cấu Hình Route Table Thực Tế

### Public Route Table

```
Tên: rtb-prod-public
Subnets: public-1a, public-1b, public-1c
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │
│ 0.0.0.0/0           │ igw-0abc123def456   │
└─────────────────────┴─────────────────────┘
```

### Private Route Table (Per-AZ)

```
Tên: rtb-prod-private-1a
Subnets: private-1a
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │
│ 0.0.0.0/0           │ nat-0xyz789...      │  ← NAT Gateway ở AZ-A
└─────────────────────┴─────────────────────┘

Tên: rtb-prod-private-1b
Subnets: private-1b
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │
│ 0.0.0.0/0           │ nat-0abc123...      │  ← NAT Gateway ở AZ-B
└─────────────────────┴─────────────────────┘
```

### Isolated Route Table

```
Tên: rtb-prod-isolated
Subnets: database-1a, database-1b, database-1c
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │
└─────────────────────┴─────────────────────┘
```

### Route Table Với VPC Peering

Khi kết nối với VPC khác (10.1.0.0/16) qua VPC Peering:

```
Private Route Table (sau khi thêm peering):
┌─────────────────────┬─────────────────────┐
│ Destination         │ Target              │
├─────────────────────┼─────────────────────┤
│ 10.0.0.0/16         │ local               │
│ 0.0.0.0/0           │ nat-0xyz789...      │
│ 10.1.0.0/16         │ pcx-0peerxxx        │  ← VPC Peering route
└─────────────────────┴─────────────────────┘
```

Lưu ý: **Cả hai phía** VPC Peering đều cần thêm route vào Route Table tương ứng.

---

## 6. Troubleshooting Route Issues

### Checklist Khi Instance Không Ra Được Internet

```
□ 1. VPC có Internet Gateway không? (EC2 → VPC → Internet Gateways)
□ 2. IGW đã được Attach (gắn) vào VPC chưa? (State: Attached)
□ 3. Route Table của Subnet có route 0.0.0.0/0 → IGW không?
□ 4. Instance có Public IP hoặc Elastic IP không?
□ 5. Security Group Outbound có cho phép traffic không?
□ 6. Network ACL không block traffic không?
```

### Checklist Khi Private Instance Không Ra Được Internet

```
□ 1. NAT Gateway có trong Public Subnet không?
□ 2. NAT Gateway có Elastic IP không? (State: Available)
□ 3. Route Table của Private Subnet có route 0.0.0.0/0 → NAT GW không?
□ 4. Route Table của Public Subnet có route 0.0.0.0/0 → IGW không?
□ 5. Security Group cho phép outbound traffic không?
```

### Lỗi Thường Gặp

| Triệu Chứng | Nguyên Nhân Có Thể |
|-------------|-------------------|
| Instance không ping được 8.8.8.8 | Thiếu IGW hoặc route `0.0.0.0/0` |
| Private instance không tải được packages | NAT Gateway chưa có hoặc route sai |
| Hai instances trong VPC không ping được nhau | Route `local` bị xóa (hiếm) hoặc Security Group block |
| Route đúng nhưng vẫn không kết nối được | Kiểm tra Network ACL, sau đó Security Group |

---

## 7. Câu Hỏi Phỏng Vấn

### Q1: Route Table là gì và hoạt động như thế nào?

**Trả lời mẫu:**

> Route Table là tập hợp các rules định tuyến xác định traffic từ Subnet sẽ đi đến đâu. Mỗi rule có Destination (dải IP đích) và Target (nơi forward traffic đến). Khi có nhiều routes phù hợp, Route Table chọn route có prefix dài nhất — cụ thể nhất. Route `local` luôn tồn tại và không thể xóa, đảm bảo giao tiếp nội bộ VPC.

### Q2: Sự khác biệt giữa Internet Gateway và NAT Gateway?

**Trả lời:**

> Internet Gateway cho phép traffic hai chiều — từ internet vào instance (nếu instance có Public IP và SG cho phép) và từ instance ra internet. NAT Gateway chỉ cho phép traffic một chiều ra — instances trong Private Subnet có thể ra internet nhưng internet không thể khởi tạo kết nối vào. NAT Gateway cần Public IP và phải nằm trong Public Subnet để hoạt động.

### Q3: Tại sao nên tạo Route Table riêng cho mỗi loại Subnet thay vì dùng Main Route Table?

**Trả lời:**

> Vì Main Route Table áp dụng cho tất cả Subnets chưa được gán Route Table riêng. Nếu tôi thêm route `0.0.0.0/0 → IGW` vào Main Route Table, mọi Subnet mới tạo mà quên gán Route Table riêng sẽ tự động trở thành Public Subnet — đây là security risk. Tốt hơn là giữ Main Route Table chỉ có route `local`, và tạo Custom Route Tables riêng cho Public, Private, và Isolated Subnets.

### Q4: Làm thế nào để một Private Subnet trong AZ-B vẫn hoạt động khi NAT Gateway ở AZ-A gặp sự cố?

**Trả lời:**

> Tạo một NAT Gateway riêng trong Public Subnet của AZ-B, và cấu hình Route Table của Private Subnet AZ-B chỉ đến NAT Gateway ở AZ-B. Mỗi AZ nên có NAT Gateway riêng. Khi AZ-A gặp sự cố, NAT Gateway ở AZ-B vẫn hoạt động bình thường, không ảnh hưởng đến Private Subnet ở AZ-B.

---

## 📝 Tóm Tắt

| Thành Phần | Vai Trò |
|-----------|---------|
| Route Table | Xác định traffic đi đến đâu dựa trên Destination IP |
| Local Route | Đảm bảo giao tiếp nội bộ VPC — không thể xóa |
| Internet Gateway | Cổng vào/ra internet hai chiều |
| 0.0.0.0/0 → IGW | Route làm Subnet trở thành Public |
| 0.0.0.0/0 → NAT | Route cho Private Subnet ra internet |
| Main Route Table | Áp dụng cho Subnets chưa có Route Table riêng |

---

## 🔗 Điều Hướng

- ← [2. Subnets](./2-subnets.md)
- → [4. NAT Gateway](./4-nat-gateway.md)
- [Chỉ Mục Đầy Đủ](../INDEX.md)
