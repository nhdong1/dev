# Health Checks — Kiểm Tra Sức Khỏe Endpoint

> Route 53 Health Checks (Kiểm Tra Sức Khỏe) theo dõi tính khả dụng của endpoints và tự động loại bỏ các endpoint không hoạt động khỏi DNS responses — tạo nên DNS failover tự động không cần can thiệp thủ công.

## 📚 Mục Lục

1. [Health Checks Là Gì?](#health-checks-là-gì)
2. [Các Loại Health Check](#các-loại-health-check)
3. [Cấu Hình Health Check](#cấu-hình-health-check)
4. [Health Check + Routing Policies](#health-check--routing-policies)
5. [DNS Failover Automation](#dns-failover-automation)
6. [CloudWatch Alarm Integration](#cloudwatch-alarm-integration)
7. [Best Practices](#best-practices)

---

## Health Checks Là Gì?

Route 53 Health Checks là dịch vụ **giám sát liên tục** gửi requests đến endpoints và xác định xem chúng có hoạt động bình thường không.

### Cơ Chế Hoạt Động

```
Route 53 Health Checkers (toàn cầu)
    ↓ Gửi request mỗi X giây
Endpoint của bạn (EC2, ALB, URL bất kỳ)
    ↓ Trả về response
Health Checker đánh giá: Healthy hay Unhealthy?
    ↓
Nếu Unhealthy: Route 53 loại endpoint khỏi DNS responses
Nếu Healthy: Route 53 tiếp tục đưa endpoint vào DNS responses
```

### Route 53 Health Checker Locations

AWS có **health checkers** đặt tại nhiều vị trí trên toàn cầu:
- Bắc Mỹ, Nam Mỹ, Châu Âu, Châu Á, Úc
- Health check được coi là **healthy** khi **hơn 18% health checkers báo cáo healthy**

---

## Các Loại Health Check

### Loại 1: Endpoint Health Check

Kiểm tra trực tiếp một **HTTP/HTTPS/TCP endpoint**:

```
Giao thức hỗ trợ:
  HTTP  → Port 80 (mặc định)
  HTTPS → Port 443 (mặc định)
  TCP   → Bất kỳ port

Kiểm tra bằng cách:
  HTTP/HTTPS: Route 53 gửi GET request, mong chờ HTTP 2xx hoặc 3xx
  TCP:        Route 53 thử thiết lập TCP connection

Cấu hình:
  IP address hoặc domain name của endpoint
  Port
  Path (với HTTP/HTTPS, ví dụ: /health)
  String matching (tùy chọn): tìm chuỗi trong response body
```

**Yêu cầu quan trọng:**

```
Endpoint phải accessible từ Route 53 health checker IPs
→ Nếu endpoint sau Security Group: phải cho phép IP của Route 53 health checkers
→ Danh sách IP: https://ip-ranges.amazonaws.com/ip-ranges.json (service: ROUTE53_HEALTHCHECKS)
```

### Loại 2: Calculated Health Check (Tổng Hợp)

Kết hợp kết quả của **nhiều health checks con** thành một kết quả tổng:

```
Calculated Health Check: "App Stack Healthy"
├── Health Check 1: Web Server
├── Health Check 2: Database
├── Health Check 3: Cache Server
└── Health Check 4: Background Worker

Cấu hình: Healthy khi ≥ 3/4 checks healthy (threshold tùy chỉnh)
```

**Use case:**
- Hệ thống phức tạp với nhiều components
- Chỉ failover khi một số lượng component nhất định bị lỗi
- Avoid false positives từ một component không quan trọng

### Loại 3: CloudWatch Alarm Health Check

Health check dựa trên trạng thái của **CloudWatch Alarm**:

```
CloudWatch Alarm: "High CPU Usage on RDS"
    ↓ Khi alarm state = ALARM
Route 53 Health Check → Unhealthy
    ↓
Route 53 failover sang secondary endpoint
```

**Lợi ích:**
- Monitor bất kỳ metric nào (CPU, latency, error rate, custom metrics)
- Không cần endpoint accessible từ internet
- Phù hợp cho **private endpoints** trong VPC

---

## Cấu Hình Health Check

### Thông Số Quan Trọng

```
Request Interval (Khoảng Thời Gian Gửi Request):
  Standard: 30 giây (mặc định, rẻ hơn)
  Fast:     10 giây (phát hiện lỗi nhanh hơn, đắt hơn 3x)

Failure Threshold (Ngưỡng Thất Bại):
  Số lần consecutive health checkers báo unhealthy trước khi coi là unhealthy
  Mặc định: 3 (tương đương 90 giây với standard interval)
  Range: 1-10

→ Thời gian phát hiện lỗi = Interval × Threshold
  Ví dụ: 30s × 3 = 90 giây (standard)
          10s × 1 = 10 giây (fast, aggressive)
```

### String Matching (Khớp Chuỗi)

```
HTTP/HTTPS health check có thể tìm chuỗi trong response body:

Path: /health
Expected string: "status":"ok"

Response body mẫu:
{
  "status": "ok",
  "database": "connected",
  "cache": "connected"
}

→ Route 53 kiểm tra 5120 bytes đầu tiên của response
→ Nếu không tìm thấy chuỗi → Unhealthy
```

### Health Check Endpoint /health

**Best practice:** Tạo dedicated `/health` endpoint trong ứng dụng:

```python
# FastAPI example
@app.get("/health")
async def health_check():
    # Kiểm tra các dependencies
    db_ok = await check_database()
    cache_ok = await check_cache()
    
    if db_ok and cache_ok:
        return {"status": "ok", "database": "connected", "cache": "connected"}
    else:
        raise HTTPException(status_code=503, detail="Service unavailable")
```

```go
// Go example
func healthHandler(w http.ResponseWriter, r *http.Request) {
    if !checkDatabase() {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{"status": "db_error"})
        return
    }
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
```

### Terraform

```hcl
resource "aws_route53_health_check" "app" {
  fqdn              = "app.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = "3"
  request_interval  = "30"

  # String matching
  search_string = "\"status\":\"ok\""

  tags = {
    Name        = "app-health-check"
    Environment = "production"
  }
}

# Calculated health check
resource "aws_route53_health_check" "combined" {
  type                   = "CALCULATED"
  child_health_threshold = 2  # Healthy khi ít nhất 2/3 children healthy
  
  child_healthchecks = [
    aws_route53_health_check.web.id,
    aws_route53_health_check.api.id,
    aws_route53_health_check.database.id,
  ]

  tags = {
    Name = "combined-health-check"
  }
}
```

---

## Health Check + Routing Policies

### Failover Routing với Health Check

```hcl
resource "aws_route53_record" "primary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "A"
  set_identifier = "primary"

  failover_routing_policy {
    type = "PRIMARY"
  }

  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "secondary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "api.example.com"
  type           = "A"
  set_identifier = "secondary"

  failover_routing_policy {
    type = "SECONDARY"
  }

  # Secondary không cần health check (optional)
  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

### Weighted Routing với Health Check

```
Weighted records với health checks:
  Endpoint A (weight=50): Unhealthy ❌
  Endpoint B (weight=30): Healthy ✅
  Endpoint C (weight=20): Healthy ✅

Route 53 tự động:
  Loại bỏ Endpoint A
  Phân phối lại: B = 30/(30+20) = 60%, C = 20/(30+20) = 40%
```

---

## DNS Failover Automation

### Kiến Trúc Failover Đầy Đủ

```
┌─────────────────────────────────────────────────────┐
│              Route 53 Health Checks                  │
│  [Health Checker 1] [Health Checker 2] [Checker N]   │
│           ↓              ↓                ↓          │
│  ┌──────────────────────────────────────────┐       │
│  │    Endpoint: api.example.com/health       │       │
│  │    Kiểm tra mỗi 30 giây                  │       │
│  └──────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────┘
                         ↓
              3 lần unhealthy liên tiếp
                         ↓
┌─────────────────────────────────────────────────────┐
│         Route 53 DNS Failover                        │
│                                                     │
│  TRƯỚC: api.example.com → Primary (us-east-1)       │
│  SAU:   api.example.com → Secondary (us-west-2)     │
└─────────────────────────────────────────────────────┘
                         ↓
              SNS Notification → CloudWatch → PagerDuty
```

### Timeline của Failover

```
T+0:   Primary endpoint bắt đầu fail
T+30:  Health checker 1 báo unhealthy (lần 1)
T+60:  Health checker 1 báo unhealthy (lần 2)
T+90:  Health checker 1 báo unhealthy (lần 3) → Threshold đạt
       Route 53 cập nhật DNS response
T+90+TTL: DNS cache hết hạn, clients nhận IP mới

→ Tổng thời gian failover: ~90 giây + TTL (ví dụ 60s = ~150 giây)
```

**Tối ưu failover time:**
```
Dùng Fast Health Check (10s interval) + Threshold = 1:
T+0:   Primary fail
T+10:  Route 53 phát hiện → DNS failover
T+10+TTL: Clients nhận IP mới

→ Với TTL = 60s: Tổng ~70 giây
→ Với TTL = 30s: Tổng ~40 giây
```

### Notifications Khi Failover

```hcl
# CloudWatch Alarm khi health check fails
resource "aws_cloudwatch_metric_alarm" "health_check_failed" {
  alarm_name          = "route53-health-check-failed"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 1
  metric_name         = "HealthCheckStatus"
  namespace           = "AWS/Route53"
  period              = 60
  statistic           = "Minimum"
  threshold           = 1

  dimensions = {
    HealthCheckId = aws_route53_health_check.app.id
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}
```

---

## CloudWatch Alarm Integration

### Khi Endpoint Không Accessible Từ Internet

Private endpoints (trong VPC, behind NAT) không thể bị health check trực tiếp từ Route 53. Giải pháp: dùng **CloudWatch Alarm Health Check**.

```
Kiến trúc:
EC2/RDS trong private subnet
    ↓ Metrics
CloudWatch (CPU, connections, errors)
    ↓ Alarm khi threshold vượt
Route 53 Health Check (type = CloudWatch Alarm)
    ↓ Unhealthy khi alarm = ALARM state
DNS Failover tự động
```

```hcl
# CloudWatch metric alarm cho private RDS
resource "aws_cloudwatch_metric_alarm" "rds_connections" {
  alarm_name          = "rds-too-many-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 60
  statistic           = "Average"
  threshold           = 100

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }
}

# Route 53 health check dựa trên alarm
resource "aws_route53_health_check" "rds" {
  type                            = "CLOUDWATCH_METRIC"
  cloudwatch_alarm_name           = aws_cloudwatch_metric_alarm.rds_connections.alarm_name
  cloudwatch_alarm_region         = "us-east-1"
  insufficient_data_health_status = "Healthy"
}
```

---

## Best Practices

### 1. Thiết Kế /health Endpoint

```
✅ Kiểm tra tất cả critical dependencies (DB, cache, external APIs)
✅ Trả về HTTP 200 khi healthy, 5xx khi unhealthy
✅ Response time dưới 2 giây (Route 53 timeout là 4 giây)
✅ Không require authentication (Route 53 không gửi credentials)
✅ Không cache response (thêm Cache-Control: no-cache)

❌ Không kiểm tra external services không quan trọng
❌ Không thực hiện operations nặng (database migration, etc.)
❌ Không expose thông tin nhạy cảm trong response
```

### 2. TTL và Health Checks

```
Khi dùng health check + failover:
  TTL nên thấp: 60-300 giây
  → Nếu TTL = 86400: Sau failover, clients còn cache IP cũ trong 24 giờ!

Công thức RTO (Recovery Time Objective — Thời Gian Phục Hồi Mục Tiêu):
  RTO ≈ Health Check Time + TTL
  RTO ≈ (Interval × Threshold) + TTL
  RTO ≈ (30s × 3) + 60s = 150 giây ≈ 2.5 phút
```

### 3. Security Groups cho Health Checks

```
Phải cho phép Route 53 health checker IPs vào Security Group:
  Protocol: HTTP/HTTPS
  Port: 80/443
  Source: Route 53 health checker CIDR ranges

# Lấy danh sách IPs:
curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | \
  jq -r '.prefixes[] | select(.service=="ROUTE53_HEALTHCHECKS") | .ip_prefix'
```

### 4. Monitoring Health Checks

```
CloudWatch Metrics quan trọng:
  HealthCheckStatus:        1 = Healthy, 0 = Unhealthy
  HealthCheckPercentageHealthy: % health checkers báo healthy
  ConnectionTime:           Thời gian kết nối TCP
  TimeToFirstByte:          Thời gian đến byte đầu tiên của response
```

---

## 🎯 Câu Hỏi Phỏng Vấn Thường Gặp

### Câu 1: Route 53 Health Check hoạt động thế nào?

**Trả lời:** Route 53 có health checkers phân tán toàn cầu gửi request đến endpoint của bạn theo chu kỳ (mặc định 30s). Nếu số lần thất bại liên tiếp vượt quá threshold (mặc định 3), Route 53 đánh dấu endpoint là unhealthy và loại nó khỏi DNS responses (khi kết hợp với các routing policies hỗ trợ health check như Failover, Weighted, Latency).

### Câu 2: RTO (Recovery Time) khi dùng Route 53 Failover là bao lâu?

**Trả lời:** RTO ≈ (Interval × Threshold) + TTL. Với cấu hình mặc định: (30s × 3) + TTL. Nếu TTL = 60s → RTO ≈ 150 giây. Để giảm RTO: dùng Fast Health Check (10s) + Threshold=1 + TTL=30s → RTO ≈ 40 giây.

### Câu 3: Endpoint trong private subnet có dùng Route 53 Health Check được không?

**Trả lời:** Không thể dùng trực tiếp endpoint health check vì Route 53 health checkers không thể reach private IPs. Giải pháp: dùng **CloudWatch Alarm Health Check** — monitor metrics trong CloudWatch, khi alarm triggered, Route 53 coi là unhealthy.

### Câu 4: Calculated Health Check dùng khi nào?

**Trả lời:** Khi hệ thống có nhiều components và muốn failover chỉ khi một số lượng components nhất định bị lỗi. Ví dụ: hệ thống có Web, API, DB — chỉ failover khi ≥2/3 components down, tránh false positive từ một component không quan trọng bị flapping.

---

## 🔗 Điều Hướng

- ← [3-routing-policies.md](./3-routing-policies.md) — Routing Policies
- → [5-resolver.md](./5-resolver.md) — Route 53 Resolver
- [README.md](./README.md) — Tổng quan section

---

**Cập Nhật Lần Cuối:** 2026-05-14
