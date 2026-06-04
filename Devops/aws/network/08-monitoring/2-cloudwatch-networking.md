# CloudWatch Networking — Metrics & Alarms Cho Mạng AWS

> Amazon CloudWatch (Đồng Hồ Đám Mây) là trung tâm giám sát của AWS. Đối với networking, CloudWatch cung cấp metrics (số liệu), alarms (cảnh báo), và dashboards (bảng điều khiển) cho mọi dịch vụ mạng — từ ALB, NLB, CloudFront đến VPN và Direct Connect.

---

## 1. Tổng Quan Kiến Trúc CloudWatch Cho Networking

```
Dịch vụ Mạng AWS        CloudWatch              Hành Động
─────────────────        ─────────────           ────────────────────
ALB / NLB           →   Metrics             →   Dashboard (trực quan)
CloudFront          →   Alarms              →   SNS Notification
VPN Tunnel          →   Logs Insights       →   Auto Scaling trigger
Direct Connect      →   Contributor Insights →   Lambda function
Transit Gateway     →   Dashboards          →   Incident ticket
NAT Gateway         →                           PagerDuty / OpsGenie
```

---

## 2. ALB — Application Load Balancer Metrics

ALB (Application Load Balancer — Cân Bằng Tải Ứng Dụng) phát ra metrics mỗi 1 phút.

### Metrics Quan Trọng Nhất

#### Metrics Về Lưu Lượng (Traffic)

| Metric                  | Mô Tả                                          | Ngưỡng Cảnh Báo Gợi Ý  |
|-------------------------|------------------------------------------------|-------------------------|
| `RequestCount`          | Tổng số request nhận được                      | Đột tăng > 200%         |
| `ActiveConnectionCount` | Kết nối TCP đang hoạt động đồng thời           | > 10,000 (tùy workload) |
| `NewConnectionCount`    | Kết nối mới mỗi phút                          | Đột tăng bất thường     |
| `ProcessedBytes`        | Tổng bytes ALB xử lý (inbound + outbound)      | Monitor xu hướng        |

#### Metrics Về Hiệu Suất (Performance)

| Metric                     | Mô Tả                                                        | Ngưỡng Cảnh Báo Gợi Ý |
|----------------------------|--------------------------------------------------------------|------------------------|
| `TargetResponseTime`       | Thời gian targets phản hồi (giây) — P50, P90, P99           | P99 > 3 giây           |
| `RequestCountPerTarget`    | Request mỗi target trong target group                        | Bất đồng đều → skew   |

#### Metrics Về Lỗi (Errors)

| Metric                   | Mô Tả                                                           | Ngưỡng Cảnh Báo    |
|--------------------------|------------------------------------------------------------------|---------------------|
| `HTTPCode_ELB_4XX_Count` | 4xx errors do ALB (bad request, not found)                     | > 1% của requests   |
| `HTTPCode_ELB_5XX_Count` | 5xx errors do ALB (ALB không thể route request)                | > 0.1% → alert      |
| `HTTPCode_Target_5XX_Count` | 5xx errors từ targets/backends                             | > 1% → alert        |
| `RejectedConnectionCount`| Request bị từ chối vì ALB đạt giới hạn connection             | > 0 → alert ngay    |
| `TargetConnectionErrorCount` | Lỗi kết nối từ ALB đến targets                          | > 0 → alert         |

#### Metrics Về Health (Sức Khỏe)

| Metric                          | Mô Tả                                         |
|---------------------------------|------------------------------------------------|
| `HealthyHostCount`              | Số targets đang healthy trong target group     |
| `UnHealthyHostCount`            | Số targets đang unhealthy                      |

### CloudWatch Alarm cho ALB — Mẫu Thực Tế

```bash
# Alarm khi 5xx error rate > 1% trong 5 phút liên tiếp
aws cloudwatch put-metric-alarm \
  --alarm-name "ALB-High-5XX-Rate" \
  --alarm-description "ALB 5XX error rate exceeded 1%" \
  --metric-name HTTPCode_ELB_5XX_Count \
  --namespace AWS/ApplicationELB \
  --dimensions Name=LoadBalancer,Value=app/my-alb/50dc6c495c0c9188 \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts \
  --treat-missing-data notBreaching
```

---

## 3. NLB — Network Load Balancer Metrics

NLB (Network Load Balancer — Cân Bằng Tải Mạng) hoạt động tại Layer 4, có metric set khác ALB.

### Metrics Quan Trọng

| Metric                         | Mô Tả                                                    | Ghi Chú                   |
|--------------------------------|----------------------------------------------------------|---------------------------|
| `ActiveFlowCount`              | Số TCP flows đang hoạt động đồng thời                    | Thể hiện concurrent load  |
| `NewFlowCount`                 | Số TCP flows mới mỗi phút                               |                           |
| `ProcessedBytes`               | Bytes được xử lý (cả hai chiều)                         |                           |
| `TCP_Client_Reset_Count`       | Số RST packets từ client                                | Cao → client disconnect   |
| `TCP_Target_Reset_Count`       | Số RST packets từ target                                | Cao → app crash/restart   |
| `TCP_ELB_Reset_Count`          | Số RST packets do NLB tạo ra                            | Cao → NLB troubleshoot    |
| `HealthyHostCount`             | Targets healthy                                          |                           |
| `UnHealthyHostCount`           | Targets unhealthy                                        | > 0 → alert               |
| `ConsumedLCUs`                 | Load Balancer Capacity Units (đơn vị công suất) đã dùng | Tính chi phí              |

---

## 4. CloudFront Metrics

CloudFront (Mạng Phân Phối Nội Dung) phát metrics từ **edge locations** (điểm biên) toàn cầu.

### Metrics Mặc Định (Miễn Phí)

| Metric            | Mô Tả                                                         | Ngưỡng Gợi Ý            |
|-------------------|---------------------------------------------------------------|--------------------------|
| `Requests`        | Tổng số HTTP requests đến CloudFront                          | Monitor xu hướng         |
| `BytesDownloaded` | Bytes CloudFront gửi về viewers                              | Tính chi phí bandwidth   |
| `BytesUploaded`   | Bytes viewers gửi lên CloudFront                             |                          |
| `4xxErrorRate`    | Tỷ lệ % responses 4xx (client errors)                        | > 5% → kiểm tra          |
| `5xxErrorRate`    | Tỷ lệ % responses 5xx (server/origin errors)                 | > 1% → alert             |
| `TotalErrorRate`  | Tổng tỷ lệ lỗi (4xx + 5xx)                                  |                          |

### Additional Metrics (Tính Phí — $0.01/metric/month)

| Metric               | Mô Tả                                                           |
|----------------------|-----------------------------------------------------------------|
| `CacheHitRate`       | Tỷ lệ % requests được serve từ cache (không hit origin)        |
| `OriginLatency`      | Thời gian round-trip từ CloudFront đến origin                  |
| `401ErrorRate`       | Tỷ lệ 401 (Unauthorized — Không Được Phép) cụ thể             |
| `403ErrorRate`       | Tỷ lệ 403 (Forbidden — Bị Cấm) cụ thể                         |
| `404ErrorRate`       | Tỷ lệ 404 (Not Found — Không Tìm Thấy) cụ thể                  |
| `502ErrorRate`       | Tỷ lệ 502 (Bad Gateway — Cổng Lỗi) cụ thể                      |

### CloudFront Dashboard Mẫu

```
Widget 1: CacheHitRate — mục tiêu > 80%
Widget 2: RequestCount by region — phân bố địa lý
Widget 3: 5xxErrorRate — alert khi > 1%
Widget 4: OriginLatency — P99 < 500ms
Widget 5: BytesDownloaded — theo dõi chi phí bandwidth
```

---

## 5. VPN Metrics

### Site-to-Site VPN (VPN Địa Điểm-đến-Địa Điểm)

| Metric                | Mô Tả                                                  | Ngưỡng Alert        |
|-----------------------|--------------------------------------------------------|---------------------|
| `TunnelState`         | 1 = UP (hoạt động), 0 = DOWN (ngừng hoạt động)         | = 0 → alert ngay    |
| `TunnelDataIn`        | Bytes nhận qua VPN tunnel                              | 0 trong 5 phút → kiểm tra |
| `TunnelDataOut`       | Bytes gửi qua VPN tunnel                               | 0 trong 5 phút → kiểm tra |

**Alarm quan trọng nhất cho VPN:**

```bash
# Cảnh báo khi VPN tunnel xuống
aws cloudwatch put-metric-alarm \
  --alarm-name "VPN-Tunnel-Down" \
  --metric-name TunnelState \
  --namespace AWS/VPN \
  --dimensions Name=VpnId,Value=vpn-0a1b2c3d \
               Name=TunnelIpAddress,Value=203.0.113.1 \
  --statistic Minimum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:vpn-alerts
```

### Client VPN (VPN Khách Hàng — Remote Access)

| Metric                    | Mô Tả                               |
|---------------------------|-------------------------------------|
| `ActiveConnectionsCount`  | Số kết nối Client VPN đang hoạt động |
| `AuthenticationFailures`  | Số lần xác thực thất bại            |
| `IngressBytes`            | Bytes nhận từ clients               |
| `EgressBytes`             | Bytes gửi đến clients               |

---

## 6. Direct Connect Metrics (Kết Nối Chuyên Dụng)

| Metric               | Mô Tả                                              | Ngưỡng Alert          |
|----------------------|----------------------------------------------------|-----------------------|
| `ConnectionState`    | 1 = UP, 0 = DOWN                                   | = 0 → critical alert  |
| `ConnectionBpsIngress` | Bits per second đi vào qua DX connection         | > 80% capacity        |
| `ConnectionBpsEgress`  | Bits per second đi ra qua DX connection          | > 80% capacity        |
| `ConnectionPpsIngress` | Packets per second đi vào                        |                       |
| `ConnectionPpsEgress`  | Packets per second đi ra                         |                       |
| `ConnectionCRCErrorCount` | CRC errors (lỗi kiểm tra dư thừa vòng) — chỉ ra vấn đề vật lý | > 0 → investigate |
| `ConnectionLightLevelTx` | Light level dBm (decibel-milliwatt) khi gửi — chỉ ra chất lượng cáp quang | < -30 dBm |
| `ConnectionLightLevelRx` | Light level dBm khi nhận                        | < -30 dBm             |

---

## 7. NAT Gateway Metrics

| Metric                      | Mô Tả                                                    |
|-----------------------------|----------------------------------------------------------|
| `ActiveConnectionCount`     | Kết nối TCP/UDP đang hoạt động đồng thời                 |
| `ConnectionEstablishedCount`| Kết nối mới được tạo mỗi phút                            |
| `ConnectionAttemptCount`    | Số lần cố kết nối                                        |
| `BytesInFromDestination`    | Bytes nhận từ internet về NAT Gateway                    |
| `BytesInFromSource`         | Bytes nhận từ private subnet đến NAT Gateway             |
| `BytesOutToDestination`     | Bytes gửi từ NAT Gateway ra internet                     |
| `BytesOutToSource`          | Bytes gửi từ NAT Gateway vào private subnet              |
| `ErrorPortAllocation`       | NAT Gateway không thể cấp phát port — hết port            |
| `PacketDropCount`           | Packets bị drop                                          |

**Khi `ErrorPortAllocation` > 0 → NAT Gateway đang thiếu port:**
```
Nguyên nhân: Quá nhiều concurrent connections từ private subnet
Giải pháp: Tạo thêm NAT Gateway, dùng NAT Gateway cho từng AZ riêng
```

---

## 8. Transit Gateway Metrics

| Metric                      | Mô Tả                                                      |
|-----------------------------|------------------------------------------------------------|
| `BytesIn`                   | Bytes nhận bởi Transit Gateway                            |
| `BytesOut`                  | Bytes gửi từ Transit Gateway                              |
| `PacketsIn`                 | Packets nhận                                              |
| `PacketsOut`                | Packets gửi                                               |
| `PacketDropCountBlackhole`  | Packets bị drop vì route đến blackhole (lỗ đen định tuyến) |
| `PacketDropCountNoRoute`    | Packets bị drop vì không tìm thấy route                   |
| `BytesDropCountBlackhole`   | Bytes bị drop vì blackhole route                          |
| `BytesDropCountNoRoute`     | Bytes bị drop vì no route                                  |

---

## 9. Tạo CloudWatch Dashboard Cho Network

### Dashboard Tổng Hợp — Mẫu Terraform

```hcl
resource "aws_cloudwatch_dashboard" "network_overview" {
  dashboard_name = "NetworkOverview"

  dashboard_body = jsonencode({
    widgets = [
      # Widget 1: ALB Request Count
      {
        type = "metric"
        properties = {
          title   = "ALB - Request Count & Error Rate"
          period  = 300
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", var.alb_name],
            ["AWS/ApplicationELB", "HTTPCode_ELB_5XX_Count", "LoadBalancer", var.alb_name]
          ]
          view = "timeSeries"
        }
      },
      # Widget 2: Target Response Time
      {
        type = "metric"
        properties = {
          title   = "ALB - Target Response Time (P99)"
          period  = 60
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", var.alb_name, { stat = "p99" }]
          ]
        }
      },
      # Widget 3: Healthy/Unhealthy Hosts
      {
        type = "metric"
        properties = {
          title   = "ALB - Host Health"
          metrics = [
            ["AWS/ApplicationELB", "HealthyHostCount", "LoadBalancer", var.alb_name],
            ["AWS/ApplicationELB", "UnHealthyHostCount", "LoadBalancer", var.alb_name]
          ]
        }
      },
      # Widget 4: CloudFront Cache Hit Rate
      {
        type = "metric"
        properties = {
          title   = "CloudFront - Cache Hit Rate"
          metrics = [
            ["AWS/CloudFront", "CacheHitRate", "DistributionId", var.cloudfront_id, "Region", "Global"]
          ]
        }
      },
      # Widget 5: VPN Tunnel Status
      {
        type = "metric"
        properties = {
          title   = "VPN Tunnel State (1=UP, 0=DOWN)"
          metrics = [
            ["AWS/VPN", "TunnelState", "VpnId", var.vpn_id]
          ]
        }
      }
    ]
  })
}
```

---

## 10. CloudWatch Composite Alarms (Cảnh Báo Tổng Hợp)

Composite Alarms (Cảnh Báo Tổng Hợp) kết hợp nhiều alarms lại để giảm alarm noise (tiếng ồn cảnh báo):

```bash
# Alarm tổng hợp: chỉ alert khi CẢ HAI điều kiện đều vi phạm
aws cloudwatch put-composite-alarm \
  --alarm-name "ALB-Critical-Issue" \
  --alarm-rule "ALARM(ALB-High-5XX-Rate) AND ALARM(ALB-High-Latency)" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:pagerduty-critical \
  --alarm-description "ALB has both high 5xx errors AND high latency simultaneously"
```

---

## 11. Network Performance Monitoring (NPM)

### CloudWatch Agent với Nhật Ký Hệ Thống

```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "net": {
        "measurement": [
          "bytes_recv",
          "bytes_sent",
          "drop_in",
          "drop_out",
          "err_in",
          "err_out"
        ],
        "resources": ["eth0"]
      }
    }
  }
}
```

### Enhanced Networking Metrics (Mạng Hiệu Suất Cao)

Với EC2 instances hỗ trợ Enhanced Networking (ENA — Elastic Network Adapter):

```bash
# Xem network performance metrics từ EC2
ethtool -S eth0 | grep -E "(bw_in|bw_out|pps_in|pps_out|allowance)"
# bw_in_allowance_exceeded: bị throttle vào
# bw_out_allowance_exceeded: bị throttle ra
# conntrack_allowance_exceeded: hết connection tracking table
```

---

## 12. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Metrics nào quan trọng nhất cần monitor cho ALB?**

> A: Ưu tiên theo thứ tự: (1) `UnHealthyHostCount` — nếu > 0, ứng dụng đang có vấn đề; (2) `HTTPCode_ELB_5XX_Count` — lỗi server-side; (3) `TargetResponseTime` P99 — trải nghiệm người dùng; (4) `RequestCount` — phát hiện traffic spike bất thường.

**Q: Sự khác biệt giữa `HTTPCode_ELB_5XX` và `HTTPCode_Target_5XX` là gì?**

> A: `HTTPCode_ELB_5XX` là lỗi do bản thân ALB tạo ra (ví dụ: không thể kết nối đến target, timeout). `HTTPCode_Target_5XX` là lỗi từ backend targets (EC2, containers) trả về cho ALB. Khi debug, phân biệt hai loại này giúp xác định vấn đề ở layer nào.

**Q: Làm thế nào để giám sát CacheHitRate của CloudFront?**

> A: Bật Additional Metrics trong CloudFront distribution settings (tốn phí). Dùng CloudWatch metric `CacheHitRate` với dimension `DistributionId` và `Region=Global`. Mục tiêu > 80%. Nếu thấp → kiểm tra Cache-Control headers của origin, TTL settings, và query string handling.

**Q: Metric nào cho biết VPN tunnel đang down?**

> A: Metric `TunnelState` trong namespace `AWS/VPN`. Giá trị 1 = UP, 0 = DOWN. Luôn tạo alarm với `statistic=Minimum`, `period=60`, `threshold=1`, `comparison=LessThanThreshold` để phát hiện ngay khi tunnel down.

---

## Tóm Tắt

| Dịch Vụ          | Metrics Quan Trọng Nhất                                          |
|------------------|------------------------------------------------------------------|
| ALB              | UnHealthyHostCount, 5XX errors, TargetResponseTime P99          |
| NLB              | HealthyHostCount, TCP_Target_Reset_Count, ActiveFlowCount        |
| CloudFront       | CacheHitRate, 5xxErrorRate, OriginLatency                       |
| VPN Site-to-Site | TunnelState (0=DOWN → alert ngay)                               |
| Direct Connect   | ConnectionState, CRCErrorCount, bandwidth utilization            |
| NAT Gateway      | ErrorPortAllocation, ActiveConnectionCount                       |
| Transit Gateway  | PacketDropCountBlackhole, PacketDropCountNoRoute                 |

**Tiếp Theo:** [3-network-access-analyzer.md](./3-network-access-analyzer.md) — Phân tích quyền truy cập mạng tự động
