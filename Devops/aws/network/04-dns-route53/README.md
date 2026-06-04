# 🌐 DNS & Route 53 — Tổng Quan

> Amazon Route 53 — DNS (Domain Name System — Hệ Thống Tên Miền) có tính khả dụng cao và khả năng mở rộng của AWS, tích hợp sâu với các dịch vụ AWS khác và hỗ trợ nhiều routing policies (chính sách định tuyến) thông minh.

## 📚 Mục Lục Section Này

| File | Nội Dung | Mức Độ |
|------|----------|--------|
| [1-dns-fundamentals.md](./1-dns-fundamentals.md) | DNS cơ bản, record types, TTL, resolver chain | Cơ bản |
| [2-hosted-zones.md](./2-hosted-zones.md) | Public & Private hosted zones, split-horizon DNS | Cơ bản |
| [3-routing-policies.md](./3-routing-policies.md) | 7 routing policies, use cases, so sánh | Trung cấp |
| [4-health-checks.md](./4-health-checks.md) | Health checks, DNS failover automation | Trung cấp |
| [5-resolver.md](./5-resolver.md) | Route 53 Resolver, hybrid DNS, endpoints | Nâng cao |

---

## 🎯 Tại Sao Route 53 Quan Trọng?

Route 53 không chỉ là DNS đơn thuần — nó là **traffic management platform** (nền tảng quản lý lưu lượng) toàn cầu:

```
Người dùng truy cập example.com
         ↓
   Route 53 nhận query
         ↓
   Áp dụng Routing Policy
   (Simple / Weighted / Latency / Failover / Geo)
         ↓
   Kiểm tra Health Check (nếu cấu hình)
         ↓
   Trả về IP address tối ưu
         ↓
   Người dùng kết nối đến server phù hợp nhất
```

---

## 🏗️ Kiến Trúc Route 53

### Các Thành Phần Chính

```
Route 53
├── Domain Registration (Đăng Ký Tên Miền)
│   ├── Đăng ký domain mới
│   ├── Transfer domain vào Route 53
│   └── Tự động tạo Hosted Zone
│
├── Hosted Zones (Vùng Lưu Trữ DNS)
│   ├── Public Hosted Zone — truy cập từ internet
│   └── Private Hosted Zone — nội bộ VPC
│
├── DNS Records (Bản Ghi DNS)
│   ├── A, AAAA, CNAME, MX, TXT, NS, SOA
│   └── Alias — đặc biệt của AWS
│
├── Routing Policies (Chính Sách Định Tuyến)
│   ├── Simple (Đơn Giản)
│   ├── Weighted (Có Trọng Số)
│   ├── Latency-based (Dựa Trên Độ Trễ)
│   ├── Failover (Chuyển Đổi Dự Phòng)
│   ├── Geolocation (Vị Trí Địa Lý)
│   ├── Geoproximity (Khoảng Cách Địa Lý)
│   └── Multi-value Answer (Đa Giá Trị)
│
├── Health Checks (Kiểm Tra Sức Khỏe)
│   ├── HTTP/HTTPS endpoint checks
│   ├── TCP checks
│   └── Calculated health checks
│
└── Route 53 Resolver (Bộ Phân Giải)
    ├── Inbound Endpoints (Điểm Cuối Vào)
    └── Outbound Endpoints (Điểm Cuối Ra)
```

---

## 📊 Route 53 vs DNS Thông Thường

| Tính Năng | DNS Thông Thường | Route 53 |
|-----------|-----------------|---------|
| SLA (Cam Kết Khả Dụng) | Tùy nhà cung cấp | **100% uptime SLA** |
| Routing thông minh | Không | Có (7 policies) |
| Health checks tích hợp | Không | Có |
| Tích hợp AWS services | Không | Native (ALB, CloudFront, S3) |
| Private DNS trong VPC | Không | Có |
| Anycast routing | Không | Có (toàn cầu) |
| Quản lý domain | Không | Có (registrar) |

---

## 🌍 Phân Phối Toàn Cầu

Route 53 hoạt động trên mạng **Anycast** — mỗi DNS query được phục vụ bởi **edge location gần nhất**:

```
Người dùng ở Hà Nội  → Kết nối đến edge location Singapore/Tokyo
Người dùng ở London  → Kết nối đến edge location Ireland/Frankfurt
Người dùng ở New York → Kết nối đến edge location Ashburn/New York
```

**Kết quả:** Độ trễ DNS thấp — thường dưới **5ms** ở Việt Nam.

---

## 💰 Chi Phí Tham Khảo (2024)

| Thành Phần | Giá |
|-----------|-----|
| Public Hosted Zone | $0.50/tháng |
| Private Hosted Zone | $0.50/tháng/VPC |
| DNS Queries (Standard) | $0.40/triệu queries đầu tiên |
| DNS Queries (Latency/Geo) | $0.70/triệu queries |
| Health Check (AWS endpoint) | $0.50/tháng |
| Health Check (non-AWS) | $0.75/tháng |

> **Lưu ý:** Domain registration tính riêng theo từng TLD (Top-Level Domain — Tên Miền Cấp Cao Nhất), thường $12-15/năm cho `.com`.

---

## 🔑 Điểm Khác Biệt Quan Trọng Khi Phỏng Vấn

### 1. Alias vs CNAME

```
CNAME: Không thể dùng ở zone apex (root domain)
       example.com → KHÔNG thể dùng CNAME
       www.example.com → Có thể dùng CNAME

Alias: Có thể dùng ở zone apex
       example.com → Có thể dùng Alias trỏ đến ALB/CloudFront
       Không tính phí DNS query (free!)
       Tự động cập nhật khi IP của target thay đổi
```

### 2. TTL (Time To Live — Thời Gian Sống) ảnh hưởng đến failover

```
TTL cao (86400 giây = 24 giờ):
  → Cache lâu hơn → Ít query đến Route 53 → Tốn ít tiền
  → Nhưng khi failover: DNS changes mất đến 24 giờ để có hiệu lực!

TTL thấp (60 giây):
  → Cache ngắn → Nhiều query → Tốn tiền hơn
  → Failover nhanh hơn — chỉ mất ~60 giây

Best practice: Giảm TTL xuống thấp (60-300s) trước khi thực hiện maintenance
```

### 3. Health Check + Failover là combo quan trọng

```
Route 53 Health Check theo dõi endpoint
         ↓
Nếu endpoint unhealthy (không khỏe)
         ↓
Route 53 TỰ ĐỘNG chuyển traffic sang endpoint khác
         ↓
Không cần can thiệp thủ công
```

---

## 🗺️ Lộ Trình Học Section Này

### Ngày 1: DNS Fundamentals
1. Đọc `1-dns-fundamentals.md` — hiểu cách DNS hoạt động
2. Thực hành: `dig` và `nslookup` các domain
3. Hiểu TTL và resolver caching

### Ngày 2: Hosted Zones & Records
1. Đọc `2-hosted-zones.md`
2. Tạo Public Hosted Zone trên AWS Console
3. Thêm các record types khác nhau
4. Test split-horizon DNS với Private Hosted Zone

### Ngày 3-4: Routing Policies
1. Đọc `3-routing-policies.md` — đây là phần quan trọng nhất
2. Thực hành từng policy một
3. Vẽ sơ đồ quyết định: "Khi nào dùng policy nào?"

### Ngày 5: Health Checks
1. Đọc `4-health-checks.md`
2. Tạo health check cho endpoint thực tế
3. Cấu hình Failover routing với health check

### Ngày 6: Resolver (Hybrid DNS)
1. Đọc `5-resolver.md`
2. Hiểu kiến trúc Inbound/Outbound Resolver Endpoints
3. Lab: Resolve tên miền nội bộ từ on-premises

---

## ✅ Checklist Tự Đánh Giá

Sau khi học xong section này, bạn có thể:

- [ ] Giải thích DNS resolution chain từ browser đến authoritative server
- [ ] Tạo và quản lý Hosted Zone (Public & Private)
- [ ] Cấu hình tất cả 7 routing policies với use case phù hợp
- [ ] Phân biệt Alias record và CNAME record
- [ ] Thiết kế DNS failover tự động với Health Checks
- [ ] Cấu hình split-horizon DNS (tên miền khác nhau cho internal/external)
- [ ] Thiết lập Route 53 Resolver cho hybrid DNS resolution
- [ ] Tính toán và chọn TTL phù hợp cho từng trường hợp

---

## 🔗 Điều Hướng

- ← [03-load-balancing/](../03-load-balancing/README.md) — Cân bằng tải
- → [05-cdn-cloudfront/](../05-cdn-cloudfront/README.md) — CloudFront CDN
- [INDEX.md](../INDEX.md) — Chỉ mục đầy đủ

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Phiên Bản:** 1.0
