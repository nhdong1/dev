# 03 — Load Balancing — Cân Bằng Tải AWS

> ELB (Elastic Load Balancing — Cân Bằng Tải Đàn Hồi) là dịch vụ tự động phân phối lưu lượng đến vào nhiều mục tiêu: EC2 instances, containers, IP addresses, Lambda functions — giúp đảm bảo HA (High Availability — Tính Sẵn Sàng Cao) và khả năng mở rộng cho ứng dụng.

## 📚 Nội Dung Section Này

| File | Chủ Đề | Mức Độ |
| --- | --- | --- |
| [1-alb.md](1-alb.md) | ALB — Application Load Balancer, Layer 7, listener rules | ⭐⭐ |
| [2-nlb.md](2-nlb.md) | NLB — Network Load Balancer, Layer 4, ultra-low latency | ⭐⭐ |
| [3-target-groups.md](3-target-groups.md) | Target Groups, Health Checks, Deregistration Delay | ⭐⭐ |
| [4-ssl-tls.md](4-ssl-tls.md) | SSL/TLS Termination, ACM, HTTPS Best Practices | ⭐⭐⭐ |
| [5-advanced-patterns.md](5-advanced-patterns.md) | Cross-zone LB, Sticky Sessions, Weighted Routing | ⭐⭐⭐ |

---

## 🎯 Tại Sao Cân Bằng Tải Quan Trọng?

### Vấn Đề Khi Không Có Load Balancer

```
Client ──► Single Server ──► Ứng Dụng

Vấn đề:
- SPOF (Single Point of Failure — Điểm Hỏng Duy Nhất): server chết → toàn bộ hệ thống sập
- Bottleneck (Điểm Nghẽn): một server không thể xử lý hàng nghìn request đồng thời
- Không có khả năng scale-out (mở rộng theo chiều ngang)
- Maintenance (Bảo Trì) gây downtime (thời gian ngừng hoạt động)
```

### Giải Pháp Với Load Balancer

```
                    ┌─► EC2 Instance A (Healthy)
Client ──► ALB ─────├─► EC2 Instance B (Healthy)
                    └─► EC2 Instance C (Healthy)

Lợi ích:
- HA (High Availability): instance chết → traffic tự động chuyển sang instance khác
- Scalability (Khả Năng Mở Rộng): thêm/xóa instance không ảnh hưởng client
- Health Checks: tự động phát hiện instance không lành mạnh
- SSL Termination (Kết Thúc SSL): xử lý HTTPS tập trung tại load balancer
```

---

## 🏗️ Các Loại Load Balancer Của AWS

### ELB (Elastic Load Balancing) — Ba Thế Hệ

```
Thế hệ 1: CLB (Classic Load Balancer — Cân Bằng Tải Cổ Điển)
├── Ra mắt: 2009
├── Layer: 4 & 7 (cơ bản)
├── Trạng thái: Legacy (Kế Thừa) — KHÔNG khuyến nghị dùng mới
└── Nên migrate sang: ALB hoặc NLB

Thế hệ 2: ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng)
├── Ra mắt: 2016
├── Layer: 7 (HTTP/HTTPS/WebSocket/gRPC)
├── Trạng thái: Hiện đại, được khuyến nghị cho web apps
└── Chi tiết: 1-alb.md

Thế hệ 3: NLB (Network Load Balancer — Cân Bằng Tải Mạng)
├── Ra mắt: 2017
├── Layer: 4 (TCP/UDP/TLS)
├── Trạng thái: Hiện đại, dành cho performance-critical workloads
└── Chi tiết: 2-nlb.md

Thế hệ 4: GWLB (Gateway Load Balancer — Cân Bằng Tải Cổng)
├── Ra mắt: 2020
├── Layer: 3 (IP packets)
├── Dùng cho: Network appliances (firewalls, IDS/IPS)
└── Phạm vi: Ngoài scope của section này
```

---

## 🔄 So Sánh Nhanh ALB vs NLB vs CLB

| Tiêu Chí | ALB | NLB | CLB |
| --- | --- | --- | --- |
| **OSI Layer** | 7 (Application) | 4 (Transport) | 4 & 7 |
| **Giao Thức** | HTTP, HTTPS, WebSocket, gRPC | TCP, UDP, TLS | HTTP, HTTPS, TCP |
| **Routing** | Content-based (path, header, host) | IP:Port | Round-robin cơ bản |
| **Latency** | ~400ms (milliseconds) | ~100μs (microseconds) | Trung bình |
| **Static IP** | Không (dùng DNS) | Có (Elastic IP per AZ) | Không |
| **Preserve Client IP** | X-Forwarded-For header | Có (mặc định) | Tùy cấu hình |
| **WebSocket** | Có | Có | Không |
| **gRPC** | Có | Không | Không |
| **SSL Termination** | Có | Có | Có |
| **Lambda Target** | Có | Không | Không |
| **Giá** | Cao hơn | Tương đương ALB | Thấp nhất |
| **Use Case** | Web apps, microservices, API | Gaming, IoT, TCP apps | Legacy workloads |

---

## 🏛️ Kiến Trúc Tổng Quan

### Vị Trí Của Load Balancer Trong Hệ Thống AWS

```
Internet
    │
    ▼
Route 53 (DNS phân giải tên miền → IP load balancer)
    │
    ▼
ALB / NLB  ◄── Security Group (chỉ mở port 80/443)
    │
    ├── AZ-A: Target Group → EC2, ECS, Lambda
    ├── AZ-B: Target Group → EC2, ECS, Lambda
    └── AZ-C: Target Group → EC2, ECS, Lambda
                │
                ▼
        RDS (cơ sở dữ liệu, private subnet)
```

### Multi-tier Architecture (Kiến Trúc Đa Tầng)

```
Internet
    │
    ▼
External ALB (Public Subnet)
  ├── /* → Frontend EC2 / ECS (React/Next.js app)
  └── /api/* → Internal ALB
                    │
                    ▼
              Internal ALB (Private Subnet)
                ├── /users → User Service
                ├── /orders → Order Service
                └── /payments → Payment Service
```

---

## 📋 Thành Phần Cốt Lõi

### Listener (Bộ Lắng Nghe)

```
Listener = Cổng tiếp nhận request từ client

Ví dụ:
- Listener HTTP :80 → Redirect to HTTPS
- Listener HTTPS :443 → Forward to target group
- Listener TCP :3306 → Forward to RDS (NLB)
```

### Target Group (Nhóm Mục Tiêu)

```
Target Group = Tập hợp các mục tiêu nhận lưu lượng

Loại mục tiêu:
- Instance: EC2 instances
- IP: Địa chỉ IP (on-premises hoặc trong VPC)
- Lambda: AWS Lambda functions
- ALB: Application Load Balancer (chỉ từ NLB)

Chi tiết: 3-target-groups.md
```

### Health Check (Kiểm Tra Sức Khỏe)

```
Health Check = Cơ chế phát hiện target không lành mạnh

Quy trình:
1. Load balancer gửi request kiểm tra đến từng target
2. Nếu target trả về HTTP 200 → Healthy (Lành Mạnh)
3. Nếu target không trả lời hoặc lỗi → Unhealthy (Không Lành Mạnh)
4. Traffic không gửi đến target Unhealthy cho đến khi phục hồi

Chi tiết: 3-target-groups.md
```

---

## 🎯 Khi Nào Dùng Loại Nào?

### Chọn ALB Khi:

```
✅ Ứng dụng web HTTP/HTTPS
✅ Cần routing dựa trên URL path (/api, /static, /admin)
✅ Cần routing dựa trên hostname (api.example.com vs app.example.com)
✅ Microservices architecture
✅ WebSocket connections (kết nối WebSocket)
✅ gRPC services
✅ Lambda functions làm backend
✅ Container (ECS/EKS) workloads
✅ Cần redirect HTTP → HTTPS tự động
```

### Chọn NLB Khi:

```
✅ Ultra-low latency (độ trễ cực thấp) — microseconds
✅ TCP/UDP applications (game servers, VoIP, streaming)
✅ Cần Static IP hoặc Elastic IP
✅ Network performance cực cao (millions of RPS — requests per second)
✅ TLS passthrough (không giải mã tại load balancer)
✅ On-premises servers qua VPN/Direct Connect
✅ AWS PrivateLink endpoint services
✅ IoT (Internet of Things) devices
```

### Giữ CLB Khi:

```
⚠️ Chỉ khi: Hệ thống legacy đang dùng CLB và chưa có kế hoạch migrate
❌ Không khuyến nghị cho bất kỳ workload mới nào
```

---

## 🔐 Bảo Mật Load Balancer

### Security Group Best Practices

```
ALB Security Group (Inbound):
- Port 80 (HTTP) from 0.0.0.0/0
- Port 443 (HTTPS) from 0.0.0.0/0

Target EC2 Security Group (Inbound):
- Port 8080 (app port) from ALB Security Group ID ONLY
  → KHÔNG mở trực tiếp từ internet

NLB Security Group:
- NLB không có Security Group
- Cần cấu hình Network ACL hoặc Security Group trên targets
```

### WAF Integration (Tích Hợp Tường Lửa Ứng Dụng Web)

```
WAF chỉ hỗ trợ: ALB, CloudFront, API Gateway
NLB: Không hỗ trợ WAF trực tiếp

Để bảo vệ NLB: Dùng AWS Network Firewall hoặc third-party appliances
```

---

## 📊 Monitoring & Observability (Giám Sát & Quan Sát)

### CloudWatch Metrics Quan Trọng

```
ALB Metrics:
- RequestCount: Tổng số request
- TargetResponseTime: Thời gian phản hồi của target
- HTTPCode_ELB_5XX_Count: Lỗi 5xx từ ALB (server-side errors)
- HTTPCode_Target_5XX_Count: Lỗi 5xx từ targets
- HealthyHostCount: Số target đang healthy
- UnHealthyHostCount: Số target đang unhealthy

NLB Metrics:
- ActiveFlowCount: Số kết nối TCP/UDP đang hoạt động
- NewFlowCount: Số kết nối mới mỗi giây
- ProcessedBytes: Dữ liệu đã xử lý
- HealthyHostCount / UnHealthyHostCount
```

### Access Logs (Nhật Ký Truy Cập)

```
Bật Access Logs để phân tích:
- Client IP, request time, target IP, response code
- Lưu vào S3 bucket
- Phân tích bằng Amazon Athena hoặc CloudWatch Logs Insights

ALB Access Log fields quan trọng:
- type, time, elb, client:port, target:port
- request_processing_time, target_processing_time
- response_processing_time, elb_status_code, target_status_code
- received_bytes, sent_bytes, request
```

---

## 💡 Câu Hỏi Phỏng Vấn Thường Gặp

```
1. ALB vs NLB — sự khác biệt chính và khi nào dùng loại nào?
2. Sticky sessions (phiên dính) là gì? Ưu/nhược điểm?
3. Cross-zone load balancing hoạt động như thế nào?
4. Cách ALB routing dựa trên URL path?
5. Health check threshold — healthy threshold và unhealthy threshold?
6. Connection draining / deregistration delay là gì?
7. SSL termination tại ALB vs end-to-end encryption?
8. Slow start mode trong ALB là gì?
```

---

## 🚀 Học Theo Thứ Tự

```
1. Đọc 1-alb.md để nắm Application Load Balancer
2. Đọc 2-nlb.md để hiểu Network Load Balancer
3. Đọc 3-target-groups.md để hiểu Target Groups & Health Checks
4. Đọc 4-ssl-tls.md để cấu hình HTTPS đúng cách
5. Đọc 5-advanced-patterns.md để nắm các pattern nâng cao
```

---

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn Thành
