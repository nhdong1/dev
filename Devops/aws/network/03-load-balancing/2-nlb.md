# NLB — Network Load Balancer — Cân Bằng Tải Mạng

> NLB (Network Load Balancer — Cân Bằng Tải Mạng) hoạt động ở Layer 4 (tầng giao vận) của mô hình OSI, xử lý lưu lượng TCP, UDP, TLS với độ trễ cực thấp (ultra-low latency — ~100 microseconds) và thông lượng cực cao (millions of RPS — requests per second). NLB duy trì địa chỉ IP tĩnh và bảo toàn địa chỉ IP nguồn của client.

## 🏗️ Kiến Trúc NLB

### Cơ Chế Hoạt Động

```
Client (1.2.3.4)
    │
    ▼  TCP SYN đến NLB IP (54.x.x.x)
NLB Node (AZ-A) — Elastic IP: 54.x.x.x
    │  Connection forwarded (NOT proxied)
    │  Source IP preserved: 1.2.3.4
    ▼
EC2 Target (10.0.1.100:8080)
    │  Sees client real IP
    ▼
Application (ghi log IP thật của client)
```

### Sự Khác Biệt Về Cách Xử Lý Connection

```
ALB (Proxy Model — Mô Hình Proxy):
Client ──TCP connection──► ALB ──TCP connection──► Target
- ALB tạo 2 TCP connections riêng biệt
- Target thấy IP của ALB, không phải IP client
- Client IP có trong header X-Forwarded-For

NLB (Pass-through Model — Mô Hình Chuyển Tiếp):
Client ──TCP connection──────────────────────────► Target
- NLB forward packet, KHÔNG tạo TCP connection mới
- Target thấy IP thật của client (source IP preserved)
- Không cần đọc header để biết client IP
```

---

## 🚀 Tính Năng Nổi Bật Của NLB

### Ultra-Low Latency (Độ Trễ Cực Thấp)

```
ALB: ~400ms connection setup time
NLB: ~100 microseconds (μs)

NLB nhanh hơn ALB ~4000 lần về latency!

Lý do: NLB không cần:
- Parse HTTP headers
- Inspect URL path
- Match routing rules
- Maintain HTTP session state

NLB chỉ cần: Đọc destination IP:Port → Forward packet
```

### Static IP / Elastic IP Per AZ

```
Mỗi AZ mà NLB hoạt động → 1 static IP (có thể gán Elastic IP):

NLB trên 2 AZs:
├── AZ-A: IP 54.100.1.1 (Elastic IP có thể gán)
└── AZ-B: IP 54.100.2.2 (Elastic IP có thể gán)

Ứng dụng thực tế:
✅ Whitelist IP trong firewall on-premises: chỉ cần cho phép 2 IPs
✅ Clients cần biết trước IP để connect (không dùng DNS)
✅ Compliance (tuân thủ) yêu cầu IP cố định
✅ AWS PrivateLink endpoint services (bắt buộc dùng NLB)
```

### Preserve Source IP (Bảo Toàn IP Nguồn)

```
NLB mặc định: Source IP của client được giữ nguyên đến target
→ Ứng dụng nhận được IP thật của client mà không cần xử lý headers

Trường hợp ngoại lệ:
- Targets loại Instance: Client IP preserved ✅
- Targets loại IP: Client IP preserved ✅
- Targets qua VPC Peering: Client IP preserved ✅
- Khi bật Proxy Protocol v2: Client IP trong Proxy Protocol header

⚠️ Security Group trên targets phải allow traffic từ client IPs
   (không phải từ NLB IPs như trong ALB)
```

### TLS Termination (Kết Thúc TLS) Trên NLB

```
NLB hỗ trợ TLS termination kể từ 2019:
- NLB giải mã TLS, forward plaintext TCP đến target
- Dùng certificate từ ACM hoặc IAM
- Giảm tải TLS processing từ targets

Hoặc: TLS Pass-through (Chuyển Tiếp TLS)
- NLB forward encrypted TCP trực tiếp đến target
- Target tự giải mã TLS
- End-to-end encryption (mã hóa đầu cuối)
- NLB không thể inspect nội dung
```

---

## 📡 Giao Thức Hỗ Trợ

### TCP (Transmission Control Protocol)

```
Use cases:
- Database connections (MySQL :3306, PostgreSQL :5432)
- SSH (:22)
- Custom application protocols
- SMTP email servers (:25, :587)

Đặc điểm:
- Connection-oriented (hướng kết nối)
- Reliable delivery (giao hàng tin cậy)
- Flow control, congestion control (kiểm soát luồng, kiểm soát tắc nghẽn)
```

### UDP (User Datagram Protocol)

```
Use cases:
- DNS servers (:53)
- Game servers (latency-sensitive — nhạy cảm với độ trễ)
- VoIP (Voice over IP — Thoại qua Internet Protocol)
- Streaming media
- IoT sensor data

Đặc điểm:
- Connectionless (không hướng kết nối)
- No guaranteed delivery (không đảm bảo giao hàng)
- Lower overhead → thấp hơn về chi phí xử lý
- Ứng dụng tự xử lý packet loss nếu cần
```

### TLS (Transport Layer Security)

```
NLB TLS Listener:
- Port: 443, 8443, hoặc bất kỳ port nào
- Certificate: ACM (AWS Certificate Manager) hoặc IAM
- Security Policy: Giống ALB
- Alpn Policy: HTTP1Only, HTTP2Only, HTTP2Preferred, None

TLS Pass-through (không terminate tại NLB):
- Dùng TCP listener thay vì TLS listener
- NLB không có certificate
- Target servers tự manage certificates
```

---

## 🔧 Target Types (Loại Mục Tiêu) NLB

### So Sánh Instance vs IP vs ALB Target

```
1. Instance Target (Mục Tiêu Instance)
   ├── Target: EC2 Instance ID
   ├── Source IP: Client IP preserved (bảo toàn IP client)
   ├── Security Group: Cần allow từ client IP ranges
   └── Dùng khi: EC2 instances trong cùng VPC

2. IP Target (Mục Tiêu Địa Chỉ IP)
   ├── Target: IPv4 address (trong VPC, on-premises, hoặc peered VPC)
   ├── Source IP: Client IP preserved
   ├── Use case: On-premises servers qua Direct Connect/VPN
   └── Dùng khi: Targets không phải EC2 trong VPC này

3. ALB Target (Mục Tiêu ALB)
   ├── Target: Application Load Balancer
   ├── Mục đích: NLB → ALB → Targets
   ├── Use case: Cần static IP + Layer 7 routing
   └── Pattern: NLB (static IP) → ALB (path routing)
```

---

## 🌐 NLB Và AWS PrivateLink

### PrivateLink Endpoint Service (Dịch Vụ Endpoint Riêng Tư)

```
NLB là nền tảng bắt buộc cho AWS PrivateLink:

┌─ Provider VPC ────────────────────┐   ┌─ Consumer VPC ──────────────────┐
│                                   │   │                                  │
│  Service (EC2/ECS)                │   │  Consumer App                   │
│       │                           │   │       │                          │
│       ▼                           │   │       ▼                          │
│      NLB ←── Endpoint Service ────────────► VPC Interface Endpoint      │
│                                   │   │     (Private IP trong consumer) │
└───────────────────────────────────┘   └──────────────────────────────────┘

Lợi ích:
- Traffic không qua internet
- Consumer không cần biết IP/VPC của provider
- Provider control ai có thể connect (whitelist AWS account)
- Dùng cho: SaaS products trên AWS, chia sẻ internal services
```

---

## ⚡ Connection Termination (Kết Thúc Kết Nối)

### Idle Timeout (Thời Gian Chờ Không Hoạt Động)

```
NLB TCP Idle Timeout:
- Mặc định: 350 giây
- Không cấu hình được (khác ALB)
- Sau 350 giây không có packet → NLB gửi RST (reset) đóng connection

NLB UDP:
- Idle timeout: 120 giây (flow timeout)

Ảnh hưởng:
- Long-running TCP connections cần keepalive trước 350s
- Cấu hình TCP keepalive trên ứng dụng/OS
```

### Connection Draining (Thoát Kết Nối Dần)

```
Tương đương Deregistration Delay trong ALB:
- Khi target bị deregister → NLB chờ connections hiện tại kết thúc
- Mặc định: 300 giây
- Phạm vi: 0 – 3600 giây
- Nếu = 0: Connections bị ngắt ngay lập tức (không khuyến nghị)
```

---

## 🏋️ Performance (Hiệu Suất) NLB

### Throughput (Thông Lượng) Và Scaling

```
NLB tự động scale để xử lý:
- Millions of requests per second (hàng triệu request mỗi giây)
- Sudden traffic spikes (đột biến lưu lượng) — không cần pre-warming
- Kết nối TCP đồng thời (concurrent connections) cực lớn

Pre-warming (Làm Nóng Trước):
- ALB: Cần pre-warm nếu traffic tăng đột ngột (hoặc mở support ticket)
- NLB: KHÔNG cần pre-warming — scale ngay lập tức
→ NLB ưu việt hơn khi traffic có spikes không đoán trước
```

### Cross-Zone Load Balancing (Cân Bằng Tải Liên AZ)

```
Mặc định NLB: Cross-zone load balancing = TẮT (OFF)
Lý do: Bật sẽ phát sinh phí cross-AZ data transfer

Khi TẮT:
- Traffic vào AZ-A → chỉ route đến targets trong AZ-A
- Cần phân bổ targets đều giữa các AZs

Khi BẬT:
- Traffic vào AZ-A → route đến tất cả targets trong mọi AZs
- Phí data transfer giữa AZs: $0.01/GB
- Phù hợp khi targets phân bổ không đều giữa AZs

So với ALB: Cross-zone mặc định BẬT, không tính phí data transfer
```

---

## 🔐 Bảo Mật NLB

### Security Groups và NLB

```
⚠️ Quan trọng: NLB KHÔNG có Security Group (trước 2023)

Kể từ 2023: NLB hỗ trợ Security Groups (opt-in)
- Bật qua console hoặc CLI khi tạo NLB
- Cho phép kiểm soát traffic đến NLB

Khi KHÔNG có Security Group trên NLB:
- Traffic từ internet → NLB → Targets
- Targets cần allow traffic từ CLIENT IP ranges (không phải NLB IP)
- Hoặc: Allow từ NLB node IPs (tìm trong AWS documentation)

Best Practice: Bật Security Group trên NLB + Security Group trên Targets
```

### TLS Security Policy

```
Khuyến nghị dùng policy mới nhất:
- ELBSecurityPolicy-TLS13-1-2-2021-06: Hỗ trợ TLS 1.2 và 1.3
- ELBSecurityPolicy-TLS13-1-3-2021-06: Chỉ TLS 1.3 (restrictive nhất)

Tránh dùng:
- ELBSecurityPolicy-2016-08: Hỗ trợ TLS 1.0 (deprecated — lỗi thời)
- ELBSecurityPolicy-TLS-1-1-2017-01: Hỗ trợ TLS 1.1 (không an toàn)
```

---

## 🆚 ALB vs NLB — Bảng Quyết Định

```
Câu hỏi 1: Bạn cần routing dựa trên URL path hoặc HTTP headers không?
  → Có: ALB
  → Không: Tiếp tục

Câu hỏi 2: Bạn cần latency dưới 1ms không?
  → Có: NLB
  → Không: Tiếp tục

Câu hỏi 3: Bạn cần Static/Elastic IP không?
  → Có: NLB
  → Không: Tiếp tục

Câu hỏi 4: Ứng dụng dùng TCP/UDP không phải HTTP/HTTPS không?
  → Có: NLB
  → Không: ALB

Câu hỏi 5: Bạn cần AWS PrivateLink endpoint không?
  → Có: NLB (bắt buộc)
  → Không: ALB hoặc NLB đều được

Câu hỏi 6: Bạn cần WAF (Web Application Firewall) không?
  → Có: ALB (NLB không hỗ trợ WAF)
  → Không: NLB hoặc ALB
```

---

## 🏋️ Lab Thực Hành

### Tạo NLB Cho Game Server (TCP)

```bash
# Bước 1: Tạo NLB
aws elbv2 create-load-balancer \
  --name game-server-nlb \
  --type network \
  --subnets subnet-public-aza subnet-public-azb \
  --scheme internet-facing \
  --ip-address-type ipv4

# Bước 2: Tạo Target Group TCP
aws elbv2 create-target-group \
  --name game-servers-tg \
  --protocol TCP \
  --port 7777 \
  --vpc-id vpc-12345678 \
  --health-check-protocol TCP \
  --health-check-port 7777 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 3

# Bước 3: Tạo Listener TCP
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol TCP \
  --port 7777 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# Bước 4: Đăng ký game server instances
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-gameserver1 Id=i-gameserver2 Id=i-gameserver3
```

### Tạo NLB Với Elastic IP

```bash
# Bước 1: Tạo Elastic IPs cho mỗi AZ
EIP_AZA=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
EIP_AZB=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

# Bước 2: Tạo NLB với Elastic IPs
aws elbv2 create-load-balancer \
  --name static-ip-nlb \
  --type network \
  --subnet-mappings \
    SubnetId=subnet-public-aza,AllocationId=$EIP_AZA \
    SubnetId=subnet-public-azb,AllocationId=$EIP_AZB \
  --scheme internet-facing
```

---

## 💡 Best Practices (Thực Hành Tốt Nhất)

```
✅ DOs:
1. Dùng NLB khi cần Static IP cho firewall whitelisting
2. Bật Security Group trên NLB (tính năng mới 2023)
3. Cấu hình TCP keepalive trên ứng dụng < 350 giây (idle timeout NLB)
4. Bật Cross-zone load balancing nếu targets không đồng đều giữa AZs
5. Dùng TLS listener trên NLB để giảm tải TLS từ targets
6. Monitor ActiveFlowCount và NewFlowCount để phát hiện DDoS
7. Dùng NLB làm frontend cho PrivateLink endpoint services

❌ DON'Ts:
1. Đừng dùng NLB khi cần content-based routing (HTTP path/header)
2. Đừng dùng NLB khi cần WAF protection
3. Đừng dùng NLB khi Lambda là target
4. Đừng mở Security Group trên targets quá rộng khi dùng NLB
   (vì client IP được giữ nguyên, cần allow đúng CIDR)
5. Đừng quên phí cross-AZ khi bật Cross-zone load balancing
```

---

## 🎓 Câu Hỏi Phỏng Vấn Về NLB

**Q: Tại sao NLB nhanh hơn ALB nhiều như vậy?**

> NLB hoạt động ở Layer 4 (Transport Layer — Tầng Giao Vận), chỉ xử lý TCP/UDP headers: source IP, destination IP, port, flags. NLB không parse HTTP, không match routing rules, không manage HTTP sessions. NLB đơn giản nhận packet → tìm target → forward packet. Tổng thời gian xử lý ~100 microseconds. ALB phải parse HTTP headers, match listener rules, có thể decrypt TLS, inspect path/host → ~400ms. Đây là lý do NLB là lựa chọn duy nhất cho latency-critical applications.

**Q: Khi nào cần NLB thay vì ALB?**

> Ba trường hợp chính: (1) Ultra-low latency: game servers, financial trading, real-time bidding cần < 1ms response; (2) Static IP requirement: khi on-premises firewall cần whitelist IP cố định — NLB cung cấp Elastic IP per AZ; (3) Non-HTTP protocols: TCP gaming protocols, UDP DNS/VoIP, database connections, SSH. Ngoài ra, khi cần AWS PrivateLink endpoint service, NLB là bắt buộc.

**Q: Source IP preservation trong NLB là gì và tại sao quan trọng?**

> NLB chuyển tiếp packet mà không thay đổi source IP — target servers nhận được IP thật của client. Quan trọng vì: (1) Ứng dụng có thể implement IP-based rate limiting đúng client; (2) Audit logs ghi IP thật, không phải IP NLB; (3) Geo-restriction theo IP client; (4) Ứng dụng banking/security cần biết client IP cho fraud detection. Ngược lại với ALB, nơi target chỉ thấy IP của ALB, client IP phải đọc từ header X-Forwarded-For.

---

**Tiếp Theo:** [3-target-groups.md](3-target-groups.md) — Target Groups, Health Checks, Deregistration Delay

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn Thành
