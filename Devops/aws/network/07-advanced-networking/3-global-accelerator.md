# AWS Global Accelerator — Tăng Tốc Toàn Cầu

> **Thuộc topic:** 07-advanced-networking | **Mức độ:** Nâng cao

---

## 1. Global Accelerator Là Gì?

**AWS Global Accelerator** là dịch vụ mạng sử dụng **AWS global network** (mạng nội bộ toàn cầu của AWS, không phải internet công cộng) để tối ưu hóa đường đi của traffic từ người dùng cuối đến ứng dụng của bạn, giảm latency và tăng độ tin cậy.

### Vấn đề với internet thông thường

Khi người dùng ở Singapore truy cập ứng dụng ở US-East:

```
User (Singapore)
  → ISP Singapore
  → Nhiều hops qua internet
  → Nhiều BGP routers và peering points
  → Nhiều lần packet loss tiềm năng
  → US-East Application Server
  
Latency: ~180-250ms, không ổn định
```

### Giải pháp của Global Accelerator

```
User (Singapore)
  → ISP Singapore
  → AWS Edge Location Singapore (gần nhất)    ← Bước nhảy duy nhất qua internet
  → AWS Global Network (backbone riêng tư)    ← Đường cao tốc nội bộ AWS
  → US-East Application Server
  
Latency: ~150-180ms, ổn định hơn nhiều
```

**Lợi ích chính:**
- Người dùng chỉ đi qua **1 hop** trên internet công cộng (đến Edge Location gần nhất)
- Phần còn lại đi qua **AWS backbone** — ổn định, ít packet loss, được optimize liên tục
- **2 static Anycast IP** — không thay đổi dù backend thay đổi
- **Automatic failover** trong vòng < 30 giây — không phụ thuộc DNS TTL

---

## 2. Anycast vs Unicast Routing

### Unicast (Đơn Truyền) — cách thông thường

```
IP 203.0.113.10 → chỉ thuộc về 1 server tại 1 vị trí địa lý cụ thể

User Tokyo    → (routing qua internet) → Server ở US-East: 203.0.113.10
User London   → (routing qua internet) → Server ở US-East: 203.0.113.10
User Mumbai   → (routing qua internet) → Server ở US-East: 203.0.113.10
```

### Anycast (Đa Truyền Đến Gần Nhất) — cách Global Accelerator

```
2 IP tĩnh: 75.2.100.50 và 99.83.200.100
→ Được advertise từ TẤT CẢ AWS Edge Locations trên toàn cầu đồng thời
→ Router của ISP tự động route đến Edge Location gần nhất theo BGP

User Tokyo     → Edge Location Tokyo   → AWS Backbone → US-East
User London    → Edge Location London  → AWS Backbone → US-East
User Mumbai    → Edge Location Mumbai  → AWS Backbone → US-East
```

**Cùng một IP address, khác nhau routing path tùy vị trí địa lý của user.**

```
                    AWS Anycast IP: 75.2.100.50
                           ┌──────────────────┐
                           │ AWS Edge Locations│
                           │                  │
Tokyo User ─────────────►  │ PoP Tokyo        │
London User ─────────────► │ PoP London       │ ──► AWS Backbone ──► Application
Mumbai User ─────────────► │ PoP Mumbai       │
São Paulo User ──────────► │ PoP São Paulo    │
                           └──────────────────┘
                           (Tất cả cùng IP, gần user nhất được chọn)
```

---

## 3. Các Thành Phần (Components) của Global Accelerator

### 3.1. Accelerator

Là object cấp cao nhất. Khi tạo Accelerator, bạn nhận được:
- **2 static Anycast IPv4 addresses** — không bao giờ thay đổi
- Các địa chỉ này được advertise từ AWS Edge Locations toàn cầu

```bash
# Tạo Accelerator
aws globalaccelerator create-accelerator \
  --name "my-global-app" \
  --ip-address-type IPV4 \
  --enabled \
  --region us-west-2   # Global Accelerator API luôn ở us-west-2

# Output:
# {
#   "Accelerator": {
#     "AcceleratorArn": "arn:aws:globalaccelerator::123456789012:accelerator/xxxxx",
#     "Name": "my-global-app",
#     "IpSets": [
#       {
#         "IpFamily": "IPv4",
#         "IpAddresses": ["75.2.100.50", "99.83.200.100"]   ← 2 Anycast IPs
#       }
#     ],
#     "Status": "DEPLOYED"
#   }
# }
```

### 3.2. Listener — Trình Lắng Nghe

Listener định nghĩa **port và giao thức** mà Accelerator lắng nghe.

```bash
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/xxxxx \
  --protocol TCP \
  --port-ranges FromPort=80,ToPort=80 FromPort=443,ToPort=443 \
  --client-affinity SOURCE_IP   # Giữ session của cùng user về cùng endpoint
  --region us-west-2
```

**Client Affinity (Tương Đồng Client):**
- `NONE` (mặc định): Mỗi request có thể đến endpoint khác nhau — stateless
- `SOURCE_IP`: Cùng IP source luôn đến cùng endpoint — stateful (có session)

### 3.3. Endpoint Group — Nhóm Điểm Cuối

Mỗi Endpoint Group gắn với **một AWS Region**. Listener có thể có nhiều Endpoint Groups (mỗi group cho một region).

```bash
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::123456789012:listener/xxxxx \
  --endpoint-group-region ap-southeast-1 \
  --traffic-dial-percentage 100 \
  --health-check-protocol TCP \
  --health-check-port 80 \
  --health-check-interval-seconds 10 \
  --threshold-count 3 \
  --region us-west-2
```

**Traffic Dial (Điều Chỉnh Lưu Lượng):**
- Giá trị từ 0% đến 100%
- Cho phép điều chỉnh tỷ lệ traffic vào từng region
- Ứng dụng: Blue/Green deployment, gradual rollout, disaster recovery

```
Endpoint Group Singapore: traffic-dial = 80%  → nhận 80% traffic
Endpoint Group Tokyo:     traffic-dial = 20%  → nhận 20% traffic
```

### 3.4. Endpoint — Điểm Cuối

**Supported endpoint types (Loại điểm cuối được hỗ trợ):**

| Endpoint Type | Mô tả |
|--------------|-------|
| **Application Load Balancer (ALB)** | HTTP/HTTPS load balancer |
| **Network Load Balancer (NLB)** | TCP/UDP load balancer |
| **EC2 Instance** | Gắn trực tiếp vào EC2 |
| **Elastic IP Address** | IP tĩnh |

```bash
# Thêm ALB làm endpoint
aws globalaccelerator add-endpoints \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:endpointgroup/xxxxx \
  --endpoint-configurations \
    EndpointId=arn:aws:elasticloadbalancing:ap-southeast-1:123456789012:loadbalancer/app/my-alb/xxxxx,Weight=128,ClientIPPreservationEnabled=true \
  --region us-west-2
```

**Endpoint Weights (Trọng số Điểm Cuối):**
- Mỗi endpoint trong group có weight từ 0-255
- Traffic phân phối proportional theo weight
- Weight = 0: endpoint không nhận traffic (nhưng vẫn health check)

---

## 4. Health Checks và Automatic Failover

### Health Check Configuration

```
Global Accelerator
  → Health Check mỗi 10 hoặc 30 giây đến mỗi endpoint
  → Nếu endpoint fail health check consecutively (threshold: 1-10 lần)
  → Tự động chuyển traffic sang endpoint khỏe mạnh
```

**Đặc điểm nổi bật:** Failover **không phụ thuộc DNS TTL**!

Với Route 53 failover, cần:
1. Health check phát hiện failure
2. Route 53 update DNS record
3. Chờ TTL hết hạn (thường 60-300 giây)
4. Client làm mới DNS cache

Với Global Accelerator:
1. Health check phát hiện failure
2. Global Accelerator ngay lập tức redirect traffic trong AWS backbone
3. Không cần DNS update, không phụ thuộc client cache
4. **Failover trong < 30 giây**

```bash
# Xem health status của endpoints
aws globalaccelerator describe-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:endpointgroup/xxxxx \
  --region us-west-2 \
  --query 'EndpointGroup.EndpointDescriptions[*].{ID:EndpointId,Health:HealthState,Reason:HealthReason}'
```

### Failover Scenarios

```
Bình thường:
  Traffic → AP-Southeast-1 (Primary) [HEALTHY]
  
Khi primary fail:
  Traffic → AP-Southeast-1 (Primary) [UNHEALTHY]
           ↓ Automatic failover < 30 giây
  Traffic → US-East-1 (Secondary) [HEALTHY]
  
Khi primary recover:
  Traffic → AP-Southeast-1 (Primary) [HEALTHY] ← Tự động quay lại
```

---

## 5. Traffic Dials và Use Cases Nâng Cao

### Blue/Green Deployment Toàn Cầu

```bash
# Bước 1: Deploy version mới ở region thứ hai (Green)
# Green endpoint group: ap-southeast-1 (new version)
# Blue endpoint group:  us-east-1 (old version)

# Bước 2: Chuyển dần traffic
# Lần 1: Green 10%, Blue 90%
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:endpointgroup/green \
  --traffic-dial-percentage 10 --region us-west-2

# Lần 2: Green 50%, Blue 50%
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:endpointgroup/green \
  --traffic-dial-percentage 50 --region us-west-2

# Lần 3: Green 100%, Blue 0%
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:endpointgroup/green \
  --traffic-dial-percentage 100 --region us-west-2

aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::123456789012:endpointgroup/blue \
  --traffic-dial-percentage 0 --region us-west-2
```

### Disaster Recovery — Disaster Recovery (Khôi Phục Thảm Họa)

```
Bình thường:
  AP-Southeast-1: traffic-dial = 100%  (primary)
  US-East-1:      traffic-dial = 0%    (standby, vẫn health check)

Khi AP-Southeast-1 gặp sự cố:
  Cách 1 (Manual): Tăng US-East-1 lên 100%, giảm AP về 0
  Cách 2 (Auto):   Health check phát hiện tự động failover
```

---

## 6. So Sánh: Global Accelerator vs CloudFront

Đây là câu hỏi phỏng vấn cực kỳ phổ biến. Hiểu rõ sự khác biệt là bắt buộc.

| Khía cạnh | Global Accelerator | CloudFront |
|-----------|-------------------|------------|
| **Layer** | L3/L4 (Network/Transport) | L7 (Application) |
| **Protocol** | TCP, UDP | HTTP, HTTPS (WebSocket) |
| **Caching** | Không | Có — cache tại Edge |
| **Static content** | Không optimize | Tối ưu, giảm tải origin |
| **Dynamic content** | Có (TCP acceleration) | Có nhưng vẫn phải về origin |
| **IP addresses** | 2 static Anycast IP | DNS (thay đổi theo thời gian) |
| **Failover speed** | < 30 giây (không qua DNS) | Phụ thuộc DNS TTL (60-300s) |
| **Gaming / VoIP** | Hỗ trợ (UDP, TCP) | Không phù hợp |
| **Lambda@Edge** | Không | Có |
| **WAF integration** | Không trực tiếp | Có |
| **Chi phí** | $0.025/giờ + $0.015/10GB | Theo GB + request |
| **Use case** | Real-time, multi-region HA | Static/dynamic web content |

### Khi nào dùng Global Accelerator:
- Game servers, VoIP, real-time communication (UDP/TCP)
- Ứng dụng cần **static IP** (whitelist IP ở firewall doanh nghiệp)
- Cần failover **không phụ thuộc DNS TTL**
- Non-HTTP workloads
- Multi-region active-active hoặc active-passive deployment

### Khi nào dùng CloudFront:
- Website, API với nội dung có thể cache
- Tích hợp WAF (Web Application Firewall)
- Cần Lambda@Edge để customize response
- CDN thuần túy cho static assets (images, JS, CSS)

### Có thể dùng cả hai cùng lúc không?

**Có.** Một pattern phổ biến:

```
User → CloudFront (cache static content)
     → Global Accelerator (accelerate dynamic/API requests) → ALB → App
```

---

## 7. Client IP Preservation — Bảo Toàn IP Client

Khi dùng Global Accelerator với ALB endpoint:

```bash
# Bật Client IP Preservation
aws globalaccelerator add-endpoints \
  --endpoint-configurations \
    EndpointId=arn:...:loadbalancer/app/my-alb/xxx,Weight=128,ClientIPPreservationEnabled=true
```

**Với Client IP Preservation bật:**
- ALB/ứng dụng nhìn thấy **IP thật của client**
- Hỗ trợ: ALB (mặc định bật), EC2, Elastic IP
- Không hỗ trợ: NLB (NLB luôn thấy IP của Edge Location)

**Với Client IP Preservation tắt:**
- Ứng dụng nhìn thấy IP của AWS Edge Location
- Không biết được nguồn gốc thật của request

---

## 8. Global Accelerator với IPv6

Từ 2023, Global Accelerator hỗ trợ **dual-stack** (IPv4 + IPv6):

```bash
aws globalaccelerator create-accelerator \
  --name "dual-stack-accelerator" \
  --ip-address-type DUAL_STACK \   # IPV4 hoặc DUAL_STACK
  --enabled \
  --region us-west-2
```

**Dual-stack Accelerator:**
- Nhận 2 IPv4 Anycast IPs (như thường)
- Nhận thêm 1 IPv6 Anycast IP
- ALB endpoint hỗ trợ dual-stack natively

---

## 9. Pricing — Mô Hình Tính Phí

```
Chi phí Global Accelerator = Fixed fee + Data transfer fee

1. Accelerator fee (phí cố định):
   $0.025/giờ/accelerator
   = ~$18/tháng/accelerator

2. Data Transfer Premium (phí chênh lệch so với internet transfer):
   Tùy vào source và destination region
   Ví dụ US → US: $0.015/10GB (~$0.0015/GB)
   Ví dụ Asia → US: $0.035/10GB (~$0.0035/GB)

3. Data Transfer (standard AWS transfer rates):
   Tương tự EC2 data transfer ra internet thông thường
```

**So sánh chi phí:**

```
Scenario: 100GB/tháng data, 1 accelerator, US region

Global Accelerator:
  - Accelerator: $18/tháng
  - Data Premium (US→US): 100GB × $0.0015 = $0.15
  - Data Transfer: ~$9 (standard rate)
  - Tổng: ~$27.15/tháng

Route 53 Latency Routing (thay thế đơn giản hơn):
  - $0.40/1M DNS queries
  - Data Transfer: ~$9
  - Tổng: ~$9.40/tháng + không có AWS backbone optimization

→ Global Accelerator tốn thêm ~$18/tháng nhưng có backbone acceleration + failover nhanh
```

---

## 10. Terraform Example

```hcl
# ========================================
# Global Accelerator đầy đủ
# ========================================

# Accelerator
resource "aws_globalaccelerator_accelerator" "main" {
  name            = "my-global-app"
  ip_address_type = "IPV4"
  enabled         = true

  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = aws_s3_bucket.accelerator_logs.bucket
    flow_logs_s3_prefix = "global-accelerator/"
  }
}

# Listener cho HTTP và HTTPS
resource "aws_globalaccelerator_listener" "http" {
  accelerator_arn = aws_globalaccelerator_accelerator.main.id
  protocol        = "TCP"
  client_affinity = "SOURCE_IP"

  port_range {
    from_port = 80
    to_port   = 80
  }

  port_range {
    from_port = 443
    to_port   = 443
  }
}

# Endpoint Group — Singapore (Primary)
resource "aws_globalaccelerator_endpoint_group" "singapore" {
  listener_arn          = aws_globalaccelerator_listener.http.id
  endpoint_group_region = "ap-southeast-1"
  traffic_dial_percentage = 100   # Nhận 100% traffic bình thường

  health_check_protocol        = "TCP"
  health_check_port            = 443
  health_check_interval_seconds = 10
  threshold_count              = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.singapore_alb.arn
    weight                         = 128
    client_ip_preservation_enabled = true
  }
}

# Endpoint Group — US-East (Secondary/Standby)
resource "aws_globalaccelerator_endpoint_group" "us_east" {
  listener_arn          = aws_globalaccelerator_listener.http.id
  endpoint_group_region = "us-east-1"
  traffic_dial_percentage = 0   # Standby — chỉ nhận traffic khi Singapore fail

  health_check_protocol        = "TCP"
  health_check_port            = 443
  health_check_interval_seconds = 10
  threshold_count              = 3

  endpoint_configuration {
    endpoint_id                    = var.us_east_alb_arn
    weight                         = 128
    client_ip_preservation_enabled = true
  }
}

# S3 bucket cho flow logs
resource "aws_s3_bucket" "accelerator_logs" {
  bucket = "my-app-global-accelerator-logs"
}

resource "aws_s3_bucket_policy" "accelerator_logs" {
  bucket = aws_s3_bucket.accelerator_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::434961661072:root"  # Global Accelerator account
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.accelerator_logs.arn}/*"
      }
    ]
  })
}

# Outputs
output "accelerator_ips" {
  value       = aws_globalaccelerator_accelerator.main.ip_sets
  description = "Anycast IPs - share these with users/whitelist in firewalls"
}

output "accelerator_dns" {
  value = aws_globalaccelerator_accelerator.main.dns_name
  description = "DNS name for the accelerator"
}
```

---

## 11. Flow Logs và Monitoring

```bash
# Global Accelerator Flow Logs — ghi lại chi tiết connection
# Bật qua Accelerator attributes:
aws globalaccelerator update-accelerator-attributes \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/xxxxx \
  --attributes FlowLogsEnabled=true,FlowLogsS3Bucket=my-logs-bucket,FlowLogsS3Prefix=ga-logs/ \
  --region us-west-2

# CloudWatch Metrics quan trọng:
# - NewFlowCount: số flow mới trong khoảng thời gian
# - ProcessedBytesOut: bytes gửi đến endpoint
# - ProcessedBytesIn: bytes nhận từ endpoint

# Xem metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/GlobalAccelerator \
  --metric-name NewFlowCount \
  --dimensions Name=Accelerator,Value=xxxxx \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics Sum \
  --region us-west-2
```

---

## 12. Giới Hạn (Limits)

| Resource | Giới hạn mặc định |
|---------|------------------|
| Accelerators per account | 20 |
| Listeners per Accelerator | 10 |
| Endpoint Groups per Listener | 10 (1 per region) |
| Endpoints per Endpoint Group | 10 |
| Port ranges per Listener | 10 |

---

## Câu Hỏi Phỏng Vấn Thường Gặp

### Q1: Tại sao Global Accelerator có thể failover nhanh hơn Route 53?

**Trả lời:**
Route 53 failover dựa vào **DNS record update** + chờ **TTL** của các cached DNS record hết hạn ở clients và intermediate DNS servers. Dù Route 53 update ngay lập tức, client vẫn có thể dùng IP cũ trong vòng TTL (60-300 giây hoặc hơn nếu client override TTL).

Global Accelerator không dùng DNS để route traffic đến endpoint. Nó route ở **network layer** trong AWS backbone. Khi endpoint fail health check, Global Accelerator cập nhật routing table ngay tức thì trong hệ thống nội bộ AWS. Không có DNS TTL nào phải chờ. Kết quả: failover trong vòng **dưới 30 giây**.

---

### Q2: Anycast hoạt động như thế nào? Tại sao cùng một IP mà có thể route đến nhiều location?

**Trả lời:**
Anycast là kỹ thuật mạng trong đó cùng một IP address được **advertise bởi nhiều node** thông qua BGP (Border Gateway Protocol — Giao Thức Định Tuyến Biên). Khi ISP của user nhận được route cho IP đó từ nhiều nguồn, router sẽ chọn **đường ngắn nhất** (theo BGP metrics) — thường là Edge Location gần nhất về mặt địa lý.

AWS advertise 2 Anycast IPs từ tất cả ~90+ Edge Locations. User ở Tokyo kết nối đến Edge Location Tokyo vì BGP route từ Tokyo data center là ngắn nhất. User ở London kết nối đến Edge Location London — cùng IP nhưng router khác nhau được chọn.

---

### Q3: Khi nào nên chọn Global Accelerator thay vì CloudFront?

**Trả lời:**
Dùng **Global Accelerator** khi:
- Ứng dụng dùng giao thức không phải HTTP (game server dùng UDP, VoIP, streaming dùng TCP raw)
- Cần **địa chỉ IP tĩnh** để whitelist trong enterprise firewall của khách hàng
- Cần failover **không phụ thuộc DNS TTL** (financial systems, healthcare)
- Multi-region active-active với traffic dial control

Dùng **CloudFront** khi:
- Web/API với nội dung cacheable (giảm tải origin rõ ràng)
- Cần tích hợp WAF để bảo vệ khỏi SQL injection, XSS, DDoS L7
- Cần Lambda@Edge để customize response tại edge
- Chi phí là ưu tiên (CloudFront rẻ hơn cho web workloads)

Có thể kết hợp cả hai: CloudFront cho HTTP traffic cacheable, Global Accelerator cho API/real-time.

---

### Q4: Traffic Dial percentage là gì? Dùng để làm gì?

**Trả lời:**
Traffic Dial là percentage (0-100%) kiểm soát bao nhiêu traffic được gửi đến một Endpoint Group (region). Mặc định là 100%.

Ứng dụng quan trọng:
1. **Blue/Green deployment:** Tăng dần traffic-dial của region mới từ 0% → 10% → 50% → 100% trong khi giảm region cũ, không cần thay đổi DNS.
2. **Disaster recovery:** Set region secondary về 0% bình thường (vẫn health check), khi primary fail tăng lên 100% trong vài giây.
3. **Load testing regional:** Set một region lên 100% để tập trung load test vào đó.
4. **Weighted distribution:** Set các region khác nhau (ví dụ 70/30) để phân tải theo địa lý.

---

### Q5: Global Accelerator có bảo vệ DDoS không?

**Trả lời:**
**Có**, theo hai lớp:
1. **AWS Shield Standard** (Tiêu Chuẩn): Tự động được bật cho tất cả AWS resources bao gồm Global Accelerator. Bảo vệ chống L3/L4 DDoS (SYN floods, UDP reflection).
2. **AWS Shield Advanced** (Nâng Cao): Nếu enable thêm, có DDoS protection chuyên sâu, DDoS cost protection, và hỗ trợ từ AWS DDoS Response Team (DRT).

Ngoài ra, vì Anycast IPs được phân tán qua nhiều Edge Locations, một DDoS attack nhằm vào IP sẽ bị phân tán qua nhiều PoPs thay vì tập trung vào một điểm — giúp absorb attack traffic tốt hơn.

---

### Q6: Endpoint Weight khác gì Traffic Dial?

**Trả lời:**
- **Traffic Dial** (0-100%): Kiểm soát bao nhiêu **tổng traffic** được gửi đến một **Endpoint Group** (region). Hoạt động ở cấp region.

- **Endpoint Weight** (0-255): Kiểm soát phân phối traffic **trong một Endpoint Group** giữa nhiều endpoints. Ví dụ: nếu Group có 2 endpoints với weight 100 và 50, endpoint đầu nhận 2/3 traffic, endpoint sau nhận 1/3.

Kết hợp: Traffic Dial quyết định bao nhiêu đến region, Weight quyết định phân phối trong region.

```
Ví dụ:
Region Singapore (Traffic Dial = 80%):
  - ALB-1 (Weight 200) → nhận 80% × 200/300 = 53% tổng traffic
  - ALB-2 (Weight 100) → nhận 80% × 100/300 = 27% tổng traffic
Region Tokyo (Traffic Dial = 20%):
  - ALB-3 (Weight 128) → nhận 20% tổng traffic
```

---

## Điều Hướng

- [← 2-privatelink.md](./2-privatelink.md) — AWS PrivateLink
- [→ 4-elastic-ip-eni.md](./4-elastic-ip-eni.md) — Elastic IP & ENI
- [5-ipv6.md](./5-ipv6.md) — IPv6 & Dual-Stack
- [1-vpc-endpoints.md](./1-vpc-endpoints.md) — VPC Endpoints
- [README.md](./README.md) — Tổng quan 07-advanced-networking
- [← 06-connectivity](../06-connectivity/) — VPN, Direct Connect, Transit Gateway
