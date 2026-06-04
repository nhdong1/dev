# ⚖️ Load Balancer Issues — Xử Lý Sự Cố ALB/NLB

> Hướng dẫn chẩn đoán các vấn đề phổ biến với ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng) và NLB (Network Load Balancer — Cân Bằng Tải Mạng): health check failures (Thất Bại Kiểm Tra Sức Khỏe), 5xx errors, SSL certificate issues, và performance degradation.

---

## 📚 Mục Lục

1. [Kiến Trúc ALB/NLB — Hiểu Để Debug](#kiến-trúc-albnlb--hiểu-để-debug)
2. [Lỗi 503 — Service Unavailable](#lỗi-503--service-unavailable)
3. [Health Check Failures — Targets Unhealthy](#health-check-failures--targets-unhealthy)
4. [Lỗi 502 — Bad Gateway](#lỗi-502--bad-gateway)
5. [SSL/TLS Certificate Issues](#ssltls-certificate-issues)
6. [NLB Specific Issues](#nlb-specific-issues)
7. [Performance Degradation — Hiệu Suất Suy Giảm](#performance-degradation--hiệu-suất-suy-giảm)
8. [Phân Tích ALB Access Logs](#phân-tích-alb-access-logs)
9. [Checklist Nhanh](#checklist-nhanh)

---

## 🏗️ Kiến Trúc ALB/NLB — Hiểu Để Debug

```
Client
  │
  ▼
Route 53 DNS → ALB DNS Name (nhiều IP, mỗi AZ một IP)
  │
  ▼
ALB Listener (Port 443, SSL termination)
  │
  ├── Listener Rule 1: /api/* → Target Group A (EC2 Apps)
  ├── Listener Rule 2: /admin/* → Target Group B (EC2 Admin)
  └── Default Rule: → Target Group A
          │
          ▼
     Health Check → EC2 Target /health → 200 OK?
          │
          ▼
     EC2 Instance (port 8080)
```

### Luồng Request ALB

1. Client gửi HTTPS request đến ALB
2. ALB terminate SSL (Kết Thúc SSL) — giải mã traffic
3. ALB match listener rules để chọn target group
4. ALB chọn healthy target theo thuật toán load balancing
5. ALB forward request đến target (EC2/container/Lambda)
6. Target xử lý và trả response về ALB
7. ALB forward response về client

---

## 🔴 Lỗi 503 — Service Unavailable

### Nguyên Nhân Gốc Rễ

**503 từ ALB nghĩa là: Không có healthy target nào để route request đến.**

### Bước 1: Kiểm Tra Target Group Health

```bash
# Xem trạng thái health của tất cả targets
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --query 'TargetHealthDescriptions[].[Target.Id, Target.Port, TargetHealth.State, TargetHealth.Reason, TargetHealth.Description]'
```

**Các trạng thái có thể:**
| State | Ý Nghĩa |
|-------|---------|
| `healthy` | Target đang nhận traffic |
| `unhealthy` | Health check fail |
| `initial` | Đang thực hiện health check lần đầu |
| `draining` | Target đang deregister, đợi connections cũ đóng |
| `unused` | Target không được dùng (target group không gán vào LB) |

### Bước 2: Xem Target Group Có Targets Không

```bash
# Kiểm tra targets trong target group
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

**Nếu không có target nào:** Auto Scaling Group (Nhóm Tự Động Mở Rộng) chưa gán vào target group, hoặc tất cả instances đã terminated.

### Bước 3: Kiểm Tra Load Balancer Listener

```bash
# Xem listeners và rules
aws elbv2 describe-listeners \
  --load-balancer-arn <alb-arn>

# Xem rules của listener
aws elbv2 describe-rules \
  --listener-arn <listener-arn>
```

**Lỗi thường gặp:** Default action của listener trỏ vào target group đã bị xóa hoặc trống.

---

## 🟡 Health Check Failures — Targets Unhealthy

### Tại Sao Health Check Fail?

Khi target được đánh dấu unhealthy, ALB ngừng gửi traffic đến target đó.

### Bước 1: Xem Chi Tiết Health Check Configuration

```bash
# Xem health check config của target group
aws elbv2 describe-target-groups \
  --target-group-arns <target-group-arn> \
  --query 'TargetGroups[].[HealthCheckPath, HealthCheckPort, HealthCheckProtocol, HealthCheckIntervalSeconds, HealthyThresholdCount, UnhealthyThresholdCount]'
```

**Điểm kiểm tra:**
| Parameter | Ý Nghĩa | Lỗi Thường Gặp |
|-----------|---------|----------------|
| `HealthCheckPath` | URL path ALB gọi | `/health` không tồn tại, trả về 404 |
| `HealthCheckPort` | Port ALB kiểm tra | Port app thực tế khác port này |
| `HealthCheckProtocol` | HTTP/HTTPS | App dùng HTTP nhưng config HTTPS |
| `HealthyThresholdCount` | Số lần thành công để healthy | Quá cao → mất nhiều thời gian recover |
| `UnhealthyThresholdCount` | Số lần thất bại để unhealthy | Quá thấp → flapping (unhealthy liên tục) |

### Bước 2: Test Health Check Thủ Công

```bash
# Từ trong VPC, test trực tiếp health check endpoint
curl -v http://<target-ip>:<health-check-port><health-check-path>

# Ví dụ
curl -v http://10.0.1.50:8080/health
```

**Kết quả mong đợi:** HTTP 200 OK

**Lỗi thường gặp:**
- `/health` trả về `401 Unauthorized` — app yêu cầu authentication
- `/health` trả về `404 Not Found` — path không tồn tại
- Connection refused — app không chạy trên port đó
- Connection timeout — Security Group chặn ALB

### Bước 3: Kiểm Tra Security Group Của Target

ALB cần kết nối đến target để health check. Security Group của target **phải ALLOW inbound từ ALB Security Group**.

```bash
# Xem SG của ALB
aws elbv2 describe-load-balancers \
  --load-balancer-arns <alb-arn> \
  --query 'LoadBalancers[].SecurityGroups'

# Kiểm tra SG của EC2 target có ALLOW từ ALB SG không
aws ec2 describe-security-groups \
  --group-ids <ec2-sg-id> \
  --query 'SecurityGroups[].IpPermissions[?UserIdGroupPairs[?GroupId==`<alb-sg-id>`]]'
```

### Bước 4: App Mất Thời Gian Khởi Động

Nếu targets mới launch bị unhealthy ngay lập tức:

```bash
# Xem slow start configuration và deregistration delay
aws elbv2 describe-target-group-attributes \
  --target-group-arn <target-group-arn> \
  --query 'Attributes[?Key==`slow_start.duration_seconds` || Key==`deregistration_delay.timeout_seconds`]'
```

**Giải pháp:** Tăng `UnhealthyThresholdCount` hoặc bật **slow start** — để app có thời gian warm up (Làm Nóng) trước khi nhận full traffic.

---

## 🟠 Lỗi 502 — Bad Gateway

### Nguyên Nhân

502 từ ALB nghĩa là: ALB kết nối được đến target nhưng target **trả về response không hợp lệ**.

### Phân Biệt 502 vs 503

| Lỗi | Nguyên Nhân |
|-----|-------------|
| `503` | Không có healthy target |
| `502` | Target healthy nhưng response bị lỗi |

### Nguyên Nhân 502 Thường Gặp

1. **App crash hoặc timeout** — Target xử lý quá lâu, vượt quá idle timeout của ALB (mặc định 60 giây)
2. **App trả về response không đúng format HTTP**
3. **Keep-alive timeout mismatch** — App đóng connection sớm hơn ALB expect
4. **Memory/CPU exhaustion** — App hết tài nguyên, không xử lý được request mới

```bash
# Kiểm tra idle timeout của ALB
aws elbv2 describe-load-balancer-attributes \
  --load-balancer-arn <alb-arn> \
  --query 'Attributes[?Key==`idle_timeout.timeout_seconds`]'

# Tăng idle timeout nếu app cần thời gian xử lý lâu hơn
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <alb-arn> \
  --attributes Key=idle_timeout.timeout_seconds,Value=120
```

### Debug 502 Qua Access Logs

ALB Access Log có trường `target_processing_time` (thời gian target xử lý):
```
# Tìm requests có target_processing_time lớn
# Trong S3 access log:
grep " 502 " alb-access-log.log | awk '{print $6, $7, $8}' | head -20
# target_processing_time request_processing_time response_processing_time
```

---

## 🔒 SSL/TLS Certificate Issues

### Lỗi "SSL Handshake Failed" Hoặc Certificate Warning

```bash
# Kiểm tra certificate gán vào listener
aws elbv2 describe-listener-certificates \
  --listener-arn <listener-arn>

# Xem chi tiết certificate từ ACM
aws acm describe-certificate \
  --certificate-arn <cert-arn> \
  --query 'Certificate.[DomainName, SubjectAlternativeNames, Status, NotAfter]'
```

**Checklist SSL:**
```
□ Certificate đang ở trạng thái "ISSUED" không?
□ Certificate domain match với hostname người dùng đang dùng?
□ Wildcard cert (*.example.com) có cover subdomain không?
□ Certificate chưa expired (NotAfter > today)?
□ SNI (Server Name Indication — Chỉ Định Tên Máy Chủ) được bật?
```

### Certificate Expired (Hết Hạn)

```bash
# Tìm certificates sắp hết hạn trong 30 ngày
aws acm list-certificates \
  --certificate-statuses ISSUED \
  --query 'CertificateSummaryList[].[CertificateArn, DomainName]' | \
  xargs -I {} aws acm describe-certificate --certificate-arn {} \
  --query 'Certificate.[DomainName, NotAfter]'
```

ACM (AWS Certificate Manager) tự động renew certificates. Nếu không renew được:
1. CNAME record validation vẫn còn trong hosted zone?
2. Domain ownership validation còn hợp lệ?

### HTTPS → HTTP Redirect Không Hoạt Động

```bash
# Kiểm tra listener rule redirect
aws elbv2 describe-rules \
  --listener-arn <http-listener-arn> \
  --query 'Rules[].[Conditions, Actions]'
```

**Rule redirect từ HTTP → HTTPS phải có:**
```json
{
  "Actions": [{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }]
}
```

---

## 🔧 NLB Specific Issues

### NLB Không Forward Traffic Đến Targets

NLB (Layer 4) khác ALB (Layer 7) — không terminate TCP, pass-through thực sự.

**Vấn đề đặc thù NLB:**

1. **Source IP preservation** — NLB preserve client IP, nên Security Group của target phải ALLOW từ **client IP** (không phải từ NLB IP)

```bash
# Kiểm tra target group attributes
aws elbv2 describe-target-group-attributes \
  --target-group-arn <tg-arn> \
  --query 'Attributes[?Key==`preserve_client_ip.enabled`]'
```

Nếu `preserve_client_ip = true`: Security Group của EC2 phải mở cho internet (hoặc client CIDR range).

2. **Cross-zone load balancing** — Mặc định NLB tắt cross-zone, ALB bật. EC2 ở AZ không có NLB node sẽ không nhận traffic.

```bash
# Bật cross-zone load balancing
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <nlb-arn> \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

3. **Elastic IP và Security Groups** — NLB không hỗ trợ Security Groups trực tiếp (chỉ ALB mới có). Kiểm soát traffic NLB qua Security Group của **target EC2**.

### NLB Health Check UDP/TCP

```bash
# NLB health check không support HTTP path — dùng TCP/HTTPS health check
aws elbv2 modify-target-group \
  --target-group-arn <tg-arn> \
  --health-check-protocol TCP \
  --health-check-port traffic-port
```

---

## 📉 Performance Degradation — Hiệu Suất Suy Giảm

### Kiểm Tra CloudWatch Metrics Của ALB

```bash
# Request Count trong 5 phút
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=<alb-full-name> \
  --start-time 2026-05-14T10:00:00 \
  --end-time 2026-05-14T11:00:00 \
  --period 300 \
  --statistics Sum

# Target Response Time trung bình
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=<alb-full-name> \
  --start-time 2026-05-14T10:00:00 \
  --end-time 2026-05-14T11:00:00 \
  --period 300 \
  --statistics Average
```

### Metrics Quan Trọng Cần Theo Dõi

| Metric | Ngưỡng Cảnh Báo | Ý Nghĩa |
|--------|----------------|---------|
| `TargetResponseTime` | > 1 giây | App đang xử lý chậm |
| `HTTPCode_ELB_5XX_Count` | > 0 | ALB-side errors |
| `HTTPCode_Target_5XX_Count` | > 0 | App-side errors |
| `HealthyHostCount` | < min instances | Targets đang chết |
| `UnHealthyHostCount` | > 0 | Cần điều tra ngay |
| `RejectedConnectionCount` | > 0 | ALB bị quá tải (surge queue đầy) |

### Surge Queue và Connection Spikes

Khi traffic tăng đột biến:
```
ALB có Surge Queue (Hàng Đợi Tăng Vọt) mặc định 1024 connections
Khi đầy → reject connections mới → client nhận 503
```

**Giải pháp:**
- Scale out (Mở Rộng Ra) target EC2 instances nhanh hơn
- Bật Auto Scaling với aggressive scaling policies
- Preemptively scale trước giờ peak

---

## 📋 Phân Tích ALB Access Logs

### Bật Access Logs

```bash
# Bật ALB access logs ghi vào S3
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <alb-arn> \
  --attributes Key=access_logs.s3.enabled,Value=true \
               Key=access_logs.s3.bucket,Value=my-alb-logs-bucket \
               Key=access_logs.s3.prefix,Value=prod-alb
```

### Format Log

```
timestamp elb client:port target:port request_processing_time target_processing_time response_processing_time elb_status_code target_status_code received_bytes sent_bytes "request" "user_agent" ssl_cipher ssl_protocol target_group_arn "trace_id" "domain_name" "chosen_cert_arn" matched_rule_priority request_creation_time "actions_executed" "redirect_url" "error_reason" "target:port_list" "target_status_code_list" "classification" "classification_reason"
```

### Dùng Athena Để Query ALB Logs

```sql
-- Tạo table Athena cho ALB logs
CREATE EXTERNAL TABLE alb_logs (
  type string, time string, elb string,
  client_ip string, client_port int,
  target_ip string, target_port int,
  request_processing_time double,
  target_processing_time double,
  response_processing_time double,
  elb_status_code int, target_status_code int,
  received_bytes bigint, sent_bytes bigint,
  request_verb string, request_url string,
  request_proto string, user_agent string,
  ssl_cipher string, ssl_protocol string,
  target_group_arn string, trace_id string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
...
LOCATION 's3://my-alb-logs-bucket/prod-alb/';

-- Top 10 slowest requests
SELECT request_url, target_processing_time, elb_status_code
FROM alb_logs
WHERE target_processing_time > 1.0
ORDER BY target_processing_time DESC
LIMIT 10;

-- 5xx error rate theo giờ
SELECT date_trunc('hour', from_iso8601_timestamp(time)) as hour,
       COUNT(*) as total_requests,
       SUM(CASE WHEN elb_status_code >= 500 THEN 1 ELSE 0 END) as errors_5xx,
       ROUND(SUM(CASE WHEN elb_status_code >= 500 THEN 1.0 ELSE 0 END) / COUNT(*) * 100, 2) as error_rate_pct
FROM alb_logs
GROUP BY 1
ORDER BY 1;
```

---

## ✅ Checklist Nhanh

### 503 Service Unavailable

```
□ Target group có healthy targets không?
□ Health check path trả về HTTP 200?
□ Security Group của target ALLOW từ Security Group ALB?
□ Target đang running và app đang chạy?
□ Target group có target nào được đăng ký không?
□ Auto Scaling Group gán đúng target group?
```

### Health Check Fail

```
□ Health check path có đúng không? (mặc định là /)
□ Health check port có đúng không?
□ Health check protocol HTTP hay HTTPS?
□ App trả về 200 cho health check path?
□ Security Group ALLOW ALB health checker?
□ App mới launch có đủ thời gian warm up?
```

### 502 Bad Gateway

```
□ App logs có lỗi gì không?
□ App có timeout không? (vượt ALB idle timeout 60s)
□ ALB idle timeout có đủ cao không?
□ App không crash khi nhận request?
□ App memory/CPU có đủ không?
```

### SSL Issues

```
□ Certificate trạng thái ISSUED?
□ Domain khớp với certificate SAN/CN?
□ Certificate chưa expired?
□ HTTPS listener có certificate được gán?
□ SNI enabled cho multiple certificates?
```

---

## 📚 Tài Liệu Liên Quan

- [../03-load-balancing/1-alb.md](../03-load-balancing/1-alb.md) — ALB chi tiết
- [../03-load-balancing/3-target-groups.md](../03-load-balancing/3-target-groups.md) — Target Groups & Health Checks
- [../03-load-balancing/4-ssl-tls.md](../03-load-balancing/4-ssl-tls.md) — SSL/TLS & ACM
- [../08-monitoring/2-cloudwatch-networking.md](../08-monitoring/2-cloudwatch-networking.md) — CloudWatch Metrics

---

**Cập Nhật Lần Cuối:** 2026-05-14
