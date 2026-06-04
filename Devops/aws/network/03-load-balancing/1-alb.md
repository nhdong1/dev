# ALB — Application Load Balancer — Cân Bằng Tải Ứng Dụng

> ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng) hoạt động ở Layer 7 (tầng ứng dụng) của mô hình OSI, cho phép routing (định tuyến) thông minh dựa trên nội dung request: URL path, hostname, HTTP headers, query strings. Đây là lựa chọn hàng đầu cho web applications và microservices.

## 🏗️ Kiến Trúc ALB

### Thành Phần Cốt Lõi

```
Internet / Clients
        │
        ▼
┌──────────────────────────────────────────────────┐
│                  ALB (Application Load Balancer) │
│                                                  │
│  Listener HTTPS :443                             │
│  ├── Rule 1: Host = api.example.com              │
│  │   └── Forward → Target Group: API Servers     │
│  ├── Rule 2: Path = /admin/*                     │
│  │   └── Forward → Target Group: Admin Panel     │
│  ├── Rule 3: Path = /static/*                    │
│  │   └── Redirect → S3 (hoặc CloudFront)         │
│  └── Default Rule                                │
│      └── Forward → Target Group: Web Servers     │
└──────────────────────────────────────────────────┘
        │
   ┌────┼────┐
   ▼    ▼    ▼
  AZ-A AZ-B AZ-C
  EC2  EC2  EC2
```

### Nodes Trong Mỗi AZ (Availability Zone — Vùng Sẵn Sàng)

```
ALB tự động tạo ALB Node trong mỗi AZ được chỉ định:
- ALB Node AZ-A: Subnet Public AZ-A
- ALB Node AZ-B: Subnet Public AZ-B

DNS của ALB resolve (phân giải) ra nhiều IPs → Round-robin giữa các nodes
Nodes giao tiếp với targets trong tất cả AZs (khi Cross-zone LB bật)
```

---

## 🎯 Listener (Bộ Lắng Nghe) & Rules (Quy Tắc)

### Listener Configuration (Cấu Hình Listener)

```
Listener = Port + Protocol mà ALB lắng nghe

Cấu hình phổ biến:
┌─────────────────────────────────────────┐
│ Listener 1: HTTP :80                    │
│ └── Action: Redirect to HTTPS :443      │
│                                         │
│ Listener 2: HTTPS :443                  │
│ └── SSL Certificate: ACM cert           │
│ └── Security Policy: ELBSecurityPolicy  │
│ └── Rules: (xem bên dưới)               │
└─────────────────────────────────────────┘
```

### Rule Conditions (Điều Kiện Quy Tắc)

ALB hỗ trợ routing dựa trên các điều kiện sau:

```
1. HOST_HEADER (Tiêu Đề Host)
   Điều kiện: Host = "api.example.com"
   Dùng cho: Multi-tenant apps, subdomain routing

2. PATH_PATTERN (Mẫu Đường Dẫn)
   Điều kiện: Path = "/api/*" hoặc "/v1/users"
   Dùng cho: Microservices routing

3. HTTP_HEADER (Tiêu Đề HTTP)
   Điều kiện: Header "X-Custom-Header" = "internal"
   Dùng cho: A/B testing, internal routing

4. HTTP_REQUEST_METHOD (Phương Thức HTTP)
   Điều kiện: Method = "POST" hoặc "GET"
   Dùng cho: Read/Write splitting

5. QUERY_STRING (Chuỗi Truy Vấn)
   Điều kiện: Query "version=2"
   Dùng cho: API versioning, feature flags

6. SOURCE_IP (Địa Chỉ IP Nguồn)
   Điều kiện: IP trong CIDR "10.0.0.0/8"
   Dùng cho: Internal traffic routing, IP allowlisting
```

### Rule Actions (Hành Động Quy Tắc)

```
1. FORWARD (Chuyển Tiếp)
   └── Gửi request đến Target Group
   └── Hỗ trợ Weighted Target Groups (có trọng số)

2. REDIRECT (Chuyển Hướng)
   └── HTTP → HTTPS (301/302)
   └── Có thể thay đổi host, path, port, query string

3. FIXED_RESPONSE (Phản Hồi Cố Định)
   └── Trả về HTTP response trực tiếp từ ALB
   └── Dùng cho maintenance page, health check response

4. AUTHENTICATE_OIDC (Xác Thực OIDC)
   └── Tích hợp với IdP: Google, Okta, Auth0, Cognito
   └── Yêu cầu đăng nhập trước khi truy cập ứng dụng

5. AUTHENTICATE_COGNITO (Xác Thực Cognito)
   └── Tích hợp trực tiếp với Amazon Cognito User Pools
```

### Priority (Thứ Tự Ưu Tiên) Của Rules

```
Rules được đánh giá theo thứ tự priority (số nhỏ hơn → ưu tiên cao hơn):

Priority 1: Host = "api.example.com" AND Path = "/v2/*" → Forward API v2 TG
Priority 2: Host = "api.example.com" → Forward API TG
Priority 3: Path = "/admin/*" → Forward Admin TG
Priority 4: Path = "/static/*" → Redirect to CloudFront
Default:    → Forward Web TG

⚠️ Default Rule luôn được đánh giá cuối cùng
```

---

## 🌐 Tính Năng ALB Nổi Bật

### Content-Based Routing (Định Tuyến Dựa Trên Nội Dung)

```
Ví dụ thực tế — E-commerce App:

alb-ecommerce.example.com/
├── /api/products → Product Service (ECS containers)
├── /api/orders   → Order Service (EC2 instances)
├── /api/payment  → Payment Service (Lambda)
├── /admin        → Admin Panel (EC2, internal only)
├── /static       → Redirect to CloudFront (S3)
└── /             → Frontend React App (EC2/ECS)
```

### WebSocket Support (Hỗ Trợ WebSocket)

```
ALB hỗ trợ WebSocket và WebSocket Secure (WSS):
- HTTP Upgrade request → ALB duy trì kết nối persistent
- Idle timeout mặc định: 60 giây (có thể tăng lên 4000 giây)
- Dùng cho: Chat apps, real-time dashboards, live notifications
```

### gRPC Support (Hỗ Trợ gRPC)

```
ALB hỗ trợ gRPC (Protocol Buffers over HTTP/2):
- Cần bật HTTP/2 trên listener
- Content-Type: application/grpc
- Dùng cho: Microservices giao tiếp nội bộ, mobile backends
```

### HTTP/2 Support

```
ALB hỗ trợ HTTP/2 từ client đến ALB:
- Connection Multiplexing (Ghép Kênh): nhiều request trên 1 TCP connection
- Header Compression (Nén Header): giảm overhead
- ALB → Targets: HTTP/1.1 hoặc HTTP/2 (cấu hình trên Target Group)
```

### Slow Start Mode (Chế Độ Khởi Động Chậm)

```
Slow Start = Tính năng đưa target vào traffic dần dần

Cấu hình: 30–900 giây
Mục đích:
- Cho phép target "khởi động" (warm up) JVM, cache, connection pool
- Tránh bị overloaded ngay khi vừa được đăng ký vào Target Group
- Sau khi hết thời gian slow start → nhận traffic đầy đủ

Use case: Java Spring Boot app với JVM warm-up, apps có cold-start
```

---

## 🔀 Weighted Target Groups (Nhóm Mục Tiêu Có Trọng Số)

### Blue/Green Deployment (Triển Khai Xanh/Xanh Lá)

```
Listener Rule Forward:
├── Target Group BLUE (v1.0): weight = 100
└── Target Group GREEN (v2.0): weight = 0

Sau khi deploy v2.0:
Step 1: weight BLUE=90, GREEN=10   (canary 10%)
Step 2: weight BLUE=50, GREEN=50   (50/50 split)
Step 3: weight BLUE=0,  GREEN=100  (full cutover)

Rollback nhanh: Đổi weight BLUE=100, GREEN=0
```

### Canary Release (Phát Hành Canary)

```
Phát hành tính năng mới cho % nhỏ users:
- STABLE: weight=95 → 95% traffic
- CANARY: weight=5  → 5% traffic để test

Theo dõi: Error rate, latency của canary target group
Quyết định: Rollout hoặc Rollback dựa trên metrics
```

---

## 🔒 Authentication (Xác Thực) Tích Hợp

### OIDC Authentication (Xác Thực OpenID Connect)

```
Cấu hình trên ALB Listener Rule:
Action: AUTHENTICATE_OIDC
  - Authorization Endpoint
  - Token Endpoint
  - User Info Endpoint
  - Client ID / Client Secret
  - Scope: "openid email profile"

Flow:
1. User gửi request đến ALB
2. ALB redirect đến IdP login page
3. User đăng nhập → IdP trả về authorization code
4. ALB đổi code lấy token → lưu vào session cookie
5. ALB forward request (kèm user info headers) đến target
```

### Cognito Authentication (Xác Thực Amazon Cognito)

```
Tích hợp với Amazon Cognito User Pools:
- Không cần code xác thực trong ứng dụng
- ALB xử lý toàn bộ auth flow
- Targets nhận header X-Amzn-Oidc-Identity, X-Amzn-Oidc-Accesstoken
- Phù hợp cho: Internal tools, admin portals
```

---

## 📊 Monitoring ALB

### CloudWatch Metrics Quan Trọng

```
Request Metrics:
- RequestCount: Tổng số requests trong khoảng thời gian
- ActiveConnectionCount: Số kết nối TCP đang hoạt động
- NewConnectionCount: Kết nối mới mỗi phút

Latency Metrics:
- TargetResponseTime: Thời gian từ khi ALB gửi request đến target
  đến khi nhận response đầu tiên
  → P99 > 1s: Cần tối ưu ứng dụng
  → P50 > 200ms: Kiểm tra database query

Error Metrics:
- HTTPCode_ELB_4XX_Count: Client errors (4xx từ ALB)
- HTTPCode_ELB_5XX_Count: ALB errors (5xx do ALB)
- HTTPCode_Target_4XX_Count: Client errors (4xx từ target)
- HTTPCode_Target_5XX_Count: Server errors (5xx từ target)
  → Cao bất thường: Vấn đề ứng dụng

Health Check Metrics:
- HealthyHostCount: Số target healthy trong Target Group
- UnHealthyHostCount: Số target unhealthy
  → UnHealthyHostCount > 0: Alert ngay
```

### ALB Access Logs (Nhật Ký Truy Cập ALB)

```
Bật trong Attributes → Access Logs → S3 bucket

Format log entry:
https 2026-05-14T10:23:45Z app/my-alb/1234567890abcdef \
  1.2.3.4:5678 10.0.1.100:8080 \
  0.000 0.032 0.000 \
  200 200 \
  495 10680 \
  "GET https://example.com:443/api/users HTTP/1.1" \
  "Mozilla/5.0..." \
  ECDHE-RSA-AES128-GCM-SHA256 TLSv1.2

Phân tích bằng Amazon Athena:
SELECT elb_status_code, COUNT(*) as count
FROM alb_access_logs
WHERE time > '2026-05-14'
GROUP BY elb_status_code
ORDER BY count DESC;
```

---

## ⚙️ Cấu Hình ALB Quan Trọng

### Idle Timeout (Thời Gian Chờ Không Hoạt Động)

```
Mặc định: 60 giây
Phạm vi: 1 – 4000 giây

Khi nào tăng:
- WebSocket connections cần persistent connections
- File upload lớn
- Long-polling requests

Cấu hình:
ALB → Attributes → Idle timeout = <seconds>

⚠️ Cũng phải tăng keepalive timeout trên web server (Nginx/Apache)
   để tránh "502 Bad Gateway" do server đóng kết nối trước ALB
```

### Deletion Protection (Bảo Vệ Xóa)

```
Bật để ngăn xóa nhầm ALB:
ALB → Attributes → Deletion protection → Enable

Bắt buộc tắt trước khi xóa ALB
```

### Desync Mitigation Mode (Chế Độ Giảm Thiểu Desync)

```
Defensive mode (mặc định): Từ chối request không tuân thủ HTTP spec
Monitor mode: Ghi log nhưng cho phép request
Strictest mode: Từ chối request nghiêm ngặt nhất

Khuyến nghị: Dùng Defensive cho production
```

---

## 🏋️ Lab Thực Hành

### Tạo ALB Bằng AWS CLI

```bash
# Bước 1: Tạo ALB
aws elbv2 create-load-balancer \
  --name my-web-alb \
  --type application \
  --subnets subnet-public-aza subnet-public-azb \
  --security-groups sg-alb-id \
  --scheme internet-facing \
  --ip-address-type ipv4

# Bước 2: Tạo Target Group
aws elbv2 create-target-group \
  --name web-servers-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-12345678 \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# Bước 3: Đăng ký targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-ec2instanceid1 Id=i-ec2instanceid2

# Bước 4: Tạo Listener HTTP → Redirect HTTPS
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTP \
  --port 80 \
  --default-actions \
    Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# Bước 5: Tạo Listener HTTPS
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:... \
  --default-actions \
    Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...

# Bước 6: Thêm path-based routing rule
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:... \
  --priority 10 \
  --conditions '[{"Field":"path-pattern","Values":["/api/*"]}]' \
  --actions '[{"Type":"forward","TargetGroupArn":"arn:aws:elasticloadbalancing:..."}]'
```

---

## 💡 Best Practices (Thực Hành Tốt Nhất)

```
✅ DOs:
1. Luôn redirect HTTP → HTTPS tại ALB listener
2. Dùng AWS ACM (Certificate Manager) cho SSL certs — miễn phí, tự động renew
3. Bật Access Logs → S3 để phân tích sau
4. Cấu hình Health Check path chuyên biệt (/health, /ping) — tách khỏi business logic
5. Bật Deletion Protection cho ALB production
6. Dùng Target Group với loại IP khi target là containers (Fargate)
7. Tăng idle timeout khi cần hỗ trợ WebSocket hoặc long-running connections
8. Monitor HealthyHostCount — alert khi < threshold

❌ DON'Ts:
1. Đừng mở Security Group trực tiếp đến EC2 instances từ internet
2. Đừng dùng CLB cho workload mới
3. Đừng bỏ qua TargetResponseTime metrics — dấu hiệu đầu tiên của vấn đề hiệu suất
4. Đừng đặt Health Check interval quá ngắn (overload target với health check traffic)
5. Đừng quên cấu hình Deregistration Delay phù hợp khi dùng Auto Scaling
```

---

## 🎓 Câu Hỏi Phỏng Vấn Về ALB

**Q: ALB hoạt động ở Layer mấy và điều đó có ý nghĩa gì?**

> ALB hoạt động ở Layer 7 (Application Layer — Tầng Ứng Dụng) của mô hình OSI. Điều này có nghĩa ALB có thể đọc và hiểu nội dung HTTP request: URL path, hostname, headers, method, query string. Khác với Layer 4 (NLB) chỉ nhìn thấy IP:Port và TCP flags. Nhờ Layer 7, ALB có thể routing thông minh: /api → API servers, /static → S3/CloudFront, api.example.com → API service.

**Q: Sự khác biệt giữa ALB và NLB?**

> ALB (Layer 7): Routing dựa trên HTTP content, hỗ trợ WebSocket/gRPC/HTTP2, latency ~400ms, không có Static IP, hỗ trợ Lambda target, tích hợp WAF. NLB (Layer 4): Routing dựa trên IP:Port, latency ~100μs (4000x nhanh hơn), có Static/Elastic IP, hỗ trợ TCP/UDP. Chọn ALB cho web apps/APIs, chọn NLB khi cần ultra-low latency hoặc Static IP.

**Q: Giải thích path-based routing với ví dụ thực tế?**

> Path-based routing cho phép một ALB phục vụ nhiều services dựa trên URL path. Ví dụ: `shop.example.com/products` → Product Catalog Service (10 EC2), `shop.example.com/cart` → Shopping Cart Service (ECS containers), `shop.example.com/payment` → Payment Service (Lambda). Mỗi service có Target Group riêng với Health Check riêng, scale độc lập, có thể deploy tách biệt — đây là nền tảng cho microservices architecture trên AWS.

**Q: Slow start mode là gì và khi nào cần dùng?**

> Slow start là tính năng giúp target mới vào Target Group nhận traffic tăng dần thay vì nhận đầy đủ ngay lập tức. Trong 30–900 giây (cấu hình được), ALB tăng dần tỷ lệ traffic đến target mới từ 0% lên 100%. Cần thiết cho: Java/Kotlin apps cần JVM warm-up, apps tạo connection pool đến database khi khởi động, hoặc bất kỳ app nào cần "làm nóng" trước khi xử lý full load. Giúp tránh spike errors khi Auto Scaling thêm instance mới.

---

**Tiếp Theo:** [2-nlb.md](2-nlb.md) — Network Load Balancer, Layer 4, Ultra-Low Latency

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn Thành
