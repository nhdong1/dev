# Observability Checklist — Danh Sách Kiểm Tra Giám Sát Toàn Diện

> Checklist (Danh Sách Kiểm Tra) này giúp bạn đánh giá mức độ observability (khả năng quan sát) của hệ thống mạng AWS trong môi trường production. Sử dụng như một runbook (tài liệu vận hành) khi setup hệ thống mới hoặc audit hệ thống hiện tại.

---

## 1. Tại Sao Cần Checklist?

Trong thực tế, những sự cố mạng nghiêm trọng nhất thường xảy ra vì **thiếu visibility** (khả năng nhìn thấy):

```
"Chúng tôi không biết VPN tunnel đã down từ 3 giờ sáng
 cho đến khi khách hàng ở Mỹ bắt đầu gọi điện lúc 8 giờ sáng"

"Database connection timeout xảy ra 2 tuần trước deployment,
 nhưng chúng tôi không có Flow Logs nên không biết IP nào đang bị block"

"CloudFront cache hit rate giảm từ 85% xuống 20%,
 chi phí băng thông tăng 400% trong 1 tuần mà không ai hay biết"
```

Checklist này ngăn bạn rơi vào những tình huống trên.

---

## 2. Checklist Theo Tầng (Layer-by-Layer)

### 🔵 Tầng 1 — VPC & Flow Logs

#### Thiết Lập Cơ Bản

- [ ] **VPC Flow Logs đã được bật** cho toàn bộ VPC production
  - Traffic type: `ALL` (ghi cả ACCEPT và REJECT)
  - Aggregation interval: 1 phút (không phải 10 phút) cho debug nhanh hơn
  - Destination: CloudWatch Logs (7 ngày) + S3 (90+ ngày)

- [ ] **Flow Logs gửi đến đúng Log Group** trong CloudWatch
  - Log Group name theo convention: `/aws/vpc/flow-logs/{vpc-name}`
  - Retention policy: ít nhất 7 ngày (30 ngày cho compliance)

- [ ] **IAM Role** có đủ quyền ghi CloudWatch Logs

- [ ] **S3 bucket** (nếu dùng S3 destination) có:
  - Bucket policy cho phép VPC Flow Logs service ghi
  - Lifecycle policy: archive sang Glacier sau 90 ngày
  - Versioning và MFA Delete bật (bảo vệ audit trail)

#### CloudWatch Logs Insights Queries Sẵn Sàng

- [ ] Query tìm REJECT records đã được lưu vào Saved Queries
- [ ] Query phát hiện port scanning đã được lưu
- [ ] Query top talkers (nguồn traffic lớn nhất) đã được lưu
- [ ] Query traffic ra ngoài internet trên port nhạy cảm đã được lưu

#### Athena (Nếu Dùng S3)

- [ ] Athena table đã được tạo và trỏ đúng vào S3 bucket
- [ ] Partitioning theo ngày để tối ưu query cost
- [ ] Workgroup với query result location đã cấu hình

---

### 🟢 Tầng 2 — Load Balancer Monitoring

#### ALB (Application Load Balancer)

- [ ] **CloudWatch Alarms đã tạo cho:**
  - `UnHealthyHostCount` > 0 trong 2 phút → CRITICAL alert
  - `HTTPCode_ELB_5XX_Count` rate > 1% trong 5 phút → HIGH alert
  - `TargetResponseTime` P99 > 3 giây trong 5 phút → MEDIUM alert
  - `RejectedConnectionCount` > 0 → HIGH alert (ALB đang bị overload)

- [ ] **Access Logs đã bật** (khác với CloudWatch Metrics):
  - S3 bucket destination đã cấu hình
  - Retention ít nhất 30 ngày
  - Log có thể query bằng Athena

- [ ] **Target Group Health Checks** được cấu hình đúng:
  - Health check path thực sự phản ánh app health (không chỉ `/`)
  - Threshold hợp lý: unhealthy threshold = 2 (không quá nhạy)
  - Healthy threshold = 3 (không quá chậm recovery)

#### NLB (Network Load Balancer)

- [ ] Alarm cho `HealthyHostCount` < minimum
- [ ] Alarm cho `TCP_Target_Reset_Count` cao bất thường
- [ ] Monitor `ConsumedLCUs` để dự đoán chi phí

---

### 🟡 Tầng 3 — CloudFront & CDN

- [ ] **Additional Metrics đã bật** (tốn phí nhưng cần thiết):
  - `CacheHitRate` — target > 80%
  - `OriginLatency` — alert khi P99 > 2 giây
  - `4xxErrorRate`, `5xxErrorRate` — alert theo ngưỡng

- [ ] **CloudFront Access Logs đã bật**:
  - S3 bucket destination
  - Có thể query với Athena để phân tích traffic theo country, URI, User-Agent

- [ ] **Alarm cho 5xxErrorRate** > 1% trong 5 phút

- [ ] **Dashboard Widget** cho CacheHitRate theo thời gian

- [ ] **Origin Shield** (Lá Chắn Nguồn) đã cân nhắc để giảm origin load

---

### 🔴 Tầng 4 — VPN & Direct Connect

#### Site-to-Site VPN

- [ ] **Alarm CRITICAL cho TunnelState = 0** (tunnel down)
  - Statistic: Minimum
  - Period: 60 giây
  - Evaluation: 1 phút (alert ngay lập tức)
  - Action: PagerDuty / SNS → on-call

- [ ] **Alarm cho cả 2 tunnels** (redundant tunnels phải monitor riêng)
- [ ] **Kiểm tra TunnelDataIn/Out** = 0 trong 5 phút → có thể tunnel up nhưng traffic không chạy

- [ ] **BGP (Border Gateway Protocol — Giao Thức Định Tuyến Biên Giới) session monitor** (nếu dùng dynamic routing):
  - BGP route count bình thường
  - BGP peer state: Established

#### Direct Connect

- [ ] **Alarm CRITICAL cho ConnectionState = 0**
- [ ] **Alarm cho bandwidth utilization** > 80% (gần đến giới hạn)
- [ ] **Monitor CRCErrorCount** > 0 → vấn đề vật lý cáp quang
- [ ] **Monitor LightLevel** trong ngưỡng cho phép của nhà cung cấp

---

### 🟣 Tầng 5 — DNS & Route 53

- [ ] **Health Checks đã cấu hình** cho mọi endpoint critical:
  - Protocol phù hợp: HTTP/HTTPS/TCP
  - Path kiểm tra thực sự app health
  - Failure threshold hợp lý (3 failures)
  - CloudWatch alarm gắn với health check

- [ ] **DNS Failover** được test định kỳ:
  - Primary endpoint bị tắt → DNS tự failover sang secondary không?
  - TTL thấp đủ để failover nhanh (60 giây hoặc ít hơn)

- [ ] **Alarm cho health check failures** trong CloudWatch

- [ ] **Route 53 Resolver Logs** (nếu có private hosted zones):
  - Lưu DNS query logs để debug DNS resolution failures

---

### ⚪ Tầng 6 — Security Monitoring

- [ ] **GuardDuty đã bật** (tự động phân tích Flow Logs):
  - Threat intelligence feeds (thông tin tình báo mối đe dọa) được bật
  - Findings được gửi đến Security Hub
  - Auto-remediation Lambda đã cấu hình cho high severity findings

- [ ] **AWS Config Rules đã bật**:
  - `vpc-sg-open-only-to-authorized-ports` — SG không mở port nhạy cảm ra internet
  - `restricted-ssh` — không có SG nào cho phép 0.0.0.0/0 → port 22
  - `restricted-common-ports` — port 3306, 5432, 1433 không expose ra internet
  - `vpc-flow-logs-enabled` — mọi VPC đều có Flow Logs

- [ ] **CloudTrail đã bật** cho network changes:
  - Ghi lại: SecurityGroup modifications, RouteTable changes, NACL changes
  - Log group retention ít nhất 90 ngày
  - Alert khi có thay đổi Security Group ngoài giờ làm việc

- [ ] **Network Access Analyzer scope** đã tạo để kiểm tra:
  - Internet → Production resources (chạy weekly)
  - Dev/Test → Production database (chạy sau mỗi infrastructure deploy)

---

### 🟤 Tầng 7 — NAT Gateway & Egress Monitoring

- [ ] **Alarm cho ErrorPortAllocation** > 0:
  - Nghĩa là NAT Gateway hết port → connections bị drop
  - Action: Scale out hoặc tạo thêm NAT Gateway per AZ

- [ ] **Monitor BytesOutToDestination** theo thời gian:
  - Đột tăng bất thường → có thể data exfiltration (rò rỉ dữ liệu)
  - Kết hợp với Flow Logs để xác định nguồn

- [ ] **Mỗi AZ có NAT Gateway riêng** (không dùng chung cross-AZ):
  - Giảm chi phí cross-AZ data transfer
  - Tăng high availability

---

## 3. Dashboard — Tổng Hợp Toàn Diện

### Dashboard "Network Health Overview" Nên Có

```
Row 1: Load Balancers
  Widget 1.1: ALB - RequestCount (24h trend)
  Widget 1.2: ALB - 5XX Error Rate %
  Widget 1.3: ALB - Target Response Time P99
  Widget 1.4: ALB - Healthy vs Unhealthy Hosts

Row 2: CloudFront
  Widget 2.1: Cache Hit Rate % (target: >80%)
  Widget 2.2: Requests by Distribution
  Widget 2.3: Origin Latency P99
  Widget 2.4: Error Rate (4xx + 5xx)

Row 3: Connectivity
  Widget 3.1: VPN Tunnel State (1=UP, 0=DOWN)
  Widget 3.2: VPN Data In/Out
  Widget 3.3: Direct Connect State
  Widget 3.4: Direct Connect Bandwidth Utilization

Row 4: VPC & NAT
  Widget 4.1: NAT Gateway - Active Connections
  Widget 4.2: NAT Gateway - ErrorPortAllocation
  Widget 4.3: Route 53 Health Check Status
  Widget 4.4: Flow Logs - REJECT count (last 24h)
```

---

## 4. Runbook — Phản Ứng Khi Nhận Alarm

### Khi Nhận Alarm: "UnHealthyHostCount > 0"

```
Bước 1: Kiểm tra CloudWatch Logs của target (Application Logs)
         → Log group của EC2/ECS/Lambda

Bước 2: Kiểm tra ALB Access Logs
         → Xem error response từ target là gì (5xx code cụ thể)

Bước 3: Chạy Reachability Analyzer
         → Source: ALB → Destination: unhealthy target
         → Có thể Security Group rule bị thay đổi?

Bước 4: Kiểm tra target health check settings
         → Path đúng không? Port đúng không?
         → App có đang restart loop không?

Bước 5: Nếu app issue → escalate đến dev team
         Nếu network issue → fix SG/Route Table
```

### Khi Nhận Alarm: "VPN TunnelState = 0"

```
Bước 1: Kiểm tra CloudWatch Metrics TunnelDataIn/Out
         → Có tunnel nào còn UP không? (AWS cung cấp 2 tunnels)

Bước 2: Liên hệ network admin on-premises
         → Customer Gateway device có issue không?
         → BGP session có established không?

Bước 3: Kiểm tra AWS VPN Console
         → Tunnel 1 và Tunnel 2 status
         → Last state change timestamp

Bước 4: Nếu cả 2 tunnels đều down → failover traffic
         → Route 53 health check có tự redirect không?
         → Còn Direct Connect backup không?

Bước 5: Tạo AWS Support ticket nếu issue từ phía AWS
```

### Khi Nhận Alarm: "CloudFront 5xxErrorRate > 1%"

```
Bước 1: Kiểm tra CloudFront Access Logs
         → URI nào đang có 5xx? Origin nào?
         → Có phải một path cụ thể hay toàn bộ?

Bước 2: Kiểm tra Origin (ALB/EC2/S3)
         → Origin có healthy không? (ALB healthy host count)
         → S3 bucket có accessible không?

Bước 3: Kiểm tra CloudFront Error caching settings
         → Error TTL có đang cache lỗi quá lâu không?

Bước 4: Test từ nhiều regions
         → Chỉ một region bị lỗi hay toàn cầu?
         → Dùng: https://www.whatsmydns.net/ để test từ nhiều locations

Bước 5: Invalidate cache nếu cần thiết
         aws cloudfront create-invalidation --distribution-id EXXXXXX --paths "/*"
```

---

## 5. Định Kỳ Maintenance (Bảo Trì Định Kỳ)

### Hàng Tuần

- [ ] Review Flow Logs REJECT records bất thường
- [ ] Kiểm tra GuardDuty findings → có gì mới không?
- [ ] Review AWS Config compliance → có vi phạm mới không?
- [ ] Chạy Network Access Analyzer → có unexpected findings không?
- [ ] Kiểm tra CloudFront cache hit rate trend

### Hàng Tháng

- [ ] Review và tối ưu CloudWatch Alarm thresholds (ngưỡng có còn phù hợp không?)
- [ ] Kiểm tra cost của monitoring (CloudWatch, Flow Logs, Athena queries)
- [ ] Test failover thực tế (tắt primary để verify secondary hoạt động)
- [ ] Review IAM permissions cho monitoring roles
- [ ] Cập nhật expected findings trong Network Access Analyzer

### Hàng Quý

- [ ] Full security audit: chạy AWS Security Hub findings review
- [ ] Review VPC Flow Logs retention policy (có cần adjust không?)
- [ ] Kiểm tra và cập nhật runbooks
- [ ] Penetration testing (kiểm thử xâm nhập) hoặc security assessment
- [ ] Review Direct Connect bandwidth capacity vs. actual usage
- [ ] Kiểm tra certificate expiry (chứng chỉ SSL sắp hết hạn không?)

---

## 6. Công Cụ Bổ Sung Nên Cân Nhắc

### AWS Native Tools

| Công Cụ                    | Chức Năng                                              | Khi Nào Dùng                      |
|----------------------------|--------------------------------------------------------|-----------------------------------|
| AWS Security Hub           | Tổng hợp security findings từ nhiều dịch vụ           | Bắt buộc cho production           |
| Amazon Detective           | Điều tra security incidents với graph analysis        | Khi có security incident          |
| AWS X-Ray                  | Distributed tracing (theo dõi phân tán) ở layer ứng dụng | Microservices                  |
| VPC Traffic Mirroring      | Copy traffic thực để phân tích sâu                    | Advanced security investigation   |
| Amazon Macie               | Phát hiện dữ liệu nhạy cảm trong S3                   | Data classification/compliance    |

### Third-Party Tools (Công Cụ Bên Thứ Ba)

| Công Cụ        | Điểm Mạnh                                                |
|----------------|----------------------------------------------------------|
| Datadog NPM    | Network Performance Monitoring với service map đẹp       |
| Kentik         | Network analytics và DDoS detection chuyên sâu           |
| Sumo Logic     | Log analytics với machine learning anomaly detection     |
| Grafana        | Dashboard visualization với nhiều data sources          |
| PagerDuty      | On-call rotation và incident management                  |

---

## 7. Scorecard — Đánh Giá Mức Độ Observability

Tính điểm hệ thống của bạn:

```
Điểm 0 — Chưa Có Gì (Critical Risk)
  ✗ Không có Flow Logs
  ✗ Không có CloudWatch Alarms
  ✗ Không có monitoring nào cả

Điểm 1-3 — Cơ Bản (High Risk)
  ✓ Flow Logs bật nhưng chưa query
  ✓ Một vài alarms cơ bản
  ✗ Chưa có runbooks

Điểm 4-6 — Trung Bình (Medium Risk)
  ✓ Flow Logs + CloudWatch Alarms đầy đủ
  ✓ Dashboard tổng quan
  ✓ Runbooks cơ bản
  ✗ Chưa test failover định kỳ

Điểm 7-9 — Tốt (Low Risk)
  ✓ Tất cả items trên
  ✓ GuardDuty + AWS Config + CloudTrail
  ✓ Network Access Analyzer chạy định kỳ
  ✓ Failover được test hàng tháng

Điểm 10 — Xuất Sắc (Minimal Risk)
  ✓ Tất cả items trên
  ✓ Automated remediation (tự động khắc phục)
  ✓ Game days (ngày diễn tập sự cố) định kỳ
  ✓ Chaos engineering (kỹ thuật hỗn loạn) để test resilience
  ✓ Full observability với Logs + Metrics + Traces
```

---

## 8. Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Làm thế nào bạn thiết lập monitoring cho một VPC production mới?**

> A: Theo thứ tự ưu tiên: (1) Bật VPC Flow Logs với destination CloudWatch Logs + S3 ngay khi tạo VPC; (2) Bật CloudTrail để track all API calls; (3) Cài CloudWatch Alarms cho ALB metrics và VPN tunnel state; (4) Bật GuardDuty để auto-analyze Flow Logs; (5) Tạo Network Access Analyzer scope để kiểm tra internet access; (6) Xây dựng CloudWatch Dashboard tổng hợp; (7) Tạo runbooks cho từng loại alert.

**Q: Bạn sẽ phát hiện data exfiltration (rò rỉ dữ liệu) bằng cách nào?**

> A: Nhiều lớp phát hiện: (1) GuardDuty tự động phát hiện traffic đến C&C servers và DNS exfiltration; (2) VPC Flow Logs + CloudWatch Alarm khi `BytesOutToDestination` của NAT Gateway tăng đột biến; (3) Athena query phân tích top destinations theo bytes; (4) Network Access Analyzer kiểm tra có đường đi ngoài ý muốn đến internet không; (5) VPC Traffic Mirroring để deep packet inspection nếu cần điều tra sâu.

**Q: Mức độ retention (lưu giữ) nào cho Flow Logs là hợp lý?**

> A: Phụ thuộc vào compliance requirements: PCI-DSS yêu cầu 1 năm; HIPAA thường 6 năm; không có yêu cầu đặc biệt thì 90 ngày là baseline hợp lý. Thực hành tốt: CloudWatch Logs 7-30 ngày để query nhanh, S3 Standard 90 ngày, S3 Glacier cho 1+ năm. Dùng S3 Lifecycle Policy để tự động transition giữa các tiers.

---

## Tóm Tắt — Thứ Tự Ưu Tiên Triển Khai

```
Tuần 1 — Must Have (Bắt Buộc):
  ✓ VPC Flow Logs → CloudWatch Logs
  ✓ ALB Alarms: UnHealthyHostCount, 5XX rate
  ✓ VPN TunnelState alarm
  ✓ CloudTrail bật

Tuần 2 — Should Have (Nên Có):
  ✓ Flow Logs → S3 + Athena table
  ✓ CloudFront Additional Metrics
  ✓ GuardDuty bật
  ✓ Network Dashboard tổng quan

Tuần 3-4 — Good to Have (Tốt Khi Có):
  ✓ AWS Config Rules
  ✓ Network Access Analyzer scopes
  ✓ Security Hub
  ✓ Runbooks hoàn chỉnh
  ✓ Failover testing định kỳ
```

---

**Kết Thúc Topic 08 — Monitoring.** Tiếp theo xem [09-troubleshooting/](../09-troubleshooting/) để học cách xử lý sự cố mạng có hệ thống.
