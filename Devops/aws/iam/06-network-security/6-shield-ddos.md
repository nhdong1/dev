# AWS Shield — Bảo Vệ Chống DDoS

> AWS Shield bảo vệ ứng dụng AWS khỏi các cuộc tấn công DDoS (Distributed Denial of Service — Tấn Công Từ Chối Dịch Vụ Phân Tán), hoạt động tự động và liên tục 24/7.

## 📚 Mục Lục

1. [DDoS là gì?](#ddos-là-gì)
2. [Shield Standard vs Advanced](#shield-standard-vs-advanced)
3. [Kiến Trúc Bảo Vệ DDoS](#kiến-trúc-bảo-vệ-ddos)
4. [Shield Advanced — Tính Năng Chi Tiết](#shield-advanced--tính-năng-chi-tiết)
5. [Kết Hợp Shield với WAF](#kết-hợp-shield-với-waf)
6. [Best Practices](#best-practices)
7. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## DDoS là gì?

### Phân Loại Tấn Công DDoS

**Volumetric Attacks (Tấn Công Thể Tích) — Layer 3/4:**
- Mục tiêu: Làm bão hòa băng thông (bandwidth saturation)
- Ví dụ: UDP flood, ICMP flood, DNS amplification
- Quy mô: Có thể đạt Tbps (terabit per second)

**Protocol Attacks (Tấn Công Giao Thức) — Layer 3/4:**
- Mục tiêu: Làm cạn kiệt tài nguyên xử lý kết nối
- Ví dụ: SYN flood, Ping of Death, Smurf attack
- Đơn vị đo: PPS (Packets Per Second)

**Application Layer Attacks (Tấn Công Tầng Ứng Dụng) — Layer 7:**
- Mục tiêu: Làm cạn kiệt tài nguyên server bằng request hợp lệ
- Ví dụ: HTTP flood, Slowloris, DNS query flood
- Đơn vị đo: RPS (Requests Per Second)

---

## Shield Standard vs Advanced

| Tính Năng | Shield Standard | Shield Advanced |
|---|---|---|
| **Chi phí** | Miễn phí | $3,000/tháng/tổ chức |
| **Bảo vệ L3/L4** | ✅ Tự động | ✅ Nâng cao |
| **Bảo vệ L7** | ❌ | ✅ (kết hợp WAF) |
| **DRT access** (Đội Ứng Phó DDoS) | ❌ | ✅ 24/7 |
| **Cost protection** (Bảo Hiểm Chi Phí) | ❌ | ✅ |
| **Attack visibility** | Hạn chế | Toàn diện qua Dashboard |
| **Health-based detection** | ❌ | ✅ |
| **Proactive engagement** | ❌ | ✅ |

### Shield Standard

Shield Standard được bật **tự động và miễn phí** cho tất cả tài khoản AWS. Bảo vệ:
- Tất cả AWS regions
- Chống các cuộc tấn công L3/L4 phổ biến nhất
- SYN/UDP floods, reflection attacks
- Không cần cấu hình gì thêm

### Shield Advanced

Phù hợp với:
- Workload production quan trọng (banking, e-commerce, gaming)
- Cần bảo vệ Layer 7
- Cần SLA (Service Level Agreement — Thỏa Thuận Mức Dịch Vụ) và hỗ trợ 24/7
- Cần bảo hiểm chi phí khi bị tấn công lớn

---

## Kiến Trúc Bảo Vệ DDoS

### Defense Layers (Các Lớp Bảo Vệ)

```
Internet (Nguồn tấn công)
        │
        ▼
AWS Edge Network (CloudFront, Route 53)
  ← Shield Standard: Hấp thụ volumetric attacks tại edge
  ← Shield Advanced: Scrubbing centers lọc traffic sạch
        │
        ▼
AWS Backbone Network
  ← Shield Standard: Bảo vệ L3/L4 tự động
        │
        ▼
CloudFront Distribution
  ← WAF + Shield Advanced: L7 protection
        │
        ▼
ALB / NLB
  ← Shield Advanced: Health-based detection
        │
        ▼
EC2 / ECS Application
```

### Tài Nguyên Được Bảo Vệ bởi Shield Advanced

Shield Advanced bảo vệ:
- **CloudFront distributions**
- **Route 53 hosted zones**
- **Global Accelerator accelerators**
- **Elastic IP addresses** (gắn với EC2, NAT Gateway)
- **ALB** (Application Load Balancer) và **NLB** (Network Load Balancer)

---

## Shield Advanced — Tính Năng Chi Tiết

### 1. DRT — DDoS Response Team (Đội Ứng Phó DDoS)

DRT là đội chuyên gia của AWS sẵn sàng hỗ trợ 24/7 khi bị tấn công DDoS:

```bash
# Ủy quyền DRT truy cập vào account của bạn
aws shield associate-drt-role \
  --role-arn arn:aws:iam::account-id:role/AWSSHieldDRTAccessRole

# Chia sẻ S3 bucket logs với DRT
aws shield associate-drt-log-bucket \
  --log-bucket s3://my-waf-logs-bucket
```

### 2. Cost Protection (Bảo Hiểm Chi Phí)

Khi bị tấn công DDoS dẫn đến tăng chi phí AWS bất thường, Shield Advanced bảo hiểm:
- EC2 instance charges
- CloudFront transfer costs
- Route 53 query charges
- ELB charges

Cần submit request qua AWS Support sau sự kiện.

### 3. Health-Based Detection (Phát Hiện Dựa Trên Sức Khỏe)

Kết hợp Route 53 Health Check với Shield để tăng độ chính xác phát hiện tấn công:

```bash
# Tạo Route 53 health check
aws route53 create-health-check \
  --caller-reference unique-string \
  --health-check-config '{
    "IPAddress": "203.0.113.10",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'

# Liên kết health check với Shield protection
aws shield associate-health-check \
  --protection-id protection-id \
  --health-check-arn arn:aws:route53:::healthcheck/health-check-id
```

### 4. Proactive Engagement (Tương Tác Chủ Động)

Khi bật Proactive Engagement, DRT tự động liên hệ bạn khi phát hiện cuộc tấn công nghiêm trọng — không cần chờ bạn mở ticket.

```bash
aws shield enable-proactive-engagement
aws shield update-emergency-contact-settings \
  --emergency-contact-list '[
    {"EmailAddress": "security-team@company.com", "PhoneNumber": "+84901234567"},
    {"EmailAddress": "ops-team@company.com"}
  ]'
```

---

## Kết Hợp Shield với WAF

### Layer 7 DDoS Mitigation

Shield Advanced một mình không thể mitigate Layer 7 attacks. Phải kết hợp với WAF:

```
HTTP Flood Attack (1M req/s)
        │
        ▼
CloudFront ← WAF rate-based rules
  "Block IP nếu > 2000 req/5min"
        │ (traffic hợp lệ)
        ▼
Shield Advanced
  "Phát hiện pattern bất thường, notify DRT"
        │
        ▼
ALB → Application
```

### AWS Firewall Manager — Tự Động Hóa

Trong môi trường multi-account, dùng Firewall Manager để tự động bật Shield Advanced và gắn WAF cho tài nguyên mới:

```terraform
# Shield Advanced Protection Group
resource "aws_shield_protection_group" "all_cfn" {
  protection_group_id = "all-cloudfront-protections"
  aggregation         = "MAX"
  pattern             = "BY_RESOURCE_TYPE"
  resource_type       = "CLOUDFRONT_DISTRIBUTION"
}

# Shield Protection cho specific resource
resource "aws_shield_protection" "alb" {
  name         = "prod-alb-shield"
  resource_arn = aws_lb.production.arn

  tags = { Environment = "production" }
}
```

---

## Monitoring DDoS Events

### CloudWatch Metrics của Shield

```bash
# Xem DDoS attack metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/DDoSProtection \
  --metric-name DDoSAttackBitsPerSecond \
  --dimensions Name=ResourceArn,Value=arn:aws:cloudfront::...:distribution/E1XXXXX \
  --start-time 2026-05-16T00:00:00Z \
  --end-time 2026-05-16T23:59:59Z \
  --period 300 \
  --statistics Maximum
```

**Metrics quan trọng:**

| Metric | Ý Nghĩa |
|---|---|
| `DDoSDetected` | 1 = đang bị tấn công, 0 = bình thường |
| `DDoSAttackBitsPerSecond` | Băng thông tấn công |
| `DDoSAttackPacketsPerSecond` | Số packet/giây |
| `DDoSAttackRequestsPerSecond` | Số request/giây (L7) |

### Alarm Khi Bị Tấn Công

```terraform
resource "aws_cloudwatch_metric_alarm" "ddos_detected" {
  alarm_name          = "DDoS-Attack-Detected"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "DDoSDetected"
  namespace           = "AWS/DDoSProtection"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "DDoS attack detected by Shield"

  dimensions = {
    ResourceArn = aws_lb.production.arn
  }

  alarm_actions = [aws_sns_topic.security_alerts.arn]
}
```

---

## Best Practices

1. **Bật Shield Standard miễn phí** — không cần làm gì thêm, đã tự động bật
2. **Dùng CloudFront cho tất cả ứng dụng web** — CloudFront là tuyến phòng thủ đầu tiên tại edge
3. **Shield Advanced cho production critical** — workload banking, gaming, e-commerce
4. **Luôn kết hợp WAF với Shield** — Shield không bảo vệ L7 nếu không có WAF
5. **Cấu hình Proactive Engagement** — để DRT hỗ trợ ngay khi phát hiện tấn công
6. **Test định kỳ với AWS FireLens** — mô phỏng tấn công DDoS trong môi trường test
7. **Tạo Incident Runbook** — quy trình ứng phó khi nhận cảnh báo DDoS

---

## Câu Hỏi Phỏng Vấn

### Q: Shield Standard và Shield Advanced khác nhau như thế nào?

**Trả lời:**
- **Shield Standard:** Miễn phí, tự động cho tất cả account; bảo vệ L3/L4 cơ bản (SYN flood, UDP flood, reflection attacks); không có giao diện quản lý hay hỗ trợ chuyên biệt
- **Shield Advanced:** $3,000/tháng; bảo vệ L7 (kết hợp WAF); truy cập DRT 24/7; cost protection khi bị tấn công; visibility dashboard; health-based detection; proactive engagement

Với workload nhỏ, Shield Standard thường đủ. Production quan trọng cần Shield Advanced.

### Q: Tại sao CloudFront giúp giảm thiểu DDoS?

**Trả lời:** CloudFront là CDN toàn cầu với hàng trăm edge locations (điểm biên). Khi bị tấn công:
1. Traffic được phân tán qua nhiều edge location, không tập trung về origin
2. AWS absorbs volumetric attacks tại edge — attacker phải tấn công toàn bộ AWS edge network
3. CloudFront che giấu IP của origin server — attacker không thể bypass CloudFront để tấn công thẳng
4. Kết hợp với WAF ở CloudFront để chặn L7 attacks ngay tại edge

### Q: Shield có tự động respond khi bị DDoS không hay cần can thiệp thủ công?

**Trả lời:**
- **Shield Standard:** Hoàn toàn tự động — AWS tự phát hiện và mitigate L3/L4 attacks mà không cần bạn làm gì
- **Shield Advanced:** Tự động mitigate attacks; thêm visibility và notifications; với **Proactive Engagement**, DRT tự động liên hệ bạn; bạn có thể mở case với DRT để nhận hỗ trợ chuyên sâu trong quá trình tấn công

---

## 🔗 Xem Thêm

- [5-waf-setup.md](5-waf-setup.md) — WAF cho Layer 7 protection
- [7-network-firewall.md](7-network-firewall.md) — Network Firewall cho deep inspection
- [README.md](README.md) — Tổng quan Network Security

---

**Cập Nhật:** 2026-05-16 | **Phiên Bản:** 1.0
