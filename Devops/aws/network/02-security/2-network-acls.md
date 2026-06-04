# Network ACLs — Tường Lửa Phi Trạng Thái & So Sánh Với Security Groups

> Network ACL (NACL — Network Access Control List — Danh Sách Kiểm Soát Truy Cập Mạng) là tường lửa **stateless** (phi trạng thái) hoạt động ở cấp độ **subnet** trong VPC. Là lớp bảo vệ thứ hai bổ sung cho Security Groups, đặc biệt hữu ích khi cần chặn dải IP độc hại hoặc kiểm soát traffic cấp subnet.

---

## 📚 Mục Lục

1. [Khái Niệm Cơ Bản](#1-khái-niệm-cơ-bản)
2. [Cách Hoạt Động Stateless](#2-cách-hoạt-động-stateless)
3. [Cấu Trúc Rules & Số Thứ Tự](#3-cấu-trúc-rules--số-thứ-tự)
4. [Ephemeral Ports — Cổng Tạm Thời](#4-ephemeral-ports--cổng-tạm-thời)
5. [Default NACL vs Custom NACL](#5-default-nacl-vs-custom-nacl)
6. [So Sánh Security Group và Network ACL](#6-so-sánh-security-group-và-network-acl)
7. [Khi Nào Dùng NACL](#7-khi-nào-dùng-nacl)
8. [Best Practices](#8-best-practices)
9. [Ví Dụ Thực Tế](#9-ví-dụ-thực-tế)
10. [Câu Hỏi Phỏng Vấn](#10-câu-hỏi-phỏng-vấn)

---

## 1. Khái Niệm Cơ Bản

### Network ACL Là Gì?

```
┌─────────────────────────────────────────────────────────┐
│                          VPC                            │
│                                                         │
│   ┌─────────────────────────────────────────────────┐   │
│   │          ◄── NACL hoạt động tại đây ──►         │   │
│   │                  Subnet Boundary                │   │
│   │                                                 │   │
│   │   ┌──────────────────┐  ┌──────────────────┐    │   │
│   │   │   EC2 Instance   │  │   EC2 Instance   │    │   │
│   │   │  ┌────────────┐  │  │  ┌────────────┐  │    │   │
│   │   │  │    SG      │  │  │  │    SG      │  │    │   │
│   │   │  └────────────┘  │  │  └────────────┘  │    │   │
│   │   └──────────────────┘  └──────────────────┘    │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**NACL:** Bảo vệ toàn bộ subnet — áp dụng cho **mọi** instance trong subnet đó.
**Security Group:** Bảo vệ từng instance cụ thể.

### Đặc Điểm Quan Trọng

| Đặc Điểm | Chi Tiết |
|----------|----------|
| **Stateless** | Không nhớ trạng thái kết nối — mỗi gói tin xử lý độc lập |
| **Subnet-level** | Áp dụng cho toàn bộ subnet, không phải từng instance |
| **Allow + Deny** | Có cả rules Allow và Deny (khác với SG chỉ có Allow) |
| **Numbered rules** | Mỗi rule có số thứ tự — xử lý theo thứ tự tăng dần |
| **First match wins** | Dừng lại khi gặp rule khớp đầu tiên |
| **One NACL per subnet** | Một subnet chỉ liên kết với một NACL tại một thời điểm |
| **Multiple subnets** | Một NACL có thể áp dụng cho nhiều subnets |

---

## 2. Cách Hoạt Động Stateless

### Stateless — Không Nhớ Kết Nối

```
Gói Tin Đến (Inbound Packet):
  Nguồn: 1.2.3.4:54321 → Đích: 10.0.1.5:443
  NACL kiểm tra: Inbound rules theo thứ tự số
  → Rule 100: TCP 443 0.0.0.0/0 ALLOW → ✅ Cho phép vào

Gói Tin Phản Hồi (Return Packet):
  Nguồn: 10.0.1.5:443 → Đích: 1.2.3.4:54321
  NACL kiểm tra: Outbound rules theo thứ tự số (KHÔNG nhớ đây là reply)
  → Phải có outbound rule cho ephemeral ports → ✅ hoặc ❌
```

**Vấn đề thực tế:** Với NACL, khi mở port 443 inbound, bạn **BẮT BUỘC** phải mở ephemeral ports (1024-65535) outbound để response về được client.

### Ngược Lại Với Stateful (Security Group)

```
Security Group — Stateful:
  Request: Port 443 vào → Rule allow → ✅
  Response: Port cao → Tự động allow (nhớ kết nối) → ✅

Network ACL — Stateless:
  Request: Port 443 vào → Inbound rule allow → ✅
  Response: Port cao ra → PHẢI có outbound rule → ✅ (hoặc ❌ nếu không có rule)
```

---

## 3. Cấu Trúc Rules & Số Thứ Tự

### Cấu Trúc Một Rule

| Trường | Mô Tả | Ví Dụ |
|--------|-------|-------|
| **Rule number** | Số thứ tự (1–32766); thấp hơn = ưu tiên cao hơn | 100, 200, 32767 |
| **Type** | Loại traffic | HTTP, HTTPS, Custom TCP |
| **Protocol** | TCP, UDP, ICMP, All | TCP (6) |
| **Port range** | Cổng hoặc dải | 443, 1024-65535 |
| **Source/Destination** | CIDR block | 0.0.0.0/0, 10.0.0.0/8 |
| **Allow/Deny** | Hành động | ALLOW hoặc DENY |

### Quy Tắc Đánh Số (Numbering Convention)

```
AWS khuyến nghị đánh số theo bước 10 hoặc 100:

Rule 100:  Allow HTTPS từ internet
Rule 200:  Allow HTTP từ internet  
Rule 300:  Allow SSH từ corporate IP
Rule 400:  Allow ephemeral ports (return traffic)
...
Rule 32767: * Deny ALL (star rule — rule cuối mặc định của AWS)
```

**Tại sao dùng bước nhảy lớn?** Để chèn thêm rules sau này mà không phải re-number.

```
Ban đầu: 100, 200, 300
Sau khi thêm: 100, 150, 200, 250, 300
(Không cần đổi 200, 300)
```

### Star Rule (*) — Rule Mặc Định Cuối Cùng

AWS tự động thêm rule `*` (ký tự asterisk) ở cuối:
- **Số:** * (không thể thay đổi thứ tự)
- **Hành động:** DENY ALL
- **Không thể xóa hoặc sửa**

Mọi traffic không khớp rule nào trước đó sẽ bị deny bởi rule này.

---

## 4. Ephemeral Ports — Cổng Tạm Thời

### Khái Niệm

Khi client kết nối đến server, client chọn một **ephemeral port** (cổng tạm thời) ngẫu nhiên để nhận response:

```
Client (random port)        Server
  Port: 52341       ──────► Port: 443
                    ◄──────
                       Response về port 52341
```

### Dải Ephemeral Ports Theo Hệ Điều Hành

| Hệ Điều Hành | Dải Cổng |
|-------------|----------|
| Linux (kernel ≥ 3.2) | 32768 – 60999 |
| Windows (nhiều phiên bản) | 49152 – 65535 |
| NAT Gateway | 1024 – 65535 |
| Thực hành AWS | 1024 – 65535 (an toàn nhất) |

### Tác Động Thực Tế Với NACL

```
Để cho phép client internet kết nối vào EC2 (port 443):

Inbound NACL:
  Rule 100: TCP 443 0.0.0.0/0 ALLOW        ← Client gửi request

Outbound NACL:
  Rule 100: TCP 1024-65535 0.0.0.0/0 ALLOW ← Response về ephemeral port của client
  (hoặc Rule 100: ALL 0.0.0.0/0 ALLOW)
```

**Lưu ý quan trọng:** Nếu EC2 trong subnet cũng là client gọi ra ngoài (ví dụ: call API, yum update), thì:

```
Outbound NACL:
  Rule 100: TCP 443 0.0.0.0/0 ALLOW        ← EC2 gửi request

Inbound NACL:
  Rule 200: TCP 1024-65535 0.0.0.0/0 ALLOW ← Response về ephemeral port của EC2
```

---

## 5. Default NACL vs Custom NACL

### Default NACL (NACL Mặc Định)

Khi tạo VPC, AWS tạo một Default NACL tự động:

```
Default NACL — Inbound:
  Rule 100: ALL 0.0.0.0/0 ALLOW
  Rule *:   ALL 0.0.0.0/0 DENY

Default NACL — Outbound:
  Rule 100: ALL 0.0.0.0/0 ALLOW
  Rule *:   ALL 0.0.0.0/0 DENY
```

→ **Kết quả:** Cho phép tất cả traffic — không có bảo mật thêm.

Mọi subnet mới tạo trong VPC sẽ tự động liên kết với Default NACL.

### Custom NACL (NACL Tùy Chỉnh)

Khi tạo Custom NACL mới:

```
Custom NACL — Inbound:
  Rule *: ALL 0.0.0.0/0 DENY  ← Chỉ có rule deny cuối cùng!

Custom NACL — Outbound:
  Rule *: ALL 0.0.0.0/0 DENY  ← Chỉ có rule deny cuối cùng!
```

→ **Kết quả:** DENY tất cả traffic cho đến khi bạn thêm Allow rules.

**Thực hành tốt nhất:** Tạo Custom NACL cho production subnets, không dùng Default NACL.

---

## 6. So Sánh Security Group và Network ACL

| Tiêu Chí | Security Group | Network ACL |
|---------|---------------|------------|
| **Phạm vi áp dụng** | ENI (Instance level) | Subnet level |
| **Trạng thái (State)** | Stateful | Stateless |
| **Loại rules** | Allow only | Allow + Deny |
| **Return traffic** | Tự động cho phép | Phải tạo rule riêng |
| **Xử lý rules** | Tất cả rules đều đánh giá | Dừng tại rule khớp đầu tiên |
| **Số lượng** | 5 SGs per ENI, tối đa 16 | 1 NACL per subnet |
| **Thứ tự rules** | Không quan trọng (tất cả đánh giá) | Quan trọng (số thứ tự) |
| **Deny rõ ràng** | Không thể | Có thể |
| **Phù hợp cho** | Kiểm soát instance-level | Chặn dải IP độc hại, subnet-wide |
| **Default** | Deny all inbound, Allow all outbound | Tùy loại (Default SG vs Custom) |

### Cả Hai Cùng Hoạt Động

```
Traffic vào EC2:
Internet → [NACL Subnet] → [Security Group Instance] → EC2

Traffic ra từ EC2:
EC2 → [Security Group Instance] → [NACL Subnet] → Internet

Cả hai đều phải ALLOW → traffic mới đi qua được.
```

---

## 7. Khi Nào Dùng NACL

### Use Cases Phù Hợp

1. **Chặn IP độc hại (Blocklist):**
   ```
   Rule 10: DENY ALL từ 203.0.113.0/24  ← Chặn dải IP tấn công
   Rule 100: ALLOW HTTPS từ 0.0.0.0/0
   ```
   Security Group không có Deny rules — NACL là lựa chọn duy nhất.

2. **Subnet-wide policy:**
   Khi muốn áp dụng policy đồng nhất cho toàn bộ instances trong subnet (production vs. development subnets).

3. **Defense in depth (Bảo vệ nhiều lớp):**
   Thêm lớp bảo vệ bổ sung cho Security Groups. Kể cả nếu Security Group bị cấu hình sai, NACL vẫn bảo vệ.

4. **Compliance requirements:**
   Một số tiêu chuẩn (PCI-DSS, HIPAA) yêu cầu kiểm soát network ở cấp subnet.

### Khi NACL Không Phù Hợp

- **Không thể tham chiếu Security Groups** — chỉ dùng được CIDR
- **Không phù hợp cho fine-grained control** từng service — dùng SG
- **Quản lý phức tạp hơn** nếu nhiều subnets cần rules khác nhau

---

## 8. Best Practices

### ✅ Nên Làm

1. **Đặt Allow rules trước Deny** nếu cần cả hai:
   ```
   Rule 100: ALLOW từ trusted_ip/32   ← Whitelist trước
   Rule 200: DENY  từ 0.0.0.0/0       ← Chặn phần còn lại
   ```

2. **Mở ephemeral ports cho return traffic:**
   ```
   Outbound Rule: TCP 1024-65535 0.0.0.0/0 ALLOW
   ```

3. **Ghi chú rõ ràng** mỗi rule được thêm vào để làm gì

4. **Dùng bước nhảy 10 hoặc 100** khi đánh số rules để dễ chèn thêm sau

5. **Review thường xuyên** — rules cũ có thể trở thành lỗ hổng bảo mật

6. **Kết hợp với Security Groups** để có defense in depth

### ❌ Không Nên Làm

1. **Không quên ephemeral ports** — lỗi phổ biến nhất với NACL
2. **Không dùng Default NACL** cho production subnets
3. **Không tạo quá nhiều rules** — giới hạn 20 inbound + 20 outbound per NACL
4. **Không thay thế Security Groups** bằng NACL — chúng bổ sung cho nhau

### Giới Hạn NACL

| Giới Hạn | Mặc Định |
|---------|----------|
| NACLs per VPC | 200 |
| Rules per NACL (inbound + outbound) | 20 + 20 = 40 |
| Subnets per NACL | Không giới hạn |

---

## 9. Ví Dụ Thực Tế

### Ví Dụ 1: Public Subnet NACL (Subnet Công Khai)

```
# NACL cho Public Subnet (Web Tier)
# Áp dụng cho: subnet-public-1a, subnet-public-1b

INBOUND RULES:
  Rule 100: TCP   80   0.0.0.0/0  ALLOW  (HTTP)
  Rule 110: TCP   443  0.0.0.0/0  ALLOW  (HTTPS)
  Rule 120: TCP   22   OFFICE_IP  ALLOW  (SSH từ văn phòng)
  Rule 130: TCP   1024-65535  0.0.0.0/0  ALLOW  (Return traffic)
  Rule 200: ICMP  ALL  0.0.0.0/0  ALLOW  (Ping — tùy chọn)
  Rule *:   ALL   0.0.0.0/0  DENY   (Mặc định chặn tất cả)

OUTBOUND RULES:
  Rule 100: TCP   443  0.0.0.0/0  ALLOW  (HTTPS ra ngoài)
  Rule 110: TCP   80   0.0.0.0/0  ALLOW  (HTTP ra ngoài)
  Rule 120: TCP   1024-65535  0.0.0.0/0  ALLOW  (Return traffic cho inbound)
  Rule 130: TCP   5432  10.0.2.0/24  ALLOW  (Kết nối database subnet)
  Rule *:   ALL   0.0.0.0/0  DENY
```

### Ví Dụ 2: Private Subnet NACL (Subnet Riêng Tư — App Tier)

```
# NACL cho Private Subnet (App Tier)
# Chỉ nhận traffic từ Public Subnet

INBOUND RULES:
  Rule 100: TCP  8080  10.0.1.0/24  ALLOW  (Từ public subnet)
  Rule 110: TCP  22    10.0.1.0/24  ALLOW  (SSH từ public subnet)
  Rule 120: TCP  1024-65535  0.0.0.0/0  ALLOW  (Return traffic)
  Rule *:   ALL  0.0.0.0/0   DENY

OUTBOUND RULES:
  Rule 100: TCP  443   0.0.0.0/0   ALLOW  (HTTPS ra ngoài qua NAT)
  Rule 110: TCP  5432  10.0.3.0/24 ALLOW  (Kết nối DB subnet)
  Rule 120: TCP  1024-65535  10.0.1.0/24  ALLOW  (Return về public subnet)
  Rule *:   ALL  0.0.0.0/0   DENY
```

### Ví Dụ 3: Database Subnet NACL (Subnet Cơ Sở Dữ Liệu)

```
# NACL cho Isolated Subnet (Database Tier)
# Rất hạn chế — chỉ nhận từ app tier

INBOUND RULES:
  Rule 100: TCP  5432  10.0.2.0/24  ALLOW  (PostgreSQL từ app subnet)
  Rule 110: TCP  3306  10.0.2.0/24  ALLOW  (MySQL/Aurora từ app subnet)
  Rule 120: TCP  1024-65535  10.0.2.0/24  ALLOW  (Return traffic)
  Rule *:   ALL  0.0.0.0/0   DENY

OUTBOUND RULES:
  Rule 100: TCP  1024-65535  10.0.2.0/24  ALLOW  (Response về app subnet)
  Rule *:   ALL  0.0.0.0/0   DENY
```

### Ví Dụ 4: Chặn IP Tấn Công Nhanh

```
# Phát hiện DDoS từ dải IP 198.51.100.0/24
# Thêm rule DENY trước rule ALLOW hiện có:

INBOUND RULES:
  Rule 10:  ALL  198.51.100.0/24  DENY   ← Thêm rule chặn ngay
  Rule 100: TCP  443  0.0.0.0/0  ALLOW   ← Rule cũ giữ nguyên
  ...
```

---

## 10. Câu Hỏi Phỏng Vấn

### Câu 1: Tại sao cần cả Security Group lẫn Network ACL?

**Trả lời:**
Chúng bổ sung cho nhau theo nguyên tắc **defense-in-depth**:
- **NACL:** Chặn sớm ở biên giới subnet, có Deny rules rõ ràng — phù hợp chặn IP độc hại, subnet-wide policy
- **Security Group:** Kiểm soát fine-grained từng instance, stateful (không cần quản lý ephemeral ports)

Dùng NACL như "perimeter firewall" (tường lửa vành đai) và SG như "host-based firewall" (tường lửa host).

---

### Câu 2: Tôi thêm inbound rule TCP 443 ALLOW vào NACL nhưng traffic vẫn không qua — tại sao?

**Trả lời:** Rất có thể đã quên **outbound rule cho ephemeral ports**. NACL là stateless — phải tạo cả hai:

```
Inbound:  TCP 443 0.0.0.0/0 ALLOW  (nhận request)
Outbound: TCP 1024-65535 0.0.0.0/0 ALLOW  (gửi response về ephemeral port của client)
```

Ngoài ra kiểm tra: Security Group có Allow không? Route Table có đúng không?

---

### Câu 3: NACL có thể reference Security Group như SG không?

**Trả lời:** **Không.** Network ACL chỉ hoạt động với **CIDR blocks** (IPv4 và IPv6). Không thể tham chiếu Security Group ID. Đây là lý do khi dùng NACL, bạn cần biết IP/CIDR của source/destination, không thể dùng SG name.

---

### Câu 4: Default NACL và Custom NACL có gì khác nhau?

**Trả lời:**
- **Default NACL:** Tự động tạo khi tạo VPC, cho phép tất cả inbound và outbound traffic. Mọi subnet mới gắn vào default NACL này.
- **Custom NACL:** Khi tạo mới, mặc định **deny tất cả** cho đến khi bạn thêm Allow rules. Phải gắn thủ công vào subnet.

Thực hành tốt nhất: Tạo Custom NACL cho production và gắn vào các production subnets.

---

### Câu 5: Rule NACL được đánh giá thế nào?

**Trả lời:** Rules được đánh giá theo thứ tự số tăng dần. Rule **đầu tiên khớp** sẽ được áp dụng — các rules sau **không được đánh giá nữa**. Cuối cùng là rule `*` (deny all).

Ví dụ:
```
Rule 100: DENY  từ 10.0.0.5
Rule 200: ALLOW từ 10.0.0.0/24

→ Traffic từ 10.0.0.5 sẽ bị DENY (khớp rule 100 trước)
→ Traffic từ 10.0.0.6 sẽ ALLOW (bỏ qua rule 100, khớp rule 200)
```

---

## 🔗 Điều Hướng

| Trước | Hiện Tại | Tiếp Theo |
|-------|----------|-----------|
| [1-security-groups.md](./1-security-groups.md) | **2-network-acls.md** | [3-waf.md](./3-waf.md) |

---

**Cập Nhật Lần Cuối:** 2026-05-14
