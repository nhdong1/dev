# Routing Policies — Chính Sách Định Tuyến DNS

> Route 53 cung cấp **7 routing policies** (chính sách định tuyến) cho phép kiểm soát chính xác cách DNS queries được trả lời — từ đơn giản đến thông minh dựa trên vị trí địa lý, độ trễ, hay trọng số lưu lượng.

## 📚 Mục Lục

1. [Tổng Quan 7 Routing Policies](#tổng-quan-7-routing-policies)
2. [Simple Routing — Định Tuyến Đơn Giản](#1-simple-routing--định-tuyến-đơn-giản)
3. [Weighted Routing — Định Tuyến Có Trọng Số](#2-weighted-routing--định-tuyến-có-trọng-số)
4. [Latency-based Routing — Định Tuyến Dựa Trên Độ Trễ](#3-latency-based-routing--định-tuyến-dựa-trên-độ-trễ)
5. [Failover Routing — Định Tuyến Chuyển Đổi Dự Phòng](#4-failover-routing--định-tuyến-chuyển-đổi-dự-phòng)
6. [Geolocation Routing — Định Tuyến Theo Vị Trí](#5-geolocation-routing--định-tuyến-theo-vị-trí)
7. [Geoproximity Routing — Định Tuyến Theo Khoảng Cách](#6-geoproximity-routing--định-tuyến-theo-khoảng-cách)
8. [Multi-value Answer — Đa Giá Trị](#7-multi-value-answer--đa-giá-trị)
9. [So Sánh Tổng Hợp](#so-sánh-tổng-hợp)
10. [Kết Hợp Policies](#kết-hợp-policies)

---

## Tổng Quan 7 Routing Policies

```
Câu hỏi định hướng: "Mục tiêu của bạn là gì?"

Đơn giản, 1 endpoint         → Simple
Phân bổ lưu lượng theo tỷ lệ → Weighted (A/B testing, canary deploy)
Nhanh nhất cho người dùng    → Latency-based
Chuyển đổi khi có sự cố      → Failover
Theo quốc gia/khu vực        → Geolocation
Theo khoảng cách + bias       → Geoproximity (Traffic Flow)
Nhiều IPs, tránh SPOF        → Multi-value Answer
```

---

## 1. Simple Routing — Định Tuyến Đơn Giản

### Đặc Điểm

- Một record với **một hoặc nhiều IP addresses**
- Khi có nhiều IP: Route 53 trả về **ngẫu nhiên** (random)
- **Không hỗ trợ health checks** trên từng giá trị
- Không thể kết hợp với Health Checks để loại bỏ unhealthy endpoints

### Khi Nào Dùng

```
✅ Môi trường đơn giản, 1 server
✅ Development/staging
✅ Static website (S3, CloudFront)
✅ Khi không cần failover hay traffic splitting
```

### Cách Hoạt Động

```
DNS Query: www.example.com?

Record: www.example.com A [203.0.113.10, 203.0.113.20, 203.0.113.30]

Route 53 trả về: 203.0.113.20 (ngẫu nhiên)
→ Không phải load balancing thực sự — chỉ là random selection
```

### Terraform

```hcl
resource "aws_route53_record" "simple" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"
  ttl     = 300
  records = ["203.0.113.10", "203.0.113.20"]
  # Không cần routing_policy — Simple là mặc định
}
```

---

## 2. Weighted Routing — Định Tuyến Có Trọng Số

### Đặc Điểm

- Phân phối traffic theo **tỷ lệ phần trăm** định sẵn
- Mỗi record có một **weight** (trọng số) từ 0-255
- Tỷ lệ = weight của record / tổng weight của tất cả records
- Hỗ trợ Health Checks — loại bỏ unhealthy endpoints

### Công Thức

```
Traffic đến endpoint A = Weight_A / (Weight_A + Weight_B + Weight_C)

Ví dụ:
  app-v1: weight = 80
  app-v2: weight = 20
  
Traffic app-v1 = 80 / (80 + 20) = 80%
Traffic app-v2 = 20 / (80 + 20) = 20%
```

### Use Cases

```
✅ A/B Testing:
   Version A: weight=90, Version B: weight=10
   → 90% user thấy version cũ, 10% thấy version mới

✅ Canary Deployment (Triển Khai Canary):
   Old: weight=95, New: weight=5
   → Dần dần tăng weight của new version

✅ Blue/Green Deployment (Triển Khai Xanh/Lục):
   Blue (cũ): weight=100 → weight=0
   Green (mới): weight=0 → weight=100

✅ Multi-region load distribution:
   us-east-1: weight=60
   us-west-2: weight=40
```

### Weight = 0: Đặc Biệt!

```
Weight = 0: Record TẮT hoàn toàn — không nhận traffic
            (dùng để tắt một endpoint mà không cần xóa record)

Tất cả records weight = 0: Route 53 trả về tất cả bình đẳng
```

### Terraform

```hcl
resource "aws_route53_record" "v1" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "v1"
  ttl            = 60

  weighted_routing_policy {
    weight = 80
  }

  records = ["203.0.113.10"]
}

resource "aws_route53_record" "v2" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "v2"
  ttl            = 60

  weighted_routing_policy {
    weight = 20
  }

  records = ["203.0.113.20"]
}
```

---

## 3. Latency-based Routing — Định Tuyến Dựa Trên Độ Trễ

### Đặc Điểm

- Route 53 chuyển hướng người dùng đến **AWS region có độ trễ thấp nhất**
- Dựa trên **đo lường độ trễ thực tế** giữa người dùng và các AWS regions
- Không dựa trên vị trí địa lý — mà dựa trên network latency thực tế
- Hỗ trợ Health Checks

### Khi Nào Dùng

```
✅ Ứng dụng multi-region muốn tối ưu user experience
✅ API backend cần response time thấp nhất
✅ Gaming, real-time applications
✅ Global SaaS (Software as a Service) products
```

### Cách Hoạt Động

```
Người dùng ở Hà Nội query: api.example.com

Route 53 đo độ trễ từ vị trí người dùng đến:
  us-east-1:      180ms
  ap-southeast-1: 35ms  ← Thấp nhất!
  eu-west-1:      250ms

→ Trả về IP của endpoint tại ap-southeast-1
```

**Lưu ý quan trọng:** Đây là latency đến **AWS region**, không phải đến endpoint cụ thể. Nếu server trong region đó bị chậm, Latency routing không thể biết.

### Terraform

```hcl
resource "aws_route53_record" "api_us" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "A"
  set_identifier = "us-east-1"
  ttl            = 60

  latency_routing_policy {
    region = "us-east-1"
  }

  records = ["203.0.113.10"]
}

resource "aws_route53_record" "api_sg" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "A"
  set_identifier = "ap-southeast-1"
  ttl            = 60

  latency_routing_policy {
    region = "ap-southeast-1"
  }

  records = ["203.0.114.20"]
}
```

---

## 4. Failover Routing — Định Tuyến Chuyển Đổi Dự Phòng

### Đặc Điểm

- Cấu hình **Primary** (chính) và **Secondary** (dự phòng) endpoints
- Khi Primary unhealthy → **tự động** chuyển sang Secondary
- **Bắt buộc** phải có Health Check trên Primary record
- Secondary có thể là static page, S3 bucket, hoặc region khác

### Cách Hoạt Động

```
Trạng thái bình thường:
  api.example.com → Primary (us-east-1, 203.0.113.10) ✅ Healthy

Khi Primary down:
  Health Check phát hiện Primary unhealthy
  Route 53 tự động chuyển:
  api.example.com → Secondary (us-west-2, 203.0.114.20)

Khi Primary phục hồi:
  Health Check phát hiện Primary healthy
  Route 53 tự động chuyển lại Primary
```

### Active-Passive vs Active-Active

```
Active-Passive (Failover Routing):
  Primary xử lý 100% traffic
  Secondary chờ sẵn, chỉ nhận traffic khi Primary down
  → Dùng khi Secondary không đủ capacity để xử lý toàn bộ traffic

Active-Active (Weighted/Latency với Health Checks):
  Cả hai endpoints đều nhận traffic
  Khi một endpoint down, Route 53 loại bỏ nó
  → Dùng khi cần scale out và high availability
```

### Failover + S3 Static Page (Maintenance Page)

```hcl
# Primary: ALB chạy ứng dụng
resource "aws_route53_record" "primary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "www.example.com"
  type           = "A"
  set_identifier = "primary"

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}

# Secondary: S3 static maintenance page
resource "aws_route53_record" "secondary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "www.example.com"
  type           = "A"
  set_identifier = "secondary"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_s3_bucket_website_configuration.maintenance.website_endpoint
    zone_id                = "Z3AQBSTGFYJSTF" # S3 zone ID us-east-1
    evaluate_target_health = false
  }
}
```

---

## 5. Geolocation Routing — Định Tuyến Theo Vị Trí

### Đặc Điểm

- Route 53 xác định **quốc gia hoặc khu vực** của người dùng
- Chuyển hướng đến endpoint được cấu hình cho khu vực đó
- Dựa trên **IP geolocation** — không phải độ trễ
- Phải cấu hình **Default record** cho các vị trí không khớp

### Granularity (Mức Độ Phân Tách)

```
Cấp độ cấu hình (từ cụ thể đến tổng quát):
1. Continent (Châu Lục): Asia, Europe, North America...
2. Country (Quốc Gia): Vietnam (VN), USA (US), UK (GB)...
3. Subdivision (Bang/Tỉnh, chỉ US): California, New York...
4. Default: Tất cả vị trí còn lại
```

**Ưu tiên:** Specific > Continent > Default

### Use Cases

```
✅ Tuân thủ pháp lý (GDPR, data residency):
   EU users → EU servers (dữ liệu không rời EU)
   US users → US servers

✅ Nội dung ngôn ngữ địa phương:
   Vietnam users → Tiếng Việt website (VN servers)
   Japan users → Tiếng Nhật website (JP servers)

✅ Restriction (Hạn Chế):
   Block specific countries → Trả về 404 hoặc "Service not available"

✅ Giá khác nhau theo khu vực:
   US pricing → US servers
   EU pricing → EU servers
```

### Ví Dụ Cấu Hình

```hcl
# Vietnam users → Singapore endpoint
resource "aws_route53_record" "vn" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "www.example.com"
  type           = "A"
  set_identifier = "vietnam"
  ttl            = 300

  geolocation_routing_policy {
    country = "VN"
  }

  records = ["ap-southeast-1-ip"]
}

# EU users → Frankfurt endpoint
resource "aws_route53_record" "eu" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "www.example.com"
  type           = "A"
  set_identifier = "europe"
  ttl            = 300

  geolocation_routing_policy {
    continent = "EU"
  }

  records = ["eu-central-1-ip"]
}

# Default → US East endpoint (bắt buộc có default!)
resource "aws_route53_record" "default" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "www.example.com"
  type           = "A"
  set_identifier = "default"
  ttl            = 300

  geolocation_routing_policy {
    country = "*"  # Default
  }

  records = ["us-east-1-ip"]
}
```

**QUAN TRỌNG:** Nếu không có Default record và người dùng từ vị trí không được cấu hình → Route 53 trả về **"no answer"** → Website không truy cập được!

### Geolocation vs Latency

| Tiêu Chí | Geolocation | Latency-based |
|---------|-------------|---------------|
| Cơ chế | Dựa trên IP location | Dựa trên độ trễ mạng |
| Mục đích | Kiểm soát/Tuân thủ | Tối ưu hiệu suất |
| Ví dụ | EU users → EU servers | Singapore users → Singapore (nếu nhanh hơn Tokyo) |
| Kiểm soát nội dung | Có | Không |

---

## 6. Geoproximity Routing — Định Tuyến Theo Khoảng Cách

### Đặc Điểm

- Dựa trên **khoảng cách địa lý** giữa người dùng và endpoints
- Hỗ trợ **Bias** (độ lệch) để điều chỉnh "vùng ảnh hưởng" của từng endpoint
- Yêu cầu **Route 53 Traffic Flow** (tính năng bổ sung, $50/tháng/policy)
- Hỗ trợ cả AWS resources (theo region) và non-AWS resources (theo lat/long)

### Bias: Kéo Giãn Vùng Ảnh Hưởng

```
Bias dương (+1 đến +99): Mở rộng vùng ảnh hưởng (kéo traffic vào)
Bias âm (-1 đến -99):   Thu nhỏ vùng ảnh hưởng (đẩy traffic ra)

Ví dụ:
  us-east-1 bias = +25  → Kéo thêm traffic vào US East (ngay cả user gần EU)
  eu-west-1 bias = 0    → Giữ nguyên
```

### Khi Nào Dùng

```
✅ Multi-region, muốn kiểm soát chính xác vùng ảnh hưởng
✅ Khi cần di chuyển traffic từ region này sang region khác dần dần
✅ Non-AWS endpoints (datacenter on-premises)
✅ Khi Latency-based không đủ kiểm soát
```

### So Sánh Geolocation vs Geoproximity

```
Geolocation:  "User ở Vietnam → luôn đến VN endpoint"
              (Cứng nhắc theo quốc gia/châu lục)

Geoproximity: "User ở Vietnam → đến endpoint gần nhất"
              (Linh hoạt theo khoảng cách, có thể điều chỉnh bằng bias)
```

---

## 7. Multi-value Answer — Đa Giá Trị

### Đặc Điểm

- Trả về **nhiều IP addresses** (tối đa 8) trong mỗi DNS response
- **Hỗ trợ Health Checks** — chỉ trả về các healthy endpoints
- Client chọn một IP ngẫu nhiên từ danh sách
- **Không phải load balancer** — chỉ là client-side random selection

### Khác với Simple Routing

```
Simple Routing:
  Có thể có nhiều IPs → trả về tất cả (bao gồm cả unhealthy)
  Không có health check per record

Multi-value Answer:
  Nhiều IPs → chỉ trả về healthy IPs
  Mỗi record có health check riêng
```

### Khi Nào Dùng

```
✅ Muốn client-side load distribution cơ bản
✅ Cần loại bỏ unhealthy endpoints khỏi DNS response
✅ Tăng tính dự phòng cho các ứng dụng nhỏ
✅ Khi không muốn dùng ALB nhưng cần cơ bản failover
❌ Không thay thế được load balancer thực sự (ALB/NLB)
```

### Cách Hoạt Động

```
Multi-value record: api.example.com

Endpoints:
  203.0.113.10 → Health Check: ✅ Healthy
  203.0.113.20 → Health Check: ✅ Healthy
  203.0.113.30 → Health Check: ❌ Unhealthy

DNS Response: [203.0.113.10, 203.0.113.20]
(203.0.113.30 bị loại bỏ vì unhealthy)

Client chọn ngẫu nhiên → kết nối đến 203.0.113.10 hoặc 203.0.113.20
```

---

## So Sánh Tổng Hợp

| Policy | Mục Đích | Health Check | Use Case Điển Hình |
|--------|----------|-------------|---------------------|
| Simple | 1 endpoint đơn giản | ❌ (không per-record) | Static site, dev env |
| Weighted | Phân tải theo tỷ lệ | ✅ | A/B test, canary deploy |
| Latency | Nhanh nhất cho user | ✅ | Multi-region global app |
| Failover | Active-passive DR | ✅ (bắt buộc Primary) | Disaster recovery |
| Geolocation | Theo quốc gia/vùng | ✅ | GDPR compliance, localization |
| Geoproximity | Theo khoảng cách + bias | ✅ | Traffic migration, fine-grained geo |
| Multi-value | Nhiều IPs, loại unhealthy | ✅ | Basic redundancy |

### Sơ Đồ Quyết Định

```
Muốn gì?
│
├─ Chỉ 1 endpoint đơn giản → Simple
│
├─ Phân lưu lượng theo tỷ lệ (A/B, canary) → Weighted
│
├─ Người dùng vào endpoint nhanh nhất → Latency-based
│
├─ Tự động failover khi có sự cố → Failover
│
├─ Kiểm soát theo quốc gia/châu lục (GDPR, localization) → Geolocation
│
├─ Kiểm soát theo khoảng cách + điều chỉnh bias → Geoproximity
│
└─ Nhiều endpoints, loại unhealthy, client chọn → Multi-value
```

---

## Kết Hợp Policies

Route 53 cho phép **kết hợp policies** bằng cách dùng nhiều record cùng tên nhưng khác `set_identifier`.

### Ví Dụ: Latency + Weighted (Blue/Green Per Region)

```
api.example.com:
  us-east-1:
    Blue (weight=80): 203.0.113.10
    Green (weight=20): 203.0.113.20

  ap-southeast-1:
    Blue (weight=80): 203.0.114.10
    Green (weight=20): 203.0.114.20

→ Route 53 trước tiên dùng Latency để chọn region
→ Sau đó dùng Weighted để phân chia trong region
```

### Ví Dụ: Geolocation + Failover (Compliance + DR)

```
EU users → eu-west-1 Primary → eu-west-1 Secondary (Failover trong EU)
US users → us-east-1 Primary → us-west-2 Secondary (Failover trong US)

→ Đảm bảo EU data không rời khỏi EU
→ Có failover tự động trong từng khu vực
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Giải thích tất cả 7 routing policies của Route 53

**Trả lời ngắn gọn:**
1. **Simple:** 1 record → 1+ IPs, random, không health check per-record
2. **Weighted:** Phân lưu lượng theo weight, A/B testing
3. **Latency:** Endpoint có độ trễ thấp nhất từ user's location
4. **Failover:** Primary/Secondary, auto-failover qua health check
5. **Geolocation:** Theo quốc gia/châu lục, cần có Default
6. **Geoproximity:** Theo khoảng cách + bias, cần Traffic Flow
7. **Multi-value:** Nhiều IPs, loại unhealthy endpoints

### Câu 2: Khác nhau giữa Latency-based và Geolocation routing?

**Trả lời:** Latency dựa trên network latency thực tế → cho kết quả nhanh nhất. Geolocation dựa trên vị trí địa lý → cho phép kiểm soát data residency và nội dung theo vùng. Người dùng ở VN có thể được Latency routing gửi sang Singapore (nếu nhanh hơn), nhưng Geolocation routing sẽ luôn gửi đến endpoint được cấu hình cho VN.

### Câu 3: Multi-value có thay thế được Load Balancer không?

**Trả lời:** Không. Multi-value chỉ là DNS-level client-side selection — không có connection pooling, không có health monitoring liên tục, không phân phối đều (random). ALB/NLB có layer 7/4 routing thực sự, health checks chi tiết hơn, session handling, SSL termination. Multi-value phù hợp cho simple redundancy, không phải production load balancing.

### Câu 4: Khi nào dùng Failover vs Weighted với weight=0?

**Trả lời:**
- **Failover:** Khi muốn auto-failover tự động dựa trên health check, primary/secondary concept rõ ràng
- **Weighted weight=0:** Khi muốn TẮT hoàn toàn một endpoint thủ công mà không cần xóa record, hoặc khi đang maintenance

---

## 🔗 Điều Hướng

- ← [2-hosted-zones.md](./2-hosted-zones.md) — Hosted Zones
- → [4-health-checks.md](./4-health-checks.md) — Health Checks
- [README.md](./README.md) — Tổng quan section

---

**Cập Nhật Lần Cuối:** 2026-05-14
