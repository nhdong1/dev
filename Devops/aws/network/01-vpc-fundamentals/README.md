# 01 — VPC Fundamentals (Nền Tảng VPC)

> Phần này bao gồm toàn bộ kiến thức nền tảng về VPC (Virtual Private Cloud — Đám Mây Riêng Ảo) trên AWS — từ thiết kế kiến trúc, chia Subnet (Mạng Con), định tuyến, đến kết nối giữa các VPC.

---

## 📚 Nội Dung Phần Này

| File | Chủ Đề | Trạng Thái |
|------|--------|-----------|
| [1-vpc-architecture.md](./1-vpc-architecture.md) | VPC, CIDR, AZ, Region Design | ✅ Hoàn thành |
| [2-subnets.md](./2-subnets.md) | Public / Private / Isolated Subnets | ✅ Hoàn thành |
| [3-route-tables.md](./3-route-tables.md) | Route Tables & Internet Gateway | ✅ Hoàn thành |
| [4-nat-gateway.md](./4-nat-gateway.md) | NAT Gateway vs NAT Instance | ✅ Hoàn thành |
| [5-vpc-peering.md](./5-vpc-peering.md) | VPC Peering & Resource Sharing | ✅ Hoàn thành |

---

## 🎯 Mục Tiêu Học Tập

Sau khi hoàn thành phần này, bạn có thể:

- [ ] Thiết kế VPC production-ready từ đầu mà không cần tham khảo
- [ ] Giải thích CIDR notation và tính toán địa chỉ IP cho Subnets
- [ ] Phân biệt Public Subnet, Private Subnet, và Isolated Subnet
- [ ] Cấu hình Route Tables đúng cách cho từng loại Subnet
- [ ] Chọn giữa NAT Gateway và NAT Instance dựa trên yêu cầu thực tế
- [ ] Thiết lập VPC Peering và hiểu giới hạn của nó
- [ ] Trả lời tự tin các câu hỏi phỏng vấn về VPC

---

## 🏗️ Kiến Trúc Tổng Quan

```
AWS Region (ví dụ: ap-southeast-1 — Singapore)
│
└── VPC: 10.0.0.0/16
    │
    ├── Availability Zone A (ap-southeast-1a)
    │   ├── Public Subnet:   10.0.1.0/24  ──► Internet Gateway (IGW)
    │   ├── Private Subnet:  10.0.11.0/24 ──► NAT Gateway
    │   └── Isolated Subnet: 10.0.21.0/24 ──► (không ra internet)
    │
    ├── Availability Zone B (ap-southeast-1b)
    │   ├── Public Subnet:   10.0.2.0/24
    │   ├── Private Subnet:  10.0.12.0/24
    │   └── Isolated Subnet: 10.0.22.0/24
    │
    └── Availability Zone C (ap-southeast-1c)
        ├── Public Subnet:   10.0.3.0/24
        ├── Private Subnet:  10.0.13.0/24
        └── Isolated Subnet: 10.0.23.0/24
```

**Giải thích:**
- **Public Subnet** — Subnet có route đến Internet Gateway, dùng cho Load Balancer, Bastion Host
- **Private Subnet** — Subnet có route qua NAT Gateway, dùng cho Application Servers
- **Isolated Subnet** — Subnet không có route ra internet, dùng cho Database, Cache

---

## 🔑 Khái Niệm Cốt Lõi

### VPC (Virtual Private Cloud — Đám Mây Riêng Ảo)

Môi trường mạng ảo riêng biệt trong AWS, cho phép bạn kiểm soát hoàn toàn cấu hình mạng: dải địa chỉ IP, Subnets, Route Tables, và Network Gateways.

### CIDR (Classless Inter-Domain Routing — Định Tuyến Liên Miền Không Phân Lớp)

Ký hiệu xác định dải địa chỉ IP. Ví dụ: `10.0.0.0/16` có nghĩa là 65.536 địa chỉ IP (từ 10.0.0.0 đến 10.0.255.255).

### Availability Zone — AZ (Vùng Khả Dụng)

Trung tâm dữ liệu vật lý riêng biệt trong một Region. Triển khai trên nhiều AZ đảm bảo High Availability (Tính Sẵn Sàng Cao).

### Route Table (Bảng Định Tuyến)

Tập hợp các rules (luật) xác định traffic (lưu lượng mạng) đi đến đâu. Mỗi Subnet phải được liên kết với một Route Table.

### Internet Gateway — IGW (Cổng Internet)

Thành phần cho phép VPC giao tiếp với internet. Có tính High Availability và không cần quản lý.

### NAT Gateway (Network Address Translation Gateway — Cổng Dịch Địa Chỉ Mạng)

Cho phép instances trong Private Subnet kết nối ra internet nhưng ngăn internet kết nối vào. Thay thế cho NAT Instance.

---

## 📋 Thứ Tự Học Được Khuyến Nghị

```
1. vpc-architecture.md  ─── Hiểu VPC là gì, CIDR, Region/AZ
           │
           ▼
2. subnets.md           ─── Phân chia mạng con, public/private/isolated
           │
           ▼
3. route-tables.md      ─── Định tuyến traffic, Internet Gateway
           │
           ▼
4. nat-gateway.md       ─── Cho private instances ra internet
           │
           ▼
5. vpc-peering.md       ─── Kết nối giữa các VPC
```

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp (Preview)

1. **Thiết kế VPC cho ứng dụng 3-tier (3 lớp) — Web, App, DB**
2. **Sự khác biệt giữa Internet Gateway và NAT Gateway?**
3. **Khi nào nên dùng VPC Peering vs Transit Gateway?**
4. **CIDR /16 vs /24 — cái nào to hơn? Chứa bao nhiêu host?**
5. **Tại sao Private Subnet cần NAT Gateway để ra internet?**

Chi tiết câu trả lời có trong từng file tương ứng.

---

## ⏱️ Thời Gian Học Ước Tính

| Phần | Thời Gian | Độ Khó |
|------|-----------|--------|
| VPC Architecture & CIDR | 1.5 giờ | ⭐⭐ |
| Subnets Design | 1 giờ | ⭐⭐ |
| Route Tables & IGW | 1 giờ | ⭐⭐ |
| NAT Gateway | 1 giờ | ⭐⭐ |
| VPC Peering | 1.5 giờ | ⭐⭐⭐ |
| **Tổng** | **6-8 giờ** | ⭐⭐ |

---

## 🔗 Điều Hướng

- ← [Lộ Trình Tổng Quan](../README.md)
- → [VPC Architecture](./1-vpc-architecture.md) — Bắt đầu học
- [Chỉ Mục Đầy Đủ](../INDEX.md)
