# Advanced Load Balancing Patterns — Mẫu Cân Bằng Tải Nâng Cao

> Section này đề cập các pattern nâng cao trong Elastic Load Balancing: Cross-zone Load Balancing (Cân Bằng Tải Liên AZ), Sticky Sessions (Phiên Dính), Weighted Target Group Routing (Định Tuyến Có Trọng Số), và các kiến trúc phức tạp hơn như Dual Load Balancer, PrivateLink, và Global Accelerator integration.

## 🌐 Cross-Zone Load Balancing — Cân Bằng Tải Liên AZ

### Vấn Đề Không Có Cross-Zone Load Balancing

```
Kịch bản: 2 AZs, 4 targets không đồng đều

AZ-A (50% traffic từ DNS round-robin):
├── Target A1
└── Target A2

AZ-B (50% traffic từ DNS round-robin):
├── Target B1
├── Target B2
└── Target B3

Kết quả:
- A1, A2 nhận: 50% ÷ 2 = 25% traffic mỗi target
- B1, B2, B3 nhận: 50% ÷ 3 = 16.7% traffic mỗi target

→ A1 và A2 bị overloaded so với B1, B2, B3
```

### Với Cross-Zone Load Balancing BẬT

```
AZ-A: Target A1, A2
AZ-B: Target B1, B2, B3

Tổng: 5 targets → Mỗi target nhận: 100% ÷ 5 = 20% traffic

Traffic có thể đi:
AZ-A node ──► Bất kỳ target trong AZ-A hoặc AZ-B
AZ-B node ──► Bất kỳ target trong AZ-A hoặc AZ-B

Kết quả: Phân phối đồng đều!
```

### Hành Vi Cross-Zone Theo Từng Loại Load Balancer

```
┌────────────────────────────────────────────────────────────┐
│ Load Balancer │ Cross-Zone Default │ Có thể thay đổi │ Phí │
├────────────────────────────────────────────────────────────┤
│ ALB           │ BẬT (ON)          │ Có              │ Miễn phí│
├────────────────────────────────────────────────────────────┤
│ NLB           │ TẮT (OFF)         │ Có              │ $0.01/GB │
├────────────────────────────────────────────────────────────┤
│ GWLB          │ TẮT (OFF)         │ Có              │ $0.01/GB │
└────────────────────────────────────────────────────────────┘

Chi phí cross-AZ data transfer cho NLB/GWLB:
- Data transfer trong cùng region giữa AZs: ~$0.01/GB
- Với traffic lớn (TB/tháng) → Chi phí đáng kể

Quyết định:
BẬT cross-zone NLB khi: Targets không đồng đều giữa AZs
TẮT cross-zone NLB khi: Targets đồng đều + muốn tiết kiệm chi phí
```

### Cấu Hình Cross-Zone

```bash
# Bật cross-zone load balancing cho NLB
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --attributes \
    Key=load_balancing.cross_zone.enabled,Value=true

# Hoặc tắt cho ALB (tiết kiệm latency khi targets local)
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --attributes \
    Key=load_balancing.cross_zone.enabled,Value=false
```

---

## ⚖️ Weighted Target Group Routing — Định Tuyến Có Trọng Số

### Cơ Chế Weighted Routing

```
ALB Listener Rule:
Action: Forward to multiple Target Groups with weights

┌─────────────────────────────────────────┐
│ Target Group: Production-v1  Weight: 80 │
│ Target Group: Production-v2  Weight: 20 │
└─────────────────────────────────────────┘

Kết quả: 80% traffic → v1, 20% traffic → v2

Tổng weight không cần bằng 100:
- weight 80 & 20 = 80% & 20%
- weight 4 & 1 = 80% & 20% (tương đương)
- weight 1 & 0 = 100% & 0%
```

### Blue/Green Deployment (Triển Khai Xanh/Xanh Lá) Với Weighted Routing

```
Chiến lược triển khai zero-downtime (không gián đoạn):

Phase 0: 100% → BLUE (v1.0 đang chạy)
   Blue: 100, Green: 0

Phase 1: Deploy GREEN (v2.0) trong Target Group riêng
   Green targets được deploy, nhưng chưa nhận traffic

Phase 2: Canary release 5% (Phát Hành Thử Nghiệm)
   Blue: 95, Green: 5
   Monitor: Error rate, latency trong 30 phút

Phase 3: Tăng dần nếu ổn
   Blue: 75, Green: 25
   Blue: 50, Green: 50
   Blue: 25, Green: 75

Phase 4: Full cutover (Chuyển Đổi Hoàn Toàn)
   Blue: 0, Green: 100

Phase 5: Rollback nếu có vấn đề (quay về Blue)
   Blue: 100, Green: 0 (trong <2 phút!)

Lợi ích so với DNS-based routing:
- Thay đổi weight trong giây, không cần đợi DNS TTL expire
- Granular control (kiểm soát chi tiết) đến từng %
```

### A/B Testing Với Weighted Routing

```
Use case: Test UI mới vs UI cũ

Target Group A (UI cũ): weight = 50
Target Group B (UI mới): weight = 50

Kết hợp với Analytics:
- Tag requests theo header X-AB-Test
- Track conversion rate, bounce rate, time-on-page
- Sau 1 tuần: Chọn winner, set weight = 100

Header-based A/B (alternative — thay thế):
Rule 1: Header "X-Variant=B" → Target Group B
Default: → Target Group A
→ Dùng khi muốn user stick với variant (không random mỗi request)
```

---

## 🏗️ Dual Load Balancer Architecture — Kiến Trúc Hai Load Balancer

### External + Internal ALB

```
Internet
    │
    ▼
External ALB (Internet-facing — Hướng Internet)
├── Security Group: Allow 80, 443 from 0.0.0.0/0
├── Subnets: Public subnets (Multi-AZ)
└── Listeners:
    ├── HTTP :80 → Redirect HTTPS
    └── HTTPS :443:
        ├── /api/* → Internal ALB (Target Group: ip type)
        └── /*     → Frontend Target Group

Internal ALB (Internal-facing — Hướng Nội Bộ)
├── Security Group: Allow 80 from External ALB SG only
├── Subnets: Private subnets (Multi-AZ)
└── Listeners:
    └── HTTP :80:
        ├── /users    → User Service TG
        ├── /orders   → Order Service TG
        ├── /products → Product Service TG
        └── /payment  → Payment Service TG

Lợi ích:
✅ Microservices ẩn hoàn toàn khỏi internet
✅ Mỗi service có Health Check riêng
✅ Scale từng service độc lập
✅ Deploy service riêng lẻ không ảnh hưởng services khác
```

### NLB → ALB Pattern

```
Client cần: Static IP + Layer 7 routing

NLB (Static Elastic IP)
    │ TCP :443
    ▼
ALB (Target loại: alb)
    │ HTTPS :443
    ├── /api/* → API Target Group
    └── /*     → Web Target Group

Khi cần pattern này:
- On-premises firewall whitelist IP cụ thể → NLB (Elastic IP)
- Nhưng cũng cần path-based routing → ALB
- Compliance requirement về static IPs
```

---

## 🔄 Connection Persistence Patterns

### Long-lived Connections (Kết Nối Tồn Tại Lâu)

```
Thách thức: WebSocket, Server-Sent Events, gRPC streaming

ALB WebSocket:
Client ──WS Upgrade──► ALB ──WS──► Target
                       (persistent connection, không route lại)

Cấu hình:
- Idle timeout: Tăng lên 3600 giây (1 giờ)
- Stickiness: BẬT (Duration-based)
  → Đảm bảo WS connection đến cùng target

⚠️ WS connections không benefit từ cross-zone load balancing
   Mỗi connection gắn với 1 target trong suốt lifetime
```

### Connection Multiplexing (Ghép Kênh Kết Nối)

```
HTTP/2 Multiplexing từ Client đến ALB:
- 1 TCP connection với nhiều HTTP/2 streams
- ALB xử lý nhiều requests đồng thời trên 1 connection

ALB đến Target (HTTP/1.1 mặc định):
- Mỗi request là 1 kết nối riêng hoặc keepalive
- Connection pooling giữa ALB và targets

ALB đến Target (HTTP/2):
- Bật HTTP/2: Target Group → Protocol version → HTTP/2
- Tăng efficiency cho microservices communication
- Cần target hỗ trợ HTTP/2 (Nginx, H2O, gRPC servers)
```

---

## 🚀 Global Load Balancing Patterns

### Multi-Region Architecture (Kiến Trúc Đa Vùng)

```
Approach 1: Route 53 Latency-based Routing + Regional ALBs

Route 53
├── US-East-1: ALB → Targets US-East-1
├── EU-West-1: ALB → Targets EU-West-1
└── AP-Southeast-1: ALB → Targets AP-Southeast-1

Client DNS query → Route 53 trả về IP của region có latency thấp nhất
Kết hợp với Health Checks → Failover tự động nếu region down

Approach 2: AWS Global Accelerator + Regional ALBs

Global Accelerator (2 Anycast IPs toàn cầu)
├── Endpoint US-East-1: ALB (weight 100%)
├── Endpoint EU-West-1: ALB (weight 100%)
└── Endpoint AP-Southeast-1: ALB (weight 100%)

Traffic flow:
1. Client connect đến Anycast IP gần nhất (edge location)
2. Global Accelerator route trên AWS backbone (không qua internet)
3. Traffic đến regional ALB nhanh hơn 60% so với internet
4. Failover tự động trong 30 giây khi endpoint unhealthy

So sánh:
Route 53: Routing tại DNS level, free tier có sẵn
Global Accelerator: Routing tại TCP level, mất phí nhưng performance tốt hơn
```

### Active-Active vs Active-Passive (Chủ Động-Chủ Động vs Chủ Động-Bị Động)

```
Active-Active (Cả Hai Region Đều Xử Lý Traffic):
Route 53 Weighted:
- US-East-1: weight 50
- EU-West-1: weight 50

Lợi ích: Tận dụng cả 2 regions, lower latency per region
Thách thức: Data synchronization giữa regions (DB replication)

Active-Passive (Một Region Chính, Một Region Dự Phòng):
Route 53 Failover:
- PRIMARY: US-East-1 (Active, Health Check bật)
- SECONDARY: EU-West-1 (Passive, Health Check bật)

Lợi ích: Đơn giản hơn về data consistency
Nhược điểm: EU-West-1 rảnh rỗi, lãng phí resource
```

---

## 🔧 Request Routing Advanced (Định Tuyến Request Nâng Cao)

### Header-based Routing (Định Tuyến Dựa Trên Header)

```
Use cases thực tế:

1. API Versioning (Phiên Bản API):
   Header: Accept: application/vnd.example.v2+json
   Rule: HTTP_HEADER "Accept" contains "v2" → API v2 Target Group
   Default: → API v1 Target Group

2. Internal vs External Traffic:
   Header: X-Internal-Request: true (thêm bởi VPN gateway)
   Rule: HTTP_HEADER "X-Internal-Request" = "true" → Internal TG
   Default: → External TG (rate-limited, audit-logged)

3. Mobile vs Desktop:
   Header: User-Agent contains "Mobile"
   Rule: → Mobile-optimized Target Group
   Default: → Desktop Target Group

4. Feature Flags (Cờ Tính Năng):
   Header: X-Feature-Beta: enabled
   Rule: → Beta Feature Target Group
   Default: → Stable Target Group
```

### Query String Routing (Định Tuyến Theo Chuỗi Truy Vấn)

```
Use cases:

1. API Version trong URL:
   URL: /api/users?version=2
   Rule: QUERY_STRING "version=2" → API v2 TG

2. Debug Mode:
   URL: /app?debug=true
   Rule: QUERY_STRING "debug=true" → Debug Target Group
   (với logging chi tiết hơn, trace headers)

3. Region-specific:
   URL: /app?region=asia
   Rule: QUERY_STRING "region=asia" → Asia Content TG
```

### IP-based Routing (Định Tuyến Theo IP)

```
Use cases:

1. VPN Office Traffic → Internal Backend:
   SOURCE_IP: 10.0.0.0/8 (VPN CIDR)
   Rule: → Internal Admin Target Group

2. Geographic Blocking (giới hạn ở CloudFront, không ALB):
   ALB chỉ có thể match theo CIDR, không có geo database
   → Dùng WAF Geographic Match Rule thay thế

3. Developer Access:
   SOURCE_IP: Dev team IP ranges
   Rule: → Dev/Staging Target Group với debug features
```

---

## 📊 Observability Patterns (Mẫu Quan Sát)

### Distributed Tracing (Truy Vết Phân Tán)

```
ALB tự động thêm header X-Amzn-Trace-Id vào mỗi request:
X-Amzn-Trace-Id: Root=1-5f084a56-7682e07d4f7ad8ce4e9b4c8d;Sampled=1

Trace ID format:
- Root: Unique trace ID cho request này
- Sampled: 1 = được thu thập bởi X-Ray

Tích hợp với AWS X-Ray:
1. Bật X-Ray tracing trên ALB
2. Ứng dụng propagate trace ID
3. X-Ray Service Map hiển thị latency qua từng service

Khi request đi qua: ALB → Service A → Service B → DB
X-Ray cho thấy: Bao nhiêu thời gian tại mỗi hop?
```

### Custom Access Log Analysis (Phân Tích Nhật Ký Truy Cập Tùy Chỉnh)

```sql
-- Amazon Athena query phân tích ALB access logs

-- Top 10 endpoints chậm nhất (P99 latency)
SELECT
  request_url,
  COUNT(*) as request_count,
  AVG(target_processing_time) as avg_latency_s,
  APPROX_PERCENTILE(target_processing_time, 0.99) as p99_latency_s
FROM alb_access_logs
WHERE time >= '2026-05-14'
  AND target_processing_time > 0
GROUP BY request_url
ORDER BY p99_latency_s DESC
LIMIT 10;

-- Error rate theo target
SELECT
  target_ip,
  COUNT(*) as total_requests,
  SUM(CASE WHEN target_status_code >= 500 THEN 1 ELSE 0 END) as error_count,
  ROUND(100.0 * SUM(CASE WHEN target_status_code >= 500 THEN 1 ELSE 0 END) / COUNT(*), 2) as error_rate_pct
FROM alb_access_logs
WHERE time >= '2026-05-14'
GROUP BY target_ip
ORDER BY error_rate_pct DESC;

-- Phân tích client IP để phát hiện abuse
SELECT
  client_ip,
  COUNT(*) as request_count,
  COUNT(DISTINCT request_url) as unique_urls
FROM alb_access_logs
WHERE time >= '2026-05-14'
GROUP BY client_ip
HAVING COUNT(*) > 10000
ORDER BY request_count DESC;
```

---

## 🏋️ Kiến Trúc Thực Tế Tổng Hợp

### E-Commerce Production Architecture

```
                          CloudFront (CDN + WAF)
                                │
                    ┌───────────┴──────────────┐
                    │                          │
            Static assets                 Dynamic content
            (S3 bucket)                        │
                                               ▼
                                    External ALB (internet-facing)
                                    ├── HTTP:80 → Redirect HTTPS
                                    └── HTTPS:443:
                                        ├── /cdn/* → CloudFront redirect
                                        ├── /api/v1/* → Internal ALB
                                        ├── /api/v2/* → Internal ALB v2
                                        ├── /ws/* → WebSocket TG (sticky)
                                        └── /* → Frontend TG

                                    Internal ALB (internal)
                                    ├── /api/v1/users → User Service TG
                                    ├── /api/v1/products → Product TG
                                    ├── /api/v1/orders → Order TG
                                    └── /api/v1/payment → Payment TG

                                    Internal ALB v2 (canary)
                                    └── /* → v2 Services TG (10% traffic)

Blue/Green deployment:
External ALB Rule /api/v2:
  weight v1: 90, weight v2: 10 → Canary
  weight v1: 0,  weight v2: 100 → Full cutover
```

---

## 💡 Cost Optimization (Tối Ưu Chi Phí)

### Phí Của ELB

```
ALB Pricing:
- Hourly fee: ~$0.008/LCU-hour
- LCU (Load Balancer Capacity Unit — Đơn Vị Năng Lực Cân Bằng Tải)
  1 LCU = 25 new connections/s OR
           3000 active connections OR
           1 GB/hour processed bytes OR
           1000 rule evaluations/s

NLB Pricing:
- Hourly fee: ~$0.006/NLCU-hour
- NLCU (Network Load Balancer Capacity Unit)

Tips tiết kiệm:
1. Consolidate ALBs: Dùng 1 ALB với nhiều listener rules thay vì nhiều ALBs
   → Tiết kiệm hourly fee + operational overhead
2. Tắt Cross-zone NLB nếu không cần (tránh $0.01/GB inter-AZ)
3. Xóa unused listeners và target groups
4. Dùng NLB thay ALB cho workloads không cần Layer 7 features
   (NLB rẻ hơn và nhanh hơn)
```

### Idle ALBs và Target Groups

```bash
# Tìm ALBs không có traffic (unhealthy metric == 0, request count == 0)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/my-alb/1234 \
  --start-time 2026-05-07T00:00:00Z \
  --end-time 2026-05-14T00:00:00Z \
  --period 604800 \
  --statistics Sum

# Nếu Sum = 0 trong 1 tuần → ALB không được dùng, xem xét xóa
```

---

## 🎓 Câu Hỏi Phỏng Vấn Nâng Cao

**Q: Giải thích cách thiết kế zero-downtime deployment với ALB?**

> Dùng Weighted Target Groups: tạo Target Group mới (green) với version mới, ban đầu set weight = 0. Deploy và warm up instances trong green Target Group. Dần dần tăng weight green (5% → 25% → 50% → 100%) trong khi giảm weight blue, monitor error rate và latency ở mỗi bước. Nếu có vấn đề, đổi weight green=0, blue=100 để rollback trong vài giây. Kết hợp với Deregistration Delay 60-300 giây để in-flight requests của blue hoàn thành. Toàn bộ quá trình không có downtime vì luôn có healthy targets.

**Q: Khi nào nên dùng NLB → ALB pattern?**

> Khi cần cả Static IP lẫn Layer 7 routing. Ví dụ: partner integration yêu cầu whitelist 2 IP cố định vào firewall của họ, nhưng backend cần path-based routing cho microservices. NLB frontend cung cấp Elastic IP cố định, ALB backend (đăng ký làm target của NLB với target type = alb) xử lý Layer 7 routing. Trade-off: double latency của 2 LB hops, chi phí cao hơn, nhưng đáp ứng cả 2 yêu cầu mâu thuẫn nhau.

**Q: Làm thế nào để debug intermittent 502 errors từ ALB?**

> Checklist theo thứ tự: (1) Kiểm tra HealthyHostCount — nếu = 0, tất cả targets unhealthy → check ứng dụng; (2) Kiểm tra ALB Access Logs — xem target_status_code và target_processing_time; 502 từ ALB (elb_status_code=502) vs target (target_status_code=502) rất khác nhau; (3) 502 từ ALB: thường do target đóng connection trước ALB → tăng keepalive timeout trên web server (Nginx/Apache); (4) 502 từ target: ứng dụng lỗi → kiểm tra application logs; (5) Kiểm tra Deregistration Delay — nếu Auto Scaling đang terminate instances, requests in-flight có thể nhận 502 nếu delay quá ngắn.

---

**Hoàn Thành:** Bạn đã đọc xong toàn bộ section 03-load-balancing!

**Tiếp Theo:** [04-dns-route53/README.md](../04-dns-route53/README.md) — DNS & Route 53

**Cập Nhật Lần Cuối:** 2026-05-14
**Trạng Thái:** ✅ Hoàn Thành
