# DNS Fundamentals — Nền Tảng DNS

> DNS (Domain Name System — Hệ Thống Tên Miền) là "danh bạ điện thoại" của internet — dịch tên miền dễ nhớ (như `example.com`) thành địa chỉ IP mà máy tính có thể hiểu.

## 📚 Mục Lục

1. [DNS Là Gì?](#dns-là-gì)
2. [Quá Trình Phân Giải DNS](#quá-trình-phân-giải-dns)
3. [Các Loại DNS Server](#các-loại-dns-server)
4. [Record Types — Loại Bản Ghi](#record-types--loại-bản-ghi)
5. [TTL — Time To Live](#ttl--time-to-live)
6. [DNS trong AWS](#dns-trong-aws)
7. [Công Cụ Debug DNS](#công-cụ-debug-dns)

---

## DNS Là Gì?

### Vấn Đề DNS Giải Quyết

Máy tính giao tiếp bằng địa chỉ IP (`192.168.1.1`), nhưng con người nhớ tên (`google.com`). DNS là lớp ánh xạ trung gian:

```
Người dùng gõ: google.com
         ↓ DNS resolution
Máy tính hiểu: 142.250.185.46
```

### Kiến Trúc Phân Cấp DNS

DNS được tổ chức theo **cây phân cấp** (hierarchical tree):

```
. (Root — Gốc)
├── .com
│   ├── google.com
│   │   ├── www.google.com
│   │   └── mail.google.com
│   └── amazon.com
├── .vn
│   └── vnexpress.vn
└── .org
    └── wikipedia.org
```

**Điều quan trọng:** Mỗi cấp độ được quản lý bởi một **authoritative name server** (máy chủ tên miền có thẩm quyền) khác nhau.

---

## Quá Trình Phân Giải DNS

### Resolver Chain (Chuỗi Phân Giải)

```
Trình duyệt muốn truy cập www.example.com

Bước 1: Kiểm tra browser cache
        → Có cache? Dùng luôn. Không? Tiếp tục.

Bước 2: Kiểm tra OS cache (/etc/hosts, DNS cache)
        → Có cache? Dùng luôn. Không? Tiếp tục.

Bước 3: Hỏi Recursive Resolver (thường là DNS của ISP hoặc 8.8.8.8)
        → Có cache? Trả về luôn. Không? Tiếp tục.

Bước 4: Recursive Resolver hỏi Root Name Server
        → "Ai quản lý .com?"
        → Root Server trả lời: "Hỏi TLD Server tại 192.5.6.30"

Bước 5: Recursive Resolver hỏi TLD Name Server (.com)
        → "Ai quản lý example.com?"
        → TLD Server trả lời: "Hỏi Authoritative Server tại ns1.example.com"

Bước 6: Recursive Resolver hỏi Authoritative Name Server
        → "IP của www.example.com là gì?"
        → Authoritative Server trả lời: "93.184.216.34"

Bước 7: Recursive Resolver cache kết quả (theo TTL)
        và trả về IP cho trình duyệt

Bước 8: Trình duyệt kết nối đến 93.184.216.34
```

### Sơ Đồ Phân Giải

```
Browser → Local Resolver → Root Server (13 cặp root servers toàn cầu)
                        → TLD Server (.com, .vn, .org)
                        → Authoritative Server (ns1.example.com)
                        ← IP Address
         ← IP Address ←
```

**Key insight:** Sau lần đầu tiên, kết quả được **cache** ở nhiều tầng — nên DNS rất nhanh cho các lần truy vấn sau.

---

## Các Loại DNS Server

### 1. Recursive Resolver (Bộ Phân Giải Đệ Quy)

- Còn gọi là **DNS Resolver** hoặc **Recursive Nameserver**
- Nhận query từ client, thay mặt client hỏi các server khác
- Cache kết quả để phục vụ các query tương tự nhanh hơn
- Ví dụ: Google `8.8.8.8`, Cloudflare `1.1.1.1`, DNS của ISP

### 2. Root Name Server (Máy Chủ Tên Miền Gốc)

- 13 cặp root server (A đến M): `a.root-servers.net` đến `m.root-servers.net`
- Thực tế có **hàng trăm** server vật lý nhờ Anycast routing
- Biết vị trí TLD servers, không biết địa chỉ IP cụ thể

### 3. TLD Name Server (Máy Chủ Tên Miền Cấp Cao)

- Quản lý từng TLD: `.com`, `.vn`, `.org`, `.net`...
- Biết Authoritative Name Servers cho từng domain trong TLD đó
- Được quản lý bởi: Verisign (`.com`), VNNIC (`.vn`)...

### 4. Authoritative Name Server (Máy Chủ Có Thẩm Quyền)

- Server cuối cùng, chứa **DNS records thực tế**
- Trả về IP address hoặc record tương ứng
- Với Route 53: AWS quản lý authoritative servers trên toàn cầu
- Ví dụ với Route 53: `ns-123.awsdns-45.com`

---

## Record Types — Loại Bản Ghi

### A Record (Address Record — Bản Ghi Địa Chỉ IPv4)

```
Tên: www.example.com
Loại: A
Giá trị: 93.184.216.34
TTL: 300

→ Ánh xạ hostname sang địa chỉ IPv4
```

### AAAA Record (IPv6 Address Record)

```
Tên: www.example.com
Loại: AAAA
Giá trị: 2606:2800:220:1:248:1893:25c8:1946
TTL: 300

→ Ánh xạ hostname sang địa chỉ IPv6
```

### CNAME Record (Canonical Name — Tên Chính Tắc)

```
Tên: blog.example.com
Loại: CNAME
Giá trị: www.example.com
TTL: 300

→ Tạo alias — trỏ một tên miền đến tên miền khác
→ KHÔNG thể dùng ở zone apex (example.com trực tiếp)
→ Khi resolve, phải query thêm một bước để lấy IP của target
```

**Hạn chế của CNAME:**
- Không dùng được tại root domain (`example.com`)
- Tốn thêm một DNS lookup

### Alias Record (Đặc Biệt của Route 53)

```
Tên: example.com (zone apex!)
Loại: Alias
Giá trị: my-alb-123456.us-east-1.elb.amazonaws.com
TTL: Tự động (không cấu hình)

→ Giải quyết vấn đề của CNAME tại zone apex
→ Tự động resolve IP khi target thay đổi
→ Miễn phí DNS queries
→ Chỉ trỏ được đến AWS services
```

**Alias có thể trỏ đến:**
- ALB (Application Load Balancer)
- NLB (Network Load Balancer)
- CloudFront distribution
- S3 static website endpoint
- Elastic Beanstalk environment
- API Gateway
- VPC Interface endpoints
- Route 53 record trong cùng hosted zone

### MX Record (Mail Exchange — Bản Ghi Mail)

```
Tên: example.com
Loại: MX
Giá trị: 10 mail1.example.com
         20 mail2.example.com
TTL: 3600

→ Chỉ định mail server nhận email cho domain
→ Số nhỏ = ưu tiên cao hơn (10 ưu tiên hơn 20)
```

### TXT Record (Text Record — Bản Ghi Văn Bản)

```
Tên: example.com
Loại: TXT
Giá trị: "v=spf1 include:_spf.google.com ~all"
TTL: 3600

→ Lưu thông tin văn bản tùy ý
→ Dùng để verify domain ownership (SPF, DKIM, DMARC)
→ Dùng bởi Google Search Console, AWS Certificate Manager để verify
```

### NS Record (Name Server — Bản Ghi Name Server)

```
Tên: example.com
Loại: NS
Giá trị: ns-123.awsdns-45.com
         ns-456.awsdns-67.net
         ns-789.awsdns-89.org
         ns-012.awsdns-01.co.uk
TTL: 172800 (2 ngày)

→ Chỉ định Authoritative Name Servers cho domain
→ Route 53 tạo 4 NS records tự động khi tạo Hosted Zone
→ PHẢI cập nhật ở domain registrar để trỏ về Route 53
```

### SOA Record (Start of Authority — Bản Ghi Quyền Hạn)

```
Tên: example.com
Loại: SOA
Giá trị: ns-123.awsdns-45.com. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400

→ Thông tin về zone: primary nameserver, admin email, serial number
→ Route 53 tự động tạo, thường không cần chỉnh sửa
```

### PTR Record (Pointer Record — Bản Ghi Con Trỏ)

```
Tên: 34.216.184.93.in-addr.arpa
Loại: PTR
Giá trị: www.example.com
TTL: 300

→ Reverse DNS — ánh xạ IP về hostname
→ Dùng cho email deliverability và logging
→ Thường ISP hoặc AWS quản lý
```

### SRV Record (Service Record — Bản Ghi Dịch Vụ)

```
Tên: _sip._tcp.example.com
Loại: SRV
Giá trị: 10 20 5060 sip.example.com
TTL: 300

→ Chỉ định host và port cho specific services
→ Dùng với VoIP (SIP), XMPP, Kubernetes service discovery
```

---

## TTL — Time To Live

### TTL Là Gì?

TTL (Time To Live — Thời Gian Sống) là **thời gian tính bằng giây** mà DNS record được cache ở các resolver trước khi cần phải query lại authoritative server.

### Ảnh Hưởng của TTL

```
TTL = 86400 (24 giờ):
✅ Ưu điểm: Ít DNS queries → Giảm latency → Tiết kiệm chi phí Route 53
❌ Nhược điểm: Sau khi thay đổi DNS, mất tối đa 24 giờ để có hiệu lực toàn cầu

TTL = 300 (5 phút):
✅ Ưu điểm: Thay đổi DNS có hiệu lực nhanh (trong vòng 5 phút)
❌ Nhược điểm: Nhiều queries hơn → Tốn tiền hơn với Route 53

TTL = 60 (1 phút):
✅ Dùng khi cần failover nhanh hoặc đang maintenance
❌ Không nên dùng thường xuyên vì tốn chi phí và tăng latency
```

### Best Practices cho TTL

| Tình Huống | TTL Khuyến Nghị |
|-----------|----------------|
| DNS ổn định, ít thay đổi | 86400 (24 giờ) |
| DNS thông thường | 3600 (1 giờ) |
| Cần thay đổi linh hoạt | 300 (5 phút) |
| Trước khi migration | Giảm xuống 60 |
| Khi cấu hình failover | 60-300 |
| Health check + failover active | 60 |

### Chiến Lược Khi Thay Đổi DNS

```
Quy trình đúng khi thay đổi DNS:

Tuần trước migration:
  Giảm TTL từ 86400 → 300 (5 phút)
  → Đợi ít nhất 24 giờ để cache cũ hết hạn

Ngày migration:
  Thực hiện thay đổi DNS record
  → Chỉ cần đợi 5 phút để có hiệu lực

Sau migration (vài ngày):
  Tăng TTL trở lại 3600 hoặc 86400
  → Tối ưu chi phí và hiệu suất
```

---

## DNS trong AWS

### Mặc Định: AmazonProvidedDNS

Mỗi VPC (Virtual Private Cloud) có DNS server mặc định tại địa chỉ:
- `VPC_CIDR + 2` (ví dụ: VPC `10.0.0.0/16` → DNS tại `10.0.0.2`)
- Còn gọi là **Route 53 Resolver** hoặc **AmazonProvidedDNS**

```
EC2 instance tạo DNS query
    ↓
Route 53 Resolver (10.0.0.2)
    ↓
Nếu là tên miền AWS nội bộ (*.compute.internal) → Resolve nội bộ
Nếu là tên miền public → Forward ra internet qua Route 53
```

### DNS Hostname trong VPC

Hai tùy chọn DNS quan trọng cần bật trong VPC settings:

```
enableDnsHostnames = true
→ EC2 instances nhận DNS hostname
  (ví dụ: ec2-12-34-56-78.compute-1.amazonaws.com)

enableDnsSupport = true
→ VPC sử dụng Route 53 Resolver
→ Cần bật để Private Hosted Zone hoạt động
```

### Route 53 Resolver Endpoints

Cho môi trường **hybrid** (AWS + on-premises):

```
Inbound Endpoint (Điểm Cuối Vào):
  On-premises DNS → Resolver Endpoint → Route 53 Private Zone
  (Cho phép on-premises resolve các tên miền AWS nội bộ)

Outbound Endpoint (Điểm Cuối Ra):
  EC2 → Resolver Endpoint → On-premises DNS Server
  (Cho phép EC2 resolve các tên miền on-premises)
```

---

## Công Cụ Debug DNS

### dig (Domain Information Groper — Công Cụ Truy Vấn DNS)

```bash
# Query A record cơ bản
dig example.com

# Query loại record cụ thể
dig example.com MX
dig example.com NS
dig example.com TXT

# Query trực tiếp từ specific nameserver
dig @8.8.8.8 example.com

# Xem toàn bộ thông tin trace
dig +trace example.com

# Query ngắn gọn (chỉ hiện kết quả)
dig +short example.com

# Reverse DNS lookup
dig -x 8.8.8.8

# Query Route 53 authoritative server
dig @ns-123.awsdns-45.com example.com
```

### nslookup

```bash
# Query cơ bản
nslookup example.com

# Query với specific DNS server
nslookup example.com 8.8.8.8

# Interactive mode
nslookup
> set type=MX
> example.com
```

### host

```bash
# Query đơn giản
host example.com

# Query loại record
host -t MX example.com

# Reverse lookup
host 8.8.8.8
```

### Kiểm Tra DNS Record Trên AWS

```bash
# Sau khi tạo record trong Route 53, kiểm tra bằng:
dig @ns-XXX.awsdns-YY.com www.example.com

# Kiểm tra propagation (lan truyền) toàn cầu
# Dùng trang web: whatsmydns.net hoặc dnschecker.org
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Giải thích quá trình DNS resolution từ đầu đến cuối

**Trả lời:**
1. Browser kiểm tra cache → OS cache → `/etc/hosts`
2. Query đến Recursive Resolver (ISP hoặc `8.8.8.8`)
3. Recursive Resolver hỏi Root Server → nhận địa chỉ TLD Server
4. Hỏi TLD Server (.com) → nhận địa chỉ Authoritative Server
5. Hỏi Authoritative Server → nhận IP address
6. Cache kết quả theo TTL và trả về cho browser
7. Browser thiết lập TCP connection đến IP nhận được

### Câu 2: Sự khác biệt giữa CNAME và Alias trong Route 53?

**Trả lời:**
- **CNAME:** Record DNS tiêu chuẩn, không dùng được ở zone apex, tốn một DNS lookup thêm, có thể trỏ đến bất kỳ hostname nào
- **Alias:** Đặc biệt của Route 53, dùng được ở zone apex, resolve nhanh hơn (Route 53 resolve nội bộ), miễn phí DNS queries, chỉ trỏ được đến AWS services

### Câu 3: TTL ảnh hưởng như thế nào đến DNS failover?

**Trả lời:** TTL cao → cache lâu → khi failover diễn ra, người dùng vẫn được resolve ra IP cũ trong vòng TTL giây → ảnh hưởng đến recovery time. Cần giảm TTL xuống thấp (60-300s) trước khi cấu hình failover hoặc trước maintenance window.

### Câu 4: Sự khác biệt giữa Recursive Resolver và Authoritative Nameserver?

**Trả lời:**
- **Recursive Resolver:** Thực hiện toàn bộ quá trình lookup thay mặt client, cache kết quả, là trung gian
- **Authoritative Nameserver:** Lưu trữ DNS records thực tế, trả lời "câu hỏi cuối cùng", là nguồn sự thật

---

## 🔗 Điều Hướng

- ← [README.md](./README.md) — Tổng quan DNS & Route 53
- → [2-hosted-zones.md](./2-hosted-zones.md) — Public & Private Hosted Zones
- [INDEX.md](../INDEX.md) — Chỉ mục đầy đủ

---

**Cập Nhật Lần Cuối:** 2026-05-14
