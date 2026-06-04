# Target Groups, Health Checks & Deregistration Delay

> Target Group (Nhóm Mục Tiêu) là tập hợp các targets (EC2 instances, IPs, Lambda functions, hoặc ALBs) nhận lưu lượng từ Load Balancer. Health Checks (Kiểm Tra Sức Khỏe) xác định target nào đang hoạt động bình thường. Deregistration Delay (Độ Trễ Hủy Đăng Ký) đảm bảo connections hiện tại không bị ngắt đột ngột khi xóa target.

## 🏗️ Target Group — Khái Niệm Cốt Lõi

### Cấu Trúc Target Group

```
Target Group
├── Protocol & Port (giao thức & cổng giao tiếp với targets)
│   ├── ALB Target Groups: HTTP, HTTPS, HTTP/2, gRPC
│   └── NLB Target Groups: TCP, UDP, TLS, TCP_UDP
│
├── Target Type (loại mục tiêu)
│   ├── instance: EC2 Instance ID
│   ├── ip: IPv4 address
│   ├── lambda: Lambda Function ARN
│   └── alb: Application Load Balancer ARN (chỉ cho NLB)
│
├── Health Check Configuration (cấu hình kiểm tra sức khỏe)
│   ├── Protocol, Path, Port
│   ├── Healthy/Unhealthy Threshold
│   ├── Interval, Timeout
│   └── Success Codes
│
├── Load Balancing Algorithm (thuật toán cân bằng tải)
│   ├── Round Robin (luân phiên vòng tròn) — mặc định ALB
│   ├── Least Outstanding Requests (ít request đang xử lý nhất) — ALB
│   └── Flow Hash (băm luồng) — NLB
│
├── Stickiness (Tính Dính — sticky sessions)
│   ├── Duration-based: Cookie với TTL
│   └── Application-based: Cookie do app tạo
│
└── Slow Start Duration (thời gian khởi động chậm) — chỉ ALB
```

### Target Types — So Sánh Chi Tiết

```
1. instance (EC2 Instance)
   ┌────────────────────────────────────────────────────┐
   │ Target: Instance ID (i-0123456789abcdef)           │
   │ Traffic port: Port cấu hình trên Target Group      │
   │ Source IP: ALB IP (cho ALB) / Client IP (cho NLB)  │
   │ Hạn chế: Chỉ EC2 trong cùng VPC với Load Balancer │
   └────────────────────────────────────────────────────┘

2. ip (Địa Chỉ IP)
   ┌────────────────────────────────────────────────────┐
   │ Target: IPv4 address (10.0.1.100, 192.168.1.50)    │
   │ Có thể target:                                     │
   │   - ENI private IP trong VPC                       │
   │   - On-premises servers qua VPN/Direct Connect     │
   │   - Containers trong ECS/EKS (Fargate)             │
   │   - IP trong peered VPC                            │
   │ Use case: Containers cần port động (dynamic port)  │
   └────────────────────────────────────────────────────┘

3. lambda (AWS Lambda)
   ┌────────────────────────────────────────────────────┐
   │ Chỉ ALB hỗ trợ, không phải NLB                    │
   │ Target: Lambda Function ARN                         │
   │ ALB invoke Lambda với event object JSON            │
   │ Lambda trả về response object JSON                 │
   │ Multi-value headers: bật để hỗ trợ duplicate headers│
   │ Health check: Không cần (Lambda luôn "healthy")    │
   └────────────────────────────────────────────────────┘

4. alb (Application Load Balancer)
   ┌────────────────────────────────────────────────────┐
   │ Chỉ NLB hỗ trợ                                    │
   │ Pattern: NLB (static IP) → ALB (Layer 7 routing)   │
   │ Use case: Cần cả Static IP VÀ content-based routing│
   │ Ví dụ: Firewall cần whitelist IP + path routing    │
   └────────────────────────────────────────────────────┘
```

---

## 💚 Health Checks — Kiểm Tra Sức Khỏe

### Cơ Chế Hoạt Động

```
Load Balancer Node
        │
        │ Mỗi [interval] giây gửi request đến target
        ▼
Target Server  ──► HTTP 200? → Healthy check ✅
               ──► Timeout?  → Unhealthy check ❌
               ──► HTTP 5xx? → Unhealthy check ❌

Sau [healthy-threshold] lần healthy liên tiếp → Target = HEALTHY
Sau [unhealthy-threshold] lần unhealthy liên tiếp → Target = UNHEALTHY

UNHEALTHY target: Load Balancer NGỪNG gửi traffic mới
HEALTHY target: Load Balancer gửi traffic bình thường
```

### Health Check Parameters (Tham Số Kiểm Tra Sức Khỏe)

```
┌─────────────────────────────────────────────────────────────┐
│ Parameter          │ Default   │ Range       │ Mô Tả        │
├─────────────────────────────────────────────────────────────┤
│ Protocol           │ HTTP      │ HTTP/HTTPS/ │ Giao thức HC │
│                    │           │ TCP/HTTPS   │              │
├─────────────────────────────────────────────────────────────┤
│ Path               │ /         │ Bất kỳ path │ URL path HC  │
│                    │           │             │ (HTTP/HTTPS) │
├─────────────────────────────────────────────────────────────┤
│ Port               │ traffic-  │ 1-65535     │ Port dùng HC │
│                    │ port      │             │              │
├─────────────────────────────────────────────────────────────┤
│ Interval           │ 30 giây   │ 5-300 giây  │ Tần suất HC  │
├─────────────────────────────────────────────────────────────┤
│ Timeout            │ 5 giây    │ 2-120 giây  │ Chờ response │
├─────────────────────────────────────────────────────────────┤
│ Healthy Threshold  │ 5         │ 2-10        │ Lần healthy  │
│                    │           │             │ liên tiếp    │
├─────────────────────────────────────────────────────────────┤
│ Unhealthy Threshold│ 2         │ 2-10        │ Lần unhealthy│
│                    │           │             │ liên tiếp    │
├─────────────────────────────────────────────────────────────┤
│ Success Codes      │ 200       │ 200-499     │ HTTP codes   │
│                    │           │             │ coi là healthy│
└─────────────────────────────────────────────────────────────┘
```

### Tính Toán Thời Gian Phát Hiện Sự Cố

```
Thời gian phát hiện unhealthy:
= Unhealthy Threshold × Interval
= 2 × 30 = 60 giây (mặc định)

Tối ưu cho phát hiện nhanh:
= 2 × 10 = 20 giây (interval = 10s, threshold = 2)

Thời gian phục hồi (healthy lại):
= Healthy Threshold × Interval
= 5 × 30 = 150 giây (mặc định)
= 3 × 10 = 30 giây (tối ưu)

Trade-off:
- Interval ngắn hơn → Phát hiện nhanh hơn + tải nhỏ hơn cho targets
- Threshold thấp hơn → Nhạy cảm hơn với sự cố tạm thời (false positives)
```

### Health Check Endpoint Best Practices

```
Thiết kế endpoint /health chuyên biệt:

❌ Sai: Dùng path / (trang chủ)
Vấn đề:
- Trang chủ có thể load nhiều tài nguyên
- Ảnh hưởng đến performance khi nhiều HC đồng thời
- Không phân biệt được app healthy hay chỉ có HTTP server chạy

✅ Đúng: Tạo endpoint /health hoặc /ping riêng
Code ví dụ (Node.js Express):

app.get('/health', async (req, res) => {
  try {
    // Kiểm tra database connection
    await db.query('SELECT 1');
    // Kiểm tra cache connection
    await redis.ping();

    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});

Mức độ kiểm tra:
Level 1 (Shallow): Chỉ HTTP server sống → return 200 ngay
Level 2 (Deep): Kiểm tra DB, cache, external dependencies
Level 3 (Very Deep): Kiểm tra business logic, queue depth

Khuyến nghị: Level 2 cho Health Check của ALB/NLB
Level 3 chỉ cho application-level monitoring riêng
```

### Health Check Status Transitions (Chuyển Đổi Trạng Thái)

```
                    Unhealthy Check × 2
         ┌─────────────────────────────────────────┐
         │                                          ▼
     INITIAL                                   UNHEALTHY
      (mới thêm)                              (không nhận traffic)
         │                                          │
         │ Healthy Check × 5                        │ Healthy Check × 5
         ▼                                          │
      HEALTHY  ◄──────────────────────────────────┘
   (nhận traffic)
         │
         │ Unhealthy Check × 2
         ▼
     UNHEALTHY

Trạng thái đặc biệt:
- DRAINING: Target đang bị deregister, đợi connections kết thúc
- UNUSED: Target Group không được gắn vào listener nào
```

---

## ⏳ Deregistration Delay — Độ Trễ Hủy Đăng Ký

### Vấn Đề Khi Không Có Deregistration Delay

```
Kịch bản xấu (KHÔNG có Deregistration Delay):

T=0s: Client A gửi request → Target Instance-1
T=5s: Auto Scaling quyết định terminate Instance-1
T=5s: Instance-1 bị xóa khỏi Target Group NGAY LẬP TỨC
T=5s: Request của Client A đang xử lý bị ngắt → 500 Error!
T=5s: Client B đang download file lớn → Bị reset!
```

### Cơ Chế Hoạt Động

```
Kịch bản tốt (CÓ Deregistration Delay = 300s):

T=0s: Client A gửi request → Target Instance-1
T=5s: Auto Scaling quyết định terminate Instance-1
T=5s: Instance-1 vào trạng thái "DRAINING"
      - Không nhận kết nối MỚI
      - VẪN xử lý kết nối ĐANG CÓ (in-flight requests)
T=35s: Request của Client A hoàn thành ✅
T=300s: Hết Deregistration Delay → Instance-1 bị deregister hoàn toàn
T=301s: Auto Scaling terminate Instance-1
```

### Cấu Hình Deregistration Delay

```
Mặc định: 300 giây (5 phút)
Phạm vi: 0 – 3600 giây (0 – 60 phút)

Cấu hình trong Target Group Attributes:
deregistration_delay.timeout_seconds = <value>

Khi nào điều chỉnh:

Giảm xuống (30–60 giây):
✅ API servers: Requests ngắn, < 1 giây
✅ Stateless microservices
✅ Cần deploy nhanh (blue/green deployment)

Giữ mặc định (300 giây):
✅ Web applications với sessions
✅ Upload/download file vừa

Tăng lên (600–3600 giây):
✅ File upload/download lớn
✅ Long-polling requests (30+ giây)
✅ WebSocket connections cần persistent
✅ Batch processing jobs (xử lý hàng loạt)
```

---

## 🔄 Load Balancing Algorithms (Thuật Toán Cân Bằng Tải)

### Round Robin — ALB Default (Mặc Định)

```
Request 1 → Target A
Request 2 → Target B
Request 3 → Target C
Request 4 → Target A (bắt đầu lại)
Request 5 → Target B
...

Ưu điểm: Đơn giản, phân phối đều
Nhược điểm: Không xét đến tải thực của mỗi target
Khi dùng: Requests có thời gian xử lý tương đương nhau
```

### Least Outstanding Requests — ALB (Ít Request Đang Xử Lý Nhất)

```
Tại mỗi request mới:
- Target A: 10 requests đang xử lý
- Target B: 2 requests đang xử lý ← CHỌN CÁI NÀY
- Target C: 7 requests đang xử lý

Ưu điểm: Phân phối thông minh hơn, tránh overload target chậm
Nhược điểm: Phức tạp hơn, cần tracking trạng thái
Khi dùng: Requests có thời gian xử lý không đồng đều (mixed workloads)

Bật: Target Group → Attributes → Load balancing algorithm
→ Least outstanding requests
```

### Flow Hash — NLB (Băm Luồng)

```
NLB dùng Flow Hash để đảm bảo một TCP session
luôn đến cùng một target:

Hash input:
- Protocol (TCP/UDP)
- Source IP + Source Port
- Destination IP + Destination Port

→ Cùng TCP session = cùng {source IP, source port} → cùng target
→ Đảm bảo TCP connection integrity (tính toàn vẹn kết nối TCP)

Không thể thay đổi thuật toán cho NLB
```

---

## 🍪 Sticky Sessions — Phiên Dính

### Nguyên Lý Sticky Sessions

```
Vấn đề: Shopping cart lưu trong session memory của server

Không có Sticky Sessions:
Request 1 (thêm item) → Server A (session: {items: [shoe]})
Request 2 (xem cart)  → Server B (session: {} ← TRỐNG!)

Có Sticky Sessions:
Request 1 → Server A (cookie: AWSALB=hash_of_server_A)
Request 2 → Server A (cookie xác định → luôn về Server A)

Cách hoạt động:
1. ALB gán client vào target cụ thể
2. ALB set cookie vào HTTP response
3. Client gửi cookie trong mọi request tiếp theo
4. ALB đọc cookie → forward đến cùng target
```

### Hai Loại Sticky Sessions

```
1. Duration-based Stickiness (Dính Theo Thời Gian)
   ├── Cookie name: AWSALB (do ALB tạo)
   ├── Cookie duration: 1 giây – 7 ngày
   ├── Khi target unhealthy: Session "bị mất", route đến target mới
   └── Use case: Ứng dụng đơn giản, không cần control cookie

2. Application-based Stickiness (Dính Theo Ứng Dụng)
   ├── Cookie name: Tùy chỉnh (do ứng dụng set)
   ├── ALB tạo thêm cookie: AWSALBAPP
   ├── Ứng dụng control cookie content và expiry
   └── Use case: Ứng dụng cần custom session logic

So sánh với NLB:
NLB dùng Flow Hash → Tự nhiên sticky theo TCP session
Không cần cấu hình sticky sessions cho NLB
```

### Nhược Điểm Của Sticky Sessions

```
⚠️ Sticky Sessions gây ra các vấn đề:

1. Uneven Distribution (Phân Phối Không Đồng Đều):
   - Nếu 1 client gửi nhiều requests → 1 target bị overloaded
   - Các target khác rảnh nhưng không được dùng

2. Stateful Server Anti-pattern (Chống Mẫu Server Trạng Thái):
   - Microservices nên stateless (phi trạng thái)
   - Session state nên lưu trên Redis/DynamoDB thay vì server memory

3. Scaling Khó Khăn:
   - Khi terminate instance → sessions bị mất
   - Rolling deploy gây mất sessions

Best Practice:
✅ Dùng Redis/ElastiCache để lưu sessions tập trung
✅ JWT (JSON Web Tokens) cho stateless authentication
✅ Tránh Sticky Sessions nếu có thể
✅ Chỉ dùng khi ứng dụng legacy không thể refactor
```

---

## 🏋️ Lab Thực Hành

### Tạo Target Group Và Cấu Hình Health Check

```bash
# Tạo Target Group với Health Check tùy chỉnh
aws elbv2 create-target-group \
  --name api-servers-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-12345678 \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-port 8080 \
  --health-check-interval-seconds 10 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200

# Cấu hình Deregistration Delay = 60 giây (cho API servers)
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes \
    Key=deregistration_delay.timeout_seconds,Value=60 \
    Key=load_balancing.algorithm.type,Value=least_outstanding_requests \
    Key=slow_start.duration_seconds,Value=60

# Bật Sticky Sessions
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=86400

# Xem trạng thái health của targets
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...
```

### Đăng Ký Và Hủy Đăng Ký Targets

```bash
# Đăng ký EC2 instances
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets \
    Id=i-0123456789abcdef1,Port=8080 \
    Id=i-0123456789abcdef2,Port=8080

# Đăng ký IP targets (ví dụ: containers)
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets \
    Id=10.0.1.100,Port=8080 \
    Id=10.0.2.200,Port=8080

# Hủy đăng ký target (bắt đầu draining)
aws elbv2 deregister-targets \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --targets Id=i-0123456789abcdef1

# Theo dõi trạng thái draining
watch -n 5 "aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,State:TargetHealth.State}' \
  --output table"
```

---

## 📊 Monitoring Target Groups

### CloudWatch Alarms Quan Trọng

```bash
# Alert khi có target unhealthy
aws cloudwatch put-metric-alarm \
  --alarm-name "TG-UnhealthyHosts" \
  --alarm-description "Có target unhealthy trong Target Group" \
  --metric-name UnHealthyHostCount \
  --namespace AWS/ApplicationELB \
  --dimensions \
    Name=TargetGroup,Value=targetgroup/api-servers-tg/1234567890abcdef \
    Name=LoadBalancer,Value=app/my-web-alb/1234567890abcdef \
  --statistic Average \
  --period 60 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:...
```

---

## 💡 Best Practices

```
✅ Health Check:
1. Tạo /health endpoint chuyên biệt, không dùng /
2. Health check nên kiểm tra cả database và cache connection
3. Dùng interval = 10s, threshold = 2 để phát hiện nhanh
4. Đặt success codes = 200-299 (không dùng 4xx làm "healthy")

✅ Target Type:
1. Dùng IP target cho ECS Fargate (container cần dynamic port)
2. Dùng Instance target cho EC2 đơn giản
3. Dùng Lambda target cho serverless workloads

✅ Deregistration Delay:
1. API servers: 30–60 giây
2. Web apps với sessions: 300 giây (mặc định)
3. File upload services: 600–3600 giây

✅ Sticky Sessions:
1. Tránh dùng sticky sessions khi có thể
2. Nếu bắt buộc: Dùng Application-based stickiness
3. Lưu sessions vào ElastiCache để tránh phụ thuộc vào server

✅ Algorithm:
1. Dùng Least Outstanding Requests khi workload không đồng đều
2. Round Robin cho stateless APIs có workload đồng đều
```

---

## 🎓 Câu Hỏi Phỏng Vấn

**Q: Deregistration Delay là gì và tại sao cần thiết?**

> Deregistration Delay (hay Connection Draining) là khoảng thời gian load balancer chờ đợi sau khi target bị deregister, trong thời gian đó target vẫn hoàn thành các requests đang xử lý (in-flight requests) nhưng không nhận requests mới. Mặc định 300 giây. Cần thiết vì không có nó, khi Auto Scaling terminate một instance, tất cả requests đang xử lý sẽ bị ngắt đột ngột gây 500 errors cho users. Giá trị nên điều chỉnh theo thời gian xử lý request tối đa của ứng dụng.

**Q: Sticky sessions là gì? Khi nào nên tránh?**

> Sticky sessions đảm bảo requests từ cùng một client luôn đến cùng một target, thông qua cookie (AWSALB) được ALB set. Nên tránh vì: (1) Gây phân phối không đồng đều — một target có thể bị overloaded; (2) Làm hỏng horizontal scaling — nếu target bị terminate, session bị mất; (3) Microservices nên stateless. Thay vào đó, lưu session state vào Redis/ElastiCache để mọi target đều có thể phục vụ mọi request.

**Q: Least Outstanding Requests khác Round Robin như thế nào?**

> Round Robin gửi request theo vòng tuần tự: A, B, C, A, B, C — đơn giản nhưng không xét tải thực tế. Nếu request đến A mất 100ms nhưng request đến B mất 5 giây, B sẽ bị overloaded dù nhận cùng số requests. Least Outstanding Requests chọn target có ít requests đang xử lý nhất — thông minh hơn với mixed workloads. Dùng LOR khi requests có thời gian xử lý không đồng đều (ví dụ: một số queries nhanh, một số chạy lâu).

---

**Tiếp Theo:** [4-ssl-tls.md](4-ssl-tls.md) — SSL/TLS Termination, ACM, HTTPS Best Practices

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn Thành
